BONJSON: Binary Object Notation for JSON
========================================

BONJSON is a _lightning-fast_ and _efficient_ **1:1 compatible** binary drop-in replacement for [JSON](#json-standards).

BONJSON is **30 times** faster to process than [JSON](#json-standards).


Why use binary?
---------------

Text formats are great for humans to read, but they're slow and wasteful for computers. Computers should speak to humans in **text** (such as [JSON](#json-standards)), and to each other in **binary**.


Why use BONJSON?
----------------

### It's Small

* The [BONJSON specification](bonjson.md) is only 600 lines long (including the formal grammar).
* The [C reference implementation](https://github.com/kstenerud/ksbonjson/tree/main/library/src) is less than 500 LOC each for the encoder and decoder.

### It's Simple

* It doesn't use compression tricks like histograms or references or lookups. Leave those to the _real_ compression algorithms.
* It uses existing, CPU-native data encodings as much as possible.
* It keeps tricky sub-byte data to a minimum.

### It's Speedy

* Data encoding and decoding can be done using branchless algorithms ([example](https://github.com/kstenerud/ksbonjson/tree/main/library/src)).
* The most common data types and ranges are encoded in fewer bytes.
* The [C reference implementation](https://github.com/kstenerud/ksbonjson) is **34x** faster than [jq](https://github.com/jqlang/jq).

Benchmarking [the C Reference Implementation](https://github.com/kstenerud/ksbonjson) vs [jq](https://github.com/jqlang/jq) on a Core i3-1315U (using [this test data](https://github.com/kstenerud/test-data)):

**10MB:**

```
~/ksbonjson/benchmark$ ./benchmark.sh 10mb

Benchmarking BONJSON decode+encode with 9024k file 10mb.bjn

real    0m0.022s
user    0m0.012s
sys     0m0.012s

Benchmarking JSON decode+encode with 10256k file 10mb.json

real    0m0.347s
user    0m0.317s
sys     0m0.035s
```

**100MB:**

```
~/ksbonjson/benchmark$ ./benchmark.sh 100mb

Benchmarking BONJSON decode+encode with 90224k file 100mb.bjn

real    0m0.202s
user    0m0.095s
sys     0m0.131s

Benchmarking JSON decode+encode with 102556k file 100mb.json

real    0m3.245s
user    0m3.039s
sys     0m0.294s
```

**1000MB:**

```
~/ksbonjson/benchmark$ ./benchmark.sh 1000mb

Benchmarking BONJSON decode+encode with 902208k file 1000mb.bjn

real    0m1.849s
user    0m0.889s
sys     0m1.152s

Benchmarking JSON decode+encode with 1025564k file 1000mb.json

real    0m32.455s
user    0m30.295s
sys     0m2.693s
```


What about the other binary formats?
------------------------------------

**None** of them are 1:1 compatible. Most of them are overcomplicated.

| Encoding | Type Parity | Value Parity | Feature Parity | Endianness |
| -------- | ----------- | ------------ | -------------- | ---------- |
| BONJSON  |      ✔️      |      ✔️       |        ✔️       |   Little   |
| BSON     |      ❌     |      ❌      |        ❌      |   Little   |
| UBJSON   |      ✔️      |      ❌      |        ❌      |   Big      |
| BJData   |      ❌      |      ❌      |        ❌      |   Little   |
| PSON     |      ❌     |      ❌      |        ❌      |   Unclear  |
| Smile    |      ❌     |      ❌      |        ❌      |   Big      |

* **Type Parity**: No extra data types that aren't present in [JSON](#json-standards)
* **Value Parity**: Allows only the same value ranges as [JSON](#json-standards) (for example: infinities and NaN are disallowed)
* **Feature Parity**: Supports the same features as [JSON](#json-standards) (for example: progressive document construction)
* **Endianness**: Big endian formats are slower to process on modern hardware

**Wherever there's a compatibility mismatch, breakage will eventually occur** - it's only a matter of time before your complex data pipelines trigger it. Having confidence in your data plumbing is paramount.


-------------------------------------------------------------------------------

📚 Specifications and Code
--------------------------

### Specification

 * [📖 BONJSON Specification](bonjson.md)

### Formal Grammar

 * [🔡 Dogma grammar for BONJSON](bonjson.dogma)

### Implementations

 * [⚙️ C Reference Implementation](https://github.com/kstenerud/ksbonjson)

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
