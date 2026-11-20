# Tests and eval

Status: the eval rules, TST-5 to TST-9, what a `const fn` is given for
a trace, TST-16, and integration tests, TST-17, are agreed. The error
expectation contracts retain their status below.

## Grammar

```ebnf
test_decl = "test", IDENTIFIER, block
          | [ timeout_annotation, NL ], "integration_test", IDENTIFIER, block ;
timeout_annotation = "@", "timeout", "(", TEMPORAL_LITERAL, ")" ;
test_context = "test", "{", [ NL ],
               [ test_item, { NL, test_item }, [ NL ] ], "}" ;
test_item = function_decl | struct_decl | enum_decl | test_decl ;
assert_statement = "assert", ( expression | raises_assertion ) ;
raises_assertion = "raises", "(", expression, ")", block ;
eval_expression = "eval", "(", expression, [ ",", expression ], [ "," ], ")" ;
expect_error = "#", gap, "expect-error", "(", gap, error_category, gap, ",",
               gap, STRING_LITERAL, gap, ")", gap ;
gap = { ? U+0020 or U+0009 ? } ;
error_category = ? a source category of the diagnostic catalog ? ;
```

Target checking restricts eval's first expression to a call of a function;
grammar alone admits no arbitrary expression. The second, where written, is
a run configuration (RUN-1). `raises` is contextual immediately after assert.
`expect_error` gives the characters of one line comment, not a token sequence.

## Test scope and assertions

- **TST-1.** A named test sees module-private declarations. Unnamed contexts
  contribute private fn/const-fn helpers, unexported struct/enum types, and
  named tests to one module-wide test scope across all parts, independent of
  order. They create no helper namespace. Nested contexts, imports, operator
  and native declarations, instantiations, exports, and operator
  implementations are not admitted here.
- **TST-2.** Production cannot refer to test-only declarations. Artifacts omit
  their types, helpers, tests, and test-only lifts. A test-only helper or
  type takes no name of a production declaration or import (UNIT-12).
  Duplicate test names or roles reject.
- **TST-3.** A test is a composition-phase block with assertions. It cannot
  itself declare state/cache, inject, lifecycle hooks, when, yield, or return.
  Eval traces are test-local values. An eval of a target that has a result
  is a trace of that result's type, and source writes no type for it. A
  trace is bound with `let`, compared with `==` or `!=` against a trace
  written out or another eval, and given to a `const fn` as an argument.
  It is not a temporal argument outside an eval, is stored in no field, and
  no function returns one.
- **TST-4.** Assert requires a wiring-time bool. Harness comparison is intrinsic:
  compare presence, exact present data, and shape/length; overload candidates
  cannot change it. A mismatch identifies the first differing cycle/time and
  expected/observed data. Approximate float comparison is not implicit.

## Dense trace extent

- **TST-6.** Dense position i denotes the run's start plus i smallest
  steps, which core gives as a microsecond each. All input traces share that
  indexing.
  Input `_` adds no scheduled event. Leading and intervening placeholders
  locate later publications. Trailing placeholders have no effect. An input
  with only placeholders supplies an empty replay, just like `[]`. A dense
  result spans run start through its last recorded publication, with `_`
  in gaps. A recording with zero publications yields `[]`.

## Eval contract

- **TST-5.** Eval takes a call of a module function, written as any call is:
  `eval(target(value: [7, _, 9]))`. The call binds its arguments by the
  function's public positional/named rules, and its generic arguments as
  any call does. Temporal/value role selection follows FN-4; a
  selected value helper uses ordinary lifting. An operator is no target: a
  function wraps it, and that function's parameters give the traces their
  types. Const parameters receive configuration; temporal
  parameters receive contextually typed publication traces. A trace is an
  argument of that call and of no call inside it. A second argument is the
  run's configuration, a `RunConfig` (RUN-1). A trace written
  out is dense, `[7, _, 9]`, or timed, `[0s: 7, 2s: 9]`. Each cell is one
  cycle's publication, written as a `return` is (TIME-7), with T its
  parameter's type for an input and the target's result type for an
  expectation: a T, published whole, or a `delta<T>` (TYPE-16), published
  as that change. A list, a tuple, a struct and a dictionary are each
  written either way: `Quote(bid: 1.0, ask: 2.0)` or
  `delta<Quote>(ask: 3.0)`. Where T publishes whole, as a scalar does, the
  two are one: for a nullable scalar a cell written as a value is that
  value, and one written `null` a published null, as `return null` is.
  `_` is no publication. A cell that is a delta naming no change is a
  tick only where it makes the value valid, as such a `return` is
  (TIME-7): given to a valid input it wakes no reader, and where the
  result is valid a cycle that writes only such a delta is `_` in the
  expectation. An expectation's cell written as a `delta<T>` is compared
  with the delta its cycle published, and one written as a T with the
  value the result holds once its cycle has published.
