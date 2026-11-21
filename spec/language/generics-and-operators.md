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
  require equivalent types or one constant (UNIT-13); differently named
  variables are independent. Occurrence requirements intersect: ordinary
  positions require value types, temporal positions admit temporal shapes,
  and delta formation imposes its own admitted domain.
- **GEN-2.** Resolve every needed generic from actual inputs, requested output,
  or solvable equality. A generic struct application supplies all arguments or
  infers them from constructor fields and expected type. A type position, a
  parent included, has nothing to infer from and supplies them: `value: Box`
  rejects where `Box(value: 7)` infers. Where an application supplies none,
  a generic value parameter that nothing says anything of takes its
  default: for `struct Window<const size: i64 = 3>`, `value: Window` is a
  `Window<3>`. A constructor takes the arguments
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
  argument of another type rejects. One later argument changes what an
  earlier argument decided: an abstract struct that the type decided
  inherits from widens the parameter to that struct, so the order of two
  such arguments does not matter. What the call writes, or its place
  expects, is not widened: `same<Round>(round, shape)` rejects at `shape`.
  A nullable place `U?` says of a result that is a type parameter and
  nothing more only that the parameter is `U` or `U?`. The arguments then
  decide it, in declared parameter order; where they say nothing of it, the
  call writes it (GEN-8). What the call writes is taken whole:
  `same<i64?>(1, 2)` is an `i64?`.
  Partial applications and `_` struct
  arguments are not admitted. A generic value parameter may have a default,
  a function's, a candidate's, a struct's, or an operator's, which it takes
  where a call or an application neither gives it nor resolves it (GEN-8).
  A type parameter and a pack have none.
  A type pack is a sequence, not a singular source type.
- **GEN-3.** `requires` may use type/category membership, equalities, the
  membership of a constant in a set of constants, field/pack reflection, and
  named operator requirements. It cannot inspect temporal values or call
  arbitrary language functions. A membership of a type parameter in a set
  of types, `requires T in {i64, f64}`, authorizes in the body (GEN-7) an
  operation that each listed type has a candidate for, `a < b` there, and
  each instance selects its own candidate (GEN-5). An operation that a
  listed type lacks rejects at the declaration. A membership of a
  generic value parameter, `requires M in {.fast, .slow}`, authorizes in
  the body a call that gives M as an option (GEN-6) where each listed value
  has a candidate that admits it, and each instance selects its own. A
  positive conjunctive equality
  may bind one unresolved generic from known data; under negation/disjunction
  it only tests already-resolved data. Solve bindings to a fixed point; reject
  unresolved/conflicting dependencies. A call of a function whose `requires`
  does not hold with the generic arguments the call resolves is rejected,
  and so is an application of a struct whose `requires` does not hold with
  its arguments. An equality of two constants there is the `==` a program
  evaluates, and not their sameness (UNIT-13): with `const k: f64`,
  `requires k == 0.0` holds where `k` is `-0.0`.
- **GEN-4.** An operator has nominal identity `(module, name)` and owns public
  names/order, roles, defaults, generic relationships, and requirements. An
  `impl fn` is a temporal candidate of the operator it names and an
  `impl const fn` a value candidate. Each binds that operator through local
  declaration, selective import, or the prelude. An operator is neither
  temporal nor value: the candidate a call selects is. No matching operator
  is a declaration error. A plain fn is not a candidate.
