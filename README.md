BONJSON: Binary Object Notation for JSON
========================================

BONJSON is a **1:1 compatible** binary serialization format for [JSON](#json-standards).

It's a drop-in replacement that works in the same way and has the same capabilities and limitations as [JSON](#json-standards) (no more, no less), except in binary rather than text.


### Why build this?

[JSON](#json-standards) isn't going away, so let's at least be more efficient where we can!

BONJSON documents are quicker and more energy efficent to process, and are generally smaller and compress better than [JSON](#json-standards).

Machine-to-machine transmission is better done in binary, with conversion to human-readable JSON only at endpoints where a human is actually involved.


### Why not BSON or anther existing binary JSON format?

Because they _all_ add extra features that make them **NOT** 1:1 compatible.

| Encoding | Type Parity | Value Parity | Feature Parity | Endianness |
| -------- | ----------- | ------------ | -------------- | ---------- |
| BONJSON  |      ✔️      |      ✔️       |        ✔️       |   Little   |
| BSON     |      ❌     |      ❌      |        ❌      |   Little   |
| UBJSON   |      ✔️      |      ❌      |        ❌      |   Big      |
| BJData   |      ✔️      |      ❌      |        ❌      |   Little   |
| PSON     |      ❌     |      ❌      |        ❌      |   Unclear  |
| Smile    |      ❌     |      ❌      |        ❌      |   Big      |

* **Type Parity**: The format has no extra data types that aren't present in JSON
* **Value Parity**: The format allows only the same value ranges as JSON (for example infinities and NaN)
* **Feature Parity**: The formats support the same features as JSON (for example progressive document construction)

**Wherever there's a compatibility mismatch, breakage will eventually occur** - it's only a matter of time before your complex data pipelines trigger it. Having confidence in your data pipeline is paramount.

**1:1 compatible means that**:

 * Any valid document **MUST** be round-trip convertible between [JSON](#json-standards) and the binary format (in either direction) without data loss or schema requirement.
 * The binary format **MUST** support the same feature set as [JSON](#json-standards) does. For example, [JSON](#json-standards) supports progressive encoding.

A binary version of [JSON](#json-standards) **MUST** behave in exactly the same way as [JSON](#json-standards). The _only_ difference should be in the encoding mechanism.

**_This_ is what BONJSON is.**


### Why use binary at all?

A simple binary format is orders of magnitude faster to produce and consume compared to a text format. It also offers _much_ smaller sizes for number-heavy data.

**The average progression is**:

 * **When starting something new:** JSON, because it's simple and ubiquitous.
 * **As your costs begin to rise:** BONJSON, because it's a drop-in replacement for JSON with lower processing and transmission costs.
 * **As your needs expand beyond basic data:** A more advanced format specific to your use case.


### BONJSON is Small, Simple, and Efficient

BONJSON's format is designed to be simple (the reference implementation is less than 1000 LOC), and compact where it makes sense to do so. This means:

* No built-in compression (dictionaries, lookups, etc). Leave that to professional compression algorithms.
* The most likely data uses the smallest encoding overhead.
* The encoding is designed to be computationally simple for 99.99% of real-world data.

-------------------------------------------------------------------------------

📚 Specifications and Code
--------------------------

### Specification

 * [📖 BONJSON Specification](bonjson.md)

### Formal Grammar

 * [🔡 Dogma grammar for BONJSON](bonjson.dogma)

### Implementations

 * [⚙️ C Implementation](https://github.com/kstenerud/ksbonjson) (reference implementation)

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
