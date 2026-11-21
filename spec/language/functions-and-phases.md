# Functions and phases

Status: agreed function roles, classification, defaults, lifting, and
anonymous-function scope. The `ref` mode (FN-16) is agreed. The sentence
of FN-9 that begins "Proposed:" is proposed. FN-17, a parameter that takes
a function, and the pack of FN-8 declared `for` one are agreed, but for
the sentence of FN-17 that begins "Proposed:". FN-18, configuration
parameters, is proposed.

## Grammar

```ebnf
function_decl = { warning_annotation, NL },
                [ "export" | "impl" ], [ "const" ], "fn",
                callable_name, [ generic_parameters ],
                [ configuration_parameters ], function_signature,
                [ requires_clause ], function_body ;
configuration_parameters = "{", configuration_parameter,
                           { ",", configuration_parameter }, [ "," ], "}" ;
configuration_parameter = [ "const" ], IDENTIFIER, ":", type,
                          [ "=", const_expression ] ;
function_signature = "(", [ parameters ], ")", [ "->", [ "ref" ], type ] ;
parameters = parameter, { ",", parameter }, [ "," ] ;
parameter = [ "ref" ], IDENTIFIER, ":", type, [ "=", const_expression ]
          | "const", IDENTIFIER, ":", value_type, [ "=", const_expression ]
          | "var", IDENTIFIER, ":", value_type
          | [ "const" ], IDENTIFIER, ":", pack, [ "=", const_expression ]
          | IDENTIFIER, ":", function_type ;
pack = "...", type, [ cardinality ]
     | "...", "{", type, [ "for", IDENTIFIER, [ "without", without_names ] ], "}",
       [ cardinality ] ;
without_names = IDENTIFIER, { ",", IDENTIFIER } ;
function_type = [ "const" ], "fn", "(", [ function_type_parameters ], ")",
                [ "->", [ "ref" ], type ] ;
function_type_parameters = function_type_parameter,
                           { ",", function_type_parameter }, [ "," ] ;
function_type_parameter = [ IDENTIFIER, ":" ], type | pack ;
function_body = "=>", expression | block ;
function_expr = "fn", "(", [ lambda_parameters ], ")", [ "->", type ],
                ( "=>", expression | block ) ;
lambda_parameters = lambda_parameter, { ",", lambda_parameter }, [ "," ] ;
lambda_parameter = IDENTIFIER, [ ":", type ] ;
```

Parameter role is checked from the containing declaration. The production
admits `const` and a default on a pack, which FN-8 rejects. A `var`
parameter has no default, and is neither `const` nor a pack (FN-15). A
`ref` parameter is neither `const` nor a pack, and FN-16 says which
functions take the mark.
Local source/native declaration matching is defined in the native chapter.

## Rules

- **FN-1.** A `const fn` is a value function. It computes ordinary values for
  one invocation: its parameter and result types are ordinary value types.
  Three kinds of parameter are the exception: one of a capability's type
  (TIME-8); in a native value function, one of `endpoint<T>` (TIME-9); and
  one that takes a function (FN-17), which holds no value.
  It is not inherently pure or compile-time-only. Parameter-level `const`
  instead fixes configuration: a `const` parameter takes what is fixed when
  the graph is built. In composition that is any ordinary value. In a hook
  and in a value function it is a constant expression: of literals, enum
  members, constants (UNIT-13), generic value parameters, and the enclosing
  function's own `const` parameters. A temporal `fn` has temporal non-const
  parameters and a temporal result; an omitted result arrow means outputless.
  A parameter that takes a function is neither temporal nor `const`, in a
  temporal function as in a value function: it is no connection and no
  value, and is fixed where the call is compiled (FN-17).
  An operator is neither: each of its candidates is one or the other (GEN-4).
- **FN-2.** An ordinary temporal body with no node-only construct describes
  composition: the function is a graph. `state`, `cache`, `start`, `when`,
  `stop`, `yield`, or an `inject` of any capability but `global_state`
  classifies that complete body as runtime: the function is a node. A graph
  may inject `global_state` (FN-9). Classification precedes phase
  checking, even if the trigger occurs in an invalid nested position. Generic
  selection, target choice, and optimizer decisions cannot change the role.
  A node's body is its `state`, `cache`, and `inject` declarations and then
  its `start`, `when`, and `stop` hooks, and nothing else: a local that
  every hook reads is a `cache`, and work done once belongs in `start`.
  A source, a body with `yield`, is a node of the other form, whose
  declarations, hooks, and statements SOURCE-1 gives.
