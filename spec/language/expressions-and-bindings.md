# Expressions and bindings

Status: agreed expression/binding baseline. The sparse-field rules of
structural delta construction are proposed. EXPR-16, a range, and EXPR-17,
a list made by a loop, are agreed.

## Grammar

```ebnf
expression = or_expr ;
or_expr = and_expr, { "||", [ operator_configuration ], and_expr } ;
and_expr = equality_expr, { "&&", [ operator_configuration ], equality_expr } ;
equality_expr = comparison_expr,
                { ( "==" | "!=" ), [ operator_configuration ], comparison_expr } ;
comparison_expr = additive_expr,
                  { ( "<" | "<=" | ">" | ">=" | "<=>" ),
                    [ operator_configuration ], additive_expr }
                | IDENTIFIER, "in", "arg_keys", "(", IDENTIFIER, ")" ;
additive_expr = multiplicative_expr,
                { ( "+" | "-" ), [ operator_configuration ], multiplicative_expr } ;
multiplicative_expr = unary_expr,
                      { ( "*" | "/" | "//" | "%" ),
                        [ operator_configuration ], unary_expr } ;
unary_expr = ( "-" | "!" ), [ operator_configuration ], unary_expr | postfix_expr ;
operator_configuration = "{", configuration_argument,
                         { ",", configuration_argument }, [ "," ], "}" ;
configuration_argument = member_name, ":", expression ;
postfix_expr = primary_expr, { call_suffix | index_suffix | member_suffix } ;
call_suffix = [ operator_configuration ], "(", [ arguments ], ")" ;
index_suffix = "[", expression, "]" ;
member_suffix = ".", member_name ;
primary_expr = literal | IDENTIFIER | qualified_name | implied_member
             | "(", expression, ")" | tuple_literal | sequence_literal
             | set_literal | keyed_literal
             | typed_constructor | delta_constructor | error_constructor | type_of
             | pack_loop
             | function_reference | function_expr | if_expression
             | eval_expression | run_expression | block ;
arguments = argument, { ",", argument }, [ "," ] ;
argument = [ member_name, ":" ], expression | "...", expression ;
tuple_literal = "(", [ expression, ",", [ expression, { ",", expression }, [ "," ] ] ], ")" ;
set_literal = "{", ( "," | expression, ",", [ expression, { ",", expression }, [ "," ] ] ), "}" ;
keyed_literal = "{", ( ":" | keyed_entry, { ",", keyed_entry }, [ "," ] ), "}" ;
keyed_entry = ( expression | member_name ), ":", expression ;
sequence_literal = "[", [ sequence_entries | loop_entries ], "]" ;
loop_entries = "for", IDENTIFIER, "in", iterable, ":", expression ;
pack_loop = "{", "for", IDENTIFIER, "in", iterable, ":", expression, "}" ;
iterable = range | expression ;
range = additive_expr, "..<", additive_expr ;
sequence_entries = sequence_values | sequence_pairs ;
sequence_values = sequence_item, { ",", sequence_item }, [ "," ] ;
sequence_pairs = sequence_pair, { ",", sequence_pair }, [ "," ] ;
sequence_pair = sequence_item, ":", sequence_item ;
sequence_item = expression | "_" ;
typed_constructor = ( IDENTIFIER | qualified_name ), generic_arguments,
                    [ operator_configuration ], "(", [ arguments ], ")" ;
function_reference = ( IDENTIFIER | qualified_name ), generic_arguments ;
error_constructor = "error", "(", [ arguments ], ")" ;
type_of = "type", "(", expression, ")" ;
implied_member = ".", member_name ;
local_decl = ( "let" | "var" ), IDENTIFIER, [ ":", type ], "=", expression
           | "var", IDENTIFIER, ":", type ;
assignment = place, ( "=" | compound_operator, [ operator_configuration ] ),
             expression ;
place = IDENTIFIER, { index_suffix | member_suffix } ;
compound_operator = "+=" | "-=" | "*=" | "/=" | "//=" | "%=" ;
const_expression = expression ;
```

Block/control/function/eval forms are defined in their owning chapters.
The sequence grammar admits contextual forms; semantic rules restrict their use.
`const(name)` uses ordinary call syntax and lookup, as specified by FN-5.
GEN-14 resolves generic construction versus comparison using declaration lookup.
`typed_constructor` is also the form of a call that supplies generic arguments
(GEN-8).

