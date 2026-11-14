# Functions and phases

Status: agreed function roles, classification, defaults, lifting, and
anonymous-function scope.

## Grammar

```ebnf
function_decl = { warning_annotation, NL },
                ( [ "export" | "impl" ], "fn" | [ "export" ], "const", "fn" ),
                callable_name, [ generic_parameters ], function_signature,
                [ requires_clause ], function_body ;
function_signature = "(", [ parameters ], ")", [ "->", type ] ;
parameters = parameter, { ",", parameter }, [ "," ] ;
parameter = IDENTIFIER, ":", type
          | "const", IDENTIFIER, ":", value_type, [ "=", const_expression ]
          | IDENTIFIER, ":", value_type, "=", const_expression
          | IDENTIFIER, ":", "...", type, [ cardinality ]
          | IDENTIFIER, ":", "...", "{", type, "}", [ cardinality ] ;
function_body = "=>", expression | block ;
function_expr = "fn", "(", [ lambda_parameters ], ")", [ "->", type ],
                ( "=>", expression | block ) ;
lambda_parameters = lambda_parameter, { ",", lambda_parameter }, [ "," ] ;
lambda_parameter = IDENTIFIER, [ ":", type ] ;
```

Parameter role is checked from the containing declaration. The production
does not admit defaults on ordinary temporal parameters.
Local source/native declaration matching is defined in the native chapter.

## Rules

- **FN-1.** `const fn` computes ordinary values for one invocation. It is not
  inherently pure or compile-time-only. Parameter-level `const` instead fixes
  configuration. A temporal `fn` has temporal non-const parameters and a
  temporal result; an omitted result arrow means outputless.
- **FN-2.** An ordinary temporal body with no node-only construct describes
  composition. `state`, `cache`, `inject`, `start`, `when`, `stop`, or `yield`
  classifies that complete body as runtime. Classification precedes phase
  checking, even if the trigger occurs in an invalid nested position. Generic
  selection, target choice, and optimizer decisions cannot change the role.
- **FN-3.** Iteration and conditionals follow the containing phase. They do
  not alone classify a body as runtime. A composition body cannot read current
  input payloads. A runtime hook can call value helpers but cannot silently
  construct a temporal child call. Mixed incompatible phase work is rejected.
- **FN-4.** A temporal/value pair may share a name. Composition and eval choose
  the temporal definition if one exists; runtime/value evaluation chooses the
  value definition. A bad selected signature is an error, not fallback to the
  other role. Two definitions of the same exact role remain duplicates.
- **FN-5.** `const` is a modifier only in a position admitting that modifier.
  Otherwise it is an ordinary name, including for an operator or function.
  `const(argument)` is an ordinary call resolved by the usual naming and
  phase rules; it has no intrinsic function-role selection behavior.
- **FN-6.** A selected value helper with temporal arguments in composition is
  lifted: temporal arguments become inputs; scalar arguments and omitted
  defaults bind as configuration. A supplied temporal argument remains an
  input even when its parameter has a default. Any input notification activates
  the node; FN-11 defines admission and binding. Each admitted evaluation
  publishes its result, including an equal result. An outputless helper lifts
  to a sink. All-scalar calls execute
  directly and invent no source timing. Ordinary validity is not `all_valid`.
- **FN-7.** A composition return selects a connection; a runtime return
  publishes and leaves the current handler/evaluation. Completing a runtime
  handler without a write produces no tick. A value return yields an ordinary
  value. Anonymous functions obtain their parameter/result context from their
  accepting form; inference does not create a general callable value type.
- **FN-8.** Packs capture, in order, the arguments the fixed parameters leave.
  `...T` captures the positional ones: all of type T, or each of its own type
  when T is a declared type pack. `...{Fields}` captures the named ones, each
  under its source name, where `Fields` is a declared type pack. Packs have
  no defaults, cannot be const, and follow all fixed parameters. Cardinality
  and reflection follow the generic chapter.
- **FN-9.** Selected value-helper capabilities propagate transitively to the
  caller without adding parameters or activation inputs. Injection does not
  turn a value function into a node. Missing or phase-forbidden capabilities
  reject the call. Constructed nodes own their own output and scheduler.
- **FN-10.** An arrow body consumes the complete expression up to its enclosing
  delimiter or significant newline. A comma at the surrounding argument/list
  level ends the body; a comma in a nested expression does not. Existing
  newline-continuation rules still apply. Operators and calls within that
  expression belong to the body. Parenthesize an anonymous function before
  calling it or applying an operation to the function itself. These boundaries
  select the body extent rather than leaving two interpretations.
- **FN-11.** A lifted argument with a declared default does not require a valid
  input: bind the default when that input is invalid. Otherwise, an argument
  whose value contract admits null does not require a valid input: bind null
  when it is invalid. Every other input must satisfy ordinary `valid(input)`.
  A valid value, including a valid null, is supplied as published. Fallback
  changes admission/binding, not activation, and invents no input tick.
- **FN-12.** Aggregate lifting passes the current ordinary aggregate value to
  one helper invocation, not separate elementwise invocations. Use its type's
  ordinary validity contract. An unsized `list<T>` needs `valid(list_value)`;
  lifting adds no required size or recursive `all_valid` check. Fixed-size
  lists and windows retain their own shape/readiness contracts. Unchanged
  inputs supply retained values; they need not publish together.
- **FN-13.** Each invocation observes a stable ordinary argument value or an
  admitted read-only view. Retained results must remain independent of later
  temporal changes. Lifting does not turn a borrowed view into an owned value
  or extend its permitted lifetime.
- **FN-14.** An anonymous function's scope is the scope around it plus its
  parameters (UNIT-12). From the functions and blocks around it, its body
  uses only parameters, generic parameters, `let` locals, and iteration
  bindings. Use of an enclosing `var`, `state`, or `cache` rejects.