- **FN-3.** Iteration and conditionals follow the containing phase. They do
  not alone classify a body as runtime. A composition body cannot read current
  input payloads. There a temporal parameter, and the result of a temporal or
  lifted call, is a connection: the argument of a call, the operand of a
  symbol, a condition, a bound local, or what is returned. Where a constant
  is required, a `switch` label, a `const` parameter's argument, a generic
  argument, a size, a tuple's index, or the condition of a `while`
  (CTRL-15), a connection rejects. An ordinary
  value of type T is accepted where composition expects a connection of T,
  and is wired as a constant, one publication when the graph starts: a
  literal, a constant (UNIT-13), a `const` parameter, or an all-scalar call
  (FN-6).
  A runtime hook can call value helpers but cannot silently
  construct a temporal child call. In a hook a temporal parameter that is
  not `ref` (FN-16) is a value of its declared type, read as in a value
  function, a collection's child
  where it is admitted (TIME-1); TIME-9 has the operations on the endpoint
  itself. Mixed incompatible phase work is rejected.
  A composition body calls any function: a value function called there is
  evaluated during wiring, or lifted when an argument is temporal (FN-6). A
  runtime body calls only value functions, as a value function does, and
  as value code does, a test's outside an `eval` or a `run` and an
  application's statements (TST-3, UNIT-14). Any body calls an operator;
  in a runtime body or a value function the call selects a value
  candidate, and in value code the candidate selected must be one
  (GEN-5).
- **FN-4.** A temporal/value pair may share a name. Composition and eval choose
  the temporal definition if one exists; runtime/value evaluation chooses the
  value definition. A bad selected signature is an error, not fallback to the
  other role. Two definitions of the same exact role remain duplicates.
  An operator's candidates are no such pair: GEN-5 selects among them.
- **FN-5.** `const` is a modifier only in a position admitting that modifier.
  Otherwise it is an ordinary name, including for an operator or function.
  `const(argument)` is an ordinary call resolved by the usual naming and
  phase rules; it has no intrinsic function-role selection behavior.
- **FN-6.** A selected value helper, a `const fn` or a value candidate of an
  operator, with temporal arguments in composition is lifted: temporal
  arguments become inputs; scalar arguments and omitted defaults bind as
  configuration. A supplied temporal argument remains an
  input even when its parameter has a default. Any input notification activates
  the node; FN-11 defines admission and binding. Each admitted evaluation
  publishes its result, including an equal result. An outputless helper lifts
  to a sink. All-scalar calls execute
  directly and invent no source timing. Ordinary validity is not `all_valid`.
  In value code, a test's outside an `eval` or a `run` and an
  application's statements, nothing is lifted (TST-3, UNIT-14).
- **FN-7.** A composition return selects a connection; a runtime return
  publishes and exits the function's current evaluation, including any later
  `when` handlers. Completing a runtime handler without a write produces no
  tick. A value return yields an ordinary
  value. `return` with no value ends a function that declares no result;
  in a temporal function that declares one it ends the evaluation and
  publishes nothing itself: a write already made through `out` stands
  (TIME-7), and with none there is no tick. A `const fn` that declares a
  result returns a value of that type:
  `return` with no value rejects there, as `return value` does in a function
  that declares no result. A body that ends in an expression returns that
  expression, as an arrow body does: `{ value }` and `=> value` are one
  body. In a function that declares no result the expression is a statement
  and its value is dropped; only `return value` rejects there. A `const fn`
  that declares a result ends every path in `return value` or in its body's
  value: one that can reach its end with no value rejects. Anonymous
  functions obtain their parameter/result context from their accepting form,
  which a parameter of a function type is (FN-17);
  inference does not create a general callable value type.
- **FN-8.** Packs capture, in order, the arguments the fixed parameters leave.
  `...T` captures the positional ones: all of type T, or each of its own type
  when T is a declared type pack. `...{Fields}` captures the named ones, each
  under its source name, where `Fields` is a type pack among the generic
  parameters of the same declaration, named bare. Packs have no defaults,
  cannot be const, and follow all fixed parameters. A declaration has at
  most one positional pack and at most one named pack: a second of either
  kind rejects (`pack.duplicate`), since nothing would say which of the
  two an argument is for. Cardinality and
  reflection follow the generic chapter.
  A named pack may be declared for a parameter that takes a
  function (FN-17), `args: ...{As for f}`. It takes the arguments a call
  of `f` takes, positional or named, matched to `f`'s parameters as that
  call's are (EXPR-2, FN-4). The matching settles which parameter each
  argument is for, and nothing of its type. `As` holds, under the name of
  each parameter given an argument, the type of that argument, which
  need not be the parameter's own: the declaration's `requires` says
  what it may be (GEN-17). A parameter that has a default and is given no argument is
  not in `As`. `without` names the parameters the matching leaves out,
  which the declaration gives the function itself: one,
  `args: ...{As for f without key}`, or several, `without key, index`.
  Each name after it is a generic value parameter of the declaration: a
  constant `str`, which is one parameter's name, or a constant
  `set<str>`, which is the names of as many as it holds. A name `f` has
  no parameter for leaves nothing out.
