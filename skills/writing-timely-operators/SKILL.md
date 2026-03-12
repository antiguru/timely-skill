---
name: writing-timely-operators
description: >
  Guide for writing custom timely dataflow operators. Use this skill whenever the user
  asks to implement, design, or debug a timely operator — including questions about
  capability management, input draining, frontier handling, exchange patterns, or
  operator scheduling. Also trigger when the user writes operator code and you need
  to review it for correctness.
---

# Writing timely dataflow operators

This skill targets **timely v0.27**.
API details (especially closure signatures and container traits) may differ in other versions — check source files when in doubt.

This skill covers the patterns for writing correct and efficient custom operators in timely dataflow.
When writing an operator, read the relevant section below, then check the source files it references for the current API signatures.

## All workers must construct the same dataflow graph

Every worker must create every operator and every stream, even if only one worker does useful work.
Timely expects matching progress messages from all workers.
Rendering an operator on a subset of workers causes crashes or hangs because the missing workers never send the expected progress updates.

To restrict work to one worker, build the operator on all workers but have non-active workers skip the work (see "Single-worker operators" below).

## Choosing an operator shape

Timely provides several operator constructors on the `Operator` trait (`timely/src/dataflow/operators/generic/operator.rs`).
Choose based on how many inputs you need and whether you need frontier information:

| Constructor | Inputs | Frontier? | Use when |
|---|---|---|---|
| `unary` | 1 | No | Stateless transforms (map, filter, flat_map) |
| `unary_frontier` | 1 | Yes | Stateful operators that buffer across timestamps |
| `binary` | 2 | No | Joining/merging two streams statelessly |
| `binary_frontier` | 2 | Yes | Stateful joins, lookups against a changing collection |
| `sink` | 1 | Yes | Side-effects only, no output stream |
| `OperatorBuilder` | N | Yes | More than two inputs or outputs |

`unary`, `unary_frontier`, `binary`, and `binary_frontier` follow the same two-level closure pattern.
`sink` is simpler — it takes the logic closure directly, without a constructor.

```
.unary(pact, "Name", |initial_capability, operator_info| {
    // Constructor: runs once. Allocate operator state here.
    let mut state = HashMap::new();

    move |input, output| {
        // Logic: runs each time the operator is scheduled.
        // Drain input, update state, produce output.
    }
})
```

The constructor receives a `Capability<T>` at `T::minimum()` and an `OperatorInfo` with the operator's address.
It returns the logic closure that the scheduler invokes repeatedly.

For the frontier variants, the logic closure receives input handles paired with their frontier:

```
.unary_frontier(pact, "Name", |cap, info| {
    move |(input, frontier), output| {
        // `frontier` is &MutableAntichain<T>
        // Use frontier.less_than() / frontier.less_equal() to check progress
    }
})
```

For `OperatorBuilder` (in `builder_rc.rs`), you wire inputs and outputs manually:

```
let mut builder = OperatorBuilder::new("Name".to_owned(), scope.clone());
let mut input = builder.new_input(&stream, Pipeline);
let (mut output_wrapper, output_stream) = builder.new_output();

builder.build(|capabilities| {
    // capabilities: Vec<Capability<T>>, one per output
    move |frontiers| {
        // frontiers: &[MutableAntichain<T>], one per input
    }
});
```

By default, `new_input` and `new_output` assume full connectivity: every input can produce output at every output with the default (identity) path summary.
Use `new_input_connection` and `new_output_connection` to declare sparse connectivity — which inputs can affect which outputs, and with what timestamp summary.
This enables timely's progress tracker to reason more precisely: if an input is not connected to an output, messages on that input cannot block progress at that output.

```
// Output connected only to input 0, with default summary
let (output, stream) = builder.new_output_connection(
    vec![(0, Antichain::from_elem(Default::default()))]
);

// Input connected only to output 1, with a summary that increments by 1
let input = builder.new_input_connection(&stream, Pipeline,
    vec![(1, Antichain::from_elem(1))]
);
```

## Input draining

Operators should drain all available input each time they are scheduled.
Not draining causes the operator to be rescheduled, wasting cycles and delaying progress.

