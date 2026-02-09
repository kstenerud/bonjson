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
    - [32-bit Float](#32-bit-float)
    - [64-bit Float](#64-bit-float)
    - [Big Number](#big-number)
  - [Containers](#containers)
    - [Array](#array)
    - [Typed Array](#typed-array)
    - [Object](#object)
    - [Record](#record)
  - [Boolean](#boolean)
  - [Null](#null)
  - [Interoperability Considerations](#interoperability-considerations)
    - [Value Ranges](#value-ranges)
  - [Security Rules](#security-rules)
    - [Resource Limits](#resource-limits)
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

A **secure** encoder:

* **SHOULD** produce [NFC-normalized](https://unicode.org/reports/tr15/#Norm_Forms) strings when possible

A **secure** decoder:

* **MUST** validate UTF-8 encoding (reject [invalid UTF-8](#invalid-utf-8))
* **MUST** normalize strings to [NFC](https://unicode.org/reports/tr15/#Norm_Forms) before performing [duplicate key](#duplicate-object-keys) detection
* **SHOULD** return NFC-normalized strings to the application (this is strongly recommended but not required; the critical requirement is that duplicate key detection uses NFC-normalized strings)


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

    ──[record_def]*──[value]──>

**Value**:

    ──┬─>─[string]──┬─>
      ├─>─[number]──┤
      ├─>─[object]──┤
      ├─>─[array]───┤
      ├─>─[boolean]─┤
      ╰─>─[null]────╯

**Array**:

    ──[0xB4]──┬─>───────────┬──[0xB3]──>
              ├─>─[value]─>─┤
              ╰─<─<─<─<─<─<─╯

**Typed Array**:

    ──[0xF5-0xFE]──[element_count (LEB128)]──[data bytes...]──>

**Object**:

    ──[0xB5]──┬─>─────────────────────┬──[0xB3]──>
              ├─>─[string]──[value]─>─┤
              ╰─<─<─<─<─<─<─<─<─<─<─<─╯

**Record Definition**:

    ──[0xB6]──┬─>────────────┬──[0xB3]──>
              ├─>─[string]─>─┤
              ╰─<─<─<─<─<─<──╯

**Record Instance**:

    ──[0xB7]──[def_index (LEB128)]──┬─>───────────┬──[0xB3]──>
                                    ├─>─[value]─>─┤
                                    ╰─<─<─<─<─<─<─╯

**Structural validity rules**:

* A document **MUST** contain exactly one top-level value, optionally preceded by zero or more [record definitions](#record). An empty document (zero bytes) is invalid.
* Every delimiter-terminated container ([array](#array), [object](#object), [record definition, or record instance](#record)) **MUST** be properly terminated with an end marker (`0xB3`). A document that ends without the end marker for all open containers is invalid. ([Typed arrays](#typed-array) are length-prefixed and do not use end markers.)
* All [record definitions](#record) **MUST** appear before the root value. A record definition type code (`0xB6`) encountered after any non-definition data has been read is invalid.

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
| 00 - 64   |                              | Number    | [Integers 0 through 100](#small-integer)   |
| 65 - a4   | String of n bytes            | String    | [Short String](#short-string)              |
| a5 - a8   | Unsigned integer of n bytes  | Number    | [Unsigned Integer](#integer)               |
| a9 - ac   | Signed integer of n bytes    | Number    | [Signed Integer](#integer)                 |
| ad        | 32-bit ieee754 binary float  | Number    | [32-bit float](#32-bit-float)              |
| ae        | 64-bit ieee754 binary float  | Number    | [64-bit float](#64-bit-float)              |
| af        | Zigzag LEB128 + LE magnitude | Number    | [Big Number](#big-number)                  |
| b0        |                              | Boolean   | [False](#boolean)                          |
| b1        |                              | Boolean   | [True](#boolean)                           |
| b2        |                              | Null      | [Null](#null)                              |
| b3        |                              |           | Container end marker                       |
| b4        |                              | Container | [Array](#array)                            |
| b5        |                              | Container | [Object](#object)                          |
| b6        | Key strings + end marker     |           | [Record Definition](#record)               |
| b7        | Index + values + end marker  | Container | [Record Instance](#record)                 |
| b8 - f4   |                              |           | RESERVED                                   |
| f5        | Count + element data         | Container | [Typed Array float64](#typed-array)        |
| f6        | Count + element data         | Container | [Typed Array float32](#typed-array)        |
| f7        | Count + element data         | Container | [Typed Array sint64](#typed-array)         |
| f8        | Count + element data         | Container | [Typed Array sint32](#typed-array)         |
| f9        | Count + element data         | Container | [Typed Array sint16](#typed-array)         |
| fa        | Count + element data         | Container | [Typed Array sint8](#typed-array)          |
| fb        | Count + element data         | Container | [Typed Array uint64](#typed-array)         |
| fc        | Count + element data         | Container | [Typed Array uint32](#typed-array)         |
| fd        | Count + element data         | Container | [Typed Array uint16](#typed-array)         |
| fe        | Count + element data         | Container | [Typed Array uint8](#typed-array)          |
| ff        | Arbitrary length string      | String    | [Long String](#long-string)                |

**Note**: Type codes marked RESERVED are not used in BONJSON. Decoders **MUST** reject documents containing reserved type codes.



Strings
-------

Strings are UTF-8 encoded sequences of Unicode codepoints, and can be encoded in two ways.

**Note**: Encoders **SHOULD** produce [NFC-normalized](https://unicode.org/reports/tr15/#Norm_Forms) strings. See [Compliance Levels](#compliance-levels) and [Unicode Normalization](#unicode-normalization) for details.

**Note**: Unlike text-based JSON, BONJSON allows control characters (such as TAB, CR, LF) directly in strings without escaping. This is safe because the binary format has no whitespace-sensitivity issues that would cause text editors or transport layers to corrupt the data.


### Short String

Short strings have their byte length (up to 63) encoded into the type code. The length is computed as: `type_code - 0x65`.

**Examples**:

    65                                               // ""
    66 41                                            // "A"
    71 e3 81 8a e3 81 af e3 82 88 e3 81 86           // "おはよう"
    74 31 35 20 62 79 74 65 20 73 74 72 69 6e 67 21  // "15 byte string!"


### Long String

Long strings begin with the type code `0xFF`, followed by the raw UTF-8 string data, terminated by another `0xFF` byte. This is safe because the byte `0xFF` (`11111111`) can never appear in valid UTF-8: it is not a valid single-byte character (ASCII range is `00`-`7F`), not a valid leading byte (leading bytes range from `C2`-`F4`), and not a valid continuation byte (continuation bytes have the form `10xxxxxx`, i.e. `80`-`BF`).

    0xFF [data bytes] 0xFF

**Examples**:

    ff ff                                           // "" (empty long string)

    ff 61 20 73 74 72 69 6e 67 ff                   // "a string"

    ff                                              // (String of 64 Zs)
       5a 5a 5a 5a 5a 5a 5a 5a                      // ZZZZZZZZ
       5a 5a 5a 5a 5a 5a 5a 5a                      // ZZZZZZZZ
       5a 5a 5a 5a 5a 5a 5a 5a                      // ZZZZZZZZ
       5a 5a 5a 5a 5a 5a 5a 5a                      // ZZZZZZZZ
       5a 5a 5a 5a 5a 5a 5a 5a                      // ZZZZZZZZ
       5a 5a 5a 5a 5a 5a 5a 5a                      // ZZZZZZZZ
       5a 5a 5a 5a 5a 5a 5a 5a                      // ZZZZZZZZ
       5a 5a 5a 5a 5a 5a 5a 5a                      // ZZZZZZZZ
    ff



Numbers
-------

Numbers can be encoded using various integer and floating point forms. Encoders **SHOULD** use the most compact representation that stores each value without data loss.

All primitive numeric types are encoded exactly as they would appear in memory on little endian architectures.

**Notes**:

 * `NaN` and `infinity` are by default not valid, but decoders **MAY** choose to accept them anyway. See [Values incompatible with JSON](#values-incompatible-with-json).
 * Numeric type fidelity is not guaranteed. The encoding used (integer, float, big number) carries no semantic meaning — only the mathematical value matters. A decoder **MAY** decode any numeric encoding into whatever native type best represents the value.
 * Decoders **MUST** accept any valid numeric encoding for a value, even if it is not the most compact representation. Only the mathematical value matters, not the encoding used to represent it.
 * A value such as `1.0` **MAY** be encoded as an integer (`0x01`) or as a float (`ad 00 00 80 3f`). Decoders **MUST** treat these as equivalent.
 * Subnormal (denormalized) float values are valid and **MUST** be accepted by decoders.
 * Negative zero (`-0.0`) **MUST** be encoded using a float encoding that preserves the sign. Encoders **SHOULD** prefer [32-bit float](#32-bit-float) (`ad 00 00 00 80`) as the most compact representation. Encoding `-0.0` as integer `0` would lose the sign and is therefore considered data loss.


### Small Integer

Small integers (0 to 100) are encoded into the [type code](#type-codes) itself for maximum compactness in the most commonly used integer range. The value equals the type code directly.

**Examples**:

    64 // 100
    05 //   5
    00 //   0


### Integer

Integers are encoded in little-endian byte order following the [type code](#type-codes). The type code determines the size and signedness of the integer:

| Type Code | Size    | Signedness |
| --------- | ------- | ---------- |
| a5        | 1 byte  | Unsigned   |
| a6        | 2 bytes | Unsigned   |
| a7        | 4 bytes | Unsigned   |
| a8        | 8 bytes | Unsigned   |
| a9        | 1 byte  | Signed     |
| aa        | 2 bytes | Signed     |
| ab        | 4 bytes | Signed     |
| ac        | 8 bytes | Signed     |

Integer sizes are restricted to CPU-native widths (1, 2, 4, 8 bytes) for efficient decoding without byte-shifting or padding.

Encoders **SHOULD** use the smallest encoding that can represent the value:

1. If the value fits in the small integer range (0 to 100), use the small integer encoding.
2. Otherwise, use whichever of signed or unsigned requires fewer bytes.
3. If both signed and unsigned require the same number of bytes, prefer signed.

For example: 127 fits in 1 byte as either signed or unsigned, so use signed (`a9 7f`). 128 requires 2 bytes signed but only 1 byte unsigned, so use unsigned (`a5 80`).

**Note**: Using a larger-than-necessary encoding (e.g., encoding `1` as a 64-bit integer) produces a valid document that decoders **MUST** accept. However, it wastes bytes and is not recommended.

**Examples**:

    a5 b4                      //  180
    aa 18 fc                   // -1000
    a6 00 80                   //  0x8000
    a8 da da da de d0 d0 d0 de //  0xded0d0d0dedadada (is all I want to say to you)
    ac 00 00 00 00 00 00 00 80 // -0x8000000000000000


### 32-bit Float

32-bit float is encoded as a little-endian [32-bit ieee754 binary float](https://en.wikipedia.org/wiki/Single-precision_floating-point_format) following the [type code](#type-codes) (`0xAD`).

**Example**:

    ad 00 b8 1f 42 // 0x1.3f7p5


### 64-bit Float

64-bit float is encoded as a little-endian [64-bit ieee754 binary float](https://en.wikipedia.org/wiki/Double-precision_floating-point_format) following the [type code](#type-codes) (`0xAE`).

**Example**:

    ae 58 39 b4 c8 76 be f3 3f // 1.234


### Big Number

Big Number ([type code](#type-codes) `0xAF`) encodes arbitrary-precision base-10 numbers using [zigzag](https://en.wikipedia.org/wiki/Variable-length_quantity#Zigzag_encoding) [LEB128](https://en.wikipedia.org/wiki/LEB128) metadata and little-endian magnitude bytes.

The structure of a big number is as follows:

    0xAF [exponent] [signed length] [magnitude bytes]

 * The `exponent` is encoded as a zigzag LEB128 value representing a base-10 exponent.
 * The `signed length` is encoded as a zigzag LEB128 value. Its absolute value is the byte count of the magnitude field. Its sign indicates the sign of the significand: positive means the significand is positive, negative means the significand is negative, and zero means the significand is zero (with no magnitude bytes following).
 * The `magnitude bytes` are an unsigned integer in little-endian byte order, exactly `abs(signed_length)` bytes long. When `signed_length` is zero, this field is absent.

The final value is derived as: `sign(signed_length)` × `magnitude` × 10^`exponent`

**Magnitude normalization**: The magnitude **MUST** be normalized: the last byte (most significant) **MUST** be non-zero. A zero-valued significand is represented by `signed_length = 0`, not by magnitude bytes of `0x00`. Decoders **MUST** reject big numbers with non-normalized magnitude (trailing zero bytes).

**Zigzag encoding** maps signed integers to unsigned values: 0→0, -1→1, 1→2, -2→3, 2→4, etc. The encoding formula is: `unsigned = (signed << 1) ^ (signed >> (bit_width - 1))`, where `>>` is an arithmetic (sign-extending) right shift. Decoding: `signed = (unsigned >> 1) ^ -(unsigned & 1)`.

**Note**: Zigzag encoding has no representation for negative zero. Both `+0` and `-0` map to unsigned `0`. To encode negative zero (`-0.0`), use an [IEEE 754 float encoding](#32-bit-float).

**LEB128 encoding** stores unsigned integers in 7-bit groups, least significant first. Each byte's high bit indicates whether more bytes follow (1 = more, 0 = last byte). The total length of a LEB128 sequence is bounded by the decoder's [numeric range](#resource-limits) limit; decoders **MUST** reject LEB128 sequences that would decode to values outside of their supported range.

**Examples**:

    af 00 00                   // 0 (exponent=0, signed_length=0 → significand is zero)
    af 00 02 02                // 2 (exponent=0, signed_length=+1, magnitude=0x02)
    af 00 01 01                // -1 (exponent=0, signed_length=-1, magnitude=0x01)
    af 01 02 0f                // 1.5 (exponent=zigzag(0x01)=-1, signed_length=+1,
                               //   magnitude=0x0F=15) → 15 × 10⁻¹ = 1.5
    af 04 02 0a                // 1000 (exponent=zigzag(0x04)=2, signed_length=+1,
                               //   magnitude=0x0A=10) → 10 × 10² = 1000

**Encoder normalization**: Encoders **SHOULD** produce the most compact representation by normalizing trailing zeros out of the significand and into the exponent. For example, the value 1000 is more compactly encoded as `significand=1, exponent=3` than `significand=10, exponent=2`. If the significand is zero, encoders **SHOULD** also normalize the exponent to zero (i.e., encode as `af 00 00` rather than using a non-zero exponent). See also [Interoperability Considerations](#interoperability-considerations).



Containers
----------

Arrays, objects, and [records](#record) use delimiter-terminated encoding: the container [type code](#type-codes) is followed by zero or more children, terminated by an end marker (`0xB3`). [Typed arrays](#typed-array) use a length-prefixed encoding for compact storage of homogeneous numeric data.


### Array

An array consists of an `array` type code (`0xB4`), followed by zero or more values, and terminated by an end marker (`0xB3`).

    [0xB4] [value ...] [0xB3]

**Examples**:

    b4 b3                      // [] (empty array)
    b4 01 b3                   // [1]
    b4 66 61 01 b2 b3          // ["a", 1, null]


### Typed Array

A typed array is a compact encoding for homogeneous numeric arrays. The element type is encoded directly into the type code (`0xF5`-`0xFE`), followed by an unsigned [LEB128](https://en.wikipedia.org/wiki/LEB128) element count, and then the raw element data.

    [type_code] [element_count (LEB128)] [data bytes...]

The type code identifies the element type:

| Element Type | Type Code | Element Size |
| ------------ | --------- | ------------ |
| uint8        | fe        | 1 byte       |
| uint16       | fd        | 2 bytes      |
| uint32       | fc        | 4 bytes      |
| uint64       | fb        | 8 bytes      |
| sint8        | fa        | 1 byte       |
| sint16       | f9        | 2 bytes      |
| sint32       | f8        | 4 bytes      |
| sint64       | f7        | 8 bytes      |
| float32      | f6        | 4 bytes      |
| float64      | f5        | 8 bytes      |

The data section contains `element_count` elements of the specified type, encoded contiguously in little-endian byte order. The total data size in bytes is `element_count × element_size`.

A typed array is semantically identical to a regular [JSON](#json-standards) array of numbers — it is a 1:1 JSON compatible encoding optimization.

**Rules**:

* An empty typed array (element count = 0) is valid and equivalent to `[]`.
* Encoders **MAY** use typed arrays when the array is homogeneous; it is always optional. Decoders **MUST** accept typed arrays.
* The [resource limit](#resource-limits) for maximum container size applies to the element count.

**Examples**:

    fe 03 01 02 03                            // typed uint8 array [1, 2, 3]
    f5 02 58 39 b4 c8 76 be f3 3f             // typed float64 array [1.234, 5.678]
       83 c0 ca a1 45 b6 16 40
    fc 00                                     // typed uint32 array [] (empty)


### Object

An object consists of an `object` type code (`0xB5`), followed by zero or more key-value pairs, and terminated by an end marker (`0xB3`). Each pair is a [string](#strings) key followed by a value.

    [0xB5] [string value ...] [0xB3]

**Notes**:

* Keys **MUST** be [strings](#strings). A document containing a non-string type code in key position **MUST** be rejected.
* Every key **MUST** be followed by a value. A document that ends after a key but before its corresponding value is invalid.
* [Duplicate keys](#duplicate-object-keys) **MUST** cause the document to be rejected by default. Decoders **MAY** offer configuration options as described in the [Duplicate object keys](#duplicate-object-keys) section.

**Examples**:

    b5 b3                                    // {} (empty object)
    b5 66 62 00 69 74 65 73 74 66 78 b3      // {"b": 0, "test": "x"}


### Record

A record is a compact encoding for repeated object schemas. When many objects share the same set of keys, record encoding avoids repeating the key strings for every object by declaring the keys once in a **record definition** and then referencing that definition from each **record instance**.

A record instance is semantically identical to an [object](#object) — it is a 1:1 JSON compatible encoding optimization, like [typed arrays](#typed-array) are for numeric arrays.


#### Record Definition

A record definition (`0xB6`) declares a list of key strings. It consists of the type code, followed by zero or more [strings](#strings), and terminated by an end marker (`0xB3`).

    [0xB6] [string ...] [0xB3]

Record definitions **MUST** appear at the beginning of a document, before the root value. Each definition is assigned a 0-based index in the order it appears. A document **MAY** contain zero or more record definitions.

**Rules**:

* A record definition encountered after any non-definition data has been read is invalid. Decoders **MUST** reject such documents.
* Record definition keys **MUST** be [strings](#strings). A non-string type code in key position **MUST** be rejected.
* An empty record definition (zero keys) is valid.
* [Duplicate key](#duplicate-object-keys) rules apply to the keys within a record definition.
* The [resource limit](#resource-limits) for maximum container size applies to the key count.


#### Record Instance

A record instance (`0xB7`) references a previously declared record definition and supplies values for each key. It consists of the type code, followed by an unsigned [LEB128](https://en.wikipedia.org/wiki/LEB128) definition index, zero or more values, and terminated by an end marker (`0xB3`).

    [0xB7] [def_index (LEB128)] [value ...] [0xB3]

The definition index is 0-based. The values are matched positionally to the keys from the referenced definition: the first value corresponds to the first key, the second value to the second key, and so on.

**Rules**:

* The definition index **MUST** refer to a previously declared record definition. An out-of-range index is invalid.
* If fewer values are provided than keys in the definition (early end marker), the remaining keys have the value [null](#null).
* If more values are provided than keys in the definition, the document is invalid. Decoders **MUST** reject such documents.
* A record instance in a document with no record definitions is invalid.

**Examples**:

```text
    b6 69 6e 61 6d 65 68 61 67 65 b3             // Record def 0: keys ["name", "age"]
    b4                                            // [ (array start)
       b7 00 6a 41 6c 69 63 65 1e b3             //     {"name": "Alice", "age": 30}
       b7 00 68 42 6f 62 19 b3                    //     {"name": "Bob", "age": 25}
    b3                                            // ] (array end)
```

The above is equivalent to `[{"name": "Alice", "age": 30}, {"name": "Bob", "age": 25}]`.

```text
    b6 66 61 66 62 66 63 b3                       // Record def 0: keys ["a", "b", "c"]
    b7 00 01 b3                                   // {"a": 1, "b": null, "c": null}
```

Early end marker: only one value is provided for a three-key definition, so `"b"` and `"c"` default to null.



Boolean
-------

Boolean values are encoded into the [type codes](#type-codes) themselves:

 * False has type code `0xB0`
 * True has type code `0xB1`



Null
----

Null has [type code](#type-codes) `0xB2`.



Interoperability Considerations
-------------------------------

Because [JSON](#json-standards) is so weakly specified, there are numerous ways in which one implementation can become incompatible with another.


### Value Ranges

Although JSON allows an unlimited range for most values, it's important to take into consideration the limitations of the systems that will be trying to ingest your data.

Encoders **SHOULD** always use the smallest encoding for the value being encoded in order to ensure maximum compatibility.

A codec **MAY** [choose its own value range restrictions](#resource-limits), but **SHOULD** at least support reasonable industry-standard ranges for maximum interoperability, and **MUST** publish what its restrictions are.

Most systems can natively handle:

 * 64 bit floating point values
 * 64 bit integer values (signed or unsigned)

JavaScript in particular can natively handle:

 * 64 bit floating point values
 * 53 bit integer values (plus the sign)

**Warning**: [Big numbers](#big-number) can encode very large values. Converting such values to decimal strings or performing arbitrary-precision arithmetic could consume excessive memory or CPU. Decoders **MUST** impose limits on big number magnitude and exponent as specified in [Resource Limits](#resource-limits), even on platforms that support arbitrary-precision numeric types.



Security Rules
--------------

All data formats contain technically feasible coding sequences that could result in exploitable weaknesses if certain invariants aren't artificially enforced. [JSON is particularly bad in this regard](https://bishopfox.com/blog/json-interoperability-vulnerabilities). BONJSON mitigates these by being cautious, rejecting dangerous sequences by default. However, since JSON is so old, systems have inevitably arisen that rely upon such broken behaviors.

BONJSON handles this by choosing opinionated, secure default behaviors while allowing informed users to deliberately disable such security protections when necessary.

**Note**: Many parts of this specification mention "rejecting" values or encodings. What is meant is that the _entire document_ is rejected as a consequence of rejecting the unacceptable data (as opposed to ignoring or replacing the unacceptable data).


### Resource Limits

Decoders **MUST** enforce limits on resource consumption to prevent denial-of-service attacks. The following limits **SHOULD** be configurable, with secure defaults:

| Resource               | Recommended Default  | Notes                                                                                                    |
| ---------------------- | -------------------- | -------------------------------------------------------------------------------------------------------- |
| Maximum document size  | 2,000,000,000 bytes  | Total size of the encoded document                                                                       |
| Maximum nesting depth  | 500                  | Before any value is written, the depth is 0. The top-level value has depth 1. Each value inside a container is one level deeper than the container itself. Examples: `1` has depth 1; `[]` has depth 1; `[1]` has depth 2 (array at 1, element at 2); `[[]]` has depth 2 (outer array at 1, inner array at 2); `[[1]]` has depth 3 (outer at 1, inner at 2, element at 3). |
| Maximum container size | 1,000,000 elements   | Total elements in a single array or key-value pairs in a single object                                   |
| Maximum string length  | 10,000,000 bytes     | Total encoded byte length of a single string                                                             |
| Numeric range          | Platform default     | Maximum absolute value for native integers and floats. **SHOULD** default to the platform's native limits (e.g., 64-bit float/integer). See [Value Ranges](#value-ranges) |
| Big number magnitude   | 256 bytes            | Maximum byte length of the [big number](#big-number) magnitude. 256 bytes provides approximately 617 decimal digits of precision, sufficient for virtually all legitimate use cases |
| Big number exponent    | ±100,000             | Maximum absolute value of the [big number](#big-number) exponent. Limits the cost of decimal conversion and arithmetic operations |

Implementations **SHOULD** support at minimum 64-bit IEEE 754 floats and 64-bit signed and unsigned integers. JavaScript environments are limited to 53-bit integer precision without special support; implementations targeting JavaScript **SHOULD** document this limitation.

Implementations **MUST** document their default limits and any configuration options they provide.

**Warning**: Choosing excessively high limits (or no limits) can make implementations vulnerable to resource exhaustion attacks. Even seemingly reasonable limits can be dangerous in combination (e.g., 1000 nesting depth × 1000 elements per level = 1 billion total nodes). Setting a limit to 0 typically means "no limit" and is not recommended for production use.


### Invalid UTF-8

BONJSON strings are encoded as UTF-8. Invalid UTF-8 sequences have been involved in numerous security exploits and **MUST** be rejected by default.

Invalid UTF-8 sequences include:

* Overlong encodings (using more bytes than necessary to encode a codepoint)
* Sequences that decode to values greater than U+10FFFF
* Sequences that decode to surrogate codepoints (U+D800-U+DFFF)
* Invalid continuation bytes
* Truncated multi-byte sequences

Further information about how invalid UTF-8 can become a security nightmare is available at [RFC 3629, section 10](https://www.rfc-editor.org/rfc/rfc3629#section-10), [Unicode Technical Report #36](https://www.unicode.org/reports/tr36/tr36-15.html), and [Corrigendum #1: UTF-8 Shortest Form](https://www.unicode.org/versions/corrigendum1.html). Overlong UTF-8 encodings were famously involved in the 2001 exploit of IIS web servers by encoding "../" as `C0 AE 2E 2F` instead of the expected `2E 2E 2F`, thus fooling its naive rejection filters.

For compatibility with broken data sources, decoders **MAY** offer the following configuration options:

* Reject the document (this **MUST** be the default behavior)
* Replace invalid sequences with the REPLACEMENT CHARACTER U+FFFD (less secure)
* Delete invalid sequences from the string (even less secure)
* Pass through invalid sequences unchanged (dangerous and not recommended)


### NUL Codepoint Restriction

Because the `NUL` (U+0000) codepoint can so easily be exploited, it **MUST** be rejected by default in all strings, regardless of whether the string is used as an object key, an object value, or an array element.

Decoders **MAY** offer a configuration option to allow `NUL`, but the default behavior **MUST** be to reject.


### Unicode Normalization

[Unicode normalization](https://unicode.org/reports/tr15/) ensures that equivalent character sequences are represented consistently. Without normalization, strings that appear identical to users (such as "café" using precomposed `U+00E9` vs "café" using `U+0065 U+0301`) would be treated as different, which creates security vulnerabilities.

Some platforms (notably Swift and Objective-C) automatically normalize strings when using them as dictionary keys. This means two keys that passed a byte-for-byte duplicate check could still collide in the application's native map, allowing an attacker to control which value is kept.

The handling of Unicode normalization depends on the [compliance level](#compliance-levels):

* **Secure compliance**: Decoders normalize strings to [NFC](https://unicode.org/reports/tr15/#Norm_Forms) before performing [duplicate key](#duplicate-object-keys) detection. This prevents the attack described above.
* **Basic compliance**: Decoders perform byte-for-byte comparison without normalization. This is vulnerable to key collision attacks if the keys later get normalized.

**Note**: Short string type codes encode the byte length of the original (non-normalized) UTF-8 data. After NFC normalization, the resulting string may have a different byte length. For example, `café` encoded with a decomposed `é` (U+0065 U+0301) occupies 6 bytes, but after NFC normalization becomes 5 bytes (using precomposed U+00E9). This is expected behavior.

**Note**: Implementations **SHOULD** use the latest available Unicode version for NFC normalization. While different Unicode versions may produce different results for edge cases involving newly added characters, this is unavoidable and generally affects only obscure codepoints.


### Values incompatible with JSON

NaN and infinity are unrepresentable in JSON, but are technically possible to encode in BONJSON because it stores binary floating point values directly.

Codecs **MUST** by default reject NaN and infinity values (whether in [32-bit](#32-bit-float) or [64-bit](#64-bit-float) float bit patterns), and **MAY** offer the following configuration options:

* Reject the document (this **MUST** be the default behavior)
* Stringify the value (replace the numeric value with a string: `"NaN"`, `"Infinity"`, or `"-Infinity"`)
* Allow the value (passing the decoded data to a JSON encoder may fail or produce different results)

**Note**: The stringify option converts the numeric value to a string value at decode time. This allows interoperability with systems that represent these special values as strings. The application or a type-aware decoder can convert the string back to the appropriate floating-point value if needed.


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

This would involve discarding any partially decoded value (and its associated object member name - if any), and then artificially terminating all open arrays and objects to produce a well-formed tree. In practice this usually means just returning the structure that has been decoded thus far, since all containers will exist, and any key without a matching value will not have been added yet.



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
        "long string": "1234567890123456789012345678901234567890123456789012345678901234"
    }
}
```

    Size:     269 bytes
    Minified: 180 bytes

**BONJSON**:

```text
    b5                                                 // { (object start)
       6b 6e 75 6d 62 65 72                            //     "number":
       32                                              //     50,
       69 6e 75 6c 6c                                  //     "null":
       b2                                              //     null,
       6c 62 6f 6f 6c 65 61 6e                         //     "boolean":
       b1                                              //     true,
       6a 61 72 72 61 79                               //     "array":
       b4                                              //     [ (array start)
          66 78                                        //         "x",
          aa e8 03                                     //         1000,
          ad 00 00 a0 bf                               //         -1.25
       b3                                              //     ] (array end),
       6b 6f 62 6a 65 63 74                            //     "object":
       b5                                              //     { (object start)
          74 6e 65 67 61 74 69 76 65 20 6e 75 6d 62    //         "negative number":
             65 72                                     //
          a9 9c                                        //         -100,
          70 6c 6f 6e 67 20 73 74 72 69 6e 67          //         "long string":
          ff                                           //         "12345678901234567890123456789012
             31 32 33 34 35 36 37 38 39 30             //          34567890123456789012345678901234"
             31 32 33 34 35 36 37 38 39 30             //
             31 32 33 34 35 36 37 38 39 30             //
             31 32 33 34 35 36 37 38 39 30             //
             31 32 33 34 35 36 37 38 39 30             //
             31 32 33 34 35 36 37 38 39 30             //
             31 32 33 34                               //
          ff                                           //
       b3                                              //     } (object end)
    b3                                                 // } (object end)
```

    Size:    148 bytes



Formal BONJSON Grammar
----------------------

The following [Dogma](https://github.com/kstenerud/dogma/blob/master/v1/dogma_v1.0.md) grammar is **normative**. In case of any discrepancy between the prose descriptions in this document and the formal grammar, the grammar takes precedence.

**Note**: The grammar defines what constitutes a valid BONJSON encoding at the byte level. Security restrictions described in the prose (such as NUL character rejection, NaN/infinity rejection, and duplicate key detection) are runtime checks that may cause a structurally valid document to be rejected. The grammar permits these byte sequences; the security rules determine whether they are accepted at runtime.

```dogma
dogma_v1 utf-8
- identifier  = bonjson
- description = Binary Object Notation for JSON
- reference   = https://bonjson.org
- dogma       = https://github.com/kstenerud/dogma/blob/master/v1/dogma_v1.0.md

document          = byte_order(lsb, ordered_document);
ordered_document  = record_def* & value;

value             = typed_array | array | object | record | number | boolean | string | null;

# Types

# Containers use delimiter-terminated encoding.
# 0xB4/0xB5 opens, 0xB3 closes.
# Typed array is a compact encoding for homogeneous numeric arrays.
typed_array       = u8(var(code, 0xf5~0xfe)) & leb128(var(count, ~))
                  & sized(count * typed_element_size * 8,
                      ordered(uint(count * typed_element_size * 8, ~)));
typed_element_type = var(etype, 0xfe - code);
typed_element_size = [
                        etype == 0: 1;  # uint8
                        etype == 4: 1;  # sint8
                        etype == 1: 2;  # uint16
                        etype == 5: 2;  # sint16
                        etype == 2: 4;  # uint32
                        etype == 6: 4;  # sint32
                        etype == 8: 4;  # float32
                        etype == 3: 8;  # uint64
                        etype == 7: 8;  # sint64
                        etype == 9: 8;  # float64
                    ];
array             = u8(0xb4) & value* & u8(0xb3);
object            = u8(0xb5) & (string & value)* & u8(0xb3);
# Record: compact encoding for repeated object schemas.
# Definitions declare key lists; instances reference a definition by index.
record_def        = u8(0xb6) & string* & u8(0xb3);
record            = u8(0xb7) & leb128(~) & value* & u8(0xb3);

number            = int_small | int_unsigned | int_signed | float_32 | float_64 | big_number;
int_small         = u8(var(code, 0x00~0x64));  # value = code
int_unsigned      = u8(0xa5) & ordered(uint( 8, ~))
                  | u8(0xa6) & ordered(uint(16, ~))
                  | u8(0xa7) & ordered(uint(32, ~))
                  | u8(0xa8) & ordered(uint(64, ~))
                  ;
int_signed        = u8(0xa9) & ordered(sint( 8, ~))
                  | u8(0xaa) & ordered(sint(16, ~))
                  | u8(0xab) & ordered(sint(32, ~))
                  | u8(0xac) & ordered(sint(64, ~))
                  ;
float_32          = u8(0xad) & ordered(f32(~));
float_64          = u8(0xae) & ordered(f64(~));
# Big number: exponent (zigzag LEB128), signed_length (zigzag LEB128),
# then abs(signed_length) unsigned LE magnitude bytes.
# The sign of signed_length indicates the sign of the significand.
big_number        = u8(0xaf) & zigzag_leb128(~) & zigzag_leb128(var(slen, ~))
                  & sized(abs(slen) * 8, uint(abs(slen) * 8, ~));

boolean           = true | false;
false             = u8(0xb0);
true              = u8(0xb1);

null              = u8(0xb2);

string            = string_short | string_long;
string_short      = u8(var(code, 0x65~0xa4)) & sized((code - 0x65) * 8, char_string*);
string_long       = u8(0xff) & char_string* & u8(0xff);

# Zigzag LEB128: variable-length signed integer encoding.
# Zigzag maps signed to unsigned: 0→0, -1→1, 1→2, -2→3, ...
# LEB128 stores 7 data bits per byte, MSbit=1 means more bytes follow.
zigzag_leb128(v)  = leb128(zigzag(v));
zigzag(v)         = uint(~, (v << 1) ^ (v >> (bit_width(v) - 1)));  # signed to unsigned
leb128(v)         = [
                        v <= 0x7f: uint(7, v) & u1(0);
                        v >  0x7f: uint(7, v) & u1(1) & leb128(v >> 7);
                    ];

# Primitives & Functions

u1(v)             = uint(1, v);
u8(v)             = uint(8, v);
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