- **FN-9.** Capabilities belong to nodes: a node declares each with `inject`
  and uses it in its hooks, and a source in its body (SOURCE-1). An
  application injects `global_state`, which it uses only to set the run's
  store before its run statement (UNIT-14), and `run_config` (RUN-6), and
  no other. A graph injects
  `global_state` and no other, and uses it where the graph is built: each
  call on it is evaluated once then, in the order written, before any node
  of that graph starts, and is given no connection
  (`connection.as_constant`). A graph built while the run is under way, a
  child of a `switch` (CTRL-15), is built then, and its calls are
  evaluated then. Proposed: so is one core's mapping operators make for a
  member (CTRL-7). A value
  function injects none. What it needs of one is its argument: a value the
  node reads from it, `clock.evaluation_time`, or the capability itself,
  given to a parameter of its type (TIME-8).
  Constructed nodes own their own output and scheduler.
- **FN-10.** An arrow body consumes the complete expression up to its enclosing
  delimiter or significant newline. A comma at the surrounding argument/list
  level ends the body; a comma in a nested expression does not. Existing
  newline-continuation rules still apply. Operators and calls within that
  expression belong to the body. Parenthesize an anonymous function before
  calling it or applying an operation to the function itself: bare, it is a
  whole value, such as an argument or an initializer, and rejects where it is
  called, indexed, reached into, or an operand of an operator on either side.
  These boundaries select the body extent rather than leaving two
  interpretations.
- **FN-11.** A lifted argument with a declared default does not require a valid
  input: bind the default when that input is invalid. Otherwise, an argument
  whose value contract admits null does not require a valid input: bind null
  when it is invalid. Every other input must satisfy ordinary `valid(input)`.
  A valid value, including a valid null, is supplied as published. A
  connection of `T?` given to a parameter of T is read as ERR-12 says: the
  lifted node fails in a cycle in which it publishes a null. A struct,
  a tuple, a fixed list or a window is supplied only where it is
  `all_valid` as well, whatever its default or null contract (FN-12): one
  that is valid and not all valid binds no fallback, its default included,
  and the helper is not invoked. Fallback
  changes admission/binding, not activation, and invents no input tick.
- **FN-12.** Aggregate lifting passes the current ordinary aggregate value to
  one helper invocation, not separate elementwise invocations. The value is
  passed where it may be read whole (TIME-1). An unsized `list<T>` and a
  dictionary need `valid`; lifting adds no required size or
  recursive `all_valid` check. A position of an unsized list that holds no
  value is passed as TIME-1's table reads it, NaN for an `f64`; where the
  element type has no value for it, the invocation fails
  (`value.unset_read`). A struct, a tuple and a fixed-size list need
  `all_valid`, and an optional child that is not valid is passed as null.
  A window is passed only where it is `all_valid`, whatever the
  parameter's default (FN-11): its value is not read before (TYPE-11).
  Unchanged
  inputs supply retained values; they need not publish together.
- **FN-13.** Each invocation observes a stable ordinary argument value or an
  admitted read-only view. Retained results must remain independent of later
  temporal changes. Lifting does not turn a borrowed view into an owned value
  or extend its permitted lifetime.
- **FN-14.** An anonymous function's scope is the scope around it plus its
  parameters (UNIT-12). From the functions and blocks around it, its body
  uses only parameters, generic parameters, `let` locals, and iteration
  bindings. Use of an enclosing `var`, `state`, `cache`, or injected
  capability rejects.
- **FN-15.** A value function, a `const fn` or a native one, may mark a
  parameter `var`: `const fn bump(var count: i64, by: i64)`. So may a value
  candidate of an operator (GEN-6). Its type is an
  ordinary value type, and no capability's (TIME-8) or `endpoint<T>` (TIME-9),
  which no place holds (`parameter.var`). A call gives a `var` parameter a
  place, as the left of an assignment is (EXPR-3): a `var`, a `state`, or a
  `cache`, or a place inside one, that holds an ordinary value of the
  parameter's type. An entry of a dictionary, `counts[key]`, is a place
  for a key the dictionary has; for one it has not, the call fails with
  `value.missing_key`. The parameter is always given: it has no default,
  and is not left out where its type is written `T?` (UNIT-8). What is
  read-only rejects, a `let`,
  a parameter that is not `var`, a constant, or an iteration binding
  (`binding.immutable`); so does what is no place, a literal or what a call
  gives (`argument.place`). The call assigns the place. In the function the
  parameter is a `var`, whose first value is the place's when the call's
  arguments are evaluated (EXPR-2); what it holds when the function returns is
  what the place is given, and a call with several gives each place its value
  in the order of the parameters. A function that fails leaves the place
  holding what its parameter held when it failed: no change it had made is
  undone. The call changes the place and no value: a value taken from the
  place before the call is as it was (TYPE-5). A `var` parameter may be given on to a `var`
  parameter of a function the body calls. A temporal `fn`, a temporal
  candidate, and an operator declare none (`parameter.var`): an operator
  gives the parameters every candidate takes, and each candidate says which
  of its own is `var`. The mark takes no part in selecting a candidate
  (GEN-5): a call of an operator is resolved as any other, and the candidate
  it resolves to says which of its arguments are places. A function or a
  candidate that declares one is not lifted (FN-6): a call of it stands in a
  value function, in a runtime hook, or in composition where the place holds
  an ordinary value. No argument of such a call is a connection
  (`connection.as_constant`), and the function is no target of an `eval`
  (`eval.target`).
