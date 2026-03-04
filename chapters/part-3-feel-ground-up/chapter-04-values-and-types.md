# Chapter 4: Values and Types — The Atoms of FEEL

> *"Every expression evaluates to a value. Every value has a type. Understand the types and you understand the language."*

---

FEEL has a small, carefully chosen set of data types. Unlike general-purpose languages that distinguish between integers, floats, longs, and doubles, FEEL keeps things simple. There is one number type. There is one string type. Dates and durations are first-class citizens, not afterthoughts.

This chapter covers every FEEL data type, its literal syntax, and the operations defined on it.

---

## 4.1 Numbers

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

FEEL numbers are based on **IEEE 754-2008 Decimal128**: 34 decimal digits of precision. This is not a floating-point type in the way JavaScript's `number` is. There are no binary rounding surprises:

```
0.1 + 0.2 = 0.3    // true in FEEL (not 0.30000000000000004)
```

There is no separate integer type. `42` and `42.0` are the same value: `42 = 42.0` evaluates to `true`.

There is no `NaN`, no `Infinity`, no `-0`. When a numeric operation cannot produce a valid result, it produces `null`.

### Rosetta Stone

| Operation | FEEL | JavaScript | Python | SQL |
|-----------|------|-----------|--------|-----|
| Literal | `42` | `42` | `42` | `42` |
| Division by zero | `1/0` → `null` | `1/0` → `Infinity` | `1/0` → `ZeroDivisionError` | `1/0` → error |
| Precision | 34 decimal digits | ~15 binary digits | Arbitrary (int) / ~15 (float) | Varies |
| `0.1 + 0.2` | `0.3` | `0.30000000000000004` | `0.30000000000000004` | `0.3` (if DECIMAL) |

### Rounding Functions

FEEL provides explicit rounding for financial applications:

```
decimal(1/3, 2)              // 0.33 — round to 2 decimal places (half-even)
round up(5.5, 0)             // 6
round down(5.5, 0)           // 5
round half up(2.5, 0)        // 3
round half down(2.5, 0)      // 2
```

