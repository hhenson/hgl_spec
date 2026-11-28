# Errors and diagnostics

Status: agreed result, lookup, child-read (ERR-14), fallible-call (ERR-15),
nullable-read (ERR-12), capture, and structured-error contracts. A catalog
row whose condition begins "Proposed:" is a proposed code for an agreed
rejection, and a sentence that begins so is proposed. Codes for proposed features apply only within those proposed
profiles.

## Grammar

```ebnf
result_constructor = ( "ok" | "err" ), "(", expression, ")" ;
tag_test = ( "is_ok" | "is_err" ), "(", expression, ")" ;
error_constructor = "error", "(", error_argument, [ ",", error_argument ],
                    [ "," ], ")" ;
error_argument = ( "message" | "code" ), ":", expression ;
capture_call = "capture", "(", ( IDENTIFIER | qualified_name ),
               { ",", argument }, [ "," ], ")" ;
```

Each is a `postfix_expr` of the [expression grammar](expressions-and-bindings.md):
a call whose callee is the name written. `argument` is that grammar's call
argument. A failure is raised by the place a result is given (ERR-5,
ERR-15), and no operator raises one.

## Rules

- **ERR-1.** Distinguish source rejection, execution failure, ordinary result
  data, and test/infrastructure failure. Messages add context; a stable code
  identifies a catalogued condition. A warning does not reject valid source.
- **ERR-2.** `result<T,E>` owns exactly one tagged payload: a success owns one
  T, a failure one E. `ok(value)` constructs a success and `err(value)` a
  failure. Each takes one argument by position and no generic argument, and
  evaluates it once. Both type arguments come from the complete
  `result<T,E>` its place expects: `value` is read as that T for `ok` and as
  that E for `err`. Where the place expects no complete result type the
  constructor rejects (`result.context_type`), and a payload that is not of
  its side's type rejects as any value does (EXPR-5). Equality requires both
  payload types' equality and compares tag then selected payload. This adds
  no ordering/hash. Temporal outcomes publish atomically, including repeats.
- **ERR-3.** `is_ok(value)` and `is_err(value)` take one `result<T,E>` by
  position and give a `bool`; any other type rejects. An immutable result
  binding, a `let`, a parameter, or an iteration binding, gains its tag fact
  from a direct `is_ok`/`is_err` guard on it by its name: where `is_ok` holds
  it is a success and where it does not a failure, and `is_err` the reverse.
  `binding.value` is the T and `binding.error` the E; `.value`/`.error`
  requires the matching fact, and without it, or on what is no such binding,
  rejects (`result.required_variant`). Track
  negation, short-circuit paths, terminating branches, and merges. A saved bool,
  alias, fresh expression, or mutable var supplies no retained fact. Tag facts
  grant no temporal validity or modification fact. In composition a test of
  a connection is an ordinary lifted call (FN-6): it gives a connection of
  `bool` and no fact.
- **ERR-4.** `error(message: text, code: optional_code)` owns a required string
  message and optional string code, each given by its name: `message` is a
  `str`, and `code` a `str?` that is null when omitted. `.message` is the
  `str` and `.code` the `str?`. `.context` is a read-only nullable record of
  `.node: str`, `.phase: str`, one of `"evaluate"`, `"start"`, and `"stop"`,
  and `.evaluation_time: datetime`; a constructed error has none.
  Equality excludes provenance. On first
  graph failure, attach original node identity, phase, and evaluation time;
  propagation/capture preserve them. A node's display name is not a stable
  code. A nullable code or context read where a value is expected is read
  as ERR-12 says.
- **ERR-5.** A result given where its T is expected gives a success's T,
  and raises a failure (ERR-15). This rule says what raised means. In a
  value function that gives a `result<U,E>`, a `result<T,E>` of that same E
  raises its failure as the function's: the function gives that E as its
  failure. In a `when` only a `result<T,error>` raises, and the node fails.
  In a value function whose result is not a `result<U,E>`, one that gives
  an `i64` or gives nothing, only a `result<T,error>` raises too, and its
  failure is an execution failure (ERR-1), as an index outside a list is
  (EXPR-7): the function need not declare that it can fail. In a source's
  body only a `result<T,error>` raises, and its failure fails the source,
  as it fails a node in a `when`. In a lifecycle hook, in value code
  evaluated where a graph is built (ERR-15), and among a test's or an
  application's statements, only a `result<T,error>` raises, as a failure
  of that phase (RUN-4), and a null read there (ERR-12) fails that phase
  so too. Any other result given where its T is
  expected rejects (`result.propagation_type`): in a function that gives a
  `result<U,E>`, one of another E, and elsewhere one whose E is not
  `error`. No success output is written; completed effects stand.
  Returning Err as result data does not fail the node.
- **ERR-6.** Brackets on an ordinary or atomic dictionary are a strict
  point-in-time read: absence fails (`value.missing_key`). `get(dictionary,
  key)` is the read that can fail and says so (ERR-15): its place takes the
  result, whose failure has that code, or the value, raising it, or a null
  for an absent key; a `let` that writes no type holds the result. No
  other function is the strict read. Where V is nullable the result tells
  a held null from an absent key. `get(dictionary, key, fallback)` gives a
  V: the fallback has exact V type and is evaluated even for a present
  key. Only a `null` written as the fallback is no fallback, and that call
  is the plain read; any other fallback is a V, a null it evaluates to
  where V is nullable included. A present zero/false/empty value is not absent.
  An inferred immutable local that holds that result, used where a V is
  expected with no test, gives its value or raises its error (ERR-15).
  Explicit nullable annotations follow TYPE-14; this lookup contract adds no
  null-return suppression.
- **ERR-14.** A collection's child, `parts[i]`, is an endpoint (TIME-1),
  and an endpoint becomes a value only where a value is required of it:
  as an operand, as an argument a parameter takes as a value, in a
  `return` of a value, in a binding that writes a type, or in a hook's
  `var`, which is a place for a value: `var entry = parts[i]` reads the
  child where it is written. A graph's `var` denotes a connection and
  reads nothing (EXPR-4). A `let` that writes no type,
  `let entry = parts[i]`, requires none and holds the
  endpoint: nothing is read there and nothing fails there, `valid(entry)`
  and the other operations of an endpoint take it, and a later use of
  `entry` that requires a value reads it then. Where a value is required
  and the read is checked when it runs (TIME-1), it is a read that can
  fail: it gives the child's value, or, for a child that is not valid,
  the failure of an `error` whose code is `value.unset_read`, and it
  takes its form from the place that requires it as a call that gives
  `result<T, error>` does (ERR-15), T the child's type. Where the place
  expects `result<T, error>` it gives `ok` of the child's value, or the
  `err` of that error. Where it expects `T?` it gives the value, or null
  for a child that is not valid. Where it expects the T it gives the T,
  and
  a child that is not valid fails the read with `value.unset_read`: in a
  `when` the node fails, as a raised failure makes it (ERR-5). A read of a
  struct, a tuple or a fixed list whole that is checked when it runs,
  `rows[i]`, takes its form the same way, for one that is not all valid
  (TIME-1), though a field or a position of it is valid, and so does a
  read of an unsized list whole with a position that holds no
  value, where its element type has no value for that (TIME-1), a read of
  a dictionary whole with an entry that is valid and not all valid
  (TIME-1), and a read
  of a window's value before it is `all_valid`, T being the `list<V>` it
  reads as (TYPE-11). So does `delta_value` of any endpoint (TIME-9), T
  being its `delta<T>`: where the place expects `result<delta<T>, error>`
  it gives `ok` of the delta, or `err` coded `value.unset_read` where the
  endpoint has no delta in the cycle; where it expects `delta<T>?` it
  gives the delta, or null where there is none; elsewhere it gives the
  delta, and the read fails with that code where there is none. An endpoint that has
  not ticked in the cycle has none. Of those that tick, only a window of a
  duration has a tick with no delta, one in which a value only leaves it
  (TYPE-11): for every other endpoint the `result` form is `ok` where it
  is modified, so generic code may read the delta of a T it does not know
  in that form.
  `removed_value` of a window is read only where its place expects
  `result<V, error>`, V the window's value type: it gives `ok` of the
  value that left the window in the cycle, or that `err` where none left
  (TYPE-11).