- **FN-16.** A node (FN-2, NATIVE-3) may mark a temporal parameter `ref`,
  `ref low: f64`, and its result, `-> ref f64`. The mark is a mode, as
  `var` is (FN-15), and no part of the type (TYPE-12): a caller and the
  type checker see the T after it. A connection of T is given to a `ref`
  parameter, and a `ref` result is a connection of T to whoever reads it.
  A value function, a graph, and an operator mark none, and a mark there
  rejects: an operator gives the parameters every candidate takes, and a
  candidate that is a node marks its own. A `signal` parameter is not
  marked either, and a mark on one rejects: a signal has no payload to
  designate (TYPE-12). Nor does `ref` stand on a window: a `ref`
  parameter or a `ref` result whose type is a `rolling_window` rejects,
  `ref prices: rolling_window<f64, 5>`. The mark takes no part in
  selecting a candidate (GEN-5, GEN-12). A `ref` parameter and its
  children, `xs[i]` and `quote.bid`, are endpoints, not values: the
  endpoint operations take them (TIME-9), `delta_value` aside, and a
  `return` to a `ref` result designates them. A read of the value or of
  the `delta_value` of one rejects when the program is compiled.
  That read rejects with the code `ref.read`, and a mark on what is no
  node, or on a `signal` parameter, with `ref.position`. A mark on a
  window rejects with `ref.position` too. `removed_value` (TYPE-11) of a
  window that is a child of a `ref` parameter or of `out`
  of a `ref` result, a field of a struct given by `ref`, is set aside as
  `delta_value` is: a read of it there rejects with `ref.read`.
  A `return` to a `ref` result is given an expression that names an
  endpoint: a temporal input of the node, `ref` or not, or a child of one,
  `items[idx]`. Returning a whole `ref` parameter samples its current
  designation when the `return` executes; one that holds no reference
  gives the result the empty designation, which designates nothing, and a
  reader of the result is then not valid until a later designation
  (TIME-10). Readers follow that target's
  publications. If the input later designates another target, the result
  keeps the sampled designation until the node writes its result again.
  Returning an ordinary input or a child path designates the logical
  endpoint named by the path. Readers follow a later rebinding of that
  endpoint without waking the node for it. No
  payload is read, so nothing needs admitting (TIME-1), and a child is an
  endpoint whether or not it is valid. A value that names no endpoint
  rejects there (`type.mismatch`). A parameter that is not `ref` may be
  designated too, and a body may both read and designate it. A `ref`
  result designates one endpoint whole: an assignment to a part of its
  `out`, `out.bid = x` or `out[0] = x`, rejects. That assignment rejects
  with the code `binding.immutable`.
  `out = items[idx]` designates as `return items[idx]` does, and is
  rejected where the `return` would be; otherwise `out` of a `ref` result
  is as a `ref` parameter is: the endpoint operations take it,
  `delta_value` aside, and a read of its value rejects.
