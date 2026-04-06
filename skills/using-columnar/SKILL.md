---
name: using-columnar
description: >
  Guide for working with the columnar crate for struct-of-arrays data layout.
  Use this skill when the user mentions "columnar", "Columnar trait",
  "#[derive(Columnar)]", "struct of arrays", "ContainerOf", "Strings", "Vecs",
  "RankSelect", "Lookbacks", "Repeats", "AsBytes", "FromBytes", "columnar
  container", or when working with columnar data types in timely or differential
  dataflow contexts.
---

# Using the columnar crate

This skill targets **columnar v0.12**.
API details may differ in other versions — check `frankmcsherry/columnar` source when in doubt.

Columnar transforms arrays of complex structs into structs of simple arrays (AoS → SoA).
The result is fewer heap allocations, better cache locality, and zero-copy serialization to bytes.
The crate supports `no_std` environments (with `alloc`).

## Core concept

A `Vec<(String, u64, Vec<u8>)>` stores each tuple on the heap with separate allocations for the string and inner vec.
Columnar repacks this into three contiguous buffers: one for all strings (as concatenated bytes + offsets), one for all u64s, and one for all inner vecs (as concatenated bytes + offsets).
The number of allocations drops from O(n) to a fixed constant regardless of collection size.

The tradeoff: element access returns `(&str, &u64, Slice<&[u8]>)` instead of `&(String, u64, Vec<u8>)`.
You get references to each field separately, not a reference to the original struct.

## The Columnar trait

```rust
pub trait Columnar: 'static {
    type Container: ContainerBytes + for<'a> Push<&'a Self>;

    fn copy_from<'a>(&mut self, other: Ref<'a, Self>);
    fn into_owned<'a>(other: Ref<'a, Self>) -> Self;
    fn as_columns<'a, I>(selves: I) -> Self::Container
    where I: IntoIterator<Item = &'a Self>;
    fn into_columns<I>(selves: I) -> Self::Container
    where I: IntoIterator<Item = Self>;
    fn reborrow<'b, 'a: 'b>(thing: Ref<'a, Self>) -> Ref<'b, Self>;
}
```

Key type aliases:
* `ContainerOf<T>` = `<T as Columnar>::Container` — the columnar storage type for `T`
* `BorrowedOf<'a, T>` = `<ContainerOf<T> as Borrow>::Borrowed<'a>` — the lightweight borrowed view type
* `Ref<'a, T>` = `<ContainerOf<T> as Borrow>::Ref<'a>` — the reference type when indexing into a container

`as_columns` converts `&[T]` into a columnar container.
`into_owned` reconstructs an owned `T` from a columnar reference.

## Derive macro

```rust
use columnar::Columnar;

#[derive(Columnar)]
struct MyRecord {
    key: String,
    value: u64,
    tags: Vec<String>,
}
```

`#[derive(Columnar)]` generates the container type, `Borrow`, `Push`, `Index`, and all supporting trait implementations.
Each field gets its own column: keys in a `Strings`, values in a `Vec<u64>`, tags in a `Vecs<Strings>`.

The derive works for structs with named fields.
For enums, see the sum types section below.

## Container traits

Containers form a layered trait hierarchy:

```
Len                  — element count
Clear                — reset without dealloc
Push<T>              — accept items
Index                — read access: get(i) -> Ref
IndexMut             — mutable access
Borrow               — lifetime-aware reference conversion
Container            — storage: capacity, reserve, extend_from_self
ContainerBytes       — serialization: AsBytes + FromBytes
```

### Push and Index

```rust
pub trait Push<T> {
    fn push(&mut self, item: T);
    fn extend(&mut self, iter: impl IntoIterator<Item = T>);
}

pub trait Index {
    type Ref;
    fn get(&self, index: usize) -> Self::Ref;
    fn index_iter(&self) -> IterOwn<&Self>;
}
```

A container implements `Push<&'a T>` (push a reference to an owned value) and `Push<Ref<'a, T>>` (push a columnar reference).
`Index::Ref` is a lightweight `Copy` type — for primitives it is the value itself, for strings it is `&str`, for structs it is a tuple of field references.

### Borrow

```rust
pub trait Borrow: Len + Clone + 'static {
    type Ref<'a>: Copy;
    type Borrowed<'a>: Copy + Len + Index<Ref = Self::Ref<'a>>;

    fn borrow<'a>(&'a self) -> Self::Borrowed<'a>;
}
```

