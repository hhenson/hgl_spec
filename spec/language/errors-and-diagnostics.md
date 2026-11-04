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
  This introduces no general nullable annotation or null-return suppression.
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
| parse | `syntax.expected_token` | Required token missing; unexpected token or EOF |
| type | `rolling.size_kind` | Wrong/mixed size kind; offending size |
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

Accepted temporal declaration fragments, with the core `error` policy value
in scope and an admitted f64 input publication:

```hgl
fn checked_data(value: f64) -> result<f64, error> {
    when { return value /{zero: error} 0.0 }
}

fn propagated(value: f64) -> f64 {
    when { return (value /{zero: error} 0.0)? }
}
```

`checked_data` publishes an ordinary Err carrying `numeric.division_by_zero`;
it does not fail the node and cannot satisfy an escaping-error assertion.
In `propagated`, explicit `?` turns the same error into node failure under
ERR-5, with no successful output publication. This is the catalogued execution
code's condition. Core owns the [numeric observations](https://github.com/hhenson/hgl_core/blob/main/spec/numeric-operators.md);
executable conformance is planned.

## Core boundary

Core supplies failure channels, owned diagnostic data, runtime capture, and
cleanup behavior. Source checking and code/location contracts belong here.
