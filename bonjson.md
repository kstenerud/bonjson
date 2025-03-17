BONJSON: Binary Object Notation for JSON
========================================

** * * * **PRERELEASE** * * * **


BONJSON is a **1:1 compatible** binary serialization format for [JSON](#json-standards).

It's a drop-in replacement that works in the same way and has the same capabilities and limitations as [JSON](#json-standards) (no more, no less), in a compact and simple-to-process binary format that is 30x faster to process.

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
      - [About the Termination Delimiter](#about-the-termination-delimiter)
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
  - [Filename Extensions and MIME Type](#filename-extensions-and-mime-type)
  - [Full Example](#full-example)
  - [Interoperability Considerations](#interoperability-considerations)
    - [Value Ranges](#value-ranges)
  - [Security Considerations](#security-considerations)
    - [Hardening](#hardening)
  - [Convenience Considerations](#convenience-considerations)
  - [JSON Standards](#json-standards)
  - [Formal BONJSON Grammar](#formal-bonjson-grammar)
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
      │             │
      ├─>─[number]──┤
      │             │
      ├─>─[object]──┤
      │             │
      ├─>─[array]───┤
      │             │
      ├─>─[boolean]─┤
      │             │
      ╰─>─[null]────╯

**Object**:

    ──[begin object]─┬─>────────────────────┬─[end container]──>
                     │                      │
                     ├─>─[string]──[value]──┤
                     │                      │
                     ╰─<─<─<─<─<─<─<─<─<─<──╯

**Array**:

    ──[begin array]─┬─>──────────┬─[end container]──>
                    │            │
                    ├─>─[value]──┤
                    │            │
                    ╰─<─<─<─<─<──╯



Encoding
--------

BONJSON is a byte-oriented format. All values begin and end on an 8-bit boundary.

**Notes**:

 * All strings are encoded as UTF-8.
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

Strings **MUST** be encoded in UTF-8. BONJSON supports the same UTF-8 codepoints as [JSON](#json-standards) does, but does not implement escape sequences (which are unnecessary in a binary format).

Strings can be encoded in two ways:


### Short String

Short strings have their byte length (up to 15) encoded directly into the lower nybble of the type code, and have no terminator byte.

    Type Code Byte
    ───────────────
    1 0 0 0 L L L L
            ╰─┴─┴─┤
                  ╰─> Length (0-15 bytes)


**Example**:

    80                                               // ""
    81 41                                            // "A"
    8c e3 81 8a e3 81 af e3 82 88 e3 81 86           // "おはよう"
    8f 31 35 20 62 79 74 65 20 73 74 72 69 6e 67 21  // "15 byte string!"


### Long String

Long strings begin after the [type code](#type-codes) (`0x68`), and are terminated by the byte `0xff`.

#### About the Termination Delimiter

`0xff` was chosen as the string termination delimiter because it's never a valid byte within a UTF-8 sequence (and never will be until we surpass 68 _billion_ codepoints). This also gives the benefit that there's no need to decode the individual UTF-8 codepoints in order to find the end of a string (simply scan the bytes until `0xff` is encountered).

A C implementation for example could use [`memccpy()`](https://www.man7.org/linux/man-pages/man3/memccpy.3.html) and then replace the copied `0xff` with 0 to produce a null-terminated string. Alternatively, if the source buffer is writable, one could overwrite the `0xff` delimiter with 0 to yield a zero-copy null-terminated string using the source buffer as a backing store.

**Example**:

    68 ff                         // ""
    68 61 20 73 74 72 69 6e 67 ff // "a string"



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
            |   ╰───> Length (add 1 to value for 1-8 bytes)
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

**Notes**:

 * A field length of 0 implies a value of 0 for the field whose length it defines.
 * A big number **MUST** be encoded into its smallest valid representation (no stuffing `00` bytes).

#### Special Big Number Encodings

When the `significand length` field is 0 (regardless of the contents of the `exponent length` field), then there are never any `significand` or `exponent` bytes (the entire encoded value occupies a single byte).

Instead, the `exponent length` bits represent the special values listed in the following table (with the `negative` bit representing the sign as usual).

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
        81 61 //     "a",
        01    //     1
        6d    //     null
    9b        // ]


### Object

An object consists of an `object start` (`0x9a`), an optional collection of name-value pairs, and finally a `container end` (`0x9b`):

    [object start] (name+value ...) [container end]
         0x9a            ...            0x9b

**Note**: Names **MUST** be strings and **MUST NOT** be null.

**Example**:

    9a                  // {
        81 62           //     "b":
        00              //     0,
        84 74 65 73 74  //     "test":
        81 78           //     "x"
    9b                  // }



Boolean
-------

Boolean values are encoded into the [type codes](#type-codes) themselves:

 * False has type code `0x6e`
 * True has type code `0x6f`



Null
----

Null has [type code](#type-codes) `0x6d`.



Filename Extensions and MIME Type
---------------------------------

BONJSON files can use the filename extension `bonjson`, or the shorter `bjn`.

    example.bjn
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
    9a                                                       // {
        86 6e 75 6d 62 65 72                                 //     "number":
        32                                                   //     50,
        84 6e 75 6c 6c                                       //     "null":
        6d                                                   //     null,
        87 62 6f 6f 6c 65 61 6e                              //     "boolean":
        6f                                                   //     true,
        85 61 72 72 61 79                                    //     "array":
        99                                                   //     [
            81 78                                            //         "x",
            79 e8 03                                         //         1000,
            6a a0 bf                                         //         -1.25
        9b                                                   //     ],
        86 6f 62 6a 65 63 74                                 //     "object":
        9a                                                   //     {
            8f 6e 65 67 61 74 69 76 65 20 6e 75 6d 62 65 72  //         "negative number":
            9c                                               //         -100,
            8b 6c 6f 6e 67 20 73 74 72 69 6e 67              //         "long string":
            68                                               //         "1234567890123456789012345678901234567890"
                31 32 33 34 35 36 37 38 39 30                //
                31 32 33 34 35 36 37 38 39 30                //
                31 32 33 34 35 36 37 38 39 30                //
                31 32 33 34 35 36 37 38 39 30                //
            ff                                               //
        9b                                                   //     }
    9b                                                       // }
```

    Size:    121 bytes



Interoperability Considerations
-------------------------------

Because [JSON](#json-standards) is so weakly specified, there are numerous ways in which one implementation can become incompatible with another. [RFC 8259](https://www.rfc-editor.org/info/rfc8259) discusses many of these issues and how to mitigate them. BONJSON implementations **SHOULD** follow their advice.


### Value Ranges

Although JSON allows an unlimited range for most values, it's important to take into consideration the limitations of the systems that will be trying to ingest your data, and how they're likely to deal with out-of-range values.

Encoders **SHOULD** always use the smallest encoding for the value being encoded in order to ensure maximum compatibility.

A codec **MAY** choose its own value range restrictions, but **SHOULD** stick with reasonable industry-standard ranges for maximum interoperability, and **MUST** publish what its restrictions are.

Most systems can natively handle:

 * Up to 64 bit floating point values
 * Up to 64 bit integer values (signed or unsigned)

JavaScript in particular can natively handle:

 * Up to 64 bit floating point values
 * Up to 53 bit integer values (plus the sign)



Security Considerations
-----------------------

Any data format that includes length fields is by definition open to abuse. BONJSON decoders **MUST** provide protection against this. For example:

 * Maximum byte lengths (possibly user-configurable) for [big numbers](#big-number) (with sane defaults).
 * Sanity check: Does the length field contain a value greater than the remaining document length (if known)?


### Hardening

JSON is by nature [vulnerable](https://bishopfox.com/blog/json-interoperability-vulnerabilities) to attackers who can take advantage of differences between implementations due to the laxity of the [JSON specification](#json-standards). In order to mitigate such vulnerabilities, BONJSON codecs **SHOULD** implement the following hardening rules:

* Reject documents where a string contains invalid UTF-8 data (invalid encodings, reserved codepoints, surrogate pairs, etc).
* Reject documents containing disallowed values (such as NaN or infinity or values outside of the codec's [allowed range](#value-ranges)).
* Reject documents containing values that are too large for the receiving system to store without data loss.
* Reject documents where an object contains duplicate names (this check **SHOULD** be made _after_ any Unicode normalization).

A codec **MAY** offer user-configurable alternatives to document rejection (such as stringifying long numbers, or replacing bad values with `null`), but the default action **MUST** be to reject the document.



Convenience Considerations
--------------------------

Decoders **SHOULD** offer an option to allow for partial data to be recovered (along with an error condition) when decoding fails partway through.

This would involve discarding any partially decoded value (and its associated object member name - if any), and then artificially terminating all open arrays and objects to produce a well-formed tree.



JSON Standards
--------------

Any discussions about JSON are done within the context of the ECMA and RFC specifications for JSON, and the json.org website:

 * [ECMA-404](https://ecma-international.org/publications-and-standards/standards/ecma-404/)
 * [RFC 8259](https://www.rfc-editor.org/info/rfc8259)
 * [json.org](https://www.json.org)



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
                        header.sig_length > 0: ordered(uint(header.sig_length*8, ~))
                                             & ordered(sint(header.exp_length*8, ~))
                                             ;
                    ]
                  ;
big_number_header = u5(var(sig_length, ~)) & u2(var(exp_length, ~)) & u1(var(sig_negative, ~));

boolean           = true | false;
false             = u8(0x6e);
true              = u8(0x6f);

string            = string_short | string_long;
string_short      = u4(8) & u4(var(count, ~)) & sized(count*8, char_string*);
string_long       = u8(0x68) & char_string* & u8(0xff);

null              = u8(0x6d);

# Primitives & Functions

u1(v)             = uint(1, v);
u2(v)             = uint(2, v);
u3(v)             = uint(3, v);
u4(v)             = uint(4, v);
u5(v)             = uint(5, v);
u8(v)             = uint(8, v);
i8(v)             = sint(8, v);
f16(v)            = bfloat16(v);
f32(v)            = float(32, v);
f64(v)            = float(64, v);

char_string       = '\[0]' ~ '\[10ffff]'; # JSON technically supports unassigned and invalid codepoints

bfloat16(v: bits): bits = """https://en.wikipedia.org/wiki/Bfloat16_floating-point_format""";
```


License
-------

Copyright (c) 2024 Karl Stenerud. All rights reserved.

Distributed under the [Creative Commons Attribution License](https://creativecommons.org/licenses/by/4.0/legalcode) ([license deed](https://creativecommons.org/licenses/by/4.0).