- **GEN-5.** Match candidates against the requested output before inputs, then
  check their conditions. A candidate that gives `result<T, error>` meets a
  requested T or `T?` as one that gives T does, and its call takes the form
  the place expects (ERR-15): the place does not choose between them.
  Select exactly one most-specific admitted candidate.
  Concrete types outrank variables; structurally constrained variables outrank
  unconstrained ones. No match rejects; tied most-specific candidates are
  ambiguous. Library membership and registration order grant no priority.
  A call in a value function or a runtime body selects among the value
  candidates. A call in a test's or an application's value code (TST-3,
  UNIT-14) selects among all of them, a value candidate outranking a
  temporal one of the same parameter types, and a temporal candidate it
  selects rejects. A call in composition selects among all of them; a value
  candidate it selects for temporal arguments is lifted (FN-6). There, a
  temporal candidate outranks a value candidate of the same parameter
  types, as composition chooses the temporal definition of a pair (FN-4).
  A candidate that takes each integer literal as the `i64` it is outranks
  one for which a literal is widened to an `f64` (EXPR-5): `doubled(1)`
  selects `doubled(value: i64)` where there is one, and
  `doubled(value: f64)` where there is not. Whether a literal is widened
  is settled by the candidates' result and parameter types, before their
  conditions: where a candidate that gives the requested output takes
  every integer literal as an `i64`, none is widened, and the conditions
  select among the candidates that do. An option so does not change what
  a literal is. Among candidates that tie and differ only in which integer
  literals they take as an `f64`, the one that takes every one of them as
  an `f64` is selected.
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
  default. A value candidate may mark a parameter `var` (FN-15); the
  operator's declaration marks none, and the mark takes no part in selecting
  a candidate. The candidates of an operator stay resolvable together: once
  defaults fill what a call omits, no call matches two of them equally. With
  candidates `add(lhs = 1, rhs = 1)` and `add(lhs = 1.0, rhs = 2.0)`,
  `add()` would. A
  value candidate and a temporal candidate with the same parameter types
  stay resolvable: a call in composition selects the temporal one (GEN-5),
  and a call in a value function, a runtime body or value code the value
  one.
- **GEN-7.** A generic body is checked under its stated requirements; choosing
  a specialization does not authorize unrelated operations or reify a type/size
  parameter as an ordinary numeric runtime value. It is judged once, where it
  is declared, by what its `requires` states, and never again for the
  generic arguments a call of it gives. A call of an operator in it is
  admitted where those requirements make a candidate known to take it, and
  the declaration rejects where they do not (`operator.no_candidate`). They
  make one known where a candidate takes every generic argument they
  admit, where a named operator requirement states the call (GEN-16), and,
  under a membership, where each listed type has one (GEN-3). Each instance
  then selects its own candidate (GEN-5).
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
  literal, an enum member however it is written (TYPE-7), `Zero.nan` or
  `.nan`, a constant (UNIT-13), a generic value parameter in scope, or a
  constant expression of those. Any other rejects. A generic value parameter the call does not
  give takes its default, when it has one. An operator's takes part in
  the output requested, in GEN-2's order: what the call writes, then what
  its place expects, then its arguments, and the default where none of
  them says. For
  `operator window<T, const size: i64 = 3>(value: T) -> list<T, size>`,
  `let xs = window<f64>(x)` requests a `list<f64, 3>`, and
  `let ys: list<f64, 5> = window<f64>(x)` a `list<f64, 5>`.
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
- **GEN-11.** No type holds a reference (TYPE-12), so argument inference
  removes none: a generic binds from the type of the connection it is
  given, at a `ref` parameter as at any other (FN-16). `ref low: T` given a
  connection of `f64` binds T to `f64`, and so does `value: T` given a
  `ref` result of `f64`, whose target that input observes (TIME-10). A
  requested output binds a whole output variable as requested, or, where
  it is nullable, as GEN-2 says. A structural projection resolves no
  generic.
- **GEN-12.** Compare structural specificity component by component; a `ref`
  mark adds no priority (FN-16), and a repeated variable counts once. Scalar
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
  not resolve N's value. Generic arguments that are the same, each type
  canonical and each value one constant (UNIT-13), make one type, or one
  instance of a generic function; arguments that differ make another. For
  `struct Scaled<const k: f64>`, `Scaled<1>` and `Scaled<1.0>` are one
  type, and `Scaled<-0.0>` is not `Scaled<0.0>`. A call makes the
  instance its generic arguments name, and that instance the ones its body
  calls for, at any depth. Where the instances of one declaration never
  end when the program is compiled, the program rejects, whatever a run of
  it would do: an implementation has a bound on the instances of one
  declaration, and rejects the program where they pass it, with the code
  `generic.instance_limit`, at the call that makes the first of them.
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
  A requirement names a value operation, whatever declares it: it is tested
  against the operator's value candidates, and holds where one most-specific
  value candidate takes the types supplied (GEN-5). So what a graph requires
  is what a node or a value function it calls may require. A candidate that
  marks a parameter `var` (FN-15) answers no requirement: a requirement
  supplies types, and no place.

Syntax binds these core identities independently of local same-named functions.
A symbol is the call of its identity: its operands are read as that call's
arguments are, the type its place expects is requested of the candidates
(GEN-5), and its configuration gives the call's named arguments (EXPR-10).
`n + 1` is `add_(n, 1)` wherever it stands.

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

