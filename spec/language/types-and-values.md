# Types and values

Status: agreed type foundation, `delta<T>` (TYPE-16) and `tenor` (TYPE-17)
included. The storage extensions below remain proposed.

## Grammar

```ebnf
type = type_base, [ "?" ] ;
type_base = scalar_type | named_type | tuple_type | list_type | set_type
     | dict_type | rolling_window_type | ref_type | "signal" | endpoint_type
     | "atomic", "<", value_type, ">" | result_type | "error" | delta_type ;
value_type = value_type_base, [ "?" ] ;
value_type_base = scalar_type | named_type | value_tuple_type | value_list_type
           | set_type | value_dict_type | result_type | "error" | delta_type ;
scalar_type = "bool" | "i64" | "f64" | "str" | "bytes" | "any"
            | "date" | "time" | "datetime" | "duration" | "civil_datetime"
            | "timezone" | "zoned_datetime" | "zoned_time" | "tenor" ;
named_type = ( IDENTIFIER | qualified_name ), [ generic_arguments ] ;
tuple_type = "tuple", "<", type, { ",", type }, ">" ;
value_tuple_type = "tuple", "<", value_type, { ",", value_type }, ">" ;
list_type = "list", "<", type, [ ",", size_expression ], ">" ;
value_list_type = "list", "<", value_type, [ ",", size_expression ], ">" ;
size_expression = additive_expr | "unbounded" ;
set_type = "set", "<", value_type, ">" ;
dict_type = "dict", "<", value_type, ",", type, ">" ;
value_dict_type = "dict", "<", value_type, ",", value_type, ">" ;
rolling_window_type = "rolling_window", "<", value_type, ",", const_expression,
                      [ ",", const_expression ], ">" ;
ref_type = "ref", "<", type, ">" ;
endpoint_type = "endpoint", "<", type, ">" ;
result_type = "result", "<", value_type, ",", value_type, ">" ;
delta_type = "delta", "<", type, ">" ;
struct_decl = [ "export" ], [ "abstract" ], "struct", IDENTIFIER,
              [ generic_parameters ], [ ":", named_type, { ",", named_type } ],
              [ requires_clause ], "{", [ NL ],
              [ struct_member, { NL, struct_member }, [ NL ] ], "}" ;
struct_member = member_name, ":", type, [ "=", const_expression ]
              | member_name, "=", const_expression ;
enum_decl = [ "export" ], "enum", IDENTIFIER, "{", [ NL ],
            [ enum_member, { ",", [ NL ], enum_member }, [ "," ] ], [ NL ], "}" ;
enum_member = member_name, [ "=", const_expression ] ;
```

Semantic field rules distinguish a typed introduction from an inherited-default
override. The productions admit `?` on a set's element and a dictionary's key,
which TYPE-14 rejects. Where a production gives `value_type`, any `type` is
admitted as syntax, and one that is no ordinary value type is a type
rejection (TYPE-12, TYPE-14). TYPE-16 says what a `delta_type` is.

## Agreed value rules

- **TYPE-1.** Ordinary values have canonical types. `bool`, signed 64-bit `i64`,
  binary64 `f64`, Unicode `str`, and octet-sequence `bytes` retain distinct
  identities. Key/equality/order operations require the type's capabilities.
  Bytes admit values 0 through 255; length and ordering use unsigned octets.
- **TYPE-2.** Temporal scalar domains are: Gregorian `date` (years 1–9999),
  microsecond `time`, UTC-instant `datetime`, elapsed `duration`, local
  `civil_datetime`, named `timezone`, resolved `zoned_datetime`, and
  date-independent `zoned_time`. Instants/durations use signed 64-bit
  microseconds; representable literal instants retain canonical calendar range.
  Same-instant zoned values need not be equal; zone aliases are distinct names.
- **TYPE-17.** A `tenor` is a rule for moving a `date`, and no elapsed time.
  It is a count of months and a count of days, or it is a count of business
  days; each count is a signed 64-bit integer. Two tenors are equal when they
  are of the same form and their counts are equal, however each was written:
  `1Y` is `12M` and `1W` is `7D`, `1M` is not `30D`, and `0B` is not `0D`.
  The core contract gives a tenor's operators, how each form moves a date,
  and what a business day is.
