---
name: writing-differential-operators
description: >
  Guide for writing differential dataflow operators and working with columnar data.
  Use this skill when the user mentions "differential dataflow", "Collection",
  "Arranged", "arrangement", "TraceReader", "Cursor", "reduce_core", "join_core",
  "arrange_by_key", "arrange_core", "consolidate", "Semigroup", "Abelian",
  "delta join", "half_join", "dogsdogsdogs", "Variable", "iterate",
  "columnar", "Columnar trait", or when editing code that uses
  differential_dataflow::collection or differential_dataflow::operators.
---

# Writing differential dataflow operators

This skill targets **differential-dataflow v0.23** (depends on timely v0.29 and columnar v0.12).
API details may differ in other versions — check source files when in doubt.

Differential dataflow builds on timely dataflow.
It represents evolving collections as streams of `(data, time, diff)` updates and provides operators that incrementally maintain their outputs as inputs change.
For timely-level operator construction (capabilities, frontiers, draining), see the `writing-timely-operators` skill.

## The Collection type

```rust
pub struct Collection<'scope, T: Timestamp, C: 'static> {
    pub inner: Stream<'scope, T, C>,
}
```

A `Collection` wraps a timely `Stream`.
The container type `C` is typically `Vec<(D, T, R)>` where `D` is the data, and `R` is the difference type (usually `isize`).
The shorthand `VecCollection<'scope, T, D, R>` refers to `Collection<'scope, T, Vec<(D, T, R)>>`.

`+1` means an insertion, `-1` a retraction.
An update `(d, t, +1)` followed by `(d, t, -1)` cancels out — `d` was never logically present at time `t`.

## High-level operators

High-level operators (`join`, `reduce`, `arrange_by_key`, etc.) are methods directly on `Collection` — the separate `Join`, `JoinCore`, `Reduce`, `ArrangeByKey`, `ArrangeBySelf` extension traits from earlier versions are removed.

All operators below are methods on `Collection`.

### Transforms

| Operator | Effect |
|---|---|
| `map(f)` | Apply `f` to each record |
| `map_in_place(f)` | Apply `f` in place, reusing allocations |
| `flat_map(f)` | One-to-many transform |
| `filter(f)` | Keep records matching predicate |
| `negate()` | Flip all diffs (requires `Negate` container support) |
| `concat(other)` | Union two collections |
| `consolidate()` | Sum diffs for identical `(data, time)` pairs, drop zeros |
| `consolidate_stream()` | Per-batch consolidation without exchange |
| `delay(f)` | Advance timestamps by the supplied function |
| `explode(f)` | Replace records with new data and difference type |

`consolidate()` exchanges data by key (hash), sorts, and merges.
Place it before expensive operators (reduce, join) to minimize redundant work.
Do not over-consolidate — each call is an exchange + sort.
`consolidate_stream()` avoids the exchange but only consolidates within each batch.

### Keyed operators

These operate on `Collection<'scope, T, Vec<((K, V), T, R)>>` — collections of key-value pairs.

**`reduce`**: Per-key aggregation.
```rust
collection.reduce(|key, input, output| {
    // key: &K
    // input: &[(V, R)] — values with their accumulated diffs
    // output: &mut Vec<(V2, R2)> — push results here
    output.push((aggregate(input), 1));
});
```

The closure receives consolidated input: values grouped by key with diffs summed.
It must be deterministic — DD calls it repeatedly to compute retractions when inputs change.

**`join`** / **`join_map`**: Equijoin on key.
```rust
left.join(&right)  // -> VecCollection<'scope, T, (K, (V1, V2)), R>
left.join_map(&right, |key, v1, v2| result)  // -> VecCollection<'scope, T, D, R>
```

Internally, `join` arranges both inputs by key, then calls `join_core`.
If one side is already arranged, use `join_core` directly to avoid redundant arrangement.

**`semijoin`** / **`antijoin`**: Filter by key presence/absence in another collection.
```rust
collection.semijoin(&keys)   // keep (k, v) where k is in keys
collection.antijoin(&keys)   // keep (k, v) where k is NOT in keys
```

**`distinct`**: Keep records with positive total multiplicity.
**`count`**: Count records per key.
**`threshold(f)`**: Transform multiplicity per key using a custom function.

### Inspection and debugging

```rust
collection.inspect(|&(ref data, time, diff)| { ... })
collection.inspect_batch(|time, data| { ... })
collection.inspect_container(|event| { ... })
collection.assert_empty()  // panics if non-empty
```

## Arrangements

An arrangement is a persistent, indexed representation of a collection.
Multiple operators can share the same arrangement, avoiding redundant storage and computation.

```rust
pub struct Arranged<'scope, Tr>
where
    Tr: TraceReader + Clone,
{
    pub stream: Stream<'scope, Tr::Time, Vec<Tr::Batch>>,
    pub trace: Tr,
}
```

The `stream` carries vectors of new batches as they arrive.
The `trace` is a shared handle (`TraceAgent`) to the accumulated, indexed history.

### Creating arrangements

