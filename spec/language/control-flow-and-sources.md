# Control flow and sources

Status: agreed conditionals, switch design, and phase-dependent independent
iteration. Generator syntax and the shape of a source's body are agreed, as
is the sentence of SOURCE-3 that begins "Agreed:"; the rest of the timing
of a `yield` is proposed.

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
  operation from controlling child execution. A branch started while the
  program runs is the child a temporal `if` or `switch` selects, or one a
  composition loop makes for a member (CTRL-7). An input of a node in it,
  bound to an endpoint that is already valid, counts as modified in the
  cycle the branch starts, published in that cycle or not (TIME-3), and
  its `last_modified` (core's END-6) is that cycle's evaluation time: the
  branch's nodes run once with the values held.
- **CTRL-3.** A consumed temporal if without else supplies a typed never-ticking
  branch, not a default scalar. It holds no value, so while it is selected
  the `if`'s value is not valid (CTRL-13). An outputless if has no
  synthetic result; an omitted false branch performs no conditional work.
- **CTRL-4.** Escaping assignments require a prior enclosing declaration. Every
  path reaching a use supplies a binding or forwards its incoming binding.
  A branch block's locals do not escape. Multiple results preserve their
  separate typed connections; forwarding does not turn them into snapshots or
  feedback.
- **CTRL-5.** Return targets the enclosing HGL function. Work after a conditional
  early return belongs to the non-returning continuation and its lifetime.
  Definite assignment ignores paths that have already returned.
- **CTRL-6.** Switch labels are typed source constants (UNIT-13), compared by
  resolved value. Duplicate values reject, including differently spelled
  equal enum members, and `case 1:` beside `case 0x1:`. Two
  labels are duplicates where `==` of the selector's type holds for them,
  and not by their sameness as constants. There is no fallthrough. An
  unmatched selector uses `default`, and with none fails with the code
  `switch.unmatched`, in composition as in a value function or a hook.
  Enum coverage can establish exhaustiveness; partial coverage is admitted.
- **CTRL-7.** A composition loop over fixed structure receives child connections;
  a loop over ordinary wiring-time data receives values. Dynamic independent
  dictionary/list bodies create one child per member key/index. Update keeps
  the child; removal stops it; reinsertion creates fresh state. A child
  created while the program runs starts as CTRL-2 says. Captured
  temporal connections are bound whole and add no membership.
- **CTRL-8.** During node evaluation, traversal uses current values/child views
  with hook-scoped borrowed lifetime. Lists preserve index order; sets promise
  no ordering. `elements` serves lists/sets; dictionaries/bundles use keys,
  values, or items. Iteration reads what it traverses, so over a `ref`
  parameter it rejects (FN-16).
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
  the condition of an `if` or a `while` is a `bool`: a `bool?` there is
  read, and a null fails the read (ERR-12); a number, or any other type,
  rejects. An `if` whose value is used has the one type its two
  branches' values have, as the entries of a sequence have one (EXPR-9): a
  null branch makes it `T?`, and two types reject. With no `else` its value
  is a `T?`, null where the condition does not hold, which a place that
  expects a `T?` takes and one that expects a `T` reads, failing for that
  null (ERR-12). Where the place expects a type, each branch's value is
  checked against it: a `null` written as a branch where a T is expected
  rejects (TYPE-14). A branch that returns
  from the function gives the `if` no value and takes no part in its type
  (CTRL-5): `if c { return 1 } else { 2 }` is an `i64`.
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
  `for key, value in entries` for each key and value of a dictionary. A
  set and a dictionary promise no order. One
  name over a dictionary, two over a list or a set, and any other type after
  `in` reject; a nullable list, set, or dictionary there is read, and a
  null fails the read (ERR-12). `while condition { ... }` repeats its body
  while the condition holds; the form with no condition is a source's (SOURCE-4) and
  rejects there. A `switch` there selects on a `bool`, `i64`, `str`, or enum
  value. Each label is a constant of the selector's type (CTRL-6): a
  literal, an enum member, a constant (UNIT-13), or a generic value
  parameter. A `switch` is a statement and has no value.
- **CTRL-13.** In composition the condition of an `if` is a `bool`, an
  ordinary value or a connection (CTRL-1). A `bool?` there is read as
  ERR-12 says: an ordinary one where the graph is built, a null failing
  that phase (ERR-5), and a connection of one by the node that reads it,
  which fails in a cycle that publishes a null. Any other type rejects. An `if`
  whose value is used has the one type its branches' values have, and two
  types reject (CTRL-10). It is a connection when its condition is one or
  either branch's value is, and an ordinary value in a branch is then
  wired as a constant (FN-3). Under a temporal condition that connection
  follows the selected branch's result: after a change of branch it is not
  valid where that branch's result holds no value, until the branch
  publishes. With no `else`, under a temporal condition it is a connection
  of `T` that never ticks, and is not valid, while the condition is false
  (CTRL-3): `T`, not `T?`. Under a wiring-time condition a branch that is an
  ordinary value makes it a `T?`, as in a value function, and a branch that
  is a connection makes it a connection of `T` that never ticks where the
  branch is not composed.
- **CTRL-14.** In composition a `var` assigned in a branch of a wiring-time
  `if` is assigned after it as CTRL-11 says. A `var` keeps the category of
  its first value (EXPR-3): a connection assigned to one that holds an
  ordinary value, or the reverse, rejects (EXPR-4). Under a temporal
  condition a `var` declared before the `if` is assigned in a branch only
  where it is a connection: one that holds an ordinary value would have its
  value chosen at runtime, and the assignment rejects. After the `if` the
  `var` is one connection: the branch's binding while that branch is
  selected, and the binding it had before for a branch that does not assign
  it (CTRL-4). With none before, every branch that does not return assigns
  it. A `return value` in a branch of a temporal `if` makes the graph's
  result that value while the branch is selected and the continuation's
  otherwise (CTRL-5): the continuation returns a value too, and one that can
  reach its end with none rejects. Such a `var` and such a result each
  follow the binding selected as the `if`'s value follows its branch's
  result (CTRL-13).
- **CTRL-15.** In composition a `switch` over an ordinary value composes the
  one case selected, and over a connection each case is a child graph
  selected at runtime. The selector is a `bool`, `i64`, `str`, or enum value
  and each label a constant (CTRL-12); a case's assignments and returns
  follow CTRL-14, and over a connection what they give follows the case
  selected as a temporal `if`'s value follows its branch (CTRL-13).
  `for item in items` over an ordinary list, set, or
  dictionary composes its body once for each element, with ordinary values
  bound, and may carry a `var` from one pass to the next. Over a connection
  of `list<T, N>` it composes its body `N` times, `item` a connection of
  `T`; over a connection of a tuple it rejects. Over a connection of
  `list<T>`, `set<T>`, or `dict<K, V>` it makes one child for each member
  (CTRL-7): `item` is a connection of `T` for a list and an ordinary `T`,
  the member itself, for a set; `key` is an ordinary `K` and `value` a
  connection of `V` for a dictionary. In such a loop an assignment to a
  `var` declared outside it rejects (CTRL-8), and so does a `return`. The
  condition of a `while` is an ordinary `bool`, tested while the graph is
  built: a connection there rejects (FN-3), and the form with no condition
  is a source's (SOURCE-4).

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

Accepted declaration fragments, graphs (CTRL-13, CTRL-14, CTRL-15), with
`window`, `scale`, and `log_tick` functions in scope:

```hgl
fn sized(price: f64, const wide: bool) -> f64 {
    if wide { window(price, 20) } else { price }
}

fn gated(price: f64, open: bool) -> f64 {
    if open { price }
}

fn adjusted(price: f64, rising: bool) -> f64 {
    var chosen = price
    if rising {
        chosen = scale(price)
    }
    return chosen
}

fn chained(price: f64, const factors: list<f64>) -> f64 {
    var running = price
    for factor in factors {
        running = running * factor
    }
    return running
}

fn each_price(prices: dict<str, f64>) {
    for name, price in prices {
        log_tick(price)
    }
}
```

`sized` composes one branch while the graph is built: `wide` is a `const`
parameter. `gated` selects at runtime: its result is a connection of `f64`
that never ticks, and is not valid, while `open` is false, though `price`
holds a value (CTRL-13). With `open: bool?` it is accepted too, and the
node that reads `open` fails in a cycle in which it publishes a null
(ERR-12). In `adjusted`, `chosen` is one
connection after the `if`: the scaled price while `rising` holds, and the
price otherwise. `chained` is unrolled while the graph is built, one
multiplication for each factor, and carries `running` from pass to pass.
`each_price` makes one child for each key: `name` is an ordinary `str`, and
`price` a connection of `f64`. Deliberate rejections: `if price { ... }`,
a condition that is no `bool`; `var size = 3` then `size = price`, a
connection assigned to a `var` that holds an ordinary value; `var size = 3`
then `if open { size = 4 }`, an ordinary value chosen at runtime; in
`each_price`, an assignment in the body to a `var` declared before the
loop, and a `return`; and `while open { ... }`, a connection as the
condition of a `while`. Conformance planned.

Accepted illustration, complete module, a branch that starts while the
program runs (CTRL-2):

```hgl
module examples.branch_start

fn counted(value: i64) -> i64 {
    state count: i64 = 0
    when {
        count += 1
        return count
    }
}

fn counted_while(open: bool, value: i64) -> i64 {
    if open { counted(value) }
}
```

`value` publishes 10 in the first cycle and nothing after, and `open`
publishes false, true, false, true. Each true starts the branch fresh:
`value`, valid since the first cycle, counts as modified in that cycle, and
`counted` runs once with 10 and publishes 1; `last_modified(value)` read
there is that cycle's time. While `open` is false the result never
ticks and is not valid: the 1 published before is not held (CTRL-3,
CTRL-13). So
`eval counted_while(open: [false, true, false, true], value: [10, _, _, _])`
is `[_, 1, _, 1]`. Conformance planned.

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

const fn halvings(n: i64) -> i64 {
    var left = n
    var count = 0
    while left > 1 {
        left = left // 2
        count += 1
    }
    return count
}

enum Side { buy, sell }

const fn quoted(side: Side) -> str {
    switch side {
        case .buy: return "bid"
        case .sell: return "ask"
    }
}

const fn side_of(rising: bool) -> Side {
    if rising { .buy } else { .sell }
}
```

`larger` returns the value of its `if`, an `i64`. `positive` returns `a` or
null: an `if` with no `else` is a `T?`. `found` is null until it is given a
value, so `kept` may return it. `let n: i64 = if a > 0 { a }` reads the
`T?` where an `i64` is expected, and fails where `a` is not above zero;
`if flag { 1 } else { 2 }` with `flag` a `bool?` fails where it is null
(ERR-12). Deliberate rejections:
`let n: i64 = if a > 0 { a } else { null }`, a `null` written where an
`i64` is expected;
`if a { 1 } else { 2 }` with `a` an `i64`, a condition that is no bool;
`const fn f(a: i64) -> i64 { if a > 0 { return a } }`, which can reach its
end with no value; reading a `var total: i64` after an `if` with no
`else` that alone assigns it; `for key in entries` with `entries` a
dictionary, which binds two names; `while { }` with no condition; and a
`switch` with `case 1:` twice. `total` adds up a list, `halvings` loops
while a condition holds, and `quoted` returns from every case of an enum it
covers. In `side_of` each branch's value is a member of the enum the
function returns: `.member` is an expression where it is the first
statement of its block (LEX-5).

## Generators

SOURCE-1 is agreed. SOURCE-2 to SOURCE-4, the timing of a `yield`, are
proposed, but for the sentence of SOURCE-3 that begins "Agreed:".

- **SOURCE-1.** A temporal `fn` whose body has a `yield` is a source: a node
  that produces its output and has no temporal parameters. It injects
  `clock`, `logger`, and `global_state`, and not `out`,
  `scheduler`, or `alarm`: `yield time: value` writes its output and sets
  its next evaluation. It declares `cache` and `state` (TIME-6), which are
  what it keeps from one `yield` to the next: a `cache` is rebuilt on every
  start, and a `state` is recorded. A source with no declaration and no
  hook has a body of statements, `yield` among them. One with an `inject`,
  a `cache`, a `state`, a `start`, or a `stop` has its declarations, then
  `start`, then one block that holds every `yield`, then `stop`, and
  nothing else; the block is `{ ... }`, a `while`, or a `for`. Its
  statements may use `const` configuration, `if`, `while`, and `for`. A
  `let` there holds a working value and is evaluated again on each pass of
  a loop; a source declares no `var`. A source has no `when`. `stop`, where
  present, always runs when the node ends, whatever state the generator is
  in: finished, suspended at a `yield`, or not yet begun.
- **SOURCE-2.** The first evaluation begins the body. Yield evaluates its time
  once, then payload once, before target validation. A duration is relative to
  current evaluation time; a datetime is absolute. Negative duration fails.
  Resolve duration addition with checked arithmetic.
- **SOURCE-3.** Targets strictly increase across the invocation, including
  skipped past targets. The first has no predecessor. Agreed: a past
  absolute target is skipped, nothing is published for it and the body
  goes on to its next `yield`, so that a source replaying recorded values
  over a run that starts part way through them gives those from the run's
  start on; a current target publishes now; a future target retains its
  payload and suspends, and on wake it publishes and resumes after the
  `yield`.
- **SOURCE-4.** A bare return or body completion finishes the source. Locals
  survive suspension but are reconstructible, not recorded state. After restore
  the body restarts. A valued return and yield inside a value expression reject.
  While repeats while its condition holds; omitted condition is unbounded.

Accepted illustration of SOURCE-1, complete module:

```hgl
module examples.source

export fn two_values() -> i64 {
    yield 1s: 7
    yield 1s: 9
}

export fn greeting(const every: duration = 1s) -> str {
    inject logger
    start { log_info(logger, "greeting starts") }
    {
        yield every: "hello"
        yield every: "world"
    }
    stop { log_info(logger, "greeting ends") }
}

export fn counter(const every: duration = 1s, const limit: i64 = 3) -> i64 {
    inject logger
    cache count: i64 = 0
    while count < limit {
        count += 1
        yield every: count
    }
    stop { log_info(logger, str(count)) }
}

export fn replay(const values: list<f64>, const every: duration = 1s) -> f64 {
    inject logger
    start { log_info(logger, "replay starts") }
    for value in values {
        let doubled = value * 2.0
        yield every: doubled
    }
    stop { log_info(logger, "replay ends") }
}
```

`two_values` has no declaration and no hook: its body is its statements.
`greeting` has an injectable and both hooks and no loop, so its yields stand
in an anonymous block between them. `counter` keeps `count` from one `yield`
to the next as a `cache`, which `stop` reads. In `replay` the block is a
`for`, and `doubled` is a working value evaluated on each pass. Deliberate
rejections: a `when` in a source; a `yield` in `start` or `stop`; in
`greeting`, a `yield` outside the block, or a second block; `var seen = 0`
in any of them; `inject out`; and a temporal parameter.

Proposed timing in `two_values`: the second delay is relative to the resumed
evaluation, so publications occur one and two seconds after the initial
evaluation. Conformance planned.

## Core boundary

Core supplies child lifecycle, scheduling, captures, and publication. Source
operand ordering and suspension semantics are compiler-language obligations.
