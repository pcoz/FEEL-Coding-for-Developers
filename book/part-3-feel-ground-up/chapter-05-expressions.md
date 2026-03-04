# Chapter 5: Expressions — Combining Atoms

> *"FEEL has no statements. Everything is an expression. Every expression produces a value."*

---

## 5.1 Arithmetic

### Operators

| Operator | Meaning | Example | Result |
|----------|---------|---------|--------|
| `+` | Addition / string concat | `3 + 4` | `7` |
| `-` | Subtraction | `10 - 3` | `7` |
| `*` | Multiplication | `5 * 6` | `30` |
| `/` | Division | `7 / 2` | `3.5` |
| `**` | Exponentiation | `2 ** 10` | `1024` |
| `-` (unary) | Negation | `-5` | `-5` |

### Operator Precedence (Low to High)

1. `or`
2. `and`
3. Comparison (`=`, `!=`, `<`, `<=`, `>`, `>=`, `between`, `in`)
4. Addition, Subtraction (`+`, `-`)
5. Multiplication, Division (`*`, `/`)
6. Exponentiation (`**`)
7. Unary negation (`-`)
8. Path expression (`.`), filter (`[]`), function invocation (`()`)

All binary operators are **left-associative**: `a - b - c` is `(a - b) - c`.

> **GOTCHA — Exponentiation and Negation:** In standard mathematics and most languages, `-4 ** 2` = `-(4**2)` = `-16`. In FEEL, `-4 ** 2` = `(-4) ** 2` = `16`. The unary minus binds *before* exponentiation because negation has higher precedence. Always use parentheses: `-(4 ** 2)` for `-16`, or `(-4) ** 2` for `16`.

### Arithmetic with Different Types

FEEL defines arithmetic not just for numbers but for temporal types:

| `type(a)` | `type(b)` | `a + b` | `a - b` |
|-----------|-----------|---------|---------|
| number | number | number | number |
| string | string | string (concat) | *not defined* → null |
| date/time | duration | date/time (shifted) | date/time (shifted) |
| date/time | date/time | *not defined* → null | duration (difference) |
| duration | duration | duration (sum) | duration (difference) |
| duration | number | *not defined* → null | *not defined* → null |

For multiplication and division, FEEL also supports:
- `duration * number` → scaled duration
- `duration / number` → scaled duration
- `duration / duration` → number (ratio)

### Worked Example 5.1 — Compound Interest

```
{
  Principal: 10000,
  Rate: 0.05,
  Years: 10,
  Compound Monthly: Principal * (1 + Rate / 12) ** (Years * 12)
}
```

Result: `Compound Monthly` = `16470.09...`

### Worked Example 5.2 — Duration Arithmetic

```
{
  start: @"2024-01-15",
  end: @"2024-06-15",
  elapsed: end - start,
  elapsed days: elapsed.days,
  halfway: start + elapsed / 2
}
```

---

## 5.2 Comparisons

### Comparison Operators

| Operator | Meaning | Example | Result |
|----------|---------|---------|--------|
| `=` | Equal | `5 = 5` | `true` |
| `!=` | Not equal | `5 != 3` | `true` |
| `<` | Less than | `3 < 5` | `true` |
| `<=` | Less than or equal | `5 <= 5` | `true` |
| `>` | Greater than | `7 > 3` | `true` |
| `>=` | Greater than or equal | `7 >= 7` | `true` |

### Equality Is by Value

```
[1, 2, 3] = [1, 2, 3]                // true (same contents)
{name: "Alice"} = {name: "Alice"}    // true (same entries)
1 = 1.0                               // true (same numeric value)
```

### Cross-Type Comparison Produces Null

```
"1" = 1        // null (not false!)
5 < "ten"      // null
true = 1       // null
```

> **GOTCHA:** In JavaScript, `"1" == 1` is `true` (type coercion). In Python, `"1" == 1` is `False` (no coercion but definite answer). In FEEL, `"1" = 1` is `null` — FEEL refuses to answer when the comparison is meaningless.

### The `between` Operator

```
x between 5 and 10
```

is syntactic sugar for `x >= 5 and x <= 10`. Both endpoints are inclusive.

### The `in` Operator

The `in` operator tests membership. It is versatile:

```
// In a list of values
5 in (1, 3, 5, 7, 9)                     // true

// In a range
5 in [1..10]                              // true (inclusive both ends)
5 in (1..10)                              // true (exclusive both ends)
5 in [5..10]                              // true (inclusive start)
5 in (5..10]                              // false (exclusive start)

// In a set of ranges
5 in (<3, >7)                             // false
5 in (<=5, >=10)                          // true

// Negated
5 in (not(1, 2, 3))                       // true — 5 is not 1, 2, or 3
```

