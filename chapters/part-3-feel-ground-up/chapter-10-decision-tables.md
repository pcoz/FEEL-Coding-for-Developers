# Chapter 10: Decision Tables in Depth

> *"A decision table is worth a thousand if-else statements."*

---

## 10.1 Anatomy of a Decision Table

Before you can wield decision tables effectively, you need to know what you're looking at. Here are the structural components:

```
┌──────────────────────────────────────────────────────────────┐
│  Decision Name                                          [U]  │
├───────────────┬───────────────┬───────────────────────────────┤
│  Input Expr 1 │  Input Expr 2 │  Output Label                │
│  (values)     │  (values)     │  (values)                    │
├───────────────┼───────────────┼───────────────────────────────┤
│ 1  unary test │  unary test   │  output expression           │
│ 2  unary test │  unary test   │  output expression           │
│ 3  unary test │  unary test   │  output expression           │
└───────────────┴───────────────┴───────────────────────────────┘
```

- **Input expressions**: FEEL expressions evaluated against the input data. Each becomes a column.
- **Input values** (optional): Constrain what values are valid in the column. Used for completeness checking.
- **Input entries**: Unary tests — the conditions in each rule row.
- **Output label**: The name of the result.
- **Output values** (optional): Constrain the allowed outputs. Used by Priority and Output Order hit policies.
- **Output entries**: FEEL expressions producing the result.
- **Hit policy**: The letter in brackets `[U]`.
- **Rules**: Each row is a rule. A rule fires when *all* its input entries match simultaneously.
- **`-`** (dash): The wildcard — matches any value.

---

## 10.2 Hit Policies — Complete Guide

### Unique (U)

**Guarantee:** Exactly one rule matches for any valid input combination.

**Use when:** Rules are mutually exclusive by design.

**Worked Example 10.1 — Tax Bracket:**

| U | Taxable Income | Tax Rate |
|---|---------------|----------|
| 1 | [0..10000] | 0.00 |
| 2 | (10000..40000] | 0.15 |
| 3 | (40000..80000] | 0.25 |
| 4 | (80000..150000] | 0.30 |
| 5 | > 150000 | 0.35 |

Every income lands in exactly one bracket -- no gaps, no overlaps. If two rules could match the same income, your engine will throw an error, which is exactly what you want. Unique tables are self-checking.

### Any (A)

**Guarantee:** Multiple rules may match, but they all produce the same output.

**Use when:** Redundant rules improve readability.

**Worked Example 10.2 — Safety Classification:**

| A | Material | Temperature | Classification |
|---|----------|-------------|----------------|
| 1 | "Explosive" | - | "Danger" |
| 2 | - | > 500 | "Danger" |
| 3 | "Explosive" | > 500 | "Danger" |
| 4 | not("Explosive") | <= 500 | "Safe" |

Rules 1, 2, and 3 all match for an explosive material at temperature 600. But they all agree: "Danger." That's fine -- Any doesn't care *how many* rules match, only that they don't *contradict* each other. If any two overlapping rules disagreed, the engine would flag an error.

### First (F)

**Guarantee:** The first matching rule (by row order) wins.

**Use when:** Rules are ordered from most specific to most general, like `if/else if/else`.

**Worked Example 10.3 — Customer Greeting:**

| F | Customer Tier | Purchase Count | Greeting |
|---|--------------|----------------|----------|
| 1 | "Platinum" | > 100 | "Welcome back, valued Platinum member!" |
| 2 | "Platinum" | - | "Welcome, Platinum member!" |
| 3 | "Gold" | - | "Welcome, Gold member!" |
| 4 | - | - | "Welcome!" |

A Platinum member with 150 purchases matches rules 1, 2, and 4 -- but rule 1 wins because it appears first. Think of it as a `switch` with fallthrough disabled: first match takes it.

### Priority (P)

**Guarantee:** Multiple rules may match with different outputs. The output with the highest priority (earliest in the output values list) wins.

**Use when:** You want the "most important" result regardless of rule order. You define the ranking yourself in the output values list.

**Worked Example 10.4 — Discount Priority:**

Output values (ordered by priority): `0.25, 0.20, 0.15, 0.10, 0.05, 0`