Three methods are available on `InputHandleCore` (in `handles.rs`):

**`for_each_time`** (preferred): Groups input containers by timestamp before calling your closure.
One call per distinct timestamp, receiving an iterator over all containers at that time.
This is the natural fit for creating one output session per timestamp:

```
input.for_each_time(|time, data| {
    let mut session = output.session(&time);
    for container in data {
        for item in container.drain(..) {
            session.give(transform(item));
        }
    }
});
```

**`for_each`**: Calls your closure once per (capability, container) pair, without grouping.
Use only when you do not need to create output sessions (e.g., stashing into a buffer).

**`next()`**: Returns `Option<(InputCapability<T>, &mut C)>` one batch at a time.
Use when you need early exit or custom control flow.

After draining, if the operator needs to defer work (because the frontier has not advanced far enough), stash the data in operator-local state keyed by timestamp.

## Capability handling

Capabilities are the mechanism that connects operator behavior to progress tracking.
Holding a capability for timestamp `t` at an output tells the system: "I might still produce data at time `t`."
Dropping all capabilities for `t` tells the system: "I am done with `t`, downstream can proceed."

**Getting capabilities.** The constructor receives one at `T::minimum()`.
Input handles yield `InputCapability<T>` values when draining.
Convert an input capability to an output capability with:

* `time.retain(output.output_index())` — same timestamp, usable for output
* `time.delayed(&new_time, output.output_index())` — a future timestamp (must be `>=` current)

**Stashing capabilities.** When buffering data for later emission, store the retained capability alongside the data.
The capability keeps the timestamp "open" so you can produce output later.
A common pattern is `HashMap<Capability<T>, Vec<D>>`.

**Dropping capabilities.** Drop capabilities as soon as you are done with a timestamp.
Holding stale capabilities blocks progress for all downstream operators.
Use `stash.retain(|_cap, data| !data.is_empty())` after emitting to clean up.

**Downgrading.** `cap.downgrade(&new_time)` replaces a capability with one at a later time (in-place).
`cap.delayed(&new_time)` creates a new capability without consuming the original.
Both require `new_time >= cap.time()`.

**`CapabilitySet`** (in `capability.rs`): Manages a set of incomparable capabilities for partial-order timestamps.
Maintains the minimal antichain. Use `insert()` to add, `downgrade()` to advance the entire set.
Relevant whenever timestamps are `Product<T1, T2>` (nested scopes, iterative computations).

## Frontier-driven processing

The frontier of an input tells you which timestamps are still possible.
`frontier.less_equal(time)` means data at `time` might still arrive — do not finalize that time yet.
`!frontier.less_equal(time)` means `time` is complete — safe to process and emit.

The standard pattern for a stateful operator:

```
.unary_frontier(pact, "Aggregate", |_cap, _info| {
    let mut stash: HashMap<Capability<T>, Vec<D>> = HashMap::new();

    move |(input, frontier), output| {
        // 1. Drain input into stash
        input.for_each_time(|time, data| {
            stash.entry(time.retain(output.output_index()))
                 .or_default()
                 .extend(data.flat_map(|d| d.drain(..)));
        });

        // 2. Emit completed timestamps
        for (cap, data) in stash.iter_mut() {
            if !frontier.less_equal(cap.time()) {
                output.session(cap).give_iterator(data.drain(..));
            }
        }

        // 3. Drop empty entries (releases capabilities)
        stash.retain(|_cap, data| !data.is_empty());
    }
})
```

Step 3 is not just housekeeping — it is essential for progress.
If you skip the `retain`, dropped entries still hold capabilities, blocking downstream.

**`FrontierNotificator`** (in `notificator.rs`) automates this pattern.
You call `notificator.notify_at(cap)` when stashing, and later iterate ready notifications:

```
notificator.for_each(&[frontier], |cap, _count, _not| {
    if let Some(data) = stash.remove(cap.time()) {
        output.session(&cap).give_iterator(data.into_iter());
    }
});
```

The notificator delivers capabilities in partial-order-respecting sequence and deduplicates multiple requests for the same time.

