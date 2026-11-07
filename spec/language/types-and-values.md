# Types and values

Status: agreed type foundation. Ordinary delta storage and mutable aggregate
access remain proposed extensions.

## Grammar

```ebnf
type = type_base, [ "?" ] ;
type_base = scalar_type | named_type | tuple_type | list_type | set_type
     | dict_type | rolling_type | ref_type | "signal"
     | "atomic", "<", value_type, ">" | result_type | "error" | delta_type ;
value_type = value_type_base, [ "?" ] ;
value_type_base = scalar_type | named_type | value_tuple_type | value_list_type
           | set_type | value_dict_type | result_type | "error" | delta_type ;
scalar_type = "bool" | "i64" | "f64" | "str" | "bytes" | "any"
            | "date" | "time" | "datetime" | "duration" | "civil_datetime"
            | "timezone" | "zoned_datetime" | "zoned_time" ;
named_type = ( IDENTIFIER | qualified_name ), [ generic_arguments ] ;
tuple_type = "tuple", "<", type, { ",", type }, ">" ;
value_tuple_type = "tuple", "<", value_type, { ",", value_type }, ">" ;
list_type = "list", "<", type, [ ",", size_expression ], ">" ;
value_list_type = "list", "<", value_type, [ ",", size_expression ], ">" ;
size_expression = additive_expr | "unbounded" ;
set_type = "set", "<", value_type, ">" ;
dict_type = "dict", "<", value_type, ",", type, ">" ;
value_dict_type = "dict", "<", value_type, ",", value_type, ">" ;
rolling_type = "rolling", "<", value_type, ",", const_expression,
               [ ",", const_expression ], ">" ;
ref_type = "ref", "<", type, ">" ;
result_type = "result", "<", value_type, ",", value_type, ">" ;
delta_type = "delta", "<", type, ">" ;
struct_decl = [ "export" ], [ "abstract" ], "struct", IDENTIFIER,
              [ generic_parameters ], [ ":", named_type, { ",", named_type } ],
              [ requires_clause ], "{", [ NL ],
              [ struct_member, { NL, struct_member }, [ NL ] ], "}" ;
struct_member = member_name, ":", type, [ "=", const_expression ]
              | member_name, "=", const_expression ;
enum_decl = "enum", IDENTIFIER, "{", [ NL ],
            [ enum_member, { ",", [ NL ], enum_member }, [ "," ] ], [ NL ], "}" ;
enum_member = IDENTIFIER, [ "=", const_expression ] ;
```

Semantic field rules distinguish a typed introduction from an inherited-default
override. `delta_type` syntax has the proposed formation domain below.

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
- **TYPE-3.** Structs are nominal, with public ordered fields and no methods.
  Only abstract structs may be parents; abstract values are not constructible.
  Concrete structs are final. Generic applications are invariant by origin
  and complete canonical arguments.
- **TYPE-4.** A typed field introduction fixes its type and optionality. A
  null default marks it optional. An untyped inherited override changes only
  its construction default. Required fields may have defaults; optional fields
  may have non-null defaults. Parent conflicts in type/optionality are errors.
- **TYPE-5.** An ordinary constructor supplies complete required data. Written
  arguments evaluate once in source order; failure prevents later arguments.
  Defaults are evaluated only when omitted. Retained results own their data;
  later mutation of a source cannot change an earlier retained value.
- **TYPE-6.** Recursive struct edges stay within their defining module, use
  an optional `atomic<...>` field defaulted to null, and preserve finite generic
  specialization. Descendants cannot remove that null default. Cycles through
  inheritance or collection elements are rejected. Cross-module references to
  non-recursive public structs remain admitted.
- **TYPE-7.** Enum members are distinct typed constants. Unnumbered numbering
  starts at zero and increments the previous member; explicit numbers reset
  the next value. Values fit signed `i64`; duplicates and overflow reject.
  Construction from an exact member name or assigned integer is checked.
  String conversion returns the member name. Numeric conversion is explicit.
- **TYPE-8.** `any` owns a boxed value with its exact contained type; boxing
  adds no numeric conversion. Equality across contained types is false.
  Operations needing an absent capability fail rather than inventing an
  order/hash. Empty and populated boxes remain distinct. Container and retained
  copies preserve independent owned children.
- **TYPE-14.** `T?` denotes an ordinary value that is either a T or null, where
  T is an admitted ordinary value type. It may annotate parameters, results,
  bindings, and admitted container element/value positions. Nullability is
  independent of a callable's temporal/value role and of fixed configuration.
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
  constants. `unbounded` is the size sentinel. Size participates in identity;
  an unsized parameter pattern can accept a fixed size. A tuple retains
  positional identity and is not a fixed list alias.
- **TYPE-11.** A rolling maximum/minimum has one constant kind: tick count or
  duration. Omitted minimum equals maximum. Tick sizes are positive; duration
  maximum is positive and minimum may be zero. Minimum cannot exceed maximum.
  Size/kind are part of identity; window publications obey core's contract.
- **TYPE-12.** `ref<T>` designates an endpoint, not an ordinary payload type.
  It is excluded from const values, atomic payloads, keys, and window values.
  `signal` is a complete temporal input constraint with no readable payload;
  it is not a result, nested shape, field, or const value.
- **TYPE-13.** Result and error envelopes are ordinary atomic values; their
  [tag/refinement rules](errors-and-diagnostics.md) govern payload access.
  Silence, invalidity, null absence, and an error outcome are distinct.

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
source type rejection: supplying null to an f64 parameter. The suffix admits
only ordinary value types; applying it to `ref<T>` or `signal` rejects under
TYPE-12/14 rather than creating nullable endpoint handles.

## Proposed ordinary storage extensions

`delta<T>` is an owned ordinary publication value for an admitted originating
shape. Scalars reduce to themselves; composite atomic deltas are complete V;
structural deltas retain exact nominal arguments and sizes. A held complete
value is not a sparse delta. Data formation grants no publication admission.

Ordinary mutable aggregates use writable `var` access and read-only `let`
access recursively. Views/borrows do not outlive their supplying hook. Typed
empty-list construction and retained end growth support replay/record storage.
These storage proposals add no `mut` type qualifier. Explicit nullable value
annotations follow TYPE-14/15 independently of these proposals.

## Core boundary

Core owns storage, publication, reference, window, and native-value interfaces.
This chapter defines source types and relationships, not their physical layout.
