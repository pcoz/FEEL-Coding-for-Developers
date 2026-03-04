# Chapter 6: Expressions — Combining Atoms

> *"FEEL has no statements. Everything is an expression. Every expression produces a value."*

---

How do you combine values into computations? What operators does FEEL have, and how do conditionals, comparisons, and logic work when there are no statements — only expressions? FEEL has no `void`, no side effects, no imperative control flow. Every piece of code you write evaluates to a result, which makes expressions composable, testable, and predictable in ways that imperative code is not.

This chapter covers the full expression language — arithmetic, logic, comparisons, conditionals, and the rules that govern how they all fit together.

---

## 6.1 Arithmetic

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

All binary operators are **left-associative** -- `a - b - c` means `(a - b) - c`, just like you would expect.

> **GOTCHA — Exponentiation and Negation:** In standard mathematics and most languages, `-4 ** 2` = `-(4**2)` = `-16`. In FEEL, `-4 ** 2` = `(-4) ** 2` = `16`. The unary minus binds *before* exponentiation because negation has higher precedence. Always use parentheses: `-(4 ** 2)` for `-16`, or `(-4) ** 2` for `16`.

### Arithmetic with Different Types

Here is where FEEL starts to feel like a domain-specific superpower. The `+` and `-` operators are not limited to numbers -- they work across temporal types too:

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

### Worked Example 6.1 — Compound Interest

```
{
  Principal: 10000,
  Rate: 0.05,
  Years: 10,
  Compound Monthly: Principal * (1 + Rate / 12) ** (Years * 12)
}
```

Result: `Compound Monthly` = `16470.09...`

### Worked Example 6.2 — Duration Arithmetic

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

## 6.2 Comparisons

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

No reference equality headaches here. FEEL compares what things *contain*, not where they live in memory:

```
[1, 2, 3] = [1, 2, 3]                // true (same contents)
{name: "Alice"} = {name: "Alice"}    // true (same entries)
1 = 1.0                               // true (same numeric value)
```

### Cross-Type Comparison Produces Null

Comparing apples and oranges? FEEL will not coerce, will not throw, and will not guess. It gives you `null` -- the honest answer meaning "this comparison is meaningless":

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

The `in` operator is the Swiss Army knife of FEEL comparisons -- it tests membership against values, ranges, and combinations of both:

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

### Worked Example 6.3 — Risk Classification

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

## 6.3 Conditional Expressions

### Syntax

```
if condition then expression1 else expression2
```

- The `else` clause is **mandatory**. No dangling `if`, no forgotten branch, no accidental `undefined`. Every `if` must say what happens otherwise.
- The condition must evaluate to `true` for `expression1`. Both `false` and `null` take `expression2`.
- You can nest conditionals, but once you hit three or more branches, a decision table is almost always clearer.

### Worked Example 6.4 — Tiered Pricing

```
if Quantity >= 100 then Unit Price * 0.80
else if Quantity >= 50 then Unit Price * 0.90
else if Quantity >= 10 then Unit Price * 0.95
else Unit Price
```

---

## 6.4 Unary Tests — The Language Within the Language

If you have looked at a decision table and wondered how a cell containing just `>= 18` knows *what* to compare against, this section is the answer.

Unary tests are a compact sub-language used exclusively inside decision table input cells. Each test describes a condition against an implicit value -- the value supplied by the column's input expression.

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

The `?` symbol represents the implicit value being tested. Inside decision table cells, you never write `?` -- it is always implied. But understanding it helps you read the "Equivalent FEEL" column above.

### Worked Example 6.5 — Reading a Decision Table

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

**Exercise 6.1:** What is the result of `- 3 ** 2`? What about `(-3) ** 2`? What about `-(3 ** 2)`?

**Exercise 6.2:** Write a FEEL expression that computes the Body Mass Index: `weight / (height ** 2)`, where weight is in kg and height is in meters. Then classify it: "Underweight" (<18.5), "Normal" (18.5–24.9), "Overweight" (25–29.9), "Obese" (>=30).

**Exercise 6.3:** What is the result of `"abc" > 5`? Why?

**Exercise 6.4:** Write the unary tests for a decision table input column called `Temperature` that should match: (a) exactly 37.0, (b) between 37.5 and 38.5 exclusive, (c) 39 or above, (d) any value.

---

## What Comes Next

You can do a lot with single values and expressions, but real business data comes in collections. Chapter 7 introduces lists, contexts, and ranges -- the compound structures that let you model anything.

---

[Previous: Chapter 5: Values and Types](chapter-05-values-and-types.md) | [Next: Chapter 7: Collections](chapter-07-collections.md)
