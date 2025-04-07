BONJSON: Binary Object Notation for JSON
========================================

** * * * **PRERELEASE** * * * **


BONJSON is a _hardened_, _lightning-fast_ and _efficient_, **1:1 compatible** binary drop-in replacement for [JSON](#json-standards).

It's **35 times** faster to process than [JSON](#json-standards).

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
  - [Types](#types)
  - [Structure](#structure)
  - [Encoding](#encoding)
    - [Type Codes](#type-codes)
  - [Strings](#strings)
    - [Short String](#short-string)
    - [Long String](#long-string)
      - [String Chunk](#string-chunk)
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
  - [Interoperability Considerations](#interoperability-considerations)
    - [Value Ranges](#value-ranges)
  - [Security Rules](#security-rules)
    - [Field Lengths](#field-lengths)
    - [UTF-8 Codepoints and Encoding](#utf-8-codepoints-and-encoding)
    - [Mandatory Hardening](#mandatory-hardening)
      - [Out-of-range Values](#out-of-range-values)
      - [Chunking Restrictions](#chunking-restrictions)
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

-----------------------------------------------------------------------------------------------------------------------



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

**Object**:

    ──[begin object]─┬─>────────────────────┬─[end container]──>
                     ├─>─[string]──[value]──┤
                     ╰─<─<─<─<─<─<─<─<─<─<──╯

**Array**:

    ──[begin array]─┬─>──────────┬─[end container]──>
                    ├─>─[value]──┤
                    ╰─<─<─<─<─<──╯



Encoding
--------

BONJSON is a byte-oriented format. All values begin and end on an 8-bit boundary.

**Notes**:

 * All strings are encoded as [UTF-8](#utf-8-codepoints-and-encoding).
 * All numeric fields are encoded in little endian byte order.


### Type Codes

Every value is composed of an 8-bit type code, and in some cases also a payload:

| Type Code | Payload                      | Type    | Description                                |
| --------- | ---------------------------- | ------- | ------------------------------------------ |
| 00 - 64   |                              | Number  | [Integers 0 through 100](#small-integer)   |
| 65 - 67   |                              |         | RESERVED                                   |
| 68        | Arbitrary length string      | String  | [Long String](#long-string)                |
| 69        | Arbitrary length number      | Number  | [Big Number](#big-number)                  |
| 6a        | 16-bit bfloat16 binary float | Number  | [16-bit float](#16-bit-float)              |
| 6b        | 32-bit ieee754 binary float  | Number  | [32-bit float](#32-bit-float)              |
| 6c        | 64-bit ieee754 binary float  | Number  | [64-bit float](#64-bit-float)              |
| 6d        |                              | Null    | [Null](#null)                              |
| 6e        |                              | Boolean | [False](#boolean)                          |
| 6f        |                              | Boolean | [True](#boolean)                           |
| 70 - 77   | Unsigned integer of n bytes  | Number  | [Unsigned Integer](#integer)               |
| 78 - 7f   | Signed integer of n bytes    | Number  | [Signed Integer](#integer)                 |
| 80 - 8f   | String of n bytes            | String  | [Short String](#short-string)              |
| 90 - 98   |                              |         | RESERVED                                   |
| 99        |                              | Array   | [Array start](#array)                      |
| 9a        |                              | Object  | [Object start](#object)                    |
| 9b        |                              |         | [Container end](#containers)               |
| 9c - ff   |                              | Number  | [Integers -100 through -1](#small-integer) |



Strings
-------

Strings are sequences of [UTF-8](#utf-8-codepoints-and-encoding) characters, and can be encoded in two ways:


### Short String

Short strings have their byte length (up to 15) encoded directly into the lower nybble of the type code.

    Type Code Byte
    ───────────────
    1 0 0 0 L L L L
            ╰─┴─┴─┤
                  ╰─> Length (0-15 bytes)


**Examples**:

    80                                               // ""
    81 41                                            // "A"
    8c e3 81 8a e3 81 af e3 82 88 e3 81 86           // "おはよう"
    8f 31 35 20 62 79 74 65 20 73 74 72 69 6e 67 21  // "15 byte string!"


### Long String

Long strings begin with the [type code](#type-codes) (`0x68`), followed by one or more [string chunks](#string-chunk).

    0x68 [string chunk] ...

#### String Chunk

A `string chunk` is comprised of a [length field](#length-field), followed by that many bytes of string data.

    [length] [bytes]

Chunking continues until the end of a chunk whose length field's [`continuation bit`](#length-field-payload-format) is 0.

**Note**: Each string chunk **MUST** be individually verifiable as a complete and valid UTF-8 string. If a string chunk cannot be fully decoded on its own and validated, the decoder **MUST** reject the document.

**Examples**:

    68 01                               // ""
    68 21 61 20 73 74 72 69 6e 67       // "a string" (in 1 chunk)
    68 07 61 13 20 73 74 72 0d 69 6e 67 // "a string" (in chunks: 1 byte, 4 bytes, 3 bytes)
    68 02 02                            // (String of 64 Zs)
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

**Note**: `NaN` and `infinity` values **MUST NOT** be present in a document since they are not allowed in [JSON](#json-standards).


### Small Integer

Small integers (-100 to 100) are encoded into the [type code](#type-codes) itself for maximum compactness in the most commonly used integer range. Casting the [type code](#type-codes) to an 8-bit signed integer yields its value.

**Examples**:

    64 //  100
    05 //    5
    00 //    0
    c4 //  -60
    9c // -100


### Integer

Integers are encoded in little-endian byte order following the [type code](#type-codes). The lower nybble of the type code determines the size of the integer in bytes, and whether it is signed or unsigned.

    Type Code Byte
    ───────────────
    0 1 1 1 S L L L
            | ╰─┼─╯
            |   ╰───> Length (add 1 to yield a length from 1 to 8 bytes)
            ╰───────> Signed (0 = unsigned, 1 = signed)

Encoders **SHOULD** favor _signed_ over _unsigned_ when both types would encode a value into the same number of bytes.

**Examples**:

    70 b4                      //  180
    79 18 fc                   // -1000
    71 00 80                   //  0x8000
    75 bc 9a 78 56 34 12       //  0x123456789abc
    7f 00 00 00 00 00 00 00 80 // -0x8000000000000000
    77 da da da de d0 d0 d0 de //  0xded0d0d0dedadada (is all I want to say to you)


### 16-bit Float

16-bit float is encoded as a little-endian 16-bit [bfloat16](https://en.wikipedia.org/wiki/Bfloat16_floating-point_format) following the [type code](#type-codes) (`0x6a`).

**Example**:

    6a 90 3f // 1.125


### 32-bit Float

32-bit float is encoded as a little-endian [32-bit ieee754 binary float](https://en.wikipedia.org/wiki/Single-precision_floating-point_format) following the [type code](#type-codes) (`0x6b`).

**Example**:

    6b 00 b8 1f 42 // 0x1.3f7p5


### 64-bit Float

64-bit float is encoded as a little-endian [64-bit ieee754 binary float](https://en.wikipedia.org/wiki/Double-precision_floating-point_format) following the [type code](#type-codes) (`0x6c`).

**Example**:

    6c 58 39 b4 c8 76 be f3 3f // 1.234


### Big Number

Big Number ([type code](#type-codes) `0x69`) allows for encoding an incredibly large range of base-10 numbers. Although it doesn't match the unlimited range of [JSON](#json-standards), it does allow for up to 75 significant digits and an exponent range of ± 8 million, which is well beyond virtually all real-world use cases.

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

When the `significand length` field is 0 (regardless of the contents of the `exponent length` field), then there are never any `significand` or `exponent` bytes (the entire encoded value occupies a single byte).

Instead, the `exponent length` field's bits represent the special values listed in the following table (with the `negative` bit representing the sign as usual).

**Note**: Most of these special values are invalid in BONJSON due to [JSON](#json-standards)'s value restrictions against infinity and NaN. If JSON is ever updated to support such values, this is how they would be encoded.

| Exponent Length Bits | Meaning            | Valid in BONJSON |
| -------------------- | ------------------ | ---------------- |
| `0 0`                | `0`                | ✔️                |
| `0 1`                | `infinity`         | ❌                |
| `1 0`                | `NaN` (quiet)      | ❌                |
| `1 1`                | `NaN` (signaling)  | ❌                |

**Examples**:

    69 48 00 10 32 54 76 98 ba dc fe  // 0xfedcba987654321000 (9 bytes significand, no exponent, positive)
    69 0a ff 0f                       // 1.5 (15 x 10⁻¹) (1 byte significand, 1 byte exponent, positive)
    69 01                             // -0 (no significand, no exponent, negative)
    69 8d 8d 01 97 EB F2 0E C3 98 06  // -13837758495464977165497261864967377972119 x 10⁻⁹⁰⁰⁰
       C1 47 71 5E 65 4F 58 5F AA 28  // (17 bytes significand, 2 bytes exponent, negative)



Containers
----------

Containers (objects and arrays) are encoded beginning with a `container start` [type code](#type-codes), and ending with a `container end` [type code](#type-codes). Both [object](#object) and [array](#array) share the same `container end` [type code](#type-codes) (`0x9b`).


### Array

An array consists of an `array start` (`0x99`), an optional collection of values, and finally a `container end` (`0x9b`):

    [array start] (value ...) [container end]
         0x99        ...          0x9b

**Example**:

    99        // [
       81 61  //     "a",
       01     //     1
       6d     //     null
    9b        // ]


### Object

An object consists of an `object start` (`0x9a`), an optional collection of name-value pairs, and finally a `container end` (`0x9b`):

    [object start] (name+value ...) [container end]
         0x9a            ...            0x9b

**Notes**:

* Names **MUST** be [strings](#strings).
* Names **MUST NOT** be [null](#null).
* Names **MUST NOT** be duplicates of other names in the same object.

**Example**:

    9a                 // {
       81 62           //     "b":
       00              //     0,
       84 74 65 73 74  //     "test":
       81 78           //     "x"
    9b                 // }



Boolean
-------

Boolean values are encoded into the [type codes](#type-codes) themselves:

 * False has type code `0x6e`
 * True has type code `0x6f`



Null
----

Null has [type code](#type-codes) `0x6d`.



Length Field
------------

Length fields are used to delimit long data types (such as [strings](#strings)), and also support splitting data into multiple chunks (which is useful for progressive encoding, where the total length of the data is not known up front).

Length-delimited data is safer because the receiving end can make decisions about the length of the data before attempting to ingest it.

A `length field` is composed of a `header` and possible `payload bytes`.

    [header] [payload bytes]

### Length Field Header

The `header` determines how many bytes comprise the length field itself, so that it doesn't occupy more bytes than are necessary.

The lower `header` bits contain a `count` field, encoded in a reversed unary code (going from low bit to high bit - or describing visually: right-to-left rather than left-to-right)

The upper `header` bits contain `payload` data (when there's room)

| Header     | Count | Extra Payload Bytes | Total Payload Bits |
| ---------- | ----- | ------------------- | ------------------ |
| `.......1` |   1   |          0          |          7         |
| `......10` |   2   |          1          |         14         |
| `.....100` |   3   |          2          |         21         |
| `....1000` |   4   |          3          |         28         |
| `...10000` |   5   |          4          |         35         |
| `..100000` |   6   |          5          |         42         |
| `.1000000` |   7   |          6          |         49         |
| `10000000` |   8   |          7          |         56         |
| `00000000` |   9   |          8          |         64         |

* Header bits shown as `0` and `1` are the possible bit patterns of the `count` field.
* Header bits shown as `.` are the lower bits of the `payload`.

The `count` field serves two purposes:

* It tells how many bytes this length field occupies (including the `header` byte itself).
* It tells how many bits to shift right when decoding in order to eliminate the `count` field (for payload sizes <= 56 bits).

Any upper `header` bits that aren't part of the `count` field hold the lower bits of the payload data. When `count` is greater than 1, the payload data spills over into (`count - 1`) `payload bytes`.

The entire encoded stream is stored in little endian byte order so that it can be efficiently read into a zeroed register on little endian architectures.

Consequently, the `header` occupies the lowest byte when the encoded data is loaded into a register, and the `payload bytes` progressively fill the higher bytes in typical little-endian ordering. Once loaded, one simply shifts the register right by `count` bits to eliminate the `count` field and yield the `payload`.

This encoding has the same size overhead as [LEB128](https://en.wikipedia.org/wiki/LEB128) (1 bit per byte), but is far more efficient to decode because the full size of the field can be determined from the first byte, and the overhead bits can be elimitated in a single shift operation.


### Length Field Payload Format

A length field `payload` is itself composed of two components:

      Payload
    -----------
    ... L L L C
    ╰─┴─┼─┴─╯ ╰─> Continuation bit (1 means another chunk follows this one)
        ╰───────> Length (up to 0x7FFFFFFFFFFFFF)

The low bit of the `payload` is the `continuation bit`. When this bit is 1, there is another `length field` following the data chunk that this `length field` refers to.


### Length Payload Encoding Process

**For payloads containing 0 to 56 bits of data:**

* Determine the 1-based `position` of the _highest_ set-bit of the `payload` (1-56).
  * When encoding a `payload` value of 0, consider it to have `position` 1.
* The overhead tradeoff is 7 bits per byte, so our `extra bytes count` (0-7) is `floor((position - 1) / 7)`.
* Copy your `payload` to a 64-bit `register`.
* Shift `register` left by 1 and set the lowest bit to 1.
* Shift `register` left by `extra bytes count`.
  * `register` now holds the `payload` in its upper bits, and the `count` field in its lower bits.
* Write `extra bytes count`+1 bytes of `register` in little endian byte order to the `destination buffer`.
* `destination buffer` now contains the encoded length field in `extra bytes count`+1 bytes.

**For payloads containing 57 to 64 bits of data:**

* Write the `header` byte 0x00.
* Write the 8 bytes of `payload` in little endian order.


### Length Payload Decoding Process

* Read the `header` byte.

**If the `header` byte is 0x00:**

* Discard the `header`
* Read the next 8 bytes in little endian order as the `payload`.

**Otherwise:**

* Determine the 1-based bit position of the _lowest_ set-bit of the `header` (1-8). This is your `count`.
* Read `count` bytes (including re-reading the `header` byte) as little-endian data into a zeroed 64-bit `register`.
* shift `register` right by `count` bits.
* `register` now contains the decoded `payload`.


**Examples**:

    01                          // Length 0 and continuation 0
    03                          // Length 0 and continuation 1
    05                          // Length 1 and continuation 0
    07                          // Length 1 and continuation 1
    fd                          // Length 63 and continuation 0
    ff                          // Length 63 and continuation 1
    02 02                       // Length 64 and continuation 0
    06 02                       // Length 64 and continuation 1
    0c 24 f4                    // Length 1,000,000 and continuation 1
    00 fe ff ff ff ff ff ff ff  // Length 9,223,372,036,854,775,807 and continuation 0



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



Security Rules
--------------

### Field Lengths

Any data format that includes length fields is by definition open to abuse. BONJSON decoders **MUST** provide protection against this. For example:

 * Maximum lengths (possibly user-configurable with sane defaults) for potentially long data types (such as [strings](#long-string)).
 * Sanity check: Does a length field contain a value greater than the remaining document length (if known)?


### UTF-8 Codepoints and Encoding

Although [JSON](#json-standards) technically supports the full range of [UTF-8](https://en.wikipedia.org/wiki/UTF-8) codepoints from 0 to 0x10ffff, many of these are actually invalid (such as surrogates, reserved codepoints, and codepoints marked as permanently invalid). [RFC 3629, section 10](https://www.rfc-editor.org/rfc/rfc3629#section-10) explains many of the problems that come from lax treatment of UTF-8 data.

Indeed, there are a whole slew of security issues concerning [improper handling of UTF-8](https://www.unicode.org/reports/tr36/)!

BONJSON prohibits all invalid UTF-8 encodings, including [overlong UTF-8 encodings](https://en.wikipedia.org/wiki/UTF-8#Overlong_encodings) (which are [invalid as of Unicode 3.0.1](https://www.unicode.org/versions/corrigendum1.html)). Overlong UTF-8 encodings were famously involved in the 2001 exploit of IIS web severs by encoding "../" as `2F C0 AE 2E 2F` instead of the `2F 2E 2E 2F` encoding that the servers were guarding against.


### Mandatory Hardening

JSON is by nature [vulnerable](https://bishopfox.com/blog/json-interoperability-vulnerabilities) to attackers who can take advantage of differences between codec implementations due to the laxity of the [JSON specifications](#json-standards). In order to mitigate such vulnerabilities, BONJSON decoders **MUST** implement the following hardening rules:

* Reject documents where an object contains duplicate names (this check **MUST** be made on the [normalized](https://www.unicode.org/reports/tr15/) Unicode representation).
* Reject documents where a string contains [invalid UTF-8 data](#utf-8-codepoints-and-encoding) (truncation or replacement is **unsafe**, and not allowed).
* Reject documents containing disallowed values (such as NaN or infinity).

#### Out-of-range Values

Decoders **MAY** offer these configuration options (and no other) for what to do when encountering numeric values that are outside of the decoder's [allowed range](#value-ranges):

* Reject the document.
* Stringify the number, replacing the numeric value with a string value containing a [JSON](#json-standards) numeric literal or a C-style hexadecimal "0x" integer literal.

Rejecting the document **MUST** be the default behavior. If the decoder doesn't offer configuration options for out-of-range values, rejecting the document **MUST** be the _only_ behavior.

#### Chunking Restrictions

Decoders **MAY** offer configuration options for when to allow [chunking](#string-chunk), such as:

* Refuse chunking entirely. In this case, the [continuation bit](#length-field-payload-format) **MUST** always be 0.
* Limit the maximum number of chunks allowed at a time (to prevent abuses like a long series of length-1 chunks).
* Limit chunks even more after a certain amount of data has been received (to prevent sending a large amount of normal data, followed by abusive chunks).
* Allow chunking with no limitations.

Refusing chunking entirely **MUST** be the default behavior. If the decoder doesn't offer configuration options for chunking, refusing chunking entirely **MUST** be the _only_ behavior.



Convenience Considerations
--------------------------

Decoders **SHOULD** offer an option (disabled by default) to allow for partial data to be recovered (along with an error condition) when decoding fails partway through.

This would involve discarding any partially decoded value (and its associated object member name - if any), and then artificially terminating all open arrays and objects to produce a well-formed tree.



Filename Extensions and MIME Type
---------------------------------

BONJSON files can use the filename extension `bonjson`, or the shorter `boj`.

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
    9a                                                     // {
       86 6e 75 6d 62 65 72                                //     "number":
       32                                                  //     50,
       84 6e 75 6c 6c                                      //     "null":
       6d                                                  //     null,
       87 62 6f 6f 6c 65 61 6e                             //     "boolean":
       6f                                                  //     true,
       85 61 72 72 61 79                                   //     "array":
       99                                                  //     [
          81 78                                            //         "x",
          79 e8 03                                         //         1000,
          6a a0 bf                                         //         -1.25
       9b                                                  //     ],
       86 6f 62 6a 65 63 74                                //     "object":
       9a                                                  //     {
          8f 6e 65 67 61 74 69 76 65 20 6e 75 6d 62 65 72  //         "negative number":
          9c                                               //         -100,
          8b 6c 6f 6e 67 20 73 74 72 69 6e 67              //         "long string":
          68 a1                                            //         "1234567890123456789012345678901234567890"
             31 32 33 34 35 36 37 38 39 30                 //
             31 32 33 34 35 36 37 38 39 30                 //
             31 32 33 34 35 36 37 38 39 30                 //
             31 32 33 34 35 36 37 38 39 30                 //
       9b                                                  //     }
    9b                                                     // }
```

    Size:    121 bytes



Formal BONJSON Grammar
----------------------

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

array             = u8(0x99) & value* & end_container;
object            = u8(0x9a) & (string & value)* & end_container;
end_container     = u8(0x9b);

number            = int_small | int_unsigned | int_signed | float_16 | float_32 | float_64 | big_number;
int_small         = i8(-100~100);
int_unsigned      = u4(7) & u1(0) & u3(var(count, ~)) & ordered(uint((count+1)*8, ~));
int_signed        = u4(7) & u1(1) & u3(var(count, ~)) & ordered(sint((count+1)*8, ~));
float_16          = u8(0x6a) & ordered(f16(~));
float_32          = u8(0x6b) & ordered(f32(~));
float_64          = u8(0x6c) & ordered(f64(~));
big_number        = u8(0x69)
                  & var(header, big_number_header)
                  & [
                        header.sig_length > 0: ordered(sint(header.exp_length*8, ~))
                                             & ordered(uint(header.sig_length*8, ~))
                                             ;
                    ]
                  ;
big_number_header = u5(var(sig_length, ~)) & u2(var(exp_length, ~)) & u1(var(sig_negative, ~));

boolean           = true | false;
false             = u8(0x6e);
true              = u8(0x6f);

null              = u8(0x6d);

string                = string_short | string_long;
string_short          = u4(8) & u4(var(count, ~)) & sized(count*8, char_string*);
string_long           = u8(0x68) & string_chunk(1)* & string_chunk(0);
string_chunk(hasNext) = chunked(var(count, ~), hasNext) & sized(count*8, char_string*);

chunked(len, hasNext) = length(len * 2 + hasNext);
length(l)             = ordered([
                                    l >=                 0 & l <=               0x7f: uint( 7, l) & uint(1, 0x01);
                                    l >=              0x80 & l <=             0x3fff: uint(14, l) & uint(2, 0x02);
                                    l >=            0x4000 & l <=           0x1fffff: uint(21, l) & uint(3, 0x04);
                                    l >=          0x200000 & l <=          0xfffffff: uint(28, l) & uint(4, 0x08);
                                    l >=        0x10000000 & l <=        0x7ffffffff: uint(35, l) & uint(5, 0x10);
                                    l >=       0x800000000 & l <=      0x3ffffffffff: uint(42, l) & uint(6, 0x20);
                                    l >=     0x40000000000 & l <=    0x1ffffffffffff: uint(49, l) & uint(7, 0x40);
                                    l >=   0x2000000000000 & l <=   0xffffffffffffff: uint(56, l) & uint(8, 0x80);
                                    l >= 0x100000000000000 & l <= 0xffffffffffffffff: uint(64, l) & uint(8, 0x00);
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

char_string       = unicode(C,L,M,N,P,S,Z);
```



JSON Standards
--------------

Any discussions about JSON are done within the context of the ECMA and RFC specifications for JSON, and the json.org website:

 * [ECMA-404](https://ecma-international.org/publications-and-standards/standards/ecma-404/)
 * [RFC 8259](https://www.rfc-editor.org/info/rfc8259)
 * [json.org](https://www.json.org)



License
-------

Copyright (c) 2024 Karl Stenerud. All rights reserved.

Distributed under the [Creative Commons Attribution License](https://creativecommons.org/licenses/by/4.0/legalcode) ([license deed](https://creativecommons.org/licenses/by/4.0).
