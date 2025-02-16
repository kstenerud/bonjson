BONJSON: Binary Object Notation for JSON
========================================

** * * * **PRERELEASE** * * * **


BONJSON is a **1:1 compatible** binary serialization format for [JSON](#json-standards).

It's a drop-in replacement that works in the same way and has the same capabilities and limitations as [JSON](#json-standards) (no more, no less), except in a compact and simple-to-process binary format rather than text.

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
  - [Containers](#containers)
    - [Array](#array)
    - [Object](#object)
  - [Boolean](#boolean)
  - [Null](#null)
  - [Full Example](#full-example)
  - [Interoperability Considerations](#interoperability-considerations)
    - [Value Ranges](#value-ranges)
    - [Invalid or Out Of Range Data](#invalid-or-out-of-range-data)
  - [Security Considerations](#security-considerations)
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
| **OPTIONAL**     | An implementation **MAY** choose to implement this, or not.                                      |

-------------------------------------------------------------------------------



Types
-----

BONJSON has the exact same types as [JSON](#json-standards) (and no more):

 * [String](#string-encoding)
 * [Number](#number-encoding)
 * [Array](#array-encoding)
 * [Object](#object-encoding)
 * [Boolean](#boolean-encoding)
 * [Null](#null-encoding)



Structure
---------

BONJSON follows the same structural rules as [JSON](#json-standards), as illustrated in the following railroad diagrams:

**Document**:

    --[value]--

**Value**:

    --+--[string]---+--
      |             |
      +--[number]---+
      |             |
      +--[object]---+
      |             |
      +--[array]----+
      |             |
      +--[boolean]--+
      |             |
      +--[null]-----+

**Object**:

    --[begin object]--+---------------------+--[end container]--
                      |                     |
                      +--[string]--[value]--+
                      |                     |
                      +<<<<<<<<<<<<<<<<<<<<<+

**Array**:

    --[begin array]--+-----------+--[end container]--
                     |           |
                     +--[value]--+
                     |           |
                     +<<<<<<<<<<<+



Encoding
--------

BONJSON is a byte-oriented format. All values begin and end on an 8-bit boundary.

**Notes**:

 * All strings are encoded as UTF-8.
 * All numeric fields are encoded in little endian byte order.


### Type Codes

Every value is composed of an 8-bit type code, and in some cases also a payload:

| Type Code | Payload                      | Type    | Description                                         |
| --------- | ---------------------------- | ------- | --------------------------------------------------- |
| 00 - 6a   |                              | Number  | [Integers 0 through 106](#small-integer)            |
| 6b        | 16-bit bfloat16 binary float | Number  | [16-bit float](#16-bit-float)                       |
| 6c        | 32-bit ieee754 binary float  | Number  | [32-bit float](#32-bit-float)                       |
| 6d        | 64-bit ieee754 binary float  | Number  | [64-bit float](#64-bit-float)                       |
| 6e        |                              | Boolean | [False](#boolean-encoding)                          |
| 6f        |                              | Boolean | [True](#boolean-encoding)                           |
| 70 - 77   | Unsigned integer of n bytes  | Number  | [Unsigned Integer](#integer)                        |
| 78 - 7f   | Signed integer of n bytes    | Number  | [Signed Integer](#integer)                          |
| 80 - 8f   | String of n bytes            | String  | [Short String](#short-string)                       |
| 90        | Arbitrary length string      | String  | [Long String](#long-string)                         |
| 91        | Arbitrary length number      | Number  | [Big Number](#big-number)                           |
| 92        |                              | Array   | [Array start](#array-encoding)                      |
| 93        |                              | Object  | [Object start](#object-encoding)                    |
| 94        |                              |         | [Container end](#container-encoding)                |
| 95        |                              | Null    | [Null](#null-encoding)                              |
| 96 - ff   |                              | Number  | [Integers -106 through -1](#small-integer)          |



Strings
-------

Strings **MUST** be encoded in UTF-8. BONJSON supports the same UTF-8 codepoints as [JSON](#json-standards) does, but does not implement escape sequences (which are unnecessary in a binary format).

Strings can be encoded in two ways:


### Short String

Short strings have their byte length (up to 15) encoded directly into the lower nybble of the type code, and have no terminator byte.

    Type Code
    ---------
    1000 LLLL
         |
         Length (bytes)

**Example**:

    80                                               // ""
    81 41                                            // "A"
    8c e3 81 8a e3 81 af e3 82 88 e3 81 86           // "おはよう"
    8f 31 35 20 62 79 74 65 20 73 74 72 69 6e 67 21  // "15 byte string!"


### Long String

Long strings begin after the [type code](#type-codes) (`0x90`), and are terminated by the byte `0xff`.

#### About the Termination Delimiter

`0xff` was chosen as the string termination delimiter because it's never a valid byte within a UTF-8 sequence (and never will be until we surpass 68 _billion_ codepoints). This also gives the benefit that there's no need to decode the individual UTF-8 codepoints in order to find the end of a string (simply scan the bytes until `0xff` is encountered).

A C implementation for example could use [`memccpy()`](https://www.man7.org/linux/man-pages/man3/memccpy.3.html) and then replace the copied `0xff` with 0 to produce a null-terminated string. Alternatively, if the source buffer is writable, one could overwrite the `0xff` delimiter with 0 to yield a zero-copy null-terminated string using the source buffer as a backing store.

**Example**:

    90 ff                         // ""
    90 61 20 73 74 72 69 6e 67 ff // "a string"



Numbers
-------

Numbers can be encoded using various integer and floating point forms. Encoders **SHOULD** use the most compact representation that stores each value without data loss.

With the exception of [Big Number](#big-number), all numeric types are encoded exactly as they would appear in memory on little endian architectures.

**Note**: Floating point `NaN` and `infinity` values **MUST NOT** be present in a document since they are not allowed in [JSON](#json-standards).


### Small Integer

Small integers (-106 to 106) are encoded into the [type code](#type-codes) itself for maximum compactness in the most commonly used integer range. Casting the [type code](#type-codes) to an 8-bit signed integer yields its value.

**Examples**:

    6a //  106
    05 //    5
    00 //    0
    c4 //  -60
    9c // -100
    96 // -106


### Integer

Integers are encoded in little-endian byte order following the [type code](#type-codes). The type code determines the size of the integer in bytes, and whether it is signed or unsigned.

    Type Code Encoding
    ------------------
    0 1 1 1 S L L L
            | \_|_/
            |   |
            |   Length
            Signed?

**Signed**: 0 = unsigned, 1 = signed

**Length**: Add 1 to this value to give the integer's length in bytes (1-8)

Encoders **SHOULD** favor _signed_ over _unsigned_ when both types would encode a value into the same number of bytes.

**Examples**:

    71 e8 03                   //  1000
    79 18 fc                   // -1000
    72 00 80                   //  0x8000
    75 bc 9a 78 56 34 12       //  0x123456789abc
    7f 00 00 00 00 00 00 00 80 // -0x8000000000000000
    77 da da da de d0 d0 d0 de // 0xded0d0d0dedadada (is all I want to say to you)


### 16-bit Float

16-bit float is encoded as a little-endian 16-bit [bfloat16](https://en.wikipedia.org/wiki/Bfloat16_floating-point_format) following the [type code](#type-codes) (`0x6b`). This is a convenience encoding for a commonly used float type (often used in AI).

**Note**: NaN and infinity values **MUST NOT** be encoded into a BONJSON document.

**Example**:

    6b 90 3f // 1.125


### 32-bit Float

32-bit float is encoded as a little-endian [32-bit ieee754 binary float](https://en.wikipedia.org/wiki/Single-precision_floating-point_format) following the [type code](#type-codes) (`0x6c`). This is a convenience encoding for a commonly used float type.

**Note**: NaN and infinity values **MUST NOT** be encoded into a BONJSON document.

**Example**:

    6c 00 b8 1f 42 // 0x1.3f7p5


### 64-bit Float

64-bit float is encoded as a little-endian [64-bit ieee754 binary float](https://en.wikipedia.org/wiki/Double-precision_floating-point_format) following the [type code](#type-codes) (`0x6d`). This is a convenience encoding for a commonly used float type.

**Note**: NaN and infinity values **MUST NOT** be encoded into a BONJSON document.

**Example**:

    6d 58 39 b4 c8 76 be f3 3f // 1.234


### Big Number

Big Number ([type code](#type-codes) `0x91`) allows for encoding an effectively unlimited range of numbers.

**Note**: This is an **OPTIONAL** type that only exists for 100% compatibility with the theoretical limits of the JSON format, and is unlikely to see much use in the real world (except in closed systems that are prepared to deal with large numbers). A codec **MAY** [reject](#invalid-or-out-of-range-data) this type regardless of its contents.

The structure of a big number is as follows:

    [header] [significand bytes] [exponent bytes]

 * The `header` is encoded as an [unsigned LEB128](https://en.wikipedia.org/wiki/LEB128).
 * The `significand` is encoded as an unsigned integer in little endian byte order.
 * The `exponent` is encoded as a signed integer in little endian byte order.
 * The exponent is a power-of-10, just like in [JSON](#json-standards) text notation.

The `header` consists of 3 fields:

    [significand length] [exponent length] [sign]

 * The lowest bit is the "negative" bit (0 = positive significand, 1 = negative significand).
 * The next 2 bits represent the `exponent length` (0-3 bytes).
 * The rest of the header represents the `significand length` in bytes.

```
Big Number Header
-----------------------
S S S S S S S ... E E N
\               / | |  \
 \             /  |/    negative
   significand    exponent
     length       length
```

This allows for an unlimited significand size, and a ludicrous exponent range of ± 8 million.

The value of the big number is derived as: `significand` × `sign` × 10^`exponent`

**Notes**:

 * A field length of 0 represents a value of 0 for that field.
 * A Big Number with a significand value of 0 is equal to 0 with the corresponding sign, regardless of exponent contents (which are superfluous in such a case).

**Examples**:

    91 48 00 10 32 54 76 98 ba dc fe     // 0xfedcba987654321000 (9 bytes significand, no exponent, positive)
    91 0a 0f ff                          // 1.5 (15 x 10⁻¹) (1 byte significand, 1 byte exponent, positive)
    91 01                                // -0 (no significand, no exponent, negative)
    91 8d 01 97 EB F2 0E C3 98 06 C1 47
       71 5E 65 4F 58 5F AA 28 d8 dc     // -13837758495464977165497261864967377972119 x 10⁻⁹⁰⁰⁰
                                         // (17 bytes significand, 2 bytes exponent, negative)



Containers
----------

Containers (objects and arrays) are encoded beginning with a `container start` code, and ending with a `container end` code. Both [object](#object) and [array](#array) share the same `container end` [type code](#type-codes) (`0x94`).


### Array

An array consists of an `array start` (`0x92`), an optional collection of values, and finally a `container end` (`0x94`):

    [array start] (value ...) [container end]
         0x92         ...          0x94

**Example**:

    92        // [
        81 61 //     "a",
        01    //     1
        96    //     null
    94        // ]


### Object

An object consists of an `object start` (`0x93`), an optional collection of name-value pairs, and finally a `container end` (`0x94`):

    [object start] (name+value ...) [container end]
         0x93            ...            0x94

**Note**: Names **MUST** be strings and **MUST NOT** be null.

**Example**:

    93                  // {
        81 62           //     "b":
        00              //     0,
        84 74 65 73 74  //     "test":
        81 78           //     "x"
    94                  // }



Boolean
-------

Boolean values are encoded into the [type codes](#type-codes) themselves:

 * False has type code `0x6e`
 * True has type code `0x6f`



Null
----

Null has [type code](#type-codes) `0x95`.



Full Example
------------

**JSON**:

    {
      "number":1,
      "null":null,
      "boolean":true,
      "array":[
        "x",
        1000,
        1.5
      ],
      "object":{
        "negative number":-100,
        "long string":"1234567890123456789012345678901234567890"
      }
    }

    Size:     200 bytes
    Minified: 153 bytes

**BONJSON**:

    93                                                       // {
        86 6e 75 6d 62 65 72                                 //     "number":
        01                                                   //     1,
        84 6e 75 6c 6c                                       //     "null":
        95                                                   //     null,
        87 62 6f 6f 6c 65 61 6e                              //     "boolean":
        6f                                                   //     true,
        85 61 72 72 61 79                                    //     "array"
        92                                                   //     [
            81 78                                            //         "x",
            79 e8 03                                         //         1000,
            6b c0 3f                                         //         1.5
        94                                                   //     ],
        86 6f 62 6a 65 63 74                                 //     "object":
        93                                                   //     {
            8f 6e 65 67 61 74 69 76 65 20 6e 75 6d 62 65 72  //         "negative number":
            9c                                               //         -100,
            8b 6c 6f 6e 67 20 73 74 72 69 6e 67              //         "long string":
            90                                               //         "1234567890123456789012345678901234567890"
                31 32 33 34 35 36 37 38 39 30                //
                31 32 33 34 35 36 37 38 39 30                //
                31 32 33 34 35 36 37 38 39 30                //
                31 32 33 34 35 36 37 38 39 30                //
            ff                                               //
        94                                                   //     }
    94                                                       // }

    Size:    121 bytes



Interoperability Considerations
-------------------------------

Because [JSON](#json-standards) is so weakly specified, there are numerous ways in which one implementation can become incompatible with another. [RFC 8259](https://www.rfc-editor.org/info/rfc8259) discusses many of these issues and how to mitigate them. BONJSON implementations **SHOULD** follow their advice.


### Value Ranges

Although JSON allows an unlimited range for most values, it's important to take into consideration the limitations of the systems that will be trying to ingest your data, and how they're likely to deal with out-of-range values.

Encoders **SHOULD** always use the smallest encoding for the value being encoded in order to ensure maximum compatibility.

A decoder **MAY** choose its own value range restrictions, but **SHOULD** stick with reasonable industry-standard ranges for maximum interoperability.

Most systems can handle:

 * Up to 64 bit floating point values
 * Up to 64 bit integer values (signed or unsigned)

Javascript in particular can handle:

 * Up to 64 bit floating point values
 * Up to 53 bit integer values (plus the sign)


### Invalid or Out Of Range Data

It's expected that your codec will at some point encounter disallowed data:

 * Values that are outright invalid, such as floating point `NaN` or `infinity`
 * Values that are too large to be ingested by the receiving system
 * Optional types that are not supported

Codecs **SHOULD** offer the user options for what to do when these situations occur (ideally, separate options for "invalid" and for "out of range" values):

 * Abort processing
 * Stringify the value (for example to a decimal or hexadecimal string)
 * Replace the value with `null`

Codec documentation **MUST** explain what behaviors they offer, and what are the defaults. For safety and security reasons, the default behavior **SHOULD** be to abort processing.



Security Considerations
-----------------------

Any data format that includes length fields is by definition open to abuse. BONJSON decoders **MUST** provide protection against this. For example:

 * Maximum byte lengths (possibly user-configurable) for [big numbers](#big-number) (with sane defaults).
 * Sanity check: Does the length field contain a value greater than the remaining document length (if known)?



Convenience Considerations
--------------------------

Decoders **SHOULD** offer an option to allow for partial data to be recovered (along with an error condition) when decoding fails partway through.

This would involve discarding any partially decoded value (and its associated member name - if any), and then artificially terminating all open arrays and objects to produce a well-formed tree.



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

array             = u8(0x92) & value* & end_container;
object            = u8(0x93) & (string & value)* & end_container;
end_container     = u8(0x94);

number            = int_small | uint | sint | float_16 | float_32 | float_64 | big_number;
int_small         = u8(0x00~0x6a) | u8(0x96~0xff);
uint              = u4(7) & u1(0) & u3(var(count, ~)) & ordered(uint((count+1)*8, ~));
sint              = u4(7) & u1(1) & u3(var(count, ~)) & ordered(sint((count+1)*8, ~));
float_16          = u8(0x6b) & ordered(f16(~));
float_32          = u8(0x6c) & ordered(f32(~));
float_64          = u8(0x6d) & ordered(f64(~));
big_number        = u8(0x91)
                  & var(header, big_number_header)
                  & ordered(uint(header.sig_length*8, ~))
                  & ordered(sint(header.exp_length*8, ~))
                  ;
big_number_header = uleb128(uany(var(sig_length, ~)) & u2(var(exp_length, ~) & u1(var(sig_negative, ~)));

boolean           = true | false;
false             = u8(0x6e);
true              = u8(0x6f);

string            = string_short | string_long;
string_short      = u4(8) & u4(var(count, ~)) & sized(count*8, char_string*);
string_long       = u8(0x90) & char_string* & u8(0xff);

null              = u8(0x95);

# Primitives & Functions

u1(v)             = uint(1, v);
u2(v)             = uint(2, v);
u3(v)             = uint(3, v);
u4(v)             = uint(4, v);
u8(v)             = uint(8, v);
uany(v)           = uint(~,v);
f16(v)            = bfloat16(v);
f32(v)            = float(32, v);
f64(v)            = float(64, v);

char_string       = '\[0]' ~ '\[10ffff]'; # JSON technically supports unassigned and invalid codepoints

bfloat16(v: bits): bits = """https://en.wikipedia.org/wiki/Bfloat16_floating-point_format""";
uleb128(v: bits): bits = """https://en.wikipedia.org/wiki/LEB128#Unsigned_LEB128""";
```


License
-------

Copyright (c) 2024 Karl Stenerud. All rights reserved.

Distributed under the [Creative Commons Attribution License](https://creativecommons.org/licenses/by/4.0/legalcode) ([license deed](https://creativecommons.org/licenses/by/4.0).
