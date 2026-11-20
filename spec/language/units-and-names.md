# Units and names

Status: agreed module, application, visibility, default-binding, lookup, and
constant rules. The sentence of UNIT-14 that begins "Proposed:" is
proposed.

## Grammar

```ebnf
source_file = [ NL ], ( module_decl | application_decl ),
              { NL, top_level_item }, [ NL ], EOF ;
module_decl = "module", module_path, [ "part", IDENTIFIER ] ;
application_decl = "application", module_path ;
module_path = IDENTIFIER, { ".", IDENTIFIER } ;
qualified_name = IDENTIFIER, "::", import_name ;
use_decl = "use", module_path, ( "::", import_set | "as", IDENTIFIER ) ;
import_set = "{", import_name, { ",", import_name }, [ "," ], "}" ;
import_name = IDENTIFIER ;
top_level_item = use_decl | declaration | store_statement | run_statement ;
store_statement = inject_decl | IDENTIFIER, call_suffix ;
declaration = function_decl | struct_decl | enum_decl | operator_decl
            | instantiate_decl | native_type_decl | native_function_decl
            | constant_decl | test_decl | test_context ;
constant_decl = [ "export" ], "const", IDENTIFIER, ":", value_type,
                "=", const_expression ;
member_name = IDENTIFIER | ? a hard reserved word ? ;
callable_name = IDENTIFIER ;
```

The referenced declarations belong to their type, function, generic, native,
or test chapters, `run_statement` to
[program execution](program-execution.md), `inject_decl` to the
[temporal model](temporal-model.md), and `call_suffix` to
[expressions](expressions-and-bindings.md). Portable source is covered here;
target build metadata is not.
The hard reserved words are listed in [source text](source-text.md); LEX-9
admits one as a member name.

## Agreed rules

- **UNIT-1.** A file names one module and optionally one part. An assembled
  module has one declaration scope, at most one unnamed file, and unique part
  names. Every file names the same module. Part names do not enter declaration
  identity; assembly order does not change meaning.
- **UNIT-2.** Imports precede every declaration of their file, tests and test
  contexts included, and bind in their own file. A selective import binds
  public declarations; an alias qualifies a public interface with
  `alias::name` and names nothing alone. A dotted module path creates no
  privileged access to another module.
- **UNIT-3.** Ordinary functions, types, and constants are private unless
  exported, a native function and a native type among them (NATIVE-1).
  Operators are public by definition. A candidate, `impl fn` or
  `impl const fn`, is attached to its operator and is reached only through
  it. Exported signatures are closed over reachable types and required
  defining metadata; private dependencies do not become source-importable.
  Import cycles are rejected, a module that imports itself included.
- **UNIT-4.** Types have module-qualified nominal identity where specified.
  An alias does not create a new identity. Referencing the same public type
  through two imports preserves one identity and its generic arguments.
- **UNIT-5.** Duplicate ordinary functions of the same name/role, parameter
  names, part names, or local bindings in one scope are rejected. Operator
  implementations are candidates under the generic chapter. A temporal and a
  value function may share a name under [role selection](functions-and-phases.md).
  No other two declarations bind the same name; an operator and a function
  do not.
- **UNIT-6.** Module declarations and shared test helpers may be referenced
  forward. Ordinary block locals become visible after declaration. A test is
  not a callable value, and its name is not a binding. Test-only declarations
  cannot be exported or used by production declarations.
- **UNIT-7.** Resolve defaults in the declaration's enclosing binding closure
  with its generic environment, excluding the callable's formal arguments and
  the struct's instance fields. Earlier const arguments are also excluded. A
  default that names one of them rejects. Imported defaults retain their
  defining closure, including private references.
- **UNIT-8.** Defaults remain required-constant/cold expressions checked against
  the expected type. Any parameter but a pack and a `var` parameter (FN-15)
  may have one: a value function's, const configuration, or a temporal input. An omitted temporal argument is
  wired as a constant of its default, so the default is a value a constant of
  the parameter's type can publish. A default of `null` marks a temporal input
  optional: an omitted argument binds an input that is never valid. A
  parameter whose type is written `T?` needs no default to be omitted: it is
  taken to have a default of `null`, so an omitted value parameter is null
  and an omitted temporal input is never valid. A `var` parameter is the
  exception: it is always given, whatever its type, since the call assigns
  the place it is given (FN-15). A parameter that may be omitted follows
  every parameter that may not. A default cannot use caller
  bindings or invent numeric generic values.
- **UNIT-9.** Compare inherited defaults by bound expression and canonical type
  after substitution, before folding. Declaration identity matters; source
  location, alias spelling, and how a literal is written do not (UNIT-13):
  `= 1` and `= 1.0` of an `f64` field are one default. Conflicting
  inherited defaults require an explicit child override.
- **UNIT-10.** A name resolves to the one binding in scope for it (UNIT-12):
  an earlier local or iteration binding of an enclosing block; a parameter of
  an enclosing function; a generic parameter of an enclosing declaration
  (GEN-1); a module declaration; a selective import; or the prelude. A name
  that none of these binds rejects (`name.unknown`). Direct
  capability receivers select their capability operation; reserved activation
  forms are not shadowable identifiers. Resolution grants no payload read
  during composition and does not change default binding (UNIT-7). A name
  in a type position names a struct, an enum, a native type, or a generic
  parameter that is a type. The types of a signature see the declaration's
  generic parameters and the scope around the declaration, not its
  parameters: in `fn f(n: i64, items: list<f64, n>)` the size names no `n`.
