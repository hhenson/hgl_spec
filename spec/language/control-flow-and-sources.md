# Control flow and sources

Status: agreed conditionals, switch design, and phase-dependent independent
iteration. Generator syntax is agreed; its detailed source contract is proposed.

## Grammar

```ebnf
block = "{", [ NL ], [ statement, { NL, statement }, [ NL ] ], "}" ;
statement = local_decl | assignment | return_statement | expression
          | state_decl | cache_decl | inject_decl | lifecycle_block
          | when_statement | for_statement | while_statement | yield_statement
          | switch_statement | assert_statement ;
return_statement = "return", [ expression ] ;
if_expression = "if", expression, block, [ "else", ( block | if_expression ) ] ;
for_statement = "for", iteration_pattern, "in", expression, block ;
iteration_pattern = IDENTIFIER, [ ",", IDENTIFIER ] ;
while_statement = "while", [ expression ], block ;
yield_statement = "yield", expression, ":", expression ;
switch_statement = "switch", expression, "{", [ NL ],
                   { switch_case, [ NL ] }, [ switch_default, [ NL ] ], "}" ;
switch_case = "case", const_expression, ":", [ NL ],
              [ statement, { NL, statement } ] ;
switch_default = "default", ":", [ NL ], [ statement, { NL, statement } ] ;
```

Case labels delimit case bodies; a case has no scope of its own (UNIT-12).
Function/phase rules reject misplaced hooks
or statements even where this common block grammar can recognize them.
CTRL-9 places the opening brace of an `if`, `for`, `switch`, `when`, or
`while` body.

## Agreed rules

- **CTRL-1.** A wiring-time condition chooses which branch to compose. A
  temporal condition in composition selects a child graph at runtime. A value
  or node-hook condition executes ordinary local control flow. Conditional
  syntax does not classify a function as a node.
- **CTRL-2.** Only the selected temporal child runs. A changed selector stops
  the old child and starts the selected child fresh. Repeated equal selector
  values do not recreate it. Producers composed outside the conditional keep
  their own lifetime. Selecting between pre-existing outputs is a different
  operation from controlling child execution.
- **CTRL-3.** A consumed temporal if without else supplies a typed never-ticking
  branch, not a default scalar. An outputless if has no synthetic result; an
  omitted false branch performs no conditional work.
- **CTRL-4.** Escaping assignments require a prior enclosing declaration. Every
  path reaching a use supplies a binding or forwards its incoming binding.
  A branch block's locals do not escape. Multiple results preserve their
  separate typed connections; forwarding does not turn them into snapshots or
  feedback.
- **CTRL-5.** Return targets the enclosing HGL function. Work after a conditional
  early return belongs to the non-returning continuation and its lifetime.
  Definite assignment ignores paths that have already returned.
- **CTRL-6.** Switch labels are typed source constants, compared by resolved
  value. Duplicate values reject, including differently spelled equal enum
  members. There is no fallthrough. An unmatched selector uses default or fails.
  Enum coverage can establish exhaustiveness; partial coverage is admitted.
- **CTRL-7.** A composition loop over fixed structure receives child connections;
  a loop over ordinary wiring-time data receives values. Dynamic independent
  dictionary/list bodies create one child per member key/index. Update keeps
  the child; removal stops it; reinsertion creates fresh state. Captured
  temporal connections are bound whole and add no membership.
- **CTRL-8.** During node evaluation, traversal uses current values/child views
  with hook-scoped borrowed lifetime. Lists preserve index order; sets promise
  no ordering. `elements` serves lists/sets; dictionaries/bundles use keys,
  values, or items. Iteration does not cross REF boundaries implicitly.
  Dynamic composition loops do not admit loop-carried accumulation or inferred
  reductions. Conditions inside a loop retain their own phase rules.
- **CTRL-9.** After `if`, `for`, `switch`, `when`, or `while`, the first `{`
  outside parentheses, brackets, and operator configuration (EXPR-10) opens
  the body. The expression before it is the condition of an `if`, `when`, or
  `while`, the sequence of a `for`, or the selector of a `switch`; `when` and
  `while` may have none. An expression there that contains any other brace is
  parenthesized.
- **CTRL-10.** A block whose last statement is an expression has that
  expression's value and type; any other block has none. In a value function
  the condition of an `if` is a `bool`: a `bool?`, a number, or any other
  type rejects. An `if` whose value is used has the one type its two
  branches' values have, as the entries of a sequence have one (EXPR-9): a
  null branch makes it `T?`, and two types reject. With no `else` it is as
  if its `else` were null: its value is a `T?`, which a place that expects a
  `T?` takes and one that expects a `T` rejects. Where the place expects a
  type, each branch's value is checked against it.