- **FN-17.** A parameter may take a function. Its type is written
  as a declaration's signature is: `f: fn(f64, f64) -> f64` takes a function
  of two `f64` that gives an `f64`, one composed where a graph is built,
  and `f: const fn(f64) -> f64` a value function. The argument is one of
  three: the name of a declared function; the name of an operator, whose candidate the
  function type's parameter types select (GEN-5); or an anonymous function,
  which takes its parameter and result types from the function type
  (FN-7). A value function given where `fn(...)` is expected is lifted
  there (FN-6); a temporal function given where `const fn(...)` is
  expected rejects. The function given has the function type's parameter
  types, in order, and its result type. Proposed: a parameter the
  function type writes is a plain one, and the function given has a plain
  parameter there. One it marks `var`, `const`, or `ref` does not fit the
  template (`type.mismatch`): the body's call, `f(x)`, could not give it
  the place, the constant, or the designation its mark requires (FN-15,
  FN-1, FN-16). The parameters a named pack stands for (GEN-17) are taken
  as they are, marks included, and a pack declared for the function gives
  each its argument as a call of the function does (FN-8). A generic parameter of the
  function type binds from the function given, as it binds from any
  argument's type (GEN-2). A generic function is given with its generic
  arguments, `scale<f64>`, a name with its generic arguments and no call
  (`function_reference`), which is written only as such an argument, or
  bare, `scale`: given bare, it takes them
  from the function type, where that type's parameter and result types,
  as the call's other arguments have resolved them, decide every one.
  Where they do not, nothing is guessed: the call rejects
  (`generic.unresolved`) and writes the arguments. A function type is a template of the
  signature it takes. Where it writes a parameter's name,
  `fn(key: str, value: f64) -> f64`, the function given has a parameter
  of that name at that position; where it writes none, `fn(str, f64) ->
  f64`, the function's own names are free. A function that does not fit
  the template rejects as a value of another type does (`type.mismatch`).
  The function is known where the call is compiled.
  A function type is written only as a parameter's type, and the argument
  is none of a connection, a binding's value, or anything computed while
  the program runs (`function.argument`). In the body the parameter is
  called as the function it was given, `f(a, b)`, and is given on to
  another parameter that takes a function; it is not read, stored, or
  returned.
- **FN-18.** Proposed. A declaration may declare configuration parameters,
  in braces between its name and its parameters, where a call writes its
  configuration (EXPR-10):
  `const fn rounded{places: i64 = 2}(value: f64) -> f64`. A configuration
  parameter is given only in a call's configuration, by its name,
  `rounded{places: 3}(x)`. No positional argument reaches it, and a named
  argument in the parentheses does not name it (`argument.unknown`). It
  is otherwise a parameter as any other: an ordinary value in a value
  function, a connection in a temporal one, or, marked `const`, what is
  fixed when the graph is built (FN-1), and is read in the body by its
  name. What a call's types, its `requires`, its selection of a
  candidate, or a pack's `without` read is a generic value parameter
  (GEN-6), known when the call is compiled; what only the body reads is
  a configuration parameter. Both are given in the braces.
  One with a default is left out of a call, with its braces where no
  other is given; one with none is given at every call
  (`argument.missing`). None is `ref`,
  `var`, a pack, or one that takes a function, and its name is no other
  parameter's and no generic parameter's of the declaration. Where a
  declaration has a pack declared for a function (FN-8), every argument
  in the parentheses after that function is the function's own, so what
  the declaration itself takes by name is a configuration parameter or a
  generic value parameter (GEN-6), and no name of the function's can
  collide with one of the declaration's. A function that declares a
  configuration parameter with no default does not fit a function type
  (FN-17), which gives it none.

Proposed illustration, complete module, configuration parameters (FN-18):

```hgl
module examples.configuration_parameters

const fn rounded{places: i64 = 2}(value: f64) -> f64 {
    var scale = 1.0
    for step in [1, 2, 3, 4, 5, 6] {
        if step <= places { scale = scale * 10.0 }
    }
    return convert<f64>(convert<i64>(value * scale)) / scale
}

const fn two_places() -> f64 => rounded(3.14159)

const fn three_places() -> f64 => rounded{places: 3}(3.14159)

fn sampled{const every: duration}(value: f64) -> f64 {
    inject scheduler
    state last: f64 = 0.0
    start { schedule(scheduler, every) }
    when modified(value) { last = value }
    when scheduled() && valid([]) {
        schedule(scheduler, every)
        return last
    }
}

fn slow(price: f64) -> f64 => sampled{every: 1s}(price)
```

`two_places()` is 3.14, `places` taking its default, and
`three_places()` is 3.141. `sampled` has a `const` configuration
parameter with no default, fixed when the graph is built, so every call
gives it: `slow` samples `price` each second.
Deliberate rejections: `rounded(3.14159, 3)`, a second positional
argument that no parameter is left to take (`argument.unexpected`);
`rounded(3.14159, places: 3)`, a named argument that names no parameter
of the parentheses (`argument.unknown`); `sampled(price)`, which gives
`every` nothing (`argument.missing`); and
`const fn twice{places: i64}(places: i64) -> i64`, whose configuration
parameter takes the name of a parameter. Conformance planned.

Accepted declaration fragment, a temporal input with a default (UNIT-8):

```hgl
fn scaled(value: f64, factor: f64 = 2.0) -> f64 => value * factor
```

`scaled(price)` wires `factor` as a constant 2.0. `scaled(price, rate)` wires
the connection `rate`. Deliberate source rejection under FN-16: a `const fn`
parameter marked `ref`, `const fn scale(ref value: f64) -> f64`. Executable
conformance is planned.

