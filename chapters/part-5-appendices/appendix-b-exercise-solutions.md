# Appendix B: Exercise Solutions

---

## Chapter 4: Values and Types

**Exercise 4.1:**
- `1/3` → `0.3333333333333333333333333333333333` (34-digit precision)
- `decimal(1/3, 4)` → `0.3333`
- `"hello" + 42` → `null` (no implicit type coercion)
- `string length("hello" + " " + "world")` → `11`

**Exercise 4.2:**
```
years and months duration(birth date, today()).years
```
With `birth date = @"1995-12-25"`: evaluates to the number of whole years elapsed.

**Exercise 4.3:**
`true and null and false`:
- Step 1: `true and null` → `null` (see truth table)
- Step 2: `null and false` → `false` (see truth table)
- Result: `false`

**Exercise 4.4:**
```
{
  status: if Temperature = null then "Unknown"
          else if Temperature > 38.5 then "Fever"
          else "Normal",
  action: if status = "Fever" then "Administer treatment"
          else "Continue monitoring"
}
```

**Exercise 4.5:**
```
{
  Contract Start: @"2022-01-15",
  Term: @"P3Y",
  expiry: Contract Start + Term,
  days until expiry: expiry - today(),
  expired: today() > expiry
}
```

---

## Chapter 5: Expressions

**Exercise 5.1:**
- `-3 ** 2` → `9` (interpreted as `(-3) ** 2` because unary negation has higher precedence than exponentiation)
- `(-3) ** 2` → `9` (same result, explicit)
- `-(3 ** 2)` → `-9` (negation applied after exponentiation)

**Exercise 5.2:**
```
{
  bmi: weight / (height ** 2),
  category: if bmi < 18.5 then "Underweight"
            else if bmi < 25 then "Normal"
            else if bmi < 30 then "Overweight"
            else "Obese"
}
```

**Exercise 5.3:**
`"abc" > 5` → `null`. Cross-type comparison (string vs number) produces `null` in FEEL because the comparison is meaningless.

**Exercise 5.4:**
- (a) `37.0`
- (b) `(37.5..38.5)`
- (c) `>= 39`
- (d) `-`

---

## Chapter 6: Collections

**Exercise 6.1:**
- `L[0]` → `null` (no 0th element in 1-based indexing)
- `L[1]` → `10`
- `L[-1]` → `50`
- `L[6]` → `null` (out of bounds)
- `L[item > 25]` → `[30, 40, 50]`

**Exercise 6.2:**
```
{
  total: sum(Items.amount),
  count: count(Items),
  average: mean(Items.amount),
  max item: (sort(Items, function(a, b) a.amount > b.amount))[1].description
}
```

**Exercise 6.3:**
```
{
  vacation: [@"2024-07-01"..@"2024-07-14"],
  project: [@"2024-07-10"..@"2024-07-20"],
  do overlap: overlaps(vacation, project),
  overlap start: max([@"2024-07-01", @"2024-07-10"]),
  overlap end: min([@"2024-07-14", @"2024-07-20"]),
  overlap days: (overlap end - overlap start).days + 1
}
// do overlap = true, overlap days = 5
```

---

## Chapter 7: Iteration and Quantification

**Exercise 7.1:**
```
for i in 1..10 return i ** 2
```

**Exercise 7.2:**
```
for tx in Transactions return
  if count(partial) = 0 then tx.amount
  else partial[-1] + tx.amount
// [50, 30, 130, 120]
```
The `partial` variable holds the results of previous iterations (Camunda extension). At each step, the running balance is the previous balance plus the current transaction amount.

**Exercise 7.3:**
```
some emp in Employees satisfies emp.salary > 200000
```

**Exercise 7.4:**
```
every item in Order.Items satisfies item.quantity > 0 and item.sku != null
```

**Exercise 7.5:**
```
some meeting in Meetings satisfies overlaps(Proposed Slot, meeting)
```

---

## Chapter 8: Functions

**Exercise 8.1:**
```
function(value, low, high) max([low, min([high, value])])
```

**Exercise 8.2:**
```
sort(Employees, function(a, b) a.salary > b.salary)
```

**Exercise 8.3:**
```
function(d) {
  y: d.year,
  m: d.month,
  next first: if m = 12 then date(y + 1, 1, 1) else date(y, m + 1, 1),
  last day: next first - @"P1D"
}.last day
```

**Exercise 8.4:**
```
string join(for i in 1..5 return string(i), ", ")
```

---

## Chapter 9: Decision Tables

**Exercise 9.1:**

| U | Order Priority | Distance | Delivery Days |
|---|---------------|----------|---------------|
| 1 | "Overnight" | - | 1 |
| 2 | "Express" | <= 100 | 1 |
| 3 | "Express" | (100..500] | 2 |
| 4 | "Express" | > 500 | 3 |
| 5 | "Standard" | <= 100 | 3 |
| 6 | "Standard" | (100..500] | 5 |
| 7 | "Standard" | > 500 | 7 |

**Exercise 9.2:**

| C+ | Years of Experience | Partial Score |
|----|-------------------|---------------|
| 1 | < 2 | 10 |
| 2 | [2..5) | 25 |
| 3 | [5..10) | 40 |
| 4 | >= 10 | 50 |

| C+ | Highest Education | Partial Score |
|----|------------------|---------------|
| 5 | "PhD" | 30 |
| 6 | "Masters" | 25 |
| 7 | "Bachelors" | 20 |
| 8 | "Other" | 10 |

| C+ | Number of References | Partial Score |
|----|---------------------|---------------|
| 9 | >= 3 | 20 |
| 10 | [1..3) | 10 |
| 11 | 0 | 0 |

**Exercise 9.3:**
At runtime with hit policy U, if two rules match, the engine signals an error (implementation-dependent: some return `null`, some throw an exception). Fix by making the rules mutually exclusive — ensure input ranges do not overlap. For example, change `[0..10]` and `[10..20]` to `[0..10]` and `(10..20]`.