`Borrowed<'a>` is a lightweight view over the container (like a slice).
`Ref<'a>` is what you get when indexing into a `Borrowed`.
Both are `Copy` — no allocation on access.

### Container

```rust
pub trait Container: Borrow + Clear + Default + Send
    + for<'a> Push<Self::Ref<'a>>
{
    fn with_capacity_for<'a, I>(selves: I) -> Self
    where I: Iterator<Item = Self::Borrowed<'a>> + Clone;
    fn reserve_for<'a, I>(&mut self, selves: I)
    where I: Iterator<Item = Self::Borrowed<'a>> + Clone;
    fn extend_from_self(&mut self, other: Self::Borrowed<'_>, range: Range<usize>);
}
```

`extend_from_self` copies a range of elements from a borrowed view — the primary mechanism for slicing and filtering.

## Built-in container types

### Primitives

Most primitives use `Vec<T>` directly as their container: `u8`, `u16`, `u32`, `u64`, `i8`, `i16`, `i32`, `i64`, `f32`, `f64`.

Special containers for types that need conversion:

| Type | Container | Storage |
|---|---|---|
| `usize` / `isize` | `Usizes` / `Isizes` | Stored as `u64` / `i64` |
| `u128` / `i128` | `U128s` / `I128s` | Stored as `[u8; 16]` |
| `char` | `Chars` | Stored as `u32` |
| `bool` | `Bools` | Bit-packed in `u64` words |
| `()` | `Empties` | Count only, zero data overhead |
| `Duration` | `Durations` | Pair of seconds + nanoseconds |

### Strings

```rust
pub struct Strings<BC = Vec<u64>, VC = Vec<u8>> {
    pub bounds: BC,  // offset boundaries
    pub values: VC,  // concatenated UTF-8 bytes
}
```

All strings are concatenated into a single byte buffer.
`bounds` stores cumulative byte offsets for O(1) access to each string.
`Ref` is `&[u8]` (raw bytes, not `&str`) to avoid UTF-8 validation on the critical read path.
Use the convenience method `get_str(index)` when you need a validated `&str`.

### Vecs (nested lists)

```rust
pub struct Vecs<TC, BC = Vec<u64>> {
    pub bounds: BC,   // offset boundaries
    pub values: TC,   // flattened inner elements
}
```

`Vec<T>` becomes `Vecs<T::Container>`.
Inner elements are flattened into a single container; bounds track where each inner vec starts and ends.
`Ref` is `Slice<Borrowed<'a>>` — a view into the flattened container.
Nesting works recursively: `Vec<Vec<String>>` becomes `Vecs<Vecs<Strings>>`.

### Tuples

Tuples use a tuple of containers: `(A, B, C)` stores as `(A::Container, B::Container, C::Container)`.
Each field is an independent column.
`Ref` is a tuple of field references: `(Ref<'a, A>, Ref<'a, B>, Ref<'a, C>)`.

### Sum types (Result, Option)

```rust
pub struct Results<SC, TC> {
    pub indexes: RankSelect,  // bit vector: true = Ok, false = Err
    pub oks: SC,              // Ok values only
    pub errs: TC,             // Err values only
}

pub struct Options<TC> {
    pub indexes: RankSelect,  // bit vector: true = Some, false = None
    pub somes: TC,            // Some values only
}
```

`RankSelect` is a bit vector with O(1) rank queries (count of set bits up to a position).
When accessing element `i`, the container checks the discriminant bit and uses `rank(i)` to compute the offset into the appropriate variant's sub-container.

Custom enums use the derive macro, which generates a `Discriminant`-based layout with per-variant containers.
When all elements share the same variant (homogeneous), the `Discriminant` avoids storing per-element tags and offsets — only the variant tag and count are stored.
`Discriminant::is_heterogeneous()` returns `true` when elements have mixed variants and per-element metadata is present.

## Compression containers

### Repeats

```rust
pub struct Repeats<TC> { /* wraps Options<TC> */ }
```

Run-length encodes consecutive duplicates.
Stores `Some(value)` for new values and `None` when the previous value repeats.
Useful when data is sorted or clustered by a field.

### Lookbacks

```rust
pub struct Lookbacks<TC, const N: usize = 255> { /* wraps Results<TC, Vec<u8>> */ }
```

Dictionary compression within a sliding window.
Stores `Ok(value)` for new values and `Err(offset)` for values that match one up to `N` positions back.
More flexible than `Repeats` — catches non-consecutive duplicates within the window.

## Byte serialization

Containers implement `AsBytes` and `FromBytes` for zero-copy-friendly serialization.

