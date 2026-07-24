# Generics
Generics let a definition be written once and reused across many types. A *type parameter* is a stand-in for a type that is supplied later - by the caller, or by inference.

## Declaring type parameters
Type parameters are declared on the `let` name, in angle brackets, immediately after it. This is the same spot for every kind of definition, because everything is a `let`:
```
let Pair<A, B> = struct { first: A, second: B };
let Comparable<T> = trait { fn compareTo(other: T): i32; };
let identity<T> = fn(x: T): T => x;
```

At the use site, arguments fill the slots:
```
let p: Pair<i32, String> = { first: 1, second: "one" };
```

Often the arguments are inferred from the values involved, and nothing needs to be written:
```
identity(42)   # T is i32, inferred from the argument
```

## Bound generics
An unconstrained parameter can be stored and passed around, but little else - you cannot call methods on it, because nothing guarantees it has any. A *bound* constrains a parameter to types that implement given traits, which unlocks those traits' methods inside the body.

QL states bounds with `where`, and **only** `where` - there is no inline `<T: Bound>` form. It is more verbose, but every constraint is spelled out explicitly, in one place:
```
let max<T> = fn(a: T, b: T): T
    where T: Comparable<T>
{
    if a.compareTo(b) >= 0 { a } else { b }
};
```

A `where` clause always sits **immediately before the body's opening `{`**. For a function it follows the parameter list and return type, as above. For a `struct` or `trait` nothing comes between the keyword and the brace, so the clauses follow the keyword directly:
```
let SortedSet<T> = struct
    where T: Comparable<T> {
    // fields
};
```

The parameters and their constraints deliberately sit on opposite sides of the `=`: the parameters (`<T>`) are part of how the type or function is *named*, so they stay on the `let` name to the left; the constraints belong to the definition, so they live with it on the right. Name and arity on the left, constraints on the right.

> **NOTE:** Because a `where` clause always precedes a `{`, an [expression-bodied function](./functions.md#expression-bodied-functions) (the `=>` form) cannot carry one. This is deliberate - the `=>` form is for trivial one-liners, and a bounded generic signature is not one; a function that needs a bound takes a block body. Inline functions passed as arguments never need a `where` anyway, since their parameter types and any bounds come from the context they are passed into.

Multiple bounds on a single parameter combine with `&` - the same intersection used for [composing traits](./entities.md#composing-traits):
```
where T: Comparable<T> & Hashable
```

Each parameter that needs a bound gets its **own** `where`. The keyword repeats; there is no comma joining them:
```
let index<K, V> = fn(pairs: Vector<(K, V)>): Map<K, V>
    where K: Hashable & Comparable<K>
    where V: Clone
{
    // ...
};
```

Repeating `where` lines each constraint up on its own row - the same shape as the leading `&` and `|` that give each composed trait or union variant its own line.

> **NOTE:** [Queries](./queries.md#where) also use `where`, to filter rows. The two never collide: one appears in a type-parameter signature, the other in a query body. C# reuses `where` for these very same two roles - LINQ's filter and a generic constraint - for the same reason: the contexts are disjoint.

A `where` clause is not only tidier for long constraint lists - it can say things an inline bound cannot. It can constrain a type that is not the bare parameter, such as an associated or related type:
```
where T.Item: Clone
```
There is no parameter literally named `T.Item` to annotate, so this can only be expressed as a clause. That reach is part of why `where` is the single form.

## Constant generics
A parameter can also be a compile-time *value* rather than a type - a *constant generic*. `Array2d` is parameterized by its element type and by its two dimensions:
```
let Array2d<T, N, M> = struct
    where N: const i32
    where M: const i32 {
    // ...
};
```

A constant generic is declared like any other parameter and constrained with a `where` clause, where `const` marks it as a *value* of the given type rather than a type *bounded* by it. At the use site you supply the literal; its type is already fixed by the declaration, so no annotation is needed:
```
let grid: Array2d<f64, 3, 4> = ...;   # a 3x4 grid of f64
```

The `const` keyword is load-bearing. Without it, `where N: i32` would become ambiguous the moment user-defined constant types exist: given a `Rational` type with compile-time-constant support, `where T: Rational` could mean either *T is bounded by the trait `Rational`* or *T is a constant `Rational` value*. `where T: const Rational` says the latter unambiguously; the plain form is always a trait bound. The distinction is semantic, not syntactic, so it earns a keyword rather than a parsing rule.

Inside an [`implement` block](#type-parameters-in-implementation-blocks), a constant parameter is referred to by its bare name, exactly like a type parameter - its const-ness is inherited from the declaration:
```
implement Iterable<T> for Array2d<T, N, M> { ... }
```

## Type parameters in implementation blocks
An [`implement` block](./entities.md#traits) has no `let` name to hang parameters on, so it introduces them by a different rule:

> In an implement header, a type name that does not resolve to a type in scope is a freshly-introduced parameter; a name that does resolve is concrete.

```
implement Serializable for Vector<i32>    # i32 resolves - concrete
implement Serializable for Vector<T>      # T does not resolve - a parameter
```

This is what lets one header form serve both a concrete instantiation and a generic one, with no separate binder like Rust's `impl<T>`. Bounds on those parameters use the same `where` clauses:
```
implement Comparable<Vector<T>> for Vector<T>
    where T: Comparable<T>
{ ... }
```

Two conventions keep the "unresolved name is a parameter" rule from biting:

* Parameters are short uppercase names (`T`, `K`, `V`, `N`, `M`); concrete types have full names. Every implement block in these docs follows this.
* A header parameter used only once is almost always a mistyped or un-imported type name, and is flagged: *looks like a type name but is unbound - import it, or use a conventional parameter name.*
