# Temporal model

Status: agreed publication, handler, state/cache, and capability baseline.
Source alarm and run-wide keyed storage extensions remain proposed.

## Grammar

```ebnf
state_decl = "state", IDENTIFIER, [ ":", value_type ], "=", expression ;
cache_decl = "cache", IDENTIFIER, [ ":", value_type ], "=", expression ;
inject_decl = "inject", IDENTIFIER, { ",", IDENTIFIER }, [ "," ] ;
lifecycle_block = ( "start" | "stop" ), block ;
when_statement = "when", [ expression ], block ;
```

CTRL-9 places the opening brace of a `when` body.

## Rules

```mermaid
flowchart LR
    C[Composition: typed connections] --> G[Independent graph instance]
    G --> E[Evaluation: publications and effects]
    V[Value helpers] --> C
    V --> E
```

- **TIME-1.** A temporal endpoint has validity, modification at the current
  evaluation time, retained data, and publication-delta observations. Silence
  is not a value. An equal publication may still be a new tick. A payload read
  requires admission/readability; metadata may observe an invalid endpoint.
- **TIME-2.** Composition describes a typed graph without reading live
  payloads. Evaluation propagates changes in dependency order, at most once
  per node per cycle. Input activity controls notification; required validity
  controls admission. Mutation of output and mutation of configuration are
  different effects.
- **TIME-3.** Function-level `when` conditions are checked in source order.
  The first satisfied condition runs its body first. If that body completes
  without returning, checking continues with the next condition. A `return`
  exits the function's current evaluation; later handlers are not reached
  (FN-7). In a `when` header, `modified()` subscribes to all
  temporal inputs; `modified(a, b)` subscribes to both named inputs, and
  similarly for more inputs. A notification from any subscribed input can
  wake the handler, and one from any other cannot. `modified([])` subscribes
  to no input: only a scheduler or an alarm wakes the handler. `valid()`
  requires all temporal inputs to be valid;
  `valid(a, b)` requires both named inputs, and similarly for more inputs.
  Omitting either selector uses the same default as its zero-argument form.
  `valid([])` requires no inputs to be valid. A selector sets subscription or admission
  when it stands alone in the header or appears as a top-level conjunct.
  Selectors under negation or disjunction leave the default in place.
  A `when` nested in another block is rejected.
- **TIME-4.** Aggregate subscriptions cover every handler. Node-wide validity
  includes only requirements common to executable handlers; other guards stay
  per-handler. A payload read under manual admission must be dominated by a
  successful guard. A guard is a test of `valid(input)`, on the input by its
  name, that holds where the read stands: in the branch it holds for, in the
  right operand of the `&&` it guards, and after a branch that returns, the
  paths a null test is followed along (ERR-12). A saved bool proves nothing.
  `valid` and `all_valid` retain distinct child-validity rules.
- **TIME-5.** State/cache/inject declarations precede executable hooks. At most
  one start and one stop are admitted. State initializers cannot read current
  temporal inputs. Start follows initialization; stop performs semantic
  finalization. Runtime ownership handles destruction.
- **TIME-6.** State is persistent, recordable node history. Initializers do not
  overwrite restored state. Cache is reconstructible node-local value data,
  initialized on every start using available restored state. Checkpoint support
  is an additional runtime facility, not a promise that every run persists.
  Arbitrary opaque resources are not ordinary state/cache values.
- **TIME-7.** A runtime return publishes to its result. `inject out` exposes that
  same result for reads, assignment, and admitted piecewise mutation. It adds
  no second output and is rejected for outputless functions. `out` is read
  in a `when` and in `stop`, and written only in a `when`: `start` neither
  reads nor writes it, and a `return` with a value stands in neither `start`
  nor `stop`. A node that publishes when the graph starts schedules itself
  in `start` and writes in its first scheduled evaluation. No write means no
  tick; whole-output invalidation is explicit and distinct from silence.
- **TIME-8.** The injectables are `out`, `logger`, `clock`, and `scheduler`,
  and the proposed `alarm` and `global_state` below; `inject` names no other.
  They are capabilities, not first-class payloads. Clock properties are
  read-only, including `clock.evaluation_time`. Operations use direct
  receiver-first forms such as `schedule(scheduler, delay)` and
  `info(logger, value)`, with no method-call aliases. Arbitrary helpers cannot
  receive a capability as ordinary data.
- **TIME-9.** The operations on an endpoint are intrinsics of the language:
  `valid`, `modified`, `delta_value`, `last_modified`, `invalidate`,
  `scheduled`, `activate`, and `passivate`. No HGL declaration takes an
  endpoint: a `const fn` takes its payload, and a temporal `fn` is not called
  in a hook. `scheduled()` is the scheduler handler selector, and `modified`
  and `valid` are selectors in a `when` header (TIME-3). Elsewhere each takes
  a temporal input of the enclosing function by its name, or `out`;
  `activate(input)` and `passivate(input)` operate on direct temporal inputs.
  Core's endpoint contract says what each returns. These reserved
  intrinsics cannot be shadowed. A value function has no capability, output,
  or schedule of its own (FN-9).
