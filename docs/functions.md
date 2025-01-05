# Functions
Functions allow defining reusable logic. Here's an extremely simple function:
```
let greet = fn() {
    "Hello, World!"
};
```

Function definitions start with the `fn` keyword. This is followed by a comma-separated argument list, wrapped in parentheses (`()`).

The last expression in a function is returned back to the caller. 

> **NOTE:** There is no trailing semicolon on the last expression in a function. 

In many cases, the return type of a function can be inferred. You can explicitly indicate the return type by listing it after the argument list and a colon (`:`).
```
let greet = fn(): String {
    "Hello, world!"
};
```

Functions can take zero or more arguments. The type of arguments must be specified. Here's a basic example:
```
let sum = fn(x: i32, y: i32): i32 {
    x + y
};

sum(1, 2) # 3
```

