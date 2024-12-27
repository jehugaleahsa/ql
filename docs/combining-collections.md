# Combining collections
Multiple collections can be combined using the following operations:
* `append` - concatenate collections together
* `union` - retrieve the unique values from both collections
* `intersect` - retrieve values that only appear in both collections
* `except` - retrieve all the values in the first collection that do not appear in the second collection

## Append
A collection can be concatenated (i.e., appended) to another collection using the `append` keyword:
```
let first = [1, 2, 3];
let second = [4, 5, 6];
let combined = append first second; # [1, 2, 3, 4, 5, 6]
```

> The same effect can be achieved more succinctly using `[...first, ...second]`.

## Union
The unique values that appear among collections can be retrieve using the `union` operation. No duplicates will be returned:
```
let first = [1, 2, 3];
let second = [2, 3, 4];
let combined = union first second; # [1, 2, 3, 4]
```

The two collections must have the same type. If the element type is not comparable, a `using` clause can be provided:
```
let Thing = type {
    id: i32
};
let first: Thing = [{ id: 1 }, { id: 2 }];
let second: Thing = [{ id: 2 }, { id: 3 }];
let combined = union first second using { id }; 
# -> [{ id: 1 }, { id: 2 }, { id: 3 }]
```

## Intersect
Values that only appear in both collections can be retrieved using `intersect`:
```
let first = [1, 2, 3];
let second = [2, 3, 4];
let combined = intersect first second; # [2, 3]
```

The two collections must have the same type. If the element type is not comparable, a `using` clause can be provided:
```
let first: Thing = [{ id: 1 }, { id: 2 }];
let second: Thing = [{ id: 2 }, { id: 3 }];
let combined = intersection first second using { id };  # -> [{ id: 2 }]
```

## Except
Values that only appear in the first collection, and not the second, can be retrieved using `except`:
```
let first = [1, 2, 3];
let second = [2, 3, 4];
let combined = except first second; # [1]
```

The two collections must have the same type. If the element type is not comparable, a `using` clause can be provided:
```
let first: Thing = [{ id: 1 }, { id: 2 }];
let second: Thing = [{ id: 2 }, { id: 3 }];
let combined = except first second using { id };  # -> [{ id: 1 }]
```

