# Chapter 5: Values and Types — The Atoms of FEEL

> *"Every expression evaluates to a value. Every value has a type. Understand the types and you understand the language."*

---

FEEL has a small, carefully chosen set of data types. Unlike general-purpose languages that distinguish between integers, floats, longs, and doubles, FEEL keeps things simple. One number type. One string type. Dates and durations are first-class citizens, not afterthoughts bolted on through libraries.

Master these types and you will know what every FEEL expression can produce.

---

## 5.1 Numbers

### Literal Syntax

```
42          // integer
3.14        // decimal
-7          // negative
.5          // leading dot (= 0.5)
1.2e3       // scientific notation (= 1200)
1.2E-2      // scientific notation (= 0.012)
```

### Precision

FEEL numbers use **IEEE 754-2008 Decimal128** -- 34 decimal digits of precision. If you have ever been bitten by JavaScript's floating-point quirks, you will appreciate this. There are no binary rounding surprises:

```
0.1 + 0.2 = 0.3    // true in FEEL (not 0.30000000000000004)
```

There is no separate integer type -- `42` and `42.0` are the same value, and `42 = 42.0` evaluates to `true`.

There is no `NaN`, no `Infinity`, no `-0`. If a numeric operation cannot produce a valid result, you get `null` instead of a special sentinel that silently poisons downstream math.

### Rosetta Stone

| Operation | FEEL | JavaScript | Python | SQL |
|-----------|------|-----------|--------|-----|
| Literal | `42` | `42` | `42` | `42` |
| Division by zero | `1/0` → `null` | `1/0` → `Infinity` | `1/0` → `ZeroDivisionError` | `1/0` → error |
| Precision | 34 decimal digits | ~15 binary digits | Arbitrary (int) / ~15 (float) | Varies |
| `0.1 + 0.2` | `0.3` | `0.30000000000000004` | `0.30000000000000004` | `0.3` (if DECIMAL) |

### Rounding Functions

Money demands precise rounding, and FEEL delivers:

```
decimal(1/3, 2)              // 0.33 — round to 2 decimal places (half-even)
round up(5.5, 0)             // 6
round down(5.5, 0)           // 5
round half up(2.5, 0)        // 3
round half down(2.5, 0)      // 2
```