```rust
// Arrange (K, V) pairs by key:
let arranged = collection.arrange_by_key();
let arranged = collection.arrange_by_key_named("MyArrangement");

// Arrange K values (no separate value):
let arranged = collection.arrange_by_self();
```

`arrange_by_key` uses `ValSpine` (keys + values).
`arrange_by_self` uses `KeySpine` (keys only, no value column).

Under the hood, `arrange_core` builds a timely operator that:
1. Receives `(data, time, diff)` triples.
2. Batches them using a `Batcher` until the frontier advances.
3. Seals completed batches and inserts them into the trace.
4. Outputs batches on the stream for downstream operators.

### Using arrangements

Operators that accept `Arranged` inputs skip the internal arrangement step:

```rust
let by_key = collection.arrange_by_key();
// Both use the same arrangement — no redundant work:
let joined = by_key.join_core(&other_arranged, |k, v1, v2| Some((k.clone(), (v1.clone(), v2.clone()))));
let reduced = by_key.reduce_abelian("sum", |_key, input, output, _updates| {
    output.push((input.iter().map(|(_, r)| r).sum::<isize>(), 1));
});
```

**Extracting data from arrangements:**

`as_container` converts batches from an arrangement back into a collection, useful when the batch container type is not `Vec`:
```rust
let collection = arranged.as_container(|batch| { /* extract containers from batch */ });
```

`flat_map_ref` applies a transformation to each `(key, val)` pair in the arrangement without cloning into an intermediate collection:
```rust
let mapped = arranged.flat_map_ref(|key, val| Some((key.to_owned(), val.to_owned())));
```

### Trace management

The `TraceReader` trait provides:
* `cursor()` — get a cursor over all accumulated data
* `cursor_through(upper)` — cursor up to a specific frontier
* `set_logical_compaction(frontier)` — allow the trace to merge updates at old timestamps
* `set_physical_compaction(frontier)` — allow physical merging of batches
* `map_batches(|batch| ...)` — iterate over stored batches

**Compaction is essential.** Without `set_logical_compaction`, the trace grows without bound.
DD operators advance compaction to match the input frontier automatically.
If you hold a `TraceAgent` manually, you must advance compaction yourself.

## Core operators

Each high-level operator has a lower-level variant that operates directly on `Arranged` inputs or forms arrangements.
The core operators are independent of the collection interface.
Use these when you already have an arrangement to avoid redundant re-arrangement.

### `reduce_trace`

```rust
pub fn reduce_trace<'scope, Tr1, Bu, Tr2, L, P>(
    trace: Arranged<'scope, Tr1>,
    name: &str,
    logic: L,
    push: P,
) -> Arranged<'scope, TraceAgent<Tr2>>
```

The logic closure signature:
```rust
FnMut(
    Tr1::Key<'_>,                       // key
    &[(Tr1::Val<'_>, Tr1::Diff)],       // consolidated input values + diffs
    &mut Vec<(Tr2::ValOwn, Tr2::Diff)>, // output buffer
    &mut Vec<(Tr2::ValOwn, Tr2::Diff)>, // update buffer (for retractions)
)
```

The `push` closure packs key + value updates into the builder input, decoupling the packing strategy from the reduce implementation:
```rust
P: FnMut(&mut Bu::Input, Tr1::Key<'_>, &mut Vec<(Tr2::ValOwn, Tr2::Time, Tr2::Diff)>)
```

Returns an `Arranged` — the output is itself an arrangement, usable as input to further core operators.

### `join_core`

Operates on two `Arranged` inputs.
Joins new batches from each side against the full trace of the other (delta join strategy).
The output diff is `R1 * R2` (requires `Multiply`).

For maximum control, `join_core_internal_unsafe` gives the closure direct cursor access for zero-copy processing.

### `arrange_core`

```rust
pub fn arrange_core<'scope, P, Ba, Bu, Tr>(
    stream: Stream<'scope, Tr::Time, Ba::Input>,
    pact: P,
    name: &str,
) -> Arranged<'scope, TraceAgent<Tr>>
```

Parameterized by batcher (`Ba`), builder (`Bu`), and trace (`Tr`) types.
The high-level `arrange_by_key` picks sensible defaults.

## Trace and cursor API

### Trace hierarchy

```
LayoutExt (GAT-based key/val/time/diff access)
  ├── TraceReader (extends with cursor, compaction, batches)
  │     └── Trace (extends with insert, close, exert)
  ├── BatchReader (extends with cursor, description)
  │     └── Batch (extends with Merger for progressive merging)
  └── Cursor (navigation over keys → values → (time, diff) triples)
```

All traits in this hierarchy extend `LayoutExt`, which provides GAT-based associated types:
`Key<'a>`, `Val<'a>`, `TimeGat<'a>`, `DiffGat<'a>` — borrowed views that enable zero-copy access to columnar storage.
Owned types are available as `KeyOwn`, `ValOwn`, `Time`, `Diff`.

### Cursor navigation

A cursor walks sorted `(key, val, time, diff)` entries hierarchically.
All accessors return GAT references (`Key<'a>`, `Val<'a>`), not owned values:

