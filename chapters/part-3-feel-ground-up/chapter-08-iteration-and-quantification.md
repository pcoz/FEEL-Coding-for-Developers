# Chapter 8: Iteration and Quantification

> *"FEEL does not have `while` loops or `for` statements. It has `for` expressions — they always produce a value, and they always terminate."*

---

There are no `while` loops in FEEL. No mutation, no loop counters, no off-by-one bugs. Instead, FEEL gives you `for` *expressions* that transform collections and always produce a value. If you have used `.map()` in JavaScript or list comprehensions in Python, you already have the right mental model.

---

## 8.1 The `for` Expression

### Basic Syntax

```
for x in collection return expression
```

Read it aloud: "for each `x` in the collection, return this expression." The engine iterates, evaluates, and collects every result into a new list.

```
for x in [1, 2, 3] return x * 2
// [2, 4, 6]

for name in ["Alice", "Bob"] return upper case(name)
// ["ALICE", "BOB"]

for item in Order.Items return item.Quantity * item.Unit Price
// list of line totals
```

### Rosetta Stone

| FEEL | JavaScript | Python |
|------|-----------|--------|
| `for x in L return f(x)` | `L.map(x => f(x))` | `[f(x) for x in L]` |

### Range Iteration

You do not need a list to iterate -- a numeric or date range works just as well:

```
for i in 1..5 return i ** 2
// [1, 4, 9, 16, 25]

for i in 5..1 return i
// [5, 4, 3, 2, 1]  (descending)

for d in @"2024-01-01"..@"2024-01-07" return d
// [2024-01-01, 2024-01-02, ..., 2024-01-07]  (7 dates, incrementing by 1 day)
```

### Multiple Iteration Contexts (Cartesian Product)

Add a second variable and you get a nested loop -- a Cartesian product of all combinations:

```
for x in [1, 2], y in [10, 20] return x + y
// [11, 21, 12, 22]
```

The rightmost variable varies fastest (like a nested `for` in any other language):

```
Iteration 1: x=1, y=10 → 11
Iteration 2: x=1, y=20 → 21
Iteration 3: x=2, y=10 → 12
Iteration 4: x=2, y=20 → 22
```

### The `partial` Variable

> **ENGINE NOTE:** The `partial` variable is a **Camunda feel-scala extension**, not part of the DMN specification. Apache KIE and other engines may not support it. If portability is a concern, use explicit accumulation patterns instead.

Inside a `for` expression, the implicit variable `partial` holds a list of all results computed *so far*. Think of it as a running accumulator that grows with each iteration -- it lets you write things like running totals and recursive sequences:

```
for i in 0..5 return if i = 0 then 1 else i * partial[-1]
// [1, 1, 2, 6, 24, 120]  (factorials: 0!, 1!, 2!, 3!, 4!, 5!)
```

Explanation:
- Iteration 0: `partial` = `[]`, result = `1`
- Iteration 1: `partial` = `[1]`, result = `1 * 1` = `1`
- Iteration 2: `partial` = `[1, 1]`, result = `2 * 1` = `2`
- Iteration 3: `partial` = `[1, 1, 2]`, result = `3 * 2` = `6`
- etc.

**Running total:**
```
for i in [100, 200, 50, 300] return
  if count(partial) = 0 then i
  else partial[-1] + i
// [100, 300, 350, 650]
```

### Worked Example 8.1 — Fibonacci Sequence

```
for i in 1..10 return
  if i <= 2 then 1
  else partial[-1] + partial[-2]
// [1, 1, 2, 3, 5, 8, 13, 21, 34, 55]
```

### Worked Example 8.2 — Line Item Processing

```
{
  items: [
    {sku: "A1", qty: 2, price: 25.00},
    {sku: "B3", qty: 1, price: 50.00},
    {sku: "C7", qty: 5, price: 10.00}
  ],
  line totals: for item in items return item.qty * item.price,
  subtotal: sum(line totals)
}
```

Result: `line totals` = `[50.00, 50.00, 50.00]`, `subtotal` = `150.00`

---

## 8.2 Quantified Expressions

Sometimes you do not need to transform a list -- you just need to ask a yes-or-no question about it. *Does any item exceed the budget? Are all documents signed?* That is what quantifiers are for.

### `some ... satisfies` (Existential Quantifier)

```
some x in [1, 2, 3, 4, 5] satisfies x > 4
// true (5 > 4)

some x in [1, 2, 3] satisfies x > 10
// false (none exceed 10)
```

### `every ... satisfies` (Universal Quantifier)

```
every x in [2, 4, 6, 8] satisfies even(x)
// true (all are even)

every x in [2, 4, 5, 8] satisfies even(x)
// false (5 is not even)
```

### Multiple Variables

Just like `for`, quantifiers support multiple iteration variables. The engine tests the condition against every combination:

