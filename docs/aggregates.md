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

## The Aggregate trait
An aggregator is a *value* whose type implements the `Aggregate` trait - an associative fold with a finishing step (a monoid with a finisher, or what some languages call a "collector"):
```
let Aggregate<In, Acc, Out> = trait {
    let init = fn(self: Self): Acc;                       # the starting accumulator
    let step = fn(self: Self, acc: Acc, x: In): Acc;      # fold one element into the accumulator
    let combine = fn(self: Self, a: Acc, b: Acc): Acc;    # merge two partial accumulators
    let finish = fn(self: Self, acc: Acc): Out;           # produce the final result
};
```

`combine` must be *associative*: `combine(combine(a, b), c)` must equal `combine(a, combine(b, c))`. That is what makes an aggregator *parallelizable* - the runtime can fold independent chunks of the data and merge the partials (across threads, SIMD lanes, or distributed nodes) without changing the answer.

`Sum` is a primitive aggregator that implements the trait directly:
```
let Sum = struct {};

implement Aggregate<f64, f64, f64> for Sum {
    let init = fn(self: Self): f64 => 0.0;
    let step = fn(self: Self, acc: f64, x: f64): f64 => acc + x;
    let combine = fn(self: Self, a: f64, b: f64): f64 => a + b;
    let finish = fn(self: Self, acc: f64): f64 => acc;
};
```

Other primitives follow the same shape: `Count` steps by one per element and combines by adding, and `Median` collects the values and picks the middle in `finish`. A stateless aggregator like `Sum` or `Count` is a *unit value* - `Sum` names both the empty type and its single instance, and it is that instance you pass to `reduce` (`xs.reduce(Sum)`). A parameterized aggregator such as `percentile(0.95)` is a value constructed at the call site.

## Aggregator capabilities
`Aggregate` guarantees only associativity, which is enough to *chunk* a reduction. Two further, optional traits let the compiler do more. Both are *marker traits* - adding no methods at all; a type implements it purely to declare that a property holds.

### Commutative
`Commutative` declares that `combine(a, b) == combine(b, a)` - order does not matter:
```
implement Commutative for Sum {}   # Sum is order-independent
```

Associativity lets you combine chunks; commutativity lets you combine them *in any order*. A non-commutative aggregator (`Joined`, `First`, `Last`) is still parallelizable - its partials just have to be merged left to right.

