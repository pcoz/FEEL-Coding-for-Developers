# Chapter 9: Functions

> *"FEEL has no classes, no modules, no packages. It has functions. That is enough."*

---

## 9.1 User-Defined Functions

### Anonymous Functions (Lambdas)

```
function(x, y) x + y
```

That's it -- no `def`, no `fn`, no arrow syntax. Just the keyword `function`, the parameters, and the body. To actually call it, give it a name via a context entry:

```
{
  add: function(x, y) x + y,
  result: add(3, 4)
}
// result = 7
```

### Typed Parameters

Want guardrails? You can declare parameter types:

```
function(x: number, y: number) x + y
```

Pass the wrong type and the call returns `null` -- no exception, no crash, just `null`.

### Lexical Closure

FEEL functions capture their surrounding scope, just like closures in JavaScript or Python:

```
{
  tax rate: 0.20,
  apply tax: function(amount) amount * (1 + tax rate),
  result: apply tax(100)
}
// result = 120
```

The function "closes over" `tax rate` -- it remembers the value even if evaluated later in a different scope. If you've used closures in any other language, this works exactly the way you'd expect.

### Recursive Functions

> **ENGINE NOTE:** Whether a context entry name is in scope within its own definition body is engine-dependent. Camunda's feel-scala supports self-referencing context entries; other engines may not. The DMN specification supports recursion via BKMs (Business Knowledge Models) but the scoping of context entry self-references is ambiguous. Test in your target engine.

Yes, FEEL can do recursion. A named function in a context can call itself:

```
{
  factorial: function(n) if n <= 1 then 1 else n * factorial(n - 1),
  result: factorial(10)
}
// result = 3628800
```

### Worked Example 9.1 — A Reusable Discount Calculator

```
{
  calculate discount: function(tier, amount)
    if tier = "Gold" and amount >= 100 then 0.20
    else if tier = "Gold" then 0.10
    else if tier = "Silver" then 0.05
    else 0,

  order 1 discount: calculate discount("Gold", 150),
  order 2 discount: calculate discount("Silver", 50),
  order 3 discount: calculate discount("Bronze", 200)
}
// order 1 discount = 0.20, order 2 discount = 0.05, order 3 discount = 0
```

---

## 9.2 Positional vs Named Parameters

Every FEEL function -- built-in or user-defined -- supports two calling styles:

**Positional:** Arguments are matched by position.
```
substring("hello world", 7, 5)    // "world"
```

**Named:** Arguments are matched by name, in any order.
```
substring(string: "hello world", start position: 7, length: 5)    // "world"
```

Named parameters make your intent obvious at the call site, and they shine when a function has many parameters or optional ones. Any parameter you leave out defaults to `null`.

---

## 9.3 The Built-in Function Library

FEEL ships with roughly 70 built-in functions you'd otherwise have to write yourself. Below is the complete reference, organised by category -- bookmark this section.

### 9.3.1 Conversion Functions

| Function | Description | Example |
|----------|------------|---------|
| `number(from, grouping?, decimal?)` | Parse string to number | `number("1,000.50", ",", ".")` → `1000.50` |
| `string(from)` | Convert to string | `string(42)` → `"42"` |
| `date(from)` | Parse string to date | `date("2024-03-15")` → date |
| `date(year, month, day)` | Construct date | `date(2024, 3, 15)` → date |
| `date(from)` | Extract date from date-time | `date(@"2024-03-15T10:00:00")` → date |
| `time(from)` | Parse string to time | `time("14:30:00")` → time |
| `time(h, m, s, offset?)` | Construct time | `time(14, 30, 0)` → time |
| `date and time(date, time)` | Combine date and time | `date and time(@"2024-03-15", @"14:30:00")` |
| `date and time(from)` | Parse string | `date and time("2024-03-15T14:30:00Z")` |
| `duration(from)` | Parse duration string | `duration("P1Y6M")` → years and months |
| `years and months duration(from, to)` | Compute YM duration | `years and months duration(@"2020-01-01", @"2022-06-15")` → `P2Y5M` |
| `range(from)` | Parse range string | `range("[1..10)")` → range value |

### 9.3.2 Boolean Functions

| Function | Description | Example |
|----------|------------|---------|
| `not(value)` | Logical negation | `not(true)` → `false` |
| `is defined(value)` | True if not null | `is defined(42)` → `true` |
| `get or else(value, default)` | Return value if defined, else default | `get or else(null, 0)` → `0` |
| `assert(value, condition)` | Return value if condition true, else error | `assert(x, x > 0)` |
| `assert(value, condition, msg)` | With message | `assert(x, x > 0, "Must be positive")` |

### 9.3.3 String Functions

