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
- **SPEC-7 — Decisions.** Track open questions and pending decisions in issues,
  outside specification and explanation documents.

## Proposed next scope

Agree the [specification rules](specification-rules.md) and
[base language structure](language-structure.md) first. Extract source-text and
module rules, then use a scalar example to connect types, phases, temporal
meaning, and [testing and eval](testing-and-eval.md) to core's observations.
