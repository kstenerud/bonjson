# BONJSON Test Specifications

This directory contains cross-implementation test specifications for BONJSON. These tests serve as a "gold standard" for verifying that any BONJSON implementation correctly handles encoding, decoding, and error cases.

Everything is set up to make it easy for an agent to build. An agent will need to:
- Vendor in this repository
- **FIRST**: Validate the test runner against the tests in `test-runner-validation`
- **THEN**: Validate the codec against the tests in `conformance`

## Purpose

- **Single source of truth**: Define tests once, use across all implementations
- **Consistency**: Ensure all implementations handle edge cases identically
- **Validation**: New implementations can verify correctness against these tests
- **Regression prevention**: Changes to the spec are reflected in tests

There are two directories under this one that contain tests for specific purposes:

### test-runner-validation/ (CRITICAL - Run First!)

⚠️ **These tests validate that your test runner works correctly.** If these tests fail, your conformance test results are unreliable and must not be trusted.

These tests ensure your test runner correctly:
- Parses BONJSON test specification files
- Handles all five test types (encode, decode, roundtrip, encode_error, decode_error)
- Parses hex byte strings in various formats
- Compares numeric values correctly (int/float equality, NaN comparison)
- Processes the `$number` marker for special values
- Handles test options and error types

**Fix any failures here BEFORE investigating conformance test failures.**

See [bonjson-universal-test-specification.md](../bonjson-universal-test-specification.md) for the test specification format.

### conformance/

Tests that validate your BONJSON codec implementation. These verify that your encoder produces correct bytes and your decoder correctly interprets BONJSON data.

See [bonjson.md](../bonjson.md) for the BONJSON format specification.

Note: Universal tests can't be exhaustive because each language and platform will have its own ideosyncrasies. You will have to write your own tests to cover things specific to your implementation. These tests are designed to get you 80% of the way there.

## File Organization

```
tests/
├── README.md                        # This file
├── bonjson-tests.schema.json        # JSON Schema for test files
├── bonjson-test-config.schema.json  # JSON Schema for config files
├── test-runner-validation/          # CRITICAL: Tests for test runner implementations
│   ├── README.md                    # Documentation for runner validation tests
│   ├── must-pass/                   # Basic tests every runner must handle correctly
│   ├── structural-errors/           # Tests that should cause runner errors
│   ├── skip-scenarios/              # Tests that should be skipped by runners
│   ├── config/                      # Config file processing tests
│   └── value-handling/              # Numeric comparison, NaN, trailing bytes, etc.
└── conformance/                     # BONJSON codec conformance test suite
    ├── config.json                  # Test configuration file
    ├── basic-types.json             # Primitive type encoding tests
    ├── integers.json                # Integer encoding at all sizes
    ├── floats.json                  # Float encoding tests
    ├── strings.json                 # String encoding (short, long)
    ├── containers.json              # Arrays and objects
    ├── bignumber.json               # BigNumber encoding tests
    ├── errors.json                  # Error handling tests
    ├── security.json                # Security-related tests
    └── specification-examples.json  # Examples from the specification
```

## Test Case Format

All test files are JSON with the following structure:

```json
{
  "type": "bonjson-test",
  "version": "1.0.0",
  "tests": [
    {
      "//": "Optional comment explaining the test",
      "name": "test_name",
      "type": "encode",
      "input": <value>,
      "expected_bytes": "<hex>"
    }
  ]
}
```

The `type` field identifies this as a BONJSON test specification (for use in multi-format test systems).

### Comment Convention

Any key starting with `//` is ignored by test runners. The primary convention is using `//` for comments:

```json
{
  "//": "Integer 1000 requires sint16 encoding (type code 0xe5)",
  "name": "int16_1000",
  "type": "encode",
  "input": 1000,
  "expected_bytes": "e5e803"
}
```

### Comment-Only Entries (Section Dividers)

Test entries that contain **only** keys starting with `//` are treated as comment blocks and silently skipped by test runners. This allows organizing tests into logical sections:

```json
{
  "tests": [
    {
      "//": "=== Integer encoding tests ===",
      "//note": "These tests verify correct integer type selection"
    },
    {
      "name": "int8_positive",
      "type": "encode",
      "input": 100,
      "expected_bytes": "c8"
    },
    {
      "//": "=== Negative integers ==="
    },
    {
      "name": "int8_negative",
      "type": "encode",
      "input": -1,
      "expected_bytes": "63"
    }
  ]
}
```

If an entry contains any non-comment keys (keys not starting with `//`), it must include the required `name` and `type` fields.

## Test Types

