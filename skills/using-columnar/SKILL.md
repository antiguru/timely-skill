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

This skill targets **columnar v0.11**.
API details may differ in other versions — check `frankmcsherry/columnar` source when in doubt.

Columnar transforms arrays of complex structs into structs of simple arrays (AoS → SoA).
The result is fewer heap allocations, better cache locality, and zero-copy serialization to bytes.

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
`Ref` is `&str`.

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

Custom enums use the derive macro, which generates a similar discriminant + per-variant container layout.

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

## HeapSize

```rust
pub trait HeapSize {
    fn heap_size(&self) -> (usize, usize);  // (active_bytes, allocated_bytes)
}
```

Reports memory usage for profiling.
`active_bytes` is the bytes actually used; `allocated_bytes` includes spare capacity.

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

## Integration with timely and differential dataflow

Columnar containers serve as an alternative to `Vec<T>` in timely streams and differential collections.
The `Collection<G, C>` type in differential dataflow is generic over the container `C`.
When `C` is a columnar container, records are stored in struct-of-arrays layout during communication and batching.

To use columnar containers in a dataflow, the container must implement timely's `Container` trait (separate from columnar's `Container` trait) which requires `Len`, `Clear`, `Default`, and capacity management.

## Abstract data types

The `adts` module provides columnar representations of recursive data structures:
* `tree` — columnar trees
* `art` — adaptive radix trees

These require special handling because recursive types cannot use the standard derive macro.
