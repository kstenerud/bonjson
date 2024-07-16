BONJSON: Binary Object Notation for JSON
========================================

** * * * **PRERELEASE** * * * **


BONJSON is a **1:1 compatible** binary serialization format for [JSON](#json-standards).

It's a drop-in replacement that works in the same way and has the same capabilities and limitations as [JSON](#json-standards) (no more, no less), except in a binary format rather than text.

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

BONJSON follows the same structural rules as [JSON](#json-standards), as per these railroad diagrams:

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

 * Strings are always encoded as UTF-8.
 * All numeric fields are in little endian byte order.


### Type Codes

Every value is composed of an 8-bit type code, and in some cases also a payload:

| Type Code | Payload                      | Type    | Description                                         |
| --------- | ---------------------------- | ------- | --------------------------------------------------- |
| 00 - ea   |                              | Number  | [Integers -117 through 117](#small-integer)         |
| eb        |                              | Array   | [Array start](#array-encoding)                      |
| ec        |                              | Object  | [Object start](#object-encoding)                    |
| ed        |                              |         | [Container end](#container-encoding)                |
| ee        |                              | Boolean | [False](#boolean-encoding)                          |
| ef        |                              | Boolean | [True](#boolean-encoding)                           |
| f0        |                              | Null    | [Null](#null-encoding)                              |
| f1        | Biased signed integer        | Number  | [8-bit signed integer](#8-bit-signed-integer)       |
| f2 - f8   | 16 to 64 bit signed integer  | Number  | [Signed integer](#signed-integer)                   |
| f9        | 64-bit unsigned integer      | Number  | [64-bit unsigned integer](#64-bit-unsigned-integer) |
| fa        | Big Number                   | Number  | [Positive Big Number](#big-number)                  |
| fb        | Big Number                   | Number  | [Negative Big Number](#big-number)                  |
| fc        | 16-bit bfloat16 binary float | Number  | [16-bit float](#16-bit-float)                       |
| fd        | 32-bit ieee754 binary float  | Number  | [32-bit float](#32-bit-float)                       |
| fe        | 64-bit ieee754 binary float  | Number  | [64-bit float](#64-bit-float)                       |
| ff        | String                       | String  | [String](#string-encoding)                          |



String Encoding
---------------

Strings are sequences of UTF-8 characters delimited on both ends by the byte `0xff`.

Strings **MUST** be encoded in UTF-8. BONJSON supports the same UTF-8 codepoints as [JSON](#json-standards) does, but does not implement escape sequences (which are unnecessary in a binary format).

**Discussion**: Since the delimiter `0xff` is never a valid byte within a UTF-8 sequence (and never will be until we surpass 68 _billion_ codepoints), there is technically no need to interpret the UTF-8 characters themselves in the early decoding stage while scanning for the end of the string. A C/C++ implementation for example could use [`memccpy()`](https://www.man7.org/linux/man-pages/man3/memccpy.3.html), or it could overwrite the closing `0xff` delimiter with `0x00` to make a zero-copy null-terminated string using the original document buffer as a backing store.

**Example**:

    ff ff                         // ""
    ff 61 20 73 74 72 69 6e 67 ff // "a string"



Number Encoding
---------------

Numbers can be encoded using various integer and floating point forms. Encoders **SHOULD** use the most compact representation that stores each value without data loss.

[Small integer](#small-integer) and [8-bit integer](#8-bit-signed-integer) have special encodings. All other numeric types are encoded exactly as the (little endian) numbers they represent.

**Note**: Floating point `NaN` and `infinity` values **MUST NOT** be present in a document since they are not allowed in [JSON](#json-standards).

### Small Integer

Small integers (-117 to 117) are encoded into the [type code](#type-codes) itself for maximum compactness in the most commonly used integer range. Subtracting the bias value 117 from the [type code](#type-codes) produces the actual value it represents.

**Examples**:

    ea //  117
    7a //    5
    75 //    0
    39 //  -60
    11 // -100
    00 // -117


### 8-bit Signed Integer

The 8 bit signed integer encoding is another special case, taking over where [small integer](#small-integer) leaves off, covering integer ranges from 118 to 245, and from -118 to -245 within a single encoded payload byte following the [type code](#type-codes).

**Encoding**:

 * Integers from 118 to 245 have 118 subtracted from them to produce the encoded value.
 * Integers from -118 to -245 have 117 added to them to produce the encoded value.
 * No other values can be represented by this type (only 256 discrete values can be represented by a single byte).

**Decoding**:

 * Encoded values >= 0 have 118 added to them to produce the original value.
 * Encoded values < 0 have 117 subtracted from them to produce the original value.

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
    f6 bc 9a 78 56 34 12       //  0x123456789abc
    f8 00 00 00 00 00 00 00 80 // -0x8000000000000000

### 64-bit Unsigned Integer

The value is encoded as a little-endian 64-bit unsigned integer following the [type code](#type-codes). This is a convenience encoding for a commonly used integer type.

**Example**:

    f9 da da da de d0 d0 d0 de // 0xded0d0d0dedadada
                               // (They're meaningless and all that's true)

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

The general, logical form of a big number is as follows:

    [sign] [length header] [significand bytes] [exponent bytes]

 * The `sign` is encoded into the [type code](#type-codes) itself.
 * The `length header` field is encoded as an [unsigned LEB128](https://en.wikipedia.org/wiki/LEB128).
 * The `significand` is encoded as an unsigned integer in little endian byte order.
 * The `exponent` is encoded as a signed integer little endian byte order.
 * The exponent is a power-of-10, just like in [JSON](#json-standards) text notation.

The length header consists of two fields:

    [significand length] [exponent length]

The lower 2 bits of the length header represents the `exponent length` (0-3 bytes). The rest of the length header is the `significand length` in bytes.

    S S S S S S S ... E E

This allows for an unlimited significand size, and a ludicrous exponent range of ± 8 million.

**Notes**:

 * A field length of 0 represents a value of 0 for that field.
 * A Big Number with a significand value of 0 is equal to 0 with the sign from the [type code](#type-codes), regardless of exponent contents (which are superfluous in such a case).

**Discussion**: Big Number has the most complex encoding of all BONJSON types, but it's also the least likely to actually be used in real-world data (it exists to bring parity with JSON's unlimited number range).

**Examples**:

    fa 24 00 10 32 54 76 98 ba dc fe // 0xfedcba987654321000 (9 bytes significand, no exponent)
    fa 05 0f ff                      // 1.5 (15 x 10⁻¹) (1 byte significand, 1 byte exponent)
    fb 00                            // -0 (no significand, no exponent)
    fb 46 97 EB F2 0E C3 98 06 C1 47
       71 5E 65 4F 58 5F AA 28 d8 dc // -13837758495464977165497261864967377972119 x 10⁻⁹⁰⁰⁰
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

It's expected that your decoder will eventually encounter disallowed data due to the [GIGO effect](https://en.wikipedia.org/wiki/Garbage_in,_garbage_out) (for example float values containing NaN or infinity, numbers that are too large for your system, etc). Decoders **SHOULD** offer the user options for what to do when that happens:

 * Abort processing
 * Stringify the value
 * Replace the value with `null`

If a decoder provides such optional behavior, it **MUST** default to aborting. If a decoder provides no such optional behavior, it **MUST** abort processing.



Security Considerations
-----------------------

Any format that includes unbounded length fields is by definition open to abuse. BONJSON decoders **MUST** provide ways to protect against this. For example:

 * User-configurable maximum byte lengths for [big numbers](#big-number) (with sane defaults).
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

array             = u8(0xeb) & value* & end_container;
object            = u8(0xec) & (string & value)* & end_container;
end_container     = u8(0xed);

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
                  & ordered(sint(header.exp_length*8, ~))
                  ;
big_number_header = uleb128(uany(var(sig_length, ~)) & u2(var(exp_length, ~)));

boolean           = true | false;
false             = u8(0xee);
true              = u8(0xef);

string            = u8(0xff) & char_string* & u8(0xff);
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
```


License
-------

Copyright (c) 2024 Karl Stenerud. All rights reserved.

Distributed under the [Creative Commons Attribution License](https://creativecommons.org/licenses/by/4.0/legalcode) ([license deed](https://creativecommons.org/licenses/by/4.0).
