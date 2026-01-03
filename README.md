BONJSON: Binary Object Notation for JSON
========================================

BONJSON is a _hardened_, _lightning-fast_ and _efficient_, **1:1 compatible** binary drop-in replacement for [JSON](#json-standards).

It's **35 times** faster to process than [JSON](#json-standards).

It's also **far safer** than JSON:

* **Safe** against key collision attacks
* **Safe** against injection attacks
* **Safe** against truncation attacks
* **Safe** against numeric range attacks


Why use binary?
---------------

### Efficiency

Text formats are great for humans to read, but they're slow and wasteful for computers.

**35 times** slower than they could be in this case!

Computers should speak to humans in **text** (such as [JSON](#json-standards)), and to each other in **binary**.



Why use BONJSON?
----------------

### It's Small

* The [BONJSON specification](bonjson.md) is only 800 lines long (including the formal grammar).
* The [C reference implementation](https://github.com/kstenerud/ksbonjson/tree/main/library/src) is less than 500 LOC each for the encoder and decoder.

### It's Simple

* It doesn't use tricks like histograms or references or lookups. Leave those to _real_ compression algorithms.
* It uses existing, CPU-native data encodings as much as possible.
* It keeps tricky sub-byte data to a minimum.

### It's Safe

* **Safe** against key collision attacks
* **Safe** against injection attacks
* **Safe** against truncation attacks
* **Safe** against numeric range attacks

### It's Speedy

* Data encoding and decoding can be done using branchless algorithms ([example](https://github.com/kstenerud/ksbonjson/tree/main/library/src)).
* The most common data types and ranges are encoded in fewer bytes.
* The [C reference implementation](https://github.com/kstenerud/ksbonjson) is **35 times** faster than [jq](https://github.com/jqlang/jq).

Benchmarking [the C Reference Implementation](https://github.com/kstenerud/ksbonjson) vs [jq](https://github.com/jqlang/jq) on a Core i3-1315U (using [this test data](https://github.com/kstenerud/test-data)):

**10MB:**

```
~/ksbonjson/benchmark$ ./benchmark.sh 10mb

Benchmarking BONJSON (decode+encode) with 9052k file 10mb.boj

real    0m0.021s
user    0m0.009s
sys     0m0.016s

Benchmarking BONJSON (decode only) with 9052k file 10mb.boj

real    0m0.011s
user    0m0.003s
sys     0m0.012s

Benchmarking JSON (decode+encode) with 10256k file 10mb.json

real    0m0.340s
user    0m0.322s
sys     0m0.024s
```

BONJSON processed **35.8x** faster.

**100MB:**

```
~/ksbonjson/benchmark$ ./benchmark.sh 100mb

Benchmarking BONJSON (decode+encode) with 90508k file 100mb.boj

real    0m0.183s
user    0m0.085s
sys     0m0.121s

Benchmarking BONJSON (decode only) with 90508k file 100mb.boj

real    0m0.080s
user    0m0.030s
sys     0m0.071s

Benchmarking JSON (decode+encode) with 102560k file 100mb.json

real    0m3.227s
user    0m3.039s
sys     0m0.232s
```

BONJSON processed **35.8x** faster.

**1000MB:**

```
~/ksbonjson/benchmark$ ./benchmark.sh 1000mb

Benchmarking BONJSON (decode+encode) with 905076k file 1000mb.boj

real    0m1.762s
user    0m0.846s
sys     0m1.124s

Benchmarking BONJSON (decode only) with 905076k file 1000mb.boj

real    0m0.746s
user    0m0.300s
sys     0m0.645s

Benchmarking JSON (decode+encode) with 1025564k file 1000mb.json

real    0m31.522s
user    0m29.737s
sys     0m2.307s
```

BONJSON processed **35.2x** faster.



What about the other binary JSON-like formats?
----------------------------------------------

**None** of them are 1:1 compatible. None of them are safe. Most of them are overcomplicated.

| Encoding | Type Parity | Value Parity | Feature Parity | Safety | Endianness |
| -------- | ----------- | ------------ | -------------- | ------ | ---------- |
| BONJSON  |      ✔️     |      ✔️      |        ✔️       |   ✔️   |   Little   |
| BSON     |      ❌     |      ❌      |        ❌       |   ❌   |   Little   |
| CBOR     |      ❌     |      ❌      |        ❌       |   ❌   |   Big      |
| UBJSON   |      ✔️     |      ❌      |        ❌       |   ❌   |   Big      |
| BJData   |      ❌     |      ❌      |        ❌       |   ❌   |   Little   |
| PSON     |      ❌     |      ❌      |        ❌       |   ❌   |   Little   |
| Msgpack  |      ❌     |      ❌      |        ❌       |   ❌   |   Big      |
| Smile    |      ❌     |      ❌      |        ❌       |   ❌   |   Big      |

* **Type Parity**: No extra data types that aren't present in [JSON](#json-standards)
* **Value Parity**: Allows only the same value ranges as [JSON](#json-standards) (for example: infinities and NaN are disallowed)
* **Feature Parity**: Supports the same features as [JSON](#json-standards) (for example: progressive document construction)
* **Safety**: Guards against common attack vectors (key collisions, truncation, range)
* **Endianness**: Big endian formats are slower to process on modern hardware

**Wherever there's a compatibility mismatch, breakage will eventually occur** - it's only a matter of time before your complex data pipelines trigger it.

Having confidence in your data plumbing is paramount.


-------------------------------------------------------------------------------

📚 Specifications and Code
--------------------------

### Specification

 * [📖 BONJSON Specification](bonjson.md)

### Formal Grammar

 * [🔡 Dogma grammar for BONJSON](bonjson.dogma)

### Implementations

 * [⚙️ C Reference Implementation](https://github.com/kstenerud/ksbonjson)
 * [⚙️ Go Reference Implementation](https://github.com/kstenerud/go-bonjson)

-------------------------------------------------------------------------------


JSON Standards
--------------

Any discussions about JSON are done within the context of the ECMA and RFC specifications for JSON, and the json.org website:

 * [ECMA-404](https://ecma-international.org/publications-and-standards/standards/ecma-404/)
 * [RFC 8259](https://www.rfc-editor.org/info/rfc8259)
 * [json.org](https://www.json.org)



License
-------

Copyright (c) 2024 Karl Stenerud. All rights reserved.

Distributed under the [Creative Commons Attribution License](https://creativecommons.org/licenses/by/4.0/legalcode) ([license deed](https://creativecommons.org/licenses/by/4.0).
