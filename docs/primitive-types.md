# Primitive types
QL supports the following primitive types:
* `i8` - An 8-bit signed integer
* `i16` - A 16-bit signed integer
* `i32` - A 32-bit signed integer
* `i64` - A 64-bit signed integer
* `i128` - A 128-bit signed integer (where available)
* `isize` - A signed integer the size of a pointer on the current hardware architecture
* `u8` - An 8-bit unsigned integer
* `u16` - A 16-bit unsigned integer
* `u32` - A 32-bit unsigned integer
* `u64` - A 64-bit unsigned integer
* `u128` - A 128-bit unsigned integer (where available)
* `usize` - An unsigned integer the size of a pointer on the current hardware architecture
* `f32` - A 32-bit floating point ([IEEE 754](https://en.wikipedia.org/wiki/IEEE_754))
* `f64` - A 64-bit floating point ([IEEE 754](https://en.wikipedia.org/wiki/IEEE_754))
* `f128` - A 128-bit floating point ([IEEE 754](https://en.wikipedia.org/wiki/IEEE_754)) (where available)
* `char` - A 32-bit Unicode character (code point)
* `bool` - `true` or `false`

## Literals
When an integer literal appears by itself, such as:
```
let x = 123;
```
the type defaults to `i32`.

When a floating point literal appears by itself, such as:
```
let x = 3.14;
```
the type defaults to `f64`.

Floating point literals can include exponents (scientific notation):
```
let x = 2.8e16;
```

The exponent can be prefixed using `e` or `E`, and can be positive or negative.

Floating point values also support additional literals:
* `NaN` - A special value representing a non-numeric value
* `Infinity` - Positive infinity

Negative infinity is represented as `-Infinity`.

The type of an integer or floating point literal can be controlled by placing the type directly after the literal (with no space between):
```
let x = 123i64
```

A `char` is a single Unicode code point between single quotes: `'a'`.

It can also be specified using a unicode escape sequence: `'\u{7FFF}'`.

The single quote `char` (`'`) can be represented as `'\''`.

The literals `true` and `false` represent the two possible values for `bool`.

### Underscore separators
For numeric literals, underscores can be used as separators, such as `1_234`.

> **Note:** There's no restriction on the placement of underscores. They can appear at the beginning of a numeric literal, anywhere in the middle, or at the end. Multiple underscores can also appear next to each other.

## Text and strings
While a `char` is always 32-bits (i.e., 4 bytes), `String` values are comprised of characters that may be anywhere between 8- and 32-bits, encoded in UTF-8.

A `String` literal is enclosed in double quotes (`"`):
```
let x = "Hello, world!";
```

> Note: `String`s are not primitive types, but the syntax is provided here since they have literal representations. 

### Escape sequences
The following escape sequences can appear within a `char` or `String`:
* `\n` - Newline
* `\r` - Carriage return
* `\t` - Tab
* `\\` - Backslash
* `\0` - Nul
* `\"` - Double quote (just `'"'` for `char`)
* `\'` - Single quote (just `"'"` for `String`)
* `\u{7FFF}` - 24-bit Unicode character code (up to 6 digits)

### Raw and multiline strings
A `String` literal can be prefixed with an `r` (meaning "raw"):
```
let x = r"Hello, world!";
```

Within a raw `String`, escape sequences are ignored. This means backslashes are treated literally and do not need escaped:
```
let x = r"This is \ fine.";
```

Raw `String` literals can be broken across multiple lines:
```
let x = r"This
is
also
fine";
```

This is as if `\n` was inserted into the `String`.

If the `String` contains quotes, an arbitrary number of wrapping quotes can be used to indicate the start and end of the `String`:
```
let x = r"""This
"string"
is also "fine"
""";
```

The string will continue until a matching number of quotes are found.

## Conversion
Implicit conversions are disallowed. Conversions must always be explicit. 

The following converts an `f64` to an `i32`:
```
let f = 3.14;
let i = f as i32; # 3 - integer truncation
```

Converting a `bool` to an integer always results in `0` or `1`:
```
let x = true as i32;  # 1
let y = false as i32; # 0
```

Going the other way, `0` converts to `false` and all other values convert to `true`, including negative values.

A `String` can be converted to a numeric value using a `tryParse` operation:
```
let i = i32::tryParse("123");
```

The type of `i` will be `i32?`, meaning a "nullable" integer. Learn more about `null` in the section about [vectors](./collections.md).

### Overflow/Underflow
By default, a program will panic, leading to termination, if a conversion results in overflow or underflow (the representation cannot hold the previous value). An operation can be wrapped in an `unchecked` expression to prevent this from happening:
```
let result = unchecked {
    123_345_678 * 2_000
};
```

If the operation overflows, the result wraps around, which will mean something different depending on whether the value is signed/unsigned.

### Unicode checks
An integer value can be converted to a `char`; however, the compiler verifies the value represents a valid Unicode code point. At runtime, invalid code points will result in the program panicking. An operation can be wrapped in an `unchecked` expression to prevent this from happening.