| Function | Description | Example |
|----------|------------|---------|
| `substring(s, start, length?)` | Extract substring (1-based) | `substring("foobar", 3)` → `"obar"` |
| `string length(s)` | Character count | `string length("hello")` → `5` |
| `upper case(s)` | Uppercase | `upper case("aBc")` → `"ABC"` |
| `lower case(s)` | Lowercase | `lower case("aBc")` → `"abc"` |
| `contains(s, match)` | Contains substring | `contains("foobar", "ob")` → `true` |
| `starts with(s, match)` | Starts with | `starts with("foobar", "foo")` → `true` |
| `ends with(s, match)` | Ends with | `ends with("foobar", "bar")` → `true` |
| `substring before(s, match)` | Text before match | `substring before("foobar", "bar")` → `"foo"` |
| `substring after(s, match)` | Text after match | `substring after("foobar", "ob")` → `"ar"` |
| `replace(input, pattern, repl)` | Regex replace | `replace("banana", "a", "o")` → `"bonono"` |
| `matches(input, pattern)` | Regex test | `matches("foobar", "^fo+b")` → `true` |
| `split(s, delimiter)` | Split by regex | `split("a,b,c", ",")` → `["a","b","c"]` |
| `string join(list, delim?)` | Join strings | `string join(["a","b"], "-")` → `"a-b"` |

> **ENGINE NOTE:** The `replace()` and `matches()` functions use XPath-compatible regular expressions, not JavaScript or PCRE regex. The most notable difference: use `\\s` for whitespace, and character class syntax follows XML Schema.

### 9.3.4 List Functions

| Function | Description | Example |
|----------|------------|---------|
| `count(list)` | Number of elements | `count([1,2,3])` → `3` |
| `min(list)` or `min(a,b,...)` | Minimum value | `min([3,1,2])` → `1` |
| `max(list)` or `max(a,b,...)` | Maximum value | `max([3,1,2])` → `3` |
| `sum(list)` or `sum(a,b,...)` | Sum | `sum([10,20,30])` → `60` |
| `mean(list)` | Arithmetic mean | `mean([10,20,30])` → `20` |
| `median(list)` | Median | `median([1,2,3,4,5])` → `3` |
| `stddev(list)` | Standard deviation | `stddev([2,4,7,5])` → `2.08...` |
| `mode(list)` | Most frequent value(s) | `mode([1,2,2,3])` → `[2]` |
| `product(list)` | Product | `product([2,3,4])` → `24` |
| `all(list)` | All true? | `all([true,true])` → `true` |
| `any(list)` | Any true? | `any([false,true])` → `true` |
| `append(list, item)` | Add to end | `append([1,2], 3)` → `[1,2,3]` |
| `concatenate(l1, l2, ...)` | Join lists | `concatenate([1],[2],[3])` → `[1,2,3]` |
| `insert before(list, pos, val)` | Insert at position | `insert before([1,3], 2, 2)` → `[1,2,3]` |
| `remove(list, pos)` | Remove at position | `remove([1,2,3], 2)` → `[1,3]` |
| `reverse(list)` | Reverse | `reverse([1,2,3])` → `[3,2,1]` |
| `index of(list, match)` | Find positions | `index of([1,2,3,2], 2)` → `[2,4]` |
| `union(l1, l2, ...)` | Distinct merge | `union([1,2],[2,3])` → `[1,2,3]` |
| `distinct values(list)` | Remove duplicates | `distinct values([1,1,2])` → `[1,2]` |
| `flatten(list)` | Deep flatten | `flatten([[1,[2]],3])` → `[1,2,3]` |
| `sort(list, precedes)` | Sort with comparator | `sort([3,1,2], function(a,b) a < b)` → `[1,2,3]` |
| `list contains(list, val)` | Contains element | `list contains([1,2,3], 2)` → `true` |
| `list replace(list, pos, val)` | Replace at position | `list replace([1,2,3], 2, 9)` → `[1,9,3]` |
| `list replace(list, match, val)` | Replace by condition | `list replace([1,2,3], function(x) x=2, 9)` → `[1,9,3]` |
| `sublist(list, start, length?)` | Extract sublist | `sublist([1,2,3,4], 2, 2)` → `[2,3]` |

### 9.3.5 Context Functions

| Function | Description | Example |
|----------|------------|---------|
| `get value(ctx, key)` | Access by string key | `get value({a: 1}, "a")` → `1` |
| `get entries(ctx)` | List of {key, value} | `get entries({a: 1})` → `[{key:"a", value:1}]` |
| `context(entries)` | Build from entries | `context([{key:"a", value:1}])` → `{a:1}` |
| `context put(ctx, key, val)` | Add/update entry | `context put({a:1}, "b", 2)` → `{a:1, b:2}` |
| `context merge(list)` | Merge (last wins) | `context merge([{a:1},{a:2}])` → `{a:2}` |

