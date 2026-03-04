# Chapter 6: Collections — Lists, Contexts, and Ranges

> *"Lists hold things. Contexts name things. Ranges bound things. Together they model any business data structure."*

---

## 6.1 Lists

### Creating Lists

```
[1, 2, 3]                         // list of numbers
["Gold", "Silver", "Bronze"]      // list of strings
[true, false, null]               // list of booleans (with null)
[[1, 2], [3, 4]]                  // nested list
[]                                 // empty list
[1, "two", true]                  // heterogeneous (type: list<Any>)
```

Lists are **ordered**, **immutable**, and **1-based**.

> **Singleton coercion:** FEEL automatically wraps a single value into a one-element list when a list is expected: `42[item > 40]` returns `[42]`. Conversely, a one-element list may be unwrapped to a scalar when a scalar is expected. This implicit conversion is convenient but can be surprising — see Chapter 14 for the full rules.

### Accessing Elements

```
[10, 20, 30][1]       // 10   (first element)
[10, 20, 30][2]       // 20   (second element)
[10, 20, 30][-1]      // 30   (last element)
[10, 20, 30][-2]      // 20   (second from last)
[10, 20, 30][4]       // null (out of bounds — no error!)
[10, 20, 30][0]       // null (there is no 0th element)
```

> **GOTCHA — 1-based indexing:** If you are coming from JavaScript, Python, Java, C#, or virtually any other language, your muscle memory says `list[0]` is the first element. In FEEL, `list[0]` is `null`. The first element is `list[1]`. This is the single most common FEEL mistake among developers.

### Rosetta Stone

| Operation | FEEL | JavaScript | Python |
|-----------|------|-----------|--------|
| First element | `L[1]` | `L[0]` | `L[0]` |
| Last element | `L[-1]` | `L.at(-1)` | `L[-1]` |
| Length | `count(L)` | `L.length` | `len(L)` |
| Append | `append(L, x)` | `[...L, x]` | `L + [x]` |
| Prepend | `insert before(L, 1, x)` | `[x, ...L]` | `[x] + L` |
| Contains | `list contains(L, x)` | `L.includes(x)` | `x in L` |
| Concatenate | `concatenate(L1, L2)` | `[...L1, ...L2]` | `L1 + L2` |
| Distinct | `distinct values(L)` | `[...new Set(L)]` | `list(set(L))` |
| Flatten | `flatten(L)` | `L.flat(Infinity)` | (itertools) |
| Reverse | `reverse(L)` | `[...L].reverse()` | `L[::-1]` |
| Remove | `remove(L, i)` | `L.toSpliced(i-1, 1)` | `L[:i-1]+L[i:]` |
| Index of | `index of(L, x)` | `L.indexOf(x)+1` | `L.index(x)+1` |
| Sort | `sort(L, function(a,b) a < b)` | `L.sort((a,b) => a-b)` | `sorted(L)` |

### Filtering Lists

FEEL's list filter is one of its most powerful features. Place a boolean expression inside square brackets:

```
[1, 2, 3, 4, 5][item > 3]                    // [4, 5]
[1, 2, 3, 4, 5][item >= 2 and item <= 4]     // [2, 3, 4]
[1, 2, 3, 4, 5][even(item)]                  // [2, 4]
```

The keyword `item` refers to the current element being tested.

**When the list contains contexts**, the context keys are directly accessible (no `item.` prefix needed):

```
[
  {name: "Alice", dept: "Engineering", salary: 90000},
  {name: "Bob", dept: "Sales", salary: 60000},
  {name: "Carol", dept: "Engineering", salary: 95000}
][dept = "Engineering"]
// [{name: "Alice", dept: "Engineering", salary: 90000},
//  {name: "Carol", dept: "Engineering", salary: 95000}]
```

**Selection with `.`** — extracting a single field from a list of contexts:

```
Employees.name
// ["Alice", "Bob", "Carol"]

Employees[dept = "Engineering"].salary
// [90000, 95000]
```

