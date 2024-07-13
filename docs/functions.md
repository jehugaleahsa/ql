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
let sum = fn(x: i23, y: i32): i32 {
    select x + y
};
```

One thing that might be surprising is that QL doesn't have the concept of [scalar values](./vectors.md). That means `sum` can be called like this:
```
let result1 = sum(1, 2);           # 3
let result2 = sum([1, 2], [3, 4]); # [4, 6]
let result3 = sum([1, 2], 3);      # [4]
```

It can be a bit difficult to think about how a function should behave when its arguments are collections. However, in doing so, a lot of classic errors is programming can be avoided.

This becomes clear when you realize `null` is just an empty collection in QL. If functions are defined to work on collections, receiving a `null` is usually not a failure condition - it usually results in nothing happening.

Another way to implement `sum` is to do something like this:
```
let sum = fn(x: i32?): i32 {
    if x == null { 
        select 0 
    } else {
        let x, ...rest = x;
        select x + sum(rest)
    }
};
```

Notice there's just one `i32?` argument. The `?` is necessary because we allow the given collection to be empty. 

If the collection is not empty (a.k.a., not `null`), we strip off the first element and add it to rest of the elements passed to `sum`, *recursively*. Eventually we'll run out of elements and the recursive calls will stop.

As a second example, we can implement `count` in a similar way:
```
let count = fn(x: i32?): i32 {
    if x == null {
        select 0
    } else {
        let _, ...rest = x;
        return count(rest) + 1
    }
}
```

Here, we didn't even both assigning the first element to a variable, using `_` to indicate the variable is ignored.

Now let's try implementing a similar function to compute an average:
```
let avg = fn(values: i32?): f64? {
    if values == null {
        select null
    } else {
        let total = sum(values) as f64;
        let count = count(values) as f64;
        select total / count
    }
};
```

Again, we allow for `null` to be passed to `avg`. One classic gotcha in programming is that dividing by zero results in an error. To avoid that, we check for `null` and return `null` in that case.

> Within the `else` block, the compiler knows `values` cannot be `null`. In this situation, both `sum` and `count` handle `null`, so that's not that important.

> In other languages, at runtime, `sum` and `count` will be run one after the other while executing `avg`. QL makes no such guarantee. QL may be smart and replace these recursive implementations with simple loops in machine code. Furthermore, QL may run `sum` and `count` in parallel.

Now image we have a different `count`-like method:
```
let countOrNull = fn(x: i32?): i32? {
    if x == null {
        select null
    } else {
        let _, ...rest = x;
        let restCount = countOrNull(rest) ?? 0;
        select restCount + 1
    }
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
let sum = fn(...x: i32): i32 {
    # ... same as before
};
```

When we add `...` before the last parameter, it means the function can be called with an arbitrary number of parameters. The compiler will simply wrap all such parameters in a collection automatically.

> **NOTE:** All vararg parameters are implicitly null-able, therefore a trailing `?` is not allowed.

Now we can call `sum` like this:
```
sum([])
sum(null)
sum()
sum([1, 2, 3])
sum(1, 2, 3)
```