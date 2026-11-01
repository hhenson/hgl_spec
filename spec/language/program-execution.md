# Program execution

Status: reviewed entry/run model. Tool-specific configuration formats are
outside this language contract.

## Rules

- **RUN-1.** A module describes computations. An entry is an exported temporal
  function with no temporal parameters. Its const parameters come from run
  configuration and its result, if any, is the run's output. HGL has no special
  main declaration or in-source run-control expression.
- **RUN-2.** A unique entry can be selected by default; otherwise configuration
  identifies one exported entry. Resolve and type-check its arguments, required
  interfaces, and selected native providers before starting the graph.
  Missing/unknown configuration and incompatible providers fail admission.
- **RUN-3.** Execution supplies a mode, clock, start, and optional end. Simulation
  uses logical evaluation time; real-time execution admits outside events.
  Start is inclusive, end exclusive. An idle simulation ends when no scheduled
  work remains. Run configuration and core's execution contract determine the
  origin and permitted bounds; a source file does not read the host clock to
  choose its own semantics.
- **RUN-4.** Each instantiation has independent runtime-owned state, captures,
  capabilities, and lifecycle. Start and stop follow the required graph order;
  construction/start/evaluation/stop failures preserve their failure phase.
  Normal teardown is required after failure. End-of-run is not an extra tick.
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

The entry relationship is reviewed; generator behavior remains proposed. Each
yield captures its payload before waiting, so its datetime is the earlier
evaluation time, not the future publication time. This distinction is an
observable source rule, not an implementation detail. Conformance planned.

Deliberate rejection: an entry supplied an incompatible const parameter value.
An ordinary exported fn with temporal inputs remains callable but is not an entry.

## Core boundary

Core specifies engine bounds, clocks, lifecycle order, outside-event delivery,
and stop/failure mechanics. Compiler/runner documentation specifies CLI options,
configuration-file syntax, packaging, and presentation of entry output.
