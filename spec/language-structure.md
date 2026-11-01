# Base language specification structure

Status: proposed reading and drafting order. Entries inventory work; they do
not accept a language feature or its spelling. Filenames are planned, not links
to completed chapters.

## Proposed chapters

| Order | Chapter / planned file | What must be specified |
| --- | --- | --- |
| 0 | Specification rules / `specification-rules.md` | Authority, status, rule IDs, EBNF, examples, and explanations |
| 1 | Source text / `language/source-text.md` | Encoding, identifiers, keywords, literals, comments, punctuation, token boundaries, whitespace, newlines, and EOF |
| 2 | Units and names / `language/units-and-names.md` | Source/module/part structure, declarations, names, scopes, imports, exports, forward references, duplicates, and shadowing |
| 3 | Types and values / `language/types-and-values.md` | Identity, scalar domains, aggregates, temporal shapes, absence, references, atomic boundaries, delta types, ownership, and conversions |
| 4 | Expressions and bindings / `language/expressions-and-bindings.md` | Precedence, associativity, evaluation order, literals, calls, projection, bindings, assignment, mutability, and contextual typing |
| 5 | Functions and phases / `language/functions-and-phases.md` | Value/composition/evaluation phases, function forms, classification, parameters/results, return, captures, effects, and lifting |
| 6 | Generics and operators / `language/generics-and-operators.md` | Type/value parameters, constraints, inference, nominal contracts, token binding, candidates, specificity, ambiguity, and properties |
| 7 | Temporal model / `language/temporal-model.md` | Time, publications, validity, deltas, activity, admission, lifecycle, state/cache, output, and capability phases |
| 8 | Control flow and sources / `language/control-flow-and-sources.md` | Conditionals, iteration, branching, early exits, generators, suspension, captures, and child lifetime in each phase |
| 9 | Native boundaries / `language/native-boundaries.md` | Native types and declarations, capability contracts, dependency discovery, and core realization boundaries |
| 10 | Errors and diagnostics / `language/errors-and-diagnostics.md` | Source rejection, execution failure, result data, propagation/capture, cleanup, diagnostic categories, and stable codes |
| 11 | Tests and eval / `language/tests-and-eval.md` | Test scopes, visibility, assertions, targets, input/expected traces, comparison, run isolation, and test outcomes |
| 12 | Program execution / `language/program-execution.md` | Entry selection, constant configuration, observable run bounds, termination, and results |
| 13 | Source documentation / `language/source-documentation.md` | Comment attachment, declaration association, documentation meaning, and non-semantic presentation |

The table is a dependency-aware reading order, not a compiler pipeline. A
chapter may reference a later definition. Forward references are explicit;
they do not require repeating the rule. Types describe available forms before
later chapters define their temporal behavior.

## Runtime and core boundary

The temporal-model chapter defines HGL's observable execution concepts. Core
owns the API that realizes them and the required runtime observations. Native
and test-host facilities with no HGL source form remain core contracts.

CLI spellings, target code generation, host packaging, and internal compiler
representations are implementation documentation.

## Proposed work order

1. Agree specification conventions and this outline.
2. Draft source text, then units and names. Extract each candidate rule and
   spelling explicitly; record unresolved cases in the issue tracker.
3. Draft the scalar subset of types/values, expressions/bindings, and
   functions/phases. Use one small example to expose their dependencies.
4. Specify enough of the temporal model, diagnostics, and test language to
   describe a scalar runtime function and its eval case completely.
5. Define the required core API and shared observations for that slice.
6. Expand the remaining chapters by practical examples. Accept rules
   individually; partial coverage never implies that omitted features are absent.

This stages the review. It does not freeze a smaller language edition or make
the remaining inventory optional.

## First extraction checklist

For each example: list its tokens and productions; resolve its declarations
and names; assign types and phases; state evaluation and temporal observations;
identify errors and required core capabilities. If an observation has no rule,
record it in the issue tracker before proposing implementation code.

For each chapter: check complete syntax, static meaning, dynamic meaning,
boundary/failure cases, and examples. A grammar fragment alone is incomplete.
