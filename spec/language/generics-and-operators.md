# Generics and operators

Status: agreed generic, pack, and nominal-operator baseline. Detailed core
operation domains belong to core and retain their own decision status.

## Grammar

```ebnf
generic_parameters = "<", generic_parameter, { ",", generic_parameter }, [ "," ], ">" ;
generic_parameter = IDENTIFIER | "...", IDENTIFIER, [ cardinality ]
                  | "const", IDENTIFIER, ":", value_type, [ "=", const_expression ] ;
cardinality = "{", INTEGER_LITERAL, [ ":", ( INTEGER_LITERAL | "*" ) ], "}" ;
generic_arguments = "<", generic_argument, { ",", generic_argument }, [ "," ], ">" ;
generic_argument = type | const_expression ;
requires_clause = "requires", constraint_expression ;
constraint_expression = constraint_or ;
constraint_or = constraint_and, { "||", constraint_and } ;
constraint_and = constraint_term, { "&&", constraint_term } ;
constraint_term = "!", constraint_term | "(", constraint_expression, ")"
                | constraint_relation | constraint_call
                | "each", IDENTIFIER, "in", constraint_operand,
                  "{", constraint_expression, "}" ;
constraint_relation = constraint_operand, ( "==" | "in" | "is" ), constraint_operand ;
constraint_operand = type | additive_expr | constraint_set ;
constraint_set = "{", constraint_operand, { ",", constraint_operand }, [ "," ], "}" ;
constraint_call = ( IDENTIFIER | qualified_name ), "(",
                  [ constraint_operand, { ",", constraint_operand } ], ")",
                  [ "->", type ] ;
operator_decl = "operator", callable_name, [ generic_parameters ],
                function_signature, [ requires_clause ], { operator_properties } ;
operator_properties = "properties", "<", value_type, { ",", value_type }, ">",
                      "{", operator_property, { ",", operator_property }, [ "," ], "}" ;
operator_property = "associative" | "commutative" | "identity", "=", const_expression ;
instantiate_decl = "instantiate", instantiation, { ",", instantiation }, [ "," ] ;
instantiation = callable_name, "<", materialization_argument,
                { ",", materialization_argument }, [ "," ], ">" ;
materialization_argument = type | const_expression | "_" ;
```

Constraint semantic forms disambiguate type equality, constant equality, and
operator signatures. This is a restricted compile-time language.
Unparenthesized relations and Boolean connectives belong to the constraint
expression; group a constant expression that uses them as operand data.

## Rules

- **GEN-1.** Generic scope covers the whole declaration. Repeated parameters
  require equivalent types or equal constants; differently named variables are
  independent. Occurrence requirements intersect: ordinary positions require
  value types, temporal positions admit temporal shapes, and delta formation
  imposes its own admitted domain.
- **GEN-2.** Resolve every needed generic from actual inputs, requested output,
  or solvable equality. A generic struct application supplies all arguments or
  infers them from constructor fields and expected type. A type position, a
  parent included, has nothing to infer from and supplies them: `value: Box`
  rejects where `Box(value: 7)` infers. A constructor takes the arguments
  of the type expected when that is the same struct. Otherwise it matches
  each field's type, part for part, with the type of the value the field is
  given: `T` is `i64` for `values: list<T>` given a `list<i64>`. The first
  field that says what a parameter is decides it, and the others are checked
  against that. A call resolves a generic of its callee from what it writes
  after the name (GEN-8), then from the type its place expects of the
  result, then from its arguments in declared parameter order (GEN-12),
  whatever order they are written in. The first of these that says what a
  parameter is decides it, and what follows is checked against that: an
  integer literal is widened to an `f64` so decided (EXPR-5), and another
  argument of another type rejects. Partial applications and `_` struct
  arguments are not admitted. A generic value parameter of a function or a
  candidate may have a default (GEN-8); no other generic parameter has one.
  A type pack is a sequence, not a singular source type.
- **GEN-3.** `requires` may use type/category membership, equalities, the
  membership of a constant in a set of constants, field/pack reflection, and
  named operator requirements. It cannot inspect temporal values or call
  arbitrary language functions. A positive conjunctive equality
  may bind one unresolved generic from known data; under negation/disjunction
  it only tests already-resolved data. Solve bindings to a fixed point; reject
  unresolved/conflicting dependencies.
- **GEN-4.** An operator has nominal identity `(module, name)` and owns public
  names/order, roles, defaults, generic relationships, and requirements. An
  `impl fn` is a temporal candidate of the operator it names and an
  `impl const fn` a value candidate. Each binds that operator through local
  declaration, selective import, or the prelude. An operator is neither
  temporal nor value: the candidate a call selects is. No matching operator
  is a declaration error. A plain fn is not a candidate.
