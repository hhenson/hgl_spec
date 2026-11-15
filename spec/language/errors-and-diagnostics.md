# Errors and diagnostics

Status: agreed result, lookup, capture, and structured-error contracts. Codes
for proposed features apply only within those proposed profiles.

## Rules

- **ERR-1.** Distinguish source rejection, execution failure, ordinary result
  data, and test/infrastructure failure. Messages add context; a stable code
  identifies a catalogued condition. A warning does not reject valid source.
- **ERR-2.** `result<T,E>` owns exactly one tagged payload. `ok(value)` and
  `err(value)` require a complete expected result type. Equality requires both
  payload types' equality and compares tag then selected payload. This adds
  no ordering/hash. Temporal outcomes publish atomically, including repeats.
- **ERR-3.** An immutable result binding gains its tag fact from a direct
  `is_ok`/`is_err` guard; `.value`/`.error` requires the matching fact. Track
  negation, short-circuit paths, terminating branches, and merges. A saved bool,
  alias, fresh expression, or mutable var supplies no retained fact. Tag facts
  grant no temporal validity or modification fact.
- **ERR-4.** `error(message: text, code: optional_code)` owns a required string
  message and optional string code. Equality excludes provenance. On first
  graph failure, attach original node identity, phase, and evaluation time;
  propagation/capture preserve them. A node's display name is not a stable
  code. Nullable code/context observations require presence checking.
- **ERR-5.** Postfix `?` evaluates once. Success yields T. In a value function
  returning `result<U,E>`, failure returns the same E as Err. In when, only
  `result<T,error>` may propagate to node failure. No success output is written;
  completed effects stand. Composition, lifecycle, and generator uses reject.
  Returning Err as result data does not fail the node.
- **ERR-6.** Ordinary/atomic dictionary brackets and `at` are strict point-in-
  time reads; absence fails. `get` with no fallback or null fallback yields
  contextual absence. A non-null fallback has exact V type and is evaluated
  even for a present key. A present zero/false/empty value is not absent.
  Guard an inferred immutable nullable local against null before payload use.
  Explicit nullable annotations follow TYPE-14; this lookup contract adds no
  null-return suppression.
- **ERR-7.** In composition, `capture(callable, args...)` creates a temporal
  error boundary. `.value` retains the normal output type; outputless calls
  expose only `.error`. Capture of wiring-time scalar work rejects. Each
  evaluation failure ticks its canonical error; before failure the stream is
  invalid, and later success leaves its retained error valid but unmodified.
- **ERR-8.** Capture ends the failed pass without rollback or immediate retry.
  The child remains live for future ordinary work; unrelated outer/sibling
  work continues. The nearest boundary consumes the event once. Start/stop
  failures escape, including lifecycle work triggered during evaluation.
  Child errors follow child membership/lifetime. Earlier writes remain visible.
- **ERR-9.** Required constant failures reject where specified. Optional
  folding of executed operations must preserve their execution-failure phase.
  Failure during argument evaluation precedes later admission checks. Do not
  demand cascaded diagnostics after an earlier error prevents checking.
- **ERR-11.** Numeric policies prefer representable ordinary outcomes and
  explicit HGL errors over exception mechanisms. NaN, infinity, and an unordered
  comparison outcome are ordinary data, not execution errors or silence. Core
  specifies each policy's error outcome; the policy must not silently convert
  ordinary data into an exception or an error into a successful empty output.

## Diagnostic catalog

Source categories: `parse`, `name`, `type`, `shape`, `constraint`,
`function-kind`, `phase`, `injectable`, `operator`, and `module`. `build` is
infrastructure and cannot satisfy a source-rejection expectation.