- **TST-7.** Timed traces use the sequence-of-pairs syntax in EXPR-9; a
  one-entry trace is still a sequence. Timed entries use duration offsets from
  run start or absolute datetime keys. Keys strictly increase; no `_` occurs
  in timed traces. Within one eval all inputs and expectation use the same
  dense/timed style. Sparse expectations enumerate exactly the actual output
  ticks and times.
- **TST-8.** Eval is a run (RUN-4) of an entry made for it: core's replay for
  each temporal input, then the selected target, whose result is the entry's.
  The run is a simulation (RUN-3) under the eval's configuration: with none
  written, from the earliest start until nothing is scheduled. Its output is
  the eval's value. An outputless target has no
  recording or comparable result. Its eval stands as a statement and has no
  value: binding or comparing it rejects. Each invocation is a fresh run with
  independent retained captures. An empty recording is valid; missing
  recording or failure is never successful emptiness. Test imports/overloads
  cannot replace core's replay or record.
- **TST-9.** Scalar false/zero/empty values are present publications. A
  structural input cell is published as TST-5 has it: a T whole, which
  replaces what the input held, and a `delta<T>` as that change. Validate
  ordinary arguments and profile admission before graph start. A
  constructor failure retains its own code; it is not renamed as a profile error.

- **TST-16.** A trace given to a `const fn` is an ordinary value there, the
  output of the eval's run (RUN-1): for a result type T a
  `list<tuple<datetime, delta<T>>>`, one tuple for each publication, and
  for each exit of a window (TYPE-11), in time order. A tuple holds the
  instant of the publication and what the target's result published
  then, a `delta<T>` (TYPE-16), which is T itself for a scalar and V for
  an `atomic<V>`: a `list<tuple<datetime, i64>>` for an `i64` result. A T
  published whole is held as its delta too, as
  `delta_value` reads it (TIME-9). Nothing published is no entry, and a
  published null is the delta that holds one. The trace is an eval's,
  bound or not. A sequence written where a `const fn` takes an argument
  is the list it is written as: `ticks([(@2026-01-01T00:00:00Z, 7)])`
  gives `ticks` one tuple. The parameter is written with that type, and
  what the function returns is ordinary data.
- **TST-17.** An integration test, `integration_test name { ... }`, is a
  named test that may run in real time: beside `eval` it writes
  `run(call, config)` (RUN-3), whose value is the run's output as an eval's
  is. It is for what a simulation cannot reach: the host clock, adaptors,
  and outside events. In a test's block `run` as the callee of a call is
  that form, whatever else is in scope (LEX-10); in a test that is no
  integration test it rejects. An integration test is run only when it is
  asked for; a run of a module's tests leaves it out. It is time-bound: it
  runs within a time limit by the host clock, the duration `@timeout(5s)`
  gives on the line before it, or one minute where none is written. One
  still running when its limit passes is stopped, and has failed (TST-11).
  The annotation's literal is a duration, and `timeout` is a name with that
  meaning only there (LEX-10). In every other
  respect it is a test: its name is one of its module's test names (TST-2),
  and its block is a test's (TST-3, TST-4).

Accepted illustration, complete module:

```hgl
module examples.tests

fn pass_through_i64(value: i64) -> i64 {
    when { return delta_value(value) }
}

test repeated_ticks {
    assert eval(pass_through_i64(value: [7, _, 7, 9])) == [7, _, 7, 9]
    assert eval(pass_through_i64(value: [_, _, _])) == []
    assert eval(pass_through_i64(value: [_, 7, _, _])) == [_, 7]
    assert eval(pass_through_i64(value: [])) == []
}

test bound_and_timed {
    let seen = eval(pass_through_i64(value: [7, _, 7, 9]))
    assert seen == [7, _, 7, 9]
    assert eval(pass_through_i64(value: [0s: 7, 2s: 9])) == [0s: 7, 2s: 9]
}

const fn ticks(seen: list<tuple<datetime, i64>>) -> i64 {
    var count = 0
    for tick in seen { count += 1 }
    return count
}

test counted {
    assert ticks(eval(pass_through_i64(value: [7, _, 7, 9]))) == 3
}

struct Quote {
    bid: f64
    ask: f64
}

fn relayed(quote: Quote) -> Quote {
    when all_valid(quote) { return quote }
}

test whole_and_delta_cells {
    assert eval(relayed(quote: [
        Quote(bid: 1.0, ask: 2.0),
        delta<Quote>(ask: 3.0),
    ])) == [Quote(bid: 1.0, ask: 2.0), Quote(bid: 1.0, ask: 3.0)]
}

fn beats(const every: duration) -> i64 {
    state count: i64 = 0
    while {
        count += 1
        yield every: count
    }
}

test bounded {
    assert ticks(eval(beats(every: 1s), RunConfig(length: 4s))) == 3
}

@timeout(5s)
integration_test beats_live {
    let seen = run(beats(every: 10ms), RunConfig(length: 1s))
    assert ticks(seen) > 0
}
```