Accepted declaration fragment, an input that may be omitted (TYPE-14):

```hgl
fn sampled(value: f64, tick: signal?) -> f64 => value
```

`sampled(price)` omits `tick`, an input that is then never valid (UNIT-8).
Deliberate source rejections: a result of type `rolling_window<f64, 5>?`
(TYPE-14);
`tick: signal = 1`, a default a signal cannot take (TYPE-12). Executable
conformance is planned.

Accepted declaration fragments, a `const` parameter of a value function:

```hgl
native type Store
native const fn held(store: Store, const key: str) -> bool

const fn counted(store: Store) -> bool => held(store, "count")
const fn named(store: Store, const key: str) -> bool => held(store, key)
```

`counted` gives the key as a literal, and `named` its own `const` parameter,
which is fixed wherever `named` is called. Deliberate source rejection:
`const fn found(store: Store, key: str) -> bool => held(store, key)`, whose
`key` is a value of one call (`argument.constant`). Executable conformance is
planned.

Accepted declaration fragments, a `var` parameter (FN-15):

```hgl
const fn bump(var count: i64, by: i64) {
    count += by
}

const fn set_first(var items: list<i64>, value: i64) {
    items[0] = value
}

const fn total(values: list<i64>) -> i64 {
    var sum: i64 = 0
    for value in values {
        bump(sum, value)
    }
    return sum
}

const fn kept() -> bool {
    var entries: list<i64> = [1, 2]
    let before = entries
    set_first(entries, 9)
    return before == [1, 2] && entries == [9, 2]
}

const fn fill(var rows: list<i64>, count: i64) {
    append(rows, 1)
    append(rows, 6 // count)
}

operator push<C, V>(values: C, item: V)

impl const fn push(var values: list<i64>, item: i64) {
    append(values, item)
}

impl const fn push(var values: dict<str, i64>, item: str) {
    values[item] = 1
}
```

`bump(sum, value)` assigns `sum`, the caller's `var`. In `kept`, the call
gives `entries` a new value, and `before`, which took the list first, is as
it was. For `var rows: list<i64> = []`, `fill(rows, 0)` fails at its second
`append`, where it divides by zero, and `rows` is then `[1]`: what the first
had added stays. `push` is an operator whose candidates each take a place: the
operator's declaration marks none, `push(entries, 3)` resolves by the types
of its arguments to the candidate for a list, and that candidate's mark makes
`entries` a place. For `var counts: dict<str, i64> = ["a": 1]`,
`bump(counts["a"], 2)` leaves `counts["a"]` 3, and `bump(counts["b"], 2)`
fails with `value.missing_key`, where `counts["b"] = 2` adds the key
(EXPR-3). Deliberate source rejections: `bump(1, 2)`, whose first
argument is no place (`argument.place`); `push([1], 2)`, the same once the
candidate is selected; `bump(fixed, 2)` for `let fixed = 0`
(`binding.immutable`); `fn ticking(var value: i64) -> i64`, a temporal
function, and `operator push<C, V>(var values: C, item: V)`, an operator
(`parameter.var`); and, for
`const fn clear(var value: i64?) { value = null }`, the call `clear()`: a
`var` parameter is given though its type is nullable (`argument.missing`).
Executable conformance is planned.

Accepted illustration, complete module, a `ref` result (FN-16):

```hgl
module examples.routes

fn route<T>(idx: i64, items: list<T, 3>) -> ref T {
    when modified(idx) { return items[idx] }
}

fn follow(value: f64) -> f64 {
    when { return value }
}

fn routed(idx: i64, items: list<f64, 3>) -> f64 => follow(route(idx, items))
```

`when modified(idx)` selects `idx` alone, so `items` is passive: its ticks
do not wake `route`. `return items[idx]` reads no payload: it designates
the endpoint `items[idx]`, which needs no admission, and `items` is not
marked `ref`. A caller sees `route` take a `list<f64, 3>` and give an
`f64`. With `items[0]` and `items[1]` published, and `idx` publishing 1 and
then 2: while `idx` is 1, a tick of `items[1]` reaches `follow` and one of
`items[0]` does not; after the change to 2, `follow` is silent until
`items[2]` first publishes, and then publishes it. `route` is woken for none
of these ticks. Where what `items[1]` is bound to is re-bound upstream,
`follow` observes the new target, and `route` is not woken. With
`ref items: list<T, 3>`, `route` is accepted as well, and a read of `items`
in its body then rejects. Deliberate source rejections: `ref idx: i64` on
`routed`, a graph (FN-16); for
`fn first(items: list<f64, 3>) -> ref f64`, the body
`when all_valid(items) { return items[0] * 2.0 }`, a value where the `ref`
result takes an endpoint (`type.mismatch`); for
`fn firsts(items: list<f64, 3>) -> ref list<f64, 3>` with `inject out`,
the body `when all_valid(items) { out[0] = items[0] }`, a part of `out`
assigned; and `ref tick: signal` on a node's parameter, a signal, which
has no payload to designate (FN-16). Conformance is planned.