## Rules

- **EXPR-1.** The grammar gives precedence from low `||` to high postfix forms.
  Repeated binary forms associate left; unary forms associate right. A chained
  comparison is not implicit mathematical conjunction. Parentheses select
  grouping; every nonempty tuple literal has a comma. Newline constraints follow source-text rules.
- **EXPR-2.** Evaluate ordinary call/constructor arguments once in written
  order. Positional arguments precede named ones; reject duplicates, unknown
  names, missing required arguments, and wrong types. Named arguments use
  the selected public callable's parameter names; a named argument may
  also give a generic value parameter of the callee (GEN-8), and a call of
  an operator an option its candidates declare (GEN-6). Positional arguments
  fill the parameters from the first and stop where they stop; named
  arguments come in any order, and a parameter that may be omitted may be
  left out. A parameter that may be omitted, one with a default or a type
  written `T?` that is no `var` parameter, follows every parameter that may
  not (UNIT-8). A struct's
  constructor takes named arguments only (TYPE-5). An enum's takes one
  argument, by position (TYPE-7). An argument of a `var` parameter is a
  place, which the call assigns (FN-15).
- **EXPR-3.** Initialization fixes a binding's value/connection category and
  type; an annotation checks that category rather than changing it. `let` is
  immutable, and so is a parameter that is not `var` (FN-15); `var` may be
  assigned within the same category/type. An uninitialized
  typed var supplies no value or connection; reaching assignments must resolve
  consistently and every path reaching a read must assign it. In a value
  function, one whose type is written `T?` is null instead, and is assigned
  from its declaration (CTRL-11). An inner
  declaration remains local. A compound assignment reads its existing target.
  An assignment may name a place inside a value, at any depth: a field,
  `point.x = 1`, or an element, `row[0] = 1`, `book["a"] = 1`. The place's
  root is a `var`, a `var` parameter, a `state`, a `cache`, or, in a
  `when`, `out` (TIME-7). A `let`, any other parameter, and an iteration
  binding are read-only all the way down. The value is of the field's type;
  of a list's element type, under an `i64` index; of a dictionary's value
  type, under a key of its key type; or of a tuple's position, under a
  constant index. A key the dictionary has not is added; given to a `var`
  parameter, the entry of such a key fails the call instead (FN-15). An
  index outside a list, below zero or at or past its size, fails where it
  is evaluated, with code `value.index_out_of_range`.
  A set has no place to assign. A compound assignment reads the place
  first, `total.count += 1`, and binds as EXPR-18 says. `+=` on a `var` of type `str` appends to it
  in place (TYPE-1).
- **EXPR-4.** Ordinary values and temporal connections remain distinct. A graph
  binding denotes a connection without reading a payload. Reassigning a graph
  var changes the described connection; it creates neither mutable per-tick
  scalar storage nor feedback. Earlier aliases/consumers keep their original
  connection. Scalar/connection reassignment across categories rejects.
  Persistent runtime history uses state/cache.
- **EXPR-5.** An expected type contextualizes literals, constructors, a
  collection's child read where that is checked when it runs (ERR-14), and
  a result (ERR-5, ERR-15), but
  cannot erase structural/atomic boundaries or nominal identity. An integer
  literal is an `f64` where one is expected: `let x: f64 = 1` binds `1.0`
  (UNIT-13). A value that
  has a type is not converted by its place: an `i64` binding where an `f64`
  is expected rejects, and its conversion is written with core's
  `convert<T>(value)`. Two are the exception, read by the place and not
  converted: a `result<T, error>`, a call's or a binding's, takes its
  form from the place (ERR-15), as a `result<T, E>` of another E does
  where its T is expected in a function that gives a `result<U, E>`
  (ERR-5); and a `T?` where a T is expected is read (ERR-12). A const
  expression
  uses the same syntax with required-constant semantic restrictions; it
  cannot inspect temporal values.
- **EXPR-6.** In value/runtime contexts, Boolean connectives short-circuit left
  to right. The operands of `&&`, `||`, and `!` are bools.
  A valid or tag guard must precede the protected payload read; a nullable
  one is read as ERR-12 says.
  Composition forms describe dependencies and select core identities; they do
  not read current payloads or inherit scalar execution order as topology.