```rust
pub trait AsBytes<'a> {
    fn as_bytes(&self) -> impl Iterator<Item = (u64, &'a [u8])>;
}

pub trait FromBytes<'a> {
    fn from_bytes(bytes: &mut impl Iterator<Item = &'a [u8]>) -> Self;
}
```

Each `(u64, &[u8])` pair is an alignment hint and a byte slice.
The wire format (`Indexed`) stores:
1. An offset table (u64 per slice) for O(1) random access.
2. Data payload with each slice padded to 8-byte alignment.

Primitive columns serialize as direct byte casts (`bytemuck`).
Composite containers (tuples, vecs, strings) recursively serialize each sub-container.

**`DecodedStore`**: A zero-allocation random-access view into indexed-encoded data.
It provides constant-time field access regardless of tuple width, replacing the legacy `from_u64s` / `decode_u64s` decoding methods.
Derived types support a `from_store` constructor for structured decoding.

**`Stash`**: A container that accumulates byte-serialized data for deferred decoding.
Use `Stash::try_from_bytes` to validate and construct from raw bytes.

## Slice and iteration

`Slice<S>` is a range-bounded view into an `Index`-implementing type:

```rust
pub struct Slice<S> {
    pub lower: usize,
    pub upper: usize,
    pub slice: S,
}
```

`Slice` implements `Index`, `Len`, and produces an `IterOwn` iterator.
This is the `Ref` type for `Vecs` — when you index a `Vecs<Strings>`, you get a `Slice<Borrowed<Strings>>`.

`IterOwn<S>` iterates an `Index + Len` type by calling `get(i)` for each position.
It implements `ExactSizeIterator`.

## Integration with timely dataflow

Columnar containers do not implement timely's container traits directly.
A `Column<C>` wrapper bridges the two worlds, handling typed storage, byte deserialization, and alignment.

### Column wrapper

`Column<C>` is an enum with three variants:

```rust
pub enum Column<C> {
    Typed(C),           // owned columnar container
    Bytes(arc::Bytes),  // zero-copy from network, u64-aligned
    Align(Arc<[u64]>),  // relocated copy when Bytes is misaligned
}
```

All three variants support `.borrow()` → `C::Borrowed<'_>`, so operator code does not need to distinguish them.
The `Bytes` variant avoids deserialization entirely — data arrives from the network as aligned bytes and is read in place.

`Column<C>` implements timely's traits by delegating to the inner container:

| Timely trait | Column implementation |
|---|---|
| `PushInto<T>` | Delegates to `C::push()` (only works on `Typed` variant) |
| `ContainerBytes` | `from_bytes` creates `Bytes` or `Align`; `into_bytes` serializes via `Indexed::write` |
| `SizableContainer` | `at_capacity` checks if serialized size exceeds ~1 MB |
| `DrainContainer` | Returns `IterOwn` over the borrowed contents |
| `Accountable` | `record_count` returns `borrow().len()` |

### ColumnBuilder

`ColumnBuilder<C>` accumulates items into a typed columnar container and flushes completed batches as `Column<C>`:

```rust
pub struct ColumnBuilder<C> {
    current: C,
    empty: Option<Column<C>>,
    pending: VecDeque<Column<C>>,
}
```

On each `push_into`, it checks the serialized word count.
When close to a 2 MB boundary (within 10% slop), it encodes the container into aligned bytes and enqueues it.
This batching amortizes the encoding cost.

`ColumnBuilder` implements timely's `ContainerBuilder` + `LengthPreservingContainerBuilder`.

### Timely example: columnar wordcount

Define a columnar record type using the derive macro, then use `Column` and `ColumnBuilder` in the dataflow:

```rust
#[derive(columnar::Columnar)]
struct WordCount {
    text: String,
    diff: i64,
}

type InnerContainer = <WordCount as columnar::Columnar>::Container;
type Container = Column<InnerContainer>;

// Input handle uses ColumnBuilder to batch into Column containers:
let mut input = <InputHandle<_, CapacityContainerBuilder<Container>>>::new();

// Operators receive Column<InnerContainer> and borrow it for access:
.unary(Pipeline, "Split", |_cap, _info| {
    move |input, output| {
        input.for_each_time(|time, data| {
            let mut session = output.session(&time);
            for container in data {
                // .borrow() works on any Column variant (Typed, Bytes, or Align):
                for wc in container.borrow().into_index_iter() {
                    // wc.text is &[u8], wc.diff is &i64 — columnar references
                    session.give(WordCountReference { text: wc.text, diff: wc.diff });
                }
            }
        });
    }
})
```

