# Appendix A: FEEL Quick Reference Card

---

## Data Types

| Type | Literal Examples | Properties |
|------|-----------------|------------|
| `number` | `42`, `3.14`, `.5`, `1.2e3` | 34-digit Decimal128 precision |
| `string` | `"hello"`, `"a\nb"` | Unicode, `+` concatenation |
| `boolean` | `true`, `false` | Three-valued logic with `null` |
| `date` | `@"2024-03-15"` | `.year`, `.month`, `.day`, `.weekday` |
| `time` | `@"14:30:00Z"` | `.hour`, `.minute`, `.second`, `.time offset`, `.timezone` |
| `date and time` | `@"2024-03-15T14:30:00Z"` | All date + time properties |
| `days and time duration` | `@"P2DT3H"` | `.days`, `.hours`, `.minutes`, `.seconds` |
| `years and months duration` | `@"P1Y6M"` | `.years`, `.months` |
| `list` | `[1, 2, 3]` | 1-based indexing, `count()` for length |
| `context` | `{name: "Alice", age: 30}` | Sequential evaluation, `.key` access |
| `range` | `[1..10]`, `(0..100)` | `.start`, `.end`, `.start included`, `.end included` |
| `function` | `function(x) x + 1` | Lexical closure |
| `null` | `null` | Propagates through operations |

---

## Operators (by Precedence, Low to High)

| Precedence | Operators | Associativity |
|-----------|-----------|---------------|
| 1 (lowest) | `or` | left |
| 2 | `and` | left |
| 3 | `=`, `!=`, `<`, `<=`, `>`, `>=`, `between`, `in`, `instance of` | left |
| 4 | `+`, `-` | left |
| 5 | `*`, `/` | left |
| 6 | `**` | left |
| 7 | `-` (unary negation) | right |
| 8 (highest) | `.` (path), `[]` (filter), `()` (invocation) | left |

---

## Truth Tables (Ternary Logic)

### `and`

| | true | false | null |
|---|------|-------|------|
| **true** | true | false | null |
| **false** | false | false | false |
| **null** | null | false | null |

### `or`

| | true | false | null |
|---|------|-------|------|
| **true** | true | true | true |
| **false** | true | false | null |
| **null** | true | null | null |

### `not()`

| a | not(a) |
|---|--------|
| true | false |
| false | true |
| null | null |

---

## Built-in Functions

### Conversion
`date()`, `time()`, `date and time()`, `duration()`, `years and months duration()`, `number()`, `string()`, `range()`

### Boolean
`not()`, `is defined()`, `get or else()`, `assert()`

### String
`substring()`, `string length()`, `upper case()`, `lower case()`, `contains()`, `starts with()`, `ends with()`, `substring before()`, `substring after()`, `replace()`, `matches()`, `split()`, `string join()`

### Numeric
`decimal()`, `round up()`, `round down()`, `round half up()`, `round half down()`, `floor()`, `ceiling()`, `abs()`, `modulo()`, `sqrt()`, `log()`, `exp()`, `odd()`, `even()`

### List
`count()`, `min()`, `max()`, `sum()`, `mean()`, `median()`, `stddev()`, `mode()`, `product()`, `all()`, `any()`, `append()`, `concatenate()`, `insert before()`, `remove()`, `reverse()`, `index of()`, `union()`, `distinct values()`, `flatten()`, `sort()`, `list contains()`, `list replace()`, `sublist()`

### Context
`get value()`, `get entries()`, `context()`, `context put()`, `context merge()`

### Range / Temporal
`before()`, `after()`, `meets()`, `met by()`, `overlaps()`, `includes()`, `during()`, `starts()`, `finishes()`, `coincides()`, `now()`, `today()`, `day of week()`, `day of year()`, `week of year()`, `month of year()`

---

## Hit Policies

| Letter | Name | Returns | Behaviour |
|--------|------|---------|-----------|
| U | Unique | Single | Exactly one rule matches |
| A | Any | Single | Multiple matches, same output |
| F | First | Single | First match wins |
| P | Priority | Single | Highest priority output |
| C | Collect | List | All matching outputs |
| C+ | Collect Sum | Number | Sum of outputs |
| C# | Collect Count | Number | Count of matches |
| C< | Collect Min | Value | Minimum output |
| C> | Collect Max | Value | Maximum output |
| R | Rule Order | List | Matches in rule order |
| O | Output Order | List | Matches in output value order |

---

## Common Unary Tests

| Syntax | Meaning |
|--------|---------|
| `42` | Equals 42 |
| `"Gold"` | Equals "Gold" |
| `< 100` | Less than 100 |
| `>= 18` | At least 18 |
| `[18..65]` | Between 18 and 65 (inclusive) |
| `(0..1)` | Between 0 and 1 (exclusive) |
| `"A", "B", "C"` | One of these values |
| `not("X")` | Not X |
| `not(< 0)` | Not negative |
| `-` | Any value (wildcard) |
