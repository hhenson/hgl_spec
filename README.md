# HGL specification

The language specification and runtime concepts for HGL, a language for temporal
computation. This repository contains specifications and explanations only.

- [Foundation charter](spec/foundation.md)
- [Proposed specification rules](spec/specification-rules.md)
- [Proposed base language structure](spec/language-structure.md)
- [Testing and eval boundary](spec/testing-and-eval.md)
- [Mandatory core contract](https://github.com/hhenson/hgl_core)
- [Open decisions](https://github.com/hhenson/hgl_spec/issues/2)

Specifications contain rules, illustrative code/APIs, and useful diagrams.
Explanations are added under `explanations/` when a concept needs justification.
Executable HGL, native declarations, and conformance code live in the core or
library that owns them. Read this repository independently of any implementation.

Status: foundation. Agree the specification conventions and chapter order first;
then extract grammar and semantic rules through practical examples. Proposed
behavior is not yet an accepted language contract.
