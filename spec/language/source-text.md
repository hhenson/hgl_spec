# Source text

Status: agreed lexical baseline carried forward. Literal-domain and contextual
admission rules are completed by the linked chapters. The lexical completion
section retains its proposed status.

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
string_character = ? Unicode scalar other than quote, backslash, line feed, or carriage return ? ;
escape = "\\", ( "\"" | "\\" | "n" | "r" | "t" ) ;
line_break = line_feed | carriage_return, [ line_feed ] ;
line_feed = ? U+000A ? ;
carriage_return = ? U+000D ? ;
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
  can end. Ignore it inside parentheses, brackets, generic argument lists,
  and operator-configuration braces (EXPR-10);
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
- **LEX-15.** A line break is a line feed, a carriage return immediately
  followed by a line feed, or a carriage return alone. A carriage return and
  the line feed directly after it are one line break, not two. The three forms
  are equivalent in every rule: ending a line comment, bounding a string,
  significant newlines, and counting physical lines. A source may mix them;
  none is preferred.

Hard reserved words:

```text
module part use as export abstract impl instantiate operator fn cpp struct enum
requires is let var state cache inject return if else start when stop
for while yield test assert eval true false null bool i64 f64 str bytes any
date time datetime duration civil_datetime zoned_datetime zoned_time timezone
```

- **LEX-9.** `time` is also admitted as a field/member name and named-argument
  label. `const` is a modifier only in positions that admit it; elsewhere it
  is an ordinary identifier, including callable and imported names (FN-5).
- **LEX-10.** Contextual names include `const`, `atomic`, `tuple`, `list`, `set`,
  `dict`, `rolling`, `result`, `error`, `ref`, `signal`, `delta`, `native`, and `type`.
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
- **LEX-14.** `<=>` is one full-comparison operator token, not `<` followed
  by `=>` or `<=` followed by `>`. Its newline behavior follows LEX-5's binary
  operator rule. Operator configuration uses ordinary brace/colon tokens.

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

Accepted illustration, line breaks, with LF and CR naming the two characters:
the source `a` LF `b` CR LF `c` CR `d` has four physical lines and `d` begins
line 4. CR CR LF is two line breaks, as is LF CR. A source rewritten from one
form to another keeps its tokens, their lines and columns, and its meaning.

## Proposed lexical completion

These rules settle token formation where the agreed rules are silent. They
apply only once agreed.

```ebnf
WHITESPACE = whitespace_character, { whitespace_character } ;
whitespace_character = ? U+0020 or U+0009 ? ;
PUNCTUATION = "..." | "<=>"
            | "::" | "->" | "=>" | "==" | "!=" | "<=" | ">=" | "&&" | "||"
            | "//" | "+=" | "-=" | "*=" | "/="
            | "(" | ")" | "[" | "]" | "{" | "}" | "," | ";" | ":" | "." | "@"
            | "?" | "!" | "+" | "-" | "*" | "/" | "%" | "=" | "<" | ">" ;
```

- **LEX-16.** Source that is not well-formed UTF-8 is a source error. One
  U+FEFF at the start of a source is ignored. Outside strings and comments,
  WHITESPACE separates tokens and has no other meaning; any other character
  that begins no token is a source error.
- **LEX-17.** Each PUNCTUATION alternative is one token. At every position
  the longest matching token is taken, whatever the context; LEX-14 is an
  instance. No token joins `>` to a following `>`, which gives LEX-8's separate
  closers. `>` directly followed by `=` is `>=` even after generic arguments:
  write `list<i64> = value`.
- **LEX-18.** A numeric, duration, or temporal literal extends over every
  ASCII letter, digit, and underscore that directly follows it. A run that is
  not one literal is rejected; it is never split into a literal and a name.
  `@` directly followed by a digit or `[` begins a temporal literal and is
  rejected when none matches.
- **LEX-19.** A UTC offset has hours `00` to `23` and minutes `00` to `59`.
- **LEX-20.** A float literal denotes the f64 nearest its exact decimal value,
  ties to even. One that rounds to infinity is rejected under LEX-7. A nonzero
  literal that rounds to zero or to a subnormal is accepted.
- **LEX-21.** A block or documentation comment separates tokens as whitespace
  does. One that contains a line break also counts as one line break at its
  position. A line comment ends before its line break, which remains. `/**/`
  is an empty block comment, not a documentation comment.
- **LEX-22.** `cpp` is an ordinary identifier: no grammar uses it, and foreign
  source embedding belongs to providers ([native boundaries](native-boundaries.md)).
  Until this is agreed the reserved list above stands.

Proposed illustrations, lexical fragments:

| Source | Observation |
| --- | --- |
| `list<list<i64>>` | Two `>` tokens close the two argument lists |
| `1e-400` | Float zero |
| `/**/` | Empty block comment |

Deliberate rejections, proposed: `1_000`, `0x10`, `1e`, and `1h_` (LEX-18);
`@2026-09-03T09:30+24:00` (LEX-19); `1e400` (LEX-20); `xs: list<i64>= value`
(LEX-17); a form feed between tokens (LEX-16). Conformance is planned.

## Core boundary

Tokenization requires no running graph. Calendar domains and provider-backed
construction use the [value contracts](types-and-values.md) and core services.
