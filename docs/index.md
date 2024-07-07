# Intoduction
**Q**uery **L**anguage is an attempt to informally define a grammar for a next-gen programming language. It can also behave as a SQL transpiler. The premise is that nearly all computations on a modern day computer can be described using a series of SQL-like queries that are composable. Operations which cannot be easily performed in QL are best left done in another programming language. The hope of this specification is reduce the overall number of situations where QL cannot be used, defining a general-purpose language that suits all purposes.

Those familiar with SQL, functional programming, or vector programming concepts should feel right at home. For those coming from a procedural programming background, it might take some getting used to.

# Table of contents
1. [Introduction](#intoduction)
2. [In-memory sources](#in-memory-sources)
    * [Concatenating](#concatenating)
    * [Querying](#querying)
    * [Splicing](#splicing) 
    * [Mutable collections](#mutable-collections)
    * [Appending values](#appending-values)
    * [Updating values](#updating-values)
    * [Deleting values](#deleting-values)

# In-memory sources
All queries in QL begin with a source of data. For starters, we'll focus on an in-memory source, or collection.

An in-memory collection can be defined using the `[]` syntax. For example:
```
let values = [1, 2, 3];
```

This defines a collection with 3-elements: `1`, `2`, and `3`. This should be familiar to anyone who has worked in most other programming languages.

## Concatenating
Collections created with `[]` are immutable by default. This means the contents of a collection cannot be updated or deleted once created.

Adding a new item can be accomplished using the `append` operation. This results in a new collection:
```
let values = values append 4; # produces [1, 2, 3, 4]
```

Notice that we reassign `values`; this shadows the previous variable, making it inaccessible in the rest of the scope. The `#` symbol indicates a comment.

## Querying
The values in a collection can be queried using a query. We will go much more into what a query looks like later on. For now, this is a very basic example:
```
let evens = 
    from values as v
    where v % 2 == 0
    select v;         # produces [2, 4] 
```

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

## Mutable collections
A mutable collection can be created using the `mut` keyword:
```
let values = mut [];
```

A `mut` collection can be added to, updated, or removed from.

## Appending values
Values can be inserted onto the end of `values` using an insert command.
```
from [1, 2, 3, 4] as v
select v
into values;
```

The `into` keyword specifies the target for a query. A target is anything that is appendable, such as an in-memory collection.

## Updating values
Values can be updated in-place using the `update` operation:
```
from values as v
where values % 2 == 0
update v (v + 1); # produces [1, 3, 3, 5]
```

After the `update` keyword, the next value (`v`) is the value being updated. After that is the new value (`v + 1`). This get more interesting when working with more complex types.

## Deleting values
Values can also be removed in-place, using the `delete` operation:
```
from values as v
where value % 2 == 0
delete v; # produces [1, 3]
```