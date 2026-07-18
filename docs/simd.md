# SIMD

> **NOTE:** This is a design direction under consideration, not settled syntax.

QL aims to let ordinary array code compile to SIMD instructions without writing any assembly or intrinsics. The approach is to reshape a collection into fixed-size chunks and lean on the existing [broadcasting](./collections.md#dimensions-and-broadcasting) rules: an element-wise operation over a fixed-size array is exactly the shape a backend can lower to a vector instruction.

## Chunking
The `chunks` function reshapes a one-dimensional collection into fixed-size arrays. Because a collection's length may not divide evenly, it returns two things: the full chunks, and whatever is left over.
```
let v: f64[] = ...;
let (blocks, tail) = v.chunks<8>();  # blocks : f64[8][],  tail : f64[]
```

Each full chunk has type `f64[8]`, a compile-time size (a [constant generic](./generics.md)). That fixed size is the whole point: because the length is known at compile time, arithmetic over a chunk broadcasts across a known number of lanes, which a backend can emit as a single vector instruction. The `tail` holds the leftover elements - empty when the length divides evenly - and is an ordinary `f64[]`, since its length is not known until run time.
```
let doubled =
    from blocks as chunk   # chunk : f64[8]
    select chunk * 2.0      # one vector multiply per chunk
    ...;
```

Chunking, then, is just a function call that splits a collection into a statically-sized bulk and a dynamically-sized remainder.

## Handling the tail
The `tail` needs its own treatment, since it is not a full-width chunk. There are two idioms:

* **Scalar epilogue.** Process `blocks` with vector code and handle `tail` with an ordinary scalar loop. (This is what Rust's `chunks_exact` and its `remainder` do.)
* **Padding.** Extend the tail up to full width with a fill value, so it can run through the same vector code as the full chunks. Padding needs no special machinery - it is `concat` with a lazy repeat, where the "infinite generator" is just an infinite [range](./ranges.md#ranges-as-sequences) mapped to a constant:

```
let repeat = fn(x) { from 0.. select x };   # an infinite, lazy source of x
let filled = [...tail, ...repeat(0.0)];     # the tail followed by endless zeros
```

Taking the first `8` of `filled` yields a full-width chunk. For a reduction, pad with the [aggregator's](./aggregates.md#the-aggregate-interface) identity - `0` for `Sum`, `1` for `Product` - so the padding cannot affect the result.

> **NOTE:** This is the second reason `identity` earns its place on the aggregate interface: beyond enabling parallel reduction, it makes the ragged tail of a vectorized loop safe to pad. And because padding is only `concat` plus a lazy `repeat`, it is the same tool that reconciles any ragged edge - including an unequal-length [`zip`](./combining-collections.md#zip).

## Overlapping windows
`chunks<N>` produces *non-overlapping* tiles. A related reshaper, `windows<N>`, produces *overlapping* fixed-size windows - the current element together with its neighbors - which suits fixed-width moving computations that should skip, rather than shrink at, the edges. Overlapping windows vectorize less cleanly than tiles because of the repeated loads, but the fixed size still removes the edge-nullability that the variable [window frames](./queries.md#windows) carry.

## Combining inputs
Element-wise operations over two chunk streams are multi-input SIMD. [Zipping](./combining-collections.md#zip) two chunked collections and operating lane-wise is how something like a dot product vectorizes without intrinsics:
```
let dot =
    from a.chunks<8>() as ca      # f64[8]
    zip b.chunks<8>() as cb        # f64[8]
    select (ca * cb).sum()         # lane-wise multiply, then reduce the eight lanes
    ... .sum();                    # sum the per-chunk partials (tail padded with 0)
```

Nothing here is a SIMD primitive. It is chunking (a reshape), broadcasting (element-wise arithmetic over fixed-size arrays), and reduction (an [aggregator](./aggregates.md)) - each of which exists for its own reasons - combined so that the backend has enough static information to emit vector code.