| Category | Code | Condition / primary location |
| --- | --- | --- |
| parse | `syntax.expected_token` | Required token missing; unexpected token, or the end of the file for EOF |
| parse | `syntax.invalid_encoding` | Source is not well-formed UTF-8 (LEX-16); first offending byte |
| parse | `syntax.unexpected_character` | Character begins no token (LEX-16); character |
| parse | `string.unclosed` | String reaches a line break or EOF, or a text block reaches EOF (LEX-2, LEX-25); literal |
| parse | `string.unknown_escape` | Escape outside the admitted set, a `\` before a text block's line break included (LEX-2, LEX-25); escape |
| parse | `string.invalid_escape` | `\u{...}` names no Unicode scalar value (LEX-24); escape |
| parse | `string.malformed_text_block` | Text block breaks LEX-25's layout; offending delimiter or line |
| parse | `comment.unclosed` | Block or documentation comment reaches EOF (LEX-2); comment |
| parse | `number.malformed` | Numeric or duration run is not one literal (LEX-8, LEX-18, LEX-22); run |
| parse | `number.integer_range` | Integer literal outside its required domain (LEX-7, LEX-23); literal |
| parse | `number.float_range` | Float literal rounds to infinity (LEX-7, LEX-20); literal |
| parse | `duration.invalid` | Duration violates LEX-13; literal |
| parse | `date_time.malformed` | `@` run matches no temporal form (LEX-18); run |
| parse | `date_time.invalid_field` | Date, time, or offset field violates LEX-11 or LEX-19; literal |
| parse | `date_time.invalid_zone` | Zone name violates LEX-12's source form, an empty component included; literal |
| parse | `doc.unattached` | Documentation comment documents no declaration (SDOC-1); comment |
| parse | `doc.repeated` | Documentation comment before the one a declaration has (SDOC-1); earlier comment |
| module | `module.duplicate_unnamed` | Module has a second unnamed file (UNIT-1); module path of every unnamed file but one |
| module | `module.duplicate_part` | Two files of a module share a part name (UNIT-1); part name in every such file but one |
| module | `import.misplaced` | Import after a declaration of its file (UNIT-2); `use` |
| module | `import.unknown_module` | No source declares the module (UNIT-2); module path |
| name | `import.unknown_name` | Module declares no such name (UNIT-2); imported name |
| module | `import.private` | Imported declaration is not public (UNIT-2, UNIT-3); imported name |
| module | `import.cycle` | Modules import one another (UNIT-3); first import of the cycle |
| name | `name.duplicate` | Binding takes a name already in scope (UNIT-5, UNIT-12, TST-2, NATIVE-2); later name |
| name | `name.used_before_declaration` | Local used before the statement that declares it (UNIT-6); use |
| name | `name.test_only` | Test-only declaration used outside the tests (TST-2); use |
| name | `name.captured` | Anonymous function uses an enclosing `var`, `state`, `cache`, or injected capability (FN-14); use |
| name | `default.formal_argument` | Default names a parameter of its declaration or a field of its struct (UNIT-7); use |
| name | `alias.unqualified` | Import alias used without `::name` (UNIT-2); alias |
| name | `alias.unknown` | Name before `::` is no import alias of its file (UNIT-2); name |
| name | `test.duplicate_name` | Two tests of a module share a name (TST-2); later name |
| module | `test.export` | Exported declaration in a test context (TST-1); `export` |
| operator | `test.implementation` | Operator candidate in a test context (TST-1); operator name |
| operator | `impl.unknown_operator` | Operator candidate names no operator (GEN-4); operator name |
| name | `native.missing_declaration` | Native implementation completes no declaration (NATIVE-2); function name |
| phase | `native.const_hook` | Native const implementation has a lifecycle hook (NATIVE-3); hook |
| phase | `native.hook_without_when` | Start or stop hook with no `when` hook (NATIVE-3); hook |
| phase | `native.duplicate_hook` | Second start or stop hook (NATIVE-3); later hook |
| injectable | `inject.unknown_capability` | `inject` names no capability (TIME-8); name |
| type | `generic.invalid_application` | Generic type's name before a `<` that opens no arguments (GEN-14); name |
| type | `pack.default` | Pack parameter has a default (FN-8); `=` |
| type | `pack.const` | Pack parameter is const (FN-8); `const` |
| type | `pack.order` | Fixed parameter follows a pack (FN-8); parameter name |
| type | `pack.undeclared` | `...{Name}` names no type pack of its declaration (FN-8); name in the braces |
| type | `pack.cardinality` | Cardinality's upper bound is below its lower (GEN-10); upper bound |
| type | `type.not_a_type` | Name in a type position is no type: a function, operator, local, generic value parameter, or a type pack where one type is needed (UNIT-10, GEN-2, GEN-13); name |
| type | `type.not_ordinary` | `rolling`, `ref`, `atomic` or `signal` where an ordinary value type is required, or under a `?` that is not on the whole type of an input or a field (TYPE-12, TYPE-14, FN-1); offending type |
| type | `signal.position` | `signal` other than as the whole type of a function's temporal parameter (TYPE-12); `signal` |
| type | `nullable.key` | Set element or dictionary key has a `?` (TYPE-14); the nullable type |
| type | `generic.argument_count` | Generic type given the wrong number of arguments, or arguments given to a type or generic parameter that takes none (GEN-2, GEN-10, GEN-13); name |
| type | `list.size` | List size is not a nonnegative count (TYPE-10); size |
| type | `generic.kind_conflict` | Generic parameter used as a type and as a value (GEN-13); later use |
| type | `generic.kind_unresolved` | No field or constraint says what a struct's generic parameter is (GEN-13); parameter name |
| type | `generic.argument_kind` | Generic argument is a value where the parameter is a type, or a type where it is a value (GEN-14); argument |
| type | `struct.parent` | Parent is not an abstract struct (TYPE-3); parent |
| type | `struct.duplicate_parent` | Struct lists one parent twice (TYPE-3); later parent |
| type | `struct.inheritance_cycle` | Structs inherit from one another (TYPE-6); a parent of the first struct of the cycle |
| type | `field.conflict` | Field's type or optionality differs between two parents, or between a struct and a parent (TYPE-4); field, or the later parent |
| type | `field.not_inherited` | `name = value` names no field of a parent (TYPE-4); name |
| type | `field.duplicate` | Two members of a struct share a name (TYPE-4); later name |
| type | `field.recursive` | Field that makes its struct hold itself is not optional, or holds it through collection elements outside an `atomic<...>`, or gives a struct of its cycle a type built from a generic parameter, or a descendant's default removes its null (TYPE-6); field name, or the parent whose argument makes the field hold the struct |
| type | `field.null_default` | Null default on a field that is not optional (TYPE-4); default |
| type | `enum.value` | Enum number is not an integer (TYPE-7); number |
| type | `enum.duplicate_value` | Two members of an enum share a number (TYPE-7); later member |
| type | `enum.overflow` | Unnumbered member follows the largest `i64` (TYPE-7); member |
| type | `enum.duplicate_member` | Two members of an enum share a name (TYPE-7); later name |
| type | `type.mismatch` | Value's type is not the type its place expects: an annotation, a `var`, a result, an operand, an element, an argument (EXPR-5); `null` where the type has no `?` (TYPE-14); a `return` that does not fit its function (FN-7); a list of another length than the size expected (TYPE-10); value |
| type | `null.context_type` | `null` where nothing gives it a type (TYPE-14); the literal |
| type | `sequence.context_type` | Sequence whose entries give no type, where none is expected (EXPR-9); the sequence |
| type | `enum.context_type` | `.member` where no enum type is expected (TYPE-7); the member |
| name | `binding.unassigned` | `var` read on a path that gives it no value (EXPR-3); the read |
| name | `binding.immutable` | Assignment to an immutable binding: a `let`, a parameter, an iteration binding (EXPR-3); target |
| type | `argument.order` | Argument with no name after a named one (EXPR-2); the argument |
| type | `argument.unexpected` | Positional argument that no parameter is left to take (EXPR-2), or one given to a constructor (TYPE-5); the argument |
| name | `argument.unknown` | Named argument names no parameter or field (EXPR-2); name |
| name | `argument.duplicate` | Parameter or field given twice (EXPR-2); later name |
| type | `argument.missing` | Parameter or field that may not be left out is given nothing (EXPR-2, TYPE-5); callee name |
| function-kind | `call.role` | Temporal function called from a value function (FN-3); callee name |
| type | `struct.abstract` | Abstract struct constructed (TYPE-3); name |
| name | `member.unknown` | Struct has no field, or enum no member, of the name (EXPR-7, TYPE-7); name |
| operator | `operator.operand` | Operand types the operator's contract does not admit; operator |
| type | `default.order` | Parameter that may not be omitted follows one that may (UNIT-8); the later parameter |
| type | `tuple.index` | Tuple's index is no constant, or is outside the tuple (EXPR-7); index |
| type | `rolling.size_kind` | Wrong/mixed size kind; offending size, the minimum when the two differ in kind |
| type | `rolling.size_bounds` | Invalid bounds; offending size/minimum |
| type | `yield.time_type` | Wrong time type; yield time operand |
| type | `test.raises_code` | Not a literal catalogued execution code; argument |
| type | `nullable.required_presence` | Unguarded nullable payload; payload use |
| type | `lookup.default_type` | Non-null fallback is not exact V; fallback |
| type | `result.context_type` | Missing complete outcome context; constructor |
| type | `result.required_variant` | Unguarded tag payload; property |
| type | `result.propagation_type` | Incompatible propagated result/error; `?` |
| phase | `result.propagation_phase` | Forbidden propagation context; `?` |
| phase | `capture.phase` | Non-temporal capture/context; call |
| type | `capture.output_field` | Value requested from outputless capture; property |
| type | `value.constant_capability` | Required constant lacks contained capability; operation |
| shape | `delta.unsupported_shape` | No admitted delta type; shape/application |
| type | `delta.type_mismatch` | Incompatible originating delta/destination; value |
| name | `delta.argument_name` | Unknown delta field/argument; name |
| name | `delta.duplicate_argument` | Repeated delta argument; later name |
| type | `delta.entry_constant` | Entry is not admitted constant/cold data; entry |
| type | `delta.entry_type` | Wrong entry key/index/member type; entry |
| type | `delta.duplicate_entry` | Known equal entries; later entry |
| type | `delta.index_bounds` | Negative/out-of-fixed-range index; index |
| type | `delta.overlap` | Known overlapping addition/removal; later entry |
| phase | `test.statement_phase` | Node/lifecycle/return statement in test; keyword |

| Execution code | Required condition |
| --- | --- |
| `value.missing_key` | Strict ordinary dictionary lookup has no member |
| `value.unset_read` | Executed operation requires an unset ordinary payload |
| `value.capability` | Boxed executed operation lacks a required contained capability |
| `value.byte_range` | Bytes construction receives an octet outside 0–255 |
| `numeric.division_by_zero` | Explicit `?` propagation in `when` turns a checked-division Err into node failure; see the [core numeric contract](https://github.com/hhenson/hgl_core/blob/main/spec/numeric-operators.md) |
| `yield.negative_duration` | Both yield operands succeeded; delay is negative |
| `yield.non_increasing_time` | Resolved target does not exceed the previous target |
| `eval.input_delta_profile` | Executed eval input violates its admitted publication profile before start |

Where a primary location is the later of two, order a module's files with its
unnamed file first and its parts by part name, and each file by source order.
A test-only declaration is later than any other declaration of its module, and
a declaration later than any import. `import.cycle` is reported once for each
set of modules that import one another: at the first import into the set, in
that order, of its first module by path. `struct.inheritance_cycle` is
reported once for each set of structs that inherit from one another: at the
parent, among the set, of its first struct in that order.

The source and execution catalogs are disjoint. Unlisted failures still fail;
they cannot satisfy a coded expectation. Delta equality/bounds checks require
prior typing/formation; provider-backed key checks use the required run context.
The numeric code applies to explicit propagation, not to IEEE division or an
unpropagated Err. Other arithmetic codes are defined only when separately agreed.

## Warning annotation

```ebnf
warning_annotation = "@", "suppress_warning", "(", STRING_LITERAL, ")" ;
```

- **ERR-10.** At a named function, `@suppress_warning("node.no_activation")`
  acknowledges the dormant-node warning for that declaration. Other strings
  are checked against the warning catalog; suppression cannot hide source
  errors. `node.no_activation` is not an expected source/execution error code.

Accepted illustration, complete module:

```hgl
module examples.recovery

