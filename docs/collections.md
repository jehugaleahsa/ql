# Collections
The default collection type is a vector. This is a sequence of values of the same type. For example, `[1, 2, 3]` is a vector of three `i32` values.

Vectors can be mutable or immutable, depending on how they are declared:
```
let x = [1, 2, 3];     # immutable
let mut y = [1, 2, 3]; # mutable
```

Making a vector mutable will allow operations such as `into` (a.k.a., `insert`), `update`, and `delete` to be performed, as explained starting [here](./in-memory-sources.md#appending-values).

## Vectorized operations
When adding, subtracting, etc., operations are vectorized. Consider this example:
```
let evens = [0, 2, 4, 6, 8];
let odds = [1, 3, 5, 7, 9];
let sum = evens + odds; # [1, 5, 9, 13, 17]
```

Here, you can see addition is performed element-wise over the two collections.

## Scalars, vectors, matrices, and arrays
Scalar values always have a "dimension" of 0, and a normal vector has a dimension of 1. Consider these examples:
```
let x = 1;               # 0-dimensional scalar
let y = [1, 2];          # 1-dimensional vector
let z: i32[2, 2] = [     # 2-dimensional matrix
    [1, 2], 
    [3, 4]
];
let a: i32[2, 2, 2] = [  # 3-dimensional array
    [
        [1, 2], 
        [3, 4]
    ], 
    [
        [5, 6], 
        [7, 8]
    ]
];
```

Matrices and arrays require special syntax to initialize. Notice after the variable name, we specified `: i32[2, 2]` and `: i32[2, 2, 2]`. By listing the dimensions explicitly as comma-separated values, we opt-in to using matrices and arrays. If we just did:
```
let z = [[1, 2], [3, 4]];
```
we would be creating a vector of vectors, where the length of each child vector could be a different length. In other words, only by restricting the length of each child vector do we opt into a matrix or array. Overall, a matrix or array can be thought of as a vector of vectors but with additional functionality and restrictions.

## Dimensions and broadcasting
With vectors, matrices, and arrays, we can perform arithmetic operations on them using scalars. For example:
```
[1, 2, 3] * 2 # [2, 4, 6]
```

This is referred to as "broadcasting". Similarly, we can multiply a vector times a matrix like so:
```
let v = [1, 2];
let m: i32[2, 2] = [
    [1, 2],
    [3, 4]
];
m * v # [[1, 4], [3, 8]]
```

> If the dimensions are not compatible, the operation results in a panic, resulting in the program terminating!

You can also multiply a matrix by another matrix; however there are two options. When using `*`, the two matrices must have identical dimensions
and the elements are multiplied element-wise. When using `@`, the dimensions only need to be compatible and normal matrix multiplication is performed. For example: 
```
let a: i32[2, 2] = [
    [1, 2],
    [3, 4]
];
a * a # [[1, 4], [9, 16]]

let m: i32[2, 3] = [
    [1, 2, 3],
    [4, 5, 6]
];
let n: i32[3, 2] = [
    [1, 2],
    [3, 4],
    [5, 6]
];
m @ n # [[22, 49], [28, 64]]
```

> When multiplying matrices using either operator, the dimensions must be compatible or the operation results in a panic.

The result of broadcasting can be surprising when working with higher dimensions, so merit some experimentation.

## Null
The literal `null` represents no value.

> **NOTE:** Unlike SQL, `null == null` returns `true` in QL.

When declaring a variable (with `let`), the type can be specifed explicitly:
```
let x: i32 = 123;
```

If `x` in the example above can also be `null`, it should be declared like this:
```
let x: i32? = getValueOrNull();
```

Under the hood, the type is actually `Optional<i32>`, where `null` is the special marker for when no value is present, so `T?` is just a syntactic sugar. 

### Lifted arithmetic operations
Performing arithmetic operations on `null` results in `null`.
```
1 + null + 2 # null
```

If a `null` appears anywhere within an arithmetic operation, the whole expression becomes `null`.

Some conversion operations will return `null` if the conversion fails:
```
let bad = "12abc";      # this won't parse
i32::tryParse(bad) + 23 # null
```

Many functions are designed to return `null` when passed `null` values. Arithmetic functions are just another form of function.

## Common collection traits

* `Iterable<T>` - Indicates that the elements of the collection can be iterated over.
    * `iterator()` - Gets an iterator that returns `Optional<T>`, returning `null` when exhausted. The underlying collection controls the order the items are returned.
    * `count()` - Gets the number of elements in the collection. The underlying collection can optimize the implementation.
    * `where()` - Returns an iterable only containing the items matching a predicate
    * `select()` - Return an iterable containing values mapped from the source collection using a mapping function.
    * `flatten()` - Returns an iterable containing the flattened collections of the mapping operation.
    * etc.
* `Indexed<K, V>` - Allows a collection to be indexed. For example, vectors are indexed by 0-based position and maps are indexed by their keys. 
* `Appendable<T>: Iterable<T>` - Allows a collection to be on the receiving end of an `into` (a.k.a. `insert`) operation.
    * `add(T)` - adds an item to the collection
* `Updatable<T>: Iterable<T>` - Allows a collection be updated in-place using an `update` operation.
    * `update(Iterator, V)` - replace the value at the given iterator location with the given value. 
* `Deletable<T>: Iterable<T>` - Allows removing elements in-place from a collection.
    * `delete(Iterator)` - remove the value at the given iterator location.
* `Splicable<K, V>: Indexable<K, V>, Iterable<V>` - Allows removing and adding multiple elements in-place at a particular index.

## Vector
The trait for a vector looks something like this:
```
let Vector<T> = type Appendable<T>, Updatable<T>, Deletable<T>, Splicable<usize, T> {
    # ...implementation details
};
```

Therefore, the following two statements are identical:
```
let x: i32[] = [1, 2, 3];
# or...
let x: Vector<i32> = [1, 2, 3];
```

## Matrix and Array
The underlying trait for matrix and array are the same, with compile-time dimensions:
```
let Array2d<T, N, M> = type Updatable<T>, Splicable<usize, T> {
    # ...implementation details
};
```

A separate `Array` class exists for each additional dimension. Note that all the elements in a matrix/array must be the same type.

Therefore, the following two statements are identifical:
```
let m: i32[2, 2] = [[1, 2], [3, 4]];
# or...
let m: Array2d<i32, 2, 2> = [[1, 2], [3, 4]];
```

## Tuple
A tuple is a fixed-size list of values. Rather than indexing into a tuple by index, each value is accessed with a special property. For example:
```
let t = (1, 2, 3);
let first = t._0;
let second = t._1;
let third = t._2;
```

Unlike tuples in many other languages, tuples provide 0-based properties. Each property is the 0-based index, preceeded by an underscore (_). The example above can more succinctly be written as:
```
let (first, second, third) = (1, 2, 3);
```

Values within a tuple can be ignored using the `_` placeholder:
```
let (_, second, ..._) = (1, 2, 3);
```

This says to ignore the first value, bind the second value to `second`, and ignore any values that follow.

## Optional
You can think of `Optional` as a special type of vector where there is either zero or one elements. `Optional` is used to wrap scalar values when `null` is possible. When `null`, the `Optional` is empty; otherwise, there is exactly one value - the scalar. You rarely need to work directly with an `Optional` because of the many syntactic short-cuts. For example, simply declaring a type with a trailing `?` makes it nullable, and therefore becomes `Optional`. Similarly, the `??` operator allows replacing `null` with another value, and `?.` allows expanding properties on a potentially `null` scalar.

Where `Optional` becomes useful is when treating it as a collection. `Optional` provides an `asIterable()` method to provide access to the many collection operations and/or treat it as a source within a query.

## Set
A set is similar to a vector except each value must be unique and the order the elements appear may not be guaranteed.
```
let Set<T> = type Appendable<T>, Updatable<T>, Deletable<T> {
    # ...implementation details
};
```

Sets must be constructed using a factory method, like so:
```
let s: Set<i32> = Set.of(1, 2, 3);
```

If you have an existing collection, you can use `from` instead:
```
let v = [1, 2, 3];
let s = Set.from(v);
```

Sets can be ordered or unordered, meaning the order that the items are added is retained or not, respectively. To create an ordered set, use a builder instead:
```
let s: Set<i32> = Set.ordered().of(1, 2, 3);
```

If equality is not defined for a type, a comparator can be provided:
```
let comparator = Hash<Customer>.using(c => c.id);
let s: Set<i32> = Set.ordered(comparator).of(c1, c2, c3);
```

## Map
Maps define mappings from a unique key to a value. The entries in a Map can be ordered or unordered, similar to Sets.
```
let Map<K, V> = type Indexed<K, V>, Appendable<(K, V)>, Updatable<(K, V)>, Deletable<(K, V)> {
    # ...implementation details
};
```

Each entry in a Map is a key/value tuple (or pair). Maps can be inserted into, updated, or deleted using the `into`, `update`, and `delete` operations so long as those operations operate over tuples. Map exposes a `keySet()` method, as well, whose type looks like this:
```
let KeySet<K> = type Deletable<K> {
    # ...implementation details
};
```

A `KeySet` is just a view over the underlying `Map`'s entries, so removing from the key set also removes from the `Map`.

Here is an example of some common Map operations:
```
let mut customerLookup = Map.from(
    from customers as c
    select (c.id, c) # Select a tuple of key/value pairs
);

# Find the customer with an ID of 123 - could be null
let customer123 = customerLookup[123];

from customerLookup as l
where customer123 != null
where l.id == customer123.id
delete l;
```