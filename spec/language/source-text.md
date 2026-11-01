# Source text

Status: agreed lexical baseline carried forward. Literal-domain and contextual
admission rules are completed by the linked chapters.

## Grammar

```ebnf
IDENTIFIER = ( ascii_letter | "_" ), identifier_tail ;
identifier_tail = { ascii_letter | digit | "_" } ;
ascii_letter = ? ASCII A through Z or a through z ? ;
digit = "0" | "1" | "2" | "3" | "4"
      | "5" | "6" | "7" | "8" | "9" ;
digits = digit, { digit } ;
INTEGER_LITERAL = digits ;
FLOAT_LITERAL = digits, ".", digits, [ exponent ] | digits, exponent ;
exponent = ( "e" | "E" ), [ "+" | "-" ], digits ;
STRING_LITERAL = "\"", { string_character | escape }, "\"" ;
string_character = ? Unicode scalar other than quote, backslash, or line break ? ;
escape = "\\", ( "\"" | "\\" | "n" | "r" | "t" ) ;
NL = ? one or more significant line breaks after continuation handling ? ;
EOF = ? end of the source file ? ;
literal = INTEGER_LITERAL | FLOAT_LITERAL | STRING_LITERAL
        | TEMPORAL_LITERAL | "true" | "false" | "null" ;
```

A lone `_` is classified as a placeholder token, not an identifier.

## Rules

- **LEX-1.** Source is UTF-8. Identifiers match an ASCII letter or underscore
  followed by zero or more ASCII letters, digits, or underscores. Reserved
  words and the lone placeholder are classified separately.
- **LEX-2.** Strings use double quotes. Their contents may be Unicode. Unknown
  escapes, unclosed strings, and unclosed comments are source errors.
- **LEX-3.** `#` starts a line comment; `/* ... */` is a non-nesting block
  comment. `//` is floor division. Comment contents do not become tokens;
  source locations and significant line breaks remain meaningful.
- **LEX-4.** Braces delimit blocks. Indentation is non-semantic. A semicolon is
  not an ordinary statement terminator. Native hook markers are a separate
  [declaration form](native-boundaries.md).
- **LEX-5.** A newline separates declarations or statements only where one
  can end. Ignore it inside parentheses, brackets, and generic argument lists;
  after a comma, binary/assignment operator, `=`, `=>`, `->`, or a type/named-
  argument colon; and before a binary operator, result arrow, or matching
  `else`. `!` does not continue a line. Postfix calls/indexing do not cross a
  significant newline. A body's opening brace stays with its header.
- **LEX-6.** Inside blocks, indentation alone cannot join statements. Two
  ordinary statements on one physical line are rejected. An inject list may
  continue after a comma when the next line begins with a name.
- **LEX-7.** Minus is an operator, not part of a numeric literal. Required
  literal range checks use the i64/f64 domains. Enum numbering admits the full
  signed minimum. Runtime overflow policy is not supplied by lexing.
- **LEX-8.** Nested generic closers use separate `>` tokens. A digit run
  immediately followed by letters is a duration candidate; an unknown unit
  is rejected rather than split into a number and identifier. A valid exponent
  without a following unit remains a float: `1e5` is valid, `1e5m` is not.

Hard reserved words:

```text
module part use as export abstract impl instantiate operator fn cpp struct enum
const requires is let var state cache inject return if else start when stop
for while yield test assert eval true false null bool i64 f64 str bytes any
date time datetime duration civil_datetime zoned_datetime zoned_time timezone
```

- **LEX-9.** `time` is also admitted as a field/member name and named-argument
  label. `const` may name a function, operator, instantiation, or import; it
  remains a modifier or value-role selector in the corresponding context.
- **LEX-10.** Contextual names include `atomic`, `tuple`, `list`, `set`, `dict`,
  `rolling`, `result`, `error`, `ref`, `signal`, `delta`, `native`, and `type`.
  `switch`, `case`, `default`, `in`, `each`, `unbounded`, and capability names
  acquire their special meaning only in their defined positions. A contextual
  spelling is not globally reserved. `str(value)` is a conversion expression
  despite `str` being reserved in ordinary name positions.

## Temporal literals

```ebnf
TEMPORAL_LITERAL = "@", temporal_body | duration_literal ;
temporal_body = calendar_date, [ "T", clock_time, [ utc_offset, [ zone_annotation ] ] ]
              | clock_time, [ zone_annotation ] | zone_annotation ;
calendar_date = digit4, "-", digit2, "-", digit2 ;
clock_time = digit2, ":", digit2, [ ":", digit2, [ ".", fraction ] ] ;
utc_offset = "Z" | ( "+" | "-" ), digit2, [ ":", digit2 ] ;
zone_annotation = "[", zone_component, { "/", zone_component }, "]" ;
zone_component = zone_character, { zone_character } ;
zone_character = ascii_letter | digit | "_" | "-" | "+" | "." ;
digit2 = digit, digit ;
digit4 = digit2, digit2 ;
fraction = digit, [ digit, [ digit, [ digit, [ digit, [ digit ] ] ] ] ] ;
duration_literal = duration_part, { duration_part } ;
duration_part = digits, [ ".", digits ], duration_unit ;
duration_unit = "d" | "h" | "m" | "s" | "ms" | "us" ;
```

- **LEX-11.** `T`/`Z` are uppercase. Seconds and offset minutes default to
  zero. Fractional seconds have one to six digits. Calendar fields must exist;
  time excludes `24:00:00` and leap seconds. Offset-free date/time is civil,
  not an implicit UTC instant. A zoned datetime requires an explicit offset.
- **LEX-12.** Zone components are nonempty, neither `.` nor `..`, with no
  leading/trailing/repeated slash; the name is at most 255 bytes. The selected
  run provider checks exact catalog membership and offset consistency. Zone
  aliases retain their names. Provider checks do not depend on the compiler's
  local time-zone database.
- **LEX-13.** Duration units occur once each, in descending order. Only the
  final part may have a fraction. Exact decimal scaling must yield whole
  microseconds in the duration domain. No exponent or internal whitespace is
  admitted. One day is exactly 86,400 seconds.

Accepted illustrations, lexical fragments:

| Source | Observation |
| --- | --- |
| `1h30m` | One duration token, ninety minutes |
| `-250ms` | Minus followed by a duration token |
| `@09:30` | Time, seconds zero |
| `@2026-09-03T09:30Z` | UTC instant |
| `@[Europe/London]` | Named zone, subject to provider validation |

Deliberate rejections: `30m1h`, `0.5us`, `@2026-02-29`, and an unclosed string
violate the corresponding lexical/literal rules. Conformance is planned.

## Core boundary

Tokenization requires no running graph. Calendar domains and provider-backed
construction use the [value contracts](types-and-values.md) and core services.