### Worked Example 5.3 — Risk Classification

```
if Credit Score between 750 and 850 then "Excellent"
else if Credit Score between 650 and 749 then "Good"
else if Credit Score between 550 and 649 then "Fair"
else if Credit Score >= 300 then "Poor"
else null
```

Alternative using `in` with ranges:

```
if Credit Score in [750..850] then "Excellent"
else if Credit Score in [650..749] then "Good"
else if Credit Score in [550..649] then "Fair"
else if Credit Score in [300..549] then "Poor"
else null
```

---

## 5.3 Conditional Expressions

### Syntax

```
if condition then expression1 else expression2
```

- The `else` clause is **mandatory**. There is no dangling `if`.
- The condition must evaluate to `true` for `expression1`. Both `false` and `null` take `expression2`.
- Conditionals can be nested (but prefer decision tables for 3+ branches).

### Worked Example 5.4 — Tiered Pricing

```
if Quantity >= 100 then Unit Price * 0.80
else if Quantity >= 50 then Unit Price * 0.90
else if Quantity >= 10 then Unit Price * 0.95
else Unit Price
```

---

## 5.4 Unary Tests — The Language Within the Language

Unary tests are a special sub-language of FEEL, used exclusively in decision table input cells. They describe a condition on an implicit value — the value being tested by the column.

### How They Work

When a decision table has an input column with expression `Age` and a cell contains `>= 18`, the engine evaluates:

```
Age in (>= 18)
```

The unary test `>= 18` is the right-hand side of an implicit `in` expression.

### Unary Test Syntax

| Unary Test | Meaning | Equivalent FEEL |
|-----------|---------|----------------|
| `5` | Equals 5 | `? = 5` |
| `"Gold"` | Equals "Gold" | `? = "Gold"` |
| `< 100` | Less than 100 | `? < 100` |
| `>= 18` | At least 18 | `? >= 18` |
| `[18..65]` | Between 18 and 65 inclusive | `? >= 18 and ? <= 65` |
| `(0..100)` | Between 0 and 100 exclusive | `? > 0 and ? < 100` |
| `"Gold", "Silver"` | One of these values | `? = "Gold" or ? = "Silver"` |
| `not("Declined")` | Not this value | `? != "Declined"` |
| `not(< 0)` | Not negative | `? >= 0` |
| `-` | Any value (wildcard) | `true` |

The `?` symbol is used in FEEL to reference the implicit value in certain contexts. In decision table cells, it is always implicit.

### Worked Example 5.5 — Reading a Decision Table

Given this decision table:

| U | Age | Employment Status | Eligible |
|---|-----|------------------|----------|
| 1 | >= 18 | "Employed", "Self-employed" | true |
| 2 | >= 18 | "Retired" | true |
| 3 | < 18 | - | false |
| 4 | >= 18 | not("Employed", "Self-employed", "Retired") | false |

Reading rule 1: "If Age >= 18 AND Employment Status is 'Employed' or 'Self-employed', then Eligible = true."

Rule 3: "If Age < 18, regardless of Employment Status (the `-` wildcard), then Eligible = false."

---

## Summary

| Construct | Syntax | Key Point |
|-----------|--------|-----------|
| Arithmetic | `+`, `-`, `*`, `/`, `**` | Works for numbers, strings (+), and temporal types |
| Comparison | `=`, `!=`, `<`, `<=`, `>`, `>=` | Cross-type comparison → `null` |
| Between | `x between a and b` | Sugar for `x >= a and x <= b` |
| In | `x in (values/ranges)` | Flexible membership test |
| Conditional | `if c then a else b` | `else` is mandatory; null condition → else branch |
| Unary tests | `>= 18`, `[1..10]`, `-` | Used in decision table cells, implicit value |

---

## Exercises

**Exercise 5.1:** What is the result of `- 3 ** 2`? What about `(-3) ** 2`? What about `-(3 ** 2)`?

**Exercise 5.2:** Write a FEEL expression that computes the Body Mass Index: `weight / (height ** 2)`, where weight is in kg and height is in meters. Then classify it: "Underweight" (<18.5), "Normal" (18.5–24.9), "Overweight" (25–29.9), "Obese" (>=30).

**Exercise 5.3:** What is the result of `"abc" > 5`? Why?

**Exercise 5.4:** Write the unary tests for a decision table input column called `Temperature` that should match: (a) exactly 37.0, (b) between 37.5 and 38.5 exclusive, (c) 39 or above, (d) any value.

---

## What Comes Next

Chapter 6 introduces FEEL's compound data structures: lists (and their powerful filtering syntax), contexts (step-by-step calculations), and ranges (first-class interval values).
