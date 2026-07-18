# In-memory sources
All queries in QL begin with a source of data. For starters, we'll focus on an in-memory source, or collection.

An in-memory collection can be defined using the `[]` syntax. For example:
```
let values = [1, 2, 3];
```

This defines a collection with 3 elements: `1`, `2`, and `3`.

Items can be retrieved from the collection using the `[]` operator:
```
let first = values[0];  # 1
let second = values[1]; # 2
let third = values[2];  # 3  
```

The `#` symbol indicates a comment. These are ignored by the compiler. They last until the end of the current line.

## Empty collections
An empty collection would look like this:
```
let empty: i32[] = [];
```

An empty collection is a real collection that simply has zero elements. `[]` is a present value: you can iterate it (zero times), count it (`0`), and reduce it.

Because an empty collection has no elements, its element type cannot be inferred from its contents; it must come from context, such as the `: i32[]` annotation above. Written bare, `[]` is an empty collection of whatever element type the surrounding code expects:
```
let evens: i32[] = [];      # an empty i32[]
let names: String[] = [];   # an empty String[]
```

## Concatenating
Collections are immutable by default. This means the contents of the collection cannot be updated or deleted once initialized.

Two vectors can be concatenated together using copy notation:
```
let values = [...values, 4]; # produces [1, 2, 3, 4]
```

> **NOTE:** that we reassign `values`; this *shadows* the previous variable, making it inaccessible in the rest of the scope.

## Querying
The values in a collection can be queried using a query. We will go much more into what a query looks like [later on](./queries.md). For now, this is a very basic example:
```
let evens = 
    from values as v
    where v % 2 == 0
    select v;         # produces [2, 4] 
```

> **NOTE:** A query is lazily evaluated, meaning it will not produce `[2, 4]` until the values are actually retrieved.

## Splicing
A new collection can also be created using splices. Here are some example splices:
```
let a = values[..2];   # produces [1, 2]        (end index excluded)
let b = values[1..];   # produces [2, 3, 4]     (index 1 through the end)
let c = values[1..3];  # produces [2, 3]        (indices 1 and 2; end excluded)
let d = values[1..=3]; # produces [2, 3, 4]     (indices 1, 2, and 3; end included)
let e = values[..];    # produces [1, 2, 3, 4]  (everything)
let f = values[0..:2]; # produces [1, 3]        (every other element)
let g = values[..:-1]; # produces [4, 3, 2, 1]  (reversed, via a step of -1)
```

> **NOTE:** A `..` range *excludes* its end, while a `..=` range *includes* it. This is why `values[0..2]` yields two elements while `values[0..=2]` yields three. Excluding the end is convenient for counts, since `0..n` always produces exactly `n` elements. Omitting either bound (as in `[..2]`, `[1..]`, or `[..]`) uses the collection's own start or end. An optional step can follow a `:`, and a negative step walks the collection in reverse. See [Range operators](./operators.md#range-operators).

We can assign multiple variables at once using this syntax:
```
let [first, second, third] = values[0..3]; # binds 1, 2, 3
```

Using the splice syntax, the number of elements in the range must match the number of variables being defined; otherwise, an error occurs.

We don't need to splice the collection if we know how many elements there are:
```
let [v1, v2, v3, v4] = values;
```

The following is also legal, so long as the collection is not empty:
```
let [first, ...remaining] = values;
```

If `remaining` is unused, it can be replaced with `_` to be ignored:
```
let [first, ..._] = values;
```

Finally, if the number of elements in a collection is unknown, the `?` operator can be used to indicate the value is optional:
```
let [first?, ..._] = []; # first will be `null`
```

A check must be performed on nullable variables before accessing their values. The `??` operator can also be used to provide a default value when needed:
```
let first = first ?? 0;
``` 

Or, more succinctly:
```
let [first ?? 0, ..._] = values;
```

## Mutable collections
A mutable collection can be created using the `mut` keyword:
```
let mut values: i32[] = [];
```

A `mut` collection can be added to, updated, or removed from. 

> **NOTE:** An empty collection has no elements to infer an element type from, so a mutable one must be given an explicit type when the context does not supply it.
