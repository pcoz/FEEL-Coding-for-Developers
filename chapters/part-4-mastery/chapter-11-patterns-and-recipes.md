# Chapter 11: Patterns and Recipes

> *"You do not need to reinvent the wheel. The wheel is as follows..."*

---

This chapter provides ready-to-use patterns for common business logic tasks. Each pattern includes the problem statement, the FEEL solution, and an explanation of why it works.

---

## 11.1 Null-Guarding Patterns

### Pattern: Default Value

```
get or else(Customer.Email, "no-reply@example.com")
```

> **ENGINE NOTE:** `get or else()` is a Camunda feel-scala extension, not part of the DMN specification. For portable code, use the `if/then/else` equivalent below.

Or equivalently:
```
if Customer.Email != null then Customer.Email else "no-reply@example.com"
```

### Pattern: Coalesce (First Non-Null)

FEEL does not have a built-in `coalesce`. Build one:

```
{
  sources: [Primary Phone, Secondary Phone, Office Phone],
  contact: (sources[item != null])[1]
}
```

Or as a reusable function:
```
{
  coalesce: function(values) (values[item != null])[1],
  phone: coalesce([Primary Phone, Secondary Phone, "N/A"])
}
```

### Pattern: Null-Safe Property Chain

```
if Order != null then
  if Order.Customer != null then
    Order.Customer.Name
  else null
else null
```

Simplified (FEEL's null propagation does this automatically):
```
Order.Customer.Name
// Returns null if Order is null, or Order.Customer is null
```

### Pattern: Guard with assert

```
{
  validated age: assert(Age, Age >= 0 and Age <= 150, "Invalid age"),
  category: if validated age < 18 then "Minor" else "Adult"
}
```

If `Age` is negative, `assert` halts evaluation with the error message rather than silently producing wrong results.

---

## 11.2 String Processing Patterns

### Pattern: Case-Insensitive Comparison

```
lower case(Input) = lower case(Expected)
```

### Pattern: Extract Domain from Email

```
substring after(Email, "@")
// "alice@example.com" → "example.com"
```

### Pattern: Build a Comma-Separated List

```
string join(Items.Name, ", ")
// ["Apple", "Banana", "Cherry"] → "Apple, Banana, Cherry"
```

### Pattern: Pad a Number with Leading Zeros

```
{
  raw: string(Invoice Number),
  padded: substring("000000", 1, 6 - string length(raw)) + raw
}
// Invoice Number = 42 → "000042"
```

### Pattern: Simple Template

```
"Dear " + Customer.Name + ",\n\n" +
"Your order #" + string(Order.ID) + " totalling " +
string(Order.Total) + " has been " + Order.Status + ".\n\n" +
"Thank you."
```

---

## 11.3 Date and Time Patterns

### Pattern: Age in Years

```
years and months duration(Date of Birth, today()).years
```

### Pattern: Days Until Event

```
(Event Date - today()).days
```

### Pattern: Is Today a Weekday?

```
day of week(today()) in ("Monday", "Tuesday", "Wednesday", "Thursday", "Friday")
```

### Pattern: End of Month

```
{
  next month first: date(Year, Month + 1, 1),
  end of month: next month first - @"P1D"
}
```

Handle December:
```
{
  next month first: if Month = 12 then date(Year + 1, 1, 1) else date(Year, Month + 1, 1),
  end of month: next month first - @"P1D"
}
```

### Pattern: Business Days Approximation

Count weekdays between two dates (excluding weekends, not holidays):
```
{
  all days: for d in Start Date..End Date return d,
  weekdays: all days[day of week(item) in
    ("Monday", "Tuesday", "Wednesday", "Thursday", "Friday")],
  business day count: count(weekdays)
}
```

### Pattern: Add Business Days

```
{
  add biz days: function(start, n) {
    calendar days: for d in start..(start + duration("P" + string(n * 2) + "D")) return d,
    weekdays: calendar days[day of week(item) in
      ("Monday", "Tuesday", "Wednesday", "Thursday", "Friday")],
    result: weekdays[n]
  }.result,
  deadline: add biz days(today(), 5)
}
```

---

## 11.4 List Processing Patterns

### Pattern: Group By (Simulated)

FEEL has no built-in `group by`. Simulate it:

```
{
  departments: distinct values(Employees.Department),
  groups: for dept in departments return {
    department: dept,
    employees: Employees[Department = dept],
    count: count(Employees[Department = dept]),
    avg salary: mean(Employees[Department = dept].Salary)
  }
}
```

### Pattern: Top N

```
{
  sorted: sort(Scores, function(a, b) a > b),
  top 3: sublist(sorted, 1, 3)
}
```

### Pattern: Running Total

```
for i in 1..count(Transactions) return
  sum(sublist(Transactions.Amount, 1, i))
```

Or using `partial`:
```
for tx in Transactions return
  sum(append(partial, tx.Amount))
```

### Pattern: Zip Two Lists

```
for i in 1..count(Keys) return {
  key: Keys[i],
  value: Values[i]
}
```

### Pattern: Deduplicate by Key

```
{
  seen: distinct values(Records.ID),
  unique: for id in seen return
    (Records[ID = id])[1]
}
```

### Pattern: Partition into Chunks

```
{
  chunk size: 3,
  num chunks: ceiling(count(Items) / chunk size),
  chunks: for i in 1..num chunks return
    sublist(Items, (i - 1) * chunk size + 1,
            min([chunk size, count(Items) - (i - 1) * chunk size]))
}
```

---

## 11.5 Decision Table Patterns

### Pattern: Classification with Fallback

Use a default output value or a catch-all rule with `-`:

| U | Score | Category |
|---|-------|----------|
| 1 | >= 80 | "A" |
| 2 | [60..80) | "B" |
| 3 | [40..60) | "C" |
| 4 | < 40 | "F" |

If you want a default for `null` input, add: rule 5 with input `-` and output `"Unknown"`.

### Pattern: Lookup Table (Relation + Filter)

For large lookup tables, use a relation instead of a decision table:

```
{
  rates: [
    {country: "US", rate: 0.10},
    {country: "UK", rate: 0.20},
    {country: "DE", rate: 0.19},
    {country: "FR", rate: 0.20}
  ],
  applicable rate: (rates[country = Customer.Country])[1].rate
}
```

---

## 11.6 Recursive Patterns

### Pattern: Factorial

```
{
  factorial: function(n) if n <= 1 then 1 else n * factorial(n - 1),
  result: factorial(10)   // 3628800
}
```

### Pattern: Power (Integer Exponent)

```
{
  power: function(base, exp)
    if exp = 0 then 1
    else base * power(base, exp - 1),
  result: power(2, 10)   // 1024
}
```

### Pattern: Flatten Nested Structure

```
{
  tree: {value: 1, children: [
    {value: 2, children: []},
    {value: 3, children: [
      {value: 4, children: []}
    ]}
  ]},
  collect values: function(node)
    append(
      [node.value],
      flatten(for child in node.children return collect values(child))
    ),
  all values: collect values(tree)   // [1, 2, 3, 4]
}
```

---

## Summary

This chapter is a reference you will return to. When you encounter a business logic task, check here first — the pattern you need is likely already solved.

---

## What Comes Next

Chapter 12 covers FEEL in real systems — how to embed FEEL in Camunda, Apache KIE, and other engines, and how to test FEEL expressions in CI/CD pipelines.