- **GEN-5.** Match candidates against the requested output before inputs, then
  check their conditions. Select exactly one most-specific admitted candidate.
  Concrete types outrank variables; structurally constrained variables outrank
  unconstrained ones. No match rejects; tied most-specific candidates are
  ambiguous. Library membership and registration order grant no priority.
  A call in a value function or a runtime body selects among the value
  candidates. A call in composition selects among all of them; a value
  candidate it selects for temporal arguments is lifted (FN-6). There, a
  temporal candidate outranks a value candidate of the same parameter
  types, as composition chooses the temporal definition of a pair (FN-4).
  A candidate that takes each integer literal as the `i64` it is outranks
  one for which a literal is widened to an `f64` (EXPR-5): `doubled(1)`
  selects `doubled(value: i64)` where there is one, and
  `doubled(value: f64)` where there is not.
- **GEN-6.** An operator declares the parameters every candidate takes.
  A candidate's shape honors them and the operator's generic relationships,
  and it may declare further options of its own, each with a default or
  without. An option that selects among candidates is a generic value
  parameter, `<const zero: Zero = .ieee>` (GEN-13): its value is known when
  the call is checked, which is what lets it select. An option that does
  not select is a parameter. Across the candidates of one operator an
  option's name is one kind: a generic value parameter in every candidate
  that declares it, or a parameter in every one, so that what
  `{zero: .error}` names is known before a candidate is selected. Scalar
  specialization may accept a constant where the contract admits temporal
  lifting; it cannot require temporal data for a declared const argument.
  A call is matched against each candidate's own signature. One that names
  an option admits only the candidates that declare it. An option the call
  omits takes its candidate's default, and a candidate whose option has
  none is admitted only when the call gives it. The operator owns the
  defaults of the parameters it declares. A candidate states in `requires`
  the values of a generic value parameter it is for, by comparing it with
  constants: `requires zero == .error`, `requires zero in {.ieee, .nan}`. A
  call admits the candidate only where that holds for the parameter's value
  at the call, a default included. Candidates for different values may
  differ in result type (EXPR-12). A candidate may give any of its inputs a
  default. The candidates of an operator stay resolvable together: once
  defaults fill what a call omits, no call matches two of them equally. With
  candidates `add(lhs = 1, rhs = 1)` and `add(lhs = 1.0, rhs = 2.0)`,
  `add()` would. A
  value candidate and a temporal candidate with the same parameter types
  stay resolvable: a call in composition selects the temporal one (GEN-5),
  and a call in a value function or a runtime body the value one.
- **GEN-7.** A generic body is checked under its stated requirements; choosing
  a specialization does not authorize unrelated operations or reify a type/size
  parameter as an ordinary numeric runtime value.
- **GEN-8.** `instantiate` requests candidates, not calls or exports. `_` there
  retains a generic parameter. A call may supply its generic arguments as a
  generic constructor does, `count<i64>(3)`. It may supply the first of them
  and leave the rest to be resolved under GEN-2: with
  `operator convert<T, V>(value: V) -> T`, `convert<f64>(count)` supplies T
  and resolves V from `count`. A struct application supplies all or none
  (GEN-2). A call rejects when it leaves a generic parameter unresolved.
  A call may give a generic value parameter of its callee by name, among
  its named arguments or in a symbol's configuration (EXPR-10):
  `div_(a, b, zero: .error)`. A name binds the callee's generic value
  parameter of that name, or else its parameter of that name. The value a
  call gives a generic value parameter is a compile-time constant: a
  literal, a `.member`, a generic value parameter in scope, or a constant
  expression of those. Any other rejects. A generic value parameter the
  call does not give takes its default, when it has one.
- **GEN-9.** Property clauses apply to explicit concrete domains. Missing
  metadata gives no algebraic guarantee. Recorded laws are contracts, not
  proofs authorizing reassociation across overflow or floating-point effects.
- **GEN-10.** Pack views preserve positional order or supplied named keys.
  `each T in types(Ts)` checks every selected type; an empty conjunction is true.
  A cardinality follows the pack it bounds, on a pack parameter or a generic
  pack parameter: `{n}`, `{n:*}`, and `{n:m}` mean exact, minimum, and
  inclusive bounded arity, with m at least n. A pack without one admits any
  number of arguments, none included. A generic struct takes an argument for
  each parameter that is no pack and, for its pack, as many as the pack
  admits; one whose only parameter is a pack that admits none may be named
  bare. An argument that names a type pack stands for as many as that pack
  holds; where that is a range, the application rejects only when no length
  in it fits. `schemas(pack)` is a runtime-only
  borrowed schema view, not a storable value/result or general reflection
  facility.
