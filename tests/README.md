# BONJSON Test Specifications

This directory contains cross-implementation test specifications for BONJSON. These tests serve as a "gold standard" for verifying that any BONJSON implementation correctly handles encoding, decoding, and error cases.

## Purpose

- **Single source of truth**: Define tests once, use across all implementations
- **Consistency**: Ensure all implementations handle edge cases identically
- **Validation**: New implementations can verify correctness against these tests
- **Regression prevention**: Changes to the spec are reflected in tests

## File Organization

```
tests/
├── README.md                        # This file
├── bonjson-tests.schema.json        # JSON Schema for test files
├── bonjson-test-config.schema.json  # JSON Schema for config files
├── conformance/                     # BONJSON conformance test suite
│   ├── config.json                  # Test configuration file
│   ├── basic-types.json             # Primitive type encoding tests
│   ├── integers.json                # Integer encoding at all sizes
│   ├── floats.json                  # Float encoding tests
│   ├── strings.json                 # String encoding (short, long, chunked)
│   ├── containers.json              # Arrays and objects
│   ├── bignumber.json               # BigNumber encoding tests
│   ├── errors.json                  # Error handling tests
│   ├── security.json                # Security-related tests
│   └── specification-examples.json  # Examples from the specification
└── runner/                          # Tests for test runner implementations
    ├── README.md                    # Documentation for runner tests
    ├── valid/                       # Tests that should pass
    ├── structural-errors/           # Tests that should cause errors
    ├── skip-scenarios/              # Tests that should be skipped
    ├── config/                      # Config file processing tests
    └── special-values/              # Edge cases for value parsing
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
  "//": "Integer 1000 requires int16 encoding (type code 0x79)",
  "name": "int16_1000",
  "type": "encode",
  "input": 1000,
  "expected_bytes": "79e803"
}
```

## Test Types

Test type values are case-insensitive.

### `encode` - Value to Bytes

Verifies that encoding a value produces specific bytes.

```json
{
  "name": "encode_int_1000",
  "type": "encode",
  "input": 1000,
  "expected_bytes": "79e803"
}
```

### `decode` - Bytes to Value

Verifies that decoding bytes produces a specific value.

```json
{
  "name": "decode_float16",
  "type": "decode",
  "input_bytes": "6a903f",
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
  "input_bytes": "79e8",
  "expected_error": "truncated"
}
```

## Special Values

JSON cannot directly represent some numeric values. Use the `$number` marker:

```json
{"$number": "NaN"}
{"$number": "Infinity"}
{"$number": "-Infinity"}
{"$number": "18446744073709551615"}
{"$number": "0x1.921fb54442d18p+1"}
{"$number": "1.23456789e-5"}
```

| Format | Interpretation |
|--------|----------------|
| `NaN`, `Infinity`, `-Infinity` | IEEE 754 special values (case-insensitive) |
| `0x...p...` | Hex float (C99 format, case-insensitive) |
| Decimal | Arbitrary-precision number (case-insensitive for `e`/`E`) |

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

Error type values are case-insensitive.

| Error | Description |
|-------|-------------|
| `truncated` | Unexpected end of data |
| `invalid_type_code` | Unrecognized or reserved type code |
| `invalid_utf8` | Invalid UTF-8 sequence in string |
| `nul_character` | NUL (0x00) in string (default rejection) |
| `duplicate_key` | Duplicate key in object |
| `unclosed_container` | Missing container end marker |
| `invalid_data` | Generic invalid data (e.g., NaN in BigNumber) |
| `value_out_of_range` | Value exceeds allowed range |
| `too_many_chunks` | String has too many chunks |
| `empty_chunk_continuation` | Empty chunk with continuation bit set |
| `max_depth_exceeded` | Container nesting too deep |
| `max_string_length_exceeded` | String exceeds length limit |

## Test Options

Some tests require non-default decoder/encoder settings. Option names are case-insensitive:

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
- `allow_nul`: Allow NUL characters in strings
- `allow_nan_infinity`: Allow NaN and Infinity values
- `max_depth`: Maximum container nesting depth
- `max_string_length`: Maximum string length
- `max_chunks`: Maximum number of string chunks

## Implementing a Test Runner

Each implementation needs a test runner that:

1. Reads JSON test specification files
2. Parses test cases (ignoring keys starting with `//`)
3. Executes tests based on type
4. Reports pass/fail status with file path and test name (e.g., `integers.json:int16_overflow`)

### Pseudocode

```
function runTests(specFile):
    spec = parseJSON(specFile)

    for test in spec.tests:
        // Skip comment keys
        if test starts with "//": continue

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
- `70 b4` (uint8)
- `79 b4 00` (sint16)

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

| Range | Type |
|-------|------|
| `00-64` | Small positive integers (0-100) |
| `65-67` | Reserved |
| `68` | Long string |
| `69` | BigNumber |
| `6a` | Float16 (bfloat16) |
| `6b` | Float32 |
| `6c` | Float64 |
| `6d` | Null |
| `6e` | False |
| `6f` | True |
| `70-77` | Unsigned integers (8-64 bit) |
| `78-7f` | Signed integers (8-64 bit) |
| `80-8f` | Short strings (0-15 bytes) |
| `90-98` | Reserved |
| `99` | Array start |
| `9a` | Object start |
| `9b` | Container end |
| `9c-ff` | Small negative integers (-100 to -1) |
