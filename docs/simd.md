# SIMD

> **NOTE:** This is a design direction under consideration, not settled syntax.

QL aims to let ordinary array code compile to SIMD instructions without writing any assembly or intrinsics. The approach is to reshape a collection into fixed-size chunks and lean on the existing [broadcasting](./collections.md#dimensions-and-broadcasting) rules: an element-wise operation over a fixed-size array is exactly the shape a backend can lower to a vector instruction.

## Chunking
The `chunks` function reshapes a one-dimensional collection into a sequence of fixed-size arrays:
```
let v: f64[] = ...;
let blocks = v.chunks<8>();  # f64[8][] - a sequence of length-8 arrays
```

The chunk size is a compile-time constant (a [constant generic](./generics.md)), so each chunk has type `f64[8]`. That fixed size is the whole point: because the length is known at compile time, arithmetic over a chunk broadcasts across a known number of lanes, which a backend can emit as a single vector instruction.
```
let scaled =
    from v.chunks<8>() as chunk   # chunk : f64[8]
    select chunk * 2.0            # one vector multiply per chunk
    ...;
```

Chunking is, in the end, just a function call. The only thing that needs special attention is the last chunk.

## The last chunk
When a collection's length is not a multiple of the chunk size, the final chunk is undersized. There are two strategies:

* **Scalar epilogue.** Process the whole chunks with vector code and handle the leftover elements with an ordinary scalar loop. (This is what Rust's `chunks_exact` and its `remainder` do.)
* **Identity padding.** Pad the final chunk up to full size with an [aggregator's](./aggregates.md) identity element - `0` for `Sum`, `1` for `Product` - so the partial chunk runs through the same vector code without affecting the result.

Identity padding is why the `identity` on the [Aggregate interface](./aggregates.md#the-aggregate-interface) matters beyond parallel reduction: it makes the ragged tail of a vectorized loop safe.

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