Accepted declaration fragments showing modifier and ordinary-name positions:

```hgl
const fn const(value: i64) -> i64 => value
const fn example() -> i64 => const(7)
```

The first `const` modifies each declaration; the declared name and call use
ordinary lookup. `example()` returns 7. Executable conformance is planned.

Accepted expression fragments in an accepting context supplying callable types:

```hgl
apply(fn(x) => x + 1, 7)
(fn(x) => x + 1)(7)
```

In the first, `apply` accepts a unary function and a value: the body ends at
the comma. In the second, the outer call invokes the grouped function and
returns 8 for integer input. Executable conformance is planned.

Accepted declaration fragment, with the same `apply`:

```hgl
const fn shifted(n: i64) -> i64 => apply(fn(x) => x + n, 7)
```

The body reads `n`, a parameter of `shifted` (FN-14); `shifted(1)` returns 8.
Deliberate source rejections: an anonymous-function parameter named `n` takes
a name already in scope (UNIT-12); a body reading a `var` local of the
enclosing function uses a binding FN-14 excludes. Executable conformance is
planned.

Accepted illustration, complete module, parameters that take a function
(FN-17):

```hgl
module examples.function_parameters

const fn apply(f: const fn(i64) -> i64, x: i64) -> i64 => f(x)

const fn add_one(x: i64) -> i64 => x + 1

const fn scale<T>(x: T) -> T requires T in {i64, f64} { return x + x }

const fn three() -> i64 => apply(add_one, 2)

const fn four() -> i64 => apply(scale<i64>, 2)

const fn five() -> i64 => apply(fn(x) => x + 2, 3)

const fn apply_to<V>(f: const fn(V) -> V, x: V) -> V => f(x)

const fn anything<R, ...Ps>(f: const fn(...{Ps}) -> R) -> i64 => 1

const fn six() -> i64 => apply(scale, 3)

const fn seven() -> f64 => apply_to(scale, 3.5)

fn smoothed(price: f64) -> f64 {
    state last: f64 = 0.0
    when {
        last = (last + price) / 2.0
        return last
    }
}

fn twice(f: fn(f64) -> f64, value: f64) -> f64 => f(f(value))

fn calmer(price: f64) -> f64 => twice(smoothed, price)

fn doubled(price: f64) -> f64 => twice(scale<f64>, price)
```

`apply` takes a value function of one `i64`: `three()` is 3, `four()` is
4, with `scale` given its generic argument, and `five()` is 5, the
anonymous function taking `x` as an `i64` from the parameter's type.
`six()` is 6: `scale` is given bare, and `fn(i64) -> i64` leaves only
`i64` for its T. `seven()` is 7.0: `3.5` makes V an `f64`, and so T.
`twice` takes a function composed in a graph: `calmer` composes two nodes
of `smoothed`, each with its own state, and `doubled` lifts the value
function `scale<f64>` twice (FN-6). Deliberate rejections:
`apply(smoothed, 2)`, a temporal function where a value function is
expected; `anything(scale)`, where the function type takes any signature
and so decides nothing of `scale`'s T (`generic.unresolved`), and
`anything(scale<i64>)` is accepted; `apply(fn(x) => "a", 2)`, whose result is no `i64`; and, in a
function with a `let g = add_one`, `apply(g, 2)`, since a binding holds no
function (`function.argument`). A function type that writes a name is a
tighter template: `const fn at_price(f: const fn(price: i64) -> i64, x:
i64) -> i64 => f(x)` takes a function whose parameter is named `price`,
so `at_price(add_one, 2)` rejects, `add_one` naming its parameter `x`
(`type.mismatch`), where `apply(add_one, 2)` is accepted. A function that
marks its parameter does not fit a template that writes a plain one:
with `const fn bump(var count: i64) -> i64`, `apply(bump, 2)` rejects
(`type.mismatch`), since `apply`'s `f(x)` gives no place. Conformance
planned.

| Phase | A temporal argument denotes | Ordinary calls |
| --- | --- | --- |
| Composition | A typed connection | Describe graph work or lift a value helper |
| Runtime hook | A readable admitted payload plus endpoint metadata | Execute value helpers in the current node |
| Value function | Ordinary invocation data | Execute value helpers |
| Value code: a test outside an `eval` or a `run`, an application's statements (TST-3, UNIT-14) | None: no connection is bound there | Execute value helpers; a temporal call rejects |

Accepted eval illustration, complete module:

```hgl
module examples.functions

const fn scale(value: f64, factor: f64 = 2.0) -> f64 => value * factor
fn doubled(value: f64) -> f64 => scale(value)

test lifted_helper {
    assert eval doubled(value: [1.0, _, 3.0]) == [2.0, _, 6.0]
}
```

The function/lifting relationship is agreed. The eval assertion illustrates the
proposed harness contract. Deliberate rejection: a temporal call placed inside
`when`. Conformance is planned.

Accepted declaration fragments, a graph and what it admits of a connection
(FN-3), with `scaled` and `scale` above:

```hgl
fn priced(price: f64, const lots: i64) -> f64 {
    let doubled = scale(price)
    if price > 0.0 { scaled(doubled, 3.0) } else { 0.0 }
}
```

`price` is a connection: `scale(price)` lifts the value helper, `doubled`
names the lifted connection, and `price > 0.0` is a temporal condition.
`3.0` and the `0.0` of the other branch are ordinary values where a
connection is expected, each wired as a constant. `lots` is an ordinary
value. Deliberate source rejections: `switch lots { case price: ... }` and
`pair[price]`, a connection where a constant is required
(`connection.as_constant`).

Accepted FN-11 binding illustration using `scale` above, with temporal inputs
supplied for both parameters. Both endpoints start invalid; a dash denotes no
publication and does not invalidate a retained input:

| Cycle | Value publication | Factor publication | Arguments bound for the invocation |
| --- | --- | --- | --- |
| 0 | 1.0 | — | value 1.0, default factor 2.0; invalid factor does not block admission |
| 1 | — | 4.0 | retained value 1.0, published factor 4.0 |

The first invocation computes 2.0 and the second 4.0. Factor is a supplied
temporal argument, so its publication activates evaluation; it is not omitted
configuration. The ordinary all-required-inputs-valid rule would incorrectly
suppress cycle 0 here.

Accepted binding cases for a parameter whose admitted value contract permits
null, with a non-null default D of that contract when stated:

| Declaration contract | Input at an otherwise admitted notification | Bound argument |
| --- | --- | --- |
| Nullable, default D | Invalid | D |
| Nullable, default D | Valid null publication | null |
| Nullable, no default | Invalid | null |

An explicit declaration illustrating that nullable/default contract is:

```hgl
const fn nullable_echo(value: f64? = 2.0) -> f64? => value
```

Nullability uses TYPE-14's `T?` syntax. It is independent of the function-level
`const` marker. Core owns the [lifting observations](https://github.com/hhenson/hgl_core/blob/main/spec/value-function-lifting.md);
executable conformance is planned.

Accepted FN-12 aggregate illustration, with a temporal unsized list supplied
to this value helper:

```hgl
const fn count_items(values: list<f64>) -> i64 => len(values)
```

The fixture below exposes list membership independently of child payload
validity. The helper reads length only; it never reads an invalid child's
payload. Each admitted row invokes the helper once with the whole list.

| Cycle | Temporal list state | Ordinary valid | all_valid | Invocation / result |
| --- | --- | --- | --- | --- |
| 0 | Invalid parent | false | false | No invocation |
| 1 | Length 2; first child valid, second invalid | true | false | `count_items(list_value)` returns 2 |
| 2 | Second child publishes; length remains 2 | true | true | One invocation returns 2 again |
| 3 | A valid third child joins | true | true | One invocation returns 3 |

Cycle 1 distinguishes ordinary validity from an added full-validity gate;
cycles 2 and 3 distinguish repeated publication and an unsized shape. This
is not an elementwise call to `count_items` with individual floats.

Accepted FN-13 retention illustration, with complete valid list inputs:

```hgl
const fn keep_value(values: list<i64>) -> list<i64> => values
```

| Event | Current input | Independently retained result |
| --- | --- | --- |
| First admitted invocation | `[1, 2]` | R0 is `[1, 2]` |
| Later publication and invocation | `[1, 7]` | R1 is `[1, 7]`; R0 remains `[1, 2]` |

An implementation may use a stable read-only view during the invocation, but
retaining the ordinary result preserves an owning value's independent contents.
Deliberate lifetime rejection, a bounded provider-interface case: a provider
declares a view valid only until the current helper returns; attempting to
retain that same borrowed view beyond the call, without obtaining an owning
value, rejects under FN-13 during lifetime checking. Retaining an admitted
independent owning copy instead is accepted. No new view type or copy API is
implied. Core owns the [concrete observations](https://github.com/hhenson/hgl_core/blob/main/spec/value-function-lifting.md);
executable conformance is planned.

## Core boundary

Core supplies node admission, capabilities, output and source facilities.
Language classification determines which facilities a declaration requires.
Core owns the [lifting observations](https://github.com/hhenson/hgl_core/blob/main/spec/value-function-lifting.md).