## Total vs. partial orders

Timely timestamps implement `PartialOrder`.
Some timestamps additionally implement `TotalOrder` (a marker trait with no new methods).
The distinction matters for how you reason about frontiers and stashed state.

**Total order** (e.g., `usize`, `u64`): The frontier is always a single time.
"Not less_equal" means strictly greater. You can process stashed data in simple sorted order.
Iterating `stash` keys from smallest to largest and emitting everything below the frontier is correct and efficient.

**Partial order** (e.g., `Product<u64, u64>` in iterative computations): The frontier is an antichain of multiple incomparable times.
"Not less_equal" means no element of the frontier is `<=` the time — but there may be incomparable frontier elements.
You must check each stashed time against the full antichain.
Use `CapabilitySet` to manage multiple live capabilities.

When writing a generic operator, do not assume total order unless you add a `where T: TotalOrder` bound.
If you can specialize for total order, it often simplifies the logic and enables optimizations (e.g., a single "watermark" instead of an antichain scan).

## Antichain partial order

`PartialOrder` for `Antichain<T>`: `a <= b` iff every element in `b` has some element in `a` that is `<=` it.
This means:

* `{0} <= {}` is **true** (vacuously — no elements in `{}` to check)
* `{} <= {0}` is **false**
* The empty antichain `{}` is the **maximum** (most advanced frontier)

An empty frontier means "no more data will ever arrive."
A frontier containing `T::minimum()` means "any timestamp is still possible."

## Run to completion vs. yielding

By default, an operator's logic closure runs to completion each time it is scheduled.
The scheduler invokes it when input is available or when a frontier changes (if `notify_me()` is true, which is the default).

This works well for operators that can process all input promptly.
But if an operator produces far more output than it receives (e.g., `flat_map(|x| 0..x)` on large values), running to completion can flood memory.

**Yielding with `build_reschedule`** (on `OperatorBuilder`): The logic closure returns `bool` — `true` means "schedule me again even without new input."
This lets you process input incrementally:

```
builder.build_reschedule(|caps| {
    let mut stash = VecDeque::new();
    let mut current_work = None;

    move |frontiers| {
        // drain new input into stash
        // ...

        // process a bounded amount of work
        let mut fuel = 1000;
        while fuel > 0 {
            if let Some(item) = current_work.take().or_else(|| stash.pop_front()) {
                // process item, decrement fuel
                fuel -= 1;
            } else {
                break;
            }
        }

        // return true if there's more work to do
        !stash.is_empty() || current_work.is_some()
    }
});
```

**Yielding with `Activator`**: Request explicit re-scheduling from within a standard operator:

```
let activator = scope.activator_for(info.address.clone());

move |input, output| {
    // ... do bounded work ...
    if more_work_remaining {
        activator.activate();
    }
}
```

The activator pattern is available in all operator types, not just `build_reschedule`.

**Flow control with feedback loops**: For dataflows that amplify data (each record produces many), a feedback loop can regulate throughput.
See `mdbook/src/chapter_4/chapter_4_3.md` for the pattern: use `delay` to spread work across timestamps, then a `binary_frontier` operator that buffers records and only emits when the feedback input's frontier confirms prior work has drained.

## Data exchange

The *pact* (parallelization contract) determines how data routes between the upstream operator and yours.

**`Pipeline`**: Data stays on the same worker. Zero overhead — no serialization, no cloning.
Use for operators where locality does not matter (map, filter, inspect) or where the upstream already partitioned the data correctly.

**`Exchange::new(|item| -> u64)`**: Routes each record to `hash % num_workers`.
Every record crosses worker boundaries (serialization + deserialization), though records that happen to target the local worker take a shortcut.
Use when the operator needs all records with a given key on the same worker (aggregation, joins, distinct).

The exchange function should distribute keys uniformly.
A poor distribution (e.g., all records mapping to worker 0) negates parallelism.
For string keys, hash them; do not use `str::len()` (as the wordcount example does — it acknowledges this is "deranged").

