# Units and names

Status: agreed module, visibility, and default-binding rules. General lookup
precedence is retained as a proposed baseline below.

## Grammar

```ebnf
source_file = [ NL ], module_decl, { NL, top_level_item }, [ NL ], EOF ;
module_decl = "module", module_path, [ "part", IDENTIFIER ] ;
module_path = IDENTIFIER, { ".", IDENTIFIER } ;
qualified_name = IDENTIFIER, "::", import_name ;
use_decl = "use", module_path, ( "::", import_set | "as", IDENTIFIER ) ;
import_set = "{", import_name, { ",", import_name }, [ "," ], "}" ;
import_name = IDENTIFIER | "const" ;
top_level_item = use_decl | declaration ;
declaration = function_decl | struct_decl | enum_decl | operator_decl
            | instantiate_decl | native_type_decl | native_function_decl
            | test_decl | test_context ;
member_name = IDENTIFIER | "time" ;
callable_name = IDENTIFIER | "const" ;
```

The referenced declarations belong to their type, function, generic, native,
or test chapters. Portable source is covered here; target build metadata is not.

## Agreed rules

- **UNIT-1.** A file names one module and optionally one part. An assembled
  module has one declaration scope, at most one unnamed file, and unique part
  names. Every file names the same module. Part names do not enter declaration
  identity; assembly order does not change meaning.
- **UNIT-2.** Imports precede ordinary declarations. A selective import binds
  public declarations; an alias qualifies a public interface with `alias::name`.
  A dotted module path creates no privileged access to another module.
- **UNIT-3.** Ordinary functions are private unless exported. Operators and
  their candidates are public by definition. Exported signatures are closed
  over reachable types and required defining metadata; private dependencies do
  not become source-importable. Import cycles are rejected.
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

Accepted illustration, complete module:

```hgl
module examples.defaults

const fn seed() -> i64 => 7
const fn choose(seed: i64, value: i64 = seed()) -> i64 => seed + value

test defining_closure {
    assert choose(1) == 8
}
```

The default calls the enclosing function; the body reads its parameter.
Deliberate rejection: replacing `seed()` with another formal argument's name
cannot read that argument through a default. Conformance is planned.

## Proposed ordinary lookup

Innermost earlier local/iteration binding, then parameters/generics, module
declarations, selective imports, and the prelude. Inner bindings may shadow
outer bindings. Direct capability receivers select their capability operation;
reserved activation forms are not shadowable identifiers. This ordering grants
no payload read during composition and does not change default binding.

## Core boundary

Core supplies the mandatory module and syntax-bound identities. A target's
declared dependency closure supplies extension interfaces; ambient registries
cannot silently add candidates to a portable target.
