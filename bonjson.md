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


### Why use binary at all?

A simple binary format is orders of magnitude faster to produce and consume compared to a text format. It also offers much smaller sizes for number-heavy data (text-heavy data will be about on-par with JSON in terms of size).

**The average progression is**:

 * **When starting something new:** JSON, because it's simple and ubiquitous.
 * **As your costs begin to rise:** BONJSON, because it's a drop-in replacement for JSON that's less expensive in processing and transmission.
 * **As your needs expand beyond basic data:** A more advanced format specific to your use case.

-------------------------------------------------------------------------------

Contents
--------

- [BONJSON: Binary Object Notation for JSON](#bonjson-binary-object-notation-for-json)
    - [Why build this?](#why-build-this)
    - [Why not BSON or anther existing binary JSON format?](#why-not-bson-or-anther-existing-binary-json-format)
    - [Why use binary at all?](#why-use-binary-at-all)
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
  - [Number Encoding](#number-encoding)
    - [Small Integer](#small-integer)
    - [8-bit Signed Integer](#8-bit-signed-integer)
    - [Signed Integer](#signed-integer)
    - [64-bit Unsigned Integer](#64-bit-unsigned-integer)
    - [16-bit Float](#16-bit-float)
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

| Type Code | Payload                      | Type    | Description                                         |
| --------- | ---------------------------- | ------- | --------------------------------------------------- |
| 00 - ea   |                              | Number  | [Integers -117 through 117](#small-integer)         |
| eb        |                              | Array   | [Array start](#array-encoding)                      |
| ec        |                              | Object  | [Object start](#object-encoding)                    |
| ed        |                              |         | [Container end](#container-encoding)                |
| ee        |                              | Boolean | [False](#boolean-encoding)                          |
| ef        |                              | Boolean | [True](#boolean-encoding)                           |
| f0        |                              | Null    | [Null](#null-encoding)                              |
| f1        | signed integer               | Number  | [8-bit signed integer](#8-bit-signed-integer)       |
| f2 - f8   | 16 to 64 bit signed integer  | Number  | [signed integer](#signed-integer)                   |
| f9        | 64-bit unsigned integer      | Number  | [64-bit unsigned integer](#64-bit-unsigned-integer) |
| fa        | Big Number                   | Number  | [Positive Big Number](#big-number)                  |
| fb        | Big Number                   | Number  | [Negative Big Number](#big-number)                  |
| fc        | 16-bit bfloat16 binary float | Number  | [16-bit float](#16-bit-float)                       |
| fd        | 32-bit ieee754 binary float  | Number  | [32-bit float](#32-bit-float)                       |
| fe        | 64-bit ieee754 binary float  | Number  | [64-bit float](#64-bit-float)                       |
| ff        | String                       | String  | [String](#string-encoding)                          |



String Encoding
---------------

Strings are sequences of UTF-8 characters delimited on both ends by the byte `0xff`. Since `0xff` is never a valid byte within a UTF-8 sequence, there is no need to interpret the UTF-8 characters themselves while scanning for the end of the string (A C/C++ implementation could use `memccpy()`, for example).

Strings **MUST** be encoded in UTF-8. BONJSON supports the same UTF-8 codepoints as [JSON](#json-standards) does, but does not implement escape sequences (which are unnecessary in a binary format).

**Example**:

    ff 61 20 73 74 72 69 6e 67 ff // "a string"



Number Encoding
---------------

Numbers can be encoded using various integer and floating point forms. Encoders **SHOULD** use the most compact representation that stores each value without data loss.

[Small integer](#small-integer) and [8-bit integer](#8-bit-signed-integer) have special encodings. All other numeric types are encoded exactly as the numbers they represent.

**Note**: Floating point `NaN` and `infinity` values **MUST NOT** be present in a document!

### Small Integer

Small integers (-117 to 117) are encoded into the [type code](#type-codes) itself for maximum compactness in the most commonly used integer range. Subtracting the bias value 117 from the [type code](#type-codes) gives the actual value it represents.

**Examples**:

    ea //  117
    7a //    5
    75 //    0
    39 //  -60
    11 // -100
    00 // -117


### 8-bit Signed Integer

The 8 bit signed integer takes over where [small integer](#small-integer) leaves off, covering integer ranges from 118 to 245, and from -118 to -245 within a single byte value.

**Encoding**:

 * Integers from 118 to 245 have 118 subtracted from them.
 * Integers from -118 to -245 have 117 added to them.

**Decoding**:

 * Encoded values >= 0 have 118 added to them.
 * Encoded values < 0 have 117 subtracted from them.

**Examples**:

    f1 02 //  120
    f1 7f //  245
    f1 ff // -118
    f1 80 // -245

### Signed Integer

The value is encoded as a little-endian signed integer following the [type code](#type-codes). The lower nybble of the [type code](#type-codes) selects how many bytes (2-8) of little endian integer data follows.

**Examples**:

    f2 e8 03                   //  1000
    f2 18 fc                   // -1000
    f3 00 80 00                //  0x8000
    f6 bc 7a 67 56 34 12       //  0x123456789abc
    f8 00 00 00 00 00 00 00 80 // -0x8000000000000000

### 64-bit Unsigned Integer

The value is encoded as a little-endian 64-bit unsigned integer following the [type code](#type-codes). This is a convenience encoding for a commonly used integer type.

**Example**:

    f9 ff ff ff ff ff ff ff ff // 0xffffffffffffffff

### 16-bit Float

The value is encoded as a little-endian 16-bit [bfloat16](https://en.wikipedia.org/wiki/Bfloat16_floating-point_format) following the [type code](#type-codes). This is a convenience encoding for a commonly used float type (often used in AI).

**Example**:

    fc 90 3f // 1.125

### 32-bit Float

The value is encoded as a little-endian [32-bit ieee754 binary float](https://en.wikipedia.org/wiki/Single-precision_floating-point_format) following the [type code](#type-codes). This is a convenience encoding for a commonly used float type.

**Example**:

    fd 00 b8 1f 42 // 0x1.3f7p5

### 64-bit Float

The value is encoded as a little-endian [64-bit ieee754 binary float](https://en.wikipedia.org/wiki/Double-precision_floating-point_format) following the [type code](#type-codes). This is a convenience encoding for a commonly used float type.

**Example**:

    fe 58 39 b4 c8 76 be f3 3f // 1.234

### Big Number

The Big Number type allows for encoding an effectively unlimited range of numbers.

It's the most complicated encoding, but it's also the least likely to actually be used in real-world data (mostly, it exists to bring parity with JSON's unlimited number range).

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

**Note**: A Big Number with a significand length of 0 or value 0 is equal to 0 with the sign from the [type code](#type-codes), regardless of exponent contents.

**Examples**:

    fa 20 ff ff ff ff ff ff ff ff    // 0xffffffffffffffff (8 bytes significand, no exponent)
    fa 05 0f 01                      // 1.5 (15 x 10⁻¹) (1 byte significand, 1 byte exponent)
    fb 00                            // -0 (no significand, no exponent)
    fb 46 97 EB F2 0E C3 98 06 C1 47
       71 5E 65 4F 58 5F AA 28 4f 46 // -13837758495464977165497261864967377972119 x 10⁻⁹⁰⁰⁰
                                     // (17 bytes significand, 2 bytes exponent)



Container Encoding
------------------

Containers (objects and arrays) are encoded beginning with a container start code, and ending with a container end code. Both `object` and `array` share the same `container end` [type code](#type-codes) (0xed).

### Array Encoding

An array consists of an `array start`, an optional collection of values, and finally a `container end`:

    [array start] (value ...) [container end]
         0xeb         ...          0xed

**Example**:

    eb                  // [
        ff 61 ff        //     "a",
        76              //     1
        f0              //     null
    ed                  // ]

### Object Encoding

An object consists of an `object start`, an optional collection of name-value pairs, and finally a `container end`:

    [object start] (name+value ...) [container end]
         0xec            ...            0xed

**Note**: Names **MUST** be strings and **MUST NOT** be null.

**Example**:

    ec                     // {
        ff 62 ff           //     "b":
        75                 //     0,
        ff 74 65 73 74 ff  //     "test":
        ff 78 ff           //     "x"
    ed                     // }



Boolean Encoding
----------------

Boolean values are encoded into the [type codes](#type-codes) themselves:

 * False is type code `0xee`
 * True is type code `0xef`



Null Encoding
-------------

Null is encoded into the [type code](#type-codes) `0xf0`.



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

    ec                                     // {
        ff 61 20 6e 75 6d 62 65 72 ff      //     "a number":
        76                                 //     1,
        ff 61 6e 20 61 72 72 61 79 ff      //     "an array":
        eb                                 //     [
            ff 78 ff                       //         "x",
            f2 e8 03                       //         1000,
            fc c0 3f                       //         1.5
        ed                                 //     ],
        ff 61 20 6e 75 6c 6c ff            //     "a null":
        f0                                 //     null,
        ff 61 20 62 6f 6f 6c 65 61 6e ff   //     "a boolean":
        ef                                 //     true,
        ff 61 6e 20 6f 62 6a 65 63 74 ff   //     "an object":
        ec                                 //     {
            ff 61 ff                       //         "a":
            11                             //         -100,
            ff 62 ff                       //         "b":
            ff 2e 2e 2e 2e 2e 2e 2e 2e
               2e 2e 2e 2e 2e 2e 2e 2e
               2e 2e 2e 2e 2e 2e 2e 2e
               2e 2e 2e 2e 2e 2e 2e 2e
               2e 2e 2e 2e 2e 2e 2e 2e
            ff                             //         "........................................"
        ed                                 //     }
    ed                                     // }

    Size:    117 bytes
    GZipped: 113 bytes



Interoperability Considerations
-------------------------------

Because [JSON](#json-standards) is so weakly specified, there are numerous ways in which one implementation can become incompatible with another. [RFC 8259](https://www.rfc-editor.org/info/rfc8259) discusses many of these issues and how to mitigate them. BONJSON implementations **SHOULD** follow their advice.

It's expected that your decoder will eventually encounter disallowed data due to the [GIGO effect](https://en.wikipedia.org/wiki/Garbage_in,_garbage_out) (for example float values containing NaN or infinity, numbers that are too large, etc). Decoders **SHOULD** offer the user options for what to do when that happens:

 * Abort processing
 * Stringify the value
 * Replace the value with `null`

If a decoder provides such optional behavior, it **MUST** default to aborting. If a decoder provides no such optional behavior, it **MUST** abort on invalid/disallowed data.



Security Considerations
-----------------------

Any format that includes unbounded length fields is by definition open to abuse. BONJSON decoders **MUST** provide ways to protect against this. For example:

 * User-configurable maximum byte lengths for [big numbers](#big-number) (with sane defaults).
 * Sanity check: Does the length field contain a value greater than the total document length (if known)?



Convenience Considerations
--------------------------

Decoders **SHOULD** offer an option to allow for partial data to be recovered (along with an error condition) when decoding fails partway through. This would involve discarding any partially decoded value (and its associated member name - if any), and then artificially terminating all open arrays and objects to produce a well-formed tree.



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

string            = u8(0xff) & char_string* & u8(0xff);

number            = int_small | int_8 | int_16 | int_24 | int_32 | int_40 | int_48 | int_56
                  | int_64 | uint_64 | float_16 | float_32 | float_64 | big_number;
int_small         = u8(0x00~0xea);
int_8             = u8(0xf1) & s8(~);
int_16            = u8(0xf2) & s16(~);
int_24            = u8(0xf3) & s24(~);
int_32            = u8(0xf4) & s32(~);
int_40            = u8(0xf5) & s40(~);
int_48            = u8(0xf6) & s48(~);
int_56            = u8(0xf7) & s56(~);
int_64            = u8(0xf8) & s64(~);
uint_64           = u8(0xf9) & u64(~);
float_16          = u8(0xfc) & f16(~);
float_32          = u8(0xfd) & f32(~);
float_64          = u8(0xfe) & f64(~);
big_number        = big_number_pos | big_number_neg;
big_number_pos    = u8(0xfa) & big_number_value;
big_number_neg    = u8(0xfb) & big_number_value;
big_number_value  = var(header, big_number_header)
                  & ordered(uint(header.sig_length*8, ~))
                  & ordered(zigzag(uint(header.exp_length*8, ~)))
                  ;
big_number_header = uleb128(uany(var(sig_length, ~)) & u2(var(exp_length, ~)));

array             = u8(0xeb) & value* & end_container;
object            = u8(0xec) & (string & value)* & end_container;
end_container     = u8(0xed);

boolean           = true | false;
false             = u8(0xee);
true              = u8(0xef);

null              = u8(0xf0);

# Primitives & Functions

s8(v)             = sint(8, v);
s16(v)            = ordered(sint(16, v));
s24(v)            = ordered(sint(24, v));
s32(v)            = ordered(sint(32, v));
s40(v)            = ordered(sint(40, v));
s48(v)            = ordered(sint(48, v));
s56(v)            = ordered(sint(56, v));
s64(v)            = ordered(sint(64, v));
u64(v)            = ordered(uint(64, v));
u2(v)             = uint(2, v);
u8(v)             = uint(8, v);
uany(v)           = uint(~,v);
f16(v)            = ordered(bfloat16(v));
f32(v)            = ordered(float(32, v));
f64(v)            = ordered(float(64, v));

char_string       = '\[0]' ~ '\[10ffff]'; # JSON technically supports unassigned and invalid codepoints

bfloat16(v: bits): bits = """https://en.wikipedia.org/wiki/Bfloat16_floating-point_format""";
uleb128(v: bits): bits = """https://en.wikipedia.org/wiki/LEB128#Unsigned_LEB128""";
zigzag(v: bits): bits = """https://en.wikipedia.org/wiki/Variable-length_quantity#Zigzag_encoding""";
```


License
-------

Copyright (c) 2024 Karl Stenerud. All rights reserved.

Distributed under the [Creative Commons Attribution License](https://creativecommons.org/licenses/by/4.0/legalcode) ([license deed](https://creativecommons.org/licenses/by/4.0).
