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

When decoding, reading either a literal or an escape sequence that does not correspond to a Unicode scalar value is an *error*. In particular, Unicode code points that are not scalar values are not allowed, even when they form valid surrogate pairs. Reading a string of more than `(2^64) - 1` scalar values is an *error* as well.

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

The binary encoding is a compact representation of SSDD values, for machine consumption. It uses techniques similar to [cbor](https://tools.ietf.org/html/rfc7049). There is no concept of whitespace (and thus comments) in the binary encoding.

Values are encoded as a single byte that tags the kind of value, followed by the actual data. The first bit of all tags is a one, so that binary and human-readable codes can immediately be recognized. This is followed by three bits indicating the kind of the value. The remaining four bits carry additional information about the following data.

Some values have multiple valid encodings. Implementations are strongly encouraged to always use the shortest possible encoding, but reading an overlong code is *not* a decoding error.

### Null

`null` is encoded as the tag `0b1_000_0000`, followed by no additional data.

### Booleans

`false` is encoded as the tag `0b1_000_0001`, followed by no additional data.

`true` is encoded as the tag `0b1_000_0010`, followed by no additional data.

### Ints

Ints are encoded as the tag `0b1_001_xxxx`, where the least significant four bits and the following bytes are determined as follows:

- for least significant bits less than `0b1100`, the bits themselves represent the encoded int (in the range from zero to eleven), no more bytes follow the tag
- for least significant bits `0b1100`, the tag is followed by a single byte, which encodes the int as two's complement (ranging from `-(2^7)` to `(2^7) - 1`)
- for least significant bits `0b1101`, the tag is followed by two bytes in big-endian order, which encode the int as two's complement (ranging from `-(2^15)` to `(2^15) - 1`)
- for least significant bits `0b1110`, the tag is followed by four bytes in big-endian order, which encode the int as two's complement (ranging from `-(2^31)` to `(2^31) - 1`)
- for least significant bits `0b1111`, the tag is followed by eight bytes in big-endian order, which encode the int as two's complement (ranging from `-(2^63)` to `(2^63) - 1`)

### Floats

Floats are encoded as the tag `0b1_000_0011`, followed by the eight bytes of the float (sign, exponent, fraction in that order). All NaNs must use the bytes `0xffffffffffffffffff` instead (which is one particular, arbitrary NaN representation).

### Chars

Chars are encoded as the tag `0b1_010_11xx`, where the least significant two bits and the following bytes are determined as follows:

- for least significant bits `0b00`, the tag is followed by a singly byte, which encodes the scalar value (less than `2^8`)
- for least significant bits `0b01`, the tag is followed by two bytes in big-endian order, which encode the scalar value (less than `2^16`)
- for least significant bits `0b10`, the tag is followed by four bytes in big-endian order, which encode the scalar value (less than `2^32`)
- for least significant bits `0b11`, the tag is followed by eight bytes in big-endian order, which encode the scalar value (less than `2^64`)

### Utf-8 Strings

Utf-8 strings are encoded as the tag `0b1_011_xxxx`, where the least significant four bits and the following bytes are determined as follows:

- for least significant bits less than `0b1100`, the bits themselves represent the length of the string, the tag is followed by that many bytes of utf-8
- for least significant bits `0b1100`, the tag is followed by a single byte, which encodes the length of the string, followed by that many bytes of utf-8
- for least significant bits `0b1101`, the tag is followed by two bytes in big-endian order, which encode the length of the string, followed by that many bytes of utf-8
- for least significant bits `0b1110`, the tag is followed by four bytes in big-endian order, which encode the length of the string, followed by that many bytes of utf-8
- for least significant bits `0b1111`, the tag is followed by eight bytes in big-endian order, which encode the length of the string, followed by that many bytes of utf-8

When decoding, reading any invalid utf-8 for the string is an *error*.

### Binary Strings

Binary strings are encoded just like utf-8 strings, with the following differences:

- they use the tags `0b1_100_xxxx`
- they can contain arbitrary bytes, not just utf-8

### Arrays

Arrays are encoded just like strings, with the following differences:

- they use the tags `0b1_101_xxxx`
- the length is not given in bytes but in items
- after the length, the code is followed by the binary encodings of all items in order

### Sets

Sets are encoded just like arrays, with the following differences:

- they use the tags `0b1_110_xxxx`
- when decoding, reading a duplicate item is an *error*
  - unlike puny humans hand-writing human-readable codes, machines are expected to not make this mistake

### Maps

Maps are encoded just like arrays, with the following differences:

- the use the tags `0b1_111_xxxx`
- the length is not given in items but in entries
- after the length, the code is followed by the encodings of all entries, where an entry is encoded as the binary encoding of its key followed by the binary encoding of its value
- when decoding, reading a duplicate key is an *error*
  - unlike puny humans hand-writing human-readable codes, machines are expected to not make this mistake

## Hybrid Input

The first bit of all valid human-readable encoded values is a zero, the first bit of all valid binary encoded values is a one. Programs that read SSDD values as input are strongly encouraged to allow both encodings, dispatching based on the first bit. This way, humans can conveniently supply input, but programs talking to each other can use the vastly more efficient binary format.

## Canonic Encoding

A canonic encoding is one where there is a one-to-one correspondence between values and codes. The SSDD canonic encoding is a subset of the binary encoding, obtained through the following restrictions:

- ints, chars and collections must use the shortest possible encodings for their length/size
- set items must be sorted ascendingly according to the order defined below
- map entries must be sorted ascendingly by their keys, according to the order defined below

The order on SSDD values used for sorting is defined as follows:

- `null` < booleans < integers < floats < chars < utf-8 strings < byte strings < arrays < sets < maps
- `false` < `true`
- integers are sorted by numeric value
- floats are sorted NaN < negative infinity < "regular floats" < positive infinity, where the "regular floats" are sorted by numeric value, and `-0.0` < `0.0`
- chars are sorted numerically
- utf-8 strings are sorted lexicographically
- byte strings are sorted lexicographically
- arrays are sorted lexicographically (the items themselves are sorted by the general SSDD value order though, *not* lexicographically)
- sets are sorted lexicographically (the items are compared smallest to largest, using the general SSDD value order)
- maps are sorted lexicographically (the entries are compared from smallest key to largest, entry comparision works by first comparing the keys, and if they are equal, comparing the values)
