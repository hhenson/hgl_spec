# Program execution

Status: RUN-1 to RUN-4 and RUN-6 are agreed. RUN-5 is reviewed. Tool-specific
configuration formats are outside this language contract.

## Grammar

```ebnf
run_statement = run_expression | eval_expression ;
run_expression = "run", [ operator_configuration ], run_target ;
run_target = ( IDENTIFIER | qualified_name ), [ generic_arguments ],
             call_suffix ;
```

`eval_expression` is in [tests and eval](tests-and-eval.md),
`operator_configuration` and `call_suffix` in
[expressions](expressions-and-bindings.md), and `generic_arguments` in
[generics](generics-and-operators.md). The target is a call of a function
(RUN-1, TST-5), and the configuration, where one is written, the run's
(RUN-6). As an expression, `run_expression` is read in an integration
test's block and nowhere else (TST-17).

## Rules

- **RUN-1.** A module describes computations. An entry is a temporal function
  with no temporal parameters. An application (UNIT-14) runs one: its run
  statement is `run call` or `eval call`, where the call is of an entry,
  written as any call is and giving the entry's const arguments. The
  statement gives no configuration: the run's is supplied to it (RUN-6).
  The entry's result, if any, is the run's output: for a result type T a
  `list<tuple<datetime, delta<T>>>`, one tuple for each publication, in
  time order (TST-16).
- **RUN-2.** Resolve and type-check the entry's arguments, the
  configuration, required interfaces, and selected native providers before
  starting the graph. What starts an application supplies its run's
  configuration (RUN-6), and may replace an argument of its entry with a
  constant checked against the type it replaces: what the application
  writes is the default. It does not change the word of the run statement.
  Missing/unknown configuration and incompatible providers fail admission.
- **RUN-3.** The word of a run statement is its mode. `eval` evaluates the
  call in simulation: logical evaluation time, each cycle without waiting,
  until no scheduled work remains or the configuration's end. `run` runs it
  in real time: it follows the host clock and admits outside events, from
  the present or the configuration's start, until the configuration's end
  or until its host stops it. Start is inclusive, end exclusive. `eval` is
  written in a test and in an application; `run` in an application and in
  an integration test (TST-17). Core's
  execution contract gives the earliest start, the smallest step between two
  cycles, what a start or an end left out means, and how a configuration
  gives its end, as an instant or as a length from its start; a source file
  does not read the host clock to choose its own semantics.
- **RUN-4.** Each run has independent runtime-owned state, captures,
  capabilities, store, and lifecycle. Start and stop follow the required
  graph order; construction/start/evaluation/stop failures preserve their
  failure phase. Normal teardown is required after failure. End-of-run is not
  an extra tick. An application and a test's eval are run under this one
  contract.
- **RUN-5.** Interpretation and native compilation must preserve the same
  observable traces, effects, failures, and cleanup. Optimizations cannot invent
  publications, erase required equal ticks, or change capability lifetimes.
  No particular generated code, cache, file format, or tool invocation is required.
- **RUN-6.** A run has one configuration, a value of core's `RunConfig`.
  The program does not make it. `RunConfig` is the type of the capability
  `run_config` (TIME-8): no expression constructs one, no `let` holds one
  that it built, and no function returns one. A run statement in a test
  gives its run's configuration as an operator is given its own (EXPR-10),
  in braces after the word: `eval{start: a, end: b} call`. Each entry
  names a field of the configuration, and its value is value code of that
  field's type; a field no entry names is left out. An entry that names no
  field is an argument that names no parameter (`argument.unknown`). An
  application's run statement gives none: what starts the application
  supplies its run's configuration, and the application gives the fields
  that leaves out their defaults, `inject run_config{length: 3s}`, written
  the same way, among the statements before its run statement (UNIT-14).
  A field neither gives is left out. `run_config` reads the configuration
  as supplied, what starts the application over those defaults, and not
  as the run resolves it: a start left out is null there. It is read in the
  statements that follow its `inject`, where no other capability is
  (UNIT-14, TST-17). An application
  injects `run_config`, and so does an integration test (TST-17), whose
  runs' configuration what runs the test supplies. A test that is no
  integration test does not, and neither does a function: one that needs a
  bound of the run takes it as a `const` argument. An `end` or a `length`
  that what starts an application gives replaces a default of either. In
  an integration test that injects `run_config`, an entry of a run
  statement's configuration is that statement's own value for its field,
  and a field the statement does not name is the injected configuration's.
  `inject run_config` where it has no place rejects (`inject.position`),
  and so do braces on an `inject` of any other capability
  (`inject.configuration`) and a configuration on an application's run
  statement (`run.configuration`).

