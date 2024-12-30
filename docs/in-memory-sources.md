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

## Empty collections and `null`
An empty collection would look like this:
```
let empty = []; # compiles
```

Without values to initialize a collection, its type is determined to be `null`. A `null` collection can be used in the context of any other collection. To give a collection an explicit type, the following syntax is used:
```
let empty: i32[] = [];
```

This collection can only be used where a collection of 32-bit integers are expected. More on `i32` when we discuss [primitive types](./primitive-types.md).

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
let a = values[..2];  # produces [1, 2, 3]
let b = values[1..];  # produces [2, 3, 4]
let c = values[1..2]; # produces [2, 3]
let d = values[..]    # produces [1, 2, 3, 4]
let e = values[0..:2] # produces [1, 3]
let f = values[..:-1] # produces [4, 3, 2, 1]
```

We can assign multiple variables at once using this syntax:
```
let [first, second, third] = values[0..2];
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

> **NOTE:** There is no mutable variant of `null`, so a type must be specified if it cannot be inferred.