- **EXPR-7.** Structural projection selects a child connection. Atomic
  projection computes from a whole payload. Temporal dictionary key selection
  may give a reference, a designation of the keyed child (TIME-10);
  strict/safe ordinary dictionary reads follow
  [lookup rules](errors-and-diagnostics.md). A structural projection of a
  `ref` parameter selects its child and reads no payload, whatever the
  member's name; an atomic projection of one, `quote.bid` of
  `ref quote: atomic<Quote>`, reads its payload and rejects (FN-16).
  Brackets read an element of a
  list or a tuple by an `i64` index, and the value a dictionary has for a
  key. An index outside an ordinary list fails the read with code
  `value.index_out_of_range`, as it fails an assignment (EXPR-3). A tuple's
  index is a constant within its length: it selects the element's type.
- **EXPR-8.** Ordinary tuples may contain runtime payload expressions.
  `()` is the empty tuple of type `tuple<>`, `(a,)` is a singleton tuple,
  and `(a)` groups an expression. A call `f(())` supplies one empty tuple;
  `f()` supplies zero arguments. A value function returning `()` produces one tuple value.
  Harness
  `_` and timed cells exist only in test traces; outside them a sequence is
  ordinary data and a placeholder is not a value. Tuple syntax does not
  construct graph connections from separate temporal endpoints.
- **EXPR-9.** Brackets always delimit a sequence. Entries are either all values
  or all colon-separated pairs; mixing these forms is a source error. `[a]`
  has one value entry; `[a: b]` has one pair entry, not a standalone pair value.
  Ordinary set and dictionary literals may specify their kind with braces.
  `{a, b}` and `{a,}` are sets. Where the expected type determines a
  concrete struct schema, `{field: value}` constructs that struct (TYPE-5).
  In a dictionary literal, both sides of each colon are expressions.
  Element and entry admission for sets and dictionaries follows the same type
  and duplicate rules as for a sequence in the corresponding set or
  dictionary context. `{,}` is an empty set and `{:}` is an empty
  dictionary. Their element or key and value types come from context, as in
  `let members: set<i64> = {,}` and `let prices: dict<str, f64> = {:}`.
  `{}` and `{a}` are blocks (CTRL-10).
  In an ordinary expression, brackets default to a list. When they form a
  list, `[a: b, c: d]` is shorthand for `[(a, b), (c, d)]` and follows
  the same typing rules. Each entry contributes one tuple in written order,
  including entries whose first elements compare equal. Empty brackets
  `[]` supply no element type of their own: `let xs: list<tuple<str, i64>> = []`
  is an empty list of pairs.
  The use site and expected type constrain
  element types, keys, ordering, uniqueness, and placeholder admission; syntax
  alone does not select a dictionary, timed trace, or sparse delta. `_` is not
  an ordinary value and remains restricted by EXPR-8 and the trace profile.
  Both `let xs: list<i64> = [8, 13]` and `let ys: set<i64> = [8, 13]`
  use the same sequence syntax. The annotation selects the collection type.
  GEN-5 specifies the order when operator candidates supply both contexts.
  With no expected type, value entries make a list of the one type they
  have: `[1, 2.5]` is a `list<f64>`, its integer literal an `f64` (EXPR-5),
  and `[1, null]` a `list<i64?>`. Entries of two types reject, and so do
  entries that give no type: `[]`, `[null]`. Of pairs that make a
  dictionary, two whose keys are the same constant (UNIT-13) reject:
  `["a": 1, "a": 2]` (`dict.duplicate_key`). Where the compiler does not
  see that two keys are one, the value written last for a key is the one
  held: `[k: 1, "a": 2]`, for a parameter `k` that is `"a"`, holds 2 under
  `"a"`. Of entries that make a set, two that are the same constant reject
  too, as what a slip of the hand writes: `[1, 2, 1]`. Proposed: with the
  code `set.duplicate_member`. Where the compiler does not see that two
  are one, the set holds the member once.