### Worked Example 6.1 — Filtering and Selecting

Given input data:

```
Employees: [
  {name: "Alice", dept: "Engineering", salary: 90000, years: 5},
  {name: "Bob", dept: "Sales", salary: 60000, years: 2},
  {name: "Carol", dept: "Engineering", salary: 95000, years: 8},
  {name: "Dave", dept: "Sales", salary: 70000, years: 12},
  {name: "Eve", dept: "Engineering", salary: 85000, years: 3}
]
```

**Find all Engineering employees earning above 88000:**
```
Employees[dept = "Engineering" and salary > 88000]
// [{name: "Alice", ...}, {name: "Carol", ...}]
```

**Get their names:**
```
Employees[dept = "Engineering" and salary > 88000].name
// ["Alice", "Carol"]
```

**Get the highest salary in Engineering:**
```
max(Employees[dept = "Engineering"].salary)
// 95000
```

**Count employees with 5+ years:**
```
count(Employees[years >= 5])
// 3
```

---

## 6.2 Contexts

A context is an ordered collection of key-value pairs. It is FEEL's equivalent of a JavaScript object, a Python dictionary, or a JSON object — but with one critical difference: **entries are evaluated sequentially and can reference earlier entries.**

### Creating Contexts

```
{
  name: "Alice",
  age: 30,
  active: true
}
```

Keys can be names (unquoted) or strings (quoted):
```
{
  "first name": "Alice",     // string key (allows spaces)
  department: "Engineering"   // name key
}
```

### Accessing Entries

```
person.name                     // "Alice"
person."first name"             // "Alice" (for string keys)
get value(person, "name")       // "Alice" (dynamic access)
```

### Sequential Evaluation — The Power Feature

Unlike JSON objects, FEEL context entries are evaluated **in order**, and each entry can reference entries that appear before it:

```
{
  gross: Salary * 12,
  tax: gross * Tax Rate,
  net: gross - tax,
  monthly net: net / 12
}
```

This is like a column in a spreadsheet, or a chain of `let` bindings in a functional language. Each step is named, readable, and debuggable.

> **KEY INSIGHT:** Use contexts to break complex calculations into named intermediate steps. Never write a single massive expression when a context can make each step explicit.

### Nested Contexts

Contexts can contain other contexts:

```
{
  customer: {
    name: "Alice Chen",
    tier: "Gold"
  },
  order: {
    total: 250,
    items: 3
  }
}
```

Access nested values with chained dots: `data.customer.tier` → `"Gold"`.

### Context Functions

| Function | Description | Example |
|----------|------------|---------|
| `get value(ctx, key)` | Get a value by string key | `get value(m, "name")` |
| `get entries(ctx)` | Get list of `{key, value}` pairs | `get entries({a: 1})` → `[{key: "a", value: 1}]` |
| `context(entries)` | Create context from entries list | `context([{key: "a", value: 1}])` → `{a: 1}` |
| `context put(ctx, key, value)` | Add/update an entry | `context put({a: 1}, "b", 2)` → `{a: 1, b: 2}` |
| `context merge(list)` | Merge contexts (last wins) | `context merge([{a: 1}, {a: 2, b: 3}])` → `{a: 2, b: 3}` |

### Worked Example 6.2 — Step-by-Step Affordability Calculation

From the DMN spec's loan origination example:

```
{
  Monthly Income: Applicant.Income,
  Monthly Expenses: Applicant.Expenses,
  Monthly Repayments: Applicant.Repayments,
  Required Installment: Required Monthly Installment,
  Disposable Income: Monthly Income - Monthly Expenses - Monthly Repayments,
  Credit Contingency Factor: if Risk Category = "High" then 0.6
                             else if Risk Category = "Medium" then 0.7
                             else 0.8,
  Affordable Amount: Disposable Income * Credit Contingency Factor,
  Affordable: Affordable Amount >= Required Installment
}
```

