# Generics and operators

Status: agreed generic, pack, and nominal-operator baseline. Detailed core
operation domains belong to core and retain their own decision status.

## Grammar

```ebnf
generic_parameters = "<", generic_parameter, { ",", generic_parameter }, [ "," ], ">" ;
generic_parameter = IDENTIFIER | "...", IDENTIFIER, [ cardinality ]
                  | "const", IDENTIFIER, ":", value_type ;
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
  against that. Partial applications,
  generic defaults, and `_` struct arguments are not admitted. A type pack is
  a sequence, not a singular source type.
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
- **GEN-6.** Candidate shape must honor the operator's public arguments and
  generic relationships. Scalar specialization may accept a constant where
  the contract admits temporal lifting; it cannot require temporal data for
  a declared const argument. The public declaration owns the defaults of
  const configuration. A candidate states in `requires` the values of a
  const option it is for, by comparing the option with constants:
  `requires zero == .error`, `requires zero in {.ieee, .nan}`. A call admits
  the candidate only where that holds for the option's value at the call,
  the public default included. Candidates for different values may differ
  in result type (EXPR-12). A temporal candidate marks the option `const`
  as its operator does. Every parameter of a value candidate is a value, so
  it writes the option with no marker. A candidate may give any of its
  inputs a default. The candidates of an operator stay resolvable together:
  once defaults fill what a call omits, no call matches two of them equally.
  With candidates
  `add(lhs = 1, rhs = 1)` and `add(lhs = 1.0, rhs = 2.0)`, `add()` would. A
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

Accepted declaration fragment, candidates for the values of a const option
(GEN-6); `Zero` and `div_` here stand for core's:

```hgl
export enum Zero { ieee, nan, error }

operator div_<L, R, O>(lhs: L, rhs: R, const zero: Zero = .ieee) -> O

native impl fn div_(lhs: f64, rhs: f64, const zero: Zero) -> f64
    requires zero in {.ieee, .nan}

native impl fn div_(lhs: f64, rhs: f64, const zero: Zero) -> result<f64, error>
    requires zero == .error
```

For temporal `a` and `b` of `f64`, `a / b` takes the default `.ieee`,
selects the first candidate and is an `f64`, as `a /{zero: .nan} b` is.
`a /{zero: .error} b` selects the second and is a `result<f64, error>`. A
value candidate is written the same way with `zero: Zero`. Deliberate
rejection under GEN-3: `requires rhs == 0.0` on either candidate, which
reads a temporal input.

## Core boundary

Core owns syntax-bound identities and mandatory operation behavior. The language
owns nominal binding and selection. Extension candidates preserve the core
identity and applicable contract rather than redeclaring it in a library.