```rust
let (mut cursor, storage) = trace.cursor();
while cursor.key_valid(&storage) {
    let key = cursor.key(&storage);       // returns Key<'_> (borrowed)
    while cursor.val_valid(&storage) {
        let val = cursor.val(&storage);   // returns Val<'_> (borrowed)
        cursor.map_times(&storage, |time, diff| {
            // time: TimeGat<'_>, diff: DiffGat<'_>
        });
        cursor.step_val(&storage);
    }
    cursor.step_key(&storage);
}
```

Use `seek_key` / `seek_val` to skip to a specific position — this is the hot path for join implementations.
`rewind_keys` / `rewind_vals` reset the cursor position.

### Batch descriptions

Every batch has a `Description` with `lower` and `upper` antichains.
The batch contains updates at times `t` where `lower <= t` and `!(upper <= t)`.
This half-open interval enables seamless batch sequencing.

## Iteration

`iterate` enters a nested scope with `Product<OuterTime, u64>` timestamps, where the inner coordinate is the iteration counter.

```rust
let result = collection.iterate(|scope, inner| {
    // inner: Collection in the iterative scope
    // Return the collection for the next iteration.
    // Fixed point when no new updates are produced.
    inner
        .map(|x| step(x))
        .concat(&input.enter(scope))
        .distinct()
});
```

The `Variable` type manages the feedback loop.
`Variable::new` returns a `(Variable, Collection)` pair — the collection is the loop output.
`set()` binds the variable's definition.
Multiple variables enable mutual recursion.

Inside the iterative scope, the feedback edge has summary `Product::new(Default::default(), 1)` — each iteration increments the inner timestamp by 1.

**`enter` / `leave`**: `enter(scope)` wraps each timestamp in `Product<T, u64>` with inner = 0.
`leave(outer_scope)` strips the inner coordinate.
Both take the target scope by value (`Scope` is `Copy`).

## Difference algebra

DD's difference type `R` must satisfy algebraic properties depending on the operators used.
`IsZero` is a standalone trait; the others form a chain: `Semigroup` → `Monoid` → `Abelian`.
`Semigroup` accepts a generic `Rhs` parameter (defaulting to `Self`) for heterogeneous addition.

| Trait | Requires | Needed by |
|---|---|---|
| `IsZero` | `fn is_zero(&self) -> bool` | All operators (zero diffs are dropped) |
| `Semigroup<Rhs>` | `Clone + IsZero + plus_equals(&mut self, &Rhs)` | All operators (compaction) |
| `Monoid` | `Semigroup + fn zero() -> Self` | Most operators |
| `Abelian` | `Monoid + fn negate(&mut self)` | `negate`, `distinct`, retractions |
| `Multiply<Rhs>` | `fn multiply(self, &Rhs) -> Output` | `join` (diff = R1 * R2) |

The default `isize` satisfies all of these.
Custom difference types (e.g., lattice-valued) need only implement the traits required by the operators they appear in.

**`Present`**: A zero-sized difference type that represents "this record exists" without tracking multiplicity.
It implements `IsZero`, `Semigroup`, and `Multiply` but not `Abelian` (no negation).
Use `Present` when you only need set semantics (present/absent) rather than multiset semantics (counted), saving memory on the diff column.

## Delta joins (dogsdogsdogs)

The `dogsdogsdogs` crate (published as `differential-dogs3`) provides multi-way join patterns that avoid the quadratic blowup of nested binary joins.

### half_join

```rust
pub fn half_join<'scope, K, V, R, Tr, FF, CF, DOut, S>(
    stream: VecCollection<'scope, Tr::Time, (K, V, Tr::Time), R>,
    arrangement: Arranged<'scope, Tr>,
    frontier_func: FF,
    comparison: CF,
    output_func: S,
) -> VecCollection<'scope, Tr::Time, (DOut, Tr::Time), <R as Mul<Tr::Diff>>::Output>
```

A half join matches updates from one stream against an arrangement where the arranged data's timestamp satisfies a comparison function under a total order on time.
Multiple `half_join` operators compose into a full delta join: each input stream is independently joined against all other arrangements, with the ordering ensuring each output tuple is produced exactly once.

### Count-propose-validate pattern

The `PrefixExtender` trait implements a three-phase multi-way join:
1. **Count**: Estimate extension candidates per prefix (for join ordering).
2. **Propose**: Generate candidate extensions from the cheapest index.
3. **Validate**: Filter proposals against remaining indices.

This enables adaptive query execution — the runtime picks which relation to probe based on cardinality estimates.

Key types:
* `CollectionIndex<K, V, T, R>` — maintains three internal traces (count, propose, validate)
* `CollectionExtender` — concrete `PrefixExtender` implementation combining an index with a key-selection function

## Columnar data representation

DD can use columnar containers (from the `columnar` crate v0.12) as backing storage for trace batches.
The `ColumnarLayout` type maps each column (keys, vals, times, diffs) to columnar storage, and `Coltainer<C>` wraps a columnar container to implement DD's `BatchContainer` trait.

For the full columnar API (traits, derive macro, container types, serialization), see the `using-columnar` skill.