- **GEN-11.** Argument inference removes reference boundaries recursively from
  an otherwise unbound generic; such an input observes values. An explicit
  ref pattern binds its underlying type. A stated resolution preserves its
  reference boundaries. A requested output binds a whole output variable as
  requested; nested structural variables follow ordinary reference removal.
  A structural projection resolves no generic and preserves a declared ref.
- **GEN-12.** Compare structural specificity component by component; reference
  qualification adds no priority, and a repeated variable counts once. Scalar
  configuration separates only otherwise equally specific candidates. A prior
  binding constrains every later occurrence in declared parameter order.
- **GEN-13.** A generic parameter denotes a type, type pack, or compile-time
  ordinary value, never a temporal connection or current payload. Infer its
  kind from signature positions and constraints: the type of a parameter, a
  result, or a field, or a part of one, makes it a type; a size or a default
  makes it a value; as an argument of a generic type it is what that type's
  parameter is. An explicit `const N: V` declares a value parameter of
  ordinary type V. `type(N) in {i64, f64}` constrains the ordinary type of a
  value parameter N; `T in {i64, f64}` instead constrains a type parameter T.
  `type(N)` here is compile-time reflection, not a runtime payload read or a
  general first-class type value. Reject conflicting kinds, and a struct's
  parameter whose kind no field or constraint determines. A function whose
  signature determines no kind for a parameter is accepted: a call supplies
  the parameter, and one that does not rejects. Constraining N's type does
  not resolve N's value.
- **GEN-14.** Retain `<...>` for generic arguments. In a type position, or
  directly following (LEX-27) a name resolved to a generic type, function,
  or operator in an expression, `<` opens generic arguments rather than a
  comparison. Check each argument against the corresponding parameter kind;
  shared name syntax is not a second interpretation. A lone name among the
  arguments is a constant when it names a value: a generic value parameter,
  a local, or a parameter. Reject an invalid application without comparison
  fallback.
  Ordinary `<` and `>` do not order type names. At the current generic nesting
  level, `>` closes the argument list; parenthesize a greater-than expression
  used as a constant argument. Nested generic closers follow LEX-8.
- **GEN-15.** Constraint negation binds before conjunction, which binds before
  disjunction: `A || B && C` means `A || (B && C)`, for Boolean constraints
  A, B, and C. Evaluate binding equalities under GEN-3's restrictions.
- **GEN-16.** Constraint calls share one syntactic form. Resolve their meaning
  from the referenced declaration or intrinsic: a constraint predicate tests
  compile-time data; a named operator requirement tests callability with the
  supplied types. A result arrow constrains that operator's result type and
  is not admitted on a predicate call. This grants no arbitrary function calls.

Syntax binds these core identities independently of local same-named functions:

| Syntax | Identity |
| --- | --- |
| `+`, binary `-`, `*`, `/` | `add_`, `sub_`, `mul_`, `div_` |
| `//`, `%`, unary `-` | `floordiv_`, `mod_`, `neg_` |
| `==`, `!=`, `<`, `<=`, `>`, `>=` | `eq_`, `ne_`, `lt_`, `le_`, `gt_`, `ge_` |
| `<=>` | `compare_`, the core full-comparison identity |
| Index and member projection | `getitem_`, `getattr_` where projection rules require an operation |

