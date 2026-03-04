# Chapter 11: Boxed Expressions — Composing Logic Visually

> *"A boxed expression is a FEEL expression you can draw on a whiteboard."*

---

## 11.1 What Are Boxed Expressions?

In DMN, all decision logic is represented as **boxed expressions** — visual, structured representations of FEEL expressions. The "box" is a graphical frame that:

1. Names the expression (the name box at the top).
2. Shows the expression's structure (the body).
3. Makes the logic reviewable by non-developers.

Boxed expressions are defined recursively: a boxed expression can contain other boxed expressions. The top-level boxed expression is always named after the DRG element (decision, BKM) it belongs to.

---

## 11.2 The Complete Catalog

### 11.2.1 Boxed Literal Expression

The simplest form: a named FEEL expression.

```
┌────────────────────────────────┐
│  Monthly Payment               │
├────────────────────────────────┤
│  Amount * Rate / 12            │
└────────────────────────────────┘
```

FEEL equivalent: `Amount * Rate / 12`

### 11.2.2 Boxed Context

A sequence of named entries, evaluated top to bottom.

```
┌────────────────────────────────────┐
│  Affordability                      │
├──────────────────┬─────────────────┤
│  Disposable      │ Income - Costs  │
│  Factor          │ if Risk = "High"│
│                  │ then 0.6        │
│                  │ else 0.8        │
│  Affordable      │ Disposable *    │
│                  │ Factor >=       │
│                  │ Installment     │
└──────────────────┴─────────────────┘
```

FEEL equivalent:
```
{
  Disposable: Income - Costs,
  Factor: if Risk = "High" then 0.6 else 0.8,
  Affordable: Disposable * Factor >= Installment
}
```

The final entry (without a key) is the result of the context.

### 11.2.3 Boxed List

A list of expressions.

```
┌──────────────────┐
│  Risk Factors     │
├──────────────────┤
│  Age Factor       │
│  Income Factor    │
│  History Factor   │
└──────────────────┘
```

FEEL equivalent: `[Age Factor, Income Factor, History Factor]`

### 11.2.4 Relation

A tabular representation of a list of same-shape contexts. Think of it as a table of data (not a decision table — no hit policy, no conditions).

```
┌────────────────────────────────────┐
│  Exchange Rates                     │
├──────────┬──────────┬──────────────┤
│ Currency │ Rate     │ Last Updated │
├──────────┼──────────┼──────────────┤
│ "EUR"    │ 1.08     │ @"2024-03-01"│
│ "GBP"    │ 1.27     │ @"2024-03-01"│
│ "JPY"    │ 0.0067   │ @"2024-03-01"│
└──────────┴──────────┴──────────────┘
```

FEEL equivalent:
```
[
  {Currency: "EUR", Rate: 1.08, Last Updated: @"2024-03-01"},
  {Currency: "GBP", Rate: 1.27, Last Updated: @"2024-03-01"},
  {Currency: "JPY", Rate: 0.0067, Last Updated: @"2024-03-01"}
]
```

### 11.2.5 Boxed Function Definition

Defines a reusable function with named parameters and a body.

```
┌────────────────────────────────┐
│  PMT                            │
├────────────────────────────────┤
│  function(Rate, Term, Amount)   │
├────────────────────────────────┤
│  {                              │
│    Monthly Rate: Rate / 12,     │
│    N: Term * 12,                │
│    PMT: Amount * Monthly Rate / │
│         (1 - (1 + Monthly Rate) │
│         ** (-N))                │
│  }                              │
└────────────────────────────────┘
```

### 11.2.6 Boxed Invocation

Calls a Business Knowledge Model (BKM) with named arguments.

```
┌────────────────────────────────────┐
│  Required Monthly Installment       │
├────────────────────────────────────┤
│  Installment Calculation            │
├─────────────────┬──────────────────┤
│  Rate           │ Product.Rate     │
│  Term           │ Product.Term     │
│  Amount         │ Product.Amount   │
└─────────────────┴──────────────────┘
```

This calls the `Installment Calculation` BKM with three named arguments.

### 11.2.7 Decision Table

Already covered in depth in Chapter 10. The most common boxed expression type.

### 11.2.8 Boxed Conditional

An if/then/else rendered as a box.

```
┌────────────────────────────────────┐
│  Bureau Required                    │
├─────┬──────────────────────────────┤
│ if  │ Risk Category = "High"       │
│ then│ true                         │
│ else│ false                        │
└─────┴──────────────────────────────┘
```

### 11.2.9 Boxed Filter

```
┌────────────────────────────────────┐
│  High Value Orders                  │
├────────────────────────────────────┤
│  Orders[Amount > 1000]              │
└────────────────────────────────────┘
```

### 11.2.10 Boxed Iterator (for, some, every)

```
┌────────────────────────────────────┐
│  Line Totals                        │
├────────────────────────────────────┤
│  for item in Order.Items            │
│  return item.Qty * item.Price       │
└────────────────────────────────────┘
```

---

## 11.3 Composing Boxed Expressions

The power of boxed expressions is composition. A context entry can contain a decision table. A decision table cell can contain an invocation. An invocation argument can be a literal expression.

### Worked Example 11.1 — Full Loan Affordability

```
┌──────────────────────────────────────────────────────────────┐
│  Pre-Bureau Affordability                                     │
├──────────────────────────────────────────────────────────────┤
│  Affordability Calculation                                    │
├────────────────────────┬─────────────────────────────────────┤
│  Monthly Income        │ Applicant.Income                    │
│  Monthly Repayments    │ Applicant.Repayments                │
│  Monthly Expenses      │ Applicant.Expenses                  │
│  Risk Category         │ Pre-Bureau Risk Category            │
│  Required Installment  │ Required Monthly Installment        │
└────────────────────────┴─────────────────────────────────────┘
```

This invokes the `Affordability Calculation` BKM, which is itself a boxed function containing a boxed context, which contains a boxed invocation of the `Credit Contingency Factor` decision table.

Three levels of nesting — each independently testable, each independently reviewable.

---

## Summary

| Boxed Expression | Purpose | FEEL Equivalent |
|-----------------|---------|----------------|
| Literal | Single expression | `expr` |
| Context | Step-by-step calculation | `{k: v, ...}` |
| List | Ordered collection | `[e1, e2, ...]` |
| Relation | Table of data | `[{...}, {...}]` |
| Function Definition | Reusable logic | `function(p) body` |
| Invocation | Call a BKM | `BKM(args)` |
| Decision Table | Rule set | (See Chapter 10) |
| Conditional | If/then/else | `if c then a else b` |
| Filter | Subset of list | `list[condition]` |
| Iterator | Loop construct | `for x in L return e` |

---

## What Comes Next

Part IV begins with Chapter 12: Patterns and Recipes — practical, copy-paste solutions for common business logic scenarios.

---

[Previous: Chapter 10: Decision Tables in Depth](chapter-10-decision-tables.md) | [Next: Chapter 12: Patterns and Recipes](../part-4-mastery/chapter-12-patterns-and-recipes.md)
