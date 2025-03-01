BONJSON: Binary Object Notation for JSON
========================================

BONJSON is a **1:1 compatible** binary serialization format for [JSON](#json-standards).

It's a drop-in replacement that works in the same way and has the same capabilities and limitations as [JSON](#json-standards) (no more, no less), in a compact and simple-to-process binary format rather than text.



Why make a binary JSON?
-----------------------

[JSON](#json-standards) isn't going anywhere, so let's at least be more efficient where we can!

BONJSON documents are quicker and more energy efficent to process, and are generally smaller and compress better than JSON.

Machine-to-machine transmission is better done in binary, with conversion to human-readable JSON only where a human actually needs to be involved.


### Why not BSON or anther existing binary JSON format?

Because they _all_ add extra things that make them **NOT** 1:1 compatible.

| Encoding | Type Parity | Value Parity | Feature Parity | Endianness |
| -------- | ----------- | ------------ | -------------- | ---------- |
| BONJSON  |      ✔️      |      ✔️       |        ✔️       |   Little   |
| BSON     |      ❌     |      ❌      |        ❌      |   Little   |
| UBJSON   |      ✔️      |      ❌      |        ❌      |   Big      |
| BJData   |      ✔️      |      ❌      |        ❌      |   Little   |
| PSON     |      ❌     |      ❌      |        ❌      |   Unclear  |
| Smile    |      ❌     |      ❌      |        ❌      |   Big      |

* **Type Parity**: Meaning the format has no extra data types that aren't present in JSON
* **Value Parity**: Meaning the format allows only the same value ranges as JSON (for example: infinities and NaN are disallowed)
* **Feature Parity**: Meaning the format supports the same features as JSON (for example: progressive document construction)

**Wherever there's a compatibility mismatch, breakage will eventually occur** - it's only a matter of time before your complex data pipelines trigger it. Having confidence in your data plumbing is paramount.

**1:1 compatible means that**:

 * Any valid document **MUST** be round-trip convertible between [JSON](#json-standards) and the binary format (in either direction) without data loss or schema requirement.
 * The binary format **MUST** support the same feature set as [JSON](#json-standards) does. For example, JSON supports progressive encoding.

A binary version of JSON **MUST** behave in exactly the same way as [JSON](#json-standards). The _only_ difference should be in the encoding mechanism.

**_This_ is what BONJSON is.**


### Why use binary at all?

A simple binary format is orders of magnitude faster to produce and consume compared to a text format, and has a smaller data footprint.

**Most systems follow a similar progression of data needs**:

| Stage                               | Data Choice                                                        |
| ----------------------------------- | ------------------------------------------------------------------ |
| New project                         | JSON, because it's simple and ubiquitous and piss simple to debug. |
| Your costs begin to rise            | JSON-compatible binary format, because it's (hopefully) a drop-in replacement for JSON, with lower processing and transmission costs (such as BONJSON). |
| Your needs expand beyond basic data | A more advanced binary format specific to your use case (such as Protobufs). |



BONJSON is Small, Simple, and Efficient
---------------------------------------

**The BONJSON format is designed for simplicity and efficiency:**

 * No compression tricks. Leave that to the _real_ compression algorithms.
 * Use existing, CPU-native data encodings as much as possible.
 * Encode the most common data using the smallest encoding overhead.
 * Be computationally simple.

**As a result:**

 * [The BONJSON specification](bonjson.md) is only 600 lines long.
 * [The C reference implementation](https://github.com/kstenerud/ksbonjson) is less than 500 LOC each for the encoder and decoder)

**It's 10x faster:**

Benchmarking [the C Reference Implementation](https://github.com/kstenerud/ksbonjson) vs [jq](https://github.com/jqlang/jq) on [large-file](https://github.com/json-iterator/test-data/blob/master/large-file.json)

    ~/ksbonjson/benchmark$ ./benchmark.sh large-file

    Benchmarking BONJSON decode+encode with 23364k file large-file.bjn

    real    0m0.086s
    user    0m0.060s
    sys     0m0.034s

    Benchmarking JSON decode+encode with 25532k file large-file.json

    real    0m0.747s
    user    0m0.689s
    sys     0m0.073s

-------------------------------------------------------------------------------

📚 Specifications and Code
--------------------------

### Specification

 * [📖 BONJSON Specification](bonjson.md)

### Formal Grammar

 * [🔡 Dogma grammar for BONJSON](bonjson.dogma)

### Implementations

 * [⚙️ C Reference Implementation](https://github.com/kstenerud/ksbonjson) (reference implementation)

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