Boolean forms `&&`, `||`, and `!` bind `and_`, `or_`, and `not_` respectively.
A chain of three or more operands joined by `&&` alone, with no parentheses
among them, binds `all_` over its operands in order: `ready && open && live`
is `all_(ready, open, live)`. One joined by `||` alone binds `any_`. The
candidate selected for the operands' types (GEN-5) takes the chain whole.
The core [scalar operator contract](https://github.com/hhenson/hgl_core/blob/main/spec/scalar-operators.md)
owns the default candidates of both identities. `(ready && open) && live`
is no chain: it binds `and_` twice.
Operator configuration uses the same identity; EXPR-10 defines its source form.
The core [numeric contract](https://github.com/hhenson/hgl_core/blob/main/spec/numeric-operators.md)
owns default and configured numeric behavior, including full comparison.

Harness comparison, null presence checks, constraint predicates, and structural
projections have their own language rules; token shape alone does not make
them extensible candidate calls.

Accepted illustration, complete module:

```hgl
module examples.generic

fn same<T>(left: T, right: T) -> T => left
```

Both arguments require the same resolved source type. Deliberate rejection:
an `impl fn missing` without an operator `missing` in scope. Conformance planned.

Accepted value-function fragments for the order of GEN-2, with
`const fn same<T>(left: T, right: T) -> T`,
`const fn repeat<T>(n: i64) -> list<T>`, and `count` an `i64` in scope:

```hgl
same<f64>(1, 2)
let real: f64 = same(1, 2)
same(1, 2)
let names: list<str> = repeat(3)
```

T is `f64` as written; `f64` as the place expects, each literal widened;
`i64` from the `1`; and `str` as the place expects.
`same(right: 1, left: 2.0)` is an `f64`: `left` is declared first and
decides T, and the `1` is widened. Deliberate rejections: `same("a", 1)`,
at the `1`;
`let real: f64 = same(1, count)`, at `count`, which is no literal; and
`let names = repeat(3)`, which leaves T unresolved (`generic.unresolved`).

Accepted declaration fragments illustrating kind inference and constraints:

```hgl
struct Box<T> { value: T }
const fn boxed() -> Box<i64> => Box<i64>(value: 7)

fn preserve<T, N>(items: list<T, N>) -> list<T, N> => items

fn numeric<T>(value: T) -> T requires T in {i64, f64} { return value }

const fn configuration<N>() -> bool requires type(N) in {i64, f64} { return true }
```

`Box<i64>(value: 7)` constructs the declared generic type; its angle brackets
are not comparisons. In `preserve`, T is a type and N is a compile-time integer
size. In `configuration`, N is a compile-time value restricted to those ordinary types;
its value must still be resolved under GEN-2. These are source illustrations,
not execution claims. Deliberate source rejection under GEN-13: using the same
generic as both an element type and a list size. Conformance is planned.

Accepted declaration fragments illustrating pack cardinality:

```hgl
operator first_of<T>(values: ...T{1:*}) -> T

operator pair_of<...Ts{2}>(values: ...Ts) -> str

operator labelled<...Fields>(values: ...{Fields}{1:4}) -> str
```

`first_of` takes one or more values of one type. `pair_of` takes exactly two
values, of the two types in `Ts`. `labelled` takes one to four named
arguments and keeps their names. Deliberate rejection, in parsing: a
cardinality on a parameter that is no pack, as in `value: f64{2}`.

Accepted declaration fragment, a value candidate and a temporal candidate of
one operator (GEN-4, GEN-5):

```hgl
operator doubled<T>(value: T) -> T

impl const fn doubled(value: i64) -> i64 => value * 2

impl fn doubled(value: f64) -> f64 => value * 2.0
```

`doubled(21)` in a `const fn` or a runtime body selects the value candidate.
In composition, `doubled(count)` for a temporal `count` of `i64` selects it
too and lifts it, and `doubled(price)` for a temporal `price` of `f64`
selects the temporal candidate. With `impl fn doubled(value: i64) -> i64`
declared too, `doubled(count)` in composition selects that temporal
candidate, and `doubled(21)` in a `const fn` still the value one.
Deliberate rejection: `doubled(1.5)` in a `const fn`, where no value
candidate takes an `f64`.

Accepted declaration fragment, candidates for the values of an option that
selects among them (GEN-6):

```hgl
export enum Zero { ieee, nan, error }

operator ratio<L, R, O>(lhs: L, rhs: R) -> O

native impl fn ratio<const zero: Zero = .ieee>(lhs: f64, rhs: f64) -> f64
    requires zero in {.ieee, .nan}

native impl fn ratio<const zero: Zero>(lhs: f64, rhs: f64) -> result<f64, error>
    requires zero == .error
```

For temporal `a` and `b` of `f64`, `ratio(a, b)` takes the default `.ieee`,
selects the first candidate and is an `f64`, as `ratio(a, b, zero: .nan)`
is. `ratio(a, b, zero: .error)` selects the second and is a
`result<f64, error>`. A value candidate is written the same way. Deliberate
rejections: `ratio(a, b, zero: policy)` for a parameter `policy`, which is
no compile-time constant (`generic.argument_constant`); and under GEN-3
`requires rhs == 0.0` on either candidate, which reads a temporal input.

Accepted declaration fragment, an option that does not select and that one
candidate declares (GEN-6):

```hgl
operator scaled<L, R, O>(lhs: L, rhs: R) -> O

impl const fn scaled(lhs: f64, rhs: f64, exact: bool = false) -> f64 =>
    lhs * rhs

impl const fn scaled(lhs: str, rhs: i64) -> str => lhs * rhs
```

The operator declares what both candidates take, and the first declares
`exact` beside it. `scaled(1.5, 2.0)` selects the first with `exact`
false, and `scaled(1.5, 2.0, exact: true)` selects it too.
`scaled("ab", 2)` selects the second. Deliberate rejections:
`scaled("ab", 2, exact: true)`, `operator.no_candidate`, where no candidate
for a `str` and an `i64` declares `exact`; and a third candidate declaring
`<const exact: bool>`, since `exact` is a parameter in the first.

## Core boundary

Core owns syntax-bound identities and mandatory operation behavior. The language
owns nominal binding and selection. Extension candidates preserve the core
identity and applicable contract rather than redeclaring it in a library.
