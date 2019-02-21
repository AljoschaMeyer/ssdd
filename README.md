# SSDD

> Some Self-Describing Data

SSDD is a self-describing free-form data format, i.e. yet another alternative to json and friends. It has a handful of distinguishing features:

- precise specification: Life is too short to troubleshoot divergent implementations because of unspecified corner cases.
- clear distinction between the logical data model and actual encodings
- a human-readable encoding, a binary encoding, and a canonic subset of the binary encoding
- byte strings: Supporting utf8 strings (i.e. a selected subset of byte strings) but no general byte strings is just plain *weird*.
- maps with arbitrary keys (strings aren't special)
- sets: Implicitly representing sets as lists creates ambiguity.

## Logical Data Model

An SSDD value is one of the following:

- `null`: A value that carries [no further information](https://en.wikipedia.org/wiki/Unit_type).
- `boolean`: Either `true` or `false`.
- `int`: An integer between `-(2^63)` and `(2^63) - 1` (inclusive).
- `float`: An [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754) double precision float, except that there is only one `NaN` value.
- `char`: A [Unicode scalar value](http://www.unicode.org/glossary/#unicode_scalar_value) (*not* a [Unicode code point](http://www.unicode.org/glossary/#code_point)).
- `utf8-string`: An ordered sequence of [Unicode scalar values](http://www.unicode.org/glossary/#unicode_scalar_value) whose [utf-8](https://en.wikipedia.org/wiki/UTF-8) encoding takes up no more than `(2^64) - 1` bytes.
- `byte string`: An ordered sequence of up to `(2^64) - 1` bytes.
- `array`: An ordered sequence of up to `(2^64) - 1` SSDD values.
- `set`: An unordered collection of up to `(2^64) - 1` pairwise distinct SSDD values.
- `map`: An unordered collection of pairs (`entries`) of SSDD values, where the first values of all entries are pairwise distinct. The first value of an entry is called a `key`, the second value called a `value`.

The choice of values a self-describing format provides is to some degree arbitrary. For SSDD, decisions between different approaches have often been decided based on machine-friendlyness. Fixed-width integers are much easier to handle than arbitrary precision integers. Floats are easier than true rationals. Maximum collection and string sizes enable implementations to precisely follow the spec rather than introducing arbitrary limits that would inevitably differ between distinct implementations.

## Human-Readable Encoding

The human-readable encoding is a subset of utf-8 that represents SSDD values. It is intended to be read, created and edited by humans directly. The recommended file extension is `.ssdd`.

The bytes `0x0a` (newline) and `0x20` (space) are considered whitespace. A sequence of valid utf-8 beginning with `0x23` (`#`) and ending with either `0x0a` (newline) or the end of input is considered whitespace (called a (line) comment). A valid code consists of any amount of whitespace, followed by the encoding of an ssdd value as described next, followed by any amount of whitespace.

### Null

`null` is encoded as the utf-8 string `null` (`[0x6e, 0x75, 0x6c, 0x6c]`).

### Booleans

`true` is encoded as the utf-8 string `true` (`[0x74, 0x72, 0x75, 0x65]`).

`false` is encoded as the utf-8 string `false` (`[0x66, 0x61, 0x6c, 0x73, 0x65]`).

### Ints

A negative int is encoded as a `-` (`0x2d`) directly followed by the encoding of the positive integer with the same absolute value (this definition is *not* tied to any particular two's complement representation, so there is no overflow issue with `- 2^63`).

A positive int can be encoded in one of two ways: Either as a sequence of ASCII decimal digits (`0x30` to `0x39`), or as the utf-8 string `0x` (`[0x30, 0x78]`) followed by a sequence of ASCII hexadecimal digits (`0x30` to `0x39`, `0x41` to `0x46`, and `0x61` to `0x66`).

When decoding, reading an integer outside the allowed range (between `-(2^63)` and `(2^63) - 1` inclusive) is an *error*. Those are not valid human-readbly encoded ssdd values.

### Floats

Not-a-number is encoded as the utf-8 string `NaN` (`[0x4e, 0x61, 0x4e]`). Applications should encode all NaNs this way, even if they have different machine representations. If the exact representation needs to be preserved, use some sort of compound value, or consider switching to a different data format. When decoding `NaN`, it is encouraged to represent it internally as the float with the bytes `0xffffffffffffffffff`, since this is how the binary encoding serializes NaNs. This recommendation is however purely optional, from the point of view of ssdd, machine representations don't exist.

Positive infinity is encoded as the utf-8 string `Inf` (`[0x49, 0x6e, 0x0x66]`).

A (not NaN) negative float (including negative zero and negative infinity) is encoded as a `-` (`0x2d`) directly followed by the encoding of the same float but with the sign bit flipped.

Non-negative floats are encoded as a decimal representation: one or more decimal ASCII digits, followed by a `.` (`0x2e`), followed by one or more decimal ASCII digits, optionally followed by an exponent: `e` (`0x65`) or `E` (`0x45`), an optional sign (`+` (`0x2b`) or `-` (`0x2d`)), followed by one or more decimal ASCII digits.

When decoding a float, the input may not be representable exactly. In these cases, use ["round to nearest, ties to even"](https://en.wikipedia.org/wiki/IEEE_754#Roundings_to_nearest) rounding mode. As a consequence, floats do not need to be encoded as their exact decimal representation, implementations may instead use a smaller decimal representation that still rounds to the correct float. Note that the rounding may result in an infinity, as a corollary, it is possible to encode infinities as large decimal numbers (e.g. `9999.9e999999`) rather than `Inf`.

### Chars

A char can be encoded either literally or through an escape sequence. The literal encoding can be used for all chars other than `'` (`0x27`) and `\` (`0x5c`) and consists of a `'` (`0x27`), followed by the utf-8 encoding of the Unicode scalar value, followed by another `'` (`0x27`). The escape sequence encoding consists of a `'` (`0x27`), followed by an escape sequence, followed by another `'` (`0x27`). The following escape sequences are defined:

- `\'` for the char `'` (`0x27`)
- `\\` for the char `\` (`0x5c`)
- `\t` for the char `horizontal tab` (`0x09`)
- `\n` for the char `new line` (`0x0a`)
- `\0` for the char `null` (`0x00`)
- `\{DIGITS}`, where `DIGITS` is the ASCII decimal representation of any scalar value. `DIGITS` must consist of one to six characters.

When decoding, reading either a literal or an escape sequence that does not correspond to a Unicode scalar value is an *error*.

### Utf-8 Strings

A string is encoded as a `"` (`0x22`), followed by up to `(2^64) - 1` characters (scalar values), followed by another `"` (`0x22`).

Each character can either be encoded literally or through an escape sequence. The literal encoding cn be used for all scalar values other than `"` (`0x22`) and `\` (`0x5c`) and consists of the utf-8 encoding of the scalar value. Alternatively, any of the following escape sequences can be used:

- `\"` for the character `"` (`0x22`)
- `\\` for the character `\` (`0x5c`)
- `\t` for the character `horizontal tab` (`0x09`)
- `\n` for the character `new line` (`0x0a`)
- `\0` for the character `null` (`0x00`)
- `\{DIGITS}`, where `DIGITS` is the ASCII decimal representation of any scalar value. `DIGITS` must consist of one to six characters.

When decoding, reading either a literal or an escape sequence that does not correspond to a Unicode scalar value is an *error*. In particular, Unicode code points that are not scalar values are not allowed, even when they form valid surrogate pairs.

### Byte Strings

A binary string is encoded as a comma-separated (`,`, `0x2c`) list of the bytes, enclosed between `b[` ([`0x62`, `0x5b`]) and `]` (`0x5d`). The bytes are encoded just like `ints`. An optional trailing comma before the closing bracket is allowed. Any amount of whitespace can be placed between brackets, contained values, and commas.

When decoding, reading a byte string of more than `(2^64) - 1` contained byte encodings is an *error*.

### Arrays

An array is encoded as a comma-separated (`,`, `0x2c`) list of the encodings of the contained values, enclosed between brackets `[` (`0x5b`) and `]` (`0x5d`). An optional trailing comma before the closing bracket is allowed. Any amount of whitespace can be placed between brackets, contained values, and commas.

When decoding, reading an array of more than `(2^64) - 1` contained values is an *error*.

### Sets

A set is encoded as a comma-separated (`,`, `0x2c`) list of the encodings of the contained values, enclosed between `@{` ([`0x40`, `0x7b`]) and `}` (`0x7d`). An optional trailing comma before the closing brace is allowed. Any amount of whitespace can be placed between braces, contained values, and commas.

When decoding, duplicate values are allowed and are simply discarded (the logical model does *not* allow multisets). Reading a set of more than `(2^64) - 1` distinct contained values is an *error*.

### Maps

A map is encoded as a comma-separated (`,`, `0x2c`) list of the contained pairs (see below), enclosed between braces `{` (`0x7b`) and `}` (`0x7d`). An optional trailing comma before the closing brace is allowed. Any amount of whitespace can be placed between braces, contained values, and commas.

An entry is encoded as the encoding of the key, followed by any amount of whitespace, followed by a `:` (`0x3a`) followed by any amount of whitespace, followed by the encoding of the value.

When decoding multiple entries with identical keys, the later entry replaces the previous entry in the map. Reading a map of more than `(2^64) - 1` contained entries with distinct keys is an *error*.

## Binary Encoding

TODO

## Hybrid Input

The first bit of all valid human-readable encoded values is a zero, the first bit of all valid binary encoded values is a one. Programs that read SSDD values as input are strongly encouraged to allow both encodings, dispatching based on the first bit. This way, humans can conveniently supply input, but programs talking to each other can use the vastly more efficient binary format.

## Canonic Encoding

TODO