Accepted declaration fragments, nullable places of a result that is a type
parameter and nothing more (GEN-2):

```hgl
const fn same<T>(a: T, b: T) -> T => a
const fn larger<T>(a: T, b: T) -> T
    requires T in {i64, f64} => if a < b { b } else { a }

const fn kept(a: i64, b: i64) -> i64? => same(a, b)
const fn best(a: i64, b: i64) -> i64? => larger(a, b)
```

In `kept` the arguments decide T, an `i64`, and the result fits the `i64?`
place (TYPE-14), as it does in `let x: i64? = same(a, b)` and as an
argument for an `i64?` parameter. In `best` T is `i64` too, and the
membership holds (GEN-3), as it would not of an `i64?`.
`same<i64?>(a, b)` is an `i64?` as written. Conformance planned.

Accepted declaration fragments, each with a default on a generic value
parameter (GEN-2):

```hgl
const fn keep<const size: i64 = 3>(values: list<f64>) -> list<f64> => values
struct Window<const size: i64 = 3> { values: list<f64, size> }
operator window<T, const size: i64 = 3>(value: T) -> list<T, size>
```

`keep(xs)` has `size` 3 and `keep(xs, size: 5)` has it 5. `value: Window`
is a `Window<3>`, and `Window<5>` is as written. `let xs = window<f64>(x)`
requests a `list<f64, 3>`; `window<f64, 5>(x)` a `list<f64, 5>`; and
`let ys: list<f64, 5> = window<f64>(x)` a `list<f64, 5>`, since the place
says what `size` is before the default does. An argument does too: for
`operator copy<const size: i64 = 3>(value: list<f64, size>) -> list<f64, size>`
and a `list<f64, 5>` `xs`, `copy(xs)` requests a `list<f64, 5>`.
Deliberate rejections:
`value: Box` for `struct Box<T>`, whose parameter has no default and which
a type position does not infer; and `let zs: list<f64, 5> = window<f64, 3>(x)`,
where what the call writes decides and the place then expects another size.

Accepted declaration fragments, generic value arguments that are one
constant or two (GEN-13, UNIT-13):

```hgl
struct Scaled<const k: f64> { value: f64 }
const fn reciprocal<const k: f64>() -> f64 => 1.0 / k
```

`Scaled<1>`, `Scaled<1.0>` and `Scaled<1.00>` are one type, and
`reciprocal<1>()` and `reciprocal<1.0>()` call one instance.
`reciprocal<0.0>()` is positive infinity and `reciprocal<-0.0>()` negative
infinity, whichever is called first. With `requires k == 0.0` added,
`reciprocal` admits both of those calls (GEN-3). Deliberate rejection: a
`Scaled<-0.0>` where a `Scaled<0.0>` is expected (`type.mismatch`).

Deliberate rejection, declaration fragments, a generic whose instances
never end (GEN-13):

```hgl
const fn climb<const N: i64>(x: i64) -> i64 {
    if x <= 0 { return N }
    return climb<N + 1>(x - 1)
}

const fn climbed() -> i64 => climb<0>(3)
```

`climb<N>` calls for `climb<N + 1>` whatever `x` is, so the call
`climb<0>(3)` makes `climb<0>`, `climb<1>`, `climb<2>` and on without end
when the program is compiled, and the program rejects, though a run of it
would return 3 once `x` reached zero. It rejects with the code
`generic.instance_limit`, at that call, which makes the first of them.

Accepted value-function fragments for a symbol as its call, with `n` an
`i64`, `xs` a `list<f64>`, and structs `Round` and `Square` that inherit
from the abstract `Shape`:

```hgl
let a: f64 = n + 1
let b = xs == [1, 2]
let c: f64 = 1 + 2
let d = round == shape
```

`n + 1` is `add_(n, 1)`: its place requests an `f64`, and the candidate for
an `i64` and an `f64` takes the `1` widened. `[1, 2]` is read as the
`list<f64>` its parameter is. `1 + 2` ties three candidates that give an
`f64`, and selects the one that takes both literals as an `f64`.
`round == shape` compares two `Shape`: the second operand widens what the
first decided, as `shape == round` decides it outright. Deliberate
rejections: `n //{zero: .nan} 2`, where a division of two `i64` takes the
`2` as the `i64` it is and none admits `.nan` (`operator.no_candidate`); and
`round == square`, two structs of which neither inherits from the other.

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
not execution claims. Deliberate source rejections: under GEN-13, using the
same generic as both an element type and a list size; under GEN-3, a call of
`numeric` with a `str`, whose `requires` does not hold (`requires.unmet`).
Conformance is planned.

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

