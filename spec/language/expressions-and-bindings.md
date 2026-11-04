# Expressions and bindings

Status: agreed expression/binding baseline. Structural delta constructors and
recursive ordinary mutation retain their proposed status.

## Grammar

```ebnf
expression = or_expr ;
or_expr = and_expr, { "||", [ operator_configuration ], and_expr } ;
and_expr = equality_expr, { "&&", [ operator_configuration ], equality_expr } ;
equality_expr = comparison_expr,
                { ( "==" | "!=" ), [ operator_configuration ], comparison_expr } ;
comparison_expr = additive_expr,
                  { ( "<" | "<=" | ">" | ">=" | "<=>" ),
                    [ operator_configuration ], additive_expr } ;
additive_expr = multiplicative_expr,
                { ( "+" | "-" ), [ operator_configuration ], multiplicative_expr } ;
multiplicative_expr = unary_expr,
                      { ( "*" | "/" | "//" | "%" ),
                        [ operator_configuration ], unary_expr } ;
unary_expr = ( "-" | "!" ), [ operator_configuration ], unary_expr | postfix_expr ;
operator_configuration = "{", configuration_argument,
                         { ",", configuration_argument }, [ "," ], "}" ;
configuration_argument = member_name, ":", expression ;
postfix_expr = primary_expr, { call_suffix | index_suffix | member_suffix | "?" } ;
call_suffix = "(", [ arguments ], ")" ;
index_suffix = "[", expression, "]" ;
member_suffix = ".", member_name ;
primary_expr = literal | IDENTIFIER | qualified_name
             | "(", expression, ")" | tuple_literal | sequence_literal
             | typed_constructor | delta_constructor | type_call
             | function_expr | if_expression
             | eval_expression | block ;
arguments = argument, { ",", argument }, [ "," ] ;
argument = [ member_name, ":" ], expression ;
tuple_literal = "(", expression, ",", [ expression, { ",", expression }, [ "," ] ], ")" ;
sequence_literal = "[", [ sequence_entries ], "]" ;
sequence_entries = sequence_values | sequence_pairs ;
sequence_values = sequence_item, { ",", sequence_item }, [ "," ] ;
sequence_pairs = sequence_pair, { ",", sequence_pair }, [ "," ] ;
sequence_pair = sequence_item, ":", sequence_item ;
sequence_item = expression | "_" ;
typed_constructor = ( IDENTIFIER | qualified_name ), generic_arguments,
                    "(", [ arguments ], ")" ;
type_call = ( "str" | "bytes" | "any" ), "(", [ arguments ], ")" ;
local_decl = ( "let" | "var" ), IDENTIFIER, [ ":", type ], "=", expression
           | "var", IDENTIFIER, ":", type ;
assignment = place, assignment_operator, expression ;
place = IDENTIFIER, { index_suffix | member_suffix } ;
assignment_operator = "=" | "+=" | "-=" | "*=" | "/=" ;
const_expression = expression ;
```

Block/control/function/eval forms are defined in their owning chapters.
The sequence grammar admits contextual forms; semantic rules restrict their use.
`const(name)` uses ordinary call syntax and lookup, as specified by FN-5.
GEN-14 resolves generic construction versus comparison using declaration lookup.

## Rules

- **EXPR-1.** The grammar gives precedence from low `||` to high postfix forms.
  Repeated binary forms associate left; unary forms associate right. A chained
  comparison is not implicit mathematical conjunction. Parentheses select
  grouping; a tuple has a comma. Newline constraints follow source-text rules.
- **EXPR-2.** Evaluate ordinary call/constructor arguments once in written
  order. Positional arguments precede named ones; reject duplicates, unknown
  names, missing required arguments, and wrong types. Named arguments use
  the selected public callable's parameter names.
- **EXPR-3.** Initialization fixes a binding's value/connection category and
  type; an annotation checks that category rather than changing it. `let` is
  immutable; `var` may be assigned within the same category/type. An uninitialized
  typed var supplies no value or connection; reaching assignments must resolve
  consistently and every path reaching a read must assign it. An inner
  declaration remains local. A compound assignment reads its existing target.
