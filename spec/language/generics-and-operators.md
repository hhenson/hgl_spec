# Generics and operators

Status: agreed generic, pack, and nominal-operator baseline. Detailed core
operation domains belong to core and retain their own decision status.

## Grammar

```ebnf
generic_parameters = "<", generic_parameter, { ",", generic_parameter }, [ "," ], ">" ;
generic_parameter = IDENTIFIER | "...", IDENTIFIER
                  | "const", IDENTIFIER, ":", value_type ;
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
  infers them from constructor fields and expected type. Partial applications,
  generic defaults, and `_` struct arguments are not admitted. A type pack is
  a sequence, not a singular source type.
- **GEN-3.** `requires` may use type/category membership, equalities, field/pack
  reflection, and named operator requirements. It cannot inspect temporal
  values or call arbitrary language functions. A positive conjunctive equality
  may bind one unresolved generic from known data; under negation/disjunction
  it only tests already-resolved data. Solve bindings to a fixed point; reject
  unresolved/conflicting dependencies.
- **GEN-4.** An operator has nominal identity `(module, name)` and owns public
  names/order, roles, defaults, generic relationships, and requirements. An
  `impl fn` binds that operator through local declaration, selective import,
  or the prelude. No matching operator is a declaration error. A plain fn is
  not a candidate.
- **GEN-5.** Match candidates against the requested output before inputs, then
  check their conditions. Select exactly one most-specific admitted candidate.
  Concrete types outrank variables; structurally constrained variables outrank
  unconstrained ones. No match rejects; tied most-specific candidates are
  ambiguous. Library membership and registration order grant no priority.
- **GEN-6.** Candidate shape must honor the operator's public arguments and
  generic relationships. Scalar specialization may accept a constant where
  the contract admits temporal lifting; it cannot require temporal data for
  a declared const argument. Defaults remain owned by the public declaration.
- **GEN-7.** A generic body is checked under its stated requirements; choosing
  a specialization does not authorize unrelated operations or reify a type/size
  parameter as an ordinary numeric runtime value.
- **GEN-8.** `instantiate` requests candidates, not calls or exports. `_` there
  retains a generic parameter. Explicit type arguments on ordinary calls have
  no admitted general form in this baseline.
- **GEN-9.** Property clauses apply to explicit concrete domains. Missing
  metadata gives no algebraic guarantee. Recorded laws are contracts, not
  proofs authorizing reassociation across overflow or floating-point effects.
- **GEN-10.** Pack views preserve positional order or supplied named keys.
  `each T in types(Ts)` checks every selected type; an empty conjunction is true.
  `{n}`, `{n:*}`, and `{n:m}` cardinalities mean exact, minimum, and inclusive
  bounded arity. `schemas(pack)` is a runtime-only borrowed schema view, not a
  storable value/result or general reflection facility.
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
  kind from signature positions and constraints. An explicit `const N: V`
  declares a value parameter of ordinary type V. `type(N) in {i64, f64}`
  constrains the ordinary type of a value parameter N; `T in {i64, f64}`
  instead constrains a type parameter T. `type(N)` here is compile-time
  reflection, not a runtime payload read or a general first-class type value.
  Reject conflicting kinds or an unresolved kind lacking a determining
  declaration/constraint. Constraining N's type does not resolve N's value.
- **GEN-14.** Retain `<...>` for generic arguments. In a type position, or
  following a name resolved to a generic type in an expression, `<` opens
  generic arguments rather than a comparison. Check each argument against
  the corresponding parameter kind; shared name syntax is not a second
  interpretation. Reject an invalid application without comparison fallback.
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

## Core boundary

Core owns syntax-bound identities and mandatory operation behavior. The language
owns nominal binding and selection. Extension candidates preserve the core
identity and applicable contract rather than redeclaring it in a library.
