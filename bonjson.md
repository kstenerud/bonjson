BONJSON: Binary Object Notation for JSON
========================================

** * * * **PRERELEASE** * * * **


BONJSON is a **1:1 compatible** binary serialization format for [JSON](#json-standards).

It works in the same way and has the same capabilities and limitations as [JSON](#json-standards), except in binary rather than text.


### Why build this?

[JSON](#json-standards) isn't going away for a **loooooooong** time, so let's at least make it less wasteful where we can! BONJSON documents are quicker and more energy efficent to process, and are generally smaller and compress better than [JSON](#json-standards). Machine-to-machine transmission is better done in binary, with conversion to human-readable JSON only at endpoints where a human is actually involved (or via debugging tools along the path).

BONJSON is a drop-in replacement for [JSON](#json-standards), not an actually better format (if you want better, you might consider [Concise Encoding](https://concise-encoding.org/)). Structurally and logically, BONJSON works in exactly the same way as [JSON](#json-standards), suffering from all of the same problems (lack of types, weak specification, undefined edge cases, etc), **but** it can also benefit from the existing JSON ecosystem.

_The world has settled on JSON, so here we are._


### Why not BSON or anther existing binary JSON format?

Because they _all_ add extra features and gimmicks or special restrictions that make them **NOT** 1:1 compatible.

**Wherever there's a compatibility mismatch, breakage will eventually occur** - it's only a matter of time before your complex data pipelines trigger it. Having confidence in your data pipeline is paramount.

**1:1 compatible means that**:

 * Any valid document in the binary format **MUST** be 100% convertible to [JSON](#json-standards) format without data loss or schema requirement.
 * Any valid document in [JSON](#json-standards) format **MUST** be 100% convertible to the binary format without data loss or schema requirement.
 * Any valid document **MUST** be round-trip convertible between [JSON](#json-standards) and the binary format (in either direction) without data loss or schema requirement.
 * The binary format **MUST** support the same feature set as [JSON](#json-standards) does. For example, [JSON](#json-standards) supports progressive encoding.

A "binary" version of [JSON](#json-standards) **MUST** behave in exactly the same way as [JSON](#json-standards) (inasmuch as is possible with such a weakly specified format). The _only_ difference should be in the encoding mechanism.

_This_ is what BONJSON is.

-------------------------------------------------------------------------------

Contents
--------

- [BONJSON: Binary Object Notation for JSON](#bonjson-binary-object-notation-for-json)
    - [Why build this?](#why-build-this)
    - [Why not BSON or anther existing binary JSON format?](#why-not-bson-or-anther-existing-binary-json-format)
  - [Contents](#contents)
  - [Terms and Conventions](#terms-and-conventions)
  - [Types](#types)
  - [Structure](#structure)
    - [Document](#document)
    - [Value](#value)
    - [Object](#object)
    - [Array](#array)
  - [Encoding](#encoding)
    - [Type Codes](#type-codes)
  - [String Encoding](#string-encoding)
    - [Short Form](#short-form)
    - [Long Form](#long-form)
      - [Chunk Header](#chunk-header)
      - [Chunk Header Example](#chunk-header-example)
  - [Number Encoding](#number-encoding)
    - [Small Integer](#small-integer)
    - [16-bit Signed Integer](#16-bit-signed-integer)
    - [32-bit Float](#32-bit-float)
    - [64-bit Float](#64-bit-float)
    - [Big Number](#big-number)
  - [Container Encoding](#container-encoding)
    - [Array Encoding](#array-encoding)
    - [Object Encoding](#object-encoding)
  - [Boolean Encoding](#boolean-encoding)
  - [Null Encoding](#null-encoding)
  - [Full Example](#full-example)
  - [Interoperability Considerations](#interoperability-considerations)
  - [Security Considerations](#security-considerations)
  - [Convenience Considerations](#convenience-considerations)
  - [JSON Standards](#json-standards)
  - [Formal BONJSON Grammar](#formal-bonjson-grammar)
  - [License](#license)


Terms and Conventions
---------------------

**The following bolded, capitalized terms have specific meanings in this document**:

| Term             | Meaning                                                                                                               |
| ---------------- | --------------------------------------------------------------------------------------------------------------------- |
| **MUST (NOT)**   | If this directive is not adhered to, the document or implementation is invalid.                                       |
| **SHOULD (NOT)** | Every effort should be made to follow this directive, but the document/implementation is still valid if not followed. |
| **MAY (NOT)**    | It is up to the implementation to decide whether to do something or not.                                              |
| **CAN**          | Refers to a possibility which **MUST** be accommodated by the implementation.                                         |
| **CANNOT**       | Refers to a situation which **MUST NOT** be allowed by the implementation.                                            |

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

BONJSON follows the same structural rules as [JSON](#json-standards), as per the following railroad diagrams:

### Document

    --[value]--

### Value

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

### Object

    --[begin object]--+---------------------+--[end container]--
                      |                     |
                      +--[string]--[value]--+
                      |                     |
                      +<<<<<<<<<<<<<<<<<<<<<+

### Array

    --[begin array]--+-----------+--[end container]--
                     |           |
                     +--[value]--+
                     |           |
                     +<<<<<<<<<<<+



Encoding
--------

BONJSON is a byte-oriented format. All values begin and end on an 8-bit boundary.

**Notes**:

 * Strings are always encoded as UTF-8.
 * All numeric fields are in little endian byte order.


### Type Codes

Every value is composed of an 8-bit type code and in some cases a payload:

| Type Code | Payload                     | Type    | Description                                |
| --------- | --------------------------- | ------- | ------------------------------------------ |
| 00 - 69   |                             | Number  | [Integers 0 through 105](#small-integer)   |
| 6a        |                             | Null    | [Null](#null-encoding)                     |
| 6b        | 16-bit signed integer       | Number  | [16-bit integer](#16-bit-signed-integer)   |
| 6c        | 32-bit ieee754 binary float | Number  | [32-bit float](#32-bit-float)              |
| 6d        | 64-bit ieee754 binary float | Number  | [64-bit float](#64-bit-float)              |
| 6e        | Big Number                  | Number  | [Big Number (positive)](#big-number)       |
| 6f        | Big Number                  | Number  | [Big Number (negative)](#big-number)       |
| 70 - 8f   | 0-31 bytes of UTF-8 data    | String  | [String (short form)](#short-form)         |
| 90        | String chunk list           | String  | [String (long form)](#long-form)           |
| 91        |                             | Array   | [Array start](#array-encoding)             |
| 92        |                             | Object  | [Object start](#object-encoding)           |
| 93        |                             |         | [Container end](#container-encoding)       |
| 94        |                             | Boolean | [False](#boolean-encoding)                 |
| 95        |                             | Boolean | [True](#boolean-encoding)                  |
| 96 - ff   |                             | Number  | [Integers -106 through -1](#small-integer) |



String Encoding
---------------

Strings **MUST** be encoded in UTF-8. BONJSON supports the same UTF-8 codepoints as [JSON](#json-standards) does, but does not implement escape sequences (which are unnecessary in a binary format).

Strings can be encoded in short or long form. Encoders **SHOULD** store string values that are less than 32 bytes long using the short form.

### Short Form

In short form strings, the length is encoded into the [type code](#type-codes) itself. Strings that are up to 31 **bytes** (_not_ characters) long can be stored using [type codes](#type-codes) 0x70-0x8f. Subtracting 0x70 from the [type code](#type-codes) derives the string length in bytes.

### Long Form

Long form strings are stored in chunks, where sequences of string data are preceded by [chunk headers](#chunk-header).

    --+--[chunk header]--[string bytes]--+--
      |                                  |
      +<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<+

Most commonly, you'd store the entire string in a single chunk.

#### Chunk Header

Each string chunk is preceded by a chunk header, which contains a `length` and a `continuation bit`. When the `continuation bit` is 1, there's at least one more chunk to decode for the current string.

    --+--[length+continuation]--[string bytes]--+--
      |                                         |
      +<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<+

The `continuation bit` is the **lowest** bit in the chunk header. The upper bits of the header contain the `length` field.

    L L L L L L L L L ... C

The combined `chunk header` is then encoded as [unsigned LEB128](https://en.wikipedia.org/wiki/LEB128).

**Note**: You can terminate any unterminated chunked string by encoding the chunk header `0x00` (length 0, continuation 0).

The benefit of chunked encoding is that strings can be encoded progressively, for example if the final string length wasn't yet known when encoding started:

    [length 500, continuation 1] [500 bytes of string data]  // Initial available data from some buffer
    [length 153, continuation 1] [153 bytes of string data]  // Some more data becomes available
    [length 281, continuation 0] [281 bytes of string data]  // Finally, we got the last of it
    -------------------------------------------------------
    String of length 934 bytes, encoded in 3 chunks

Alternatively:

    [length 500, continuation 1] [500 bytes of string data]  // Initial available data from some buffer
    [length 153, continuation 1] [153 bytes of string data]  // Some more data becomes available
    [length 281, continuation 1] [281 bytes of string data]  // Even more data... Where will it end?
    [length 0, continuation 0]                               // Oh! We're actually at the end of the string.
    -------------------------------------------------------
    String of length 934 bytes, encoded in 4 chunks

#### Chunk Header Example

In this example, we decode a chunk header that has been encoded into the 2-byte ULEB128 sequence: `0xa1 0x1f`

First, extract the 14 bit payload from its 16 bit ULEB128 envelope (every 8 bits of ULEB128 data contains 7 bits of payload):

    hex: 0xa1       0x1f
    bin: 1 0100001  0 0011111
           payload    payload

ULEB128 is little endian ordered, so the 7-bit payload segments are actually swapped.

    0011111 0100001

The decoded `chunk header` looks like this: `00111110100001`

The lowest bit is the `continuation bit`. The rest of the bits comprise the `length field`.

    L L L L L L L L L L L L L C
    0 0 1 1 1 1 1 0 1 0 0 0 0 1

The `length field` is `0011111010000` = 2000 bytes

The `continuation bit` is `1` (meaning that there will be another chunk header after the 2000 bytes of string data)



Number Encoding
---------------

Numbers can be encoded using various integer and floating point forms. Encoders **SHOULD** use the most compact representation that stores each value without data loss.

**Note**: Floating point `NaN` and `infinity` values **MUST NOT** be present in a document!

### Small Integer

Small integers are encoded into the [type code](#type-codes) itself for maximum compactness in the most commonly used integer range. The [type code](#type-codes) can be directly cast to a signed 8-bit integer to produce the correct signed integer value.

[Type codes](#type-codes) 0x00 to 0x69 encode values from 0 to 105, and [type codes](#type-codes) 0x96 to 0xff encode values from -106 to -1.

### 16-bit Signed Integer

The value is encoded as a little-endian 16-bit signed integer following the [type code](#type-codes). This encoding is included for compactness in a commonly used integer range.

### 32-bit Float

The value is encoded as a little-endian 32-bit ieee754 binary float following the [type code](#type-codes). This is a convenience encoding for a commonly used float type.

### 64-bit Float

The value is encoded as a little-endian 64-bit ieee754 binary float following the [type code](#type-codes). This is a convenience encoding for a commonly used float type.

### Big Number

The Big Number type allows for encoding an effectively unlimited range of numbers. It can encode small floats (up to 2 significant digits and in some cases 3) in even less space than a float32.

The general, logical form of a big number is as follows:

    [sign] [length header] [significand bytes] [exponent bytes]

 * The `sign` is encoded into the [type code](#type-codes) itself.
 * The `length header` field is encoded as an [unsigned LEB128](https://en.wikipedia.org/wiki/LEB128).
 * The `significand` is encoded as an unsigned integer in little endian byte order.
 * The `exponent` (if present) is encoded as a 1, 2, or 3 byte [zigzag integer](https://en.wikipedia.org/wiki/Variable-length_quantity#Zigzag_encoding) in little endian byte order.
 * The exponent is a power-of-10, just like in [JSON](#json-standards) text notation.

The length header consists of two fields:

    [significand length] [exponent length]

The lower 2 bits of the length header represents the `exponent length` (0-3 bytes). The rest of the length header is the `significand length` in bytes.

    S S S S S S S ... E E

This allows for an unlimited significand size, and a ludicrous exponent range of ± 8 million.

**Note**: A Big Number with a significand length of 0 and an exponent length of 0 is equal to 0 with the sign from the [type code](#type-codes).

**Examples**:

    6e 20 ff ff ff ff ff ff ff ff    // 0xffffffffffffffff (8 bytes significand, no exponent)
    6e 05 0f 01                      // 1.5 (15 x 10⁻¹) (1 byte significand, 1 byte exponent)
    6f 00                            // -0 (no significand, no exponent)
    6f 46 97 EB F2 0E C3 98 06 C1 47
       71 5E 65 4F 58 5F AA 28 4f 46 // -13837758495464977165497261864967377972119 x 10⁻⁹⁰⁰⁰
                                     // (17 bytes significand, 2 bytes exponent)



Container Encoding
------------------

Containers (objects and arrays) are encoded beginning with a container start code, and ending with a container end code. Both `object` and `array` share the same `container end` [type code](#type-codes) (0x93).

### Array Encoding

An array consists of an `array start`, an optional collection of values, and finally a `container end`:

    [array start] (value ...) [container end]
         0x91         ...          0x93

**Example**:

    91                  // [
        71 61           //     "a",
        01              //     1
    93                  // ]

### Object Encoding

An object consists of an `object start`, an optional collection of name-value pairs, and finally a `container end`:

    [object start] (name+value ...) [container end]
         0x92            ...            0x93

**Note**: Names **MUST** be strings and **MUST NOT** be null.

**Example**:

    92                  // {
        71 62           //     "b":
        00              //     0,
        74 74 65 73 74  //     "test":
        71 78           //     "x"
    93                  // }



Boolean Encoding
----------------

Boolean values are encoded into the [type codes](#type-codes) themselves:

 * False is type code `0x94`
 * True is type code `0x95`



Null Encoding
-------------

Null is encoded into the [type code](#type-codes) `0x6a`.


Full Example
------------

**JSON**:

    {
        "a number": 1,
        "an array": ["x", 1000, 1.5],
        "a null": null,
        "a boolean": true,
        "an object": {
            "a": -100,
            "b": "........................................"
        }
    }

    Size:     200 bytes
    Minified: 141 bytes
    GZipped:  121 bytes


**BONJSON**:

    92                                     // {
        78 61 20 6e 75 6d 62 65 72         //     "a number":
        01                                 //     1,
        78 61 6e 20 61 72 72 61 79         //     "an array":
        91                                 //     [
            71 78                          //         "x",
            6b e8 03                       //         1000,
            6f 05 0f 01                    //         1.5
        93                                 //     ],
        76 61 20 6e 75 6c 6c               //     "a null":
        6a                                 //     null,
        79 61 20 62 6f 6f 6c 65 61 6e      //     "a boolean":
        95                                 //     true,
        79 61 6e 20 6f 62 6a 65 63 74      //     "an object":
        92                                 //     {
            71 61                          //         "a":
            9c                             //         -100,
            71 62                          //         "b":
            90 50 2e 2e 2e 2e 2e 2e 2e 2e
                  2e 2e 2e 2e 2e 2e 2e 2e
                  2e 2e 2e 2e 2e 2e 2e 2e
                  2e 2e 2e 2e 2e 2e 2e 2e
                  2e 2e 2e 2e 2e 2e 2e 2e  //         "........................................"
        93                                 //     }
    93                                     // }

    Size:    110 bytes
    GZipped:  99 bytes



Interoperability Considerations
-------------------------------

Because [JSON](#json-standards) is so weakly specified, there are numerous ways in which one implementation can become incompatible with another. [RFC 8259](https://www.rfc-editor.org/info/rfc8259) discusses many of these issues and how to mitigate them. It is recommended that any BONJSON implementation also follow their advice.



Security Considerations
-----------------------

Any format that includes unbounded length fields is by definition open to abuse. BONJSON decoders **MUST** provide ways to protect against this. For example:

 * User-configurable maximum byte lengths for numbers and for strings (with sane defaults).
 * Sanity check: Does the length field contain a value greater than the total document length (if known)?



Convenience Considerations
--------------------------

Decoders **SHOULD** allow for partial data to be recovered (along with an error condition) when decoding fails partway through. This would involve discarding any partially decoded value (and its associated member name - if any), and then artificially terminating all open arrays and objects to produce a well-formed tree.



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

value             = string | number | array | object | boolean | null;

# Types

string            = string_short | string_long;
string_short      = u8(var(count, 0x70~0x8f)) & sized((count-0x70)*8, char_string*);
string_long       = u8(0x90) & string_chunk;
string_chunk      = var(header, chunk_header)
                  & sized(header.count*8, char_string*)
                  & [header.continuation = 1: string_chunk;]
                  ;
chunk_header      = uleb128(uany(var(count, ~)) & u1(var(continuation, ~)));

number            = int_small | int_16 | float_32 | float_64 | big_number;
int_small         = s8(-106~105);
int_16            = u8(0x6b) & s16(~);
float_32          = u8(0x6c) & f32(~);
float_64          = u8(0x6d) & f64(~);
big_number        = big_number_pos | big_number_neg;
big_number_pos    = u8(0x6e) & big_number_value;
big_number_neg    = u8(0x6f) & big_number_value;
big_number_value  = var(header, big_number_header)
                  & ordered(uint(header.sig_length*8, ~))
                  & ordered(zigzag(uint(header.exp_length*8, ~)))
                  ;
big_number_header = uleb128(uany(var(sig_length, ~)) & u2(var(exp_length, ~)));

array             = u8(0x91) & value* & end_container;
object            = u8(0x92) & (name & value)* & end_container;
end_container     = u8(0x93);
name              = string;

boolean           = true | false;
false             = u8(0x94);
true              = u8(0x95);

null              = u8(0x6a);

# Primitives & Functions

s8(v)             = sint(8, v);
s16(v)            = ordered(sint(16, v));
u1(v)             = uint(1, v);
u2(v)             = uint(2, v);
u8(v)             = uint(8, v);
uany(v)           = uint(~,v);
f32(v)            = ordered(float(32, v));
f64(v)            = ordered(float(64, v));

char_string       = '\[0]' ~ '\[10ffff]'; # JSON technically supports unassigned and invalid codepoints

uleb128(v: bits): bits = """https://en.wikipedia.org/wiki/LEB128#Unsigned_LEB128""";
zigzag(v: bits): bits  = """https://en.wikipedia.org/wiki/Variable-length_quantity#Zigzag_encoding"""
```


License
-------

Copyright (c) 2024 Karl Stenerud. All rights reserved.

Distributed under the [Creative Commons Attribution License](https://creativecommons.org/licenses/by/4.0/legalcode) ([license deed](https://creativecommons.org/licenses/by/4.0).
