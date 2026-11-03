# Functions and phases

Status: agreed function roles, classification, defaults, and lifting.

## Grammar

```ebnf
function_decl = { warning_annotation, NL },
                ( [ "export" | "impl" ], "fn" | "const", "fn" ),
                callable_name, [ generic_parameters ], function_signature,
                [ requires_clause ], function_body ;
function_signature = "(", [ parameters ], ")", [ "->", type ] ;
parameters = parameter, { ",", parameter }, [ "," ] ;
parameter = IDENTIFIER, ":", type
          | "const", IDENTIFIER, ":", value_type, [ "=", const_expression ]
          | IDENTIFIER, ":", value_type, "=", const_expression
          | IDENTIFIER, ":", "...", type
          | IDENTIFIER, ":", "...", "{", type, "}" ;
function_body = "=>", expression | block ;
function_expr = "fn", "(", [ lambda_parameters ], ")", [ "->", type ],
                ( "=>", expression | block ) ;
lambda_parameters = lambda_parameter, { ",", lambda_parameter }, [ "," ] ;
lambda_parameter = IDENTIFIER, [ ":", type ] ;
```

Parameter role is checked from the containing declaration. The production
does not admit defaults on ordinary temporal parameters or export on const fn.
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
- **FN-5.** `const(name)` explicitly selects the value definition. It neither
  calls the function nor constructs a node. For an ordinary function named
  `const`, the selector applies only when its argument names a function.
- **FN-6.** A selected value helper with temporal arguments in composition is
  lifted: temporal arguments become inputs; scalar/default arguments remain
  configuration. Any input notification activates the node; all required inputs
  must be valid. Each admitted evaluation publishes its result, including an
  equal result. An outputless helper lifts to a sink. All-scalar calls execute
  directly and invent no source timing. Ordinary validity is not `all_valid`.
- **FN-7.** A composition return selects a connection; a runtime return
  publishes and leaves the current handler/evaluation. Completing a runtime
  handler without a write produces no tick. A value return yields an ordinary
  value. Anonymous functions obtain their parameter/result context from their
  accepting form; inference does not create a general callable value type.
- **FN-8.** Packs capture positional or named arguments in order. Positional
  packs may be homogeneous or use a declared type pack; named packs retain
  source names. Packs have no defaults, cannot be const, and follow all fixed
  parameters. Cardinality and reflection follow the generic chapter.
- **FN-9.** Selected value-helper capabilities propagate transitively to the
  caller without adding parameters or activation inputs. Injection does not
  turn a value function into a node. Missing or phase-forbidden capabilities
  reject the call. Constructed nodes own their own output and scheduler.

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

## Core boundary

Core supplies node admission, capabilities, output and source facilities.
Language classification determines which facilities a declaration requires.
