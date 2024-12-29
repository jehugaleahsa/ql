# Functions
Functions allow defining reusable logic. Here's an extremely simple function:
```
let greet = fn() {
    select "Hello, World!"
};
```

Function definitions start with the `fn` keyword. This is followed by a comma-separated argument list, wrapped in parentheses (`()`).

The last expression in a function is returned back to the caller. 

> **NOTE:** There is no trailing semicolon on the last expression in a function. 

In many cases, the return type of a function can be inferred. You can explicitly indicate the return type by listing it after the argument list and a colon (`:`).
```
let greet = fn(): String {
    select "Hello, world!"
};
```

Functions can take zero or more arguments. The type of arguments must be specified. Here's a basic example:
```
let sum = fn(x: i32, y: i32): i32 {
    select x + y
};

sum(1, 2) # 3
```

## Non-scalar thinking
One of the goals of QL is to get away from thinking about operations in terms of individual scalar values or objects. Instead, thought should go into making the same function work against a scalar and a vector at the same time. We can redefine the function above so it works against either:
```
let sum = fn(xs: i32*, ys: i32*): i32* {
    select x + y
};
```

That means `sum` can be called like this:
```
let result1 = sum(1, 2);           # 3
let result2 = sum([1, 2], [3, 4]); # [4, 6]
let result3 = sum([1, 2], 3);      # [4, 5]
let result4 = sum(null, null)      # null
```

It can be a bit difficult to think about how a function should behave when its arguments are collections. However, in doing so, a lot of classic errors is programming can be avoided.

As a second example, we can implement `count` like so:
```
let count = fn(x: i32*): usize* {
    let mut result = 0usize;
    let mut it = x.iterator();
    while let Some(next) = it.next() {
        result += 1usize;
    }
    result
}
```

Here, we didn't even both assigning the first element to a variable, using `_` to indicate the variable is ignored.

Now let's try implementing a similar function to compute an average:
```
let sumAndCount = fn(values: i32*): (i64, usize)* {
    let mut sum = 0i64;
    let mut count = 0usize;
    let mut it = values.iterator();
    while let Some(it) = it.next() {
        sum += it.value() as i64;
        count += 1usize;
    }
    (sum, count)
};
let avg = fn(values: i32*): f64* {
    if values == null {
        return select null
    }
    let (sum, count) = sumAndCount(values);
    select total as f64 / count as f64
};
```

Again, we allow for `null` to be passed to `avg`. One classic gotcha in programming is that dividing by zero results in an error. To avoid that, we check for `null` and return `null` in that case.

> **NOTE:** Here we use an explicit `return` to force and immediate exit if we enter the `if`. For the rest of the method, the compiler knows `values` cannot be `null`. In this situation, both `sum` and `count` handle `null`, so that's not that important.

Now image we have a different `count`-like method:
```
let countOrNull = fn(x: i32*): i32?* {
    if x == null {
        return select null
    }
    let _, ...rest = x;
    let restCount = countOrNull(rest) ?? 0;
    select restCount + 1
};
```

With this special method, we can reimplement `avg` with less mental overhead:
```
let avg2 = fn(x: i32?): i32? {
    select sum(x) / countOrNull(x)
};
```

Recall that when performing arithmetic involving `null`, the result is always `null`. If the collection is `null` (i.e., empty), then `countOrNull` will return `null` and the entire expression will result in `null`.

> **NOTE:** This new `avg2` might be less efficient than `avg` because it evaluates `sum` unnecessarily. The compiler might be able to optimize this away, though.

## Varargs
With our current definitions, calling `sum`, `count` or `avg` with an empty collection (i.e., `null`) will look like this:
```
sum([])
sum(null)
```

What if we also want to support `sum()`? Furthermore, what if we wanted to support `sum(1, 2, 3)`? QL doesn't support function overloading, so we can't just have two `sum` functions.

We can slightly modify our function signatures to accomplish this:
```
let sum = fn(...x: i32*): i32* {
    # ... same as before
};
```

When we add `...` before the last parameter, it means the function can be called with an arbitrary number of parameters. The compiler will simply wrap all such parameters in a collection automatically.

> **NOTE:** All vararg parameters are implicitly null-able, since passing no arguments is allowed.

Now we can call `sum` like this:
```
sum([])                   # null
sum(null)                 # null
sum()                     # null
sum([1, 2, 3])            # 6
sum(1, 2, 3)              # 6
sum([1, 2, 3], [4, 5, 6]) # [5, 7, 9]
```