- **ERR-15.** A function that can fail declares `result<T, error>` as its
  result. That result takes its form from its place wherever it is given
  (EXPR-5): what a call of the function gives, what a call of an operator,
  by its name or by its symbol, gives where its selected candidate gives
  `result<T, error>`, and a result value that a binding holds. A place
  that expects T or `T?` is met by such a candidate as by one that gives T
  (GEN-5), so it selects no policy: the configuration does (EXPR-12), and
  the place chooses the form of the result. Where the place expects
  `result<T, error>`, or `result<T, error>?`, which takes a result as any
  nullable type takes its value (TYPE-14), the result is given itself.
  Where it expects T, it
  gives a success's value, and a failure's own error is raised, where
  raising is admitted (ERR-5). Where it expects `T?`, it gives a success's
  value, or null for a failure, whose error is not kept.
  An operand of an operator expects the T where the operator's candidates
  take one: `checked(value) + 1` adds to the value or raises. Where a
  candidate takes the result itself, as `==` and `!=` of two
  results do (ERR-2), that candidate is selected and the operands are
  given whole: `left == right`, of two `result<i64, error>`, compares tag
  then payload, and raises nothing. Where one candidate is admitted by
  more than one choice of which operands to give whole and which to take
  the T of, and no choice takes fewer than the others, nothing chooses
  between them: the call rejects (`operator.ambiguous`), and the
  candidate's own conditions are what say which it means. Between two
  candidates that both admit the call, the one that takes the T of fewer
  operands is selected, whatever else ranks them: given two
  `result<i64, error>`, a candidate that takes `(result<i64, error>, i64)`
  is selected over one that takes `(i64, i64)`, as one that takes both
  whole is over either. Candidates that take the T of as many are ranked
  as any are (GEN-5), and tie where that ranks them equal. Compared
  with `null`, where T is not nullable, it expects the `T?`:
  `checked(value) == null` tests for a failure. Where T is nullable that
  place expects T, and takes the T form: the comparison is of a success's
  value, and a failure is raised. What a member or an index is read from,
  a condition, and what a `for` iterates each expect the T too, but for
  `.value` and `.error` read from a result, which are the result's own
  (ERR-3). Where nothing gives the place a type, a `let` that writes none
  among them, the result is held whole: `let outcome = checked(value)`
  binds the `result<i64, error>`, which `outcome + 1` then takes as its
  `i64`, `outcome == null` as its `i64?`, and `is_err(outcome)` as itself;
  `let price: i64 = checked(value)` binds the `i64`. Where T is itself
  nullable, a place that expects T takes the T form: a null there is a
  success's value and never a failure. Where T is a nullable `U?`, a
  place that expects the U takes the T form and then reads it (ERR-12):
  a failure's own error is raised, and a success that holds null fails
  the read with `value.null`. A place that expects any other type
  rejects as a value of another type does (`type.mismatch`). In
  composition a call that gives a connection, of a temporal function or
  one that is lifted (FN-6), takes its form as the connection's: of
  `result<T, error>`, which publishes each outcome; of T, where a success
  publishes its value and a failure fails the call's node in that cycle,
  as a failure raised in a `when` does, within whatever `capture` bounds
  it (ERR-7); and of `T?`, where a failure publishes null. A call that is
  evaluated where the graph is built and gives no connection (FN-6, FN-9)
  takes its form as value code does. A connection of `result<T, error>`
  that a binding holds, given where a connection of T or of `T?` is taken,
  takes that form as a call's does, in the node that reads it. Where a T
  is taken, that node fails in the cycle the failure is published, whether
  or not a handler of it would have read that input: a handler that tests
  `all_valid` of the input, or selects on another, does not run first.
- **ERR-7.** In composition, `capture(callable, args...)` creates a temporal
  error boundary. It names a callable and gives it the arguments its own call
  takes, positional or named, read, checked, and selected as that call's are
  (EXPR-2, FN-4, GEN-5). In a value function or a hook it rejects
  (`capture.phase`). What it gives is bound with `let` and read through two
  connections: `.value` retains the normal output type; outputless calls
  expose only `.error`, a connection of `error`, and `.value` rejects there
  (`capture.output_field`). Capture of wiring-time scalar work rejects
  (`capture.phase`). Capturing a node covers its evaluation; capturing a
  graph covers failures that leave that subgraph. Each
  evaluation failure ticks its canonical error; before failure the stream is
  invalid, and later success leaves its retained error valid but unmodified.
  A result the callable returns as data, an Err included, is its `.value`
  and no failure.
- **ERR-8.** Capture ends the failed pass without rollback or immediate retry.
  The child remains live for future ordinary work; unrelated outer/sibling
  work continues. The nearest boundary consumes the event once. Start/stop
  failures escape, including lifecycle work triggered during evaluation.
  Child errors follow child membership/lifetime. Earlier writes remain visible.
  A consumer that raises a retained error again (ERR-5) makes a new failure
  event, whose error keeps its first provenance.
- **ERR-9.** Required constant failures reject where specified. Optional
  folding of executed operations must preserve their execution-failure phase.
  Failure during argument evaluation precedes later admission checks. Do not
  demand cascaded diagnostics after an earlier error prevents checking.
- **ERR-11.** Numeric policies prefer representable ordinary outcomes and
  explicit HGL errors over exception mechanisms. NaN, infinity, and an unordered
  comparison outcome are ordinary data, not execution errors or silence. Core
  specifies each policy's error outcome; the policy must not silently convert
  ordinary data into an exception or an error into a successful empty output.
  A policy whose outcome is a `result<T, error>` gives that result whole;
  the form it takes is its place's (ERR-15).