> **GOTCHA:** The `decimal()` function uses **half-even** rounding (banker's rounding), not half-up. `decimal(2.5, 0)` is `2`, not `3`. For financial applications that require half-up, use `round half up()` explicitly.

### Worked Example 5.1 — Financial Rounding

A bank calculates interest daily and rounds to cents:

```
{
  principal: 10000,
  annual rate: 0.045,
  daily rate: annual rate / 365,
  daily interest: principal * daily rate,
  rounded: round half up(daily interest, 2)
}
```

Result: `daily interest` is `1.2328767123287671...`, `rounded` is `1.23`.

---

## 5.2 Strings

### Literal Syntax

```
"hello world"
"She said \"hello\""
"line 1\nline 2"
"tab\there"
"unicode: \u00E9"       // é
```

Strings are always enclosed in double quotes. Single quotes are **not** string delimiters -- forget everything PHP taught you.

### Escape Sequences

| Escape | Meaning |
|--------|---------|
| `\"` | Double quote |
| `\\` | Backslash |
| `\n` | Newline |
| `\r` | Carriage return |
| `\t` | Tab |
| `\'` | Single quote |
| `\uXXXX` | Unicode code point (4 hex digits) |
| `\UXXXXXX` | Unicode code point (6 hex digits) |

### Operations

**Concatenation** uses `+`:
```
"hello" + " " + "world"    // "hello world"
```

**No implicit conversion.** `"value: " + 42` is `null`, not `"value: 42"`. You must convert explicitly:
```
"value: " + string(42)     // "value: 42"
```

> **GOTCHA:** Unlike JavaScript (`"x" + 1` → `"x1"`) and Python (`"x" + str(1)`), FEEL's `+` between a string and a non-string produces `null`. Always use `string()` to convert.

### Rosetta Stone

| Operation | FEEL | JavaScript | Python |
|-----------|------|-----------|--------|
| Length | `string length("abc")` | `"abc".length` | `len("abc")` |
| Uppercase | `upper case("abc")` | `"abc".toUpperCase()` | `"abc".upper()` |
| Contains | `contains("foobar", "ob")` | `"foobar".includes("ob")` | `"ob" in "foobar"` |
| Substring | `substring("foobar", 3)` | `"foobar".slice(2)` | `"foobar"[2:]` |
| Split | `split("a,b,c", ",")` | `"a,b,c".split(",")` | `"a,b,c".split(",")` |
| Join | `string join(["a","b"], ",")` | `["a","b"].join(",")` | `",".join(["a","b"])` |

Note: `substring()` in FEEL uses **1-based indexing**. `substring("foobar", 3)` returns `"obar"` (starting at the 3rd character), which is equivalent to JavaScript's `"foobar".slice(2)` (0-based).

### Worked Example 5.2 — Building a Notification Message

```
{
  customer: "Alice Chen",
  order id: 4821,
  status: "shipped",
  message: "Dear " + customer + ", your order #" + string(order id) + " has been " + status + "."
}
```

Result: `"Dear Alice Chen, your order #4821 has been shipped."`

---

## 5.3 Booleans and Ternary Logic

### Literal Syntax

```
true
false
```

There is no literal for the third truth value — it is `null`, the absence of a value.

### The Three-Valued Truth Tables

FEEL uses **ternary logic**, like SQL. If you take only one thing from this chapter, make it this: `true`, `false`, and `null` are all possible outcomes of a boolean operation.

**Conjunction (`and`):**

| a | b | `a and b` |
|---|---|-----------|
| true | true | **true** |
| true | false | **false** |
| true | null | **null** |
| false | true | **false** |
| false | false | **false** |
| false | null | **false** |
| null | true | **null** |
| null | false | **false** |
| null | null | **null** |

**Disjunction (`or`):**

| a | b | `a or b` |
|---|---|----------|
| true | true | **true** |
| true | false | **true** |
| true | null | **true** |
| false | true | **true** |
| false | false | **false** |
| false | null | **null** |
| null | true | **true** |
| null | false | **null** |
| null | null | **null** |

**Negation (`not()`):**

| a | `not(a)` |
|---|----------|
| true | false |
| false | true |
| null | **null** |

### The Critical Rule

In an `if condition then A else B` expression:
- If `condition` is `true` → the result is `A`.
- If `condition` is `false` **or** `null` → the result is `B`.

So null conditions always take the `else` branch. Think of it as FEEL saying, "I don't know if this is true, so I'm not going to act on it." FEEL never acts on uncertain data.

### Rosetta Stone

| FEEL | JavaScript | Python | SQL |
|------|-----------|--------|-----|
| `true and null` → `null` | `true && null` → `null` (falsy) | `True and None` → `None` | `TRUE AND NULL` → `NULL` |
| `false or null` → `null` | `false \|\| null` → `null` | `False or None` → `None` | `FALSE OR NULL` → `NULL` |
| `false and null` → `false` | `false && null` → `false` | `False and None` → `False` | `FALSE AND NULL` → `FALSE` |
| `not(null)` → `null` | `!null` → `true` (!) | `not None` → `True` (!) | `NOT NULL` → `NULL` |

> **GOTCHA:** JavaScript's `!null` is `true` — JavaScript treats `null` as falsy. FEEL's `not(null)` is `null`. FEEL never guesses.

### Worked Example 5.3 — Null-Safe Eligibility Check

An eligibility rule that must handle missing data:

```
{
  age check:     if Applicant.Age != null
                 then Applicant.Age >= 18 and Applicant.Age <= 70
                 else false,

  income check:  if Applicant.Income != null
                 then Applicant.Income >= 20000
                 else false,

  eligible:      age check and income check
}
```

Without the null guards, `null >= 18` would produce `null`, which would propagate through `and`, and the entire `eligible` result would be `null` — a silent failure. The explicit null checks ensure a definite `true` or `false`.

---

## 5.4 Dates, Times, and Durations

If you have ever wrestled with `java.util.Date`, JavaScript's `Date`, or Python's `datetime` module, you know how painful time handling can be. FEEL gives you five clean temporal types with built-in arithmetic -- no libraries required.

### 5.4.1 Date

A pure calendar date -- year, month, day -- with no time or timezone attached.

```
@"2024-03-15"                    // at-literal (preferred, DMN 1.3+)
date("2024-03-15")               // function form
date(2024, 3, 15)                // component form
```

Properties:
```
@"2024-03-15".year       // 2024
@"2024-03-15".month      // 3
@"2024-03-15".day        // 15
@"2024-03-15".weekday    // 5 (Friday; 1=Monday, 7=Sunday per ISO 8601)
```

### 5.4.2 Time

A time of day, optionally pinned to a timezone.

```
@"14:30:00"                          // local time
@"14:30:00+02:00"                    // with offset
@"14:30:00@Europe/Paris"             // with IANA timezone
time(14, 30, 0)                      // component form
time(14, 30, 0, @"PT2H")            // with offset as duration
```

Properties: `hour`, `minute`, `second`, `time offset`, `timezone`.

### 5.4.3 Date and Time

The full package -- a date and a time in a single value.

```
@"2024-03-15T14:30:00"               // local
@"2024-03-15T14:30:00Z"              // UTC
@"2024-03-15T14:30:00+02:00"         // with offset
@"2024-03-15T14:30:00@Europe/Paris"  // with IANA timezone
date and time(date("2024-03-15"), time("14:30:00"))   // from components
```

Properties: all date properties + all time properties.

### 5.4.4 Days and Time Duration

A span of time measured in days, hours, minutes, and seconds -- the kind of duration where every unit has a fixed, unambiguous length.

```
@"P2D"                    // 2 days
@"PT3H"                   // 3 hours
@"P1DT12H30M"             // 1 day, 12 hours, 30 minutes
@"PT0.5S"                 // half a second
duration("P2DT3H")        // function form
```

Properties: `days`, `hours`, `minutes`, `seconds`.

### 5.4.5 Years and Months Duration

A span of time measured in years and months -- calendar-relative units whose actual length varies.

```
@"P1Y"                    // 1 year
@"P6M"                    // 6 months
@"P1Y6M"                  // 1 year, 6 months
@"P18M"                   // normalised to P1Y6M
duration("P1Y6M")         // function form
```

Properties: `years`, `months`.

### Why Two Duration Types?

You might wonder why FEEL doesn't just have one duration type. The answer: "one month" is not a fixed number of days. January has 31, February has 28 (or 29), and so on. Days and hours, on the other hand, always have exact, fixed lengths. Mixing these two kinds of duration would introduce the same calendar ambiguity bugs that plague other languages. FEEL keeps them separate so you never accidentally add "1 month" and get an undefined number of days.

### Temporal Arithmetic

| Expression | Result | Type |
|-----------|--------|------|
| `@"2024-03-15" + @"P30D"` | `@"2024-04-14"` | date |
| `@"2024-03-15" + @"P1M"` | `@"2024-04-15"` | date |
| `@"2024-03-15" - @"2024-01-01"` | `@"P74D"` (days and time duration) | duration |
| `@"14:30:00" + @"PT2H"` | `@"16:30:00"` | time |
| `now() - @"P1Y"` | date-time one year ago | date and time |
| `@"P1Y6M" + @"P6M"` | `@"P2Y"` | years and months duration |

### Worked Example 5.4 — Age Calculation

```
{
  birth date: @"1990-07-22",
  age in years: years and months duration(birth date, today()).years,
  is adult: age in years >= 18
}
```

### Worked Example 5.5 — SLA Deadline Check

```
{
  ticket created: @"2024-03-01T09:00:00",
  sla duration: @"P3D",
  deadline: ticket created + sla duration,
  overdue: now() > deadline,
  time remaining: if overdue then @"PT0S" else deadline - now()
}
```

### Worked Example 5.6 — Contract Renewal Window

```
{
  contract start: @"2023-06-01",
  contract term: @"P2Y",
  expiry: contract start + contract term,
  renewal window start: expiry - @"P90D",
  in renewal window: today() >= renewal window start and today() <= expiry
}
```

---

## 5.5 The null Value

`null` pulls triple duty in FEEL: it means "no value," "missing data," and "something went wrong." One sentinel to rule them all.

### Null Is Not Zero, Not Empty, Not False

```
null = 0         // false
null = ""        // false
null = false     // false
null = null      // true
null != null     // false
```

> **GOTCHA — null equality vs SQL:** Unlike SQL, where `NULL = NULL` evaluates to `NULL` (unknown), FEEL's `null = null` evaluates to `true`. FEEL treats null equality as a definite answer. This is a deliberate design choice — it makes null checks simpler (`x = null` works), but it will surprise anyone with SQL habits.

### Null Propagation

`null` is contagious -- almost every operation that touches `null` produces `null`:

```
null + 1         // null
null + "hello"   // null
null > 5         // null
null.name        // null
[1,2,3][null]    // null
```

The exceptions:
- `null = null` → `true`
- `null != null` → `false`
- `and` and `or` short-circuit: `false and null` → `false`, `true or null` → `true`

### Checking for Null

```
x = null         // true if x is null
x != null        // true if x is not null
x instance of Null  // true if x is null (type check form)
```

### Defensive Patterns

```
// Pattern 1: Default value via if/else
if x != null then x else 0

// Pattern 2: get or else (built-in; returns x if non-null, otherwise the default)
get or else(x, 0)

// Pattern 3: Null-safe chain
if Order != null and Order.Customer != null
then Order.Customer.Name
else "Unknown"
```

### Worked Example 5.7 — Defensive Null Handling

A calculation that must handle missing inputs gracefully:

```
{
  income: get or else(Applicant.Monthly Income, 0),
  expenses: get or else(Applicant.Monthly Expenses, 0),
  disposable: income - expenses,
  can afford: if income = 0
              then false
              else disposable >= Required Installment
}
```

---

## 5.6 Type Checking with `instance of`

Sometimes you need to ask "what kind of thing is this?" at runtime. FEEL gives you `instance of`:

```
42 instance of number                      // true
"abc" instance of string                   // true
true instance of boolean                   // true
@"2024-01-01" instance of date             // true
[1, 2, 3] instance of list<number>         // true
["a", 1] instance of list<Any>             // true
{name: "Alice"} instance of context<name: string>  // true
null instance of Null                       // true
```

The type lattice has `Any` at the top (all values conform) and `Null` at the bottom (`null` conforms to every type).

### Practical Use

This comes in handy when your inputs arrive from external systems or JSON payloads and you cannot trust the types:

```
if Score instance of number then
  if Score >= 700 then "Good" else "Review"
else
  "Invalid input"
```

---

## Summary

| Type | Literal Examples | Key Properties |
|------|-----------------|----------------|
| **number** | `42`, `3.14`, `1.2e3` | Decimal128, 34-digit precision, no NaN/Infinity |
| **string** | `"hello"`, `"a\nb"` | Unicode, `+` for concat, no implicit coercion |
| **boolean** | `true`, `false` | Three-valued logic with `null` |
| **date** | `@"2024-03-15"` | `.year`, `.month`, `.day`, `.weekday` |
| **time** | `@"14:30:00Z"` | `.hour`, `.minute`, `.second`, `.time offset` |
| **date and time** | `@"2024-03-15T14:30:00Z"` | All date + time properties |
| **days and time duration** | `@"P2DT3H"` | `.days`, `.hours`, `.minutes`, `.seconds` |
| **years and months duration** | `@"P1Y6M"` | `.years`, `.months` |
| **null** | `null` | Propagates through operations, check with `= null` |

---

## Exercises

**Exercise 5.1:** What is the result of each expression?
- `1/3`
- `decimal(1/3, 4)`
- `"hello" + 42`
- `string length("hello" + " " + "world")`

**Exercise 5.2:** Given `birth date = @"1995-12-25"`, write a FEEL expression that computes the person's age in whole years as of today.

**Exercise 5.3:** What is the result of `true and null and false`? Explain step by step.

**Exercise 5.4:** Write a FEEL context that takes an input `Temperature` (which may be `null`) and produces:
- `status`: "Fever" if Temperature > 38.5, "Normal" if Temperature <= 38.5, "Unknown" if Temperature is null.
- `action`: "Administer treatment" if fever, "Continue monitoring" otherwise.

**Exercise 5.5:** Given `Contract Start = @"2022-01-15"` and `Term = @"P3Y"`, compute the expiry date, the number of days until expiry (as a duration), and whether the contract has expired.

---

## What Comes Next

Now that you know the atoms, Chapter 6 shows you how to combine them into expressions: arithmetic, comparisons, conditionals, and the special sub-language of unary tests that powers decision tables.

---

[Previous: Chapter 4: From Rules to Flows](../part-2-business-logic/chapter-04-from-rules-to-flows.md) | [Next: Chapter 6: Expressions](chapter-06-expressions.md)