Exchange uses `ExchangeCore` with a `ColumnBuilder` to re-encode after routing:

```rust
.unary_frontier(
    ExchangeCore::<ColumnBuilder<InnerContainer>, _>::new_core(
        |x: &WordCountReference<&[u8], &i64>| x.text.len() as u64
    ),
    "WordCount",
    |_cap, _info| { /* frontier-driven operator logic */ }
)
```

The derive macro generates `WordCountReference<T, D>` — a struct with borrowed field types used as the columnar `Ref` type.
Inside operators, `container.borrow().into_index_iter()` yields `WordCountReference<&[u8], &i64>` values.
Use `get_str(index)` on the underlying `Strings` container when you need validated `&str` instead of raw `&[u8]`.

### Sending input

Input handles accept columnar references directly:

```rust
input.send(WordCountReference { text: "hello world", diff: 1 });
```

The container builder batches these into columnar form automatically.

## Integration with differential dataflow

DD uses columnar containers at a deeper level: as the backing storage for trace batches (arrangements).

### ColumnarLayout

The `ColumnarLayout` type implements DD's `Layout` trait, mapping each column (keys, vals, times, diffs) to a `Coltainer<T>`:

```rust
pub struct ColumnarLayout<U: ColumnarUpdate> { ... }

impl<U: ColumnarUpdate> Layout for ColumnarLayout<U> {
    type KeyContainer  = Coltainer<U::Key>;
    type ValContainer  = Coltainer<U::Val>;
    type TimeContainer = Coltainer<U::Time>;
    type DiffContainer = Coltainer<U::Diff>;
    type OffsetContainer = OffsetList;
}
```

### Coltainer

`Coltainer<C>` wraps a `C::Container` and implements DD's `BatchContainer` trait:

```rust
pub struct Coltainer<C: Columnar> {
    pub container: C::Container,
}

impl<C: Columnar + Ord + Clone> BatchContainer for Coltainer<C>
where for<'a> columnar::Ref<'a, C>: Ord
{
    type ReadItem<'a> = columnar::Ref<'a, C>;
    type Owned = C;
    // push_ref, push_own, into_owned, clone_onto, clear, ...
}
```

The `Ord` bound on `Ref<'a, C>` is required because DD's trace internals use sorted access.

### Trie-shaped batch storage

DD's columnar example defines `KeyStorage` and `ValStorage` types that store batches as a trie with columnar columns:

```rust
pub struct KeyStorage<U: Update> {
    pub keys: Column<ContainerOf<U::Key>>,
    pub upds: Column<Vecs<(ContainerOf<U::Time>, ContainerOf<U::Diff>)>>,
}

pub struct ValStorage<U: Update> {
    pub keys: ContainerOf<U::Key>,
    pub vals: Vecs<ContainerOf<U::Val>>,
    pub upds: Vecs<(ContainerOf<U::Time>, ContainerOf<U::Diff>)>,
}
```

Keys are stored once; for each key, `vals` holds a nested list of values; for each value, `upds` holds `(time, diff)` pairs.
The `Vecs` bounds arrays track where each key's values start and end.

`form()` constructs the trie from sorted tuples by detecting key/value transitions:

```rust
let storage = KeyStorage::form(sorted_refs.into_iter());
```

### Custom exchange (KeyPact)

Standard `Exchange` hashes full records.
With columnar storage, a custom `KeyPact` + `KeyDistributor` routes by key without round-tripping through tuples:

```rust
let pact = KeyPact { hashfunc: |k: columnar::Ref<'_, Vec<u8>>| k.hashed() };
let arranged = arrange_core::<_, _, KeyBatcher<_,_,_>, KeyBuilder<_,_,_>, KeySpine<_,_,_>>(
    stream, pact, "Data"
);
```

The distributor copies keys and their associated updates directly between columnar containers using `extend_from_self`, avoiding deserialization.

### Type aliases for columnar DD

```rust
type ValSpine<K, V, T, R> = Spine<Rc<OrdValBatch<ColumnarLayout<(K,V,T,R)>>>>;
type KeySpine<K, T, R>    = Spine<Rc<OrdKeyBatch<ColumnarLayout<(K,T,R)>>>>;
```

These substitute directly where DD normally uses `ValSpine` / `KeySpine` with the default layout.

## Abstract data types

The `adts` module provides columnar representations of recursive data structures:
* `tree` — columnar trees
* `art` — adaptive radix trees

These require special handling because recursive types cannot use the standard derive macro.