> **GOTCHA:** The `decimal()` function uses **half-even** rounding (banker's rounding), not half-up. `decimal(2.5, 0)` is `2`, not `3`. For financial applications that require half-up, use `round half up()` explicitly.

### Worked Example 4.1 — Financial Rounding

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

## 4.2 Strings

### Literal Syntax

```
"hello world"
"She said \"hello\""
"line 1\nline 2"
"tab\there"
"unicode: \u00E9"       // é
```

Strings are enclosed in double quotes. Single quotes are not string delimiters.

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

### Worked Example 4.2 — Building a Notification Message

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

## 4.3 Booleans and Ternary Logic

### Literal Syntax

```
true
false
```

There is no literal for the third truth value — it is `null`, the absence of a value.

### The Three-Valued Truth Tables

FEEL uses **ternary logic**, like SQL. This is the single most important thing to understand about FEEL's type system:

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

This means null conditions always take the `else` branch. This is a safety feature — FEEL never acts on uncertain data.

### Rosetta Stone

| FEEL | JavaScript | Python | SQL |
|------|-----------|--------|-----|
| `true and null` → `null` | `true && null` → `null` (falsy) | `True and None` → `None` | `TRUE AND NULL` → `NULL` |
| `false or null` → `null` | `false \|\| null` → `null` | `False or None` → `None` | `FALSE OR NULL` → `NULL` |
| `false and null` → `false` | `false && null` → `false` | `False and None` → `False` | `FALSE AND NULL` → `FALSE` |
| `not(null)` → `null` | `!null` → `true` (!) | `not None` → `True` (!) | `NOT NULL` → `NULL` |

> **GOTCHA:** JavaScript's `!null` is `true` — JavaScript treats `null` as falsy. FEEL's `not(null)` is `null`. FEEL never guesses.

### Worked Example 4.3 — Null-Safe Eligibility Check

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

## 4.4 Dates, Times, and Durations

FEEL treats temporal values as first-class citizens. There are five temporal types.

### 4.4.1 Date

A calendar date (year, month, day) without time or timezone.

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

### 4.4.2 Time

A time of day, optionally with timezone.

```
@"14:30:00"                          // local time
@"14:30:00+02:00"                    // with offset
@"14:30:00@Europe/Paris"             // with IANA timezone
time(14, 30, 0)                      // component form
time(14, 30, 0, @"PT2H")            // with offset as duration
```

Properties: `hour`, `minute`, `second`, `time offset`, `timezone`.

### 4.4.3 Date and Time

A combined date and time.

```
@"2024-03-15T14:30:00"               // local
@"2024-03-15T14:30:00Z"              // UTC
@"2024-03-15T14:30:00+02:00"         // with offset
@"2024-03-15T14:30:00@Europe/Paris"  // with IANA timezone
date and time(date("2024-03-15"), time("14:30:00"))   // from components
```

Properties: all date properties + all time properties.

### 4.4.4 Days and Time Duration

A duration expressed in days, hours, minutes, and seconds.

```
@"P2D"                    // 2 days
@"PT3H"                   // 3 hours
@"P1DT12H30M"             // 1 day, 12 hours, 30 minutes
@"PT0.5S"                 // half a second
duration("P2DT3H")        // function form
```

Properties: `days`, `hours`, `minutes`, `seconds`.

### 4.4.5 Years and Months Duration

A duration expressed in years and months.

```
@"P1Y"                    // 1 year
@"P6M"                    // 6 months
@"P1Y6M"                  // 1 year, 6 months
@"P18M"                   // normalised to P1Y6M
duration("P1Y6M")         // function form
```

Properties: `years`, `months`.

### Why Two Duration Types?

Years and months have variable lengths (28–31 days, 365–366 days). Days and hours have fixed lengths. These two kinds of duration cannot be mixed because `@"P1M"` is not a fixed number of days. FEEL keeps them separate to prevent ambiguity.

### Temporal Arithmetic

| Expression | Result | Type |
|-----------|--------|------|
| `@"2024-03-15" + @"P30D"` | `@"2024-04-14"` | date |
| `@"2024-03-15" + @"P1M"` | `@"2024-04-15"` | date |
| `@"2024-03-15" - @"2024-01-01"` | `@"P74D"` (days and time duration) | duration |
| `@"14:30:00" + @"PT2H"` | `@"16:30:00"` | time |
| `now() - @"P1Y"` | date-time one year ago | date and time |
| `@"P1Y6M" + @"P6M"` | `@"P2Y"` | years and months duration |

### Worked Example 4.4 — Age Calculation

```
{
  birth date: @"1990-07-22",
  age in years: years and months duration(birth date, today()).years,
  is adult: age in years >= 18
}
```

### Worked Example 4.5 — SLA Deadline Check

```
{
  ticket created: @"2024-03-01T09:00:00",
  sla duration: @"P3D",
  deadline: ticket created + sla duration,
  overdue: now() > deadline,
  time remaining: if overdue then @"PT0S" else deadline - now()
}
```

### Worked Example 4.6 — Contract Renewal Window

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

## 4.5 The null Value

`null` is FEEL's universal sentinel for "no value", "missing data", and "error occurred."

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

Almost every operation on `null` produces `null`:

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

### Worked Example 4.7 — Defensive Null Handling

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

## 4.6 Type Checking with `instance of`

FEEL supports runtime type checking:

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

Type checking is useful when inputs may arrive in unexpected types (e.g., from external systems or JSON payloads):

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

**Exercise 4.1:** What is the result of each expression?
- `1/3`
- `decimal(1/3, 4)`
- `"hello" + 42`
- `string length("hello" + " " + "world")`

**Exercise 4.2:** Given `birth date = @"1995-12-25"`, write a FEEL expression that computes the person's age in whole years as of today.

**Exercise 4.3:** What is the result of `true and null and false`? Explain step by step.

**Exercise 4.4:** Write a FEEL context that takes an input `Temperature` (which may be `null`) and produces:
- `status`: "Fever" if Temperature > 38.5, "Normal" if Temperature <= 38.5, "Unknown" if Temperature is null.
- `action`: "Administer treatment" if fever, "Continue monitoring" otherwise.

**Exercise 4.5:** Given `Contract Start = @"2022-01-15"` and `Term = @"P3Y"`, compute the expiry date, the number of days until expiry (as a duration), and whether the contract has expired.

---

## What Comes Next

Chapter 5 teaches you to combine these atomic values into expressions: arithmetic, comparisons, conditionals, and the special sub-language of unary tests used inside decision tables.