Test type values are case-sensitive (must be lowercase: `"encode"`, `"decode"`, etc.).

### `encode` - Value to Bytes

Verifies that encoding a value produces specific bytes.

```json
{
  "name": "encode_int_1000",
  "type": "encode",
  "input": 1000,
  "expected_bytes": "e5e803"
}
```

### `decode` - Bytes to Value

Verifies that decoding bytes produces a specific value.

```json
{
  "name": "decode_float32",
  "type": "decode",
  "input_bytes": "cb0000903f",
  "expected_value": 1.125
}
```

### `roundtrip` - Encode then Decode

Verifies a value survives encoding and decoding unchanged. No specific byte sequence is verified.

```json
{
  "name": "roundtrip_nested_object",
  "type": "roundtrip",
  "input": {
    "name": "test",
    "values": [1, 2, 3]
  }
}
```

### `encode_error` - Encoding Should Fail

Verifies that certain values cannot be encoded.

```json
{
  "name": "encode_error_nan",
  "type": "encode_error",
  "input": {"$number": "NaN"},
  "expected_error": "invalid_data"
}
```

### `decode_error` - Decoding Should Fail

Verifies that certain byte sequences are rejected.

```json
{
  "name": "decode_error_truncated_int16",
  "type": "decode_error",
  "input_bytes": "e5e8",
  "expected_error": "truncated"
}
```

## Special Values

JSON cannot directly represent some numeric values. Use the `$number` marker:

```json
{"$number": "NaN"}
{"$number": "sNaN"}
{"$number": "Infinity"}
{"$number": "-Infinity"}
{"$number": "18446744073709551615"}
{"$number": "0x1.921fb54442d18p+1"}
{"$number": "1.23456789e-5"}
```

| Format                                 | Interpretation                                                                     |
|----------------------------------------|------------------------------------------------------------------------------------|
| `NaN`, `sNaN`, `Infinity`, `-Infinity` | IEEE 754 special values (case-insensitive). `NaN` is quiet NaN; `sNaN` is signaling NaN. |
| `0x...p...`                    | Hex float (C99 format, case-insensitive)                  |
| Decimal                        | Arbitrary-precision number (case-insensitive for `e`/`E`) |

For raw byte sequences in string contexts (testing `invalid_utf8: "pass_through"`), use the `$bytes` marker:

```json
{"$bytes": "68 65 6c 6c 6f ff 77 6f 72 6c 64"}
```

Tests using `$bytes` must include `requires: ["raw_string_bytes"]`.

## Binary Data Format

Bytes are represented as hex strings (case insensitive). Whitespace is allowed for readability and must be stripped by test runners:

```json
{
  "expected_bytes": "9a 86 6e 75 6d 62 65 72 32"
}
```

Compact format (no spaces) is also valid:

```json
{
  "expected_bytes": "9a866e756d62657232"
}
```

## Error Types

Error type values are case-sensitive (must be lowercase).

| Error                         | Description                                   |
|-------------------------------|-----------------------------------------------|
| `truncated`                   | Unexpected end of data                        |
| `trailing_bytes`              | Unconsumed bytes after decoding               |
| `invalid_type_code`           | Unrecognized or reserved type code            |
| `invalid_utf8`                | Invalid UTF-8 sequence in string              |
| `nul_character`               | NUL (0x00) in string (default rejection)      |
| `duplicate_key`               | Duplicate key in object                       |
| `unclosed_container`          | Container missing `0xB3` end marker           |
| `invalid_data`                | Generic invalid data (e.g., NaN in BigNumber) |
| `value_out_of_range`          | Value exceeds allowed range                   |
| `max_depth_exceeded`          | Container nesting too deep                    |
| `max_string_length_exceeded`  | String exceeds length limit                   |
| `max_container_size_exceeded` | Container has too many elements               |
| `max_document_size_exceeded`  | Document exceeds size limit                   |

## Test Options

Some tests require non-default decoder/encoder settings. Option names are case-sensitive (must be lowercase):

```json
{
  "name": "nul_allowed",
  "type": "roundtrip",
  "input": "hello\u0000world",
  "options": {
    "allow_nul": true
  }
}
```

