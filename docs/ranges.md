# Ranges
A range describes an ordered sequence of values without listing them out. `1..10` is a range, just as `[1, 2, 3]` is a vector. It is an ordinary value: it can be bound to a variable, passed to a function, and iterated.

Every range is described by four components:

* a **start** - where the sequence begins,
* an **end** - where it stops,
* a **step** - how far apart consecutive values are (defaulting to `1`), and
* an **inclusivity** - whether the end value itself is part of the sequence.

Once you see these four components, the various places ranges appear - generating sequences, slicing collections, and defining window frames - are all the same concept used in different places.

## Creating ranges
The end of a range can be excluded with `..` or included with `..=`:
```
let a = 1..5;   # 1, 2, 3, 4
let b = 1..=5;  # 1, 2, 3, 4, 5
```

Excluding the end is the convenient default for counts, because `0..n` always produces exactly `n` values.

A step can be supplied after a colon (`:`):
```
let c = 0..10:2;   # 0, 2, 4, 6, 8
let d = 0..=10:2;  # 0, 2, 4, 6, 8, 10
```

The default step is `1`. To count downward, provide a negative step:
```
let e = 5..0:-1;   # 5, 4, 3, 2, 1
let f = 5..=0:-1;  # 5, 4, 3, 2, 1, 0
```

> **NOTE:** A range whose direction contradicts its step is empty. `5..0` (with the default step of `1`) produces nothing, because it can never count upward from `5` to `0`. Use `5..0:-1` to count down.

Ranges are not limited to integers. Any ordered, steppable type works, such as `char`:
```
let g = 'a'..='e'; # 'a', 'b', 'c', 'd', 'e'
```

> **NOTE:** Floating-point ranges are permitted (e.g., `0.0..=1.0:0.25`) but should be used with care, since accumulated rounding can cause the final value to be missed.

## Ranges as sequences
Because a range is an ordered sequence, it can be the source of a query:
```
from 1..=10 as v
where v % 2 == 0
select v; # [2, 4, 6, 8, 10]
```

A range with no end is infinite. This is only useful because ranges (like all sequences) are evaluated lazily, so an infinite range can be combined with an operation that stops early:
```
from 1.. as v
take 5
select v; # [1, 2, 3, 4, 5]
```

## Ranges as indexes
When a range is used to index a collection, it produces a slice - a sub-collection. Here the range describes *which positions* to take:
```
let values = [10, 20, 30, 40];
let a = values[1..3];  # [20, 30]    (positions 1 and 2; end excluded)
let b = values[1..=3]; # [20, 30, 40] (positions 1, 2, and 3; end included)
let c = values[..:-1]; # [40, 30, 20, 10] (reversed, via a step of -1)
```

When used as an index, either bound may be omitted, in which case the collection's own start or end is used:
```
let d = values[..2]; # [10, 20]        (start through position 1)
let e = values[2..]; # [30, 40]        (position 2 through the end)
let f = values[..];  # [10, 20, 30, 40] (everything)
```

See [Splicing](./in-memory-sources.md#splicing) for more on slicing collections.

## Ranges as window frames
A [window](./queries.md#windows) uses a range to describe its *frame* - the set of rows around the current row that a computation can see. In this position, the range is measured relative to the current row rather than to position `0`:
```
from orders as o
partition by o.customerId
order by o.date
let movingAvg = o.amount.average() over [-2..=0] # current row and the two before it
select { ...o, movingAvg };
```

Here `0` is the current row, negative offsets are preceding rows, and positive offsets are following rows. See [Windows](./queries.md#windows) for the full treatment.

## Position vs. value
This is the distinction that ties everything together, and the one most worth internalizing. A range always ranges over *some axis*, and there are two axes it might use:

* **Position** - the range counts elements. `values[1..3]` takes the elements *at* positions 1 and 2. A window frame `[-2..=0]` takes the current row and the *two rows* before it. The length of the result is fixed by the range itself.
* **Value** - the range selects by the magnitude of some ordered value. "Every order whose day falls within the previous week" may match any number of rows.

For a plain vector these two coincide, because an element's position *is* effectively its coordinate. They diverge the moment the axis is something other than a dense position - most visibly in a window ordered by a value like a date, where "the previous 3 rows" and "the previous 3 days" are genuinely different questions.

QL keeps both available and distinguishes them by what the range is written against:

* A range of bare offsets - `[-2..=0]` - counts by **position**.
* A range written in terms of an ordered value - `[o.day - 6 ..= o.day]` - counts by **value**.

The two forms even line up shape-for-shape. A running frame over positions is `[..=0]` (from the start through the current row); the same running frame over a value is `[..= o.day]` (from the start through the current row's value). Same range, different axis. This is developed further in [Windows](./queries.md#windows).
