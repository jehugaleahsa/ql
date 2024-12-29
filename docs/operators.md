# Operators
QL allows computing arithmetic on numeric primitive values.

## Binary math operators

* `+` - Adds two values together, such as `3 + 4` equaling `7`.
* `-` - Subtracts two values, such as `4 - 3` equaling `1`.
* `*` - Multiplies two values, such as `3 * 4` equaling `12`.
* `/` - Floating point division, such as `5 / 2` equaling `2.5`.
* `//` - Truncated integer division, such as `5 / 2` equaling `2`.
* `**` - Exponentiation, such as `10 ** 2` equaling `100`.
* `%` - Modulus, such as `5 / 2` equaling `1`.

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

## Null checking

* `??` - null coalesce, such as `null ?? 123` equaling 123
* `?.` - null-safe member accessor, such as `x?.y`
* `?[]` - null-safe index accessor, such as `x?[0]`

## Matrix and array operators

* `@` - Matrix multiplication

## Range operators

* `..` - Define a numeric range, with an excluded end value, with an optional `:` for controlling the increment.
* `..=` - Define a numeric range, with an included end value, with an optional `:` for controlling the increment.

## Lambdas

* `=>` - define an inline function

## Precedence

Parentheses can be used to control precedence, such that `(2 + 3) * 5 == 25`. Normally, `*` has a higher precedence than `+`, so `2 + 3 * 5 == 17` instead. Precedence in QL is as follows, from highest to lowest:

* `()`
* `x.y`
* `x?.y`
* `x?[y]`
* `checked`
* `unchecked`
* `x ** y`
* `+x`
* `-x`
* `!x`
* `~x`
* `^x`
* `x..y`
* `x..=y`
* `x * y`
* `x / y`
* `x // y`
* `x % y`
* `x + y`
* `x - y`
* `x << y`
* `x >> y`
* `x >>> y`
* `x < y`
* `x > y`
* `x <= y`
* `x >= y`
* `x == y`
* `x != y`
* `x & y`
* `x ^ y`
* `x | y`
* `x && y`
* `x || y`
* `x ?? y`
* `=>`
