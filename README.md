BONJSON: Binary Object Notation for JSON
========================================

BONJSON is a **1:1 compatible** binary serialization format for [JSON](#json-standards).

It works in the same way and has the same capabilities and limitations as [JSON](#json-standards), except in binary rather than text.


### Why build this?

[JSON](#json-standards) isn't going away for a **loooooooong** time, so let's at least make it less wasteful where we can! BONJSON documents are quicker and more energy efficent to process, and are generally smaller and compress better than [JSON](#json-standards). Machine-to-machine transmission is better done in binary, with conversion to human-readable JSON only at endpoints where a human is actually involved (or via debugging tools along the path).

BONJSON is a drop-in replacement for [JSON](#json-standards), not an actually better format (if you want better, you might consider [Concise Encoding](https://concise-encoding.org/)). Structurally and logically, BONJSON works in exactly the same way as [JSON](#json-standards), suffering from all of the same problems (lack of types, weak specification, undefined edge cases, etc), **but** it can also benefit from the existing JSON ecosystem.

_The world has settled on JSON, so here we are._


### Why not BSON or anther existing binary JSON format?

Because they _all_ add extra features and gimmicks or special restrictions that make them **NOT** 1:1 compatible.

**Wherever there's a compatibility mismatch, breakage will eventually occur** - it's only a matter of time before your complex data pipelines trigger it. Having confidence in your data pipeline is paramount.

**1:1 compatible means that**:

 * Any valid document in the binary format **MUST** be 100% convertible to [JSON](#json-standards) format without data loss or schema requirement.
 * Any valid document in [JSON](#json-standards) format **MUST** be 100% convertible to the binary format without data loss or schema requirement.
 * Any valid document **MUST** be round-trip convertible between [JSON](#json-standards) and the binary format (in either direction) without data loss or schema requirement.
 * The binary format **MUST** support the same feature set as [JSON](#json-standards) does. For example, [JSON](#json-standards) supports progressive encoding.

A "binary" version of [JSON](#json-standards) **MUST** behave in exactly the same way as [JSON](#json-standards) (inasmuch as is possible with such a weakly specified format). The _only_ difference should be in the encoding mechanism.

_This_ is what BONJSON is.


-------------------------------------------------------------------------------

📚 Specifications and Code
--------------------------

### Specification

 * [📖 BONJSON Specification](bonjson.md)

### Formal Grammar

 * [🔡 Dogma grammar for BONJSON](bonjson.dogma)

### Implementations

 * [⚙️ Go Implementation](https://github.com/kstenerud/go-bonjson) (reference implementation)

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
