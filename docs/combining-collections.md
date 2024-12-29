# Combining collections
Multiple collections can be combined using concatenation, `distinct`, and joins.

## Concatenate
A collection can be concatenateds to another collection using a copy operation, returning a new collection:
```
let first = [1, 2, 3];
let second = [4, 5, 6];
let combined = [...first, ...second]; # [1, 2, 3, 4, 5, 6]
```

## Distinct
The unique values that appear among collections can be retrieved using the `distinct` operation. No duplicates will be returned:
```
let first = [1, 2, 3];
let second = [2, 3, 4];
let combined =
    from [...first, ...second] as v
    distinct v; # [1, 2, 3, 4]
```

If the element type is not comparable, a `using` clause can be provided:
```
let Thing = type {
    id: i32
};
let first: Thing[] = [{ id: 1 }, { id: 2 }];
let second: Thing[] = [{ id: 2 }, { id: 3 }];
let combined = 
    from [...first, ...second] as t
    distinct t using(t.id)
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
    select first; # -> [{ id: 2 }]
```

## Set difference
Values that only appear in the first collection, and not the second, can be retrieved using a `left join`:
```
let first = [1, 2, 3];
let second = [2, 3, 4];
let combined = 
    from first as f
    left join second as s on f == s
    where s == null
    select f; # [1]
```

Here is the same example using a complex type:
```
let first: Thing[] = [{ id: 1 }, { id: 2 }];
let second: Thing[] = [{ id: 2 }, { id: 3 }];
let combined =
    from first as f
    left join second as s on f == s
    where s == null
    select f; # -> [{ id: 1 }]
```
