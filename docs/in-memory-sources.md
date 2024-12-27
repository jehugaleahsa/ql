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

Without values to initialize a collection, its type is determined to be `null`. A `null` can be used in the context of any other type. To give a collection an explicit type, the following syntax is used:
```
let empty: i32 = [];
```

This collection can only be used where a collection of 32-bit integers are expected. More on `i32` when we discuss [primitive types](./primitive-types.md).

## Concatenating
Collections created with `[]` are immutable by default. This means the contents of the collection cannot be updated or deleted once initialized.

Appending to a collection can be achieved using the `append` operation. This results in a new collection:
```
let values = append values [4]; # produces [1, 2, 3, 4]
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

The previous example that assigns `first`, `second`, and `third` can be more succinctly written:
```
let first, second, third = values[0..2];
```

The number of elements in the range must match the number of variables being defined; otherwise, an error occurs.

The following is also legal, so long as the collection is not empty:
```
let first, ...remaining = values;
```

If `remaining` is unused, it can be replaced with `_` to be ignored:
```
let first, ..._ = values;
```

Finally, if the number of elements in a collection is unknown, the `?` operator can be used to indicate the value is optional:
```
let first?, ..._ = values; # values may be empty
```

A check must be performed on optional variables before accessing their values. The `??` operator can also be used to provide a default value when needed:
```
let first = first ?? 0;
``` 

## Mutable collections
A mutable collection can be created using the `mut` keyword:
```
let values: i32 = mut [];
```

A `mut` collection can be added to, updated, or removed from. 

> **NOTE:** There is no mutable variant of `null`, so a type must be specified if it cannot be inferred.

## Appending values
Values can be inserted onto `values` using the `into` keyword:
```
from [1, 2, 3, 4] as v
select v
into values;
```

The `into` keyword specifies the target for a query. A target is anything that is appendable, such as a mutable in-memory collection.

## Updating values
Values can be updated in-place using the `update` operation:
```
from values as v
where values % 2 == 0
update v (v + 1); # produces [1, 3, 3, 5]
```

After the `update` keyword, the next value is assigned to a variable (named `v` here). After that is the new value (`v + 1` here). This get more interesting when working with a complex type:
```
from customer as c
where c.id == 123
update c { ...c, firstName: "Bob" };
```

Here, we are saying replace the first name of the customer whose `id == 123` with `"Bob"`. 

## Deleting values
Values can also be removed in-place, using the `delete` operation:
```
from values as v
where value % 2 == 0
delete v; # produces [1, 3]
```