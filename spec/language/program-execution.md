# Program execution

Status: the entry, and RUN-3 and RUN-4, are agreed. How a run is written
in source, the last sentence of RUN-1, is under review. RUN-2 and RUN-5 are
reviewed. Tool-specific configuration formats are outside this language
contract.

## Rules

- **RUN-1.** A module describes computations. An entry is an exported temporal
  function with no temporal parameters, and an application is an entry. Its
  const arguments are given with the run. Its result, if any, is the run's
  output: for a result type T a `list<tuple<datetime, delta<T>>>`, one tuple
  for each publication, in time order (TST-16). HGL has no special main
  declaration or in-source run-control expression.
- **RUN-2.** A unique entry can be selected by default; otherwise configuration
  identifies one exported entry. Resolve and type-check its arguments, required
  interfaces, and selected native providers before starting the graph.
  Missing/unknown configuration and incompatible providers fail admission.
- **RUN-3.** A run is given a run configuration, a value of core's
  `RunConfig`: a mode, a start, and an end. Simulation uses logical
  evaluation time; real-time execution follows the host clock and admits
  outside events. Start is inclusive, end exclusive. An idle simulation ends
  when no scheduled work remains. Core's execution contract gives the
  earliest start, the smallest step between two cycles, and what a start or
  an end left out means; a source file does not read the host clock to
  choose its own semantics.
- **RUN-4.** Each run has independent runtime-owned state, captures,
  capabilities, store, and lifecycle. Start and stop follow the required
  graph order; construction/start/evaluation/stop failures preserve their
  failure phase. Normal teardown is required after failure. End-of-run is not
  an extra tick. An application and an eval are run under this one contract.
- **RUN-5.** Interpretation and native compilation must preserve the same
  observable traces, effects, failures, and cleanup. Optimizations cannot invent
  publications, erase required equal ticks, or change capability lifetimes.
  No particular generated code, cache, file format, or tool invocation is required.

Proposed illustration, complete module using the reviewed entry model:

```hgl
module examples.entry

export fn heartbeat(const every: duration = 1s) -> datetime {
    inject clock
    while {
        yield every: clock.evaluation_time
    }
}
```

The entry relationship is reviewed; the timing of a `yield` remains proposed. Each
yield captures its payload before waiting, so its datetime is the earlier
evaluation time, not the future publication time. This distinction is an
observable source rule, not an implementation detail. Conformance planned.

Deliberate rejection: an entry supplied an incompatible const parameter value.
An ordinary exported fn with temporal inputs remains callable but is not an entry.

## Core boundary

Core specifies the run: its configuration type, bounds, clocks, lifecycle
order, store, output, outside-event delivery, and stop/failure mechanics.
Compiler/runner documentation specifies CLI options, configuration-file
syntax, packaging, and presentation of entry output.