- **TIME-10.** A reference ticks on changed designation, not every target tick.
  Ordinary followers observe the selected target; saved routes do not extend
  an expired child lifetime. Structural child membership, child validity, and
  child publications are separate observations supplied by core.

Accepted illustration, complete module:

```hgl
module examples.handlers

fn manual_forward(value: i64) -> i64 {
    inject out
    when modified() && valid([]) {
        if valid(value) { return value }
        invalidate(out)
    }
}

fn guarded_forward(value: i64) -> i64 {
    inject out
    when modified() && valid(value) { return value }
    when modified() && valid([]) { invalidate(out) }
}
```

`modified()` subscribes to input notifications. `valid([])` lets the body check
validity itself. The two forms have the same observations: a valid input
publishes its value and an invalid input invalidates the output. In the guarded
form, the first body's return prevents the invalidation clause from running.
Deliberate rejections: replacing `if valid(value) { return value }` in
`manual_forward` with `return value + 1`, an unguarded payload read under
manual admission (`payload.unguarded`);
`valid(value + 1)`, which names no input (`endpoint.argument`); a `let`
beside the `when` (`node.statement`); and a `start` that reads or writes
`out` (`out.lifecycle`).
Conformance planned.

Accepted illustration, complete module, a node that publishes when the graph
starts (TIME-7):

```hgl
module examples.first_tick

fn counted(value: i64) -> i64 {
    state seen: i64 = 0
    inject scheduler
    start { schedule(scheduler, 0s) }
    when scheduled() && valid([]) { return seen }
    when modified() {
        seen += 1
        return seen
    }
}
```

`start` schedules the node and writes nothing. The first evaluation is the
scheduled one, which publishes 0: the graph is connected by then, and the
publication is a tick like any other. `valid([])` admits that handler before
`value` has published; it reads no input, so it needs no guard.
Conformance planned.

Accepted illustration, complete module, the selectors of a `when` header
(TIME-3):

```hgl
module examples.selectors

fn default_sum(a: f64, b: f64) -> f64 {
    when { return a + b }
}

fn explicit_sum(a: f64, b: f64) -> f64 {
    when modified(a, b) && valid(a, b) { return a + b }
}

fn on_a(a: f64, b: f64) -> f64 {
    when modified(a) { return a + b }
}

fn while_a_valid(a: f64, b: f64) -> f64 {
    when valid(a) { return a }
}

fn sum_or_zero(a: f64, b: f64) -> f64 {
    when valid([]) {
        if valid(a) && valid(b) { return a + b }
        return 0.0
    }
}

fn dormant(a: f64, b: f64) -> f64 {
    when modified([]) && valid([]) { return 0.0 }
}
```

With `a` publishing 1.0, nothing, 3.0 and `b` publishing nothing, 10.0, 20.0
over three cycles, each input keeping its value while it is silent:

| Function | Header | Publishes | Why |
| --- | --- | --- | --- |
| `default_sum`, `explicit_sum` | None; `modified(a, b) && valid(a, b)` | Nothing, 11.0, 23.0 | The two say the same: any input wakes it, and both are valid from the second cycle |
| `on_a` | `modified(a)` | Nothing, nothing, 23.0 | `b` cannot wake it; validity is the default, both inputs |
| `while_a_valid` | `valid(a)` | 1.0, 1.0, 3.0 | Subscription is the default, so `b` wakes it while `a` holds 1.0 |
| `sum_or_zero` | `valid([])` | 0.0, 11.0, 23.0 | Nothing is required of the inputs, and the body guards each read |
| `dormant` | `modified([]) && valid([])` | Nothing | No input wakes it and it has no scheduler: the warning `node.no_activation` |

Deliberate rejections: in `while_a_valid`, `return a + b`, a read of `b`
that neither the header requires nor a guard proves (`payload.unguarded`);
`modified([a, b])` and `valid([a, b])`, where the inputs are named as
arguments. Conformance planned.

## Proposed source and storage capabilities

`alarm` is source-only, exclusive with scheduler, and keeps the earliest
requested wake without recovery/query state. A plain when handles its wake.
The [generator form](control-flow-and-sources.md) owns its own wake mechanism.

`global_state` is a run-wide typed store with const string keys prepared before
root start. All uses/seeds bind the same exact ordinary type. Binding alone
does not initialize a value. Get of an absent entry fails; set preserves type.
Writable borrows are hook-scoped; retained results own independent values.
Storage access adds no activation dependency or temporal endpoint type.

## Core boundary

Core defines clock/scheduler interfaces, graph lifecycle, publication/delta
operations, storage, references, and child lifetime. This chapter fixes their
source interpretation and capability phases without prescribing a layout.