Accepted declaration fragments, a named operator requirement (GEN-7,
GEN-16):

```hgl
operator tick<V>(value: V) -> V

impl fn tick(value: i64) -> i64 {
    when { return value }
}

impl const fn tick(value: f64) -> f64 => value

operator pass<T>(a: T) -> T

impl fn pass<T>(a: T) -> T requires tick(T) -> T => tick(a)

fn reals(x: f64) -> f64 => pass(x)
```

`pass` requires `tick` of its `T`, and its body calls it under that
requirement. `reals` gives `pass` an `f64`, for which `tick` has a value
candidate: the requirement holds and the call selects `pass`. Deliberate
source rejection: `fn wholes(n: i64) -> i64 => pass(n)`. For an `i64`,
`tick` has a temporal candidate alone, which a call in composition may
select and a requirement is not tested against: the requirement does not
hold, no candidate of `pass` admits the call, and it rejects
(`operator.no_candidate`). Conformance planned.

Accepted declaration fragments, generic bodies judged where they are
declared (GEN-3, GEN-7):

```hgl
const fn less<T>(a: T, b: T) -> bool requires T in {i64, f64} => a < b

operator kind_of<V>(value: V) -> i64

impl const fn kind_of(value: i64) -> i64 => 1

impl const fn kind_of<T>(value: T) -> i64 requires !(T == i64) => 2

const fn wrap<U>(value: U) -> i64 requires kind_of(U) -> i64 => kind_of(value)
```

`<` has a candidate for two `i64` and one for two `f64`, so `less`
compares under its membership, and the instances that `less(1, 2)` and
`less(1.5, 2.0)` make each select their own. The requirement of `wrap`
states its call: `wrap(7)` is 1, its instance selecting the first
candidate of `kind_of`, and `wrap(true)` is 2, selecting the second.
Deliberate rejections, each at its declaration, whatever
its calls give (`operator.no_candidate`): `less` with no `requires`, as no
candidate of `<` takes every `T`; `less` with `requires T in {i64, Point}`,
for a `struct Point { x: f64 }` that `<` has no candidate for; and `wrap`
with no `requires`, as the two candidates of `kind_of` between them take
every `U` and neither takes every one. Conformance planned.

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
is. `ratio(a, b, zero: .error)` selects the second, as
`ratio(a, b, zero: Zero.error)` does, wherever it stands: the place takes
no part in that selection, and chooses the form of the
`result<f64, error>` it gives (ERR-15). It is a connection of that result
where one is expected; of `f64` where an `f64` is, whose node fails on a
zero divisor; and of `f64?` where that is, which publishes null for one. A
value candidate is written the same way. Deliberate rejections:
`ratio(a, b, zero: policy)` for a parameter `policy`, which is no
compile-time constant (`generic.argument_constant`); and under GEN-3
`requires rhs == 0.0` on either candidate, which reads a temporal input.

Accepted declaration fragment, with the candidates of
`ratio` above, an option passed on through a generic value parameter
(GEN-3, GEN-7):

```hgl
fn ratio_in<const z: Zero>(lhs: f64, rhs: f64) -> f64
    requires z in {.ieee, .nan} => ratio(lhs, rhs, zero: z)
```

The first candidate admits each listed value, and the instance that
`ratio_in(a, b, z: .nan)` makes selects it. With
`requires z in {.ieee, .error}` the declaration is accepted too: the
candidate for `.error` gives a `result<f64, error>`, which meets the
`f64` the return expects (GEN-5), and the instance that
`ratio_in(a, b, z: .error)` makes fails its node on a zero divisor
(ERR-15). Deliberate rejection, at the declaration
(`operator.no_candidate`): the same with no `requires`, as no candidate
admits every `z`.

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
`<const exact: bool>`, since `exact` is a parameter in the first
(`impl.option_kind`, at the first candidate's `exact`).

## Core boundary

Core owns syntax-bound identities and mandatory operation behavior. The language
owns nominal binding and selection. Extension candidates preserve the core
identity and applicable contract rather than redeclaring it in a library.
