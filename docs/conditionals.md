# Conditionals
QL provides `if`/`else` and `match` as *expressions*: each evaluates to a value, the way SQL's `CASE` does. Because they are expressions, they can appear anywhere a value is expected - in a `let`, inside a `select`, as an argument, or on their own.

> **NOTE:** These are expressions, not imperative statements. An `if` always produces a value (which is why `else` is required), and there is no statement form that reassigns a variable. Branching that mutates state is a separate topic QL does not yet address.

## If and else
An `if` expression chooses between two values based on a boolean condition:
```
let title = if c.sex == "male" { "sir" } else { "lady" };
```

The `else` is required. Because an `if` is an expression, it must always produce a value, so there is no "missing" branch. Both branches must also produce the same type.

The body of each branch is a block whose last expression is its value - the same rule as a function's block body (see [Functions](./functions.md)). A branch can therefore do intermediate work before yielding its value:
```
let shipping =
    if order.isExpedited {
        let base = order.weight * 2.0;
        base + 5.0
    } else {
        order.weight * 1.0
    };
```

Additional conditions are chained with `else if`:
```
let grade =
    if score >= 90 { "A" }
    else if score >= 80 { "B" }
    else if score >= 70 { "C" }
    else { "F" };
```

Being an expression, an `if` slots directly into a query:
```
from customers as c
select {
    name: c.name,
    greeting: if c.isVip { "Welcome back!" } else { "Hello" }
};
```

> **NOTE:** QL has no ternary operator. The `if` expression fills that role, and it reads the same whether it produces one value or chains many.

## Match
A `match` expression compares a value against a series of patterns and evaluates the arm of the first one that matches. It is the general form of a multi-way choice - QL's equivalent of a `CASE`:
```
let title = match c.sex {
    "male"   => "sir",
    "female" => "lady",
    _        => "person"
};
```

Each arm is written `pattern => expression`, using the same `=>` arrow as an expression-bodied function. The `_` pattern matches anything; it is the same placeholder used elsewhere to mean "ignore" (see [Tuples](./collections.md#tuple)).

### Exhaustiveness
A `match` must be exhaustive: every possible value of the subject must be covered by some arm, or it is a compile-time error. This guarantees a `match` always produces a value, and it means that introducing a new case forces you to handle it rather than silently falling through. A trailing `_` arm is the usual way to cover everything not named explicitly.

### Patterns and capture
Patterns can destructure structured values, such as tuples. A pattern can also *capture* the value it matched by binding it to a name with `@`:
```
match (x, y) {
    (1, 2) @ p => p._0 + p._1,   # matches the pair (1, 2), binding it to p
    _         => 0
};
```

Here `(1, 2) @ p` matches only when the tuple is `(1, 2)`, and binds the whole tuple to `p` so the arm can use it. (Tuple members are accessed as `._0`, `._1`, and so on - see [Tuples](./collections.md#tuple).)

> **NOTE:** The capture is written *pattern* `@` *name*. Some languages (such as Rust) write this in the opposite order, *name* `@` *pattern*; QL puts the pattern first so an arm reads left-to-right as "match this, then call it that."

### Guards
An arm can add a condition - a *guard* - with `if`. The arm applies only when its pattern matches *and* the guard is true:
```
let sign = match n {
    0          => "zero",
    m if m > 0 => "positive",
    _          => "negative"
};
```

The second arm binds `n` to `m` and applies only when `m > 0`. Guards let a single pattern branch on extra conditions without nesting another `match`.

> **NOTE:** A guard does not count toward exhaustiveness, since the compiler cannot prove a guard is always true. A `match` whose most general arm is guarded still needs a catch-all (`_`) to be exhaustive.
