# Vectors
In QL, there are no scalar values. In other words, literals such as `1`, `true`, and `3.14` are actually just vectors (or collections) with one element, and could also be written as `[1]`, `[true]`, and `[3.14]`.

When adding, subtracting, etc., these operations are vectorized. Consider this example:
```
let evens = [0, 2, 4, 6, 8];
let odds = [1, 3, 5, 7, 9];
let sum = evens + odds; # [1, 5, 9, 13, 17]
```

Here, you can see addition is performed element-wise over the two collections.

> **NOTE:** If performing arithmetic on vectors of different lengths, the resulting vector is only as long as the shorter vector. There are alternatives to achieving different results, such as padding out the vectors.

## Dimensions and broadcasting
The literal `1` and `[1]` equate to the same value.

For example, the `append` operation concatenates two vectors (or collections), such that `1 append 2` results in the vector `[1, 2]`. Similarly, `[1] append [2]` results in `[1, 2]`.

However, the behavior changes when working with higher-dimension vectors. For example:
```
[[1, 2], [3, 4]] append [[5, 6]] # [[1, 2], [3, 4], [5, 6]]
```

> **NOTE:** To avoid confusion, it often makes sense to wrap literals in `[]` when planning on performing vector operations on them. Therefore, `[1] append [2]` is to be preferred over `1 append 2` even though it is more typing.

Arithmetic operations can be applied to vectors with different dimensions. For example:
```
[[1, 2], [3, 4]] * 3 # [[3, 6], [9, 12]]
```

The result of these operations can be surprising when working with higher dimensions, so merit some experimentation. The compiler knows the dimensions of vectors, so will prevent combining vectors that are incompatible.

## Null
The empty vector (a.k.a., a collection with no elements) is represented using the literal `null`. It can also be written as `[]`.

Appending `null` is a no-op, returning the original vector:
```
[1, 2, 3] append null # [1, 2, 3]
null append [1, 2, 3] # [1, 2, 3]
```

The `null` vector is *all* types and *no* types at the same time. For example, you can append `null` to a vector of strings and a vector of integers - the result is the same - nothing happens. Therefore, `null` can appear almost anywhere `null` is allowed.

> **NOTE:** Unlike SQL, `null == null` returns `true` in QL.

When declaring a variable (with `let`), the type can be specifed explicitly:
```
let x: i32 = 123;
```

If `x` in the example above can also be `null`, it should be declared like this:
```
let x: i32? = 123;
```

> **NOTE:** Often, the compiler cannot know at compile time whether an operation will result in `null`, so the implicit type of `let` operations will be nullable. Remember: `null` just means an empty collection.

> **NOTE:** There is no special syntax for initializing a variable to a vector (so `let x: i32[] = [123, 234]` is invalid). Since everything is a vector, you only need to specify the element type (so `let x: i32 = [123, 234];`). 

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

Many functions are also designed to return `null` when passed `null` values. Arithmetic functions are just a form of function.