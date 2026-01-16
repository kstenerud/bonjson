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
  - [Data Representation](#data-representation)
    - [Binary Data](#binary-data)
    - [JSON Values](#json-values)
    - [Special Values](#special-values)
      - [Numbers](#numbers)
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
- Field names (`"expected_bytes"`, not `"Expected_Bytes"`)
- Marker keys (`"$number"`, not `"$Number"`)

The following elements are **case-insensitive**:

- Test `name` field (for duplicate detection only; names may contain any case)
- `$number` special values (`"NaN"`, `"Infinity"`, `"-Infinity"`)
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
  "expected_bytes": "79 e8 03"
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
  "name": "decode_float16",
  "type": "decode",
  "input_bytes": "6a 90 3f",
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
  "input_bytes": "79 e8",
  "expected_error": "truncated"
}
```

### Optional Fields

#### Options

The `options` field is an optional JSON object that configures encoder/decoder behavior for a specific test. If present, it **MUST** be a JSON object; any other type (e.g., string, array, null) is a **STRUCTURAL ERROR**.

| Option                 | Type    | Description                                            |
|------------------------|---------|--------------------------------------------------------|
| `allow_nul`            | boolean | Allow NUL characters in strings                        |
| `allow_nan_infinity`   | boolean | Allow NaN and Infinity values                          |
| `allow_trailing_bytes` | boolean | Allow unconsumed bytes after decoding (default: false) |
| `max_depth`            | integer | Maximum container nesting depth (non-negative)         |
| `max_container_size`   | integer | Maximum elements in a container (non-negative)         |
| `max_string_length`    | integer | Maximum string length in bytes (non-negative)          |
| `max_chunks`           | integer | Maximum string chunks (non-negative)                   |
| `max_document_size`    | integer | Maximum document size in bytes (non-negative)          |

Option values **MUST** have the correct JSON type (boolean for boolean options, integer for integer options). Null is not a valid option value. An option with the wrong type (e.g., `"allow_nul": "true"` or `"allow_nul": null`) is a **STRUCTURAL ERROR**. Integer options **MUST** be non-negative; negative values are a **STRUCTURAL ERROR**.

Default values for these options are defined in the BONJSON specification. Tests typically use small values (e.g., `max_depth: 5`) to verify that limit-checking machinery works correctly in an implementation.

Test runners **MUST** skip (and log a warning for) any tests that contain:
- Option names not recognized by the test runner (e.g., typos like `"alow_nul"`)
- Option settings not supported by the underlying library

Note: Setting options that contradict the expected outcome (e.g., `allow_nan_infinity: true` on an `encode_error` test expecting NaN to fail) is a malformed test. Test authors are responsible for writing sensible tests; test runners are not required to detect such contradictions.

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


Data Representation
-------------------

### Binary Data

Binary data (byte sequences) is represented as a hexadecimal string (uppercase or lowercase) with optional spaces for readability. An empty string (`""`) represents zero bytes, which is useful for testing how implementations handle empty input.

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

Marker objects **MUST** contain only the marker key. Additional keys are a **STRUCTURAL ERROR**. For example, `{"$number": "1.5", "extra": "field"}` is invalid.

#### Numbers

The `$number` marker represents stringified numeric values that cannot be safely represented as JSON numbers:

```json
{"$number": "NaN"}
{"$number": "Infinity"}
{"$number": "-Infinity"}
{"$number": "18446744073709551615"}
{"$number": "0x1.921fb54442d18p+1"}
{"$number": "1.23456789e-5"}
```

The test runner parses the string based on its format:

| Format                         | Interpretation                                                                     |
|--------------------------------|------------------------------------------------------------------------------------|
| `NaN`, `Infinity`, `-Infinity` | IEEE 754 special float values (case-insensitive)                                   |
| `0x...p...`                    | IEEE 754 hex float (C99 `%a` format) for precise representation (case-insensitive) |
| `0x...` (no `p`)               | Hex integer (case-insensitive)                                                     |
| Decimal integer or float       | Arbitrary-precision decimal number (case-insensitive for `e`/`E` exponent)         |

**Hex integer format**: Standard hexadecimal integer format `[±]0xH...` where `H...` are hex digits. At least one hex digit is required (i.e., `0x` alone is invalid). Parsed as an integer, or BigNumber if too large for native integer types. Examples: `0x100` (256), `-0xff` (-255), `0xffffffffffffffff` (2^64-1).

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
- Integer strings (no decimal point or exponent, e.g., `"1"`, `"100"`) → integer
- Hex integers (e.g., `"0x100"`) → integer
- Decimal or scientific notation (e.g., `"1.0"`, `"1e2"`) → float
- Hex floats → float with the exact bit pattern specified
- Values exceeding native integer/float range → BigNumber

Passing float values like `"1.0"` to the encoder allows tests to verify that the encoder correctly optimizes whole-number floats into smaller integer representations.

The `$number` string **MUST NOT** be empty and **MUST NOT** contain leading or trailing spaces. An unparseable `$number` value (e.g., `{"$number": "hello"}`) is a **STRUCTURAL ERROR**.


Error Types
-----------

The `expected_error` field uses standardized error type identifiers:

| Error Type                      | Description                                |
|---------------------------------|--------------------------------------------|
| `truncated`                     | Unexpected end of input data               |
| `trailing_bytes`                | Unconsumed bytes after decoding            |
| `invalid_type_code`             | Unrecognized or reserved type code         |
| `invalid_utf8`                  | Invalid UTF-8 byte sequence                |
| `nul_character`                 | NUL (0x00) byte in string                  |
| `duplicate_key`                 | Duplicate key in object                    |
| `unclosed_container`            | Missing container end marker               |
| `invalid_data`                  | Generic invalid data (e.g., BigNumber NaN) |
| `value_out_of_range`            | Value exceeds allowed range                |
| `non_canonical_length`          | Non-canonical (oversized) length encoding  |
| `too_many_chunks`               | String exceeds chunk count limit           |
| `empty_chunk_continuation`      | Zero-length chunk with continuation bit    |
| `max_depth_exceeded`            | Container nesting too deep                 |
| `max_string_length_exceeded`    | String exceeds length limit                |
| `max_container_size_exceeded`   | Container has too many elements            |
| `max_document_size_exceeded`    | Document exceeds size limit                |

Implementations **MUST** map errors from their library to these standardized identifiers for test matching. If an implementation cannot distinguish between error types, it **MAY** treat any error as a successful match for error tests (verifying only that an error occurred, not its specific type).

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
- Arrays: Equal if they are the same length and all elements are equal and in the same order
- Objects: Equal if they have the same keys and every associated value is equal (order-independent)
- Special values: NaN equals NaN for testing purposes; positive infinity and negative infinity are distinct values

**Implementation note**: IEEE 754 defines NaN as not equal to anything, including itself. Implementations must use `isnan(a) && isnan(b)` (or equivalent) rather than `a == b` when comparing NaN values. The specific NaN payload is irrelevant for test comparison purposes.

When comparing `$number` values from test expectations against decoded results, the same mathematical equality rules apply. The test runner **SHOULD** parse both values to a common representation (e.g., arbitrary-precision decimal) before comparison. For floating-point values that cannot be exactly represented, comparison **SHOULD** use the closest representable value in the target type.


File Organization
-----------------

Test specifications **SHOULD** be organized into multiple files by category:

| File                          | Contents                        |
|-------------------------------|---------------------------------|
| `basic-types.json`            | null, boolean, empty containers |
| `integers.json`               | Integer encoding (all sizes)    |
| `floats.json`                 | Float encoding (16/32/64 bit)   |
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


Appendix A: Type Code Reference
-------------------------------

| Range   | Type                                 |
|---------|--------------------------------------|
| `00-64` | Small positive integers (0-100)      |
| `65-67` | Reserved                             |
| `68`    | Long string                          |
| `69`    | BigNumber                            |
| `6a`    | Float16 (bfloat16)                   |
| `6b`    | Float32                              |
| `6c`    | Float64                              |
| `6d`    | Null                                 |
| `6e`    | False                                |
| `6f`    | True                                 |
| `70-77` | Unsigned integers (8-64 bit)         |
| `78-7f` | Signed integers (8-64 bit)           |
| `80-8f` | Short strings (0-15 bytes)           |
| `90-98` | Reserved                             |
| `99`    | Array start                          |
| `9a`    | Object start                         |
| `9b`    | Container end                        |
| `9c-ff` | Small negative integers (-100 to -1) |


Appendix B: Complete Example
----------------------------

```json
{
  "type": "bonjson-test",
  "version": "1.0.0",
  "//": "Example test specification file",
  "tests": [
    {
      "//": "Null encodes to single byte 0x6d",
      "name": "null_value",
      "type": "encode",
      "input": null,
      "expected_bytes": "6d"
    },
    {
      "//": "Small integer 42 encodes directly",
      "name": "smallint_42",
      "type": "encode",
      "input": 42,
      "expected_bytes": "2a"
    },
    {
      "//": "Truncated int16 should fail",
      "name": "truncated_int16",
      "type": "decode_error",
      "input_bytes": "79e8",
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
      "//": "Large unsigned integer",
      "name": "uint64_large",
      "type": "encode",
      "input": {"$number": "18446744073709551615"},
      "expected_bytes": "77 ff ff ff ff ff ff ff ff"
    },
    {
      "//": "BigNumber decodes to 1.5",
      "name": "bignumber_1_5",
      "type": "decode",
      "input_bytes": "69 0a ff 0f",
      "expected_value": {"$number": "1.5"}
    }
  ]
}
```
