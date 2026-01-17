# BONJSON Test Runner Tests

This directory contains tests designed to verify that a BONJSON test runner implementation correctly handles all aspects of the BONJSON test specification.

## Purpose

These tests exercise the test runner's:
- Document parsing and validation
- Comment handling
- Test name validation
- Hex string parsing
- `$number` marker parsing
- Options validation
- Config file processing
- Directory traversal
- Value comparison semantics

## Directory Structure

```
runner/
├── README.md                    # This file
├── valid/                       # Tests the runner should process successfully
├── structural-errors/           # Tests that should cause STRUCTURAL ERRORs
├── skip-scenarios/              # Tests that should be skipped with warnings
├── config/                      # Config file processing tests
│   ├── errors/                  # Config structural errors
│   └── directory-source/        # Test directory for directory processing
└── special-values/              # Edge cases for value/hex parsing
```

## Test Categories

### valid/

Contains test files that a correct test runner should process successfully. Use these to verify basic functionality:

| File | Tests |
|------|-------|
| `basic-test-types.json` | All five test types (encode, decode, roundtrip, encode_error, decode_error) |
| `comments.json` | Comment keys at document and test levels |
| `comment-only-entries.json` | Comment-only entries (section dividers) are properly skipped |
| `hex-formats.json` | Hex strings with spaces, mixed case, empty |
| `number-formats.json` | `$number` with various formats (hex, scientific, special values) |
| `empty-tests.json` | Valid file with empty tests array |
| `options.json` | All recognized options with valid values |

### structural-errors/

Contains test files that should cause the test runner to exit with a STRUCTURAL ERROR. Each file tests one specific error condition:

| File | Expected Error |
|------|----------------|
| `missing-type.json` | Missing `type` field |
| `wrong-type.json` | Unrecognized `type` value |
| `invalid-version.json` | Malformed semver (e.g., "1.0") |
| `missing-version.json` | Missing `version` field |
| `missing-tests.json` | Missing `tests` array |
| `invalid-test-name-start.json` | Name starting with digit |
| `invalid-test-name-chars.json` | Name with invalid characters (hyphen) |
| `duplicate-names.json` | Duplicate names (case-insensitive) |
| `unknown-test-type.json` | Unrecognized test type |
| `invalid-hex-odd.json` | Odd number of hex digits |
| `invalid-hex-chars.json` | Non-hex characters in hex string |
| `invalid-number-empty.json` | Empty `$number` string |
| `invalid-number-unparseable.json` | Unparseable `$number` value |
| `invalid-number-hex-no-digits.json` | `0x` without digits |
| `marker-extra-keys.json` | Marker object with extra keys |
| `option-wrong-type-string.json` | String instead of boolean for option |
| `option-wrong-type-bool.json` | Boolean instead of integer for option |
| `option-null.json` | Null option value |
| `option-negative-int.json` | Negative integer option |
| `options-not-object.json` | Options field is not an object |
| `missing-test-name.json` | Test without name |
| `missing-test-type.json` | Test without type |
| `mixed-comment-missing-name.json` | Entry with comment and non-comment keys but missing name |
| `missing-encode-input.json` | Encode test without input |
| `missing-encode-expected-bytes.json` | Encode test without expected_bytes |
| `missing-decode-input-bytes.json` | Decode test without input_bytes |
| `missing-decode-expected-value.json` | Decode test without expected_value |
| `missing-roundtrip-input.json` | Roundtrip test without input |
| `missing-encode-error-expected.json` | Encode error test without expected_error |
| `missing-decode-error-expected.json` | Decode error test without expected_error |
| `test-not-object.json` | Test element is not an object |
| `tests-not-array.json` | Tests field is not an array |
| `type-not-string.json` | Type field is not a string |
| `version-not-string.json` | Version field is not a string |

### skip-scenarios/

Contains test files where specific tests should be skipped with warnings (but processing should continue):

| File | Scenario |
|------|----------|
| `unrecognized-option.json` | Test with unknown option name |
| `unrecognized-error-type.json` | Test with unknown expected_error value |
| `typo-in-option.json` | Common typos in option names |