| P | Customer Tier | Order Amount | Discount |
|---|--------------|-------------|----------|
| 1 | "Gold" | > 200 | 0.25 |
| 2 | "Gold" | > 100 | 0.20 |
| 3 | "Gold" | - | 0.15 |
| 4 | "Silver" | > 200 | 0.15 |
| 5 | "Silver" | - | 0.10 |
| 6 | - | - | 0 |

A Gold customer with order amount 250 matches rules 1, 2, 3, and 6. The outputs are 0.25, 0.20, 0.15, 0. Priority picks 0.25 because it appears first in the output values list. Notice: this is *not* about rule order -- it's about output ranking. You could shuffle the rules and get the same answer.

### Collect (C)

**Guarantee:** All matching rules fire. Outputs are collected into a list.

**Worked Example 10.5 — Applicable Fees:**

| C | Account Type | Balance | Fee |
|---|-------------|---------|-----|
| 1 | "Checking" | < 1000 | "Low Balance Fee" |
| 2 | "Checking" | - | "Monthly Maintenance" |
| 3 | - | < 0 | "Overdraft Fee" |

A checking account with balance -50 matches all three rules. Result: `["Low Balance Fee", "Monthly Maintenance", "Overdraft Fee"]`. Unlike the single-hit policies above, Collect doesn't pick a winner -- it keeps everything.

### Collect with Aggregation (C+, C#, C<, C>)

| Hit Policy | Aggregation | Example Use |
|-----------|-------------|-------------|
| **C+** | Sum of outputs | Scorecard: sum of partial scores |
| **C#** | Count of matches | "How many rules apply?" |
| **C<** | Minimum output | "Lowest applicable rate" |
| **C>** | Maximum output | "Highest applicable fee" |

**Worked Example 10.6 — Application Risk Scorecard (C+):**

| C+ | Age | Partial Score |
|----|-----|---------------|
| 1 | [18..25) | 30 |
| 2 | [25..35) | 45 |
| 3 | [35..50) | 50 |
| 4 | [50..65) | 40 |
| 5 | >= 65 | 25 |

| C+ | Employment Status | Partial Score |
|----|------------------|---------------|
| 6 | "Employed" | 50 |
| 7 | "Self-employed" | 35 |
| 8 | "Retired" | 30 |
| 9 | "Unemployed" | 10 |

*Note:* In practice, both dimensions would be columns in a single table. The engine sums the Partial Score from every matching rule. An employed 30-year-old? 45 (age) + 50 (employment) = 95. This is the classic scorecard pattern, and `C+` is tailor-made for it.

### Rule Order (R) and Output Order (O)

**R** returns matching outputs in the order the rules appear in the table.

**Example (R):** A notification routing table:

| R | Event Type | Channel |
|---|-----------|---------|
| 1 | "Urgent" | "SMS" |
| 2 | "Urgent" | "Email" |
| 3 | - | "Email" |

For `Event Type = "Urgent"`: result is `["SMS", "Email"]` (rules 1 and 2 match, returned in rule order). The SMS goes out first because it is rule 1.

**O** returns matching outputs in the order defined by the output values list.

**Example (O):** If the same table used hit policy O with output values ordered as `"Email", "SMS"`, the result for `"Urgent"` would be `["Email", "SMS"]` — reordered by the output values list, regardless of rule order.

Both produce a list, like Collect, but with a guaranteed ordering. The difference matters when downstream logic depends on which item comes first. Note: Output Order (O) **requires** output values to be defined -- without them, the engine has no ordering to apply.

### When No Rule Matches

What happens when *none* of your rules fire? No exception. No warning. Just a silent default:

