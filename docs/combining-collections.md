# Combining collections
Multiple collections can be combined using concatenation, `distinct`, and joins.

## Concatenate
A collection can be concatenated to another collection using a copy operation, returning a new collection:
```
let first = [1, 2, 3];
let second = [4, 5, 6];
let combined = [...first, ...second]; # [1, 2, 3, 4, 5, 6]
```

> **NOTE:** When concatenating immutable collections, concatenation may be performed lazily.

## Distinct
The unique values that appear among collections can be retrieved using the `distinct` operation. No duplicates will be returned:
```
let first = [1, 2, 3];
let second = [2, 3, 4];
let combined =
    from [...first, ...second] as v
    distinct v; # [1, 2, 3, 4]
```

If the element type is not comparable, a `on` clause can be provided:
```
let Thing = struct {
    id: i32
};
let first: Thing[] = [{ id: 1 }, { id: 2 }];
let second: Thing[] = [{ id: 2 }, { id: 3 }];
let combined = 
    from [...first, ...second] as t
    distinct t on t.id
    select t; # -> [{ id: 1 }, { id: 2 }, { id: 3 }]
```

## Intersection
Values that only appear in both collections can be retrieved using a `join`:
```
let first = [1, 2, 3];
let second = [2, 3, 4];
let combined = 
    from first as f
    join second as s on f == s
    select f; # [2, 3]
```

Here is the same example using a complex type:
```
let first: Thing[] = [{ id: 1 }, { id: 2 }];
let second: Thing[] = [{ id: 2 }, { id: 3 }];
let combined = 
    from first as f
    join second as s on f.id == s.id
    select f; # -> [{ id: 2 }]
```

## Set difference
Values that only appear in the first collection, and not the second, can be retrieved using a `left join`:
```
let first = [1, 2, 3];
let second = [2, 3, 4];
let combined = 
    from first as f
    left join second as s on f == s
    where s is None
    select f; # [1]
```

Here is the same example using a complex type:
```
let first: Thing[] = [{ id: 1 }, { id: 2 }];
let second: Thing[] = [{ id: 2 }, { id: 3 }];
let combined =
    from first as f
    left join second as s on f.id == s.id
    where s is None
    select f; # -> [{ id: 1 }]
```

## Zip
The operations above combine collections by *content* - matching or comparing values. `zip` instead combines them by *position*: it pairs the first element of one collection with the first of another, the second with the second, and so on. It turns two collections into one collection of pairs:
```
let numbers = [1, 2, 3];
let letters = ['a', 'b', 'c'];
let paired =
    from numbers as n
    zip letters as l
    select { n, l }; # [{ n: 1, l: 'a' }, { n: 2, l: 'b' }, { n: 3, l: 'c' }]
```

Where a `join` pairs elements by a condition (producing every matching combination), `zip` pairs them by their order alone. For that reason `zip` requires ordered collections; zipping an unordered [set](./collections.md#set) is not meaningful.

A `zip` pairs the new collection against the *whole query so far*, not just the immediately preceding source. So when a `zip` follows a `join`, it zips the joined rows against the new collection:
```
from customers as c
join orders as o on c.id == o.customerId
zip trackingNumbers as t   # pair each customer/order row with a tracking number, positionally
select { c, o, t };
```

If the two collections differ in length, `zip` stops at the shorter one. When both are fixed-size arrays their lengths are known at compile time, so the result length is known as well - see [SIMD](./simd.md).

> **NOTE:** The special case of pairing each element with its index (what some languages call `enumerate`) needs no dedicated operation: zip against an index [range](./ranges.md), as in `from values as v zip 0.. as i`.
