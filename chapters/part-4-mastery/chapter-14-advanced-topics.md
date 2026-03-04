# Chapter 14: Advanced Topics

> *"You can use FEEL productively without reading this chapter. But when something surprising happens, this is where you will find the explanation."*

---

## 14.1 The Type Lattice

FEEL types are organised as a **lattice** — a partially ordered structure with:

- **Any** at the top: every value conforms to `Any`.
- **Null** at the bottom: `null` conforms to every type.
- Primitive types in the middle: `number`, `string`, `boolean`, `date`, `time`, `date and time`, `days and time duration`, `years and months duration`.
- Parameterised types: `list<T>`, `range<T>`, `context<k1:T1, ...>`, `function<T1,...> -> R`.

```
                        Any
         ┌───────┬───────┼───────┬──────────────┐
       number  string boolean date  ...   list<Any>
         │       │      │      │              │
         │       │      │      │          list<number>
         │       │      │      │              │
         └───────┴──────┴──────┴──────────────┘
                        Null
```

### Type Equivalence (T ≡ S)

Two types are equivalent if they are interchangeable in all contexts:

- Primitive types: `number ≡ number` (same name = equivalent).
- Lists: `list<T> ≡ list<S>` iff `T ≡ S`.
- Contexts: `context<k1:T1, ..., kn:Tn> ≡ context<l1:S1, ..., lm:Sm>` iff same keys with equivalent types (order does not matter).
- Functions: `(T1,...,Tn) → U ≡ (S1,...,Sm) → V` iff same arity, equivalent parameter types, equivalent return type.
- Ranges: `range<T> ≡ range<S>` iff `T ≡ S`.

### Type Conformance (T <: S)

Conformance means "T can be used where S is expected":

- Every type conforms to `Any`.
- `Null` conforms to every type.
- `list<T>` conforms to `list<S>` iff `T <: S`.
- A context conforms to another if it has **at least** the same keys with conforming types (extra keys are OK — structural subtyping).
- Functions are **contravariant** in parameters and **covariant** in return type: `(S1,...) → U <: (T1,...) → V` iff `Ti <: Si` and `U <: V`.

### Practical Impact

Type conformance determines:
- Whether a function argument is accepted.
- Whether a decision output matches its declared type.
- Whether implicit conversions apply.

---

## 14.2 Implicit Type Conversions

FEEL performs four kinds of implicit conversion:

### To Singleton List

When a value of type `T` is expected to be `list<T>`, it is automatically wrapped:

```
3[item > 2]    // 3 is converted to [3], filter applies → [3]
```

### From Singleton List

When a `list<T>` with exactly one element is expected to be `T`, it is unwrapped:

```
contains(["foobar"], "of")
// ["foobar"] is unwrapped to "foobar", then contains("foobar", "of") → false
```

### Date to Date-Time

When a `date` is expected to be a `date and time`, it is converted with time of day = UTC midnight:

```
// If a function expects date and time and receives a date:
@"2024-03-15" → @"2024-03-15T00:00:00"
```

### Decimal to Integer

When a function expects an integer (e.g., `substring` position), a decimal number's fractional part is silently dropped:

```
substring("hello", 2.7)    // position treated as 2
```

### Conforms-To Conversion

If the result type does not conform to the target type and no implicit conversion applies, the result is `null`.

---

## 14.3 Scope and the Context Stack

Every FEEL expression is evaluated in a **scope** — an ordered list of contexts that determines how names (qualified names) are resolved.

### The Context Stack (from first to last)

1. **Local context**: If the expression is inside a context entry, the containing context is first. Earlier entries are in scope.
2. **Enclosing contexts**: If the context is nested inside another context, the outer context is next.
3. **Global context**: The context created by the DMN runtime containing input data, decision results, and BKM definitions.
4. **Built-in context**: Contains all built-in functions (`sum`, `count`, `date`, etc.).

### Special Contexts

Some expressions push a special context onto the front of the stack:

- **Filter expressions**: Push `{item: current_element}` (and all context entries of the element if it is a context).
- **For expressions**: Push `{variable: current_element, partial: accumulated_results}`.
- **Quantified expressions**: Push `{variable: current_element}`.

### Name Resolution

Names are resolved by searching the context stack from first to last. The **longest match** wins:

```
{
  a: 1,
  b: 2,
  a - b: 10,
  result: a - b    // Is this name "a - b" (= 10) or expression "a minus b" (= -1)?
}
```

The answer depends on the scope: since `a - b` is a key in the same context, the name match wins: `result` = `10`.

---

## 14.4 Name Ambiguity

FEEL allows names to contain characters that are also operators: `-`, `+`, `*`, `/`, `.`, `'`.

This creates ambiguity:

```
Profit and loss       // Is this the name "Profit and loss"?
                      // Or is it "Profit" AND "loss"?

a-b                   // Name "a-b" or expression "a minus b"?

what if?              // Name "what if?" or... something else?
```