Accepted declaration fragments showing modifier and ordinary-name positions:

```hgl
const fn const(value: i64) -> i64 => value
const fn example() -> i64 => const(7)
```

The first `const` modifies each declaration; the declared name and call use
ordinary lookup. `example()` returns 7. Executable conformance is planned.

Accepted expression fragments in an accepting context supplying callable types:

```hgl
apply(fn(x) => x + 1, 7)
(fn(x) => x + 1)(7)
```

In the first, `apply` accepts a unary function and a value: the body ends at
the comma. In the second, the outer call invokes the grouped function and
returns 8 for integer input. Executable conformance is planned.

Accepted declaration fragment, with the same `apply`:

```hgl
const fn shifted(n: i64) -> i64 => apply(fn(x) => x + n, 7)
```

The body reads `n`, a parameter of `shifted` (FN-14); `shifted(1)` returns 8.
Deliberate source rejections: an anonymous-function parameter named `n` takes
a name already in scope (UNIT-12); a body reading a `var` local of the
enclosing function uses a binding FN-14 excludes. Executable conformance is
planned.

| Phase | A temporal argument denotes | Ordinary calls |
| --- | --- | --- |
| Composition | A typed connection | Describe graph work or lift a value helper |
| Runtime hook | A readable admitted payload plus endpoint metadata | Execute value helpers in the current node |
| Value function | Ordinary invocation data | Execute value helpers |

Proposed eval illustration, complete module with agreed function/lifting forms:

```hgl
module examples.functions

const fn scale(value: f64, factor: f64 = 2.0) -> f64 => value * factor
fn doubled(value: f64) -> f64 => scale(value)

test lifted_helper {
    assert eval(doubled, value: [1.0, _, 3.0]) == [2.0, _, 6.0]
}
```

The function/lifting relationship is agreed. The eval assertion illustrates the
proposed harness contract. Deliberate rejection: a temporal call placed inside
`when`. Conformance is planned.

Accepted FN-11 binding illustration using `scale` above, with temporal inputs
supplied for both parameters. Both endpoints start invalid; a dash denotes no
publication and does not invalidate a retained input:

| Cycle | Value publication | Factor publication | Arguments bound for the invocation |
| --- | --- | --- | --- |
| 0 | 1.0 | — | value 1.0, default factor 2.0; invalid factor does not block admission |
| 1 | — | 4.0 | retained value 1.0, published factor 4.0 |

The first invocation computes 2.0 and the second 4.0. Factor is a supplied
temporal argument, so its publication activates evaluation; it is not omitted
configuration. The ordinary all-required-inputs-valid rule would incorrectly
suppress cycle 0 here.

Accepted binding cases for a parameter whose admitted value contract permits
null, with a non-null default D of that contract when stated:

| Declaration contract | Input at an otherwise admitted notification | Bound argument |
| --- | --- | --- |
| Nullable, default D | Invalid | D |
| Nullable, default D | Valid null publication | null |
| Nullable, no default | Invalid | null |

An explicit declaration illustrating that nullable/default contract is:

```hgl
const fn nullable_echo(value: f64? = 2.0) -> f64? => value
```

Nullability uses TYPE-14's `T?` syntax. It is independent of the function-level
`const` marker. Core owns the [lifting observations](https://github.com/hhenson/hgl_core/blob/main/spec/value-function-lifting.md);
executable conformance is planned. The eval profile retains its proposed status.

Accepted FN-12 aggregate illustration, with a temporal unsized list supplied
to this value helper:

```hgl
const fn count_items(values: list<f64>) -> i64 => len(values)
```

The fixture below exposes list membership independently of child payload
validity. The helper reads length only; it never reads an invalid child's
payload. Each admitted row invokes the helper once with the whole list.

| Cycle | Temporal list state | Ordinary valid | all_valid | Invocation / result |
| --- | --- | --- | --- | --- |
| 0 | Invalid parent | false | false | No invocation |
| 1 | Length 2; first child valid, second invalid | true | false | `count_items(list_value)` returns 2 |
| 2 | Second child publishes; length remains 2 | true | true | One invocation returns 2 again |
| 3 | A valid third child joins | true | true | One invocation returns 3 |

Cycle 1 distinguishes ordinary validity from an added full-validity gate;
cycles 2 and 3 distinguish repeated publication and an unsized shape. This
is not an elementwise call to `count_items` with individual floats.

Accepted FN-13 retention illustration, with complete valid list inputs:

```hgl
const fn keep_value(values: list<i64>) -> list<i64> => values
```

| Event | Current input | Independently retained result |
| --- | --- | --- |
| First admitted invocation | `[1, 2]` | R0 is `[1, 2]` |
| Later publication and invocation | `[1, 7]` | R1 is `[1, 7]`; R0 remains `[1, 2]` |

An implementation may use a stable read-only view during the invocation, but
retaining the ordinary result preserves an owning value's independent contents.
Deliberate lifetime rejection, a bounded provider-interface case: a provider
declares a view valid only until the current helper returns; attempting to
retain that same borrowed view beyond the call, without obtaining an owning
value, rejects under FN-13 during lifetime checking. Retaining an admitted
independent owning copy instead is accepted. No new view type or copy API is
implied. Core owns the [concrete observations](https://github.com/hhenson/hgl_core/blob/main/spec/value-function-lifting.md);
executable conformance is planned.

## Core boundary

Core supplies node admission, capabilities, output and source facilities.
Language classification determines which facilities a declaration requires.
Core owns the [lifting observations](https://github.com/hhenson/hgl_core/blob/main/spec/value-function-lifting.md).