The target publishes its own output; a composition identity would not exercise
compute publication. `ticks` is given the trace as a list with a tuple for
each of its three publications, and none for the cycle in which nothing was
published (TST-16). In `whole_and_delta_cells` the first input cell
publishes a quote whole and the second its ask alone; `relayed` publishes
the whole quote each time, and each expectation cell, written whole, is
compared with the quote the result holds after its cycle (TST-5).
`beats` never stops of itself, so each run of it is
given a length: `bounded` evaluates four seconds of it in simulation, and
`beats_live` runs it for a second of the host clock, when it is asked for
(TST-17), and has failed if it is still running after five. Conformance
source is planned, not supplied by this example.

Deliberate source rejections: `eval(7)`, `eval(pass_through_i64)`, and
`eval(add_(lhs: [7], rhs: [9]))`, whose first argument is no call of a
function (`eval.target`); `let seen = eval(sink(value: [7]))` with
an outputless `sink` (`eval.no_result`); `[0s: 7, 2s: _]` and a `_` outside
a trace (`trace.placeholder`); `eval(pass_through_i64(value: [7, 9])) ==
[0s: 7]`, a dense input with a timed expectation (`trace.style`);
`[2s: 7, 1s: 9]` (`trace.key_order`); `Kept(values: seen)`, a trace
stored in a field (`trace.escape`); `ticks([7, 9])`, a list of `i64`
where a list of tuples is taken (`type.mismatch`); and `@timeout(5s)` before
a `test` that is no integration test, and `@timeout(5)`, whose argument is no
duration (`syntax.expected_token`).

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

- **TST-12.** `# expect-error(category, "code")` is a standalone annotation: a
  line comment of the form `expect_error`, alone on its line. It applies to
  the next physical source line that is not itself an annotation, so several
  may precede one line. A line comment that begins `expect-error` after `#`
  and any spaces or tabs, without that form or not alone on its line, is
  invalid metadata (TST-14). The code is read as a string literal and, with
  its category, must be a row of the diagnostic catalog. The enclosing named
  test/declaration/import owns the whole rejection case: the one whose lines
  include the annotated line, a helper or test of an unnamed context owning
  its own lines. Reliable source boundaries are required: an item that shares
  a line with another item, a module line, or a context's brace owns no case.
  Unnamed context/module headers cannot own a case, nor can a blank or
  comment line. The file's last line, when no item's lines include it,
  belongs to the item that ends the file, if one does. An annotation that
  applies to a line with no owner is invalid metadata.
- **TST-13.** Exclude all rejection owners from executable source. Check each
  selected owner independently alongside the surviving module; no stubs or
  placeholder overloads are inserted. Every primary diagnostic must match one
  annotation exactly by category, code, file, and next-line location; every
  annotation must match once. A warning rejects nothing (ERR-1) and is not
  matched. Use the primary range's first character and the EOF position's line
  for EOF errors. Notes/messages/order are not match keys. A lexical or syntax
  error belongs to the owner it is inside: within the owner's text, or
  directly after its last token for something missing there. An end-of-file
  error belongs to the item that ends the file, and to no item when an
  unnamed context or text that fits no declaration ends it. Every other
  lexical or syntax error is the surviving source's (TST-14). Text that fits
  no declaration, after an item on its line, is not inside the item; comments
  after an item that the end of the file cuts short are. Every other error
  that checking an owner alongside the surviving module reports is that
  case's, wherever it is located.
- **TST-14.** Invalid metadata, ambiguous boundaries, or surviving-source errors
  prevent execution. A rejection mismatch fails that case while other valid
  cases continue. Named selectors include rejection tests; declaration-owned
  cases always run. A selector is a test's name, alone or as
  `module.path::name`; a bare name selects that test in every module.
  Unknown selectors reject. Report executed/rejected counts
  separately; omitted cases, empty output, crashes, and infrastructure failures
  cannot prove conformance. Other commands treat annotations as comments.
- **TST-15.** Success requires every selected executable and required rejection
  case to pass. A rejection-only run is valid. A run with no cases succeeds
  and supplies no conformance evidence. Report each selected case's identity,
  kind, and outcome;
  declaration-owned identities include original file and owner-start line.

Proposed/reviewed rejection illustration, complete module:

```hgl
module examples.rejection

# expect-error(type, "rolling_window.size_kind")
fn invalid(value: rolling_window<f64, 5m, 3>) { when {} }

test survives { assert true }
```

The function is checked for rejection; survives runs. The expected error is
located at the mixed-kind minimum. No executable conformance is claimed here.

## Core boundary

Core owns portable runtime cases, fixtures, replay/record, and host observations.
The compiler owns source admission and the runner. Language-only fixture
packaging is tracked outside this document.