This is the property that decides whether an aggregator can run on an *unordered* collection. Reducing an unordered collection - a [set](./collections.md#set) or an unordered partition - requires `Commutative`, since otherwise the result would depend on an order that is not defined. Reducing an *ordered* collection requires only `Aggregate`. The gate is a trait bound, so `someSet.reduce(Joined)` is a compile-time error while `someSet.reduce(Sum)` is fine.

| aggregator | `Commutative`? | valid on unordered collections |
|---|---|---|
| `Sum`, `Count`, `Min`, `Max`, `Median` | yes | yes |
| `Joined`, `First`, `Last` | no | no - ordered only |

> **NOTE:** The safe default is *non*-commutative: an aggregator is order-sensitive unless it implements `Commutative`. That way a user-defined aggregator that forgets to opt in is rejected on unordered data rather than silently producing a nondeterministic result.

### Invertible
`Invertible` declares that an element can be *removed* from an accumulator, not just added:
```
implement Invertible for Sum { 
    let uncombine = fn(self: Self, acc: f64, x: f64): f64 => acc - x;
}
```

This unlocks efficient *sliding* frames. As a trailing window like `p[-2..=0]` moves across a partition, an invertible aggregator can add the entering element and subtract the leaving one in O(1) per step, instead of re-reducing the whole frame. `Sum` and `Count` are invertible; `Max` and `Min` are not (you cannot cheaply recover the previous maximum once it leaves), so their sliding frames fall back to re-reduction.

A derived aggregator has a capability when all of its parts do: `Average`, built from the commutative, invertible `Sum` and `Count`, is itself commutative and invertible.

## Composing aggregators
`Aggregate` is a *trait*, not a fixed set of variants: any type can implement it, and aggregators are combined through ordinary combinators, each returning a new aggregator value. Two combinators cover the common cases, and both keep their parts as visible values, which is what lets the compiler fuse and share them.

### Compound
`Compound` takes a record of aggregators and a finisher, and returns an aggregator that runs them all in a single pass:
```
let Average = Compound(
    { sum: Sum, count: Count },
    fn(a) => if a.count == 0 { None } else { a.sum / f64.from(a.count) }
);
```

The record `{ sum: Sum, count: Count }` is ordinary data - a record whose fields happen to be aggregator values. It becomes an aggregator only because `Compound` wraps it, and that `Compound(...)` is the signal: no guessing whether a bare record is data or a reducer. `Compound` folds each field over the elements, collects the results into a record with the *same field names* (`{ sum, count }`), and hands that to the finisher, which returns the final value - here `f64?`, guarding the empty count. That intermediate record is the same shape as the [`aggregate`](./queries.md#aggregation) block in a query, and a field of a `Compound` may itself be a `Compound`, so aggregators nest freely.

### Projected
`Projected` adapts what a *single* aggregator reads: each element is passed through a projection *before* the aggregator folds it - it transforms the *input* to `step`, never its result:
```
let SumOfSquares = Sum.before(fn(x) => x * x);
# [1, 2, 3].reduce(SumOfSquares) == 14   (1 + 4 + 9)
```
`Sum` itself is unchanged; `before` squares each element on the way in. `agg.before(projection)` is sugar for `Projected(projection, agg)` - and because `agg` is unambiguously an aggregator value, the method form is safe here even though `Compound` needs the explicit constructor.

`Projected` earns its keep inside a `Compound`, where every field sees the *same* element but each may need a different view of it in one pass. `Variance` sums the values and their squares together:
```
let Variance = Compound(
    { sumSq: Sum.before(fn(x) => x * x), sum: Sum, count: Count },
    fn(a) => if a.count == 0 { None } else {
        let n = f64.from(a.count);
        a.sumSq / n - (a.sum / n) ** 2
    }
);
```

> **NOTE:** Because their finishers can yield `None`, `Average` and `Variance` produce `f64?`. Any aggregator that divides by a count should guard the empty collection this way; the [optional rules](./queries.md#optionals) then carry the `None` through any downstream expression.

### Why derived aggregators stay cheap
Two properties fall out of building aggregators from visible parts.

**No runtime map.** A Compound is conceptually a map from field names to sub-aggregators, but those names are compile-time literals, so nothing is a runtime `Map`. The parts lower to a fixed record (a struct with fields `sum`, `count`, and so on), and the running accumulator is a record of the sub-accumulators. `step` and `combine` delegate field by field, and `finish` calls each sub-finisher and then the compound finisher - all statically, with no hashing and no per-element allocation beyond the fixed accumulator.

**Shared work.** Because the parts are visible values, the compiler can share them. When an [`aggregate`](./queries.md#aggregation) block requests `sum`, `count`, and `average` together, it sees that `Average`'s `Sum` and `Count` are the same aggregators as the standalone fields and keeps a single running sum and count for all three. An opaque aggregator that hid its own sum and count would force a redundant second pair - which is why the standard library builds aggregators by composition.

## Backends
Because an aggregator is an ordinary value, a backend can treat it as data:

* An **in-memory** or **streaming** backend simply executes `init`, `step`, `combine`, and `finish`. Every aggregator works, including user-defined ones, with no registration.
* A **SQL** backend maps recognized aggregators to native operations (`Sum` to `SUM`, `Average` to `AVG`, and so on). An aggregator it does not recognize is a capability gap: the backend can pull the rows back and run the aggregator locally, or report that it cannot translate the query.
* To tailor an aggregator to a specific technology, it may carry backend-specific lowerings. For example, a `Median` aggregator could provide a SQL form (`PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY ...)`) so that it lowers natively on databases that support it, while still falling back to its `init`/`step`/`combine`/`finish` in memory.

## Reduce and scan
An aggregator only says *how* to accumulate; there are two ways to *drive* it across a collection.

**Reduce** collapses the whole collection to a single value - the final accumulator, finished:
```
[1, 2, 3, 4].reduce(Sum)   # 10
```

**Scan** keeps every intermediate result, producing a collection the same length as the input, each element the running result up to that point:
```
[1, 2, 3, 4].scan(Sum)     # [1, 3, 6, 10]
```

`scan` is a `reduce` that emits its progress instead of only its endpoint:

* `xs.scan(agg)[i]` equals `xs[..=i].reduce(agg)` - each scan output is a reduce of a prefix.
* `xs.reduce(agg)` equals `xs.scan(agg).last()` - reduce is the final element of a scan.

Same aggregator, two drivers: `reduce` observes only the endpoint, `scan` the whole trajectory. Both rely on the associative `combine`, so both parallelize - `reduce` as a tree reduction, `scan` as a parallel prefix scan.

These are the same two drivers behind the query clauses:

| | collapse (reduce) | keep rows (scan) |
|---|---|---|
| **query** | `aggregate` | `partition by` + a running frame |
| **method** | `.reduce(agg)` | `.scan(agg)` |

An [`aggregate`](./queries.md#aggregation) collapses a collection (or each group) to one row - it is `reduce`. A [window](./queries.md#windows) over a *running* frame keeps every row and attaches the result so far - that is `scan`: `p[..=0].amount.sum()` per row is exactly `scan(Sum)` over the partition. Other frame shapes generalize it - a trailing frame `p[-2..=0]` is a sliding-window reduce (cheap when the aggregator is [`Invertible`](#invertible)), and a whole-partition frame `p[..]` is a reduce broadcast to every row. All of them use the same aggregators; nothing is special-cased.