- **Single-hit policies** (U, A, F, P): the result is `null`.
- **Multi-hit policies** (C, C+, C#, C<, C>, R, O): the result is an empty list `[]` (or `0` for C+/C#, `null` for C</C>).

This is why completeness matters (Section 10.4). An incomplete table silently returns `null` for uncovered inputs, and that `null` can propagate through your entire decision model before anyone notices.

---

## 10.3 Multi-Output Decision Tables

Sometimes a single output column isn't enough. A decision table can have multiple output columns, and when it does, each matching rule produces a context (think of it as returning an object instead of a scalar):

| U | Vehicle Category | Annual Mileage | Base Premium | Surcharge |
|---|-----------------|----------------|-------------|-----------|
| 1 | "Sedan" | <= 15000 | 600 | 0 |
| 2 | "Sedan" | > 15000 | 600 | 120 |
| 3 | "SUV" | <= 15000 | 800 | 0 |
| 4 | "SUV" | > 15000 | 800 | 160 |

Result for a Sedan with 20000 km: `{Base Premium: 600, Surcharge: 120}`.

> **IMPORTANT:** Multi-output decision tables do not support aggregation hit policies (C+, C#, C<, C>). Only U, A, P, F, C (without aggregation), and R are allowed.

---

## 10.4 Completeness and Consistency

### Completeness

A decision table is **complete** if every possible input combination matches at least one rule. Most DMN modeling tools can check this for you automatically.

Two practical habits that prevent gaps:
- Add a catch-all rule at the bottom: `-` (wildcard) in every input column. This is your safety net.
- Set a default output value for tables that should never return `null`.

### Consistency (for Unique policy)

A decision table with hit policy U is **consistent** if no input combination matches more than one rule. Again, tools can verify this automatically.

The most common mistake: overlapping range boundaries. `[0..10]` and `[10..20]` both match 10. Fix it with an exclusive bound on one side: `[0..10]` and `(10..20]`.

---

## 10.5 Decision Tables and FEEL Together

Decision tables don't have to live in isolation. In real models, they're often embedded in larger FEEL expressions via contexts -- mixing table-driven rules with procedural logic:

```
{
  risk category: Risk Category Table,
  affordability: {
    disposable: Income - Expenses,
    can afford: disposable >= Required Installment
  },
  eligible: risk category != "Decline" and affordability.can afford
}
```

Here, `Risk Category Table` is a decision table that produces a string. The surrounding context adds computed fields and a final eligibility check. This is how real decision models work: tables handle the rule-heavy parts, FEEL handles the glue.

---

## Summary

| Hit Policy | Letter | Returns | Multiple Matches |
|-----------|--------|---------|-----------------|
| Unique | U | Single value | Error if > 1 match |
| Any | A | Single value | OK if all agree |
| First | F | Single value | First match wins |
| Priority | P | Single value | Highest priority wins |
| Collect | C | List | All matches |
| Collect Sum | C+ | Number | Sum of matches |
| Collect Count | C# | Number | Count of matches |
| Collect Min | C< | Value | Minimum match |
| Collect Max | C> | Value | Maximum match |
| Rule Order | R | List | In rule order |
| Output Order | O | List | In output value order |

### Choosing the Right Hit Policy

| If you need... | Use |
|---------------|-----|
| Exactly one rule to match (and you want the engine to enforce this) | **U** (Unique) |
| Multiple rules may match but they all produce the same output | **A** (Any) |
| Rules overlap and the first match should win (order-dependent) | **F** (First) |
| Rules overlap and the "best" output should win (output-priority-dependent) | **P** (Priority) |
| All matching outputs collected into a list | **C** (Collect) |
| A numeric summary (sum, count, min, max) of matching outputs | **C+**, **C#**, **C<**, **C>** |
| All matching outputs in a predictable order | **R** (Rule Order) or **O** (Output Order) |

When in doubt, start with **U**. It's the safest choice because the engine will flag overlapping rules as an error, catching bugs before they reach production. Reach for **F** when you intentionally want rules to overlap (like a cascading default). Reach for **C+** for scorecards.

---

## Exercises

**Exercise 10.1:** Design a decision table (hit policy U) for determining the shipping speed based on: order priority ("Standard", "Express", "Overnight") and destination distance (<=100km, 100-500km, >500km). The output should be estimated delivery days.

**Exercise 10.2:** Design a scorecard (hit policy C+) for a job application. Inputs: years of experience, highest education level, and number of references. Each factor contributes a partial score.

**Exercise 10.3:** A decision table has hit policy U and two overlapping rules. What happens at runtime? How would you fix it?

---

## What Comes Next

You've now seen everything decision tables can do. But a decision table is just one kind of *boxed expression*. Chapter 11 covers the full set -- the visual notation that wraps FEEL expressions into composable, graphical building blocks that even non-developers can review.

---

[Previous: Chapter 9: Functions](chapter-09-functions.md) | [Next: Chapter 11: Boxed Expressions](chapter-11-boxed-expressions.md)