Available options:
- `allow_nul`: Allow NUL characters in strings (boolean)
- `allow_trailing_bytes`: Allow unconsumed bytes after decoding (boolean)
- `nan_infinity_behavior`: How to handle NaN/Infinity: `"reject"`, `"allow"`, or `"stringify"` (string)
- `duplicate_key`: How to handle duplicate object keys: `"reject"`, `"keep_first"`, or `"keep_last"` (string)
- `invalid_utf8`: How to handle invalid UTF-8: `"reject"`, `"replace"`, `"delete"`, or `"pass_through"` (string)
- `max_depth`: Maximum container nesting depth (integer)
- `max_container_size`: Maximum elements in a container (integer)
- `max_string_length`: Maximum string length in bytes (integer)
- `max_document_size`: Maximum document size in bytes (integer)

## Required Capabilities

Some tests require capabilities that not all implementations support. Use the `requires` field to declare these dependencies. Implementations should skip tests requiring capabilities they don't support.

```json
{
  "name": "bignumber_high_precision",
  "type": "roundtrip",
  "input": {"$number": "1.23456789012345678901234567890"},
  "requires": ["arbitrary_precision_bignumber"]
}
```

Available capabilities:

| Capability                       | Description                                                                                                                                                                             |
|----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `arbitrary_precision_bignumber`  | Support for BigNumber values with more than ~17 significant digits (exceeds float64 precision). Implementations using float64 for decoded BigNumbers will lose precision and should skip these tests. |
| `bignumber_exponent_gt_127`      | Support for BigNumber exponents greater than 127. Some implementations (e.g., Swift's Decimal) limit exponents to -128 to 127.                                                          |
| `bignumber_exponent_lt_neg128`   | Support for BigNumber exponents less than -128. Some implementations limit exponents to -128 to 127.                                                                                    |
| `nan_infinity_stringify`         | Support for the `nan_infinity_behavior: "stringify"` option, which converts NaN/Infinity float values to string representations. Not all implementations support this mode.             |
| `uint64`                         | Support for full 64-bit unsigned integers. Some platforms (e.g., JavaScript) cannot represent integers beyond 2^53-1.                                                                   |
| `int64`                          | Support for full 64-bit signed integers. Some platforms cannot represent integers beyond ±2^53-1.                                                                                       |
| `negative_zero`                  | Support for IEEE 754 negative zero (-0.0) preservation. Some platforms or type systems cannot distinguish -0.0 from +0.0.                                                               |
| `raw_string_bytes`               | Support for representing strings as raw byte sequences (for testing `invalid_utf8: "pass_through"`). Implementations using native string types that require valid UTF-8 cannot support this. |
| `signaling_nan`                  | Support for preserving the signaling bit of NaN values. Most platforms convert signaling NaN to quiet NaN on any operation.                                                          |

Test runners should:
1. Define which capabilities their implementation supports
2. Skip tests whose `requires` array contains unsupported capabilities
3. Report skipped tests with the reason (missing capability)

## Implementing a Test Runner

Each implementation needs a test runner that:

1. Reads JSON test specification files
2. Parses test cases (ignoring keys starting with `//`)
3. Executes tests based on type
4. Reports pass/fail status with file path and test name (e.g., `integers.json:int16_overflow`)

### Pseudocode

```
// Define which capabilities this implementation supports
supportedCapabilities = {"arbitrary_precision_bignumber", "bignumber_exponent_gt_127"}

function runTests(specFile):
    spec = parseJSON(specFile)

    for test in spec.tests:
        // Skip comment keys
        if test starts with "//": continue

        // Skip tests requiring unsupported capabilities
        if test.requires:
            for cap in test.requires:
                if cap not in supportedCapabilities:
                    skip(test.name, "requires " + cap)
                    continue

        switch test.type:
            case "encode":
                actual = encode(parseValue(test.input))
                assert actual == hexDecode(test.expected_bytes)

            case "decode":
                actual = decode(hexDecode(test.input_bytes))
                assert actual == parseValue(test.expected_value)

            case "roundtrip":
                original = parseValue(test.input)
                encoded = encode(original)
                decoded = decode(encoded)
                assert decoded == original

            case "encode_error":
                try:
                    encode(parseValue(test.input))
                    fail("expected error")
                catch error:
                    assert errorType(error) == test.expected_error

            case "decode_error":
                try:
                    decode(hexDecode(test.input_bytes))
                    fail("expected error")
                catch error:
                    assert errorType(error) == test.expected_error

function parseValue(v):
    if v is object with "$number":
        return parseNumber(v["$number"])
    // ... handle other special markers
    return v  // Regular JSON value
```

## Test Configuration File

A configuration file allows test runners to load tests from multiple sources:

```json
{
  "type": "bonjson-test-config",
  "version": "1.0.0",
  "//": "My project's test configuration",
  "sources": [
    {"path": "./basic-types.json"},
    {"path": "./integers.json"},
    {"path": "./custom-tests/", "recursive": true},
    {"//": "Disabled until issue #42 fixed", "path": "./broken-tests/", "skip": true}
  ]
}
```

- Keys starting with `//` are treated as comments (ignored by test runners)
- Tests execute in the order sources are listed
- `path`: relative path to a file or directory (resolved from config file location)
- `recursive` (optional): process subdirectories recursively (default: false)
- `skip` (optional): skip this source entirely (default: false)
- Directory processing: files first (alphabetically by ASCII/byte order), then subdirectories; only `.json` files loaded
- Dotfiles/directories are silently ignored; other non-JSON files are skipped with a log message
- Malformed JSON files cause the test runner to exit with an error

## Validation

Files can be validated against their respective schemas:

```bash
# Validate a test file
npx ajv validate -s bonjson-tests.schema.json -d basic-types.json

# Validate a config file
npx ajv validate -s bonjson-test-config.schema.json -d my-config.json
```

## Contributing Tests

When adding new tests:

1. Choose the appropriate file based on test category
2. Use descriptive `name` values (snake_case recommended)
3. Ensure names are unique within the file (compared case-insensitively; e.g., `Integer_overflow` and `integer_overflow` would conflict)
4. Add a `//` comment explaining non-obvious tests
5. Validate against the schema before committing (note: schema requires lowercase `type` values)
6. Ensure the test passes on at least one implementation

## Tests to Avoid

These conformance tests are meant to verify BONJSON format correctness across all implementations. Avoid tests that are inherently language or implementation-specific:

### 1. Type representation in generic containers

**Don't test** what type is returned when decoding into a generic type like `interface{}`, `any`, `Object`, or `dynamic`.

Different languages return different types for the same BONJSON value:
- A BigNumber might decode to `float64`, `string`, `BigInt`, `Decimal`, or `BigInteger` depending on the language
- An integer might decode to `int64`, `number`, `long`, or `BigInt`

These are valid implementation choices, not format errors.

### 2. Encode tests for values with multiple valid encodings

**Don't use `encode` tests** for values that have multiple valid byte representations.

For example, the integer `180` can validly be encoded as:
- `e0 b4` (uint8)
- `e5 b4 00` (sint16)

Both are correct per the BONJSON specification. Instead, use:
- **`roundtrip`** tests to verify the value survives encoding/decoding
- **`decode`** tests to verify the implementation accepts all valid encodings

### 3. Tests requiring parsing of extreme values

**Don't assume** test runners can parse arbitrary numeric strings.

Values like `"1e1000"` or 100-digit integers may overflow standard parsing functions in some languages. If such values are essential to test, ensure they're used in `decode` tests with pre-encoded bytes rather than `roundtrip` tests that require parsing.

### 4. Unrealistic configuration limits

**Use realistic limits** when testing configurable options.

For example, don't test `max_string_length: 10` with short strings (0-15 bytes) - no real implementation would set limits that low, and checking limits on every short string would be wasteful. Test with long strings and limits > 16 bytes instead.

### 5. Implementation-specific error distinctions

**Accept equivalent errors** when the distinction is semantic rather than format-based.

For example, "truncated data" and "unclosed container" might be the same error in implementations that don't track container state separately. Test for the behavior (rejection of invalid input) rather than specific error taxonomy.

## Type Code Reference

For reference when writing byte sequences:

| Range   | Type                                 |
|---------|--------------------------------------|
| `00-64` | Small integers (0 to 100)            |
| `65-a4` | Short strings (0-63 bytes)           |
| `a5-a8` | Unsigned integers (8, 16, 32, 64 bit)|
| `a9-ac` | Signed integers (8, 16, 32, 64 bit)  |
| `ad`    | Float32                              |
| `ae`    | Float64                              |
| `af`    | BigNumber                            |
| `b0`    | False                                |
| `b1`    | True                                 |
| `b2`    | Null                                 |
| `b3`    | Container end marker                 |
| `b4`    | Array                                |
| `b5`    | Object                               |
| `b6`    | Record definition                    |
| `b7`    | Record instance                      |
| `b8-f4` | Reserved                             |
| `f5`    | Typed array float64                  |
| `f6`    | Typed array float32                  |
| `f7`    | Typed array sint64                   |
| `f8`    | Typed array sint32                   |
| `f9`    | Typed array sint16                   |
| `fa`    | Typed array sint8                    |
| `fb`    | Typed array uint64                   |
| `fc`    | Typed array uint32                   |
| `fd`    | Typed array uint16                   |
| `fe`    | Typed array uint8                    |
| `ff`    | Long string                          |
