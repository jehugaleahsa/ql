# Aggregate functions

> **NOTE:** This is a design direction under consideration, not settled syntax.

## The problem
Languages like SQL and PRQL treat aggregates - `sum`, `count`, `average`, and so on - as a fixed set of built-in keywords. This is limiting in two ways. You cannot add a new aggregate, such as a `median`, a `percentile`, a geometric mean, or a domain-specific rollup, without changing the language itself. And you cannot tailor *how* an aggregate is carried out on a particular backend.

QL aims for aggregates to be an *open* set: ordinary values that anyone can define, that plug into a single universal `reduce`, and that a backend can either translate to a native operation or execute directly.

## Universal reduce
Every aggregation goes through one method, `reduce`, which takes an *aggregator* - a value describing how to combine elements:
```
let total = p.amount.reduce(Sum)
let avg   = p.amount.reduce(Average)
let mid   = p.amount.reduce(Median)
let p95   = p.amount.reduce(percentile(0.95))  # a parameterized aggregator
```

The common built-ins also have sugar methods, so everyday queries stay concise:
```
let total = p.amount.sum()   # sugar for p.amount.reduce(Sum)
let n     = p.count()        # sugar for p.reduce(Count)
```

> **NOTE:** The sugar methods are defined in terms of `reduce` - they add nothing you could not write yourself. This keeps the common case readable while leaving the set of aggregators open. It also means the set of "keywords" is really just a standard library of aggregator values.

## The Aggregate interface
An aggregator implements a small interface: an associative fold with a finishing step. This is the same shape as a monoid with a finisher (or what some languages call a "collector"):
```
let Aggregate<In, Acc, Out> = type {
    init:    fn(): Acc,           # the starting accumulator
    step:    fn(Acc, In): Acc,    # fold one element into the accumulator
    combine: fn(Acc, Acc): Acc,   # merge two partial accumulators
    finish:  fn(Acc): Out         # produce the final result
};
```

The `combine` step is what makes an aggregator *parallelizable*. Because it is associative, the runtime is free to fold different chunks of the data independently and merge the partial results - across threads, SIMD lanes, or distributed nodes - without changing the answer.

A few *primitive* examples - aggregators that implement the interface directly:

* `Sum` - `init = fn() => 0`, `step = fn(a, x) => a + x`, `combine = fn(a, b) => a + b`, `finish = fn(a) => a`.
* `Count` - `init = fn() => 0`, `step = fn(a, _) => a + 1`, `combine = fn(a, b) => a + b`, `finish = fn(a) => a`.
* `Median` - accumulates all values, then computes the middle in `finish`.

## Primitive and derived aggregators
Aggregators come in two kinds.

A *primitive* aggregator implements the interface directly, as `Sum` and `Count` do above. These are the building blocks.

A *derived* aggregator is built by combining existing aggregators; it never writes its own `init`/`step`/`combine`. Three combinators cover the common cases:

* **Product** - a tuple of aggregators is itself an aggregator. It runs them together in a single pass and yields a tuple of their results, so `(Sum, Count)` produces `(f64, usize)`.
* **Output map** - `agg.map(finisher)` transforms an aggregator's result after it finishes.
* **Input map** - `agg.on(projection)` adapts what an aggregator reads from each element, so that sub-aggregators of a product can read the same element differently.

`Average` is the canonical derived aggregator: run `Sum` and `Count` together, then divide. It returns `f64?` - an empty collection has a count of zero, so the finisher yields `null` rather than dividing by zero:
```
let Average = (Sum, Count).map(fn(acc) =>
    if acc._1 == 0 { null } else { acc._0 / acc._1 as f64 }
);
```

`Variance` uses all three combinators, computing the mean of the squares and the square of the mean in a single pass, and guards the empty case the same way:
```
let Variance =
    (Sum.on(fn(x) => x * x), Sum, Count).map(fn(acc) => {
        let (sumSq, sum, n) = acc;
        if n == 0 { null } else {
            let nf = n as f64;
            sumSq / nf - (sum / nf) ** 2
        }
    });
```

> **NOTE:** Because their finishers can yield `null`, `Average` and `Variance` produce `f64?`. Any aggregator that divides by a count should guard the empty collection this way; the [null-safety rules](./queries.md#null-safety) then carry the `null` through any downstream expression.

Deriving aggregators is more than convenience. Because a derived aggregator is built from visible parts, the compiler can *share* those parts. When an [`aggregate`](./queries.md#aggregation) block requests `sum`, `count`, and `average` together, it can see that `Average` is made of the same `Sum` and `Count` as the standalone fields, and maintain just one running sum and one running count for all three. A monolithic `Average` that hid its own sum and count would force a second, redundant pair. This is why the standard library favors deriving aggregators from smaller ones rather than writing each from scratch.

## Backends
Because an aggregator is an ordinary value, a backend can treat it as data:

* An **in-memory** or **streaming** backend simply executes `init`, `step`, `combine`, and `finish`. Every aggregator works, including user-defined ones, with no registration.
* A **SQL** backend maps recognized aggregators to native operations (`Sum` to `SUM`, `Average` to `AVG`, and so on). An aggregator it does not recognize is a capability gap: the backend can pull the rows back and run the aggregator locally, or report that it cannot translate the query.
* To tailor an aggregator to a specific technology, it may carry backend-specific lowerings. For example, a `Median` aggregator could provide a SQL form (`PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY ...)`) so that it lowers natively on databases that support it, while still falling back to its `init`/`step`/`combine`/`finish` in memory.

## Relationship to `aggregate` and windows
The [`aggregate`](./queries.md#aggregation) operation and [window](./queries.md#windows) frames both reduce collections, so they use the same aggregators. `g.count()` after a `group by` and `p[-2..=0].amount.average()` in a window are both `reduce` calls - through their sugar methods - over the appropriate collection. Nothing about aggregation is special-cased; it is `reduce` applied to whatever collection is in hand.
