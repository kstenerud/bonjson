BONJSON Test Specification
==========================

Version: 1.0.0

This document specifies the format for cross-implementation BONJSON test suites. Test specifications defined according to this format can be used to verify that any BONJSON implementation correctly handles encoding, decoding, and error cases.


Contents
--------

- [BONJSON Test Specification](#bonjson-test-specification)
  - [Contents](#contents)
  - [Terms and Conventions](#terms-and-conventions)
    - [Case Sensitivity](#case-sensitivity)
  - [Overview](#overview)
    - [Design Goals](#design-goals)
  - [Document Structure](#document-structure)
    - [Required Fields](#required-fields)
    - [Comment Convention](#comment-convention)
    - [Comment-Only Entries (Section Dividers)](#comment-only-entries-section-dividers)
    - [Unrecognized Fields](#unrecognized-fields)
  - [Test Case Structure](#test-case-structure)
    - [Common Fields](#common-fields)
      - [Test Name](#test-name)
      - [Test Type](#test-type)
    - [Type-Specific Fields](#type-specific-fields)
      - [`encode` Tests](#encode-tests)
      - [`decode` Tests](#decode-tests)
      - [`roundtrip` Tests](#roundtrip-tests)
      - [`encode_error` Tests](#encode_error-tests)
      - [`decode_error` Tests](#decode_error-tests)
    - [Optional Fields](#optional-fields)
      - [Options](#options)
      - [Requires](#requires)
  - [Data Representation](#data-representation)
    - [Binary Data](#binary-data)
    - [JSON Values](#json-values)
    - [Special Values](#special-values)
      - [Numbers](#numbers)
      - [Raw Byte Sequences](#raw-byte-sequences)
  - [Error Types](#error-types)
  - [Test Execution](#test-execution)
    - [Encode Test Execution](#encode-test-execution)
    - [Decode Test Execution](#decode-test-execution)
    - [Roundtrip Test Execution](#roundtrip-test-execution)
    - [Error Test Execution](#error-test-execution)
    - [Value Parsing](#value-parsing)
    - [Value Comparison](#value-comparison)
  - [File Organization](#file-organization)
  - [Test Configuration File](#test-configuration-file)
    - [Configuration Structure](#configuration-structure)
    - [Source Types](#source-types)
    - [Directory Processing](#directory-processing)
  - [Schema](#schema)
  - [Versioning](#versioning)
    - [Version Compatibility](#version-compatibility)
  - [Implementation Notes](#implementation-notes)
    - [Number Encoding Flexibility](#number-encoding-flexibility)
    - [Object Key Ordering](#object-key-ordering)
    - [Platform Differences](#platform-differences)
  - [Runner Validation](#runner-validation)
    - [Directory Structure](#directory-structure)
    - [must-pass Directory](#must-pass-directory)
    - [structural-errors Directory](#structural-errors-directory)
    - [skip-scenarios Directory](#skip-scenarios-directory)
    - [config Directory](#config-directory)
    - [value-handling Directory](#value-handling-directory)
    - [README.md](#readmemd)
  - [Appendix A: Type Code Reference](#appendix-a-type-code-reference)
  - [Appendix B: Complete Example](#appendix-b-complete-example)


Terms and Conventions
---------------------

**The following bolded, capitalized terms have specific meanings in this document**:

| Term                  | Meaning                                                                                          |
| --------------------- | ------------------------------------------------------------------------------------------------ |
| **MUST (NOT)**        | If this directive is not adhered to, the document or implementation is invalid.                  |
| **SHOULD (NOT)**      | Every effort should be made to follow this directive, but it's still conformant if not followed. |
| **MAY (NOT)**         | It is up to the implementation to decide whether to do something or not.                         |
| **CAN**               | Refers to a possibility which **MUST** be accommodated by the implementation.                    |
| **CANNOT**            | Refers to a situation which **MUST NOT** be allowed by the implementation.                       |
| **STRUCTURAL ERROR**  | Document is malformed and can't be reliably processed. Exit immediately with an error message.   |

### Case Sensitivity

The following elements are **case-sensitive** (must match exactly):

- Document `type` field values (`"bonjson-test"`, `"bonjson-test-config"`)
- Test `type` field values (`"encode"`, not `"Encode"`)
- Error type values (`"truncated"`, not `"Truncated"`)
- Option names (`"allow_nul"`, not `"Allow_Nul"`)
- Capability identifiers (`"arbitrary_precision_bignumber"`, not `"Arbitrary_Precision_BigNumber"`)
- Field names (`"expected_bytes"`, not `"Expected_Bytes"`)
- Marker keys (`"$number"`, not `"$Number"`)

The following elements are **case-insensitive**:

- Test `name` field (for duplicate detection only; names may contain any case)
- `$number` special values (`"NaN"`, `"sNaN"`, `"Infinity"`, `"-Infinity"`)
- `$number` hex prefix (`"0x"`, `"0X"`) for both hex integers and hex floats
- `$number` hex float exponent (`"p"`, `"P"`)
- `$number` hex digits (`"0xabc"` = `"0xABC"`)
- `$number` scientific notation exponent (`"e"`, `"E"`)
- File extensions (`.json`, `.JSON`)
- Hex digits in byte strings (`"9a"` = `"9A"`)

-----------------------------------------------------------------------------------------------------------------------


Overview
--------

This specification defines two JSON document types:

| Document Type         | `type` Field Value     | Purpose                              |
|-----------------------|------------------------|--------------------------------------|
| Test specification    | `"bonjson-test"`       | Contains test cases                  |
| Test configuration    | `"bonjson-test-config"`| References test files and directories|

All `.json` files processed by a test runner **MUST** have a `type` field at the top level with one of these values. A missing `type` field or unrecognized `type` value is a **STRUCTURAL ERROR**. Non-test-related JSON files **MUST NOT** be placed in test directories.

All required fields **MUST** have the correct JSON type as specified in their respective tables (e.g., `type` and `version` must be strings, `tests` and `sources` must be arrays). A required field with the wrong JSON type is a **STRUCTURAL ERROR**.

BONJSON test specifications are JSON documents containing arrays of test cases. Each test case defines an input, an expected output or error, and the type of test to perform.

### Design Goals

- **Human-readable**: Tests **SHOULD** be easy to read and write by hand
- **Machine-parseable**: **MUST** be valid JSON for universal compatibility
- **Self-contained**: Each test case contains all information needed to execute it
- **Cross-platform**: Tests **SHOULD** be implementable in any programming language


Document Structure
------------------

A test specification document is a JSON object with the following structure:

```json
{
  "type": "bonjson-test",
  "version": "<semver>",
  "tests": [<test-case>, ...]
}
```

### Required Fields

| Field     | Type   | Description                                                       |
|-----------|--------|-------------------------------------------------------------------|
| `type`    | string | Format identifier, **MUST** be `"bonjson-test"`                   |
| `version` | string | Semantic version of the test specification format (e.g., "1.0.0") |
| `tests`   | array  | Array of test case objects                                        |

A missing required field is a **STRUCTURAL ERROR**.

An empty `tests` array is valid. This allows placeholder files or files that exist solely to document future test cases via comments.

### Comment Convention

JSON does not support comments. To allow inline documentation, any key starting with `//` **MUST** be ignored by test runners when it appears at the top level of:
- Test specification documents
- Test case objects
- Configuration file documents
- Source objects in configuration files

Comment keys are **only** recognized in these four locations. They are **not** allowed in:
- The `options` object (would be treated as an unrecognized option, causing the test to be skipped)
- Marker objects like `{"$number": "..."}` (only the marker key is permitted)
- Data values within `input` or `expected_value` (the `//` key is preserved as actual data)

The conventional comment key is `//`:

```json
{
  "//": "This is a comment explaining the test",
  "name": "example_test",
  "type": "encode",
  ...
}
```

Multiple comment keys are permitted:

```json
{
  "//": "Main description",
  "//note": "Additional note",
  "//see_also": "related_test_name",
  "name": "example_test",
  ...
}
```

The `//` prefix was chosen because it is universally recognized as a comment marker by programmers and is unlikely to appear in real data keys.

### Comment-Only Entries (Section Dividers)

Entries in the `tests` array that contain **only** keys starting with `//` are comment blocks (section dividers). Test runners **MUST** silently skip these entries without treating them as errors.

```json
{
  "type": "bonjson-test",
  "version": "1.0.0",
  "tests": [
    {"//": "=== Integer Encoding Tests ==="},
    {"name": "int_zero", "type": "encode", "input": 0, "expected_bytes": "00"},
    {"name": "int_one", "type": "encode", "input": 1, "expected_bytes": "01"},

    {"//section": "Negative Integers", "//note": "Testing negative range"},
    {"name": "int_neg_one", "type": "encode", "input": -1, "expected_bytes": "a9 ff"}
  ]
}
```

Comment-only entries serve as organizational tools to divide tests into logical sections. They are useful for large test files where grouping related tests improves readability.

**Important**: If an entry contains **any** non-comment keys (keys not starting with `//`), it is treated as a test case and **MUST** include the required `name` and `type` fields. Mixed entries with both comment keys and missing required fields are a **STRUCTURAL ERROR**.

### Unrecognized Fields

Test runners **MUST** ignore unrecognized fields in:
- Test specification documents (top level)
- Test case objects
- Configuration file documents (top level)
- Source objects

This allows forward compatibility when new optional fields are added in future specification versions. Unrecognized fields **SHOULD NOT** produce warnings, as this would create noise when running newer test files with older test runners.

Note: This applies only to extra fields at the object level. Unrecognized values for known fields (e.g., an unknown test `type`) are still a **STRUCTURAL ERROR** as specified elsewhere.


Test Case Structure
-------------------

Each test case is a JSON object with required and optional fields depending on the test type. An element of the `tests` array that is not a JSON object is a **STRUCTURAL ERROR**.

### Common Fields

| Field  | Type   | Required | Description                         |
|--------|--------|----------|-------------------------------------|
| `name` | string | Yes      | Unique identifier for the test case |
| `type` | string | Yes      | Type of test to perform             |

A missing required field (whether common fields like `name` and `type`, or type-specific fields like `expected_bytes` for encode tests) is a **STRUCTURAL ERROR**.

#### Test Name

The `name` field **MUST**:
- Be unique within the test file (compared case-insensitively)
- Contain only letters (any case), digits, and underscores
- Start with a letter

A name that violates these rules (e.g., `"123test"` or `"test-name"`) is a **STRUCTURAL ERROR**.

The `name` field **SHOULD** be descriptive of what the test verifies.

Test runners **MUST** compare names case-insensitively when checking for duplicates within a single file. Duplicate names (e.g., `Integer_overflow` and `integer_overflow`) are a **STRUCTURAL ERROR**.

Test names are **not** required to be unique across different files. To disambiguate tests in reports, test runners **MUST** include both the source file path and the test name in all output (e.g., `tests/integers.json:int16_overflow`).

Examples: `smallint_0`, `Truncated_Float16`, `roundtrip_nested_object`

#### Test Type

The `type` field **MUST** be one of the following:

| Type           | Description                               |
|----------------|-------------------------------------------|
| `encode`       | Verify encoding produces specific bytes   |
| `decode`       | Verify decoding produces specific value   |
| `roundtrip`    | Verify value survives encode→decode       |
| `encode_error` | Verify encoding fails with specific error |
| `decode_error` | Verify decoding fails with specific error |

An unrecognized test type is a **STRUCTURAL ERROR**.

### Type-Specific Fields

#### `encode` Tests

Verify that encoding a value produces an exact byte sequence.

| Field            | Type   | Required | Description                                     |
|------------------|--------|----------|-------------------------------------------------|
| `input`          | any    | Yes      | Value to encode                                 |
| `expected_bytes` | string | Yes      | Expected output as hex string (spaces optional) |

```json
{
  "name": "int16_1000",
  "type": "encode",
  "input": 1000,
  "expected_bytes": "aa e8 03"
}
```

#### `decode` Tests

Verify that decoding a byte sequence produces a specific value.

| Field            | Type   | Required | Description                                     |
|------------------|--------|----------|-------------------------------------------------|
| `input_bytes`    | string | Yes      | Bytes to decode as hex string (spaces optional) |
| `expected_value` | any    | Yes      | Expected decoded value                          |

```json
{
  "name": "decode_float32",
  "type": "decode",
  "input_bytes": "ad 00 00 90 3f",
  "expected_value": 1.125
}
```

#### `roundtrip` Tests

Verify that a value survives encoding and decoding with the same semantic value. The intermediate byte representation is not verified. Type changes are acceptable if the value is mathematically equal (e.g., encoding `1.0` and decoding as integer `1` is valid).

| Field   | Type | Required | Description         |
|---------|------|----------|---------------------|
| `input` | any  | Yes      | Value to round-trip |

```json
{
  "name": "roundtrip_nested_object",
  "type": "roundtrip",
  "input": {"a": {"b": [1, 2, 3]}}
}
```

#### `encode_error` Tests

Verify that encoding a value fails with a specific error.

| Field            | Type   | Required | Description                          |
|------------------|--------|----------|--------------------------------------|
| `input`          | any    | Yes      | Value that should fail to encode     |
| `expected_error` | string | Yes      | Expected [error type](#error-types)  |

```json
{
  "name": "encode_nan",
  "type": "encode_error",
  "input": {"$number": "NaN"},
  "expected_error": "invalid_data"
}
```

#### `decode_error` Tests

Verify that decoding a byte sequence fails with a specific error.

| Field            | Type   | Required | Description                                        |
|------------------|--------|----------|----------------------------------------------------|
| `input_bytes`    | string | Yes      | Bytes that should fail to decode (spaces optional) |
| `expected_error` | string | Yes      | Expected [error type](#error-types)                |

```json
{
  "name": "truncated_int16",
  "type": "decode_error",
  "input_bytes": "aa e8",
  "expected_error": "truncated"
}
```

### Optional Fields

#### Options

The `options` field is an optional JSON object that configures encoder/decoder behavior for a specific test. If present, it **MUST** be a JSON object; any other type (e.g., string, array, null) is a **STRUCTURAL ERROR**.

| Option                 | Type    | Description                                            |
|------------------------|---------|--------------------------------------------------------|
| `allow_nul`            | boolean | Allow NUL characters in strings                        |
| `allow_trailing_bytes` | boolean | Allow unconsumed bytes after decoding (default: false) |
| `nan_infinity_behavior`| string  | How to handle NaN/Infinity: `"reject"` (default), `"allow"` (pass through as float), or `"stringify"` (convert to string representation) |
| `duplicate_key`        | string  | How to handle duplicate object keys: `"reject"` (default), `"keep_first"`, or `"keep_last"` |
| `invalid_utf8`         | string  | How to handle invalid UTF-8: `"reject"` (default), `"replace"` (with U+FFFD), `"delete"`, or `"pass_through"` |
| `max_depth`            | integer | Maximum container nesting depth (non-negative). See depth counting examples below. |
| `max_container_size`   | integer | Maximum elements in a container (non-negative)         |
| `max_string_length`    | integer | Maximum string length in bytes (non-negative)          |
| `max_document_size`    | integer | Maximum document size in bytes (non-negative)          |
| `max_bignumber_exponent` | integer | Maximum absolute value of BigNumber exponent (non-negative). See [Resource Limits](bonjson.md#resource-limits) in the BONJSON specification. |
| `max_bignumber_magnitude`| integer | Maximum byte length of BigNumber magnitude (non-negative). See [Resource Limits](bonjson.md#resource-limits) in the BONJSON specification. |
| `unicode_normalization`| string  | How to normalize Unicode strings: `"none"` (default) or `"nfc"` (normalize to NFC composed form). When `"nfc"`, decoded strings are transformed to Unicode Normalization Form C before being returned, and object keys are compared after NFC normalization (which can cause otherwise-distinct keys to become duplicates). |
| `out_of_range`         | string  | How to handle BigNumber values that exceed configured limits (`max_bignumber_exponent`, `max_bignumber_magnitude`) or the implementation's native numeric range: `"error"` (default) or `"stringify"` (convert to a decimal string representation `[±]<significand>e<exponent>`). |

**Depth counting**: Any value at the root level (including primitives and empty containers) has depth 1. Each value inside a container is one level deeper than the container itself. Examples:
- `42` → depth 1 (primitive at root)
- `[]` or `{}` → depth 1 (empty container at root)
- `[1, 2, 3]` → depth 2 (array at depth 1, elements inside at depth 2)
- `[[]]` → depth 2 (outer array at depth 1, inner empty array at depth 2)
- `[[1]]` → depth 3 (outer array at depth 1, inner array at depth 2, element at depth 3)
- `[[[1]]]` → depth 4
- `{"a": {}}` → depth 2 (outer object at depth 1, inner object value at depth 2)
- `{"a": {"b": 1}}` → depth 3 (outer object at depth 1, inner object at depth 2, value at depth 3)
- `{"a": 1, "b": [2]}` → depth 3 (object at 1, value `1` at 2, array at 2, value `2` at 3)

The maximum depth is the deepest level reached by any value. With `max_depth: 2`, the last example `{"a": 1, "b": [2]}` would be rejected because the value `2` inside the nested array reaches depth 3.

With `max_depth: 1`, primitives and empty containers are allowed at root, but containers may not have any elements inside (elements would be at depth 2). With `max_depth: 2`, containers may contain primitives but not nested containers. With `max_depth: 3`, one level of nested containers is allowed (e.g., `[[1]]`).

Option values **MUST** have the correct JSON type (boolean for boolean options, integer for integer options, string for string options). Null is not a valid option value. An option with the wrong type (e.g., `"allow_nul": "true"` or `"allow_nul": null`) is a **STRUCTURAL ERROR**. Integer options **MUST** be non-negative; negative values are a **STRUCTURAL ERROR**. String options **MUST** use one of the defined values; unrecognized values are a **STRUCTURAL ERROR**.

For integer limit options (`max_depth`, `max_container_size`, `max_string_length`, `max_document_size`, `max_bignumber_exponent`, `max_bignumber_magnitude`), a value of 0 means "no limit." **Warning**: Disabling limits can make implementations vulnerable to denial-of-service attacks and is **NOT RECOMMENDED** for production use.

Default values for these options are defined in the BONJSON specification. Tests typically use small values (e.g., `max_depth: 5`) to verify that limit-checking machinery works correctly in an implementation.

Test runners **MUST** skip (and log a warning for) any tests that contain:
- Option names not recognized by the test runner (e.g., typos like `"alow_nul"`)
- Option settings not supported by the underlying library

Note: Setting options that contradict the expected outcome (e.g., `nan_infinity_behavior: "allow"` on an `encode_error` test expecting NaN to fail) is a malformed test. Test authors are responsible for writing sensible tests; test runners are not required to detect such contradictions.

```json
{
  "name": "string_with_nul",
  "type": "roundtrip",
  "input": "hello\u0000world",
  "options": {
    "allow_nul": true
  }
}
```

#### Requires

The `requires` field is an optional JSON array that declares implementation capabilities needed to run the test. If present, it **MUST** be a JSON array of strings; any other type is a **STRUCTURAL ERROR**.

```json
{
  "name": "bignumber_high_precision",
  "type": "roundtrip",
  "input": {"$number": "1.23456789012345678901234567890"},
  "requires": ["arbitrary_precision_bignumber"]
}
```

Test runners **MUST** skip (and log a warning for) any tests that require capabilities the implementation does not support. This allows tests to be written for features that not all implementations can handle, without causing false failures.

The following capability identifiers are defined:

| Capability                       | Description                                                                                                                                                                          |
|----------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `arbitrary_precision_bignumber`  | Support for BigNumber values with more than ~17 significant digits (exceeds float64 precision). Implementations using float64 for decoded BigNumbers will lose precision.            |
| `bignumber_exponent_gt_127`      | Support for BigNumber exponents greater than 127. Some implementations (e.g., Swift's Decimal) limit exponents to -128 to 127.                                                       |
| `bignumber_exponent_lt_neg128`   | Support for BigNumber exponents less than -128. Some implementations limit exponents to -128 to 127.                                                                                 |
| `nan_infinity_stringify`         | Support for the `nan_infinity_behavior: "stringify"` option, which converts NaN/Infinity float values to string representations. Not all implementations support this mode.          |
| `uint64`                         | Support for full 64-bit unsigned integers. Some platforms (e.g., JavaScript) cannot represent integers beyond 2^53-1.                                                                |
| `int64`                          | Support for full 64-bit signed integers. Some platforms cannot represent integers beyond ±2^53-1.                                                                                    |
| `negative_zero`                  | Support for IEEE 754 negative zero (-0.0) preservation. Some platforms or type systems cannot distinguish -0.0 from +0.0.                                                            |
| `raw_string_bytes`               | Support for representing strings as raw byte sequences (for testing `invalid_utf8: "pass_through"`). Implementations using native string types that require valid UTF-8 cannot support this. |
| `signaling_nan`                  | Support for preserving the signaling bit of NaN values. Most platforms convert signaling NaN to quiet NaN on any operation; tests using `sNaN` should require this capability.       |
| `out_of_range_stringify`         | Support for the `out_of_range: "stringify"` option, which converts out-of-range BigNumber values to string representations. Not all implementations support this mode.               |

Test runners **SHOULD**:
1. Define which capabilities their implementation supports
2. Skip tests whose `requires` array contains unsupported capabilities
3. Report skipped tests with the reason (missing capability)

Capability identifiers **MUST** match exactly (case-sensitive, byte-for-byte comparison). No fuzzy matching or typo correction is performed. Unrecognized capability identifiers **MUST** cause the test to be skipped with a warning (not a **STRUCTURAL ERROR**), to ensure forward compatibility when new capabilities are added in future specification versions.


Data Representation
-------------------

### Binary Data

Binary data (byte sequences) is represented as a hexadecimal string (uppercase or lowercase) with optional spaces for readability. An empty string (`""`) represents zero bytes, which is useful for testing the empty document error case. When decoding zero bytes, implementations **MUST** report a `truncated` error since BONJSON documents require at least one value.

Hex strings **MUST**:
- Contain only hex digits (`0-9`, `a-f`, `A-F`) and spaces
- Have an even number of hex digits (since each byte requires two digits)

Non-hex, non-space characters or an odd number of hex digits is a **STRUCTURAL ERROR**.

Example:

```json
"expected_bytes": "9a866e756d62657232"
```

Is equivalent to the more readable:

```json
"expected_bytes": "9a 86 6e 75 6d 62 65 72 32"
```

### JSON Values

Standard JSON values are represented directly:

- **null**: `null`
- **boolean**: `true`, `false`
- **number**: `123`, `-45.67`, `1.23e10`
- **string**: `"hello"`, `"日本語"`
- **array**: `[1, 2, 3]`
- **object**: `{"key": "value"}`

### Special Values

Special values are values that cannot be directly represented in JSON, and instead use marker objects. These are JSON objects with a single key starting with `$` that signals to the test runner how to interpret the value.

Marker objects **MUST** contain exactly one key (the marker key). Additional keys—including comment keys starting with `//`—are a **STRUCTURAL ERROR**. For example, `{"$number": "1.5", "extra": "field"}` and `{"$number": "1.5", "//": "comment"}` are both invalid.

#### Numbers

The `$number` marker represents stringified numeric values that cannot be safely represented as JSON numbers:

```json
{"$number": "NaN"}
{"$number": "sNaN"}
{"$number": "Infinity"}
{"$number": "-Infinity"}
{"$number": "18446744073709551615"}
{"$number": "0x1.921fb54442d18p+1"}
{"$number": "1.23456789e-5"}
```

The test runner parses the string based on its format:

| Format                                 | Interpretation                                                                     |
|----------------------------------------|------------------------------------------------------------------------------------|
| `NaN`, `sNaN`, `Infinity`, `-Infinity` | IEEE 754 special float values (case-insensitive). `NaN` is quiet NaN; `sNaN` is signaling NaN. |
| `0x...p...`                    | IEEE 754 hex float (C99 `%a` format) for precise representation (case-insensitive) |
| `0x...` (no `p`)               | Hex integer (case-insensitive)                                                     |
| Decimal integer or float       | Arbitrary-precision decimal number (case-insensitive for `e`/`E` exponent)         |

**Hex integer format**: Standard hexadecimal integer format `[±]0xH...` where `H...` are hex digits. At least one hex digit is required (i.e., `0x` alone is invalid). For negative hex integers, parse the hex portion as an unsigned value, then negate the result (i.e., `-0xff` means "negate 255" = -255, not "parse 0xff as signed"). Values that fit in signed 64-bit range (-2^63 to 2^63-1) are encoded as integers; values outside this range (e.g., `0xffffffffffffffff` = 2^64-1, or `-0xffffffffffffffff` = -(2^64-1)) are encoded as BigNumber. Examples: `0x100` (256), `-0xff` (-255), `0x7fffffffffffffff` (max int64), `0xffffffffffffffff` (BigNumber).

**Hex float format**: C99 hexadecimal floating-point format as produced by `printf("%a", ...)`. The general form is `[±]0x[H...][.H...]p[±]D...` where `H...` are hex digits and `D...` is the decimal exponent (power of 2). The integer part, fractional part, and exponent sign are all optional, but at least one hex digit must be present (before and/or after the decimal point), and at least one decimal digit must be present in the exponent. Examples:
- `0x1.921fb54442d18p+1` = π
- `0x1.5bf0a8b145769p+1` = e
- `-0x1.4p+3` = -10.0
- `0x1p0` = 1.0 (no fractional part)
- `0x1p+1` = 2.0 (no fractional part, explicit positive exponent)
- `0x.8p1` = 1.0 (no integer part)
- `0x0p+0` = 0.0 (positive zero)
- `-0x0p+0` = -0.0 (negative zero)
- `0x1p-1074` = smallest positive denormalized double

**Negative zero**: Per IEEE 754, negative zero (-0.0) is distinct from positive zero (0.0). In `$number`, negative zero **MUST** be represented with the C99 hex float format `-0x0p+0` or as the decimal string `"-0.0"`. Test runners **MUST** parse these correctly and preserve the sign in comparisons.

**Decimal format**: Standard decimal notation with optional sign and scientific notation. Use this for large integers (beyond JavaScript's 2^53-1 safe range) and BigNumber values.

**Type selection**: When parsing `$number` for encoding, the test runner **SHOULD** produce:
- Integer strings (no decimal point or exponent, e.g., `"1"`, `"100"`) → integer if within signed 64-bit range, otherwise BigNumber
- Hex integers (e.g., `"0x100"`) → integer if within signed 64-bit range, otherwise BigNumber
- Decimal or scientific notation (e.g., `"1.0"`, `"1e2"`, `"100.0"`) → float
- Hex floats → float with the exact bit pattern specified
- Integer values outside signed 64-bit range (-2^63 to 2^63-1) → BigNumber

Values with decimal points or exponents (e.g., `"1.0"`, `"1e2"`, `"100.0"`) are parsed as floats. This allows tests to verify that the encoder correctly optimizes whole-number floats into smaller integer representations when appropriate. Encoders **MAY** convert float values to integers if the value is a whole number and no precision is lost, but are not required to do so.

The `$number` string **MUST NOT** be empty and **MUST NOT** contain leading or trailing spaces. An unparseable `$number` value (e.g., `{"$number": "hello"}`) is a **STRUCTURAL ERROR**.


#### Raw Byte Sequences

The `$bytes` marker represents raw byte sequences in string contexts, useful for testing invalid UTF-8 handling when `invalid_utf8: "pass_through"` is configured:

```json
{"$bytes": "68 65 6c 6c 6f ff 77 6f 72 6c 64"}
```

The string value is a hexadecimal string (like `input_bytes`), representing the exact bytes that should appear in the decoded string. The same validation rules apply: only hex digits (0-9, a-f, A-F) and spaces are allowed, with an even number of hex digits. An empty `$bytes` value or one with invalid characters is a **STRUCTURAL ERROR**. This marker is only meaningful in `expected_value` fields for decode tests where the decoded string may contain invalid UTF-8.

Tests using `$bytes` in expected values **MUST** include `requires: ["raw_string_bytes"]` since many implementations cannot represent or compare strings containing invalid UTF-8. Implementations that always validate UTF-8 (e.g., those using native string types that require valid UTF-8) should skip these tests.

```json
{
  "name": "decode_passthrough_invalid_utf8",
  "type": "decode",
  "input_bytes": "70 68 65 6c 6c 6f ff 77 6f 72 6c 64",
  "expected_value": {"$bytes": "68 65 6c 6c 6f ff 77 6f 72 6c 64"},
  "options": {"invalid_utf8": "pass_through"},
  "requires": ["raw_string_bytes"]
}
```

The `$bytes` marker **MUST** only appear in `expected_value` fields of decode tests. Appearing in `input` fields (for encode or roundtrip tests) or other contexts is a **STRUCTURAL ERROR**.


Error Types
-----------

The `expected_error` field uses standardized error type identifiers:

| Error Type                      | Description                                |
|---------------------------------|--------------------------------------------|
| `truncated`                     | Unexpected end of input data (includes empty documents, and length fields that specify more data than remain in the document) |
| `trailing_bytes`                | Unconsumed bytes after decoding            |
| `invalid_type_code`             | Unrecognized or reserved type code         |
| `invalid_utf8`                  | Invalid UTF-8 byte sequence                |
| `nul_character`                 | NUL (0x00) byte in string                  |
| `duplicate_key`                 | Duplicate key in object                    |
| `invalid_object_key`            | Non-string key in object                   |
| `unclosed_container`            | Container missing `0xB6` end marker        |
| `invalid_data`                  | Generic invalid data (e.g., BigNumber NaN) |
| `value_out_of_range`            | Value exceeds allowed range                |
| `max_depth_exceeded`            | Container nesting too deep                 |
| `max_string_length_exceeded`    | String exceeds length limit                |
| `max_container_size_exceeded`   | Container has too many elements            |
| `max_document_size_exceeded`    | Document exceeds size limit                |
| `max_bignumber_exponent_exceeded` | BigNumber exponent exceeds configured limit |
| `max_bignumber_magnitude_exceeded`| BigNumber magnitude exceeds configured byte limit |

Implementations **MUST** map errors from their library to these standardized identifiers for test matching. If an implementation cannot distinguish between error types, it **MAY** treat any error as a successful match for error tests (verifying only that an error occurred, not its specific type).

When multiple error conditions could apply to malformed input (e.g., truncated data that also contains an invalid key), implementations **SHOULD** prioritize error detection in this order:
1. Structural errors (`truncated`, `invalid_type_code`, `unclosed_container`)
2. Type/format errors (`invalid_object_key`, `invalid_utf8`, `invalid_data`)
3. Content errors (`duplicate_key`, `nul_character`)
4. Limit errors (`max_depth_exceeded`, `max_string_length_exceeded`, `max_container_size_exceeded`, `max_document_size_exceeded`, `max_bignumber_exponent_exceeded`, `max_bignumber_magnitude_exceeded`)
5. Post-decode errors (`trailing_bytes`, `value_out_of_range`)

This ordering helps different implementations converge on the same error type for ambiguous cases, improving test interoperability.

If a test specifies an `expected_error` value that is not one of the recognized error types listed above, the test runner **MUST** skip the test and log a warning. This allows forward compatibility when new error types are added in future specification versions.


Test Execution
--------------

### Encode Test Execution

```pseudocode
function executeEncodeTest(test):
    value = parseValue(test.input)
    encoder = createEncoder(test.options)

    actualBytes = encoder.encode(value)
    expectedBytes = hexDecode(test.expected_bytes)

    assert actualBytes == expectedBytes
```

### Decode Test Execution

```pseudocode
function executeDecodeTest(test):
    inputBytes = hexDecode(test.input_bytes)
    decoder = createDecoder(test.options)

    actualValue = decoder.decode(inputBytes)
    expectedValue = parseValue(test.expected_value)

    assert valuesEqual(actualValue, expectedValue)
```

### Roundtrip Test Execution

```pseudocode
function executeRoundtripTest(test):
    originalValue = parseValue(test.input)
    encoder = createEncoder(test.options)
    decoder = createDecoder(test.options)

    encoded = encoder.encode(originalValue)
    decoded = decoder.decode(encoded)

    assert valuesEqual(decoded, originalValue)
```

### Error Test Execution

```pseudocode
function executeEncodeErrorTest(test):
    value = parseValue(test.input)
    encoder = createEncoder(test.options)

    try:
        encoder.encode(value)
        fail("expected error: " + test.expected_error)
    catch error:
        assert errorType(error) == test.expected_error

function executeDecodeErrorTest(test):
    inputBytes = hexDecode(test.input_bytes)
    decoder = createDecoder(test.options)

    try:
        decoder.decode(inputBytes)
        fail("expected error: " + test.expected_error)
    catch error:
        assert errorType(error) == test.expected_error
```

**Note on contradictory configurations**: Tests should not combine options that contradict the expected error. For example, a test expecting `trailing_bytes` error should not include `allow_trailing_bytes: true`. Such tests are considered malformed. Test runners **MAY** issue a warning but are not required to detect contradictory configurations.

### Value Parsing

```pseudocode
function parseValue(v):
    if v is null:
        return null

    if v is boolean or number or string:
        return v

    if v is array:
        return [parseValue(elem) for elem in v]

    if v is object:
        if "$number" in v:
            if v has keys other than "$number":
                error("marker object must only contain the marker key")
            return parseNumber(v["$number"])

        if "$bytes" in v:
            if v has keys other than "$bytes":
                error("marker object must only contain the marker key")
            return parseBytes(v["$bytes"])  // Returns raw byte sequence

        // Regular object - parse all values
        result = {}
        for key, value in v:
            result[key] = parseValue(value)
        return result

    error("unknown value type")
```

Note: Keys starting with `//` are only treated as comments at the top level of test case objects, not within `input`, `expected_value`, or other data fields. For example, in this test:

```json
{
  "//": "This is a comment and will be stripped",
  "name": "object_with_slash_key",
  "type": "roundtrip",
  "input": {"//": "This is NOT a comment - it is real data to encode"}
}
```

The `//` key in the test case object is stripped, but the `//` key inside `input` is preserved as actual data to be encoded. The test runner should strip comment keys from the test case object before processing, then pass the `input` value (with its `//` key intact) to `parseValue`.

### Value Comparison

Values **MUST** be compared for semantic equality:

- Numbers: Equal if they represent the same mathematical value (e.g., `1.0` equals `1`, `1e2` equals `100`). Exception: `-0.0` and `0.0` are considered distinct per IEEE 754.
- Strings: Equal if they contain the same Unicode code points (no normalization; `"é"` as U+00E9 does not equal `"é"` as U+0065 U+0301)
- Raw byte sequences (`$bytes`): Equal if they contain the same bytes in the same order. Byte-by-byte comparison is used, not Unicode comparison. This is relevant only for decode tests with `invalid_utf8: "pass_through"`.
- Arrays: Equal if they are the same length and all elements are equal and in the same order
- Objects: Equal if they have the same keys and every associated value is equal (order-independent)
- Special values: Quiet NaN equals quiet NaN; signaling NaN equals signaling NaN; but quiet NaN does NOT equal signaling NaN. Positive infinity and negative infinity are distinct values.

**Implementation note**: IEEE 754 defines NaN as not equal to anything, including itself. Implementations **MUST** use special comparison logic that checks both the NaN status and the signaling bit. The specific NaN payload bits (other than the signaling bit) **MUST** be ignored for test comparison purposes—two quiet NaNs are equal regardless of their payload bits, and two signaling NaNs are equal regardless of their payload bits.

When comparing `$number` values from test expectations against decoded results, the same mathematical equality rules apply. The test runner **SHOULD** parse both values to a common representation (e.g., arbitrary-precision decimal) before comparison.

**Precision guarantee**: BONJSON guarantees lossless round-trip for all numeric values it can encode. Test values use hex float notation (e.g., `0x1.921fb54442d18p+1`) when bit-exact precision is required. Comparison of such values **MUST** be exact (no epsilon tolerance). The only exception is NaN payloads, which are not preserved.


File Organization
-----------------

Test specifications **SHOULD** be organized into multiple files by category:

| File                          | Contents                        |
|-------------------------------|---------------------------------|
| `basic-types.json`            | null, boolean, empty containers |
| `integers.json`               | Integer encoding (all sizes)    |
| `floats.json`                 | Float encoding (32/64 bit)      |
| `strings.json`                | String encoding (short/long)    |
| `containers.json`             | Arrays and objects              |
| `bignumber.json`              | BigNumber encoding              |
| `errors.json`                 | Error handling                  |
| `security.json`               | Security-related tests          |
| `specification-examples.json` | Examples from BONJSON spec      |


Test Configuration File
-----------------------

A test configuration file allows test runners to load tests from multiple sources in a controlled order. This enables mixing and matching tests from different locations (local files and directories).

### Configuration Structure

A test configuration file is a JSON object with the following structure:

```json
{
  "type": "bonjson-test-config",
  "version": "1.0.0",
  "//": "My project's test configuration",
  "sources": [
    {"path": "./basic-types.json"},
    {"path": "./integers.json"},
    {"path": "./custom-tests/"}
  ]
}
```

| Field     | Type   | Description                                                               |
|-----------|--------|---------------------------------------------------------------------------|
| `type`    | string | Format identifier, **MUST** be `"bonjson-test-config"`                    |
| `version` | string | Semantic version of the configuration format (e.g., "1.0.0")              |
| `sources` | array  | Ordered array of test source objects                                      |

A missing required field is a **STRUCTURAL ERROR**.

An empty `sources` array is valid. This results in zero tests being executed.

As with test specification files, any key starting with `//` is treated as a comment and **MUST** be ignored by test runners.

Tests are executed in the order they appear in the `sources` array. Duplicate paths (exact string match) are only processed once; subsequent occurrences are silently skipped.

### Source Types

Each source object **MUST** be a JSON object containing a `path` field. An element of `sources` that is not a JSON object is a **STRUCTURAL ERROR**.

| Field       | Type    | Description                                         |
|-------------|---------|-----------------------------------------------------|
| `path`      | string  | Relative path to a file or directory                |
| `recursive` | boolean | Process subdirectories recursively (default: false) |
| `skip`      | boolean | Skip this source entirely (default: false)          |

A missing `path` field or wrong JSON type for `path`, `recursive`, or `skip` is a **STRUCTURAL ERROR**.

Paths are resolved by appending them to the configuration file's directory. This means paths are inherently relative; absolute paths will not work. An empty path string (`""`) or a path that does not exist is a **STRUCTURAL ERROR**.

The `recursive` option only affects paths that point to directories. If specified on a file path, it has no effect and the test runner **SHOULD** issue a warning.

The `skip` field allows temporarily disabling a source without removing it from the configuration.

```json
{
  "sources": [
    {"path": "./tests/basic-types.json"},
    {"path": "./all-tests/", "recursive": true},
    {"skip": true, "//": "Temporarily disabled until issue #42 is fixed", "path": "./broken-tests/"}
  ]
}
```

The test runner **MUST** log all skipped sources, including any comments it finds in the object.

e.g. `Skipping source at path "./broken-tests/": Temporarily disabled until issue #42 is fixed`

### Directory Processing

When a `path` source points to a directory:

1. Files are processed first, in alphabetical order (case-sensitive, byte-wise comparison)
2. Subdirectories are processed after all files, in alphabetical order (if `recursive` is `true`)
3. Only files ending in `.json` (case-insensitive) are loaded as test specifications
4. Files not ending in `.json` (case-insensitive) **MUST** be skipped, and the test runner **SHOULD** log what was skipped and why
5. Dotfiles and dot-directories (names starting with `.`) **MUST** be silently ignored (no logging)
6. If `recursive` is `false` or not specified, subdirectories are ignored with a log message
7. Malformed JSON in a `.json` file is a **STRUCTURAL ERROR**
8. A missing or unrecognized `type` field in a `.json` file is a **STRUCTURAL ERROR**
9. Configuration files (`type: "bonjson-test-config"`) encountered during directory processing **MUST** be skipped. The test runner **SHOULD** log the skipped file, unless it is the configuration file currently being processed (this handles the common case of a config that includes its own directory as a source).
10. If a directory contains no loadable test files, the test runner **SHOULD** log a warning (zero tests loaded from that directory is not an error)
11. Symbolic links **MUST** be followed (the test runner is not responsible for detecting cycles)

Note: Alphabetical ordering uses byte-wise comparison (ASCII order), meaning uppercase letters sort before lowercase (e.g., `Z.json` before `a.json`). This ensures deterministic ordering across all platforms regardless of locale settings.

Example processing a directory structure where `"recursive": true`:
```bash
tests/
├── .hidden/             # Silently ignored (dot-directory)
├── advanced/            # Processed after all files
│   └── bignumber.json   # Loaded 4th
├── basic/               # Processed after advanced/
│   ├── arrays.json      # Loaded 5th
│   └── objects.json     # Loaded 6th
├── floats.json          # Loaded 1st
├── integers.json        # Loaded 2nd
├── my-config.json       # Skipped (config file) - no log if this is the current config
├── strings.json         # Loaded 3rd
├── README.md            # Skipped with log message
└── notes.txt            # Skipped with log message
```

Schema
------

JSON Schemas are provided for validation:

- [Test specification schema](tests/bonjson-tests.schema.json) - validates test files
- [Configuration schema](tests/bonjson-test-config.schema.json) - validates configuration files

Implementations **SHOULD** validate files against the appropriate schema before execution.


Versioning
----------

This specification follows [Semantic Versioning](https://semver.org/):

- **Major**: Breaking changes to test format
- **Minor**: New features (backward compatible)
- **Patch**: Bug fixes and clarifications

Test files and configuration files **MUST** include a `version` field indicating the specification version they conform to. Both file types are versioned by this single specification and share the same version space.

The version **MUST** follow the full semver format: `MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]`. Examples:
- `1.0.0` - Release version
- `1.0.0-alpha` - Pre-release version
- `1.0.0-beta.2` - Pre-release with identifier
- `1.0.0+build.123` - Release with build metadata
- `1.0.0-rc.1+build.456` - Pre-release with build metadata

An invalid semver version string (e.g., `"1.0"` or `"abc"`) is a **STRUCTURAL ERROR**.

### Version Compatibility

When a test runner loads a file (whether configuration or test file), it **MUST** check that file's version independently:

- **Major version mismatch**: The test runner **MUST** exit with an error. Files with different major versions are incompatible.
- **Minor version higher than test runner**: The test runner **SHOULD** issue a warning but attempt to continue. The file may contain features the runner doesn't understand.
- **Minor version lower or equal**: Compatible. Proceed normally.
- **Patch version**: Ignored for compatibility purposes.
- **Pre-release versions**: Follow semver precedence rules. A pre-release version has lower precedence than its release version (e.g., `1.1.0-alpha < 1.1.0`), but higher precedence than earlier minor versions (e.g., `1.1.0-alpha > 1.0.0`).


Implementation Notes
--------------------

### Number Encoding Flexibility

BONJSON allows encoding numbers at the minimum precision that preserves the value. An `encode` test specifies the expected encoding, but implementations MAY choose different valid encodings.

For strict byte-level verification, use `encode` tests. For flexible verification, use `roundtrip` tests.

### Object Key Ordering

Like JSON, BONJSON doesn't specify an order for object keys to be encoded in. This makes `encode` tests containing objects with more than one key unreliable. `roundtrip` is better for such tests.

### Platform Differences

Some tests may not apply to all platforms:

- 64-bit integer tests may not work on platforms without 64-bit integer support
- BigNumber tests require arbitrary-precision arithmetic support

Implementations **SHOULD** skip tests that cannot be executed on the target platform and report them as skipped (not failed).


Runner Validation
-----------------

In addition to conformance tests that verify BONJSON encoding/decoding behavior, implementations **SHOULD** validate that their test runner correctly handles the test specification format. These concerns are kept separate:

- **Conformance tests**: Verify that the BONJSON codec works correctly
- **Runner validation**: Verify that the test runner correctly parses test files, skips tests appropriately, and detects malformed test specifications

### Directory Structure

Runner validation files are organized in the `test-runner-validation/` directory:

```
test-runner-validation/
├── README.md              # Documents expected behavior for each file
├── must-pass/             # Tests every runner must handle correctly
├── structural-errors/     # Files that should cause STRUCTURAL ERRORs
├── skip-scenarios/        # Tests that should be skipped with warnings
├── config/                # Config file processing tests
│   ├── errors/            # Config structural errors
│   └── directory-source/  # Test directory for directory processing
└── value-handling/        # Edge cases for value/hex parsing
```

### must-pass Directory

Contains test files that a correct test runner **MUST** process successfully. These verify basic functionality:

- All five test types (`encode`, `decode`, `roundtrip`, `encode_error`, `decode_error`)
- Comment key handling at document and test levels
- Hex string parsing (spaces, mixed case)
- `$number` marker parsing (all formats)
- Valid options with correct types
- Empty tests array

### structural-errors Directory

Contains files that the runner **MUST** reject with a structural error. Each file tests one specific error condition:

- Missing or invalid document fields (`type`, `version`, `tests`)
- Invalid test names (wrong characters, duplicates)
- Unknown test types
- Invalid hex strings
- Invalid `$number` values (empty, unparseable, `0x` without digits)
- Marker objects with unrecognized keys
- Option validation errors (wrong types, null, negative integers)
- Missing type-specific required fields

### skip-scenarios Directory

Contains valid test files where specific tests should be skipped with warnings:

- Tests with unrecognized option names
- Tests with unrecognized error types
- Tests requiring unsupported capabilities

The runner should parse these files successfully but skip the affected individual tests.

### config Directory

Contains configuration file tests:

- Valid configurations with various source types
- Directory processing (alphabetical order, recursive mode)
- Skip source handling
- Comment keys in config files
- `errors/` subdirectory with config files that should cause structural errors

### value-handling Directory

Contains tests for edge cases in value parsing and comparison:

- IEEE 754 negative zero (`-0.0` vs `0.0`)
- NaN comparison (NaN equals NaN for testing)
- Object comparison (order-independent)
- Mathematical equality across numeric types
- `//` keys in data values (not treated as comments)
- Version string handling (pre-release, build metadata)

### README.md

The `README.md` file in `test-runner-validation/` provides comprehensive documentation:

- Purpose and importance of runner validation
- Detailed tables listing every test file and its expected behavior
- Coverage checklist for implementers
- Instructions for manual and automated testing

Implementers integrate these validation checks into their own test suites using their language's native testing framework.


Appendix A: Type Code Reference
-------------------------------

| Range   | Type                                       |
|---------|--------------------------------------------|
| `00-64` | Small integers (0 to 100)                  |
| `65-a7` | Short strings (0-66 bytes)                 |
| `a8-ab` | Unsigned integers (8, 16, 32, 64 bit)      |
| `ac-af` | Signed integers (8, 16, 32, 64 bit)        |
| `b0`    | Float32                                    |
| `b1`    | Float64                                    |
| `b2`    | BigNumber                                  |
| `b3`    | Null                                       |
| `b4`    | False                                      |
| `b5`    | True                                       |
| `b6`    | Container end marker                       |
| `b7`    | Array                                      |
| `b8`    | Object                                     |
| `b9`    | Record definition                          |
| `ba`    | Record instance                            |
| `bb-f4` | Reserved                                   |
| `f5`    | Typed array float64                        |
| `f6`    | Typed array float32                        |
| `f7`    | Typed array sint64                         |
| `f8`    | Typed array sint32                         |
| `f9`    | Typed array sint16                         |
| `fa`    | Typed array sint8                          |
| `fb`    | Typed array uint64                         |
| `fc`    | Typed array uint32                         |
| `fd`    | Typed array uint16                         |
| `fe`    | Typed array uint8                          |
| `ff`    | Long string                                |


Appendix B: Complete Example
----------------------------

```json
{
  "type": "bonjson-test",
  "version": "1.0.0",
  "//": "Example test specification file",
  "tests": [
    {
      "//": "Null encodes to single byte 0xB3",
      "name": "null_value",
      "type": "encode",
      "input": null,
      "expected_bytes": "b3"
    },
    {
      "//": "Small integer 42 encodes as type_code = 42 = 0x2a",
      "name": "smallint_42",
      "type": "encode",
      "input": 42,
      "expected_bytes": "2a"
    },
    {
      "//": "Truncated sint16 should fail (0xAD = sint16, missing second byte)",
      "name": "truncated_int16",
      "type": "decode_error",
      "input_bytes": "ade8",
      "expected_error": "truncated"
    },
    {
      "//": "Complex object round-trips correctly",
      "name": "roundtrip_complex",
      "type": "roundtrip",
      "input": {
        "name": "test",
        "values": [1, 2, 3],
        "nested": {"flag": true}
      }
    },
    {
      "//": "NaN cannot be encoded by default",
      "name": "encode_nan_error",
      "type": "encode_error",
      "input": {"$number": "NaN"},
      "expected_error": "invalid_data"
    },
    {
      "//": "Large unsigned integer (0xAB = uint64)",
      "name": "uint64_large",
      "type": "encode",
      "input": {"$number": "18446744073709551615"},
      "expected_bytes": "ab ff ff ff ff ff ff ff ff"
    },
    {
      "//": "BigNumber decodes to 1.5 (0xB2 = BigNumber, zigzag LEB128 + LE magnitude)",
      "name": "bignumber_1_5",
      "type": "decode",
      "input_bytes": "b2 01 02 0f",
      "expected_value": {"$number": "1.5"}
    },
    {
      "//": "Non-string object key should fail (object start, key is integer 0 instead of string, value is integer 1, end)",
      "name": "invalid_object_key",
      "type": "decode_error",
      "input_bytes": "b8 00 01 b6",
      "expected_error": "invalid_object_key"
    }
  ]
}
```