- **EXPR-4.** Ordinary values and temporal connections remain distinct. A graph
  binding denotes a connection without reading a payload. Reassigning a graph
  var changes the described connection; it creates neither mutable per-tick
  scalar storage nor feedback. Earlier aliases/consumers keep their original
  connection. Scalar/connection reassignment across categories rejects.
  Persistent runtime history uses state/cache.
- **EXPR-5.** An expected type contextualizes literals and constructors but
  cannot erase structural/atomic boundaries or nominal identity. Numeric
  widening follows the admitted destination relationship, not arbitrary native
  coercion. A const expression uses the same syntax with required-constant
  semantic restrictions; it cannot inspect temporal values.
- **EXPR-6.** In value/runtime contexts, Boolean connectives short-circuit left
  to right. A valid/presence/tag guard must precede the protected payload read.
  Composition forms describe dependencies and select core identities; they do
  not read current payloads or inherit scalar execution order as topology.
- **EXPR-7.** Structural projection selects a child connection. Atomic
  projection computes from a whole payload. Dictionary temporal key selection
  may produce a reference designation; strict/safe ordinary dictionary reads
  follow [lookup rules](errors-and-diagnostics.md). REF access is never erased
  just because a member has a familiar name.
- **EXPR-8.** Ordinary tuples may contain runtime payload expressions. Harness
  `_` and timed cells exist only in test traces; outside them a sequence is
  ordinary data and a placeholder is not a value. Tuple syntax does not
  construct graph connections from separate temporal endpoints.
- **EXPR-9.** Brackets always delimit a sequence. Entries are either all values
  or all colon-separated pairs; mixing these forms is a source error. `[a]`
  has one value entry; `[a: b]` has one pair entry, not a standalone pair value.
  Empty brackets have no entries. The use site and expected type constrain
  element types, keys, ordering, uniqueness, and placeholder admission; syntax
  alone does not select a dictionary, timed trace, or sparse delta. `_` is not
  an ordinary value and remains restricted by EXPR-8 and the trace profile.
- **EXPR-10.** A symbol operator may carry named configuration immediately
  after its token: `a /{zero: ieee} b`. Bind these arguments to the same core
  identity and its public configuration parameters, using the declaration's
  roles, defaults, and ordinary argument checks. Configuration changes neither
  precedence nor associativity and creates no ambient policy. A named-entry
  brace in this position is configuration, not the right operand's block.
  Reject empty configuration, unknown/duplicate options, incompatible types,
  and options unsupported by the selected operator contract. Intrinsic harness
  and constraint forms retain their own rules.