- **TYPE-3.** Structs are nominal, with public ordered fields and no methods.
  Only abstract structs may be parents, each listed once; abstract values are
  not constructible. Concrete structs are final. Generic applications are
  invariant by origin and complete canonical arguments. A struct is a value
  where an abstract struct it inherits from is expected, through any number
  of parents, each with the arguments the struct below gives it; a
  `list<Child>` is no `list<Base>`.
- **TYPE-4.** A typed field introduction fixes its type and optionality. A
  field is optional when its type is nullable, `T?`, or an `atomic<...>` of
  a nullable type, `atomic<T?>`, and only an optional field's default may
  be null. A field a struct has from a generic parent has the type that
  parent's arguments give it: with `abstract struct Base<T> { value: T }`,
  `value` is optional in `struct Child : Base<i64?>`, which may write
  `value = null`. So has a struct's own field, with the arguments the
  struct is given: the `value: T` of a `Box<i64?>` is optional. A type
  parameter written with no `?` is not nullable:
  `value: T = null` rejects. An untyped inherited override changes only the
  construction default of a field a parent has. Required fields may have
  defaults; optional fields may have non-null defaults. A struct's members
  have distinct names. A field a struct has from two parents, or from a
  parent and itself, has one type and optionality, a generic parent's
  arguments standing for its parameters; a conflict is an error. One field
  reached by two paths is one field.
- **TYPE-5.** An ordinary constructor supplies complete required data. Its
  arguments are named, each for a field the struct has, a parent's included,
  in any order; it takes no positional argument. A field with a default, its
  own or one a struct below gives it, may be left out, and then has that
  default. An optional field with no default may be left out too, and is
  then null. Fields are declared in any order. Written arguments evaluate
  once in source order; failure prevents later arguments. Defaults are
  evaluated only when omitted. Retained results own their data; later
  mutation of a source cannot change an earlier retained value.
- **TYPE-6.** A struct holds the structs its fields' types name at any depth,
  the fields it has from its parents included, and whatever each of those
  structs holds in turn, a generic one with its arguments in place. A field
  whose type is an abstract struct holds every struct that inherits from it.
  A field is recursive when it holds its own struct: in `struct A { b: B }`,
  `struct B { c: C }`, `struct C { a: A }` each of the three fields is.
  Recursive fields stay within their defining module. So that the recursion
  can end, a recursive field has a value that holds no struct of its cycle.
  It is optional (TYPE-4), `Node?`, `atomic<Node?>`, or `atomic<Node>?`, and
  may be null; or, inside an `atomic<...>`, it holds the struct only as
  elements of a collection that may be empty, an unsized list, a list of
  size zero, a set, or a dictionary: `atomic<list<Tree>>`. A list of any
  other size and a tuple do not end it. A recursive field's default, its own
  or a descendant's, is such a value: `null`, or the empty collection `[]`.
  Descendants cannot remove a null default it has, not for an empty
  collection either. It gives a struct of its cycle each generic parameter
  whole, `Node<T>`, and no type built from one, `Node<list<T>>`. Cycles
  through inheritance are rejected, and so are cycles through collection
  elements outside an `atomic<...>`. Cross-module references to
  non-recursive public structs remain admitted.
- **TYPE-7.** Enum members are distinct typed constants with distinct names.
  `Type.member` names one, through an alias as `alias::Type.member`. Where a
  position expects
  the enum's type, `.member` names one too: the value of an argument or
  configuration option of that type, an initializer or result declared with
  it, a value assigned to a `var` of it, an element of a tuple, list, or set
  expected to hold it, an operand compared with it, a `switch` label for a
  selector of it.
  With no enum type expected, `.member` rejects. Unnumbered numbering starts
  at zero and increments the previous member; explicit numbers reset the
  next value. Values fit signed `i64`; duplicates and overflow reject.
  A call of the enum's name constructs a member. It takes one argument, by
  position, that is an `i64` or a `str`, and any other type rejects:
  `Side(1)` is the member whose number is 1, and `Side("buy")` the member of
  that exact name. Construction is checked: a constant that is no member's
  number or name rejects, and a value that is none fails where it is
  evaluated. String conversion returns the member name. Numeric conversion
  is explicit.
