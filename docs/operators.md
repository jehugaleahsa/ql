# Operators
QL allows computing arithmetic on numeric primitive values.

## Binary math operators

* `+` - Adds two values together, such as `3 + 4` equaling `7`.
* `-` - Subtracts two values, such as `4 - 3` equaling `1`.
* `*` - Multiplies two values, such as `3 * 4` equaling `12`.
* `/` - Floating point division, such as `5 / 2` equaling `2.5`.
* `//` - Truncated integer division, such as `5 // 2` equaling `2`.
* `**` - Exponentiation, such as `10 ** 2` equaling `100`.
* `%` - Modulus, such as `5 % 2` equaling `1`.

## Unary math operators

* `+` - Indicates that a value is positive. This is a no-op.
* `-` - Negates a value, such as `-1`.

## Bitwise operators

* `~` - Bitwise complement
* `&` - Bitwise and
* `|` - Bitwise or
* `^` - Bitwise xor
* `>>` - Bitwise shift right
* `>>>` - Bitwise shift right, retaining sign.
* `<<` - Bitwise shift left

## Boolean operators

* `!` (`not`) - Negation
* `&&` (`and`) - Logical and (short-circuiting)
* `||` (`or`) - Logical or (short-circuiting)
* `&` - and (not short-circuiting)
* `|` - or (not short-circuiting)

## Comparison operators

* `==` - equality
* `!=` - inequality
* `<` - less than
* `<=` - less than or equal to
* `>` - greater than
* `>=` - greater than or equal to

## Assignment operators

* `=` - assignment, such as `let x = 123;`

## Optionals and pattern tests

* `??` - coalesce: replace `None` with a fallback, such as `None ?? 123` equaling `123`
* `?.` - optional member access, such as `x?.y` (yields `None` when `x` is `None`)
* `?[]` - optional index access, such as `x?[0]`
* `!` - assert the `Some` variant, such as `x!.y` (fails if `x` is `None`)
* `x is P` - true if `x` matches pattern `P`, binding any variables in `P` for the rest of the scope, such as `x is Some(v)`

## Matrix and array operators

* `@` - Matrix multiplication

## Range operators

* `..` - Define a numeric range, with an excluded end value, with an optional `:` for controlling the increment.
* `..=` - Define a numeric range, with an included end value, with an optional `:` for controlling the increment.

> See [Ranges](./ranges.md) for how these operators are used to generate sequences, slice collections, and define window frames.

## Precedence

Parentheses can be used to control precedence, such that `(2 + 3) * 5 == 25`. Normally, `*` has a higher precedence than `+`, so `2 + 3 * 5 == 17` instead. Precedence in QL is as follows, from highest to lowest:

* `()`
* `x.y`
* `f(x)`
* `x[y]`
* `x?.y`
* `x?[y]`
* `x!`
* `checked`
* `unchecked`
* `x ** y` (right-associative)
* `+x`
* `-x`
* `!x`
* `~x`
* `x * y`
* `x @ y`
* `x / y`
* `x // y`
* `x % y`
* `x + y`
* `x - y`
* `x << y`
* `x >> y`
* `x >>> y`
* `x & y`
* `x ^ y`
* `x | y`
* `x < y`, `x > y`, `x <= y`, `x >= y`, `x == y`, `x != y` (non-associative; comparisons cannot be chained)
* `x is P`
* `x && y`
* `x || y`
* `x ?? y`
* `x .. y`
* `x ..= y`