- **EXPR-11.** `<=>` is full comparison at the comparison precedence tier.
  It returns an ordinary core comparison outcome rather than a bool. The core
  [numeric contract](https://github.com/hhenson/hgl_core/blob/main/spec/numeric-operators.md)
  defines outcomes, numeric policies, and their relationship to Boolean
  comparisons. Source checking and required-constant failures follow ERR-9;
  choosing an error-producing policy does not introduce exception syntax.
- **EXPR-12.** Resolve configuration that selects an expression's source
  result type during source checking. A runtime-varying selector cannot change
  that type. In particular, core division's `zero` policy selects f64 or
  `result<f64, error>` as defined by its numeric contract; it must be known
  before checking uses of the result. Ordinary type and phase checks apply.
- **EXPR-13.** Evaluate a configured binary expression in written order:
  left operand, configuration entries in their written order, then right
  operand, each once. Unary configuration precedes its operand. Preserve
  EXPR-6's short-circuiting and reject unsupported configuration under EXPR-10.
  An escaping error prevents later evaluation steps; an ordinary Err value
  follows its result contract. Lowering must preserve this order. A named call
  instead follows EXPR-2's written argument order: shared parameter binding
  does not imply identical effect order for differently written expressions.
- **EXPR-14.** Scalar operator configuration binds during graph construction
  in temporal composition. In a value function or runtime hook, evaluate it
  for the current invocation. Configuration must not silently introduce a
  temporal input: time-varying temporal configuration requires a signature
  explicitly admitting that input. EXPR-12 still requires result-type selectors
  to resolve during checking. Configuration grants no purity or phase exemption.

Accepted illustration, complete module:

```hgl
module examples.bindings

const fn doubled(value: i64) -> i64 {
    var result: i64 = value
    result += value
    return result
}

test arithmetic { assert doubled(3) == 6 }
```

Deliberate rejection: reading `var result: i64` before assignment. Conformance
is planned; the shown test is an illustration, not an execution report.

Accepted syntax fragments, with suitable ordinary expressions `a`, `b`, `c`,
and `d`; use-site admission is checked separately:

```hgl
[a, b, c]
[a: b, c: d]
[a]
[1s: 7]
[]
```

The fourth fragment has one pair entry. It acquires timed-trace meaning only
at an admitted trace use site. Deliberate syntax rejection under EXPR-9:
`[a, b: c]`. Executable conformance is planned.

Accepted expression fragments, with numeric operands `a`/`b` and the core
policy values `ieee`/`nan` in scope:

```hgl
a =={abs: 1e-9, rel: 1e-6} b
a / b
a /{zero: ieee} b
a /{zero: nan} b
a <=> b
```

The division forms select the core default or an explicit policy. Full comparison
produces a core outcome that ordinary bindings and control flow can inspect.
With the public core operator in scope, `a /{zero: ieee} b` binds the same
operands and configuration as `div_(a, b, zero: ieee)`.
Deliberate source rejection under EXPR-10: duplicate `zero` configuration or an
option absent from the public operator contract. Runtime observations belong
to core; executable conformance is planned.

Accepted value-expression fragments, with admitted f64 helpers `left`, `right`,
and `tolerance` that record their own invocation names:

```hgl
left() =={abs: tolerance()} right()
eq_(left(), right(), abs: tolerance())
```

The symbolic form calls `left`, `tolerance`, `right`; the named form calls
`left`, `right`, `tolerance`. Each call occurs once. In temporal composition,
the scalar tolerance is bound during graph construction; runtime payload reads
are unavailable there. Deliberate source rejection under EXPR-14: binding a
time-series tolerance to the built-in scalar `abs` option. Core owns executable
observations; conformance is planned.

Accepted value-function fragments for EXPR-12, with the core `error` policy
value in scope:

```hgl
const fn ieee_ratio(a: f64, b: f64) -> f64 => a / b
const fn checked_ratio(a: f64, b: f64) -> result<f64, error> =>
    a /{zero: error} b
```

The return annotations check the distinct selected result types. Deliberate
source type rejection, without handling the selected result value:

```hgl
const fn unchecked_ratio(a: f64, b: f64) -> f64 =>
    a /{zero: error} b
```

No expected output type silently changes the policy.

Deliberate source rejection under EXPR-12, a bounded expression in a value
function where `a`/`b` are f64 and `selected_zero` is a runtime policy value
chosen between `ieee` and `error`:

```hgl
a /{zero: selected_zero} b
```

The selector cannot determine the source result type during checking.
Executable conformance is planned. The [error examples](errors-and-diagnostics.md#checked-division-propagation)
distinguish returning Err data from explicitly propagating node failure.

## Proposed structural delta construction

```ebnf
delta_constructor = "delta", "<", type, ">", "(", [ delta_arguments ], ")" ;
delta_arguments = delta_argument, { ",", delta_argument }, [ "," ] ;
delta_argument = member_name, ":", expression ;
```

Sparse fields use the shared sequence-of-pairs syntax, including empty brackets;
their key/index expressions must satisfy the required-constant restrictions.
Entries retain exact originating shape. Names, key/index types, duplicate
entries, and fixed bounds are checked before publication admission. Omitted
children are sparse omissions, not ordinary unset required fields. Construction
and payload evaluation follow written order and retain independent values.

## Core boundary

Core supplies syntax-bound operation behavior and temporal projection/mutation.
The language supplies binding, contextual typing, and expression evaluation.
