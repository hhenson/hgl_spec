# Native boundaries

Status: agreed portable native authoring model. Target ABI and borrowed endpoint
helper realization are outside this portable declaration contract.

## Grammar

```ebnf
native_type_decl = [ "export" ], "native", "type", IDENTIFIER,
                   [ native_type_body ] ;
native_type_body = "{", [ NL ],
                   [ native_property, { NL, native_property }, [ NL ] ], "}" ;
native_property = member_name, ":", value_type ;
native_function_decl = [ "export" ], "native", [ "const" ], "fn",
                       callable_name, [ generic_parameters ],
                       function_signature, [ requires_clause ]
                     | "native", [ "impl" ], [ "const" ], "fn", callable_name,
                       [ generic_parameters ], function_signature,
                       [ requires_clause ], [ native_contract_body ] ;
native_contract_body = "{", [ NL ],
                       { ( inject_decl, [ ";" ] | native_hook ), [ NL ] }, "}" ;
native_hook = ( "start" | "when" | "stop" ), ";" ;
```

The optional semicolon here belongs to the native contract grammar, not to
ordinary HGL statements. Target-specific bodies are separate provider artifacts.

## Rules

- **NATIVE-1.** A native declaration defines the same language contract as an
  HGL callable. It contains no target header, symbol map, layout, or body in
  the shared interface. Native temporal and native const roles remain distinct;
  non-const parameters of a temporal declaration are inputs even if their
  payload type is scalar. All-const temporal declarations may still tick.
  A `native impl fn` or `native impl const fn` declares a candidate of the
  operator it names (GEN-4), temporal or value, whose body a provider
  supplies. A native function that is no candidate is private unless
  exported, as a function is (UNIT-3). `export` is written on its
  declaration, the form with no body, and nowhere else.
- **NATIVE-2.** A selected implementation part completes exactly one shared
  declaration. Match name, public parameters/order and their `var` and
  `ref` marks (FN-15, FN-16), roles, generics, complete result type and its
  `ref` mark, constraints, and admitted failures. Native implicit conversions
  cannot satisfy an incompatible contract. Duplicate declarations or selected
  implementations are errors, not overloads. An implementation with no
  declaration of its name and role in its module rejects, selected or not.
  Implementations in different parts are alternatives for selection; two in
  one part are duplicates. Native candidates of one operator that differ in
  role or parameter types are distinct declarations, and an implementation
  completes the candidate whose signature it matches.
- **NATIVE-3.** A temporal implementation `{}` denotes a graph; `{ when; }`
  denotes a node. Start and stop hooks require `when`, and each is written at
  most once; a repeated `when` marker adds nothing. A native const
  implementation stays value-level and admits neither `inject` nor temporal
  lifecycle hooks.
  A missing body is a declaration, not an empty implementation.
- **NATIVE-4.** Target selection is explicit. Part/folder names have no magic
  selection priority, and assembly order changes no match. Selected injectables
  and hooks describe requirements; the provider must satisfy the same observable
  contract. Construction/destruction do not replace semantic start/stop.
- **NATIVE-5.** A native value function injects nothing and has no
  out/scheduler/state lifetime (FN-9). Borrowed access must be explicit in the
  checked contract; ordinary scalar parameters do not acquire endpoint
  privileges because the implementation is native.
- **NATIVE-6.** A native type has a canonical nominal identity and explicit
  capabilities. It may be an atomic payload/key only where those capabilities
  and ownership rules admit it. Matching external identity is not established
  by coincident display names. Opaque resource handles are not automatically
  ordinary temporal values. A native type may declare read-only properties,
  each a name and an ordinary value type, written as a struct's typed fields
  are, no two of one name. `value.property` reads one as an ordinary value.
  None is assigned, and a native type is not constructed in HGL: a call of
  its name rejects.
- **NATIVE-7.** Checking an interface needs no executable provider loading.
  Execution requires a compatible selected provider. An absent binding is an
  implementation/admission failure, never permission to change the role or
  return an empty successful result. Portable source adds no per-tick dynamic
  symbol lookup or registry mutation.

Accepted illustration, shared interface:

```hgl
module examples.native

export native type Handle {
    size: i64
}

native fn accumulate(value: i64) -> i64
export native const fn describe(value: i64) -> str

operator total<T>(lhs: T, rhs: T) -> T
native impl const fn total(lhs: i64, rhs: i64) -> i64
native impl const fn total(lhs: f64, rhs: f64) -> f64

const fn larger(handle: Handle, limit: i64) -> bool => handle.size > limit
```

The two `native impl` lines declare two value candidates of `total`.
`describe` is exported, and another module calls it; `accumulate` is this
module's own. `larger` reads a property of the native type it is given.

Accepted illustration, selected same-module part:

```hgl
module examples.native part provider

native fn accumulate(value: i64) -> i64 {
    inject out
    start;
    when;
    stop;
}
native const fn describe(value: i64) -> str {}
native impl const fn total(lhs: i64, rhs: i64) -> i64 {}
native impl const fn total(lhs: f64, rhs: f64) -> f64 {}
```

These are contract fragments, not target implementations. The part does not
repeat `export`: the declaration has said it. Deliberate rejections: a
scalar provider bound to the temporal accumulate declaration;
`export native impl const fn total(...)`, and `export` on the part's
`describe`; `handle.size = 1`, a property assigned; `Handle(size: 1)`, a
native type constructed; and a second `size` in `Handle`. Conformance is
planned.

## Core boundary

Core owns mandatory native declarations and provider obligations. Libraries may
declare additional native interfaces. Foreign source embedding and build
selection belong to providers, not this portable grammar.
