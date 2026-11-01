# Language specification rules

Status: agreed authoring conventions. Feature semantics retain their own status.

## Agreed foundation

Keep specifications and discussions terse. This repository contains documents
only. Specifications state rules with examples/APIs and useful diagrams;
explanations justify complex concepts when needed. State contracts independently
of an implementation. Label agreed decisions and proposals. Keep open questions
and decision checklists in the repository's issue tracker, outside specification
and explanation documents.

## Authority and status

- **WRITE-1.** Grammar defines syntax; semantic rules define meaning and
  admission. Neither implementation behavior nor an example adds a rule.
  Resolve a conflict between grammar, rules, and examples as a specification
  defect; do not choose an implicit precedence between conflicting statements.
- **WRITE-2.** Each document states its status. Mixed documents label each
  agreed/proposed section. Track open questions and pending decisions in issues;
  they specify no default behavior. Promotion to agreed requires an explicit
  design decision.
- **WRITE-3.** Give each semantic obligation a stable, unique rule ID. Preserve
  IDs across moves. State the subject, condition, required observation, and
  failure where relevant. Use `must`/`must not` for obligations and `may` for an
  explicit permitted choice; put recommendations in explanations.

## EBNF

- **WRITE-4.** Use one project-defined EBNF notation in fenced `ebnf` blocks.
  Grammar punctuation is metanotation; it does not introduce HGL punctuation.

| Notation | Meaning |
| --- | --- |
| `name = expression ;` | Production definition |
| `"text"` | Exact terminal; `\"` and `\\` escape quote and backslash in this notation |
| `a, b` | Concatenation |
| `[ a ]` | Optional: zero or one occurrence |
| `{ a }` | Repetition: zero or more occurrences |
| `( a )` | Grouping |
| `(* text *)` | Grammar comment; not an HGL comment |
| `? description ?` | Lexical special sequence, defined precisely in adjacent prose |

Alternatives use `a | b`.

Grouping, option, and repetition bind before concatenation; concatenation binds
before alternatives. A quoted comma or semicolon is an HGL terminal; an unquoted
one belongs to EBNF. This notation makes no claim of compliance with an external
grammar standard.

- **WRITE-5.** Use `lower_snake_case` for productions and `UPPER_SNAKE_CASE` for
  lexer tokens. Declare every referenced symbol, or identify its owning section.
  Give each production one authoritative definition. Reference it elsewhere.
- **WRITE-6.** Separate lexical grammar from token grammar. Lexical grammar
  defines source characters and token boundaries. Token grammar consumes tokens;
  whitespace, comments, significant newlines, and end-of-file follow explicit
  lexical rules. Never assume that whitespace or newlines are always skipped.
- **WRITE-7.** Define expression precedence and associativity explicitly. Resolve
  ambiguities such as generic brackets versus comparisons and newline-sensitive
  forms in rules. Keep type, phase, scope, and capability constraints out of
  syntax productions unless they actually change the admitted syntax.

The [module grammar](language/units-and-names.md) demonstrates these conventions;
its lexer tokens are defined in [source text](language/source-text.md).

## Section form

- **WRITE-8.** Use this order within a feature section. Omit an item when it adds
  nothing. Track unresolved obligations in issues rather than filling them by
  guesswork or appending questions to the document.

| Item | Content |
| --- | --- |
| Scope and status | What is defined and which decisions are agreed/proposed |
| Grammar or API | Syntax fragment or a link to the owning core interface |
| Rules | Naming, types, phase, evaluation order, effects, and failures as applicable |
| Examples | Small cases with assumptions and expected observations |
| Core boundary | Required core capability or the absence of a runtime effect |

- **WRITE-9.** Keep one normative home for each rule. HGL source rules belong
  here; mandatory API behavior belongs in core. Link across that boundary.
  Diagrams illustrate the written contract and introduce no extra obligations.
- **WRITE-10.** Mark examples as accepted illustrations, proposed illustrations,
  or deliberate rejections. Supply their context: complete module or explicitly
  bounded fragment, required declarations, phase, and relevant input/output.
  A rejection identifies its rule and failure phase; require a diagnostic code
  only once that code is agreed. Do not require incidental message wording.
- **WRITE-11.** Give each behavioral rule a discriminating example or expected
  observation. Include silence, invalidity, lifetime, or failure cases when they
  distinguish meanings. Link to executable conformance when it exists; otherwise
  mark it planned. Never report a document example as an executed test.
- **WRITE-12.** Put rationale and extended derivations in explanations. Keep
  target symbols, compiler passes, build commands, implementation coverage, and
  historical discussions outside normative language chapters.