- **ERR-13.** `ok`, `err`, `is_ok`, `is_err`, and `capture` are contextual
  names (LEX-10): as the callee of a call each is the form this chapter
  defines, whatever else is in scope. No declaration, import alias,
  parameter, or local takes one of them (UNIT-12); a member may (LEX-9).

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
| parse | `number.malformed` | Numeric, duration or tenor run is not one literal (LEX-8, LEX-18, LEX-22); run |
| parse | `number.integer_range` | Integer literal outside its required domain (LEX-7, LEX-23); literal |
| parse | `number.float_range` | Float literal rounds to infinity (LEX-7, LEX-20); literal |
| parse | `duration.invalid` | Duration violates LEX-13; literal |
| parse | `tenor.invalid` | Tenor violates LEX-28; literal |
| parse | `date_time.malformed` | `@` run matches no temporal form (LEX-18); run |
| parse | `date_time.invalid_field` | Date, time, or offset field violates LEX-11 or LEX-19; literal |
| parse | `date_time.invalid_zone` | Zone name violates LEX-12's source form, an empty component included; literal |
| parse | `doc.unattached` | Documentation comment documents no declaration (SDOC-1); comment |
| parse | `doc.repeated` | Documentation comment before the one a declaration has (SDOC-1); earlier comment |
| module | `module.duplicate_unnamed` | Module has a second unnamed file (UNIT-1); module path of every unnamed file but one |
| module | `module.duplicate_part` | Two files of a module share a part name (UNIT-1); part name in every such file but one |
| module | `import.misplaced` | Import after a declaration of its file (UNIT-2); `use` |
| module | `import.unknown_module` | No source declares the module (UNIT-2); module path |
| name | `doc.unknown_key` | Proposed: a key of a documentation comment's `Args:` or `Type Args:` section names no parameter, or no generic parameter, of the declaration it documents (SDOC-3); the key |
| name | `import.unknown_name` | Module declares no such name (UNIT-2); imported name |
| module | `import.private` | Imported declaration is not public (UNIT-2, UNIT-3); imported name |
| module | `import.cycle` | Modules import one another (UNIT-3); first import of the cycle |
| module | `application.duplicate_path` | Application's path is also a module's or another application's (UNIT-14); the path in every such file but one |
| module | `application.import` | `use` names an application (UNIT-14); module path |
| module | `application.export` | `export` in an application (UNIT-14); `export` |
| module | `application.run_count` | Application with no run statement, or with a second (UNIT-14); the application's path, or the later statement |
| phase | `application.statement` | Statement of an application that is no `inject` of `run_config` or of `global_state` and no call of `put` on the store, or that follows the run statement or precedes a declaration; and either of those in a module (UNIT-14); the statement |
| phase | `cache.input_read` | Proposed: a cache's initializer reads a temporal input (TIME-5, TIME-6); the read |
| phase | `run.position` | Run statement in a module or before a declaration of its application (UNIT-14), or `run` in a test that is no integration test (RUN-3); the word |
| type | `constant.failure` | A module's constant, or the default of a parameter or of a field, whose evaluation fails, an `i64` out of range or a division by zero among them (UNIT-13, UNIT-8, ERR-9); the expression |
| type | `run.entry` | A run statement's call is of no entry (RUN-1); the call |
| phase | `run.configuration` | A configuration on an application's run statement (RUN-6); the brace |
| name | `name.duplicate` | Binding takes a name that is not free to it (UNIT-5, UNIT-12, TST-2, NATIVE-2); later name |
| name | `name.used_before_declaration` | Local used before the statement that declares it (UNIT-6); use |
| name | `constant.cycle` | Constant whose value depends on itself (UNIT-13); name |
| name | `default.cycle` | Default whose evaluation needs that default again: a field's default that constructs its own struct leaving that field out, or a parameter's default that calls its own function leaving that parameter out, directly or through a function it calls (UNIT-8); the default |
| name | `name.reserved` | Proposed: a function named `scheduled` (UNIT-10, TIME-9), or a declaration introducing a type named `Type`, including a type parameter (TYPE-18); the declared name |
| name | `name.test_only` | Test-only declaration used outside the tests (TST-2); use |
| name | `name.captured` | Anonymous function uses an enclosing `var`, `state`, `cache`, or injected capability (FN-14); use |
| name | `default.formal_argument` | Default names a parameter of its declaration or a field of its struct (UNIT-7); use |
| name | `alias.unqualified` | Import alias used without `::name` (UNIT-2); alias |
| name | `alias.unknown` | Name before `::` is no import alias of its file (UNIT-2); name |
| name | `name.unknown` | Name that nothing in scope binds: no local, parameter, generic parameter, declaration, import, or prelude declaration (UNIT-10); name |
| name | `test.duplicate_name` | Two tests of a module share a name (TST-2); later name |
| module | `test.export` | Exported declaration in a test context (TST-1); `export` |
| operator | `test.implementation` | Operator candidate in a test context (TST-1); operator name |
| operator | `impl.unknown_operator` | Operator candidate names no operator (GEN-4); operator name |
| operator | `impl.pack` | Proposed: a candidate declares a fixed parameter where its operator declares a pack (GEN-6); the parameter's name |
| operator | `impl.parameter` | Proposed: a candidate writes for a parameter a type that is not the one its operator declares, or leaves out a `const` or a `var` its operator states (GEN-6); the parameter's name |
| operator | `impl.result` | Candidate's result is not the type its operator's declaration gives for the candidate's parameters (GEN-6); the result type |
| operator | `impl.requires` | Candidate for types its operator's `requires` does not admit (GEN-6); the candidate's name |
| operator | `impl.option_kind` | Candidate declares as a parameter an option that another candidate of its operator declares as a generic value parameter (GEN-6); the parameter's name |
| operator | `property.duplicate` | A domain that a second property clause of an operator names, or that one clause names twice, or a property that one clause gives twice (GEN-9); the later domain or property |
| name | `native.missing_declaration` | Native implementation completes no declaration (NATIVE-2); function name |
| phase | `native.const_hook` | Native const implementation has a lifecycle hook (NATIVE-3); hook |
| phase | `native.hook_without_when` | Start or stop hook with no `when` hook (NATIVE-3); hook |
| phase | `native.duplicate_hook` | Second start or stop hook (NATIVE-3); later hook |
| injectable | `inject.unknown_capability` | `inject` names no capability (TIME-8); name |
| injectable | `inject.out_without_result` | `inject out` in a function that declares no result (TIME-7); `out` |
| injectable | `capability.as_value` | Capability retained: the value of a `state` or `cache`, an entry of a tuple, list, set, dictionary, or struct, returned, or published; a capability given to a parameter written as a generic parameter or `any`; or a capability type written other than as a parameter's of a `const fn` or a native value function, or in a membership of an operator's declaration (TIME-8); use |
| injectable | `inject.alarm_and_scheduler` | `alarm` and `scheduler` injected in one node (TIME-8); the later name |
| injectable | `inject.position` | `inject run_config` in a function, or in a test that is no integration test (RUN-6); the name |
| injectable | `inject.configuration` | A configuration in braces on an `inject` of a capability that is not `run_config` (RUN-6); the brace |
| injectable | `inject.no_scheduler` | `scheduled()` in a node that injects neither `scheduler` nor `alarm` (TIME-9); the selector |
| injectable | `store.reserved_key` | A key of the global store that is the run's own, `"run.output"` (TIME-8); the key |
| injectable | `store.key_type` | A key of the global store used with two types in one program: a `put` or a `get` of another type than an earlier use of the key has (TIME-8, core's CAP-6); the later use |
| phase | `node.nested` | `state`, `cache`, `inject`, `start`, `when`, or `stop` inside a block of a temporal function (TIME-3, TIME-5); keyword |
| phase | `node.in_value_function` | `state`, `cache`, `inject`, `start`, `when`, or `stop` in a `const fn` (FN-1, FN-9); keyword |
| phase | `node.declaration_order` | `state`, `cache`, or `inject` after a `start`, `when`, or `stop` (TIME-5); the declaration |
| phase | `node.duplicate_hook` | Second `start` or second `stop` (TIME-5); later hook |
| phase | `node.statement` | Statement of a node's body that is no declaration and no hook, in a node that is no source (FN-2); the statement |
| phase | `scheduled.outside_node` | Proposed: `scheduled()` in a value function, a graph or a test, which has no schedule of its own (FN-9, TIME-9); the selector |
| phase | `source.return_value` | A source's `return` has a value, under SOURCE-4, which is a proposed rule: the rejection is proposed with it, and so is this code; the value |
| phase | `source.statement` | In a source with a declaration or a hook: a statement outside its one block, a second block, a hook out of order, or a `yield` in a hook. In any source: a `var`, or an `inject` of `out`, `scheduler`, or `alarm` (SOURCE-1); the statement |
| phase | `source.when` | `when` in a source (SOURCE-1); `when` |
| phase | `source.input` | Temporal parameter of a source (SOURCE-1); the parameter |
| phase | `source.result` | Proposed: a source with no result arrow (SOURCE-1); the source's name |
| phase | `state.input_read` | State initializer reads a temporal input (TIME-5); the read |
| phase | `out.lifecycle` | `out` used in `start` or written in `stop`, or a `return` with a value in either (TIME-7); the use |
| phase | `input.lifecycle` | A temporal input read, or given to an endpoint operation, in `start`, or given to `activate` or `passivate` in `stop` (TIME-5); the use |
| phase | `payload.unguarded` | Payload read under manual admission, in a `when` or in `stop`, that no guard dominates (TIME-4, TIME-5); or a read of a child at a constant index or a field that nothing admits, or of a struct, a tuple or a fixed list whole that its `all_valid` does not admit (TIME-1); or a read of the delta of a window of a duration whose place is neither a `result` nor a `delta<T>?`, or of `removed_value` of any window whose place is not a `result` (TYPE-11); read |
| phase | `ref.read` | The value or the `delta_value` of a `ref` parameter, of a child of one, or of `out` of a `ref` result, read (FN-16), or the `removed_value` of a window that is a child of one; the read |
| phase | `endpoint.argument` | Fixed endpoint parameter or borrowed pack entry given what is no temporal input of its function, no child of one, and not `out`, or `all_valid` given neither a collection (a list, tuple, struct or dictionary) nor a window (TIME-9); argument |
| type | `schedule.duration_bounds` | A wake asked for with a `duration` that is not more than zero, other than a zero in `start`, whose value the compiler sees (TIME-8); the duration |
| phase | `scheduled.lifecycle` | `scheduled()` in `start` or `stop` (TIME-9); the selector |
| phase | `connection.as_constant` | Temporal connection where composition requires a constant (FN-3), given to a function that declares a `var` parameter, which is not lifted (FN-15), given to a value function's configuration parameter (EXPR-14), or given to a call on the store a graph injects (FN-9); expression |
| type | `generic.invalid_application` | Generic type's name before a `<` that opens no arguments (GEN-14); name |
| type | `pack.default` | Pack parameter has a default (FN-8); `=` |
| type | `pack.const` | Pack parameter is const (FN-8); `const` |
| type | `pack.order` | Fixed parameter follows a pack (FN-8); parameter name |
| type | `pack.undeclared` | `...{Name}` names no type pack of its declaration (FN-8); name in the braces |
| type | `function.argument` | An argument for a parameter that takes a function names no function known where the call is compiled (FN-17); the argument |
| type | `pack.name` | `Ps[name]` for a name the pack has not (GEN-17); the name |
| type | `pack.duplicate` | A second positional pack, or a second named pack, in one declaration (FN-8); the second pack's parameter |
| type | `pack.cardinality` | Cardinality's upper bound is below its lower (GEN-10); upper bound. A call that gives a pack a number of arguments its cardinality does not admit; the first argument beyond the upper bound, or the call's closing parenthesis where too few are given |
| type | `switch.label_constant` | Proposed: a `switch` label that is no compile-time constant (CTRL-12, UNIT-13); the label |
| type | `type.not_a_type` | Name in a type position is no type: a function, operator, local, generic value parameter, or a type pack where one type is needed (UNIT-10, GEN-2, GEN-13); name |
| type | `type.not_ordinary` | `rolling_window`, `atomic` or `signal` where an ordinary value type is required, or under a `?` that is not on the whole type of an input or a field (TYPE-12, TYPE-14, FN-1); offending type |
| type | `signal.position` | `signal` other than as the whole type of a function's temporal parameter (TYPE-12); `signal` |
| type | `endpoint.position` | Endpoint type outside a native value function's fixed endpoint parameter or endpoint pack (TYPE-12, FN-8); `endpoint` |
| type | `nullable.key` | Set element or dictionary key has a `?` (TYPE-14); the nullable type |
| type | `key.unstable` | Set element or dictionary key is an `f64`, or a type that holds one (TYPE-1); the type |
| type | `generic.argument_count` | Generic type given the wrong number of arguments, or arguments given to a type or generic parameter that takes none (GEN-2, GEN-10, GEN-13); name |
| type | `list.size` | List size is not a nonnegative count (TYPE-10); size |
| type | `generic.kind_conflict` | Generic parameter used as a type and as a value (GEN-13); later use |
| type | `generic.kind_unresolved` | No field or constraint says what a struct's generic parameter is (GEN-13); parameter name |
| type | `generic.argument_kind` | Generic argument is a value where the parameter is a type, or a type where it is a value (GEN-14); argument |
| type | `generic.unresolved` | Call leaves a generic parameter of its callee unresolved (GEN-2, GEN-8); callee name |
| type | `generic.instance_limit` | Instances of one generic declaration past the implementation's bound, as instances that never end when the program is compiled are (GEN-13); the call that makes the first of them |
| type | `generic.argument_constant` | Call gives a generic value parameter of its callee no compile-time constant (GEN-8); argument |
| constraint | `requires.unmet` | Call of a function, or application of a struct, whose `requires` does not hold with its generic arguments (GEN-3, GEN-16); the call or the application |
| type | `struct.parent` | Parent is not an abstract struct (TYPE-3); parent |
| type | `struct.duplicate_parent` | Struct lists one parent twice (TYPE-3); later parent |
| type | `struct.inheritance_cycle` | Structs inherit from one another (TYPE-6); a parent of the first struct of the cycle |
| type | `field.conflict` | Field's type or optionality differs between two parents, or between a struct and a parent (TYPE-4); field, or the later parent |
| type | `field.not_inherited` | `name = value` names no field of a parent (TYPE-4); name |
| type | `field.duplicate` | Two members of a struct, or two properties of a native type, share a name (TYPE-4, NATIVE-6); later name |
| type | `field.recursive` | Field that makes its struct hold itself has no value that ends the recursion, being neither optional nor a collection that may be empty, or holds it through collection elements outside an `atomic<...>`, or gives a struct of its cycle a type built from a generic parameter, or has a default that is neither null nor empty, or a descendant's default removes its null (TYPE-6); field name, or the parent whose argument makes the field hold the struct |
| type | `field.null_default` | Null default on a field that is not optional (TYPE-4); default |
| type | `enum.value` | Enum number is not an integer (TYPE-7); number |
| type | `enum.duplicate_value` | Two members of an enum share a number (TYPE-7); later member |
| type | `enum.overflow` | Unnumbered member follows the largest `i64` (TYPE-7); member |
| type | `enum.duplicate_member` | Two members of an enum share a name (TYPE-7); later name |
| type | `type.mismatch` | Value's type is not the type its place expects, nor a `T?` the place reads (ERR-12) or a result whose form it takes (ERR-5, ERR-15): an annotation, a `var`, a result, an operand, an element, an argument (EXPR-5); `null` written where the type has no `?` (TYPE-14); a `return` that does not fit its function (FN-7), a value among them where a `ref` result takes an endpoint (FN-16); a list of another length than the size expected (TYPE-10); value |
| type | `null.context_type` | `null` where nothing gives it a type (TYPE-14); the literal |
| type | `sequence.context_type` | Sequence whose entries give no type, where none is expected (EXPR-9); the sequence |
| type | `dict.duplicate_key` | Two keys of a dictionary written as pairs are the same constant (EXPR-9); the later key |
| type | `set.duplicate_member` | Proposed: two entries of a set written out are the same constant (EXPR-9); the later entry |
| type | `format.syntax` | Proposed: a format string of core's in which a brace opens no place and closes none (core's FMT-3); the brace |
| type | `format.argument` | Proposed: a place of a format string of core's that has no argument: one that takes the next positional argument where none is left, or one that names no named argument (core's FMT-3), the evaluation time asked for in value code among them (core's FMT-9); the place |
| type | `format.unused` | Proposed: an argument of a call with a format string of core's that no place takes (core's FMT-3); the argument |
| type | `format.specifier` | Proposed: a specifier in a place of a format string of core's that its argument's type does not admit (core's FMT-3, FMT-4), or a mark on a place that takes none (core's FMT-9); the specifier or the mark |
| type | `enum.context_type` | `.member` where no enum type is expected (TYPE-7); the member |
| name | `binding.unassigned` | `var` read on a path that gives it no value (EXPR-3); the read |
| name | `binding.immutable` | Assignment to an immutable binding: a `let`, a parameter, an iteration binding, or a field or element of one (EXPR-3), or to a property of a native type (NATIVE-6) or of a delta (TYPE-16); or one given to a `var` parameter (FN-15). Also an assignment to a part of `out` of a `ref` result, `out.bid = x` (FN-16); target |
| type | `binding.category` | Assignment of a connection to a `var` that holds an ordinary value, or the reverse; assignment in a branch of a temporal `if` or `switch` to a `var` declared before it that holds an ordinary value (EXPR-4, CTRL-14); the value |
| type | `argument.order` | Argument with no name after a named one (EXPR-2); the argument |
| type | `argument.unexpected` | Positional argument that no parameter is left to take (EXPR-2), one given to a struct's constructor (TYPE-5), or a second given to an enum's (TYPE-7); the argument |
| name | `argument.unknown` | Named argument names no parameter or field (EXPR-2); name |
| name | `argument.duplicate` | Parameter or field given twice (EXPR-2); later name |
| type | `argument.missing` | Parameter or field that may not be left out is given nothing (EXPR-2, TYPE-5); callee name |
| type | `argument.constant` | In a hook or a value function, a `const` parameter given what is no constant expression (FN-1); argument |
| type | `argument.place` | `var` parameter given what is no place: a literal, what a call gives, or any other expression that nothing assigns (FN-15); argument |
| function-kind | `parameter.var` | `var` parameter of a temporal function or a temporal candidate, or one whose type is a capability's or `endpoint<T>` (FN-15); `var` |
| function-kind | `ref.position` | `ref` on a parameter or the result of what is no node, a value function, a graph or an operator, or on a `signal` parameter (FN-16, TYPE-12), or on a parameter or a result whose type is a `rolling_window` (FN-16); `ref` |
| function-kind | `call.role` | Temporal function called from a value function or a runtime hook (FN-3); in a test's or an application's value code, a temporal function called, or an operator applied whose selected candidate is temporal (TST-3, UNIT-14); callee name |
| type | `return.missing` | Value function that declares a result can reach its end with no value (FN-7), or a graph that returns a value in a branch of a temporal `if` or `switch` can (CTRL-14); function name |
| type | `switch.duplicate_label` | Switch label has the value of an earlier label (CTRL-6); later label |
| phase | `while.condition` | `while` with no condition in a value function or a graph (CTRL-12, CTRL-15); `while` |
| type | `for.sequence` | Proposed: `for` in a graph over a connection of a tuple, or over one of `list<T>`, `set<T>`, or `dict<K, V>`, whose members change while the program runs (CTRL-15); the sequence |
| type | `struct.abstract` | Abstract struct constructed (TYPE-3); name |
| type | `call.not_callable` | Call of a native type's name, which no call constructs (NATIVE-6); name |
| name | `member.unknown` | Struct has no field, or enum no member, of the name, or of the constant number or name an enum is constructed from (EXPR-7, TYPE-7); name or constant |
| operator | `operator.no_candidate` | Call of an operator, by symbol or by name, that no candidate admits (GEN-5), or, in a generic body, that its requirements make no candidate known to take (GEN-7); operator |
| operator | `operator.ambiguous` | Call of an operator that two most-specific candidates admit equally (GEN-5), or that one candidate admits by more than one choice of which results to take the value of, none taking fewer than the others (ERR-15); operator |
| type | `default.order` | Parameter that may not be omitted follows one that may (UNIT-8); the later parameter |
| type | `tuple.index` | Tuple's index is no constant, or is outside the tuple (EXPR-7); index |
| type | `rolling_window.size_kind` | Wrong/mixed size kind; offending size, the minimum when the two differ in kind |
| type | `rolling_window.size_bounds` | Invalid bounds; offending size/minimum |
| type | `yield.time_type` | Wrong time type; yield time operand |
| type | `test.raises_code` | Not a literal catalogued execution code; argument |
| type | `eval.target` | Eval's call is of no function of a module: a call of an operator among them (TST-5); or of a function that declares a `var` parameter (FN-15); the call |
| type | `eval.no_result` | The value of an eval of an outputless target is bound or compared (TST-8); the eval |
| type | `trace.placeholder` | `_` in a timed trace, or outside a trace of a test (EXPR-8, TST-7); the `_` |
| type | `trace.style` | Dense and timed traces in one eval, its expectation included (TST-7); the later trace |
| type | `trace.key_order` | Constant keys of a timed trace that do not strictly increase (TST-7); the later key |
| type | `trace.escape` | A trace stored in a field, returned, or given as a temporal argument outside an eval (TST-3); the use |
| type | `lookup.default_type` | Non-null fallback is not exact V; fallback |
| type | `result.context_type` | Missing complete outcome context; constructor |
| type | `result.required_variant` | Unguarded tag payload; property |
| type | `result.propagation_type` | A result given where its T is expected whose E may not be raised there: in a function that gives a `result<U, E>`, another E; elsewhere, an E that is not `error` (ERR-5); the result |
| phase | `capture.phase` | Non-temporal capture/context; call |
| type | `capture.output_field` | Value requested from outputless capture; property |
| type | `value.constant_capability` | Required constant lacks contained capability; operation |
| shape | `delta.unsupported_shape` | `delta<T>` of a T that has no delta: a `signal`, or proposed a `Type` or a type that holds one (TYPE-16, TYPE-18); the application |
| type | `delta.type_mismatch` | Incompatible originating delta/destination; value |
| name | `delta.argument_name` | Unknown delta field/argument, or the constructor form of a delta that is no type of its own (TYPE-16); name |
| name | `delta.duplicate_argument` | Repeated delta argument; later name |
| type | `delta.entry_constant` | Entry is not admitted constant/cold data; entry |
| type | `delta.entry_type` | Wrong entry key/index/member type; entry |
| type | `delta.duplicate_entry` | Known equal entries; later entry |
| type | `delta.index_bounds` | Negative/out-of-fixed-range index, or an `updated` index at or past the size a `size_modified` of the same delta gives (TYPE-16); index |
| type | `delta.overlap` | Known overlapping addition/removal; later entry |
| phase | `test.statement_phase` | Node/lifecycle/return statement in test; keyword |
| phase | `yield.in_value` | A `yield` inside a block or an `if` whose value is used, under SOURCE-4, which is a proposed rule: the rejection is proposed with it, and so is this code; the `yield` |
| phase | `yield.outside_source` | Proposed: a `yield` directly in a `const fn`, or inside an anonymous function, whatever function encloses it: neither is a source's own body (SOURCE-1, FN-1, FN-14); the `yield` |

