# Source documentation

Status: agreed documentation-comment and declaration-formatting contracts.
This is source-language documentation, distinct from specification authoring.

## Rules

- **SDOC-1.** `/** ... */` immediately before a module, function/native function,
  operator, constant, struct, field, enum, enum member, native type, a
  native type's property, or test documents that declaration. Only whitespace may intervene. Ordinary comments remain
  comments. Repeated/unattached documentation or a body comment claiming a
  later declaration is rejected.
- **SDOC-2.** Write a short summary followed by structured sections. Their
  text is Markdown (SDOC-7). Remove delimiters, outer blank lines, and common
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
  each indented four spaces without intervening blank lines. Keep each line
  break in its written form (LEX-15); an added line break takes the form of
  the source's first line break, or LF when the source has none. Formatting
  twice has the same result.
- **SDOC-6.** Documentation-only changes do not affect execution, overload
  resolution, or provider semantics. Tools may expose math and diagrams but
  must preserve authoritative markup rather than silently reinterpret it.
  An example that runs (SDOC-9) is a test of the program and nothing else
  of it: a change to one changes what the program's tests are, and no more.
- **SDOC-7.** A comment's text is Markdown: CommonMark, with tables. The
  summary and the prose outside a section are Markdown as written. A
  section's contents are Markdown once the section's four spaces are
  removed (SDOC-3), and an entry's description once its key is. A fenced
  block whose info string is `math` holds a formula, written in LaTeX, and
  one whose info string is `mermaid` a diagram, written in mermaid. A tool
  that presents documentation renders both, or shows their text as
  written; it changes neither (SDOC-6). A formula within a line is a
  code span between dollar signs, `` $`a^2 + b^2`$ ``, so that a dollar
  sign in prose is never taken to open one. HTML written in a comment is
  text, shown as written.
- **SDOC-8.** A code span that is a name refers to the declaration the
  name resolves to where the comment stands (UNIT-10): `` `tripled` ``, a
  name with its module, `` `examples.documented.tripled` ``, or a field or
  a member through its owner, `` `Side.buy` ``. A tool that presents
  documentation links it to that declaration. A code span that names no
  declaration is code, and no error: an expression, `` `value + 1` ``, a
  parameter of the declaration documented, or a name nothing declares.
- **SDOC-9.** In an `Examples:` section a fenced block whose info string
  is `hgl` is an example that runs. It is a named test of the comment's
  module, as if a `test` block of that module held its lines (TST-3):
  whatever a named test's block holds it holds, helper functions of its
  own among them (TST-1), and nothing else. It imports nothing: what it
  names is what a named test of its module sees. It is a test of the
  program, checked with it and run with its tests, and its failure is a
  test's (TST-15). It is known by the declaration it documents and its
  place among that comment's examples, counted from 1. A fenced `hgl` block
  anywhere else in a comment is shown, and is neither checked nor run.
  So is one in `Examples:` whose info string is `hgl no-run`.
- **SDOC-10.** A module's documented interface is the declarations it
  exports (UNIT-3), each with its comment. A tool that presents
  documentation presents those, and the rest where it is asked to.

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
Double a value, as `tripled` triples one.

Args:
    value: Input value.

Returns:
    Twice the input, $`2v`$.

Examples:
    ~~~hgl
    assert doubled(2) == 4
    assert doubled(tripled(1)) == 6
    ~~~
*/
const fn doubled(value: i64) -> i64 => value * 2

/** Triple a value. */
const fn tripled(value: i64) -> i64 => value * 3

/**
Apply a function to a value twice.

Examples:
    ~~~hgl
    const fn add_one(x: i64) -> i64 => x + 1
    assert twice(add_one, 1) == 3
    ~~~
*/
const fn twice(f: const fn(i64) -> i64, value: i64) -> i64 => f(f(value))

/** The deepest level a book is kept to. */
export const depth: i64 = 10

/** A side of the order book. */
export enum Side {
    /** The bid side. */
    buy,
    /** The offer side. */
    sell,
}
```

The comments document doubled, tripled, twice, depth, Side, and each
member of Side. A property of a native type takes its comment as a
struct's field does: in `export native type Handle { ... }`,
`/** Its size. */` on the line before `size: i64` documents that property. The example of `twice` begins with a helper of its own, `add_one`,
as a named test may (TST-1), which nothing else in the module names. In
the comment of `doubled`, `` `tripled` `` refers to that function (SDOC-8),
`` $`2v`$ `` is a formula (SDOC-7), and the block under `Examples:` is a
test of the module with two assertions, known as the first example of
`doubled` (SDOC-9); its fence is of tildes, which Markdown admits as it
does backticks. Deliberate rejection: `Args:` names a parameter absent
from that declaration. Conformance planned.

## Core boundary

Documentation has no runtime effect. Core declarations use the same comment
contract; documentation extraction and rendering are tool responsibilities.