### config/

Contains config file tests and related test data:

| File | Tests |
|------|-------|
| `valid-config.json` | Basic valid configuration |
| `empty-sources.json` | Empty sources array (valid, zero tests) |
| `skip-source.json` | Source with `skip: true` |
| `directory-config.json` | Config that processes a directory |
| `recursive-config.json` | Config with `recursive: true` |
| `duplicate-paths.json` | Duplicate paths (should process once) |
| `comments-in-config.json` | Comment keys in config and sources |

#### config/errors/

Config files that should cause STRUCTURAL ERRORs:

| File | Expected Error |
|------|----------------|
| `missing-config-type.json` | Missing type field |
| `wrong-config-type.json` | Wrong type value (bonjson-test instead of bonjson-test-config) |
| `missing-sources.json` | Missing sources field |
| `sources-not-array.json` | Sources is not an array |
| `source-not-object.json` | Source element is not an object |
| `source-missing-path.json` | Source without path field |
| `source-empty-path.json` | Empty path string |
| `source-path-not-string.json` | Path is not a string |
| `source-recursive-not-bool.json` | Recursive is not a boolean |
| `source-skip-not-bool.json` | Skip is not a boolean |
| `source-nonexistent-path.json` | Path to nonexistent file |

#### config/directory-source/

Test directory for directory processing tests:
- `Ztest.json` - Uppercase filename (tests alphabetical ordering)
- `test-a.json`, `test-b.json` - Regular test files
- `subdir/` - Subdirectory (only processed with `recursive: true`)
- `.hidden/` - Hidden directory (should be silently ignored)
- `README.md`, `notes.txt` - Non-JSON files (should be skipped with log)

### special-values/

Contains tests for edge cases in value parsing and comparison:

| File | Tests |
|------|-------|
| `negative-zero.json` | IEEE 754 negative zero (-0.0 vs 0.0) |
| `nan-comparison.json` | NaN equals NaN for testing |
| `object-slash-key.json` | `//` keys in data (not comments) |
| `object-comparison.json` | Object equality (order-independent) |
| `number-equality.json` | Mathematical equality across types |
| `trailing-bytes.json` | Trailing bytes handling |
| `version-compatibility.json` | Standard version file |
| `version-prerelease.json` | Pre-release version |
| `version-build-metadata.json` | Version with build metadata |

## How to Use These Tests

### Manual Testing

1. Run your test runner against each file in `valid/` - all should pass
2. Run against each file in `structural-errors/` - each should cause an error exit
3. Run against files in `skip-scenarios/` - specific tests should be skipped with warnings
4. Run against config files in `config/` to test config processing

### Automated Testing

Implement a meta-test that:
1. Runs the test runner against each structural error file and verifies it exits with an error
2. Runs against valid files and verifies they process without errors
3. Runs against skip scenario files and verifies appropriate warnings are logged
4. Verifies directory processing order using the config tests

## Coverage Checklist

A complete test runner implementation should handle:

- [ ] Document type validation (`bonjson-test`, `bonjson-test-config`)
- [ ] Semver version validation and compatibility
- [ ] Comment key stripping (at correct levels only)
- [ ] Test name pattern validation
- [ ] Test name uniqueness (case-insensitive)
- [ ] All five test types
- [ ] All type-specific required fields
- [ ] Hex string parsing (case-insensitive, spaces allowed)
- [ ] `$number` parsing (NaN, Infinity, hex int, hex float, decimal, scientific)
- [ ] Negative zero preservation
- [ ] Option type validation
- [ ] Unrecognized option handling (skip with warning)
- [ ] Unrecognized error type handling (skip with warning)
- [ ] Config file processing
- [ ] Directory traversal (alphabetical, files before subdirs)
- [ ] Recursive directory processing
- [ ] Dotfile/directory ignoring
- [ ] Non-JSON file skipping
- [ ] Duplicate path deduplication
- [ ] Skip source handling
- [ ] Value comparison (NaN=NaN, -0.0≠0.0, object order independence)