- **TYPE-8.** `any` owns a boxed value with its exact contained type; boxing
  adds no numeric conversion. Equality across contained types is false.
  Operations needing an absent capability fail rather than inventing an
  order/hash. Empty and populated boxes remain distinct. Container and retained
  copies preserve independent owned children.
- **TYPE-14.** `T?` denotes an ordinary value that is either a T or null, where
  T is an admitted ordinary value type. It may annotate parameters, results,
  bindings, and admitted container element/value positions. A set's element
  and a dictionary's key are never null and take no `?`. Where T is no
  ordinary value type, as `rolling_window<f64, 5>`, `atomic<Quote>`,
  `ref<f64>`, and `signal` are not, `T?` is admitted only as the whole type
  of a function's temporal parameter, whose argument may be omitted (UNIT-8), or
  of a struct field, which is then optional (TYPE-4); `signal` is no field
  with `?` or without (TYPE-12). It is no result type, local's annotation,
  or part of another type. Nullability is
  independent of a callable's temporal/value role and of fixed configuration.
  A T is a value where a `T?` is expected. `null` has the nullable type its
  place expects, and rejects where nothing gives it one: `let x = null`.
  It does not make null an admitted value of unqualified T. Payload use follows
  [nullable refinement](errors-and-diagnostics.md#nullable-value-refinement).
- **TYPE-15.** The suffix applies to the complete preceding type: `list<f64?>`
  has nullable elements; `list<f64>?` is a nullable list value. A present null
  is ordinary data, distinct from an invalid temporal input or silence.
  Nullable/default lifted argument binding follows FN-11. A type-position `?`
  declares nullability; expression-position `?` retains ERR-5's result
  propagation meaning. Field requiredness/defaults still follow TYPE-4.

## Agreed temporal shapes

- **TYPE-9.** Temporalization recursively maps tuples/structs to child fields,
  lists to child positions, dictionaries to keyed children, and sets to
  membership. `atomic<V>` stops expansion at a complete ordinary V. For
  non-composite S, `atomic<S>` normalizes to S before matching, recursively.
- **TYPE-10.** `list<T>` denotes a growing shape; fixed sizes are nonnegative
  integer constants. A generic value parameter declared with a type other
  than `i64` is no size; one declared with none is a size (GEN-13).
  `unbounded` is the size sentinel. Size participates in identity;
  an unsized parameter pattern can accept a fixed size. A tuple retains
  positional identity and is not a fixed list alias.
- **TYPE-11.** `rolling_window<V, Max, Min>` is a rolling window of V. Its
  maximum and minimum have one constant kind: tick count or duration. A size
  that is a generic value parameter has its declared type's kind: `i64` a
  tick count, `duration` a duration. One declared with no type has its kind
  once resolved. Omitted minimum equals maximum: `rolling_window<i64, 5>` and
  `rolling_window<i64, 5, 5>` are one type. Tick sizes are positive; duration
  maximum is positive and minimum may be zero: `rolling_window<f64, 5m, 0s>`
  is admitted and `rolling_window<f64, 0s>` rejects. Minimum cannot exceed
  maximum. Size/kind are part of identity; window publications obey core's
  contract.
- **TYPE-12.** `ref<T>` designates an endpoint, not an ordinary payload type.
  It is excluded from const values, atomic payloads, keys, and window values.
  An ordinary value type is written with no `rolling_window`, `ref`,
  `atomic`, or `signal` at any depth: `set<list<ref<f64>>>` rejects. A
  dictionary's value and the argument of `ref` or `delta` are any type.
  `signal` is a complete temporal input constraint with no readable payload:
  the whole type of a function's temporal parameter, or of its positional
  pack, `ticks: ...signal`. It is not a result, nested shape, field, const
  value, local's annotation, or anonymous function's parameter. A `signal`
  input takes no default but `null`.
  `endpoint<T>` is the endpoint of a temporal value of T, for a function
  that takes the endpoint and not its payload (TIME-9). It is the whole type
  of a parameter of a native value function, and of nothing else.
- **TYPE-13.** Result and error envelopes are ordinary atomic values; their
  [tag/refinement rules](errors-and-diagnostics.md) govern payload access.
  Silence, invalidity, null absence, and an error outcome are distinct.
- **TYPE-16.** `delta<T>` is the ordinary value type of what a temporal value
  of T publishes in one cycle. The table gives it for each T. For most T it
  is another type the language has: `delta<f64>` is `f64`, and
  `delta<list<f64>>` is `dict<i64, f64>`, a dictionary as any other. A
  nullable's, a set's, a dictionary's, and a struct's delta is a type of
  its own, which keeps T whole in its identity: `delta<dict<str, f64>>` and
  `delta<dict<str, i64>>` are two types. Such a delta is no T: a held
  complete value is not a delta, and neither converts to the other. It is
  atomic: it has properties, none is assigned, and as a temporal value it
  publishes whole, so its own delta is itself. `delta<T>` is written
  wherever an ordinary value type is: an annotation, a parameter or result
  of a value function, a field, an element. In a generic declaration it
  stays as written until T is known. A delta is a value like any other: it
  is bound, stored, given as an argument, and returned. One that is a type
  of its own is constructed by its properties, by name:
  `delta<set<i64>>(added: [1, 2], removed: [3])`,
  `delta<dict<i64, i64>>(updated: [7: 10], removed: [9])`,
  `delta<Quote>(bid: 1.0)`, `delta<i64?>(value: 7)`. A property left out is
  empty, or null for a struct's field and a nullable's `value`. A list's
  and a tuple's delta are an ordinary dictionary and tuple, and are written
  as those are. A dictionary's delta names its changes `updated`, and not
  `modified`, which is the endpoint operation. A nullable's delta is an
  optional: it holds what was published, a null among them, so that null
  in a delta's place always means that nothing was published there.
  `delta<i64?>()` is a published null. In a tuple's position or a struct's
  field, null leaves the position or the field as it was, and
  `delta<i64?>()` sets it to null. A `delta<V>` is a value where a
  `delta<V?>` is expected, as a T is where a `T?` is (TYPE-14).

| T | `delta<T>` |
| --- | --- |
| A scalar, an enum, a `result`, an `error`, a native type, a delta that is a type of its own | T itself |
| `V?`, V an ordinary value type | A type of its own, an optional, with the property `value`, a `delta<V>?`: null for a published null (TYPE-15) |
| `V?`, V no ordinary value type, as `rolling_window<f64, 5>?` | `delta<V>`: the `?` marks an argument or a field that may be left out (TYPE-14), and is no part of what is published |
| `atomic<V>` | V: an atomic publishes whole |
| `rolling_window<V, Max, Min>` | V: the value that arrived |
| `list<V>`, `list<V, N>` | `dict<i64, delta<V>>`: for each position that changed, its delta |
| `tuple<A, B>` | `tuple<delta<A>?, delta<B>?>`: each position's delta, null where it did not change |
| `set<K>` | A type of its own, with the properties `added` and `removed`, each a `set<K>` |
| `dict<K, V>` | A type of its own, with the properties `updated`, a `dict<K, delta<V>>` of each key added or changed with its delta, and `removed`, a `set<K>` |
| A struct | A type of its own, with a property for each field `name: F`, of the same name and of type `delta<F>?`: null where the field did not change |
| `ref<V>`, `signal` | None: `delta.unsupported_shape` |

Accepted illustration, complete module:

```hgl
module examples.types

struct Quote {
    bid: f64
    ask: f64
}

fn quote(bid: f64, ask: f64) -> Quote {
    when { return Quote(bid: bid, ask: ask) }
}
```

The result is a structural temporal Quote. `atomic<Quote>` instead denotes
whole snapshots. Deliberate rejection: a concrete struct used as a parent.
Conformance is planned.

Accepted illustration of `delta<T>` (TYPE-16), complete module:

```hgl
module examples.deltas

export struct TimedValue<T> {
    time: datetime
    value: delta<T>
}

const fn arrived(change: delta<set<str>>) -> set<str> => change.added

const fn joined(name: str) -> delta<set<str>> => delta<set<str>>(added: [name])

const fn cleared() -> delta<f64?> => delta<f64?>()

fn arrivals(members: set<str>) -> atomic<set<str>> {
    when { return arrived(delta_value(members)) }
}

fn relayed(members: set<str>) -> set<str> {
    when { return delta_value(members) }
}
```

`TimedValue<f64>` has a `value: f64`, and `TimedValue<set<str>>` a
`value: delta<set<str>>`. `delta_value(members)` is the set's delta for this
cycle, given to `arrived` as any value is. `joined` constructs a delta by
its properties: `removed`, left out, is empty. `cleared` is the delta of an
`f64?` that published a null: its `value`, left out, is null. `relayed`
publishes the delta it reads, as that change (TIME-7). Deliberate source
rejections:
`delta<signal>` (`delta.unsupported_shape`); `change.added = []`, a property
of a delta assigned (`binding.immutable`). Conformance is planned.

Accepted declaration fragments, structs that hold themselves (TYPE-6):

```hgl
struct Node {
    value: i64
    next: atomic<Node?> = null
}

struct Tree {
    children: atomic<list<Tree>> = []
}

struct Index {
    entries: atomic<dict<str, Index>> = []
}

struct Link {
    next: Link?
}

struct Chain {
    next: atomic<Chain>?
}
```

A `Node` holds the next one or null, a `Tree` its children as one snapshot,
a leaf having none, an `Index` its entries by name, a `Link` the next one or
null, and a `Chain` a snapshot of the next one or nothing. A `Tree` may also
be written `children: atomic<list<Tree>?> = null`, which tells a leaf with
no list from one with an empty list. Deliberate rejections:
`children: list<Tree>`, a cycle through collection elements outside an
`atomic<...>`; `next: atomic<Node>`, which is not optional;
`pair: atomic<tuple<Tree, Tree>>`, which always holds two;
`children: atomic<list<Tree>> = [Tree()]`, a default that holds a struct of
its cycle; `size: i64 = null`, a null default on a field that is not
optional; and, in a generic `Node<T>`, `next: atomic<Node<list<T>>?> = null`,
which names a new type at every step. Conformance is planned.

Accepted illustration, complete module:

```hgl
module examples.enums

export enum Side { buy, sell }

const fn opposite(side: Side) -> Side {
    if side == .buy { return .sell }
    return .buy
}

test members {
    assert Side.buy != Side.sell
    assert str(Side.sell) == "sell"
    assert opposite(.buy) == Side.sell
    assert Side(1) == Side.sell
    assert Side("buy") == Side.buy
}
```

`Side.buy` and `Side.sell` name the members. A module that imports `Side`
names them the same way. In `opposite`, the parameter, the result, and the
other operand each give `.buy` and `.sell` the type `Side`; in a `switch` on
a `Side`, `case .buy:` takes it from the selector. `Side(1)` constructs the
member numbered 1, `sell`, and `Side("buy")` the member named `buy`.
Deliberate source rejections: `let side = .buy`, where nothing gives a type;
`Side(2)` and `Side("BUY")`, constants that are no member's number or name;
and `Side(1.5)`, which is neither an `i64` nor a `str`. Conformance is
planned.

Accepted value-function fragment:

```hgl
const fn value_or_zero(value: f64?) -> f64 {
    if value == null { return 0.0 }
    return value
}
```

The immutable parameter is nullable despite the function being non-temporal.
The terminating null branch permits the later return as f64. `value_or_zero(null)`
returns 0.0; `value_or_zero(3.0)` returns 3.0. Core owns executable observations;
conformance is planned.

Accepted type fragments in an ordinary value parameter or result position:

| Type | Admitted value shape |
| --- | --- |
| `f64?` | A floating-point value or null |
| `list<f64?>` | A list with nullable elements |
| `list<f64>?` | A list of non-nullable floats, or null for the whole list |

These list forms are distinct: a null element is not a null list. Deliberate
source type rejections: supplying null to an f64 parameter; `set<str?>` and
`dict<str?, f64>`, whose keys take no `?`. The suffix admits only ordinary
value types; applying it to `ref<T>` or `signal` rejects under TYPE-12/14
rather than creating nullable endpoint handles.

## Proposed ordinary storage extensions

Data formation grants no publication admission. Views/borrows do not outlive their supplying hook. Typed
empty-list construction and retained end growth support replay/record storage.
These storage proposals add no `mut` type qualifier. Explicit nullable value
annotations follow TYPE-14/15 independently of these proposals.

## Core boundary

Core owns storage, publication, reference, window, and native-value interfaces.
This chapter defines source types and relationships, not their physical layout.
