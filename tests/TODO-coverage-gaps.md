# Test Coverage Gaps

Identified during audit 2026-02-13. These are new tests to add, not fixes to existing tests.

## Empty test files
- **typed-arrays.json** — type codes f5-fe completely untested
- **records.json** — type codes b9-ba completely untested

## Missing error type tests
- `max_bignumber_exponent_exceeded` — no tests
- `max_bignumber_magnitude_exceeded` — no tests
- `value_out_of_range` — no tests

## Missing option tests
- `out_of_range: "stringify"` — no tests
- `unicode_normalization: "nfc"` — no tests

## Missing capability tests
- `signaling_nan` — no sNaN-specific tests

## Incomplete reserved type code coverage
- Reserved codes 0xf1-0xf4 not tested (0xbb, 0xbe, 0xbf, 0xc0, 0xc9, 0xe8, 0xe9, 0xf0 are tested)