- **EXPR-10.** A symbol operator may carry named configuration immediately
  after its token: `a /{zero: .ieee} b`. The brace is the next token; spaces
  may stand between. Bind these arguments to the same core identity as the
  named arguments of a call of it: to the operator's parameters and the
  options its candidates declare (GEN-6), with their roles, defaults,
  and ordinary argument checks. An entry finds what it names as a named
  argument does (GEN-8): the candidate's generic value parameter of that
  name, or else its parameter of that name. Configuration changes neither
  precedence nor associativity and creates no ambient policy. A named-entry
  brace in this position is configuration. Parentheses put a keyed literal
  in the right operand, as in `p == ({x: 3})` when the operand type is known.
  `{}` there is
  configuration only when an operand follows it on its line; otherwise it is
  the operand's empty block. Reject empty configuration, duplicate
  options, incompatible types, and an option that no candidate for the
  operands declares. Intrinsic harness and constraint forms retain their own
  rules.
  A call of a named operator carries the same configuration after its
  name, before its arguments: `ratio{zero: .error}(a, b)` is
  `ratio(a, b, zero: .error)`. It binds as a symbol's does, and the
  arguments in the parentheses are then the call's others. A function's
  call carries its configuration the same way. An entry names a
  configuration parameter of the callee (FN-18) before any other: only
  the configuration gives one. In the header of an `if`, `for`, `switch`,
  `when`, or `while` such a call stands in parentheses, since the first
  `{` there opens the body (CTRL-9).