### Resolution Rule

Names are matched **left-to-right** against in-scope names, and the **longest match** is preferred.

If `a-b` is an in-scope name, it is treated as a name. If not, it is parsed as `a minus b`.

### Disambiguation

Use parentheses to force expression interpretation:

```
(a) - (b)      // always subtraction, even if "a-b" is in scope
```

### Practical Advice

1. Avoid using operator characters in names when possible.
2. If you must (because the business domain uses them), be aware of the resolution rules.
3. When in doubt, use parentheses.
4. Prefer underscores or spaces: `risk_score` or `risk score` rather than `risk-score`.

---

## 14.5 XML Data in FEEL

FEEL can consume XML data by mapping it to the FEEL semantic domain.

### Mapping Rules (XE Function)

| XML | FEEL |
|-----|------|
| `<e/>` | `"e": null` |
| `<e>v</e>` | `"e": value(v)` |
| `<e>v1</e><e>v2</e>` | `"e": [value(v1), value(v2)]` (repeated element → list) |
| `<e a="v"/>` | `"e": {"a": value(v)}` |
| `<e a="v1">v2</e>` | `"e": {"@a": value(v1), "$content": value(v2)}` |

### Value Mapping (XV Function)

| XML Type | FEEL Value |
|----------|-----------|
| number | FEEL number |
| string | FEEL string |
| date | `date("value")` |
| dateTime | `date and time("value")` |
| time | `time("value")` |
| duration | `duration("value")` |
| element | Recursively mapped via XE |

### Example

XML:
```xml
<Context>
  <Employee>
    <salary>13000</salary>
  </Employee>
  <Customer>
    <loyalty_level>gold</loyalty_level>
    <credit_limit>10000</credit_limit>
  </Customer>
  <Customer>
    <loyalty_level>silver</loyalty_level>
    <credit_limit>5000</credit_limit>
  </Customer>
</Context>
```

Equivalent FEEL context:
```
{
  Employee: { salary: 13000 },
  Customer: [
    { loyalty_level: "gold", credit_limit: 10000 },
    { loyalty_level: "silver", credit_limit: 5000 }
  ]
}
```

Note: repeated `<Customer>` elements become a list. Single `<Employee>` becomes a context.

---

## 14.6 JSON and FEEL

FEEL also maps to/from JSON:

| FEEL Type | JSON Type |
|-----------|-----------|
| number | number |
| string | string |
| boolean | `true` / `false` |
| date, time, date and time | string (ISO 8601) |
| duration | string (ISO 8601) |
| list | array |
| context | object |
| range | string (grammar rule 66) |
| null | `null` |

When a FEEL `date and time` includes an IANA timezone, the JSON string is suffixed with the timezone in brackets:
```
"2024-03-15T10:00:00+01:00[Europe/Paris]"
```

---

## 14.7 The Full FEEL Grammar

The FEEL grammar consists of 68 rules in ISO EBNF notation. Key rules:

```
1. expression = boxed expression | textual expression ;
2. textual expression = for | if | quantified | disjunction | conjunction |
                        comparison | arithmetic | instance of |
                        path | filter | invocation | literal | name | "(" expression ")" ;
...
44. for expression = "for", name, "in", iteration context,
                     {"," , name, "in", iteration context}, "return", expression ;
45. if expression = "if", expression, "then", expression, "else", expression ;
46. quantified expression = ("some"|"every"), name, "in", expression,
                            {",", name, "in", expression}, "satisfies", expression ;
...
53. boxed expression = list | function definition | context ;
54. list = "[", [expression, {",", expression}], "]" ;
55. function definition = "function", "(", [formal parameter, {",", formal parameter}], ")",
                          ["external"], expression ;
57. context = "{", [context entry, {",", context entry}], "}" ;
58. context entry = key, ":", expression ;
```

Comments are Java-style: `// to end of line` and `/* ... */`.

---

## Summary

| Topic | Key Takeaway |
|-------|-------------|
| Type lattice | `Any` at top, `Null` at bottom, structural subtyping for contexts |
| Implicit conversions | Singleton list wrapping/unwrapping, date→datetime, decimal→integer |
| Scope | Context stack: local → enclosing → global → built-in |
| Name ambiguity | Longest match wins; disambiguate with parentheses |
| XML data | Mapped via XE/XV functions; repeated elements become lists |
| JSON data | Natural mapping; dates/durations as ISO 8601 strings |

---

## What Comes Next

Chapter 15 covers testing FEEL — how to derive comprehensive test cases from decision tables and expressions, coverage metrics, debugging techniques, and CI/CD integration. After that, the appendices provide reference material: a quick reference card, exercise solutions, a cross-language Rosetta Stone, an engine compatibility matrix, a glossary, and a guide to using FEEL with machine learning.
