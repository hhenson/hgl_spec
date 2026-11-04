# Tests and eval

Status: proposed test/eval baseline. Structured execution-error and source-error
expectations retain the contracts below; proposed scopes/metadata add no rules.

## Grammar

```ebnf
test_decl = "test", IDENTIFIER, block ;
test_context = "test", "{", [ NL ],
               [ test_item, { NL, test_item }, [ NL ] ], "}" ;
test_item = function_decl | struct_decl | enum_decl | test_decl ;
assert_statement = "assert", ( expression | raises_assertion ) ;
raises_assertion = "raises", "(", expression, ")", block ;
eval_expression = "eval", "(", expression, { ",", argument }, [ "," ], ")" ;
```

Target checking restricts eval's first expression; grammar alone admits no
arbitrary callable value. `raises` is contextual immediately after assert.

## Test scope and assertions

- **TST-1.** A named test sees module-private declarations. Unnamed contexts
  contribute private fn/const-fn helpers, unexported struct/enum types, and
  named tests to one module-wide test scope across all parts, independent of
  order. They create no helper namespace. Nested contexts, imports, native
  declarations, exports, and operator implementations are not admitted here.
- **TST-2.** Production cannot refer to test-only declarations. Artifacts omit
  their types, helpers, tests, and test-only lifts. A helper may shadow a
  production function without changing production lookup; a test-local type
  cannot shadow a production type/import. Duplicate test names or roles reject.
- **TST-3.** A test is a composition-phase block with assertions. It cannot
  itself declare state/cache, inject, lifecycle hooks, when, yield, or return.
  Eval traces are test-local values: they may be bound and compared, not stored
  as temporal arguments or passed through arbitrary ordinary functions.
- **TST-4.** Assert requires a wiring-time bool. Harness comparison is intrinsic:
  compare presence, exact present data, and shape/length; overload candidates
  cannot change it. A mismatch identifies the first differing cycle/time and
  expected/observed data. Approximate float comparison is not implicit.

## Proposed eval contract

- **TST-5.** Eval selects a module function, binding arguments by its public
  positional/named rules. Temporal/value role selection follows FN-4; a
  selected value helper uses ordinary lifting. An operator is wrapped in a
  function for this profile. Const parameters receive configuration; temporal
  parameters receive contextually typed publication traces.
- **TST-6.** Dense position i denotes start plus i minimum engine steps. `_`
  means no publication. The horizon is the longest temporal input length,
  including silent positions. Materialize through the later of that horizon
  and the last output tick; silent positions remain `_`. An empty supplied
  temporal input has length zero, distinct from source-only eval.
- **TST-7.** Timed traces use the sequence-of-pairs syntax in EXPR-9; a
  one-entry trace is still a sequence. Timed entries use duration offsets from
  run start or absolute datetime keys. Keys strictly increase; no `_` occurs
  in timed traces. Within one eval all inputs and expectation use the same
  dense/timed style. Sparse expectations enumerate exactly the actual output
  ticks and times.
- **TST-8.** Eval constructs core replay, the selected target, and core record;
  an outputless target has no recorder or comparable result. Each invocation
  has fresh run state and independent retained captures. An initialized empty
  recording is valid; missing recording or failure is never successful emptiness.
  Test imports/overloads cannot replace designated core replay/record.
- **TST-9.** Scalar false/zero/empty values are present publications. Structural
  input cells follow the admitted delta profile rather than a held snapshot.
  Validate ordinary arguments and profile admission before graph start. A
  constructor failure retains its own code; it is not renamed as a profile error.

Proposed illustration, complete module with an agreed compute fixture form:

```hgl
module examples.test

fn pass_through_i64(value: i64) -> i64 {
    when { return delta_value(value) }
}

test repeated_ticks {
    assert eval(pass_through_i64, value: [7, _, 7, 9]) == [7, _, 7, 9]
    assert eval(pass_through_i64, value: [_, _, _]) == [_, _, _]
    assert eval(pass_through_i64, value: []) == []
}
```

The target publishes its own output; a composition identity would not exercise
compute publication. Conformance source is planned, not supplied by this example.

## Execution-error expectations

- **TST-10.** `assert raises("code")` requires one literal catalogued execution
  code. Run its block once. A matching escaping error passes only after normal
  cleanup succeeds; then continue after the assertion. Normal completion, a
  different code, or failed cleanup fails the test. Preserve earlier effects.
  Captured errors and returned Err are ordinary data, not escaping failures.
- **TST-11.** Assertion/source/build failures, crashes, timeouts, and errors
  raised only during cleanup cannot satisfy raises. A failed nested assertion
  cannot satisfy an enclosing raises. The block has ordinary lexical test
  scope and permits no runtime declarations or special exception mechanism.

## Source-error expectations

- **TST-12.** `# expect-error(category, "code")` is a standalone annotation on
  the next physical source line. Category/code must be catalogued. The enclosing
  named test/declaration/import owns the whole rejection case; reliable source
  boundaries are required. Unnamed context/module headers cannot own a case.
- **TST-13.** Exclude all rejection owners from executable source. Check each
  selected owner independently alongside the surviving module; no stubs or
  placeholder overloads are inserted. Every primary diagnostic must match one
  annotation exactly by category, code, file, and next-line location; every
  annotation must match once. Use the primary range's first character and the
  EOF position's line for EOF errors. Notes/messages/order are not match keys.
- **TST-14.** Invalid metadata, ambiguous boundaries, or surviving-source errors
  prevent execution. A rejection mismatch fails that case while other valid
  cases continue. Named selectors include rejection tests; declaration-owned
  cases always run. Unknown selectors reject. Report executed/rejected counts
  separately; omitted cases, empty output, crashes, and infrastructure failures
  cannot prove conformance. Other commands treat annotations as comments.
- **TST-15.** Success requires every selected executable and required rejection
  case to pass. A rejection-only run is valid. A run with no cases supplies no
  conformance evidence. Report each selected case's identity, kind, and outcome;
  declaration-owned identities include original file and owner-start line.

Proposed/reviewed rejection illustration, complete module:

```hgl
module examples.rejection

# expect-error(type, "rolling.size_kind")
fn invalid(value: rolling<f64, 5m, 3>) { when {} }

test survives { assert true }
```

The function is checked for rejection; survives runs. The expected error is
located at the mixed-kind minimum. No executable conformance is claimed here.

## Core boundary

Core owns portable runtime cases, fixtures, replay/record, and host observations.
The compiler owns source admission and the runner. Language-only fixture
packaging is tracked outside this document.