```
some x in [1, 2], y in [3, 4] satisfies x + y = 5
// true (x=1, y=4 or x=2, y=3)

every x in [1, 2], y in [3, 4] satisfies x + y > 2
// true (all four combinations exceed 2)
```

### Rosetta Stone

| FEEL | JavaScript | Python | SQL |
|------|-----------|--------|-----|
| `some x in L satisfies p(x)` | `L.some(x => p(x))` | `any(p(x) for x in L)` | `EXISTS (SELECT ... WHERE p)` |
| `every x in L satisfies p(x)` | `L.every(x => p(x))` | `all(p(x) for x in L)` | `NOT EXISTS (SELECT ... WHERE NOT p)` |

### Edge Cases

Empty collections can trip you up if you are not expecting them:
```
some x in [] satisfies true      // false (no element exists to satisfy)
every x in [] satisfies false    // true  (vacuously true)
```

The `some` result is intuitive -- there is nothing in the list, so nothing can satisfy the condition. The `every` result surprises many developers: "every item in an empty list satisfies *any* condition" is `true` because there are zero counterexamples. This is standard logic (vacuous truth), and it matches the behavior of JavaScript's `[].every(...)` and Python's `all([])`.

### Worked Example 8.3 — Validation Checks

```
{
  has overdue invoice:
    some inv in Customer.Invoices
    satisfies inv.Due Date < today() and inv.Paid = false,

  all documents signed:
    every doc in Application.Documents
    satisfies doc.Signed = true,

  any item out of stock:
    some item in Order.Items
    satisfies Inventory[SKU = item.SKU].Available < item.Quantity,

  all amounts positive:
    every item in Order.Items
    satisfies item.Amount > 0
}
```

---

## 8.3 Combining Iteration, Filtering, and Aggregation

Individually, `for`, filters, and quantifiers are useful. Together, they let you write business logic that reads like a specification. The following worked examples show how these pieces snap together in realistic scenarios.

### Worked Example 8.4 — Order Summary with Tax

```
{
  items: Order.Items,
  line totals: for item in items return item.Quantity * item.Unit Price,
  subtotal: sum(line totals),
  taxable items: items[Taxable = true],
  taxable total: sum(for item in taxable items
                     return item.Quantity * item.Unit Price),
  tax: taxable total * Tax Rate,
  shipping: if subtotal > 100 then 0 else 9.99,
  grand total: subtotal + tax + shipping
}
```

### Worked Example 8.5 — Employee Bonus Calculation

```
{
  eligible: Employees[Performance Rating >= 4 and Years of Service >= 2],
  bonuses: for emp in eligible return {
    name: emp.Name,
    bonus: emp.Salary * (if emp.Performance Rating = 5 then 0.15 else 0.10)
  },
  total bonus budget: sum(bonuses.bonus),
  headcount: count(eligible)
}
```

### Worked Example 8.6 — Date Sequence Generation

Need all the Mondays in March 2024? Combine date range iteration with filtering:

```
{
  year: 2024,
  month: 3,
  start: date(year, month, 1),
  end: if month = 12 then date(year + 1, 1, 1) - @"P1D"
       else date(year, month + 1, 1) - @"P1D",
  all days: for d in start..end return d,
  mondays: all days[day of week(item) = "Monday"]
}
```

---

## Summary

| Construct | Syntax | Produces |
|-----------|--------|----------|
| `for` | `for x in L return expr` | A new list |
| Range `for` | `for i in 1..n return expr` | A list of n values |
| Multi-variable `for` | `for x in A, y in B return expr` | Cartesian product |
| `partial` | Implicit variable in `for` | Running accumulator |
| `some ... satisfies` | `some x in L satisfies cond` | `true` / `false` |
| `every ... satisfies` | `every x in L satisfies cond` | `true` / `false` |

---

## Exercises

**Exercise 8.1:** Write a `for` expression that produces the first 10 square numbers: `[1, 4, 9, 16, 25, 36, 49, 64, 81, 100]`.

**Exercise 8.2:** Given `Transactions = [{amount: 50}, {amount: -20}, {amount: 100}, {amount: -10}]`, write a `for` expression that produces a running balance starting from 0.

**Exercise 8.3:** Write a `some` expression that checks whether any employee in a list has a salary above 200000.

**Exercise 8.4:** Write an `every` expression that verifies all items in an order have a positive quantity and a non-null SKU.

**Exercise 8.5:** Given a list of date ranges representing scheduled meetings, write an expression that checks whether a proposed time slot conflicts with any existing meeting.

---

## What Comes Next

You can now transform, query, and validate any collection. Chapter 9 takes the next step: defining your own reusable functions and exploring FEEL's built-in function library.

---

[Previous: Chapter 7: Collections](chapter-07-collections.md) | [Next: Chapter 9: Functions](chapter-09-functions.md)