const fn price_or_zero(book: dict<str, i64>, key: str) -> i64 {
    let price = get(book, key)
    if price == null { return 0 }
    return price
}
```

Deliberate rejection: use price as a payload before its presence guard.
An absent strict lookup instead fails with `value.missing_key`. Conformance planned.

## Checked-division propagation

Accepted temporal declaration fragments, with an admitted f64 input
publication; `error` is a member of the `zero` option's enum:

```hgl
fn checked_data(value: f64) -> result<f64, error> {
    when { return value /{zero: .error} 0.0 }
}

fn propagated(value: f64) -> f64 {
    when { return (value /{zero: .error} 0.0)? }
}
```

`checked_data` publishes an ordinary Err carrying `numeric.division_by_zero`;
it does not fail the node and cannot satisfy an escaping-error assertion.
In `propagated`, explicit `?` turns the same error into node failure under
ERR-5, with no successful output publication. This is the catalogued execution
code's condition. Core owns the [numeric observations](https://github.com/hhenson/hgl_core/blob/main/spec/numeric-operators.md);
executable conformance is planned.

## Nullable value refinement

- **ERR-12.** An immutable nullable parameter or local may be tested directly
  against null. Refine that same binding to T on paths proving a non-null T?
  value, including after a terminating null branch. Using it where T is
  required without that proof rejects with `nullable.required_presence`.
  A null test establishes value presence, not temporal input validity.

The accepted `value_or_zero` example in [types and values](types-and-values.md)
uses this refinement. Deliberate source rejection:

```hgl
const fn unchecked(value: f64?) -> f64 => value
```

The unguarded return requires f64 but still has f64?. Expression `value?` does
not unwrap a nullable value: ERR-5 restricts postfix propagation to result
values. Executable source conformance is planned.

## Core boundary

Core supplies failure channels, owned diagnostic data, runtime capture, and
cleanup behavior. Source checking and code/location contracts belong here.
