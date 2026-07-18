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

Some examples:

* `Sum` - `init = fn() => 0`, `step = fn(a, x) => a + x`, `combine = fn(a, b) => a + b`, `finish = fn(a) => a`.
* `Average` - accumulates a running sum and count (`Acc = (f64, usize)`), then `finish = fn(acc) => acc._0 / acc._1`. This is precisely why `average` cannot be a plain keyword but *is* a trivial aggregator.
* `Median` - accumulates all values, then computes the middle in `finish`.

## Backends
Because an aggregator is an ordinary value, a backend can treat it as data:

* An **in-memory** or **streaming** backend simply executes `init`, `step`, `combine`, and `finish`. Every aggregator works, including user-defined ones, with no registration.
* A **SQL** backend maps recognized aggregators to native operations (`Sum` to `SUM`, `Average` to `AVG`, and so on). An aggregator it does not recognize is a capability gap: the backend can pull the rows back and run the aggregator locally, or report that it cannot translate the query.
* To tailor an aggregator to a specific technology, it may carry backend-specific lowerings. For example, a `Median` aggregator could provide a SQL form (`PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY ...)`) so that it lowers natively on databases that support it, while still falling back to its `init`/`step`/`combine`/`finish` in memory.

## Relationship to `aggregate` and windows
The [`aggregate`](./queries.md#aggregation) operation and [window](./queries.md#windows) frames both reduce collections, so they use the same aggregators. `g.count()` after a `group by` and `p[-2..=0].amount.average()` in a window are both `reduce` calls - through their sugar methods - over the appropriate collection. Nothing about aggregation is special-cased; it is `reduce` applied to whatever collection is in hand.
