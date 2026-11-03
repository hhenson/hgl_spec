# Base language specification

Status: consolidated reviewed baseline. Agreed rules and proposed extensions
are labelled within each chapter. Agreement on the outline does not accept
every extension. Executable conformance is planned in core.

The EBNF fragments are proposed consolidated transcriptions of the reviewed
forms. Semantic rules retain their stated status; grammar integration does not
settle an unresolved interpretation by implication.

1. [Source text](source-text.md)
2. [Units and names](units-and-names.md)
3. [Types and values](types-and-values.md)
4. [Expressions and bindings](expressions-and-bindings.md)
5. [Functions and phases](functions-and-phases.md)
6. [Generics and operators](generics-and-operators.md)
7. [Temporal model](temporal-model.md)
8. [Control flow and sources](control-flow-and-sources.md)
9. [Native boundaries](native-boundaries.md)
10. [Errors and diagnostics](errors-and-diagnostics.md)
11. [Tests and eval](tests-and-eval.md)
12. [Program execution](program-execution.md)
13. [Source documentation](source-documentation.md)

Apply the [authoring rules](../specification-rules.md). Grammar productions have
one definition across these chapters; references may point forward. Uppercase
names identify lexer tokens defined in the source-text chapter.

Scope: portable source meaning and runtime concepts. Core supplies mandatory
interfaces and runtime conformance. Target code, build recipes, and compiler
internals belong to implementations. Open decisions remain in repository issues.
