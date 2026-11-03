# Native boundaries

Status: agreed portable native authoring model. Target ABI and borrowed endpoint
helper realization are outside this portable declaration contract.

## Grammar

```ebnf
native_type_decl = [ "export" ], "native", "type", IDENTIFIER ;
native_function_decl = "native", [ "const" ], "fn", callable_name,
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
- **NATIVE-2.** A selected implementation part completes exactly one shared
  declaration. Match name, public parameters/order, roles, generics, complete
  result type, constraints, and admitted failures. Native implicit conversions
  cannot satisfy an incompatible contract. Duplicate declarations or selected
  implementations are errors, not overloads.
- **NATIVE-3.** A temporal implementation `{}` denotes a graph; `{ when; }`
  denotes a node. Start/stop hooks require when and are unique. A native const
  implementation stays value-level and admits no temporal lifecycle hooks.
  A missing body is a declaration, not an empty implementation.
- **NATIVE-4.** Target selection is explicit. Part/folder names have no magic
  selection priority, and assembly order changes no match. Selected injectables
  and hooks describe requirements; the provider must satisfy the same observable
  contract. Construction/destruction do not replace semantic start/stop.
- **NATIVE-5.** Value-helper capability requirements propagate transitively.
  A value helper cannot claim its own out/scheduler/state lifetime. Borrowed
  access must be explicit in the checked contract; ordinary scalar parameters
  do not acquire endpoint privileges because the implementation is native.
- **NATIVE-6.** A native type has a canonical nominal identity and explicit
  capabilities. It may be an atomic payload/key only where those capabilities
  and ownership rules admit it. Matching external identity is not established
  by coincident display names. Opaque resource handles are not automatically
  ordinary temporal values.
- **NATIVE-7.** Checking an interface needs no executable provider loading.
  Execution requires a compatible selected provider. An absent binding is an
  implementation/admission failure, never permission to change the role or
  return an empty successful result. Portable source adds no per-tick dynamic
  symbol lookup or registry mutation.

Accepted illustration, shared interface:

```hgl
module examples.native

native fn accumulate(value: i64) -> i64
native const fn describe(value: i64) -> str
```

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
```

These are contract fragments, not target implementations. Deliberate rejection:
a scalar provider bound to the temporal accumulate declaration. Conformance
is planned.

## Core boundary

Core owns mandatory native declarations and provider obligations. Libraries may
declare additional native interfaces. Foreign source embedding and build
selection belong to providers, not this portable grammar.
