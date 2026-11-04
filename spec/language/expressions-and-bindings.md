# Expressions and bindings

Status: agreed expression/binding baseline. Structural delta constructors and
recursive ordinary mutation retain their proposed status.

## Grammar

```ebnf
expression = or_expr ;
or_expr = and_expr, { "||", and_expr } ;
and_expr = equality_expr, { "&&", equality_expr } ;
equality_expr = comparison_expr, { ( "==" | "!=" ), comparison_expr } ;
comparison_expr = additive_expr, { ( "<" | "<=" | ">" | ">=" ), additive_expr } ;
additive_expr = multiplicative_expr, { ( "+" | "-" ), multiplicative_expr } ;
multiplicative_expr = unary_expr, { ( "*" | "/" | "//" | "%" ), unary_expr } ;
unary_expr = ( "-" | "!" ), unary_expr | postfix_expr ;
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
