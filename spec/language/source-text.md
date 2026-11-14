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
digits = digit, { [ "_" ], digit } ;
hex_digit = digit | ? ASCII A through F or a through f ? ;
hex_digits = hex_digit, { [ "_" ], hex_digit } ;
binary_digit = "0" | "1" ;
binary_digits = binary_digit, { [ "_" ], binary_digit } ;
INTEGER_LITERAL = digits | "0x", hex_digits | "0b", binary_digits ;
FLOAT_LITERAL = digits, ".", digits, [ exponent ] | digits, exponent ;
exponent = ( "e" | "E" ), [ "+" | "-" ], digits ;
STRING_LITERAL = "\"", { string_character | escape }, "\"" | text_block ;
string_character = ? Unicode scalar other than quote, backslash, line feed, or carriage return ? ;
escape = "\\", ( "\"" | "\\" | "n" | "r" | "t"
               | "u", "{", hex_digit, { hex_digit }, "}" ) ;
text_block = "\"\"\"", line_break, { text_line, line_break },
             { whitespace_character }, "\"\"\"" ;
text_line = { text_character | escape } ;
text_character = ? Unicode scalar other than backslash, line feed, carriage return,
                   or the first of three quotes in a row ? ;
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
  can end. Ignore it inside parentheses, brackets, and angle-bracketed lists,
  and inside braces that hold a list rather than statements or members:
  import sets, constraint sets and `each` constraints, operator property
  lists, pack types, and operator configuration (EXPR-10). Elsewhere, ignore
  the one line break between two tokens where the declaration or statement
  before it is incomplete, or where the token after it cannot begin a
  declaration or statement and continues the one before. A `properties`
  clause continues its operator declaration. After a declaration or
  statement that can end, a token that can begin one does: `!`, `(`, and
  `[` start a new statement there, so a prefix `!`, a call, and an index do
  not continue a line. `-` there is the binary operator and continues the
  line: a statement that began with it would have no effect. Two or more
  line breaks between two tokens, as around a blank line, always separate.
  A body's opening brace stays with its header.
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
module part use as export abstract impl instantiate operator fn struct enum
requires is let var state cache inject return if else start when stop
for while yield test assert eval true false null bool i64 f64 str bytes any
date time datetime duration civil_datetime zoned_datetime zoned_time timezone
```

- **LEX-9.** `time` is also admitted as a field/member name and named-argument
  label. `const` is a modifier only in positions that admit it; elsewhere it
  is an ordinary identifier, including callable and imported names (FN-5).
- **LEX-10.** The contextual names are `const`, `atomic`, `tuple`, `list`, `set`,
  `dict`, `rolling`, `result`, `error`, `ref`, `signal`, `delta`, `native`, and
  `type`; `switch`, `case`, `default`, `in`, `each`, `unbounded`, `properties`,
  `associative`, `commutative`, `identity`, `raises`, and `suppress_warning`;
  and the capability names. Each acquires its special meaning only in its
  defined positions. A contextual spelling is not globally reserved.
  `str(value)` is a conversion expression despite `str` being reserved in
  ordinary name positions.

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
  by `=>` or `<=` followed by `>`. It continues a line as the other binary
  operators do (LEX-5). Operator configuration uses ordinary brace/colon
  tokens.

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

## Tokens and literals

```ebnf
WHITESPACE = whitespace_character, { whitespace_character } ;
whitespace_character = ? U+0020 or U+0009 ? ;
LINE_CONTINUATION = "\\", { whitespace_character }, line_break ;
PUNCTUATION = "..." | "<=>" | "//="
            | "::" | "->" | "=>" | "==" | "!=" | "<=" | ">=" | "&&" | "||"
            | "//" | "+=" | "-=" | "*=" | "/=" | "%="
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
- **LEX-18.** A numeric, duration, or temporal literal is never split into a
  literal and a name. It can end only where the next character is not an
  ASCII letter, digit, or underscore; when the text up to there is not one
  valid literal, all of it is rejected. `12abc` is one malformed literal, not
  `12` followed by `abc`; so are `1.5x`, `1e`, `1_`, and `1h_`. `@` directly
  followed by a digit or `[` begins a temporal literal and is rejected when
  none matches.
- **LEX-19.** A UTC offset has hours `00` to `23` and minutes `00` to `59`.
- **LEX-20.** A float literal denotes the f64 nearest its exact decimal value,
  ties to even. One that rounds to infinity is rejected under LEX-7. A nonzero
  literal that rounds to zero or to a subnormal is accepted.
- **LEX-21.** A block or documentation comment separates tokens as whitespace
  does. One that contains a line break also counts as one line break at its
  position. A line comment ends before its line break, which remains. `/**/`
  is an empty block comment, not a documentation comment.
- **LEX-22.** One `_` may stand between two digits of any digit run in an
  integer, float, or duration literal: a whole part, a fraction, an exponent,
  and hexadecimal or binary digits. It does not change the value.