Each step is named. A business analyst can read each line and verify it against the policy. A developer can inspect any intermediate value during debugging.

---

## 6.3 Ranges

Ranges (intervals) are first-class values in FEEL. They represent a bounded or unbounded set of values.

### Creating Ranges

```
[1..10]           // 1 to 10, inclusive both ends
(1..10)           // 1 to 10, exclusive both ends
[1..10)           // 1 inclusive, 10 exclusive
(1..10]           // 1 exclusive, 10 inclusive

>= 5              // 5 or greater (no upper bound)
< 100             // less than 100 (no lower bound)
```

### Range Properties

```
[1..10].start           // 1
[1..10].end             // 10
[1..10].start included  // true
(1..10].start included  // false
[1..10].end included    // true
[1..10).end included    // false
```

### Ranges Work with Many Types

```
@"2024-01-01" in [@"2023-01-01"..@"2024-12-31"]    // true (date range)
"banana" in ["apple".."cherry"]                       // true (string range, lexicographic)
@"P1D" in [@"P0D"..@"P7D"]                           // true (duration range)
```

### Range Functions (Allen's Interval Relations)

FEEL provides a complete set of interval relation functions:

| Function | Tests whether... |
|----------|-----------------|
| `before(a, b)` | a is entirely before b |
| `after(a, b)` | a is entirely after b |
| `meets(a, b)` | a ends exactly where b starts |
| `met by(a, b)` | a starts exactly where b ends |
| `overlaps(a, b)` | a and b overlap |
| `includes(a, b)` | a fully contains b |
| `during(a, b)` | a is fully within b |
| `starts(a, b)` | a starts at the same point as b |
| `finishes(a, b)` | a ends at the same point as b |
| `coincides(a, b)` | a and b are identical |

### Worked Example 6.3 — Age Band Classification

```
if Age in [0..12] then "Child"
else if Age in [13..17] then "Teenager"
else if Age in [18..64] then "Adult"
else if Age in [65..120] then "Senior"
else null
```

### Worked Example 6.4 — Schedule Overlap Detection

```
{
  meeting: [@"09:00:00"..@"10:30:00"],
  lunch: [@"12:00:00"..@"13:00:00"],
  proposed: [@"10:00:00"..@"11:00:00"],
  conflicts with meeting: overlaps(proposed, meeting),
  conflicts with lunch: overlaps(proposed, lunch)
}
```

Result: `conflicts with meeting` = `true` (10:00–10:30 overlaps), `conflicts with lunch` = `false`.

---

## Summary

| Structure | Syntax | Key Feature |
|-----------|--------|------------|
| **List** | `[1, 2, 3]` | 1-based, immutable, filterable with `[condition]` |
| **Context** | `{key: value, ...}` | Sequential evaluation, entries reference earlier entries |
| **Range** | `[1..10]`, `(a..b]` | First-class intervals, work with numbers/dates/strings |

---

## Exercises

**Exercise 6.1:** Given `L = [10, 20, 30, 40, 50]`, what is the result of: `L[0]`, `L[1]`, `L[-1]`, `L[6]`, `L[item > 25]`?

**Exercise 6.2:** Write a FEEL context that takes `Items` (a list of `{description, amount}` contexts) and computes: `total` (sum of all amounts), `count` (number of items), `average` (mean amount), `max item` (description of the highest-amount item).

**Exercise 6.3:** Given two date ranges, `vacation = [@"2024-07-01"..@"2024-07-14"]` and `project = [@"2024-07-10"..@"2024-07-20"]`, write FEEL expressions to determine if they overlap and for how long.

---

## What Comes Next

Chapter 7 introduces iteration (`for ... return`) and quantification (`some ... satisfies`, `every ... satisfies`) — FEEL's tools for looping over collections.

---

[Previous: Chapter 5: Expressions](chapter-05-expressions.md) | [Next: Chapter 7: Iteration and Quantification](chapter-07-iteration-and-quantification.md)
