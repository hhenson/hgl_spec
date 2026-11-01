# Specification foundation

Date: 2026-11-01. Status: agreed ownership; semantic details remain proposed.

## Agreed

- **SPEC-1 — Scope.** Define HGL's syntax, types, phases, functions, composition,
  modules, errors, testing, and runtime concepts independently of a target.
- **SPEC-2 — Runtime.** Specify the observable temporal model here. The mandatory
  [core](https://github.com/hhenson/hgl_core) specifies the API and behavior an
  implementation exposes, including runtime facilities not implemented in HGL.
- **SPEC-3 — Resources.** Specifications contain rules, illustrative code/APIs,
  and diagrams. Explanations provide detailed justification and references only
  when needed. This repository contains no executable source files.
- **SPEC-4 — Tests.** Define the test language and eval meaning here. Core owns
  executable runtime conformance cases and their expected observations.
- **SPEC-5 — Independence.** A reader needs no particular implementation,
  historical repository, or trial record to understand a rule.
- **SPEC-6 — Writing.** Keep rules and discussions terse. Each rule states one
  obligation. Put substantial justification in an explanation.

## Proposed next scope

Define value and temporal phases, then use
[testing and eval](testing-and-eval.md) to connect source meaning to core's
runtime observations. Freeze a small scalar profile before adding collections.

## Open

- Exact grammar, canonical types, phase boundaries, and mandatory capabilities.
- Which runtime concepts require language forms versus named core operations.
- Version alignment between the specification, core API, and conformance pack.
- Location of executable language-only conformance and rejection fixtures.
