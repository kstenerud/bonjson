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
    - [Object](#object)
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

    ──[0xFC]──┬─>────────────────┬──[0xFE]──>
              ├─>─[value]──>─>───┤
              ╰─<─<─<─<─<─<─<─<─╯

**Object**:

    ──[0xFD]──┬─>──────────────────────────┬──[0xFE]──>
              ├─>─[string]──[value]──>─>───┤
              ╰─<─<─<─<─<─<─<─<─<─<─<─<─<─╯

**Structural validity rules**:

* A document **MUST** contain exactly one top-level value. An empty document (zero bytes) is invalid.
* Every container (array or object) **MUST** be properly terminated with an end marker (`0xFE`). A document that ends without the end marker for all open containers is invalid.

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
| c9        |                              |           | RESERVED                                   |
| ca        | Zigzag LEB128 number         | Number    | [Big Number](#big-number)                  |
| cb        | 32-bit ieee754 binary float  | Number    | [32-bit float](#32-bit-float)              |
| cc        | 64-bit ieee754 binary float  | Number    | [64-bit float](#64-bit-float)              |
| cd        |                              | Null      | [Null](#null)                              |
| ce        |                              | Boolean   | [False](#boolean)                          |
| cf        |                              | Boolean   | [True](#boolean)                           |
| d0 - df   | String of n bytes            | String    | [Short String](#short-string)              |
| e0 - e3   | Unsigned integer of n bytes  | Number    | [Unsigned Integer](#integer)               |
| e4 - e7   | Signed integer of n bytes    | Number    | [Signed Integer](#integer)                 |
| e8 - fb   |                              |           | RESERVED                                   |
| fc        |                              | Container | [Array](#array)                            |
| fd        |                              | Container | [Object](#object)                          |
| fe        |                              |           | Container end marker                       |
| ff        | Arbitrary length string      | String    | [Long String](#long-string)                |

**Note**: Decoders **MUST** reject documents containing reserved type codes.



Strings
-------

Strings are UTF-8 encoded sequences of Unicode codepoints, and can be encoded in two ways.

**Note**: Encoders **SHOULD** produce [NFC-normalized](https://unicode.org/reports/tr15/#Norm_Forms) strings. See [Compliance Levels](#compliance-levels) and [Unicode Normalization](#unicode-normalization) for details.


### Short String

Short strings have their byte length (up to 15) encoded directly into the lower nybble of the type code.

    Type Code Byte
    ───────────────
    1 1 0 1 L L L L
            ╰─┴─┴─┤
                  ╰─> Length (0-15 bytes)


**Examples**:

    d0                                               // ""
    d1 41                                            // "A"
    dc e3 81 8a e3 81 af e3 82 88 e3 81 86           // "おはよう"
    df 31 35 20 62 79 74 65 20 73 74 72 69 6e 67 21  // "15 byte string!"


### Long String

Long strings begin with the type code `0xFF`, followed by the raw UTF-8 string data, terminated by another `0xFF` byte. This is safe because the byte `0xFF` never appears in valid UTF-8.

    0xFF [data bytes] 0xFF

**Examples**:

    ff ff                                           // "" (empty long string)
    ff 61 20 73 74 72 69 6e 67 ff                   // "a string"

---

    ff                                              // (String of 64 Zs)
    5a 5a 5a 5a 5a 5a 5a 5a                         // ZZZZZZZZ
    5a 5a 5a 5a 5a 5a 5a 5a                         // ZZZZZZZZ
    5a 5a 5a 5a 5a 5a 5a 5a                         // ZZZZZZZZ
    5a 5a 5a 5a 5a 5a 5a 5a                         // ZZZZZZZZ
    5a 5a 5a 5a 5a 5a 5a 5a                         // ZZZZZZZZ
    5a 5a 5a 5a 5a 5a 5a 5a                         // ZZZZZZZZ
    5a 5a 5a 5a 5a 5a 5a 5a                         // ZZZZZZZZ
    5a 5a 5a 5a 5a 5a 5a 5a                         // ZZZZZZZZ
    ff



Numbers
-------

Numbers can be encoded using various integer and floating point forms. Encoders **SHOULD** use the most compact representation that stores each value without data loss.

All primitive numeric types are encoded exactly as they would appear in memory on little endian architectures.

**Notes**:

 * `NaN` and `infinity` are not valid BONJSON values. See [Values incompatible with JSON](#values-incompatible-with-json).
 * Decoders **MUST** accept any valid numeric encoding for a value, even if it is not the most compact representation. Only the mathematical value matters, not the encoding used to represent it.
 * A value such as `1.0` **MAY** be encoded as an integer (`0x65`) or as a float (`cb 00 00 80 3f`). Decoders **MUST** treat these as equivalent.
 * Negative zero (`-0.0`) **MUST** be encoded using a float encoding that preserves the sign. Encoding `-0.0` as integer `0` would lose the sign and is therefore considered data loss.


### Small Integer

Small integers (-100 to 100) are encoded into the [type code](#type-codes) itself for maximum compactness in the most commonly used integer range. The value is computed as: `type_code - 100`.

**Examples**:

    c8 //  100
    69 //    5
    64 //    0
    28 //  -60
    00 // -100


### Integer

Integers are encoded in little-endian byte order following the [type code](#type-codes). The type code determines the size and signedness of the integer:

| Type Code | Size    | Signedness |
| --------- | ------- | ---------- |
| e0        | 1 byte  | Unsigned   |
| e1        | 2 bytes | Unsigned   |
| e2        | 4 bytes | Unsigned   |
| e3        | 8 bytes | Unsigned   |
| e4        | 1 byte  | Signed     |
| e5        | 2 bytes | Signed     |
| e6        | 4 bytes | Signed     |
| e7        | 8 bytes | Signed     |

Integer sizes are restricted to CPU-native widths (1, 2, 4, 8 bytes) for efficient decoding without byte-shifting or padding.

Encoders **SHOULD** use the smallest encoding that can represent the value:

1. If the value fits in the small integer range (-100 to 100), use the small integer encoding.
2. Otherwise, use whichever of signed or unsigned requires fewer bytes.
3. If both signed and unsigned require the same number of bytes, prefer signed.

For example: 127 fits in 1 byte as either signed or unsigned, so use signed (`e4 7f`). 128 requires 2 bytes signed but only 1 byte unsigned, so use unsigned (`e0 80`).

**Examples**:

    e0 b4                      //  180
    e5 18 fc                   // -1000
    e1 00 80                   //  0x8000
    e3 da da da de d0 d0 d0 de //  0xded0d0d0dedadada (is all I want to say to you)
    e7 00 00 00 00 00 00 00 80 // -0x8000000000000000


### 32-bit Float

32-bit float is encoded as a little-endian [32-bit ieee754 binary float](https://en.wikipedia.org/wiki/Single-precision_floating-point_format) following the [type code](#type-codes) (`0xcb`).

**Example**:

    cb 00 b8 1f 42 // 0x1.3f7p5


### 64-bit Float

64-bit float is encoded as a little-endian [64-bit ieee754 binary float](https://en.wikipedia.org/wiki/Double-precision_floating-point_format) following the [type code](#type-codes) (`0xcc`).

**Example**:

    cc 58 39 b4 c8 76 be f3 3f // 1.234


### Big Number

Big Number ([type code](#type-codes) `0xca`) encodes arbitrary-precision base-10 numbers using [zigzag](https://en.wikipedia.org/wiki/Variable-length_quantity#Zigzag_encoding) [LEB128](https://en.wikipedia.org/wiki/LEB128) encoding.

The structure of a big number is as follows:

    0xCA [exponent] [signed significand]

 * The `exponent` is encoded as a zigzag LEB128 value representing a base-10 exponent.
 * The `signed significand` is encoded as a zigzag LEB128 value. The sign of the significand is embedded in the zigzag encoding.

The final value is derived as: `significand` × 10^`exponent`

**Zigzag encoding** maps signed integers to unsigned values: 0→0, -1→1, 1→2, -2→3, 2→4, etc. The formula is: `unsigned = (signed << 1) ^ (signed >> 63)`. Decoding: `signed = (unsigned >> 1) ^ -(unsigned & 1)`.

**LEB128 encoding** stores unsigned integers in 7-bit groups, least significant first. Each byte's high bit indicates whether more bytes follow (1 = more, 0 = last byte).

**Examples**:

    ca 00 00                   // 0 (exponent=0, significand=0)
    ca 00 04                   // 2 (exponent=0, significand=zigzag(2)=4)
    ca 00 01                   // -1 (exponent=0, significand=zigzag(-1)=1)
    ca 01 1e                   // 1.5 (exponent=zigzag(-1)=1, significand=zigzag(15)=30=0x1e)
                               //   → 15 × 10⁻¹ = 1.5

**Note**: Negative zero (`-0.0`) **CANNOT** be represented as a big number because zigzag encoding maps both `+0` and `-0` significands to `0`. Use IEEE 754 float encoding for negative zero.



Containers
----------

Containers use delimiter-terminated encoding: the container [type code](#type-codes) is followed by zero or more children, terminated by an end marker (`0xFE`).


### Array

An array consists of an `array` type code (`0xFC`), followed by zero or more values, and terminated by an end marker (`0xFE`).

    [0xFC] [value ...] [0xFE]

**Examples**:

    fc fe                      // [] (empty array)
    fc 65 fe                   // [1]
    fc d1 61 65 cd fe          // ["a", 1, null]


### Object

An object consists of an `object` type code (`0xFD`), followed by zero or more key-value pairs, and terminated by an end marker (`0xFE`). Each pair is a [string](#strings) key followed by a value.

    [0xFD] [string value ...] [0xFE]

**Notes**:

* Keys **MUST** be [strings](#strings).
* Every key **MUST** be followed by a value. A document that ends after a key but before its corresponding value is invalid.

**Examples**:

    fd fe                                    // {} (empty object)
    fd d1 62 64 d4 74 65 73 74 d1 78 fe     // {"b": 0, "test": "x"}



Boolean
-------

Boolean values are encoded into the [type codes](#type-codes) themselves:

 * False has type code `0xce`
 * True has type code `0xcf`



Null
----

Null has [type code](#type-codes) `0xcd`.



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

**Warning**: [Big numbers](#big-number) can encode very large values. Converting such values to decimal strings or performing arbitrary-precision arithmetic could consume excessive memory or CPU. Decoders **MUST** impose limits on numeric range as specified in [Resource Limits](#resource-limits).



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
| Maximum nesting depth  | 500                  | Arrays and objects nested within each other. Any value at the root level has depth 1; each value inside a container is one level deeper than the container itself. Examples: `[1]` has depth 2 (array at depth 1, element at depth 2); `[[]]` has depth 2; `[[1]]` has depth 3 (outer array at 1, inner array at 2, element at 3). |
| Maximum container size | 1,000,000 elements   | Total elements in a single array or key-value pairs in a single object                                   |
| Maximum string length  | 10,000,000 bytes     | Total encoded byte length of a single string                                                             |
| Numeric range          | 64-bit float/integer | Minimum supported range; not a configurable limit. See [Value Ranges](#value-ranges)                     |

Implementations **SHOULD** support at minimum 64-bit IEEE 754 floats and 64-bit signed and unsigned integers. JavaScript environments are limited to 53-bit integer precision; implementations targeting JavaScript **SHOULD** document this limitation.

Implementations **MUST** document their default limits and any configuration options they provide.

**Warning**: Choosing excessively high limits (or no limits) can make implementations vulnerable to resource exhaustion attacks. Even seemingly reasonable limits can be dangerous in combination (e.g., 1000 nesting depth × 1000 elements per level = 1 billion total nodes). Setting a limit to 0 typically means "no limit" and is **NOT RECOMMENDED** for production use.


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

**Note**: Short string type codes encode the byte length of the original (non-normalized) UTF-8 data. After NFC normalization, the resulting string may have a different byte length. For example, `café` encoded with a decomposed `é` (U+0065 U+0301) occupies 6 bytes, but after NFC normalization becomes 5 bytes (using precomposed U+00E9). This is expected behavior.

**Note**: Implementations **SHOULD** use the latest available Unicode version for NFC normalization. While different Unicode versions may produce different results for edge cases involving newly added characters, this is unavoidable and generally affects only obscure codepoints.


### Values incompatible with JSON

NaN and infinity are unrepresentable in JSON, but are technically possible to encode in BONJSON because it stores binary floating point values directly.

Codecs **MUST** by default reject NaN and infinity values (whether in [float](#32-bit-float) bit patterns), and **MAY** offer the following configuration options:

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
    fd                                                 // { (object start)
       d6 6e 75 6d 62 65 72                            //     "number":
       96                                              //     50,
       d4 6e 75 6c 6c                                  //     "null":
       cd                                              //     null,
       d7 62 6f 6f 6c 65 61 6e                         //     "boolean":
       cf                                              //     true,
       d5 61 72 72 61 79                                //     "array":
       fc                                              //     [ (array start)
          d1 78                                         //         "x",
          e5 e8 03                                      //         1000,
          cb 00 00 a0 bf                                //         -1.25
       fe                                              //     ] (array end),
       d6 6f 62 6a 65 63 74                             //     "object":
       fd                                              //     { (object start)
          df 6e 65 67 61 74 69 76 65 20 6e 75 6d 62    //         "negative number":
             65 72                                      //
          00                                            //         -100,
          db 6c 6f 6e 67 20 73 74 72 69 6e 67          //         "long string":
          ff                                            //         "1234567890123456789012345678901234567890"
             31 32 33 34 35 36 37 38 39 30              //
             31 32 33 34 35 36 37 38 39 30              //
             31 32 33 34 35 36 37 38 39 30              //
             31 32 33 34 35 36 37 38 39 30              //
          ff                                            //
       fe                                              //     } (object end)
    fe                                                 // } (object end)
```

    Size:    123 bytes



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
ordered_document  = value;

value             = array | object | number | boolean | string | null;

# Types

# Containers use delimiter-terminated encoding.
# 0xFC/0xFD opens, 0xFE closes.
array             = u8(0xfc) & value* & u8(0xfe);
object            = u8(0xfd) & (string & value)* & u8(0xfe);

number            = int_small | int_unsigned | int_signed | float_32 | float_64 | big_number;
int_small         = u8(var(code, 0x00~0xc8));  # value = code - 100
int_unsigned      = u8(0xe0) & ordered(uint( 8, ~))
                  | u8(0xe1) & ordered(uint(16, ~))
                  | u8(0xe2) & ordered(uint(32, ~))
                  | u8(0xe3) & ordered(uint(64, ~))
                  ;
int_signed        = u8(0xe4) & ordered(sint( 8, ~))
                  | u8(0xe5) & ordered(sint(16, ~))
                  | u8(0xe6) & ordered(sint(32, ~))
                  | u8(0xe7) & ordered(sint(64, ~))
                  ;
float_32          = u8(0xcb) & ordered(f32(~));
float_64          = u8(0xcc) & ordered(f64(~));
big_number        = u8(0xca) & zigzag_leb128(~) & zigzag_leb128(~);

boolean           = true | false;
false             = u8(0xce);
true              = u8(0xcf);

null              = u8(0xcd);

string            = string_short | string_long;
string_short      = u8(var(code, 0xd0~0xdf)) & sized((code - 0xd0) * 8, char_string*);
string_long       = u8(0xff) & char_string* & u8(0xff);

# Zigzag LEB128: variable-length signed integer encoding.
# Zigzag maps signed to unsigned: 0→0, -1→1, 1→2, -2→3, ...
# LEB128 stores 7 data bits per byte, MSbit=1 means more bytes follow.
zigzag_leb128(v)  = leb128(zigzag(v));
zigzag(v)         = uint(~, (v << 1) ^ (v >> 63));  # signed to unsigned
leb128(v)         = [
                        v <= 0x7f:         uint(7, v) & u1(0);
                        v >  0x7f:         uint(7, v) & u1(1) & leb128(v >> 7);
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
