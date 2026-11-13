# Source documentation

Status: agreed documentation-comment and declaration-formatting contracts.
This is source-language documentation, distinct from specification authoring.

## Rules

- **SDOC-1.** `/** ... */` immediately before a module, function/native function,
  operator, struct, field, or test documents that declaration. Only whitespace
  may intervene. Ordinary comments remain comments. Repeated/unattached
  documentation or a body comment claiming a later declaration is rejected.
- **SDOC-2.** Write a short summary followed by structured sections with
  reStructuredText content. Remove delimiters, outer blank lines, and common
  indentation; preserve internal spacing, Unicode, escapes, and relative
  indentation. Normalize each line break (LEX-15) to LF. A same-line summary
  does not set subsequent indentation. Do not decorate every source line with
  `*`.
- **SDOC-3.** Section headings start at column zero after common-indent removal;
  contents are indented four spaces. Unknown headings remain prose. Structured
  keys must name real parameters, generics, requirements, or property domains.
  Missing descriptions are permitted. Documentation declares no new type,
  capability, property, constraint, or runtime behavior.
- **SDOC-4.** The public declaration owns public behavior. Selected implementations
  and parts may have their own documentation but cannot replace the interface's
  text. Preserve canonical identity, signature, source association, part, and
  text across checking and documentation extraction; overloads remain distinct.
- **SDOC-5.** A formatting change preserves parsed meaning, attachment, literal
  content, and comments. Separate definitions by one blank line; keep imports
  together. Requires/properties clauses stay attached to their declaration,
  each indented four spaces without intervening blank lines. Preserve source
  line-ending style; formatting twice has the same result.
- **SDOC-6.** Documentation-only changes do not affect execution, overload
  resolution, or provider semantics. Tools may expose math and diagrams but
  must preserve authoritative markup rather than silently reinterpret it.

| Section | Contents |
| --- | --- |
| `Args:` / `Type Args:` | Descriptions keyed by declared parameter/generic names |
| `Returns:` / `Raises:` | Result meaning and required failure conditions |
| `Ticks:` / `Validity:` | Publication/admission observations |
| `Properties:` / `Requires:` | Explanations keyed by declared contracts |
| `Notes:` / `Examples:` | Additional prose, examples, math, and diagrams |

Accepted illustration, complete module:

```hgl
module examples.documented

/**
Double a value.

Args:
    value: Input value.

Returns:
    Twice the input.
*/
const fn doubled(value: i64) -> i64 => value * 2
```

The comment documents doubled. Deliberate rejection: `Args:` names a parameter
absent from that declaration. Conformance planned.

## Core boundary

Documentation has no runtime effect. Core declarations use the same comment
contract; documentation extraction and rendering are tool responsibilities.
