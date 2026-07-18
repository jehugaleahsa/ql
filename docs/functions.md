# Functions
Functions allow defining reusable logic. A function has a parameter list and a body, and there are two ways to write the body.

## Block-bodied functions
A function whose body is a block (`{ ... }`) runs the block and returns the value of its last expression:
```
let greet = fn() {
    "Hello, World!"
};
```

Function definitions start with the `fn` keyword, followed by a comma-separated parameter list in parentheses (`()`).

The last expression in the block is returned to the caller.

> **NOTE:** There is no trailing semicolon on the last expression in a block.

The return type can often be inferred. To state it explicitly, list it after the parameter list and a colon (`:`):
```
let greet = fn(): String {
    "Hello, world!"
};
```

Parameters list their types after a colon. Here is a function with two parameters and an explicit return type:
```
let sum = fn(x: i32, y: i32): i32 {
    x + y
};

sum(1, 2) # 3
```

## Expression-bodied functions
When a body is a single expression, the block and its braces are just noise. An *expression-bodied* function replaces the block with `=>` followed by one expression, whose value is returned:
```
let sum = fn(x: i32, y: i32): i32 => x + y;

sum(1, 2) # 3
```

The `=>` form and the `{ ... }` form are the same construct - a function - differing only in whether the body is a single expression or a block. Use `=>` for one-liners and `{ ... }` when the body needs intermediate steps.

> **NOTE:** QL has no separate "lambda" syntax. A function passed inline as an argument is written exactly like any other function - most often the expression-bodied form.

## Inferred parameter and return types
The types on `sum` above are required because nothing else tells the compiler what `x` and `y` are. But when a function is written *at the point where it is passed as an argument*, the expected type is already known from the parameter it fills, so the parameter and return types can be omitted:
```
let hasher = Hash<Customer>.using(fn(c) => c.id);
```

Here `using` expects a function from a `Customer` to some key, so `c` is known to be a `Customer` and the return type is whatever `c.id` is - none of it needs to be written out. Spelled in full, the same argument would be `fn(c: Customer): i32 => c.id`.

This is the closest QL comes to the terse lambdas of other languages: an expression-bodied function whose types are inferred from the call. And because a [query](./queries.md) already expresses filtering, mapping, and reduction directly, inline functions like this appear mainly when passing behavior to a library - a comparator, a custom [aggregator](./aggregates.md), and the like - rather than inside queries themselves.