**Cost model**: Pipeline is effectively free. Exchange costs serialization of every record plus network transfer for remote workers.
If your operator can tolerate partial results per worker and merge later (e.g., approximate counts), Pipeline avoids exchange overhead entirely.

## Cost of tees

When a `Stream` is consumed by multiple downstream operators, timely inserts a `Tee` (in `channels/pushers/tee.rs`).

* **Single consumer**: The `Tee` wraps a single pusher (`PushOne`). No cloning, no overhead.
* **Two or more consumers**: Upgrades to `PushMany`. Every record is cloned to all consumers except the last.

This means branching a stream of large records (e.g., `Vec<String>`) to N consumers clones each record N-1 times.
If downstream operators need only a subset of the record's fields, consider mapping to a smaller type before the branch point.

To create multiple consumers, `.clone()` the stream or use `branch`/`partition`/`ok_err`:

```
let stream = source.container::<Vec<_>>();
let copy1 = stream.clone(); // implicit Tee
let copy2 = stream;         // last consumer, no clone
```

`branch` and `ok_err` route each record to exactly one output, so they avoid the N-1 clone cost — each record is sent once.
Prefer these over cloning + filtering when the branches are mutually exclusive.

## Container batching

Timely groups records into containers (typically `Vec<T>`) for amortized scheduling and communication overhead.
The `CapacityContainerBuilder` (in `container/src/lib.rs`) accumulates records and flushes when at capacity (~8KB / `size_of::<T>()` elements).

**`give` vs. `give_iterator` vs. `give_container`**: All three go through the container builder.
`give` pushes one item; `give_iterator` pushes an iterator; `give_container` moves an entire pre-built container.
For bulk output, `give_iterator` or `give_container` is more efficient than calling `give` in a loop because the builder can batch flushes.

**`.container::<Vec<_>>()`**: Required before operators like `inspect` that need a concrete container type.
This is a type annotation, not a runtime conversion — it tells the type system which container to use.

## Extension traits for streams

All built-in timely operators (`filter`, `map`, `inspect`, `exchange`, etc.) are defined as extension traits on `Stream`.
Follow this pattern to make custom operators feel like first-class methods.

Define a trait parameterized by the scope and constrain it to `Stream`:

```
use timely::dataflow::{Scope, Stream};
use timely::dataflow::operators::generic::operator::Operator;
use timely::dataflow::channels::pact::Pipeline;

/// Doubles every record in the stream.
pub trait DoubleStream<G: Scope, D: Clone + 'static> {
    fn double(self) -> Self;
}

impl<G: Scope, D: Clone + 'static> DoubleStream<G, D> for Stream<G, Vec<D>> {
    fn double(self) -> Self {
        self.unary(Pipeline, "Double", |_, _| {
            move |input, output| {
                input.for_each_time(|time, data| {
                    let mut session = output.session(&time);
                    for container in data {
                        for item in container.drain(..) {
                            session.give(item.clone());
                            session.give(item);
                        }
                    }
                });
            }
        })
    }
}
```

This pattern has a few advantages:
* Callers chain it naturally: `stream.double().inspect(...)`.
* The trait can be imported selectively, avoiding name collisions.
* Scope and timestamp constraints go on the trait or impl, not on every call site.

For operators that need `OperatorBuilder` (multiple outputs, yielding), the extension trait returns the output stream(s) and hides the builder wiring:

```
pub trait ExpandRange<G: Scope> {
    fn expand_range(self) -> Stream<G, Vec<u64>>;
}

impl<G: Scope<Timestamp = u64>> ExpandRange<G> for Stream<G, Vec<u64>> {
    fn expand_range(self) -> Stream<G, Vec<u64>> {
        let mut builder = OperatorBuilder::new("ExpandRange".to_owned(), self.scope());
        let mut input = builder.new_input(self, Pipeline);
        let (output, stream) = builder.new_output::<Vec<u64>>();
        // ... build_reschedule logic ...
        stream
    }
}
```

For operators that split a stream, return a tuple:

```
pub trait SplitOddEven<G: Scope> {
    fn split_odd_even(self) -> (Stream<G, Vec<u64>>, Stream<G, Vec<u64>>);
}
```