- **LEX-23.** `0x` directly followed by a hexadecimal digit begins a
  hexadecimal integer literal; `0b` directly followed by `0` or `1` begins a
  binary one. Neither is a duration candidate (LEX-8). The prefix letter is
  lowercase; hexadecimal digits may be of either case. The value is a
  non-negative integer under the range rule for decimal literals (LEX-7).
- **LEX-24.** `\u{...}` in a string denotes the Unicode scalar value numbered
  by its one to six hexadecimal digits. A surrogate or a number above 10FFFF
  is rejected.
- **LEX-25.** A text block is a string literal over several lines. `"""`
  opens one and is directly followed by a line break. The block closes at
  the next `"""`, which follows only spaces and tabs on its line. Its value
  is the lines between, joined by line feeds whatever line-break form the
  source uses (LEX-15); no line feed follows the last line. The spaces and
  tabs before the closing `"""` are removed from the start of every line. A
  line of only spaces and tabs contributes an empty line; any other line
  that does not start with them is rejected. Escapes apply as in other
  strings; a `"` needs one only where it would begin `"""`.
- **LEX-26.** `\` followed by a line break, with only spaces or tabs between
  them, is a line continuation. It separates tokens as whitespace does, and
  its line break is not a line break: the next line continues the current
  one whatever it begins with (LEX-5). No comment follows the `\` on its
  line. Outside a string literal or comment, any other `\` begins no token
  (LEX-16).
- **LEX-27.** An opening bracket directly follows what it applies to: no
  whitespace, comment, line break, or line continuation stands between
  them. This holds for the `(` of a call or a parameter list, the `[` of an
  index, the `<` of generic parameters or arguments, and the `{` of a
  cardinality: `f<T>(x)`, `rows[i]`, and `...T{2}`, never `f <T>(x)`,
  `f<T> (x)`, `rows [i]`, or `...T {2}`. On the next line, `(x)` and `[i]`
  begin a statement of their own (LEX-5). A call here is a call suffix, the
  arguments of a typed or delta constructor, a conversion such as
  `str(value)`, `eval(...)`, `raises(...)`, a constraint call, and a warning
  annotation. In an expression, a `<` that stands apart from the name
  before it is a comparison.

Accepted illustrations, lexical fragments:

| Source | Observation |
| --- | --- |
| `list<list<i64>>` | Two `>` tokens close the two argument lists |
| `1e-400` | Float zero |
| `/**/` | Empty block comment |
| `1_000_000` | Integer one million |
| `0xFF`, `0b1111_1111` | Integer 255 |
| `1_500ms` | Duration of one and a half seconds |
| `"\u{20AC}"` | String of one character, U+20AC |
| `rows[0](x)` | An index, then a call of what it selects |

Accepted illustration, text block:

```hgl
let usage = """
    Usage: run [options]
      -v  verbose
    """
```

`usage` is `Usage: run [options]`, a line feed, then `  -v  verbose`. Four
columns of indentation are removed and no line feed ends the value.

Accepted illustration, lines that continue:

```hgl
struct Quote {
    bid: f64
    ask: f64
}

fn spread(quote: Quote) -> f64
    => quote.ask - quote.bid

fn low(quote: Quote, margin: f64) -> f64 {
    let sum = quote
        .bid
        + quote.ask
    let half = sum /
        2.0
    let reduced = half
        - margin
    let result =
        (reduced)
    return \
        result
}
```

`spread` is one declaration: `=>` cannot begin one. In `low`, `.bid` and
`+ quote.ask` continue `let sum`, and `- margin` continues `let reduced`.
`2.0` completes `let half` and `(reduced)` completes `let result`: neither
first line can end. After a line that can end, `(reduced)` would begin a
statement of its own. Had the line after `let half = sum /` been `-2.0`, its
`-` would be the prefix operator: the line before has no operand to its
right. The `\` joins `result` to `return`: without it `return` would end
there, and `result` would begin a statement of its own.

Deliberate rejections: `12abc`, `1.5x`, `1e`, `1_`, `1h_`, `0x`, and
`0b102` (LEX-18); `1__000` (LEX-22); `0xFFFF_FFFF_FFFF_FFFF` (LEX-23);
`"\u{D800}"` (LEX-24); `"""text"""` on one line, a text-block line holding
`"""` after other text, and a text-block line indented less than its closing
delimiter (LEX-25);
`@2026-09-03T09:30+24:00` (LEX-19); `1e400` (LEX-20);
`xs: list<i64>= value` (LEX-17); a form feed between tokens (LEX-16);
a blank line between `let half = sum /` and `2.0` (LEX-5); `\` followed on
its line by a comment or by any token (LEX-26); `f (x)`, `x [i]`,
`fn f <T>(x: T)`, `list <i64>`, and `...T {2}` (LEX-27).
Conformance is planned.

## Core boundary

Tokenization requires no running graph. Calendar domains and provider-backed
construction use the [value contracts](types-and-values.md) and core services.
