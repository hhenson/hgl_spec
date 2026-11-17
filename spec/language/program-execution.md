# Program execution

Status: RUN-1 to RUN-4 are agreed. RUN-5 is reviewed. Tool-specific
configuration formats are outside this language contract.

## Grammar

```ebnf
run_statement = run_expression | eval_expression ;
run_expression = "run", "(", expression, [ ",", expression ], [ "," ], ")" ;
```

`eval_expression` is in [tests and eval](tests-and-eval.md). The first
expression of each is a call, and the second a run configuration.

## Rules

- **RUN-1.** A module describes computations. An entry is a temporal function
  with no temporal parameters. An application (UNIT-14) runs one: its run
  statement is `run(call, config)` or `eval(call, config)`, where the call
  is of an entry, written as any call is and giving the entry's const
  arguments, and the configuration is a value of core's `RunConfig`, or
  `RunConfig()` where none is written. The entry's result, if any, is the
  run's output: for a result type T a `list<tuple<datetime, delta<T>>>`, one
  tuple for each publication, in time order (TST-16).
- **RUN-2.** Resolve and type-check the entry's arguments, the
  configuration, required interfaces, and selected native providers before
  starting the graph. What starts an application may replace an argument of
  its entry or a field of its configuration with a constant checked against
  the type it replaces, and what the application writes is the default. It
  does not change the word of the run statement. Missing/unknown
  configuration and incompatible providers fail admission.
- **RUN-3.** The word of a run statement is its mode. `eval` evaluates the
  call in simulation: logical evaluation time, each cycle without waiting,
  until no scheduled work remains or the configuration's end. `run` runs it
  in real time: it follows the host clock and admits outside events, from
  the present or the configuration's start, until the configuration's end
  or until its host stops it. Start is inclusive, end exclusive. `eval` is
  written in a test and in an application; `run` in an application. Core's
  execution contract gives the earliest start, the smallest step between two
  cycles, and what a start or an end left out means; a source file does not
  read the host clock to choose its own semantics.
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

eval(heartbeat(every: 500ms), RunConfig(end: earliest_start + 3s))
```

```hgl
application examples.entry.live

use examples.entry::{heartbeat}

run(heartbeat())
```

`backtest` evaluates the entry in simulation from the earliest start for
three seconds, and its output is the run's. `live` runs the same entry in
real time from the present until its host stops it. What starts either may
give `every` another value (RUN-2). The timing of a `yield` remains
proposed. Each yield captures its payload before waiting, so its datetime
is the earlier evaluation time, not the future publication time. This
distinction is an observable source rule, not an implementation detail.
Conformance planned.

Deliberate rejections: `run(heartbeat(every: 1))`, an entry given an `i64`
for a `duration`; `run(scaled(price: [1.0]))`, a call of a function with a
temporal input, which is no entry (`run.entry`); an application with no run
statement, or a second one (`application.run_count`); a run statement in a
module (`run.position`); `export fn` in an application
(`application.export`); and `use examples.entry.live::{heartbeat}`, an
import of an application (`application.import`).

## Core boundary

Core specifies the run: its configuration type, bounds, clocks, lifecycle
order, store, output, outside-event delivery, and stop/failure mechanics.
Compiler/runner documentation specifies CLI options, configuration-file
syntax, packaging, and presentation of entry output.
