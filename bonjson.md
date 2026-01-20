BONJSON: Binary Object Notation for JSON
========================================

BONJSON is a _hardened_, _lightning-fast_ and _efficient_, **1:1 compatible** binary drop-in replacement for [JSON](#json-standards).

It's **35 times** faster to process than [JSON](#json-standards) due to the format being designed to take advantage of intrinsics present in modern day processors.

It's also **far safer** than JSON:

* **Safe** against key collision attacks
* **Safe** against injection attacks
* **Safe** against truncation attacks
* **Safe** against numeric range attacks


-------------------------------------------------------------------------------

Contents
--------

- [BONJSON: Binary Object Notation for JSON](#bonjson-binary-object-notation-for-json)
  - [Contents](#contents)
  - [Terms and Conventions](#terms-and-conventions)
  - [Compliance Levels](#compliance-levels)
    - [Basic Compliance](#basic-compliance)
    - [Secure Compliance](#secure-compliance)
    - [Compliance Documentation](#compliance-documentation)
  - [Types](#types)
  - [Structure](#structure)
  - [Encoding](#encoding)
    - [Type Codes](#type-codes)
  - [Strings](#strings)
    - [Short String](#short-string)
    - [Long String](#long-string)
  - [Numbers](#numbers)
    - [Small Integer](#small-integer)
    - [Integer](#integer)
    - [16-bit Float](#16-bit-float)
    - [32-bit Float](#32-bit-float)
    - [64-bit Float](#64-bit-float)
    - [Big Number](#big-number)
      - [Special Big Number Encodings](#special-big-number-encodings)
  - [Containers](#containers)
    - [Array](#array)
    - [Object](#object)
  - [Boolean](#boolean)
  - [Null](#null)
  - [Length Field](#length-field)
    - [Length Field Header](#length-field-header)
    - [Length Field Payload Format](#length-field-payload-format)
    - [Length Payload Encoding Process](#length-payload-encoding-process)
    - [Length Payload Decoding Process](#length-payload-decoding-process)
    - [Chunking](#chunking)
  - [Interoperability Considerations](#interoperability-considerations)
    - [Value Ranges](#value-ranges)
  - [Security Rules](#security-rules)
    - [Resource Limits](#resource-limits)
    - [Long Field Lengths](#long-field-lengths)
    - [Chunking Restrictions](#chunking-restrictions)
    - [Invalid UTF-8](#invalid-utf-8)
    - [NUL Codepoint Restriction](#nul-codepoint-restriction)
    - [Unicode Normalization](#unicode-normalization)
    - [Values incompatible with JSON](#values-incompatible-with-json)
    - [Out-of-range Values](#out-of-range-values)
    - [Duplicate object keys](#duplicate-object-keys)
    - [Trailing Data](#trailing-data)
  - [Convenience Considerations](#convenience-considerations)
  - [Filename Extensions and MIME Type](#filename-extensions-and-mime-type)
  - [Full Example](#full-example)
  - [Formal BONJSON Grammar](#formal-bonjson-grammar)
  - [JSON Standards](#json-standards)
  - [License](#license)


Terms and Conventions
---------------------

**The following bolded, capitalized terms have specific meanings in this document**:

| Term             | Meaning                                                                                          |
| ---------------- | ------------------------------------------------------------------------------------------------ |
| **MUST (NOT)**   | If this directive is not adhered to, the document or implementation is invalid.                  |
| **SHOULD (NOT)** | Every effort should be made to follow this directive, but it's still conformant if not followed. |
| **MAY (NOT)**    | It is up to the implementation to decide whether to do something or not.                         |
| **CAN**          | Refers to a possibility which **MUST** be accommodated by the implementation.                    |
| **CANNOT**       | Refers to a situation which **MUST NOT** be allowed by the implementation.                       |

**Additional Conventions**:

 * All bit diagrams in this document are drawn with the most significant bit (MSB) on the left and the least significant bit (LSB) on the right.

-----------------------------------------------------------------------------------------------------------------------



Compliance Levels
-----------------

BONJSON defines two compliance levels to accommodate different implementation environments:

### Basic Compliance

Basic compliance is suitable for resource-constrained environments (embedded systems, WASM, implementations without Unicode libraries) where full Unicode normalization is not feasible.

A **basic** decoder:

* **MUST** validate UTF-8 encoding (reject [invalid UTF-8](#invalid-utf-8))
* **MUST** perform [duplicate key](#duplicate-object-keys) detection using byte-for-byte comparison of the raw (non-normalized) UTF-8 strings
* **MUST** clearly document that it does not perform Unicode normalization

A **basic** encoder:

* **SHOULD** produce [NFC-normalized](https://unicode.org/reports/tr15/#Norm_Forms) strings when possible

**Warning**: Basic compliance is vulnerable to key collision attacks on platforms that auto-normalize strings (such as Swift and Objective-C). See [Unicode Normalization](#unicode-normalization) for details.


### Secure Compliance

Secure compliance provides protection against Unicode-based key collision attacks and is **RECOMMENDED** for all implementations that can support it.

A **secure** decoder:

* **MUST** validate UTF-8 encoding (reject [invalid UTF-8](#invalid-utf-8))
* **MUST** normalize strings to [NFC](https://unicode.org/reports/tr15/#Norm_Forms) before performing [duplicate key](#duplicate-object-keys) detection
* **MUST** return NFC-normalized strings to the application

A **secure** encoder:

* **SHOULD** produce NFC-normalized strings


### Compliance Documentation

All implementations **MUST** document which compliance level they support. Implementations **MAY** offer both levels as a configuration option.



Types
-----

BONJSON has the exact same types as [JSON](#json-standards) (and no more):

 * [String](#strings)
 * [Number](#numbers)
 * [Array](#array)
 * [Object](#object)
 * [Boolean](#boolean)
 * [Null](#null)



Structure
---------

BONJSON follows the same structural rules as [JSON](#json-standards), as illustrated in the following railroad diagrams:

**Document**:

    ──[value]──>

**Value**:

    ──┬─>─[string]──┬─>
      ├─>─[number]──┤
      ├─>─[object]──┤
      ├─>─[array]───┤
      ├─>─[boolean]─┤
      ╰─>─[null]────╯

**Array**:

    ──[array]──┬──[chunk]─┬─>──────────┬──┬─>
               │          ├─>─[value]──┤  │
               │          ╰─<─<─<─<─<──╯  │
               ╰─<─<─<─<─<─<─<─<─<─<─<─<──╯
**Object**:

    ──[object]──┬──[chunk]─┬─>────────────────────┬──┬─>
                │          ├─>─[string]──[value]──┤  │
                │          ╰─<─<─<─<─<─<─<─<─<─<──╯  │
                ╰─<─<─<─<─<─<─<─<─<─<─<─<─<─<─<─<─<──╯

**Structural validity rules**:

* A document **MUST** contain exactly one top-level value. An empty document (zero bytes) is invalid.
* Every container (array or object) **MUST** contain at least one [chunk](#chunking). Decoders **MUST** reject documents where a container's type code is followed by data that cannot be decoded as a valid [length field](#length-field).
* The final chunk of a container **MUST** have a [continuation bit](#length-field-payload-format) of 0. A document that ends with an incomplete container (continuation bit 1 with no following chunk) is invalid.

**Note**: For robustness, decoders **MAY** offer an option to recover partial data from truncated documents (see [Convenience Considerations](#convenience-considerations)).



Encoding
--------

BONJSON is a byte-oriented format. All values begin and end on an 8-bit boundary.

**Notes**:

 * All strings are encoded as UTF-8.
 * All numeric fields are encoded in little endian byte order.


### Type Codes

Every value is composed of an 8-bit type code, and in some cases also a payload:

| Type Code | Payload                      | Type      | Description                                |
| --------- | ---------------------------- | --------- | ------------------------------------------ |
| 00 - c8   |                              | Number    | [Integers -100 through 100](#small-integer)|
| c9 - cf   |                              |           | RESERVED                                   |
| d0 - d7   | Unsigned integer of n bytes  | Number    | [Unsigned Integer](#integer)               |
| d8 - df   | Signed integer of n bytes    | Number    | [Signed Integer](#integer)                 |
| e0 - ef   | String of n bytes            | String    | [Short String](#short-string)              |
| f0        | Arbitrary length string      | String    | [Long String](#long-string)                |
| f1        | Arbitrary length number      | Number    | [Big Number](#big-number)                  |
| f2        | 16-bit bfloat16 binary float | Number    | [16-bit float](#16-bit-float)              |
| f3        | 32-bit ieee754 binary float  | Number    | [32-bit float](#32-bit-float)              |
| f4        | 64-bit ieee754 binary float  | Number    | [64-bit float](#64-bit-float)              |
| f5        |                              | Null      | [Null](#null)                              |
| f6        |                              | Boolean   | [False](#boolean)                          |
| f7        |                              | Boolean   | [True](#boolean)                           |
| f8        |                              | Container | [Array](#array)                            |
| f9        |                              | Container | [Object](#object)                          |
| fa - ff   |                              |           | RESERVED                                   |

**Note**: Decoders **MUST** reject documents containing reserved type codes.



Strings
-------

Strings are UTF-8 encoded sequences of Unicode codepoints, and can be encoded in two ways.

**Note**: Encoders **SHOULD** produce [NFC-normalized](https://unicode.org/reports/tr15/#Norm_Forms) strings. See [Compliance Levels](#compliance-levels) and [Unicode Normalization](#unicode-normalization) for details.


### Short String

Short strings have their byte length (up to 15) encoded directly into the lower nybble of the type code.

    Type Code Byte
    ───────────────
    1 1 1 0 L L L L
            ╰─┴─┴─┤
                  ╰─> Length (0-15 bytes)


**Examples**:

    e0                                               // ""
    e1 41                                            // "A"
    ec e3 81 8a e3 81 af e3 82 88 e3 81 86           // "おはよう"
    ef 31 35 20 62 79 74 65 20 73 74 72 69 6e 67 21  // "15 byte string!"


### Long String

Long strings begin with the [type code](#type-codes) (`0xf0`), followed by one or more [chunks](#chunking). Each chunk's count specifies the number of UTF-8 bytes that follow.

    0xf0 [chunk] ...

**Examples**:

    f0 00                               // ""
    f0 20 61 20 73 74 72 69 6e 67       // "a string" (in 1 chunk)
    f0 06 61 12 20 73 74 72 0c 69 6e 67 // "a string" (in chunks: 1 byte, 4 bytes, 3 bytes)
    f0 01 02                            // (String of 64 Zs)
    5a 5a 5a 5a 5a 5a 5a 5a             // ZZZZZZZZ
    5a 5a 5a 5a 5a 5a 5a 5a             // ZZZZZZZZ
    5a 5a 5a 5a 5a 5a 5a 5a             // ZZZZZZZZ
    5a 5a 5a 5a 5a 5a 5a 5a             // ZZZZZZZZ
    5a 5a 5a 5a 5a 5a 5a 5a             // ZZZZZZZZ
    5a 5a 5a 5a 5a 5a 5a 5a             // ZZZZZZZZ
    5a 5a 5a 5a 5a 5a 5a 5a             // ZZZZZZZZ
    5a 5a 5a 5a 5a 5a 5a 5a             // ZZZZZZZZ



Numbers
-------

Numbers can be encoded using various integer and floating point forms. Encoders **SHOULD** use the most compact representation that stores each value without data loss.

All primitive numeric types are encoded exactly as they would appear in memory on little endian architectures.

**Notes**:

 * `NaN` and `infinity` are not valid BONJSON values. See [Values incompatible with JSON](#values-incompatible-with-json).
 * Decoders **MUST** accept any valid numeric encoding for a value, even if it is not the most compact representation. Only the mathematical value matters, not the encoding used to represent it.
 * A value such as `1.0` **MAY** be encoded as an integer (`0x65`) or as a float (`f3 00 00 80 3f`). Decoders **MUST** treat these as equivalent.


### Small Integer

Small integers (-100 to 100) are encoded into the [type code](#type-codes) itself for maximum compactness in the most commonly used integer range. The value is computed as: `type_code - 100`.

**Examples**:

    c8 //  100
    69 //    5
    64 //    0
    28 //  -60
    00 // -100


### Integer

Integers are encoded in little-endian byte order following the [type code](#type-codes). The lower nybble of the type code determines the size of the integer in bytes, and whether it is signed or unsigned.

    Type Code Byte
    ───────────────
    1 1 0 1 S L L L
            | ╰─┼─╯
            |   ╰───> Length (add 1 to yield a length from 1 to 8 bytes)
            ╰───────> Signed (0 = unsigned, 1 = signed)

Encoders **SHOULD** favor _signed_ over _unsigned_ when both types would encode a value into the same number of bytes.

**Examples**:

    d0 b4                      //  180
    d9 18 fc                   // -1000
    d1 00 80                   //  0x8000
    dd bc 9a 78 56 34 12       //  0x123456789abc
    df 00 00 00 00 00 00 00 80 // -0x8000000000000000
    d7 da da da de d0 d0 d0 de //  0xded0d0d0dedadada (is all I want to say to you)


### 16-bit Float

16-bit float is encoded as a little-endian 16-bit [bfloat16](https://en.wikipedia.org/wiki/Bfloat16_floating-point_format) following the [type code](#type-codes) (`0xf2`).

Bfloat16 is a truncated form of IEEE 754 binary32, consisting of 1 sign bit, 8 exponent bits, and 7 significand bits. It preserves the full exponent range of binary32 (±3.4 × 10³⁸) while reducing precision. Conversion to/from binary32 involves simply truncating or zero-extending the significand.

**Example**:

    f2 90 3f // 1.125


### 32-bit Float

32-bit float is encoded as a little-endian [32-bit ieee754 binary float](https://en.wikipedia.org/wiki/Single-precision_floating-point_format) following the [type code](#type-codes) (`0xf3`).

**Example**:

    f3 00 b8 1f 42 // 0x1.3f7p5


### 64-bit Float

64-bit float is encoded as a little-endian [64-bit ieee754 binary float](https://en.wikipedia.org/wiki/Double-precision_floating-point_format) following the [type code](#type-codes) (`0xf4`).

**Example**:

    f4 58 39 b4 c8 76 be f3 3f // 1.234


### Big Number

Big Number ([type code](#type-codes) `0xf1`) allows for encoding an incredibly large range of base-10 numbers. Although it doesn't match the unlimited range of [JSON](#json-standards), it does allow for up to 75 significant digits and an exponent range of ± 8 million, which is well beyond virtually all real-world use cases.

The structure of a big number is as follows:

    [header] [exponent bytes] [significand bytes]

 * The `header` is a byte containing sign and field-length information.
 * The `exponent` represents a base-10 exponent, and is encoded as a signed integer in little endian byte order.
 * The `significand` is encoded as an unsigned integer in little endian byte order. The `header` contains the significand's sign.

The `header` byte consists of 3 fields:

      Header Byte
    ───────────────
    S S S S S E E N
    ╰─┴─┼─┴─╯ ╰─┤ ╰─> Significand sign (0 = positive, 1 = negative)
        │       ╰───> Exponent Length (0-3 bytes)
        ╰───────────> Significand Length (0-31 bytes)

The final value is derived as: `sign` × `significand` × 10^`exponent`

**Note**: When the exponent length is 0, the exponent is 0.

#### Special Big Number Encodings

When the `significand length` field is 0, the `exponent length` field is repurposed to encode special values (listed in the table below), and there are no `significand` or `exponent` bytes following the header. In this case, the entire big number encoding occupies only two bytes: the type code and the header byte.

The `negative` bit represents the sign as usual.

**Note**: These special values are invalid by default. See [Values incompatible with JSON](#values-incompatible-with-json) for details and configuration options.

| Exponent Length Bits | Meaning            | Valid in BONJSON |
| -------------------- | ------------------ | ---------------- |
| `0 0`                | `0`                | ✔️               |
| `0 1`                | `infinity`         | ❌               |
| `1 0`                | `NaN` (quiet)      | ❌               |
| `1 1`                | `NaN` (signaling)  | ❌               |

**Examples**:

    f1 48 00 10 32 54 76 98 ba dc fe  // 0xfedcba987654321000 (9 bytes significand, no exponent, positive)
    f1 0a ff 0f                       // 1.5 (15 x 10⁻¹) (1 byte significand, 1 byte exponent, positive)
    f1 01                             // -0 (no significand, no exponent, negative)
    f1 8d 8d 01 97 EB F2 0E C3 98 06  // -13837758495464977165497261864967377972119 x 10⁻⁹⁰⁰⁰
       C1 47 71 5E 65 4F 58 5F AA 28  // (17 bytes significand, 2 bytes exponent, negative)



Containers
----------

Containers use [chunked](#chunking) encoding: the container [type code](#type-codes) is followed by one or more chunks. This allows decoders to pre-allocate storage while still supporting progressive encoding when the total count is not known up front.


### Array

An array consists of an `array` type code (`0xf8`), followed by one or more [chunks](#chunking). Each chunk's count specifies the number of array elements that follow.

    [array] [chunk] ...
     0xf8

**Examples**:

    f8 00                    // [] (empty array: count=0, cont=0)
    f8 0c e1 61 65 f5        // ["a", 1, null] (3 elements in one chunk)
    f8 08 e1 61 65 04 f5     // ["a", 1, null] (2 elements, then 1 element in separate chunks)


### Object

An object consists of an `object` type code (`0xf9`), followed by one or more [chunks](#chunking). Each chunk's count specifies the number of key-value pairs that follow, where each pair is a [string](#strings) key followed by a value.

    [object] [chunk] ...
      0xf9

**Notes**:

* Keys **MUST** be [strings](#strings).
* Every key **MUST** be followed by a value. A document that ends after a key but before its corresponding value is invalid.

**Examples**:

    f9 00                                // {} (empty object: count=0, cont=0)
    f9 08 e1 62 64 e4 74 65 73 74 e1 78  // {"b": 0, "test": "x"} (2 pairs in one chunk)



Boolean
-------

Boolean values are encoded into the [type codes](#type-codes) themselves:

 * False has type code `0xf6`
 * True has type code `0xf7`



Null
----

Null has [type code](#type-codes) `0xf5`.



Length Field
------------

Length fields are used to delimit long data types (such as [strings](#strings)), and also support splitting data into multiple chunks (which is useful for progressive encoding, where the total length of the data is not known up front).

Length-delimited data is safer because the receiving end can make decisions about the length of the data before attempting to ingest it.

A `length field` is composed of a `header` and possible `payload bytes`.

    [header] [payload bytes]

### Length Field Header

The `header` determines how many bytes comprise the length field itself, so that it doesn't occupy more bytes than are necessary.

The lower `header` bits contain a `count` field, encoded in a reversed unary code (going from low bit to high bit - or describing visually: right-to-left rather than left-to-right). The count field uses trailing `1` bits terminated by a `0` bit.

The upper `header` bits contain `payload` data (when there's room)

| Header     | Count | Extra Payload Bytes | Total Payload Bits |
| ---------- | ----- | ------------------- | ------------------ |
| `.......0` |   1   |          0          |          7         |
| `......01` |   2   |          1          |         14         |
| `.....011` |   3   |          2          |         21         |
| `....0111` |   4   |          3          |         28         |
| `...01111` |   5   |          4          |         35         |
| `..011111` |   6   |          5          |         42         |
| `.0111111` |   7   |          6          |         49         |
| `01111111` |   8   |          7          |         56         |
| `11111111` |   9   |          8          |         64         |

* Header bits shown as `0` and `1` are the bit patterns of the `count` field. It can be trivially decoded using a `ctz` (count trailing zeroes) compiler intrinsic on the bitwise-inverted header.
* Header bits shown as `.` are the lower bits of the `payload`.

The `count` field serves two purposes:

* It tells how many bytes this length field occupies (including the `header` byte itself).
* It tells how many bits to shift right when decoding in order to eliminate the `count` field (for payload sizes <= 56 bits).

Any upper `header` bits that aren't part of the `count` field hold the lower bits of the payload data. When `count` is greater than 1, the payload data spills over into (`count - 1`) `payload bytes`.

The entire encoded stream is stored in little endian byte order so that it can be efficiently read into a zeroed register on little endian architectures.

Consequently, the `header` occupies the lowest byte when the encoded data is loaded into a register, and the `payload bytes` progressively fill the higher bytes in typical little-endian ordering. Once loaded, one simply shifts the register right by `count` bits to eliminate the `count` field and yield the `payload`.

This encoding has the same size overhead as [LEB128](https://en.wikipedia.org/wiki/LEB128) (1 bit per byte), but is far more efficient to decode because the full size of the field can be determined from the first byte, and the overhead bits can be eliminated in a single shift operation.

**Note**: Encoders **SHOULD** use the most compact encoding possible for length fields. Decoders **MUST** accept non-canonical (oversized) length encodings. A length encoding is canonical if and only if it uses the minimum number of bytes required per the header table above. For example, length 0 with continuation 0 (payload 0) is canonically encoded as `00` (1 byte), but `01 00` (2 bytes) is also valid and decodes to the same value.


### Length Field Payload Format

A length field `payload` is itself composed of two components:

      Payload
    -----------
    ... L L L C
    ╰─┴─┼─┴─╯ ╰─> Continuation bit (1 means another chunk follows this one)
        ╰───────> Length (up to 0x7FFFFFFFFFFFFFFF)

The low bit of the `payload` is the `continuation bit`. When this bit is 1, there is another `length field` following the data chunk that this `length field` refers to.


### Length Payload Encoding Process

**For payloads containing 0 to 56 bits of data:**

* Determine the 1-based `position` of the _highest_ 1-bit (1-56) of the `payload`. If the `payload` is 0, use position 1.
* The overhead tradeoff is 7 bits per byte, so our `extra bytes count` (0-7) is `floor((position - 1) / 7)`.
* Copy your `payload` to a 64-bit `register`.
* Shift `register` left by `extra bytes count` + 1. This makes room for the `count` field (a terminating 0 bit plus `extra bytes count` trailing 1 bits).
* Set the lowest `extra bytes count` bits of `register` to 1 (the trailing 1s of the count field). The terminating 0 is already in place from the shift.
* Write `extra bytes count`+1 bytes of `register` in little endian byte order to the `destination buffer`.
* `destination buffer` now contains the encoded length field in `extra bytes count`+1 bytes.

**For payloads containing 57 to 64 bits of data:**

* Write the `header` byte 0xff.
* Write the 8 bytes of `payload` in little endian order.


### Length Payload Decoding Process

* Read the `header` byte.

**If the `header` byte is 0xff:**

* Discard the `header`
* Read the next 8 bytes in little endian order as the `payload`.

**Otherwise:**

* Bitwise-invert the `header` to get `inverted_header`.
* Determine the 1-based bit position of the _lowest_ 1-bit (1-8) of `inverted_header`. This is your `count`. (Equivalently: find the lowest 0-bit in the original `header`.)
* Read `count` bytes (including re-reading the `header` byte) as little-endian data into a zeroed 64-bit `register`.
* Shift `register` right by `count` bits.
* `register` now contains the decoded `payload`.


**Examples**:

    00                          // Length 0 and continuation 0
    02                          // Length 0 and continuation 1
    04                          // Length 1 and continuation 0
    06                          // Length 1 and continuation 1
    fc                          // Length 63 and continuation 0
    fe                          // Length 63 and continuation 1
    01 02                       // Length 64 and continuation 0
    05 02                       // Length 64 and continuation 1
    0b 24 f4                    // Length 1,000,000 and continuation 1
    ff fe ff ff ff ff ff ff ff  // Length 9,223,372,036,854,775,807 and continuation 0

### Chunking

Chunking is a mechanism for encoding variable-length data using one or more length-prefixed segments. This allows encoders to begin encoding data before the total size is known (progressive encoding), while still allowing decoders to pre-allocate storage when the size is known up front.

A `chunk` is comprised of a [length field](#length-field) specifying a count, followed by that many items of data:

    [count] [items ...]

The meaning of `count` depends on the data type being chunked:

| Type                        | Count represents           |
| --------------------------- | -------------------------- |
| [Long String](#long-string) | Bytes of UTF-8 string data |
| [Array](#array)             | Array elements             |
| [Object](#object)           | Key-value pairs            |

Chunking continues until a chunk whose length field's [continuation bit](#length-field-payload-format) is 0. If a chunk has a continuation bit of 1, another chunk **MUST** follow; a document that ends after such a chunk is invalid.

A chunk with count 0 **MUST** have a [continuation bit](#length-field-payload-format) of 0. Allowing otherwise would open up the decoder to DOS attacks.



Interoperability Considerations
-------------------------------

Because [JSON](#json-standards) is so weakly specified, there are numerous ways in which one implementation can become incompatible with another.


### Value Ranges

Although JSON allows an unlimited range for most values, it's important to take into consideration the limitations of the systems that will be trying to ingest your data.

Encoders **SHOULD** always use the smallest encoding for the value being encoded in order to ensure maximum compatibility.

A codec **MAY** choose its own value range restrictions, but **SHOULD** at least support reasonable industry-standard ranges for maximum interoperability, and **MUST** publish what its restrictions are.

Most systems can natively handle:

 * 64 bit floating point values
 * 64 bit integer values (signed or unsigned)

JavaScript in particular can natively handle:

 * 64 bit floating point values
 * 53 bit integer values (plus the sign)

**Warning**: [Big numbers](#big-number) can encode values with exponents up to ±8 million. Converting such values to decimal strings or performing arbitrary-precision arithmetic could consume excessive memory or CPU. Decoders **MUST** impose limits on numeric range as specified in [Resource Limits](#resource-limits).



Security Rules
--------------

All data formats contain technically feasible coding sequences that could result in exploitable weaknesses if certain invariants aren't artificially enforced. [JSON is particularly bad in this regard](https://bishopfox.com/blog/json-interoperability-vulnerabilities). BONJSON mitigates these by being cautious, rejecting dangerous sequences by default. However, since JSON is so old, systems have inevitably arisen that rely upon such broken behaviors.

BONJSON handles this by choosing opinionated, secure default behaviors while allowing informed users to deliberately disable such security protections when necessary.

**Note**: By "rejecting", it is meant that the entire document is rejected as a consequence of rejecting the unacceptable data (as opposed to just ignoring or replacing the unacceptable data).


### Resource Limits

Decoders **MUST** enforce limits on resource consumption to prevent denial-of-service attacks. The following limits **SHOULD** be configurable, with secure defaults:

| Resource               | Recommended Default  | Notes                                                                                                    |
| ---------------------- | -------------------- | -------------------------------------------------------------------------------------------------------- |
| Maximum document size  | 2,000,000,000 bytes  | Total size of the encoded document                                                                       |
| Maximum nesting depth  | 512                  | Arrays and objects nested within each other (root value is depth 0, first container is depth 1)          |
| Maximum container size | 1,000,000 elements   | Elements in a single array, or key-value pairs in a single object (each pair counts as one)              |
| Maximum string length  | 10,000,000 bytes     | Encoded byte length, before any normalization. See [Long Field Lengths](#long-field-lengths)             |
| Maximum chunk count    | 100                  | Per string or container; a single-chunk counts as 1. See [Chunking Restrictions](#chunking-restrictions) |
| Numeric range          | 64-bit float/integer | See [Value Ranges](#value-ranges)                                                                        |

Implementations **SHOULD** support at minimum 64-bit IEEE 754 floats and 64-bit signed and unsigned integers. JavaScript environments are limited to 53-bit integer precision; implementations targeting JavaScript **SHOULD** document this limitation.

Implementations **MUST** document their default limits and any configuration options they provide.

**Warning**: Choosing excessively high limits (or no limits) can make implementations vulnerable to resource exhaustion attacks. Even seemingly reasonable limits can be dangerous in combination (e.g., 1000 nesting depth × 1000 elements per level = 1 billion total nodes).


### Long Field Lengths

Any data format that includes length fields is by definition open to abuse. BONJSON decoders **MUST** provide protection against this:

* Enforce maximum string length limits as specified in [Resource Limits](#resource-limits).
* If the document length is known, decoders **MUST** reject any length field that specifies a value exceeding the remaining document length.


### Chunking Restrictions

Allowing unlimited [chunking](#chunking) opens the door for abusive DOS payloads (chunk bombs). Decoders **MUST** enforce maximum chunk count limits as specified in [Resource Limits](#resource-limits).

Decoders **MAY** also offer additional mitigation options:

* Limit chunks even more after a certain amount of data has been received (to prevent sending a large amount of normal data, followed by abusive chunks).
* Refuse chunking entirely (reject any chunk that contains a [continuation bit](#length-field-payload-format) of 1).


### Invalid UTF-8

BONJSON strings are encoded as UTF-8. Invalid UTF-8 sequences have been involved in numerous security exploits and **MUST** be rejected by default.

Invalid UTF-8 sequences include:

* Overlong encodings (using more bytes than necessary to encode a codepoint)
* Sequences that decode to values greater than U+10FFFF
* Sequences that decode to surrogate codepoints (U+D800-U+DFFF)
* Invalid continuation bytes
* Truncated multi-byte sequences

Further information about how invalid UTF-8 can become a security nightmare is available at [RFC 3629, section 10](https://www.rfc-editor.org/rfc/rfc3629#section-10), [Unicode Technical Report #36](https://www.unicode.org/reports/tr36/tr36-15.html), and [Corrigendum #1: UTF-8 Shortest Form](https://www.unicode.org/versions/corrigendum1.html). Overlong UTF-8 encodings were famously involved in the 2001 exploit of IIS web servers by encoding "../" as `C0 AE 2E 2F` instead of the expected `2E 2E 2F`.

For compatibility with broken data sources, decoders **MAY** offer the following configuration options:

* Reject the document (this **MUST** be the default behavior)
* Replace invalid sequences with the REPLACEMENT CHARACTER U+FFFD (less secure)
* Delete invalid sequences from the string (even less secure)
* Pass through invalid sequences unchanged (dangerous and **NOT RECOMMENDED**)


### NUL Codepoint Restriction

Because the `NUL` (U+0000) codepoint can so easily be exploited, it **MUST** be rejected by default.

Decoders **MAY** offer a configuration option to allow `NUL`, but the default behavior **MUST** be to reject.


### Unicode Normalization

[Unicode normalization](https://unicode.org/reports/tr15/) ensures that equivalent character sequences are represented consistently. Without normalization, strings that appear identical to users (such as "café" using precomposed `U+00E9` vs "café" using `U+0065 U+0301`) would be treated as different, which creates security vulnerabilities.

Some platforms (notably Swift and Objective-C) automatically normalize strings when using them as dictionary keys. This means two keys that passed a byte-for-byte duplicate check could still collide in the application's native map, allowing an attacker to control which value is kept.

The handling of Unicode normalization depends on the [compliance level](#compliance-levels):

* **Secure compliance**: Decoders normalize strings to [NFC](https://unicode.org/reports/tr15/#Norm_Forms) before performing [duplicate key](#duplicate-object-keys) detection. This prevents the attack described above.
* **Basic compliance**: Decoders perform byte-for-byte comparison without normalization. This is vulnerable to key collision attacks on auto-normalizing platforms.

**Important**: For [chunked](#chunking) [long strings](#long-string), NFC normalization **MUST** be applied after the complete string has been assembled, not per-chunk. Normalizing per-chunk would fail to compose characters that span chunk boundaries (e.g., a base character in one chunk and its combining mark in the next).

**Note**: Length fields in the encoded document refer to the byte length of the original (non-normalized) UTF-8 data. After NFC normalization, the resulting string may have a different byte length. For example, `café` encoded with a decomposed `é` (U+0065 U+0301) occupies 6 bytes, but after NFC normalization becomes 5 bytes (using precomposed U+00E9). This is expected behavior.


### Values incompatible with JSON

NaN and infinity are unrepresentable in JSON, but are technically possible to encode in BONJSON because it stores binary floating point values directly and [big numbers](#big-number) have special exponent encodings for these values.

Codecs **MUST** by default reject NaN and infinity values regardless of encoding (whether in [float](#16-bit-float) bit patterns or [big number](#big-number) special encodings), and **MAY** offer the following configuration options:

* Reject the document (this **MUST** be the default behavior)
* Stringify the value ("NaN", "infinity", or "-infinity")
* Allow the value (passing the decoded data to a JSON encoder may fail or produce different results)


### Out-of-range Values

Decoders **MUST** by default reject numeric values that are outside of the decoder's [allowed range](#value-ranges).

Decoders **MAY** offer an option to stringify such values, replacing the numeric value with a string value containing a [JSON](#json-standards) numeric literal or a C-style hexadecimal "0x" integer literal, but the default behavior **MUST** be to reject.


### Duplicate object keys

Duplicate [object](#object) keys (the same key appearing multiple times in the same object) are being actively exploited in the wild to compromise security and exfiltrate data. Allowing duplicate keys is **extremely** dangerous. However, some systems must allow it for interoperability reasons.

**Note**: Key equality is determined by byte-for-byte comparison of UTF-8 encoded strings. For [secure compliance](#secure-compliance), strings are NFC-normalized before comparison. For [basic compliance](#basic-compliance), raw (non-normalized) strings are compared. See [Unicode Normalization](#unicode-normalization) for security implications.

Decoders **MAY** offer the following configuration options:

* Reject documents containing duplicate keys (this **MUST** be the default option)
* Keep the first instance, ignoring duplicate keys (dangerous and possible to exploit in the right circumstances)
* Keep the last instance, replacing any already received keys (extremely dangerous and actively exploited)

If no configuration options are provided, documents containing duplicate keys **MUST** be rejected.


### Trailing Data

A BONJSON document consists of exactly one top-level value. Decoders **MUST** by default reject documents that contain extra data after the end of the root value.

Decoders **MAY** offer a configuration option to accept documents with trailing data, but this option **MUST** default to rejecting such documents.

If a decoder is configured to accept trailing data, it **SHOULD** return the number of bytes consumed so that the caller can process any remaining data.



Convenience Considerations
--------------------------

Decoders **SHOULD** offer an option (disabled by default) to allow for partial data to be recovered (along with an error condition) when decoding fails partway through.

This would involve discarding any partially decoded value (and its associated object member name - if any), and then artificially terminating all open arrays and objects to produce a well-formed tree.



Filename Extensions and MIME Type
---------------------------------

BONJSON files have the filename extension `bonjson`, or the shorter `boj`.

    example.boj
    example.bonjson

BONJSON's MIME type is `application/x.bonjson`, and will be `application/bonjson` once registered.



Full Example
------------

**JSON**:

```json
{
    "number": 50,
    "null": null,
    "boolean": true,
    "array": [
        "x",
        1000,
        -1.25
    ],
    "object": {
        "negative number": -100,
        "long string": "1234567890123456789012345678901234567890"
    }
}
```

    Size:     245 bytes
    Minified: 156 bytes

**BONJSON**:

```text
    f9 14                                                  // { (5 pairs)
       e6 6e 75 6d 62 65 72                                //     "number":
       96                                                  //     50,
       e4 6e 75 6c 6c                                      //     "null":
       f5                                                  //     null,
       e7 62 6f 6f 6c 65 61 6e                             //     "boolean":
       f7                                                  //     true,
       e5 61 72 72 61 79                                   //     "array":
       f8 0c                                               //     [ (3 elements)
          e1 78                                            //         "x",
          d9 e8 03                                         //         1000,
          f2 a0 bf                                         //         -1.25
                                                           //     ],
       e6 6f 62 6a 65 63 74                                //     "object":
       f9 08                                               //     { (2 pairs)
          ef 6e 65 67 61 74 69 76 65 20 6e 75 6d 62 65 72  //         "negative number":
          00                                               //         -100,
          eb 6c 6f 6e 67 20 73 74 72 69 6e 67              //         "long string":
          f0 a0                                            //         "1234567890123456789012345678901234567890"
             31 32 33 34 35 36 37 38 39 30                 //
             31 32 33 34 35 36 37 38 39 30                 //
             31 32 33 34 35 36 37 38 39 30                 //
             31 32 33 34 35 36 37 38 39 30                 //
                                                           //     }
                                                           // }
```

    Size:    121 bytes



Formal BONJSON Grammar
----------------------

The following [Dogma](https://github.com/kstenerud/dogma/blob/master/v1/dogma_v1.0.md) grammar is **normative**. In case of any discrepancy between the prose descriptions in this document and the formal grammar, the grammar takes precedence.

```dogma
dogma_v1 utf-8
- identifier  = bonjson
- description = Binary Object Notation for JSON
- reference   = https://bonjson.org
- dogma       = https://github.com/kstenerud/dogma/blob/master/v1/dogma_v1.0.md

document          = byte_order(lsb, ordered_document);
ordered_document  = value;

value             = array | object | number | boolean | string | null;

# Types

# Containers use chunked encoding with element/pair counts.
# Each chunk specifies exactly how many elements/pairs follow.
array                  = u8(0xf8) & element_chunk(1)* & element_chunk(0);
element_chunk(hasNext) = chunked(var(count, ~), hasNext) & value*;
object                 = u8(0xf9) & pair_chunk(1)* & pair_chunk(0);
pair_chunk(hasNext)    = chunked(var(count, ~), hasNext) & (string & value)*;

number            = int_small | int_unsigned | int_signed | float_16 | float_32 | float_64 | big_number;
int_small         = u8(var(code, 0x00~0xc8));  # value = code - 100
int_unsigned      = u4(0xd) & u1(0) & u3(var(count, ~)) & ordered(uint((count+1)*8, ~));
int_signed        = u4(0xd) & u1(1) & u3(var(count, ~)) & ordered(sint((count+1)*8, ~));
float_16          = u8(0xf2) & ordered(f16(~));
float_32          = u8(0xf3) & ordered(f32(~));
float_64          = u8(0xf4) & ordered(f64(~));
big_number        = u8(0xf1)
                  & var(header, big_number_header)
                  & [
                        header.sig_length > 0: ordered(sint(header.exp_length*8, ~))
                                             & ordered(uint(header.sig_length*8, ~))
                                             ;
                    ]
                  ;
big_number_header = u5(var(sig_length, ~)) & u2(var(exp_length, ~)) & u1(var(sig_negative, ~));

boolean           = true | false;
false             = u8(0xf6);
true              = u8(0xf7);

null              = u8(0xf5);

string                = string_short | string_long;
string_short          = u4(0xe) & u4(var(count, ~)) & sized(count*8, char_string*);
string_long           = u8(0xf0) & string_chunk(1)* & string_chunk(0);
string_chunk(hasNext) = chunked(var(count, ~), hasNext) & sized(count*8, char_string*);

chunked(len, hasNext) = length(len * 2 + hasNext);
length(l)             = ordered([
                                    l >=                 0 & l <=               0x7f: uint( 7, l) & uint(1, 0x00);
                                    l >=              0x80 & l <=             0x3fff: uint(14, l) & uint(2, 0x01);
                                    l >=            0x4000 & l <=           0x1fffff: uint(21, l) & uint(3, 0x03);
                                    l >=          0x200000 & l <=          0xfffffff: uint(28, l) & uint(4, 0x07);
                                    l >=        0x10000000 & l <=        0x7ffffffff: uint(35, l) & uint(5, 0x0f);
                                    l >=       0x800000000 & l <=      0x3ffffffffff: uint(42, l) & uint(6, 0x1f);
                                    l >=     0x40000000000 & l <=    0x1ffffffffffff: uint(49, l) & uint(7, 0x3f);
                                    l >=   0x2000000000000 & l <=   0xffffffffffffff: uint(56, l) & uint(8, 0x7f);
                                    l >= 0x100000000000000 & l <= 0xffffffffffffffff: uint(64, l) & uint(8, 0xff);
                                ]);

# Primitives & Functions

u1(v)             = uint(1, v);
u2(v)             = uint(2, v);
u3(v)             = uint(3, v);
u4(v)             = uint(4, v);
u5(v)             = uint(5, v);
u8(v)             = uint(8, v);
i8(v)             = sint(8, v);
f16(v)            = sized(16, f32(v)); # bfloat16
f32(v)            = float(32, v);
f64(v)            = float(64, v);

char_string       = unicode(Cc,Cf,Co,Cn,L,M,N,P,S,Z); # Category C minus Cs (surrogates)
```



JSON Standards
--------------

Any discussions about JSON are done within the context of the ECMA and RFC specifications for JSON, and the json.org website:

 * [ECMA-404](https://ecma-international.org/publications-and-standards/standards/ecma-404/)
 * [RFC 8259](https://www.rfc-editor.org/info/rfc8259)
 * [json.org](https://www.json.org)



License
-------

Copyright (c) 2025 Karl Stenerud. All rights reserved.

Distributed under the [Creative Commons Attribution License](https://creativecommons.org/licenses/by/4.0/legalcode) ([license deed](https://creativecommons.org/licenses/by/4.0)).
