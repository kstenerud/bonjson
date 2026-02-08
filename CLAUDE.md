ABOUTME: BONJSON specification, formal grammar, conformance tests, and test specification.

## Structure

- `bonjson.md` — Main spec. Contains prose, type codes, encoding rules, security rules, and an embedded Dogma grammar (normative).
- `bonjson.dogma` — Standalone Dogma v1 formal grammar (must stay in sync with the embedded copy in bonjson.md).
- `bonjson-universal-test-specification.md` — Defines the cross-implementation test format (test types, error codes, value representation, runner behavior). Contains an Appendix B example that references wire bytes.
- `tests/conformance/` — Conformance test JSON files (bignumber, errors, specification-examples, etc.).
- `tests/test-runner-validation/` — Tests for the test runner itself (structural errors, skip scenarios, etc.).

## Key conventions

- BigNumber wire format: `0x8F + zigzag_leb128(exponent) + zigzag_leb128(signed_length) + LE magnitude bytes`.
- Record definition: `0xA0 + string* + 0x95`. Record instance: `0xA1 + LEB128(def_index) + value* + 0x95`.
- Decode tests with specific `input_bytes` must match the wire format exactly.
- Roundtrip tests are format-agnostic (no byte sequences).
- Comments in JSON test files use keys starting with `//`.