- **EXPR-11.** `<=>` is full comparison at the comparison precedence tier.
  It returns core's `CmpResult`, an ordinary enum, rather than a bool; `==`,
  `!=`, `<`, `<=`, `>`, and `>=` return a bool. The core
  [numeric contract](https://github.com/hhenson/hgl_core/blob/main/spec/numeric-operators.md)
  defines outcomes, numeric policies, and their relationship to Boolean
  comparisons. Source checking and required-constant failures follow ERR-9;
  choosing an error-producing policy does not introduce exception syntax.
- **EXPR-12.** Resolve configuration that selects an expression's source
  result type during source checking. A runtime-varying selector cannot change
  that type. In particular, core division's `zero` policy selects f64 or
  `result<f64, error>` as defined by its numeric contract; it must be known
  before checking uses of the result. The operator's candidates state which
  values of the option each is for (GEN-6), and the value at the call
  selects among them. Such an option is a generic value parameter of its
  candidates, so the value a call gives it is a compile-time constant
  (GEN-8). The type the call's place expects selects no policy: the place
  takes the form of the result the selected candidate gives (ERR-15).
  Ordinary type and phase checks apply.
- **EXPR-13.** Evaluate a configured binary expression in written order:
  left operand, configuration entries in their written order, then right
  operand, each once. Unary configuration precedes its operand. Preserve
  EXPR-6's short-circuiting and reject unsupported configuration under EXPR-10.
  An escaping error prevents later evaluation steps; an ordinary Err value
  follows its result contract. Lowering must preserve this order. A named call
  instead follows EXPR-2's written argument order, the entries of its
  configuration before the arguments in its parentheses: shared parameter
  binding does not imply identical effect order for differently written
  expressions.
- **EXPR-14.** Where a value function, a value candidate of an operator
  among them, declares an option says what a graph may give it. One it
  declares as a configuration parameter, in its braces (FN-18), is
  configuration: in temporal composition it is bound when the graph is
  built, and a connection given to it rejects (`connection.as_constant`);
  in a value function or runtime hook it is evaluated for the current
  invocation. One it declares among its parameters, in its parentheses, is
  a parameter as any other: a connection given to it in composition is an
  input of the lifted call (FN-6). What the callee declares decides, in
  either spelling of the call (EXPR-10), and not where the call writes the
  entry. So configuration introduces no temporal input, and a value
  function admits one by declaring a parameter. A temporal function's
  configuration parameter is what it declares: a connection or, marked
  `const`, what is fixed when the graph is built (FN-18). EXPR-12 still
  requires result-type selectors to resolve during checking. Configuration
  grants no purity or phase exemption.
- **EXPR-15.** In a graph, an assignment into a place of a `var` is copy on
  write. A graph's `var` denotes a connection (EXPR-4), and a field of a
  connection is no storage: `point.x = price` rebinds `point` to a new
  connection, which is `point` with its `x` from `price`. Nothing is
  changed in place: what read `point` before the assignment keeps the
  connection it had. The place, the type of its value, and a compound
  assignment are as EXPR-3 has them.
- **EXPR-16.** `a..<b` is a range: the `i64` values from `a` up
  to and not including `b`, in order, and none where `b` is not more than
  `a`. It is written as what a `for` iterates, in a statement and in a
  loop form (EXPR-17, FN-19), and is no value of its own: nothing binds
  one, and no parameter takes one: a list of its values is made by a
  loop, `[for i in 0..<n: i]` (EXPR-17). Its bounds are `i64`. In value code
  they are evaluated once, before the first pass. In composition they are
  ordinary values, fixed when the graph is built, and the loop is
  unrolled (CTRL-15).
- **EXPR-17.** A list is made by a loop:
  `[for i in 0..<n: i * i]` has one entry for each pass, in order, the
  value of the expression after the `:` with the loop's name bound. In
  value code it is a `list<T>`, T the expression's type, or, where its
  place expects a `list<T, N>` and the number of passes is the constant N,
  that fixed-size list (EXPR-5). In composition
  the loop is unrolled: its iterable is a range whose bounds are
  constants, or a connection of a fixed-size list, and N is the number of
  passes. Where the iterable is a connection, or an entry is one, the
  result is a connection of `list<T, N>`, an entry that is an ordinary
  value being the constant it is wherever a connection is expected
  (FN-3). Where the iterable is a range and the expression is an
  ordinary value in every pass, the result is the ordinary value the
  same entries written out are,
  `[for i in 2..<4: i]` as `[2, 3]`: nothing in it changes while the
  program runs, and a `const` parameter takes it (FN-1). What it iterates
  is what a `for` statement iterates in that phase (CTRL-12, CTRL-15).
- **EXPR-18.** A compound assignment binds a core identity of its own, as
  a symbol binds one (GEN-4): the place on its left is given to that
  identity's first parameter, a `var` (FN-15), and the expression on its
  right to the second. The identities are `iadd_` for `+=`,
  `isub_` for `-=`, `imul_` for `*=`, `idiv_` for `/=`, `ifloordiv_` for
  `//=`, and `imod_` for `%=`, so `count += by` is `iadd_(count, by)`. The
  candidate selected says what the place holds afterwards; core owns the
  candidates of the language's own types. It gives no result: a compound
  assignment is a statement. It may carry configuration after its token,
  `count //={zero: .fail} step`, bound as a symbol's is (EXPR-10).
  In composition a compound assignment selects no such candidate, since
  none is lifted (FN-15): `a += b` there is `a = a + b`, the rebinding
  EXPR-15 gives, and its configuration is that of `+`.
  Where the root of the place is `out`, in a `when` (TIME-7), the place is
  given as its endpoint, an `endpoint<T>` (TIME-9), and selection
  continues among the candidates that take one. Proposed: such a
  candidate takes `endpoint<T>` first and marks no `var`; every other
  candidate of a compound identity marks its first parameter `var`, and
  one that does neither rejects where it is declared.

Accepted illustration, complete module:

```hgl
module examples.bindings

const fn doubled(value: i64) -> i64 {
    var total: i64 = value
    total += value
    return total
}

test arithmetic { assert doubled(3) == 6 }
```

Deliberate rejection: reading `var total: i64` before assignment. Conformance
is planned; the shown test is an illustration, not an execution report.

Accepted illustration of a place assigned (EXPR-3), complete module:

```hgl
module examples.places

struct Point {
    x: f64
    y: f64
}

const fn moved(origin: Point, by: f64) -> Point {
    var point = origin
    point.x += by
    return point
}

const fn counted(words: list<str>) -> dict<str, i64> {
    var counts: dict<str, i64> = []
    for word in words {
        counts[word] = 1
    }
    return counts
}

fn shifted(first: Point, price: f64) -> Point {
    var point = first
    point.x = price
    return point
}
```

`moved` returns a point `by` to the right and leaves `origin` as it was: a
parameter is read-only all the way down. `counts[word] = 1` adds a key the
dictionary has not. In `shifted`, a graph, `point.x = price` makes `point`
a new connection, `first` with its `x` from `price` (EXPR-15); `first` is
the connection it was. Deliberate source rejections: `origin.x = by`, a field of
a parameter (`binding.immutable`); `point.x = "a"`, a `str` for an `f64`
field (`type.mismatch`). With `var row = [1, 2]`, `row[2] = 0` and a read
of `row[2]` each fail as they run, with `value.index_out_of_range`: the
list's positions are 0 and 1. Conformance is planned.

Accepted syntax fragments, with suitable ordinary expressions `a`, `b`, `c`,
and `d`; use-site admission is checked separately:

```hgl
[a, b, c]
[a: b, c: d]
[a]
[1s: 7]
[]
```

The fourth fragment is ordinarily a list containing the tuple `(1s, 7)`.
At a timed-trace use site it supplies the tick at offset `1s` (TST-7).
An ordinary binding `let entries = ["bid": 4, "ask": 7, "bid": 9]`
has three tuples. At positions 0 and 2 the first element is `"bid"`
and the second elements are 4 and 9 respectively. Deliberate syntax rejection under EXPR-9:
`[a, b: c]`. Executable conformance is planned.

Accepted expression fragments, with numeric operands `a`/`b` in scope; core's
`zero` option is an enum with members `ieee` and `nan` (TYPE-7):

```hgl
a =={abs: 1e-9, rel: 1e-6} b
a / b
a /{zero: .ieee} b
a /{zero: .nan} b
a <=> b
```

The division forms select the core default or an explicit policy. Full comparison
produces a core outcome that ordinary bindings and control flow can inspect.
With the public core operator in scope, `a /{zero: .ieee} b` binds the same
operands and configuration as `div_(a, b, zero: .ieee)`.
Deliberate source rejection under EXPR-10: duplicate `zero` configuration or
an option that no candidate for the operands declares. Runtime observations belong
to core; executable conformance is planned.

Accepted expression fragment, with zoned datetimes `a`/`b` in scope:

```hgl
a =={utc: true} b
```

It compares the two as UTC: they are equal when their instants are, in
whichever zones (TYPE-2).

Accepted value-expression fragments, with admitted f64 helpers `left`, `right`,
and `tolerance` that record their own invocation names:

```hgl
left() =={abs: tolerance()} right()
eq_{abs: tolerance()}(left(), right())
```

The symbolic form calls `left`, `tolerance`, `right`; the named form calls
`tolerance`, `left`, `right`. Each call occurs once. In temporal composition,
the scalar tolerance is bound during graph construction; runtime payload reads
are unavailable there. Deliberate source rejection under EXPR-14: binding a
time-series tolerance to the built-in scalar `abs` option, a configuration
parameter (`connection.as_constant`). Core owns executable
observations; conformance is planned.

Accepted value-function fragments for EXPR-12; `error` is a member of the
`zero` option's enum:

```hgl
const fn ieee_ratio(a: f64, b: f64) -> f64 => a / b
const fn checked_ratio(a: f64, b: f64) -> result<f64, error> =>
    a /{zero: .error} b
const fn unchecked_ratio(a: f64, b: f64) -> f64 =>
    a /{zero: .error} b
```

The configuration selects the candidate, and with it the result type; the
return's place then takes that result's form (ERR-15). `checked_ratio`
gives the result, an `err` for a zero divisor. `unchecked_ratio` gives the
`f64`, and a zero divisor raises that error: an execution failure with
`numeric.division_by_zero`, as `unchecked_ratio` declares no result. No
expected output type silently changes the policy: the place chooses only
the form of the selected candidate's result.

Deliberate source rejection under EXPR-12, a bounded expression in a value
function where `a`/`b` are f64 and `selected_zero` is a runtime policy value
chosen between `.ieee` and `.error`:

```hgl
a /{zero: selected_zero} b
```

The selector cannot determine the source result type during checking: it is
no compile-time constant (`generic.argument_constant`).
Executable conformance is planned. The [error examples](errors-and-diagnostics.md#checked-division-propagation)
distinguish returning Err data from raising it as node failure.

## Structural delta construction

Agreed: the constructor of a delta that is a type of its own, by its
properties (TYPE-16).

```ebnf
delta_constructor = "delta", "<", type, ">", "(", [ delta_arguments ], ")" ;
delta_arguments = delta_argument, { ",", delta_argument }, [ "," ] ;
delta_argument = member_name, ":", expression ;
```

Proposed: sparse fields use the shared sequence-of-pairs syntax, including
empty brackets; their key/index expressions must satisfy the
required-constant restrictions.
Entries retain exact originating shape. Names, key/index types, duplicate
entries, and fixed bounds are checked before publication admission. Omitted
children are sparse omissions, not ordinary unset required fields. Construction
and payload evaluation follow written order and retain independent values.

## Core boundary

Core supplies syntax-bound operation behavior and temporal projection/mutation.
The language supplies binding, contextual typing, and expression evaluation.