Accepted illustration, a module and two applications over its entry:

```hgl
module examples.entry

export fn heartbeat(const every: duration = 1s) -> datetime {
    inject clock
    while {
        yield every: clock.evaluation_time
    }
}
```

```hgl
application examples.entry.backtest

use examples.entry::{heartbeat}

inject run_config{length: 3s}

eval heartbeat(every: 500ms)
```

```hgl
application examples.entry.live

use examples.entry::{heartbeat}

run heartbeat()
```

`backtest` evaluates the entry in simulation, and its output is the run's:
from the earliest start for three seconds, where what starts it gives no
bound of its own. `live` runs the same entry in real time, from the
present until its host stops it where it is given no bound. What starts
either may give `every` another value (RUN-2), and gives the run its
configuration (RUN-6). The timing of a `yield` remains
proposed. Each yield captures its payload before waiting, so its datetime
is the earlier evaluation time, not the future publication time. This
distinction is an observable source rule, not an implementation detail.
Conformance planned.

Accepted illustration, a module and an application that sets the run's store
before its run (UNIT-14):

```hgl
module examples.limited

export fn counted(const every: duration = 1s) -> i64 {
    inject global_state
    cache count: i64 = 0
    while count < get<i64>(global_state, "limit")? {
        count += 1
        yield every: count
    }
}
```

```hgl
application examples.limited.three

use examples.limited::{counted}

inject run_config{start: @2026-01-05T09:30:00Z}
inject global_state
put(global_state, "limit", 3)
put(global_state, "opened", run_config.start)

eval counted()
```

The store holds `"limit"` when the graph is built, so `counted` reads 3 from
its first evaluation and the run's output is 1, 2 and 3, a second apart.
`"opened"` holds the start the run is given: the application's default,
or the one what starts it supplies (RUN-6).
With no `put` the first `get` fails the source: the key is absent, and `?`
asks for the value (ERR-15; proposed for a source's body, ERR-5).
Deliberate rejections in the application: `inject clock`;
`has(global_state, "limit")`, a call that is no `put`; a `put` after the run
statement; and the two store statements in a module
(`application.statement`). Conformance planned.

Deliberate rejections: `run heartbeat(every: 1)`, an entry given an `i64`
for a `duration`; `run scaled(price: [1.0])`, a call of a function with a
temporal input, which is no entry (`run.entry`); `RunConfig(length: 3s)`,
a call of a capability's type, which no call constructs
(`call.not_callable`); `inject run_config{until: 3s}`, an entry that names
no field (`argument.unknown`); `eval{length: 3s} heartbeat()` as an
application's run statement (`run.configuration`); `inject clock{step: 1s}`
in `heartbeat` (`inject.configuration`); `inject run_config` there
(`inject.position`); an application with no run
statement, or a second one (`application.run_count`); a run statement in a
module, or one that a declaration follows (`run.position`); `export fn` in an application
(`application.export`); `use examples.entry.live::{heartbeat}`, an
import of an application (`application.import`); and a second file headed
`application examples.entry.live`, or one headed `module
examples.entry.live` (`application.duplicate_path`).

## Core boundary

Core specifies the run: its configuration type, bounds, clocks, lifecycle
order, store, output, outside-event delivery, and stop/failure mechanics.
Compiler/runner documentation specifies CLI options, configuration-file
syntax, packaging, and presentation of entry output.