### 9.3.6 Temporal Functions

| Function | Description | Example |
|----------|------------|---------|
| `now()` | Current date-time | Returns current timestamp |
| `today()` | Current date | Returns current date |
| `day of week(date)` | Day name | `day of week(@"2024-03-15")` → `"Friday"` |
| `day of year(date)` | Day number (1-366) | `day of year(@"2024-03-15")` → `75` |
| `week of year(date)` | ISO week number | `week of year(@"2024-03-15")` → `11` |
| `month of year(date)` | Month name | `month of year(@"2024-03-15")` → `"March"` |

### 9.3.7 Numeric Functions

| Function | Description | Example |
|----------|------------|---------|
| `decimal(n, scale)` | Round half-even | `decimal(1.235, 2)` → `1.24` |
| `round up(n, scale)` | Round away from 0 | `round up(1.231, 2)` → `1.24` |
| `round down(n, scale)` | Round toward 0 | `round down(1.239, 2)` → `1.23` |
| `round half up(n, scale)` | Round half away from 0 | `round half up(1.235, 2)` → `1.24` |
| `round half down(n, scale)` | Round half toward 0 | `round half down(1.235, 2)` → `1.23` |
| `floor(n)` | Largest integer <= n | `floor(1.7)` → `1` |
| `ceiling(n)` | Smallest integer >= n | `ceiling(1.1)` → `2` |
| `abs(n)` | Absolute value | `abs(-5)` → `5` |
| `modulo(dividend, divisor)` | Remainder | `modulo(10, 3)` → `1` |
| `sqrt(n)` | Square root | `sqrt(16)` → `4` |
| `log(n)` | Natural logarithm | `log(10)` → `2.302...` |
| `exp(n)` | e^n | `exp(1)` → `2.718...` |
| `odd(n)` | Is odd? | `odd(3)` → `true` |
| `even(n)` | Is even? | `even(4)` → `true` |

### Worked Example 9.2 — Statistical Analysis

```
{
  scores: [72, 85, 91, 68, 79, 95, 88, 76, 82, 90],
  count: count(scores),
  average: decimal(mean(scores), 1),
  median: median(scores),
  std dev: decimal(stddev(scores), 2),
  highest: max(scores),
  lowest: min(scores),
  pass count: count(scores[item >= 75]),
  pass rate: decimal(pass count / count * 100, 1),
  ranking: sort(scores, function(a, b) a > b)
}
```

---

## 9.4 External Functions

When FEEL's built-in library isn't enough, you can reach outside the sandbox. FEEL supports calling external functions written in Java, predictive models in PMML, and machine learning models in ONNX:

### Java

```
{
  cos: function(angle: number) external {
    java: {
      class: "java.lang.Math",
      method signature: "cos(double)"
    }
  },
  result: cos(0)
}
// result = 1.0
```

### PMML (Predictive Models)

```
{
  predict risk: function(age: number, income: number) external {
    pmml: {
      document: "credit_risk_model.pmml",
      model: "CreditRiskScorecard"
    }
  },
  risk score: predict risk(35, 75000)
}
```

### ONNX (ML Models, DMN 1.5+)

```
{
  classify: function(features: list<number>) external {
    onnx: {
      file: "classifier.onnx",
      function signature: "FLOAT[1,4]"
    }
  },
  prediction: classify([5.1, 3.5, 1.4, 0.2])
}
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| User-defined functions | `function(params) body` — lexical closures |
| Recursive functions | Name via context entry, call by name |
| Named parameters | `f(param name: value)` — order-independent |
| Built-in library | ~70 functions across 7 categories |
| External functions | Java, PMML, ONNX interop |

---

## Exercises

**Exercise 9.1:** Write a function `clamp(value, low, high)` that returns `low` if value < low, `high` if value > high, and `value` otherwise.

**Exercise 9.2:** Using `sort()`, sort a list of employee contexts by salary descending.

**Exercise 9.3:** Write a function that takes a date and returns the last day of that month.

**Exercise 9.4:** Using `string join()` and `for`, produce the string `"1, 2, 3, 4, 5"` from the range `1..5`.

---

## What Comes Next

Functions are FEEL's way of packaging reusable logic. But for rule-heavy logic with many conditions, you want something more structured. Chapter 10 dives into decision tables -- hit policies, multi-output tables, completeness checking, and the patterns that make them powerful.

---

[Previous: Chapter 8: Iteration and Quantification](chapter-08-iteration-and-quantification.md) | [Next: Chapter 10: Decision Tables in Depth](chapter-10-decision-tables.md)
