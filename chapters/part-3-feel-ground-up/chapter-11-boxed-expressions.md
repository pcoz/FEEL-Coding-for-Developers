# Chapter 11: Boxed Expressions — Composing Logic Visually

> *"A boxed expression is a FEEL expression you can draw on a whiteboard."*

---

Every FEEL expression you've written so far lives in text. But when a business analyst opens the DMN model, they see boxes -- structured visual frames that make logic reviewable without reading code. This chapter catalogs every boxed expression type in the spec and shows how they compose into the rich visual models that make DMN worth adopting.

## 11.1 What Are Boxed Expressions?

Every FEEL expression you've written so far has a visual counterpart in DMN called a **boxed expression** -- a graphical frame that wraps your logic so that both developers and business stakeholders can read it.

Each box does three things:

1. Names the expression (the header at the top).
2. Shows its structure (the body).
3. Makes the logic reviewable without reading code.

Boxed expressions nest: a box can contain other boxes. The top-level box is always named after the DRG element (decision, BKM) it belongs to.

---

## 11.2 The Complete Catalog

### 11.2.1 Boxed Literal Expression

The simplest box: just a named FEEL expression. One box, one formula, done.

```
┌────────────────────────────────┐
│  Monthly Payment               │
├────────────────────────────────┤
│  Amount * Rate / 12            │
└────────────────────────────────┘
```

FEEL equivalent: `Amount * Rate / 12`

### 11.2.2 Boxed Context

This is the workhorse. A boxed context is a sequence of named entries, evaluated top to bottom -- each entry can reference the ones above it.

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

If the final entry has no key, its value becomes the result of the entire context -- a handy way to compute intermediate values and then return just the answer.

### 11.2.3 Boxed List

Exactly what it sounds like: a visual list of expressions, each in its own row.

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

A compact table of data rows that all share the same columns. Think spreadsheet, not decision table -- there's no hit policy and no condition logic. It's pure data.

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

The visual version of `function(params) body`. You see the parameters in the header and the implementation in the body -- perfect for BKMs (Business Knowledge Models) that get reused across decisions.

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

A structured way to call a BKM with named arguments. The header names the function being called; the rows map parameter names to values. No positional guessing.

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

This calls the `Installment Calculation` BKM with three named arguments. You can read the binding table top-to-bottom and immediately see what data feeds the function.

### 11.2.7 Decision Table

Already covered in depth in Chapter 10. The most common boxed expression type.

### 11.2.8 Boxed Conditional

Your familiar `if/then/else`, but laid out so each branch gets its own labeled row.

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

Here's where boxed expressions earn their keep: they compose. You can nest any box inside any other box. A context entry can contain a decision table. A decision table cell can contain an invocation. An invocation argument can be a literal expression. It's boxes all the way down.

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

Three levels of nesting -- and that's the whole point. Each level is independently testable, independently reviewable, and independently reusable. A business analyst can inspect the affordability context without understanding the credit model inside it.

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

You now know every boxed expression type in the DMN spec. Part IV puts it all together: Chapter 12 is Patterns and Recipes -- practical, copy-paste solutions for the business logic problems you'll actually encounter.

---

[Previous: Chapter 10: Decision Tables in Depth](chapter-10-decision-tables.md) | [Next: Chapter 12: Patterns and Recipes](../part-4-mastery/chapter-12-patterns-and-recipes.md)