- **CTRL-11.** In a value function, a `var` declared with no value is
  assigned after an `if` when every branch that does not return assigns it,
  so not by an `if` with no `else`; after a `switch` when every case that
  does not return assigns it and the cases leave no value out, with a
  `default` or by covering an enum (CTRL-6); and never by a loop's body,
  which may run no times. A `var` whose type is written `T?` is null where
  it is declared with no value, as an omitted parameter of that type is
  (UNIT-8): it is assigned from there, and after a loop holds what the loop
  last gave it, or null.
- **CTRL-12.** In a value function, `for item in items` runs its body for
  each element of a list, in index order, or of a set, and
  `for key, value in entries` for each key and value of a dictionary. One
  name over a dictionary, two over a list or a set, and any other type after
  `in` reject. A `switch` there selects on a `bool`, `i64`, `str`, or enum
  value. Each label is a constant of the selector's type (CTRL-6): a
  literal, an enum member, or a generic value parameter. A `switch` is a
  statement and has no value.

Accepted illustrations, statement fragments with `ready` a bool, `items` a
list, and `step` a function in scope:

| Source | Reading |
| --- | --- |
| `when { step() }` | No condition; body `{ step() }` |
| `when ready { step() }` | Condition `ready` |
| `when ({ ready }) { step() }` | The block `{ ready }` is the condition |
| `if ready { step() }` | Condition `ready` |
| `if ({ ready }) { step() }` | The block `{ ready }` is the condition |
| `for item in items { step() }` | Sequence `items` |

Deliberate rejections, in parsing: `when { ready } { step() }`, where the
first block is the body, so a second statement follows on its line (LEX-6);
and `if { ready } { step() }`, which has no condition before its first `{`.

Accepted illustration, complete module:

```hgl
module examples.branch

fn choose(condition: bool, left: i64, right: i64) -> i64 {
    if condition { left + 1 } else { right - 1 }
}
```

Only the selected child computes. Deliberate rejection: using a temporal input
as a switch case constant. Conformance planned.

Accepted declaration fragments, value functions (CTRL-10, CTRL-11, FN-7):

```hgl
const fn larger(a: i64, b: i64) -> i64 {
    if a > b { a } else { b }
}

const fn positive(a: i64) -> i64? {
    if a > 0 { a }
}

const fn kept(a: i64) -> i64? {
    var found: i64?
    if a > 0 {
        found = a
    }
    return found
}

const fn total(items: list<i64>) -> i64 {
    var running = 0
    for item in items {
        running += item
    }
    return running
}

enum Side { buy, sell }

const fn quoted(side: Side) -> str {
    switch side {
        case .buy: return "bid"
        case .sell: return "ask"
    }
}
```

`larger` returns the value of its `if`, an `i64`. `positive` returns `a` or
null: an `if` with no `else` is a `T?`. `found` is null until it is given a
value, so `kept` may return it. Deliberate rejections:
`let n: i64 = if a > 0 { a }`, a `T?` where an `i64` is expected;
`if a { 1 } else { 2 }` with `a` an `i64`, a condition that is no bool;
`const fn f(a: i64) -> i64 { if a > 0 { return a } }`, which can reach its
end with no value; reading a `var total: i64` after an `if` with no
`else` that alone assigns it; `for key in entries` with `entries` a
dictionary, which binds two names; and a `switch` with `case 1:` twice.
`total` adds up a list, and `quoted` returns from every case of an enum it
covers.

## Proposed generators

- **SOURCE-1.** A temporal fn containing yield is an output-producing source
  with no temporal parameters. It may use const configuration, locals, clock,
  logger, if, and while. State/cache, output/scheduler/alarm injection, lifecycle
  hooks, when, and for are excluded from this bounded generator profile.
- **SOURCE-2.** The first evaluation begins the body. Yield evaluates its time
  once, then payload once, before target validation. A duration is relative to
  current evaluation time; a datetime is absolute. Negative duration fails.
  Resolve duration addition with checked arithmetic.
- **SOURCE-3.** Targets strictly increase across the invocation, including
  skipped past targets. The first has no predecessor. A past absolute target
  is skipped; a current target publishes now; a future target retains its
  payload and suspends. On wake it publishes and resumes after yield.
- **SOURCE-4.** A bare return or body completion finishes the source. Locals
  survive suspension but are reconstructible, not recorded state. After restore
  the body restarts. A valued return and yield inside a value expression reject.
  While repeats while its condition holds; omitted condition is unbounded.

Proposed illustration, complete module:

```hgl
module examples.source

export fn two_values() -> i64 {
    yield 1s: 7
    yield 1s: 9
}
```

The second delay is relative to the resumed evaluation: publications occur one
and two seconds after the initial evaluation. Conformance planned.

## Core boundary

Core supplies child lifecycle, scheduling, captures, and publication. Source
operand ordering and suspension semantics are compiler-language obligations.
