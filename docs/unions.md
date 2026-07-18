# Unions
A *union* type is a value that is one of several alternatives. Where `&` intersects types (a value must implement *all* parts), `|` unions them (a value is *one of* the parts). It is the composition counterpart to trait intersection, and it is what other languages call a sum type, a tagged union, or an `enum`.

## Variants
A union is written as a series of *variants* joined with `|`. Each variant has a name and may carry a payload:
```
let Shape =
    | Circle(f64) # Radius
    | Rectangle(f64, f64) # height and width
    | Point # Dimensionless
    ;
```

As with `&`, a leading `|` is allowed so each variant lines up on its own line; inline is fine too: `let Shape = Circle(f64) | Rectangle(f64, f64) | Point;`.

A value is constructed by naming a variant:
```
let a = Circle(2.0);
let b = Rectangle(3.0, 4.0);
let c = Point;
```

Each of `a`, `b`, and `c` has type `Shape`. The specific variant is remembered, but the static type is the union.

Every variant's **head** - the name before any payload - is a *fresh* constructor introduced by the union; the payload, inside the parentheses, refers to an existing type. In `Circle(f64)`, `Circle` is new and `f64` is referenced. A head may not reuse a name that already exists, which resolves an otherwise ambiguous case: `let C = A | B` always declares two fresh nullary constructors `A` and `B`, so if `A` is already a type it is a compile-time error - never a union of the existing `A` and `B`. To build a union *from* existing types, wrap each in a fresh head (see [Including an existing type](#including-an-existing-type)).

## Matching
The natural way to consume a union is [`match`](./conditionals.md#match), which destructures the payload and forces every variant to be handled:
```
let area = match shape {
    Circle(r)       => 3.14159 * r ** 2,
    Rectangle(w, h) => w * h,
    Point           => 0.0
};
```

Because a `match` must be [exhaustive](./conditionals.md#exhaustiveness), leaving out a variant is a compile-time error - and *adding* a variant to the union later forces every `match` over it to be updated. This is the main safety benefit of a union over an untyped tag: the compiler tracks the full set of cases for you.

## Optional
`Optional` is the most common union in the language:
```
let Optional<T> = Some(T) | None;
```

The `T?` type is sugar for `Optional<T>`, and `None` is how absence is written. So an optional value is simply a two-variant union, and the optional operators are `match` in disguise:

* `x ?? d` is `match x { Some(v) => v, None => d }`.
* `x?.field` is `match x { Some(v) => Some(v.field), None => None }`.
* `x!` asserts the `Some` variant, and fails if it is `None`.

To test and unwrap in one step, use `is` with a pattern: `x is Some(v)` is true when `x` is present and binds `v` to the inner value for the rest of the scope, so `x is Some(v) and v > 100` filters and reuses `v` together. Reusing the same name - `x is Some(x)` - shadows the optional with its unwrapped value, narrowing in place; this makes narrowing plain lexical shadowing rather than flow-sensitive re-typing of a variable. Use `x is Some(_)` when you only need to know it is present, `x is None` for the absent case, and `x is not None` to keep everything present without binding (see [Optionals](./queries.md#optionals)).

> **NOTE:** This is how QL avoids a `null` value entirely: absence is `Optional`'s `None` variant, handled by the same `match`/`is` machinery as any other union - no special null rules, and no null-dereference class of bug.

## Including an existing type
Because a variant's head is always a fresh constructor, there is no bare union of existing types such as `i32 | String`. To carry an existing type, give a variant a name and wrap the type as its payload:
```
let Id = Number(i32) | Text(String);
```

A value of type `Id` is a `Number` or a `Text`, and `match` unwraps by variant:
```
let text = match id {
    Number(n) => n.toString(),
    Text(s)   => s
};
```

Wrapping is what keeps variants disjoint. A bare `i32 | String` could not tell two same-typed alternatives apart - imagine `f64 | f64` - and it would *flatten*, `A | A` collapsing to `A`. Named heads make every case distinct, which is also what lets `Optional<Optional<T>>` keep `Some(None)` separate from `None`. QL's unions are always *tagged*: a disjoint sum of named constructors, never a structural set-union of types.

> **NOTE:** Where some languages spell an enumeration as a union of string literals (`"mon" | "tue" | ...`), QL uses nullary constructors (`Mon | Tue | ...`) - a real enum, not a bag of strings. Literals and `const` values still appear as *singleton patterns* inside a `match` (`match n { 0 => ..., _ => ... }`), but a union's *members* are always constructors, never bare values.

## Recursion and generics
Unions may be generic and may refer to themselves, which is how recursive structures like trees are described:
```
let Tree<T> =
    | Leaf(T)
    | Branch(Tree<T>, Tree<T>);
```

> **NOTE:** A self-referential variant needs an indirection under the hood so the type has a finite size. How that indirection is represented depends on the eventual memory model (see the discussion of reference types), so treat the exact representation as unsettled.

## The `&` / `|` duality
Intersection and union are the two ways to build a compound type, and they mirror each other:

* `A & B` - *both*. A value implements every part; used to [compose traits](./entities.md#composing-traits).
* `A | B` - *one of*. A value is a single one of the parts; consumed with `match`.

Both are ordinary type expressions bound with `let`, so a name can stand for an intersection, a union, or a plain alias, with no dedicated keyword beyond `struct` and `trait` for introducing genuinely new types.
