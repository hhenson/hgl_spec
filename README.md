# HGL specification

The language specification and runtime concepts for HGL, a language for temporal
computation. This repository contains specifications and explanations only.

- [Foundation charter](spec/foundation.md)
- [Specification rules](spec/specification-rules.md)
- [Base language structure](spec/language-structure.md)
- [Language chapters](spec/language/README.md)
- [Testing and eval boundary](spec/testing-and-eval.md)
- [Mandatory core contract](https://github.com/hhenson/hgl_core)
- [Open decisions](https://github.com/hhenson/hgl_spec/issues/2)

Specifications contain rules, illustrative code/APIs, and useful diagrams.
Explanations are added under `explanations/` when a concept needs justification.
Executable HGL, native declarations, and conformance code live in the core or
library that owns them. Read this repository independently of any implementation.

Status: initial language baseline across thirteen chapters. Authoring conventions
and chapter order are agreed. Chapters preserve agreed rules and explicitly label
proposed extensions. Executable conformance remains planned in core.
