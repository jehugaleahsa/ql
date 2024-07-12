# Combining collections
Multiple collections can be combined using the following operations:
* `append` - concatenate collections together
* `union` - retrieve all unique values from both collections
* `intersect` - retrieve values that appear in both collections
* `except` - retrieve all the values in the first collection that do not appear in the second collection

## Append
A collection can be concatenated (i.e., appended) to another collection using the `append` keyword:
```
let first = [1, 2, 3];
let second = [4, 5, 6];
let combined = first append second; # [1, 2, 3, 4, 5, 6]
```

## Union
The unique values that appear among collections can be retrieve using the `union` operation. No duplicates will be returned:
```
let first = [1, 2, 3];
let second = [2, 3, 4];
let combined = first union second; # [1, 2, 3, 4]
```

The two collections must have the same type. If the element type is not comparable, a `by` clause can be provided:
```
let Thing = type {
    id: i32
};
let first: Thing = [{ id: 1 }, { id: 2 }];
let second: Thing = [{ id: 2 }, { id: 3 }];
let combined = first union second by id; 
# -> [{ id: 1 }, { id: 2 }, { id: 3 }]
```

If multiple properties uniquely identify a value, the properties are separated by a comma.

## Intersect
Values that appear in both collections can be retrieved using `intersect`:
```
let first = [1, 2, 3];
let second = [2, 3, 4];
let combined = first intersect second; # [2, 3]
```

The two collections must have the same type. If the element type is not comparable, a `by` clause can be provided:
```
let first: Thing = [{ id: 1 }, { id: 2 }];
let second: Thing = [{ id: 2 }, { id: 3 }];
let combined = first intersect second by id;  # -> [{ id: 2 }]
```

If multiple properties uniquely identify a value, the properties are separated by a comma.

## Except
Values that only appear in the first collection, and not the second, can be retrieved using `except`:
```
let first = [1, 2, 3];
let second = [2, 3, 4];
let combined = first except second; # [1]
```

The two collections must have the same type. If the element type is not comparable, a `by` clause can be provided:
```
let first: Thing = [{ id: 1 }, { id: 2 }];
let second: Thing = [{ id: 2 }, { id: 3 }];
let combined = first except second by id;  # -> [{ id: 1 }]
```

If multiple properties uniquely identify a value, the properties are separated by a comma.