| Execution code | Required condition |
| --- | --- |
| `value.missing_key` | Strict ordinary dictionary lookup has no member; also the code of the failure that `get` of an absent key gives, of a dictionary (ERR-6) or of the global store (TIME-8, core's CAP-6) |
| `key.unstable` | An `any` that holds an `f64`, or a value of a type that holds one, is given to a set or as a dictionary's key (TYPE-1) |
| `value.null` | A `T?` that holds null is read where a T is expected, a connection's by the node that reads it among them (ERR-12) |
| `value.index_out_of_range` | An index outside a list, below zero or at or past its size, where a read or an assignment evaluates it (EXPR-3, EXPR-7) |
| `value.unset_read` | Executed operation requires an unset ordinary payload, a collection's child that is not valid, or a struct, a tuple or a fixed list read whole that is not all valid, read where that is checked when it runs among them (TIME-1, ERR-14), an unsized list read whole with a position that holds no value, where its element type has no value for that (TIME-1), a window's value read before it is `all_valid` (TYPE-11), or a `delta_value` read whose place is no `result` where its endpoint has no delta in the cycle (ERR-14) |
| `value.capability` | Boxed executed operation lacks a required contained capability |
| `value.byte_range` | Bytes construction receives an octet outside 0–255 |
| `enum.no_member` | An enum is constructed, where it is evaluated, from an `i64` or a `str` that is no member's number or name (TYPE-7) |
| `switch.unmatched` | A `switch` with no `default` is given a selector that matches none of its labels, in composition as elsewhere (CTRL-6) |
| `store.key_type` | A `put` or a `get` of a key of the global store of another type than an earlier use of the key, where the two uses are not in one program (TIME-8, core's CAP-6) |
| `store.reserved_key` | Proposed: a key of the global store that is the run's own, `"run.output"`, where the program's text does not fix the key and it takes that value as the graph is built, or in a call the run decides (TIME-8, core's CAP-6) |
| `numeric.division_by_zero` | A checked division's Err is raised, given where its `f64` is expected (ERR-5, ERR-15); see the [core numeric contract](https://github.com/hhenson/hgl_core/blob/main/spec/numeric-operators.md) |
| `numeric.out_of_range` | A result is outside its type's range: an `i64`, a `duration`, an instant or a count of a `tenor`; or a `str` is repeated a count below zero; see the core [numeric](https://github.com/hhenson/hgl_core/blob/main/spec/numeric-operators.md) and [scalar operator](https://github.com/hhenson/hgl_core/blob/main/spec/scalar-operators.md) contracts |
| `numeric.invalid_tolerance` | A tolerance of `==` or `!=`, `abs` or `rel`, is negative or not finite; see the [core numeric contract](https://github.com/hhenson/hgl_core/blob/main/spec/numeric-operators.md) |
| `tenor.mixed_units` | A sum or a difference of a `tenor` of business days and one of months and days; see the core [scalar operator contract](https://github.com/hhenson/hgl_core/blob/main/spec/scalar-operators.md) |
| `convert.invalid` | A conversion fails under `invalid: .fail`, or a checked conversion's Err is raised, given where its value is expected (ERR-5, ERR-15); see the core [scalar operator contract](https://github.com/hhenson/hgl_core/blob/main/spec/scalar-operators.md) |
| `yield.negative_duration` | Both yield operands succeeded; delay is negative |
| `schedule.non_positive_duration` | A wake asked of the scheduler or the alarm with a `duration` that is not more than zero, other than a zero in `start`, whose value the compiler does not see (TIME-8) |
| `yield.non_increasing_time` | Resolved target does not exceed the previous target |
| `eval.input_delta_profile` | Executed eval input violates its admitted publication profile before start |
| `run.end_and_length` | A run's configuration gives both an `end` and a `length`; refused before the run's graph is built. See the [core execution contract](https://github.com/hhenson/hgl_core/blob/main/spec/execution.md) (CRUN-2) |
| `run.bounds` | A run's start or end is outside the instants a run may have: a start before the earliest start, or an end after the latest end. See the [core execution contract](https://github.com/hhenson/hgl_core/blob/main/spec/execution.md) (CRUN-3) |

Where a primary location is the later of two, order a module's files with its
unnamed file first and its parts by part name, and each file by source order.
A test-only declaration is later than any other declaration of its module, and
a declaration later than any import. `import.cycle` is reported once for each
set of modules that import one another: at the first import into the set, in
that order, of its first module by path. `struct.inheritance_cycle` is
reported once for each set of structs that inherit from one another: at the
parent, among the set, of its first struct in that order.

The source and execution catalogs are disjoint, but for two codes.
`store.key_type` is a source rejection where the two uses are in one
program, and an execution failure where they are not. `key.unstable` is a
source rejection where the type is written, and an execution failure for
what an `any` holds. Proposed: `store.reserved_key` is a third, a source
rejection where the program's text fixes the key, and an execution
failure where it does not: as the graph is built, where the key is fixed
then, and as the call is evaluated, where the run decides it (TIME-8).
Unlisted failures still fail;
they cannot satisfy a coded expectation. Delta equality/bounds checks require
prior typing/formation; provider-backed key checks use the required run context.
The numeric code applies to a raised Err, not to IEEE division or an Err
kept as data. Other arithmetic codes are defined only when separately agreed.

## Warning annotation

```ebnf
warning_annotation = "@", "suppress_warning", "(", STRING_LITERAL, ")" ;
```

- **ERR-10.** At a named function, `@suppress_warning("node.no_activation")`
  acknowledges the dormant-node warning for that declaration. Other strings
  are checked against the warning catalog; suppression cannot hide source
  errors. `node.no_activation` is not an expected source/execution error code.
  An expression statement that holds no call, assignment, or `return` at
  any depth has no effect, and its dropped value is a warning,
  `statement.no_effect`: `a + 1` on a line of its own. A call is not
  reported, nor is a block or an `if` that assigns.

| Warning code | Condition |
| --- | --- |
| `name.hides_function` | A local, a parameter, or an iteration binding of a function type that takes the name of a function or an operator, which it hides in its scope (UNIT-12); the binding |
| `node.no_activation` | The dormant-node warning: `modified([])` in a node that injects neither `scheduler` nor `alarm`, or whose `start` does not use the one it injects (TIME-3); the selector |
| `operator.candidates_tie` | Two candidates of an operator that some call would match equally, with no other candidate the more specific for it (GEN-6); each of the two candidates |
| `operator.sequence_reading` | An operator call selects one candidate after comparing list and set readings (GEN-5, EXPR-9); each literal taken as a list where an admitted combination took a set |
| `statement.no_effect` | Expression statement that holds no call, assignment, or `return`, its value dropped; the expression |

Accepted illustration, complete module:

```hgl
module examples.recovery

const fn price_or_zero(book: dict<str, i64>, key: str) -> i64 {
    let price = get(book, key)
    if price == null { return 0 }
    return price
}
```

`price` holds the result of `get` (ERR-15): `price == null` takes its
`i64?` form, null for a key the book has not, and `return price` its `i64`.
Returned before that test, `price` gives the value, or raises its error,
coded `value.missing_key`, for such a key. A strict lookup, `book[key]`,
fails with that code too. Conformance planned.

Accepted fragments, the fallback of `get` (ERR-6), with `book` a
`dict<str, i64>`, `notes` a `dict<str, i64?>`, and `other` an `i64?`:

```hgl
let a: i64 = get(book, key, 0)
let b = get(book, key, null)
let c = get(notes, key, other)
```

`a` is the book's value for the key, or 0 for a key it has not: the
fallback gives an `i64`, and nothing fails. In `b` the `null` is written,
so it is no fallback and the call is the plain read: `b` holds a
`result<i64, error>`, as `get(book, key)` gives. In `c` the fallback is an
expression of exactly the dictionary's value type, `i64?`, so `c` is an
`i64?` and no result: for a key `notes` has not it is what `other`
evaluates to, a null included. Deliberate rejection: `get(book, key,
"none")`, a fallback that is not exactly V (`lookup.default_type`).

Accepted illustration, complete module, a child read when it runs (ERR-14):

```hgl
module examples.child_reads

fn at(parts: list<i64, 2>, slot: i64) -> i64 {
    when { return parts[slot] }
}

fn checked_at(parts: list<i64, 2>, slot: i64) -> result<i64, error> {
    when { return parts[slot] }
}
```

With position 0 alone published, as 3, and `slot` publishing 0 and then 1,
`at` publishes 3 and then fails with `value.unset_read`; `checked_at`
publishes `ok(3)` and then an `err` whose error has that code, and does not
fail. Deliberate rejection: `return parts[1]` in either, a child at a
constant index that nothing admits (`payload.unguarded`). Conformance planned.

## Result forms

| Form | Operand | Gives | Rejects |
| --- | --- | --- | --- |
| `ok(value)` | A T, where the place expects `result<T,E>` | That result, a success | `result.context_type` |
| `err(value)` | An E, where the place expects `result<T,E>` | That result, a failure | `result.context_type` |
| `is_ok(value)`, `is_err(value)` | A `result<T,E>` | A `bool`; a tag fact for an immutable binding | `type.mismatch` |
| `binding.value` | A binding proved a success | The T | `result.required_variant` |
| `binding.error` | A binding proved a failure | The E | `result.required_variant` |
| `outcome` where `result<T, error>` or `result<T, error>?` is expected, or bound by a `let` that writes no type | A `result<T, error>`: what a call gives whose function, or whose operator's selected candidate, gives one, or what a binding holds (ERR-15) | The result itself | None |
| `outcome` where T is expected, an operand among them | The same, or in a function that gives a `result<U, E>` a `result<T, E>` (ERR-5) | The T; a failure is raised: it leaves a function that gives a result as its failure, fails the node in a `when` and the source in its body, fails the phase of a lifecycle hook, of a graph's building, and of a test's or an application's statements, and is an execution failure in any other value function | `result.propagation_type` |
| `outcome` where `T?` is expected, compared with `null` among them where T is not nullable | The same | The T, or null for a failure | None |
| `parts[i]` where `result<T, error>` is expected | A collection's child read when it runs (ERR-14) | `ok` of its value; `err` coded `value.unset_read` for a child that is not valid | None |
| `delta_value(x)` where `result<delta<T>, error>` is expected | An endpoint (ERR-14) | `ok` of the delta; `err` coded `value.unset_read` where the endpoint has no delta in the cycle | None |
| `removed_value(x)` | A window, where `result<V, error>` is expected (ERR-14) | `ok` of the value that left; `err` coded `value.unset_read` where none left | `payload.unguarded` elsewhere |
| `error(message: text, code: code)` | A `str`, and a `str?` | An `error` | `type.mismatch` |
| `capture(callable, args...)` | A temporal call | `.value` and `.error` connections | `capture.phase`, `capture.output_field` |

Accepted illustration, complete module:

```hgl
module examples.results

const fn checked(value: i64) -> result<i64, error> {
    if value < 0 {
        return err(error(message: "negative price", code: "price.negative"))
    }
    return ok(value)
}

const fn doubled(value: i64) -> result<i64, error> {
    let price: i64 = checked(value)
    return ok(price * 2)
}

const fn recovered(value: i64) -> i64 {
    let outcome = checked(value)
    if is_err(outcome) { return 0 }
    return outcome.value
}

const fn or_null(value: i64) -> i64? => checked(value)

const fn or_zero(value: i64) -> i64 {
    let price = checked(value)
    if price == null { return 0 }
    return price
}

const fn trusted(value: i64) -> i64 => checked(value)

const fn present(outcome: result<i64?, error>) -> i64 => outcome

const fn diagnostic(outcome: result<i64, error>) -> str {
    if is_ok(outcome) { return "ok" }
    return outcome.error.message
}

fn required(value: i64) -> i64 {
    when { return checked(value) }
}

fn tagged(value: i64) -> result<i64, error> {
    when { return checked(value) }
}

fn is_failure(value: result<i64, error>) -> bool => is_err(value)

fn lifted_required(value: i64) -> i64 => checked(value)

fn lifted_or_null(value: i64) -> i64? => checked(value)

fn wired(const value: i64) -> i64 => checked(value)
```

| Evaluation | Outcome |
| --- | --- |
| `checked(7)` | A success holding 7 |
| `checked(-1)` | A failure holding the error "negative price", code `price.negative` |
| `doubled(7)` | A success holding 14 |
| `doubled(-1)` | The failure of `checked(-1)`, unchanged; the multiplication is not reached |
| `recovered(7)`, `recovered(-1)` | 7, and 0 |
| `or_null(7)`, `or_null(-1)` | 7, and null: the failure's error is not kept |
| `or_zero(7)`, `or_zero(-1)` | 7, and 0: `price` holds the result, whose `i64?` form is null for the failure |
| `trusted(7)`, `trusted(-1)` | 7, and an execution failure with that error: `trusted` declares no result |
| `present` given a success holding 7, one holding null, and a failure | 7; an execution failure coded `value.null`, the read of the null; and an execution failure with the failure's own error |
| `diagnostic(checked(-1))` | "negative price" |
| `required` over the publications 7, -1 | Publishes 7; then the node fails with that error and publishes nothing |
| `tagged` over the publications 7, -1, -1 | Three ticks: a success, a failure, and the equal failure again. The node does not fail |
| `is_failure` over a success, a failure, a failure | `false`, `true`, `true`: `is_err` lifted over the connection |
| `lifted_required` over the publications 7, -1 | Publishes 7; then its node fails with that error, as `required` does |
| `lifted_or_null` over the publications 7, -1 | Publishes 7, then null |
| `wired(value: 7)`, `wired(value: -1)` | A graph that publishes 7 once, when it starts; and a failure of the graph's building with that error: the call is evaluated as the graph is built (ERR-5) |

Each call of `checked` takes its form from its place (ERR-15): the result
in `recovered` and `or_zero`, whose `let`s write no type, and where
`diagnostic` and `tagged` take one; the `i64` in `doubled`, whose `let`
writes that type and whose failure is then its own, in `trusted`, in
`required`, in `lifted_required`, and in `wired`, a graph whose call of
`checked` gives no connection; and the `i64?` in `or_null` and
`lifted_or_null`. `required` and `tagged` differ only in what their
results expect. As an operand, `checked(value) + 1` takes the `i64`. A
result that a binding holds takes its form the same way: in `or_zero`,
`price == null` takes the `i64?` and `return price` the `i64`; in
`recovered`, `let price: i64 = outcome` before the test gives the value,
or raises the failure.
In `recovered` the early return under `is_err(outcome)` proves `outcome` a
success after it, so `outcome.value` is read; in `diagnostic` the return
under `is_ok` proves it a failure. `error.message` is a `str`; its `code`
is a `str?`, read as ERR-12 says where a `str` is expected. Deliberate
rejections:
`let missing = ok(1)`, with nothing that says the result's type;
`outcome.value` with no guard, and under `is_err(outcome)`; a
`result<i64, str>` given where an `i64` is expected, in a `when` and in a
`const fn` that returns an `i64` (`result.propagation_type`); and a
function named `ok`. Conformance planned.

Accepted illustration, complete module:

```hgl
module examples.capture

const fn checked(value: i64) -> result<i64, error> {
    if value < 0 { return err(error(message: "negative price")) }
    return ok(value)
}

fn required(value: i64) -> i64 {
    when { return checked(value) }
}

fn forward(value: i64) -> i64 {
    when { return value }
}

fn failed(failure: error, step: i64) -> bool {
    when modified(step) && valid(step) { return modified(failure) }
}

fn held(failure: error, step: i64) -> bool {
    when modified(step) && valid(step) { return valid(failure) }
}

fn captured_value(value: i64) -> i64 {
    let seen = capture(required, value)
    forward(seen.value)
}

fn captured_events(value: i64, step: i64) -> bool {
    let seen = capture(required, value)
    failed(seen.error, step)
}

fn captured_presence(value: i64, step: i64) -> bool {
    let seen = capture(required, value)
    held(seen.error, step)
}
```

With `step` published on every cycle:

| Cycle | `value` | `seen.value` | `seen.error` valid | `seen.error` modified | Error held |
| --- | --- | --- | --- | --- | --- |
| 0 | 7 | Ticks 7 | No | No | None |
| 1 | -1 | Silent, 7 retained | Yes | Yes | "negative price" |
| 2 | Silent | Silent | Yes | No | "negative price" |
| 3 | 8 | Ticks 8 | Yes | No | "negative price" |
| 4 | -1 | Silent, 8 retained | Yes | Yes | "negative price" |
| 5 | -1 | Silent | Yes | Yes | "negative price" |

`captured_value` publishes 7 at cycle 0 and 8 at cycle 3. `captured_events`
publishes `false`, `true`, `false`, `false`, `true`, `true`, and
`captured_presence` `false` and then `true`. The failure at cycle 5 equals
that at cycle 4 as a value and still ticks; the success at cycle 3 neither
clears the error nor publishes it again. Each failure's `context` names
`required`'s node, the phase `"evaluate"`, and that cycle's evaluation time.
Deliberate rejections: `capture(required, value)` in a `const fn` or in a
`when`; `capture(checked, 1)`, a call evaluated while the graph is built;
and `.value` of the capture of a callable that declares no result.
Conformance planned.

## Checked-division propagation

Accepted temporal declaration fragments, with an admitted f64 input
publication; `error` is a member of the `zero` option's enum:

```hgl
fn checked_data(value: f64) -> result<f64, error> {
    when { return value /{zero: .error} 0.0 }
}

fn propagated(value: f64) -> f64 {
    when { return value /{zero: .error} 0.0 }
}
```

`checked_data` publishes an ordinary Err carrying `numeric.division_by_zero`;
it does not fail the node and cannot satisfy an escaping-error assertion.
In `propagated` the same division is given where an `f64` is expected and
takes that form (ERR-15): the error is raised, and the node fails under
ERR-5, with no successful output publication. This is the catalogued
execution code's condition. Core owns the
[numeric observations](https://github.com/hhenson/hgl_core/blob/main/spec/numeric-operators.md);
executable conformance is planned.

## Nullable value refinement

- **ERR-12.** A `T?` value given where a T is expected is read as its
  value: the place tries it, and a null fails the read with the code
  `value.null`, an execution failure raised as a result's failure is
  (ERR-5). A connection of `T?` given where a connection of T is taken, a
  node's input, a lifted parameter (FN-11), or a condition in composition
  (CTRL-13), is read so too, the node that reads it failing in a cycle
  that publishes a null, whether or not a handler of it would have read
  that input, as a node given a failed result does (ERR-15). An immutable nullable parameter or local
  may be tested directly against null: that same binding is a T on paths
  proving a non-null value, including after a terminating null branch, and
  its read there does not fail. A `null` written where a T is expected
  rejects (TYPE-14). A null test establishes value presence, not temporal input
  validity.

The accepted `value_or_zero` example in [types and values](types-and-values.md)
uses this refinement. Accepted fragments, reads that may fail:

```hgl
const fn unchecked(value: f64?) -> f64 => value

const fn halved(value: f64) -> f64 => value / 2.0

fn relayed(value: f64?) -> f64 => halved(value)
```

The return of `unchecked` expects an `f64` and is given an `f64?`: it
reads the value, and fails with `value.null` where `value` is null.
`relayed` gives a connection of `f64?` to the lifted parameter of `halved`,
an `f64`: over the publications 3.0 and null it publishes 1.5, and then
its node fails with `value.null`. Deliberate rejection:
`const fn nothing() -> f64 => null`, a `null` written where an `f64` is
expected. Executable source conformance is planned.

## Core boundary

Core supplies failure channels, owned diagnostic data, runtime capture, and
cleanup behavior. Source checking and code/location contracts belong here.