- **UNIT-11.** The prelude is the public declarations of core's mandatory
  module, `core`, and nothing else. Every module binds them without a `use`.
- **UNIT-12.** A block's scope is the scope around it plus its own bindings;
  an anonymous function's is the scope around it plus its parameters (FN-14).
  A `switch` case has no scope of its own: its statements bind in the block
  around the `switch`. No binding takes a name already in scope: that is a
  duplicate definition. A local or iteration binding, parameter, or generic
  parameter takes no name bound there by an enclosing block or function, a
  module declaration, a `use` in its file, or the prelude. `state`, `cache`,
  and `inject` bind locals; an `each` constraint binds a generic parameter
  within its braces. A `use` binds no name another `use` of its file has
  bound. A declaration takes no name its module imports or uses as an import
  alias, and outside core's mandatory module neither a declaration nor an
  import alias takes a prelude name. The test scope (TST-1) is in scope only
  in tests and test-only declarations. UNIT-5 governs declarations that share
  a name. An `impl fn` or `impl const fn` adds a candidate to its operator
  (GEN-4), not a name. A member name is not a binding and may match any
  name. A test's name is not one either: it may match a declaration's, and
  is unique among its module's tests (TST-2).
- **UNIT-13.** A constant, `const name: T = expression`, names one ordinary
  value of the type T. Its expression is a const expression (EXPR-5) checked
  against T: of literals, enum members, constructors, other constants, and
  calls of value functions and operators on those. It reads no parameter,
  local, temporal value, or capability. The constant has that one value
  wherever it is read, and a failure in evaluating it is a required-constant
  failure (ERR-9). A constant is read by its name wherever an ordinary value
  is. It is a constant expression where a `const` parameter requires one
  (FN-1), and a compile-time constant where a generic value argument
  (GEN-8) or a `switch` label (CTRL-12) requires one. It is referenced
  forward as any module declaration is (UNIT-6), and its value depends on
  no constant that depends on it.
  A compile-time constant (GEN-8), a literal or a constant among them, is
  one value of the type its place gives it (EXPR-5): `1` where an `f64` is
  expected is `1.0`. Two compile-time constants are one where those values
  are the same, however each is written: `1`, `1.0` and `1.00` as an
  `f64`; `60s` and `1m`; the `datetime`s `@2026-09-03T09:30Z` and
  `@2026-09-03T10:30+01:00`, one instant. The same value is not `==`,
  which the [core numeric contract](https://github.com/hhenson/hgl_core/blob/main/spec/numeric-operators.md)
  gives: `-0.0` and `0.0` are two constants, though `-0.0 == 0.0`, and any
  two NaNs, `0.0 / 0.0` among them, are one constant, though `==` holds
  for no NaN. That sameness gives type identity (TYPE-3) and the instance
  a call makes (GEN-13). An equality the program writes, in a `requires`
  as elsewhere, is the language's `==` (GEN-3). How a constant is written
  is kept with it, for diagnostics and for a formatter, which writes it
  back as written, and is never compared: a constant has one value
  however it is written.
- **UNIT-14.** An application is one file, headed `application name`, where
  a module's files are headed `module name`. It holds what a module holds,
  and one run statement (RUN-1), which follows its declarations. Between its
  declarations and its run statement it may set the run's store: `inject
  global_state`, and then calls of `put` on it. Each is evaluated once, in
  the order written, before the run's graph is built, and its value is read
  as a constant's is (UNIT-13), with no parameter in scope. Any other
  statement there rejects (`application.statement`). A module holds none
  of these. Those statements, and the arguments of the run statement's
  call and its configuration, are value code; the call itself, of the
  entry, is composition (RUN-1). Value code calls `const fn`s only: a call
  of any other function there, a node, a graph or a source, rejects. An
  operator applied there is resolved by GEN-5, and the candidate it
  selects must be a value candidate, an `impl const fn`: one that is
  temporal rejects the same way. Nothing there is lifted (FN-6). Proposed:
  those rejections have the code `call.role`. An application's path is no
  module's and no other application's. Its declarations are private: an application
  exports nothing, and no `use` names one.

Accepted illustration, complete module:

```hgl
module examples.defaults

export const scale: i64 = 2
const first: i64 = 3 * scale + 1

const fn seed() -> i64 => first
const fn choose(base: i64, value: i64 = seed()) -> i64 => base + value

test defining_closure {
    assert choose(1) == 8
}
```

`first` is a constant whose expression reads another, `scale`, which other
modules may import (UNIT-13). The default calls the module's `seed`
(UNIT-7); the body reads its parameters (UNIT-10). The
[eval illustration](tests-and-eval.md) calls
`delta_value` with no `use`: it is a prelude name (UNIT-11). Two blocks,
neither enclosing the other, may each bind the same local name.
Deliberate source rejections: a default of `base` in place of `seed()` reads
a formal argument (UNIT-7); a first parameter named `seed`, a local named
`base` in a block body of `choose`, a local of an inner block named for an
earlier local of a block around it, a local declared in two cases of one
`switch`, a test helper named `choose`, or any
function, parameter, or local named `delta_value` or for a declaration its
file imports, takes a name already in scope (UNIT-12), as does a parameter
or a local named `scale`; and `const twice: i64 = twice + 1`, a constant
whose value depends on itself (`constant.cycle`). Conformance is planned.

## Core boundary

Core supplies the mandatory module, `core`, and syntax-bound identities. A target's
declared dependency closure supplies extension interfaces; ambient registries
cannot silently add candidates to a portable target.
