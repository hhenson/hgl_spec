# Units and names

Status: agreed module, visibility, default-binding, and lookup rules.

## Grammar

```ebnf
source_file = [ NL ], module_decl, { NL, top_level_item }, [ NL ], EOF ;
module_decl = "module", module_path, [ "part", IDENTIFIER ] ;
module_path = IDENTIFIER, { ".", IDENTIFIER } ;
qualified_name = IDENTIFIER, "::", import_name ;
use_decl = "use", module_path, ( "::", import_set | "as", IDENTIFIER ) ;
import_set = "{", import_name, { ",", import_name }, [ "," ], "}" ;
import_name = IDENTIFIER ;
top_level_item = use_decl | declaration ;
declaration = function_decl | struct_decl | enum_decl | operator_decl
            | instantiate_decl | native_type_decl | native_function_decl
            | test_decl | test_context ;
member_name = IDENTIFIER | "time" ;
callable_name = IDENTIFIER ;
```

The referenced declarations belong to their type, function, generic, native,
or test chapters. Portable source is covered here; target build metadata is not.

## Agreed rules

- **UNIT-1.** A file names one module and optionally one part. An assembled
  module has one declaration scope, at most one unnamed file, and unique part
  names. Every file names the same module. Part names do not enter declaration
  identity; assembly order does not change meaning.
- **UNIT-2.** Imports precede ordinary declarations and bind in their own
  file. A selective import binds public declarations; an alias qualifies a
  public interface with `alias::name`. A dotted module path creates no
  privileged access to another module.
- **UNIT-3.** Ordinary functions and types are private unless exported.
  Operators are public by definition. An `impl fn` is attached to its
  operator and is reached only through it. Exported signatures are closed
  over reachable types and required defining metadata; private dependencies
  do not become source-importable. Import cycles are rejected.
- **UNIT-4.** Types have module-qualified nominal identity where specified.
  An alias does not create a new identity. Referencing the same public type
  through two imports preserves one identity and its generic arguments.
- **UNIT-5.** Duplicate ordinary functions of the same name/role, parameter
  names, part names, or local bindings in one scope are rejected. Operator
  implementations are candidates under the generic chapter. A temporal and a
  value function may share a name under [role selection](functions-and-phases.md).
- **UNIT-6.** Module declarations and shared test helpers may be referenced
  forward. Ordinary block locals become visible after declaration. A test is
  not a callable value. Test-only declarations cannot be exported or used by
  production declarations.
- **UNIT-7.** Resolve defaults in the declaration's enclosing binding closure
  with its generic environment, excluding the callable's formal arguments and
  the struct's instance fields. Earlier const arguments are also excluded.
  Imported defaults retain their defining closure, including private references.
- **UNIT-8.** Defaults remain required-constant/cold expressions checked against
  the expected type. Value-function parameters and const configuration may
  have defaults; ordinary temporal parameters may not. A default cannot use
  caller bindings or invent numeric generic values.
- **UNIT-9.** Compare inherited defaults by bound expression and canonical type
  after substitution, before folding. Declaration identity matters; source
  location and alias spelling do not. Conflicting inherited defaults require
  an explicit child override.
- **UNIT-10.** A name resolves to the one binding in scope for it (UNIT-12):
  an earlier local or iteration binding of an enclosing block; a parameter or
  generic parameter of an enclosing function; a module declaration; a
  selective import; or the prelude. Direct capability receivers select their
  capability operation; reserved activation forms are not shadowable
  identifiers. Resolution grants no payload read during composition and does
  not change default binding (UNIT-7).
- **UNIT-11.** The prelude is the public declarations of core's mandatory
  module, and nothing else. Every module binds them without a `use`.
- **UNIT-12.** A block's scope is the scope around it plus its own bindings;
  an anonymous function's is the scope around it plus its parameters (FN-14).
  No binding takes a name already in scope: that is a duplicate definition. A
  local or iteration binding, parameter, or generic parameter takes no name
  bound there by an enclosing block or function, a module declaration, a `use`
  in its file, or the prelude. A declaration takes no name its module
  imports, and outside core's mandatory module neither a declaration nor an
  import alias takes a prelude name. UNIT-5 governs declarations that share a
  name. An `impl fn` adds a candidate to its operator (GEN-4), not a name. A
  member name is not a binding and may match any name.

Accepted illustration, complete module:

```hgl
module examples.defaults

const fn seed() -> i64 => 7
const fn choose(base: i64, value: i64 = seed()) -> i64 => base + value

test defining_closure {
    assert choose(1) == 8
}
```

The default calls the module's `seed` (UNIT-7); the body reads its
parameters (UNIT-10). The [eval illustration](tests-and-eval.md) calls
`delta_value` with no `use`: it is a prelude name (UNIT-11). Two blocks,
neither enclosing the other, may each bind the same local name.
Deliberate source rejections: a default of `base` in place of `seed()` reads
a formal argument (UNIT-7); a first parameter named `seed`, a local named
`base` in a block body of `choose`, a local of an inner block named for an
earlier local of a block around it, a test helper named `choose`, or any
function, parameter, or local named `delta_value` or for a declaration its
file imports, takes a name already in scope (UNIT-12). Conformance is planned.

## Core boundary

Core supplies the mandatory module and syntax-bound identities. A target's
declared dependency closure supplies extension interfaces; ambient registries
cannot silently add candidates to a portable target.
