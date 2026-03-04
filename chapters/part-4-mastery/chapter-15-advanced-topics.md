# Chapter 15: Advanced Topics

> *"You can use FEEL productively without reading this chapter. But when something surprising happens, this is where you will find the explanation."*

---

You can use FEEL productively without reading this chapter. But when something surprising happens — a type coercion you didn't expect, a name resolution that makes no sense, a null that appeared out of nowhere — this is where you'll find the explanation. Type lattice, implicit conversions, scope rules, and the edge cases that trip up even experienced users.

## 15.1 The Type Lattice

FEEL's types form a **lattice** -- a hierarchy where every type has a well-defined place:

- **Any** sits at the top: every value conforms to `Any`.
- **Null** sits at the bottom: `null` conforms to every type. (This is why you can pass `null` anywhere without a type error.)
- Primitive types live in the middle: `number`, `string`, `boolean`, `date`, `time`, `date and time`, `days and time duration`, `years and months duration`.
- Parameterised types add structure: `list<T>`, `range<T>`, `context<k1:T1, ...>`, `function<T1,...> -> R`.

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

Two types are equivalent when you can swap one for the other and nothing breaks:

- Primitive types: `number ≡ number` (same name = equivalent).
- Lists: `list<T> ≡ list<S>` iff `T ≡ S`.
- Contexts: `context<k1:T1, ..., kn:Tn> ≡ context<l1:S1, ..., lm:Sm>` iff same keys with equivalent types (order does not matter).
- Functions: `(T1,...,Tn) → U ≡ (S1,...,Sm) → V` iff same arity, equivalent parameter types, equivalent return type.
- Ranges: `range<T> ≡ range<S>` iff `T ≡ S`.

### Type Conformance (T <: S)

Conformance answers a practical question: "Can I pass a value of type T where the engine expects type S?"

- Every type conforms to `Any`.
- `Null` conforms to every type.
- `list<T>` conforms to `list<S>` iff `T <: S`.
- A context conforms to another if it has **at least** the same keys with conforming types (extra keys are OK — structural subtyping).
- Functions are **contravariant** in parameters and **covariant** in return type: `(S1,...) → U <: (T1,...) → V` iff `Ti <: Si` and `U <: V`.

### Why You Should Care

Type conformance is not academic trivia -- it controls three things you hit constantly:
- Whether a function argument is accepted or silently becomes `null`.
- Whether a decision output matches its declared type.
- Whether one of FEEL's implicit conversions kicks in (see the next section).

---

## 15.2 Implicit Type Conversions

FEEL quietly converts values in four situations. These are helpful once you know they exist, and baffling when you do not:

### To Singleton List

When a single value shows up where a list is expected, FEEL wraps it automatically:

```
3[item > 2]    // 3 is converted to [3], filter applies → [3]
```

### From Singleton List

The reverse also works: a one-element list is unwrapped to a scalar when a scalar is expected:

```
contains(["foobar"], "of")
// ["foobar"] is unwrapped to "foobar", then contains("foobar", "of") → false
```

### Date to Date-Time

When a function expects `date and time` but you hand it a plain `date`, FEEL promotes it by tacking on UTC midnight:

```
// If a function expects date and time and receives a date:
@"2024-03-15" → @"2024-03-15T00:00:00"
```

### Decimal to Integer

When a function expects an integer (like `substring`'s position argument), FEEL silently truncates the decimal part:

```
substring("hello", 2.7)    // position treated as 2
```

### Conforms-To Conversion

If none of these four conversions apply and the types still do not match, the result is `null`. No exception, no warning -- just `null`.

---

## 15.3 Scope and the Context Stack

When FEEL sees a name like `Applicant.Age`, it needs to figure out what `Applicant` refers to. The answer lives in the **scope** -- an ordered stack of contexts that FEEL searches from top to bottom.

### The Context Stack (from first to last)

1. **Local context**: If the expression is inside a context entry, the containing context is first. Earlier entries are in scope.
2. **Enclosing contexts**: If the context is nested inside another context, the outer context is next.
3. **Global context**: The context created by the DMN runtime containing input data, decision results, and BKM definitions.
4. **Built-in context**: Contains all built-in functions (`sum`, `count`, `date`, etc.).

### Special Contexts

Certain expressions temporarily push a special context onto the top of the stack, which is why variables like `item` and `partial` seem to appear out of thin air:

- **Filter expressions**: Push `{item: current_element}` (and all context entries of the element if it is a context).
- **For expressions**: Push `{variable: current_element, partial: accumulated_results}`.
- **Quantified expressions**: Push `{variable: current_element}`.

### Name Resolution

FEEL resolves names by searching the context stack top-to-bottom. When multiple matches are possible, the **longest match** wins -- and this can surprise you:

```
{
  a: 1,
  b: 2,
  a - b: 10,
  result: a - b    // Is this name "a - b" (= 10) or expression "a minus b" (= -1)?
}
```

Since `a - b` is a key in the same context, the name match wins: `result` = `10`, not `-1`. This is one of FEEL's most counterintuitive behaviours.

---

## 15.4 Name Ambiguity

FEEL lets you name things with characters that double as operators: `-`, `+`, `*`, `/`, `.`, `'`. This is great for readability (`Profit and loss`) and terrible for parsers:

```
Profit and loss       // Is this the name "Profit and loss"?
                      // Or is it "Profit" AND "loss"?

a-b                   // Name "a-b" or expression "a minus b"?

what if?              // Name "what if?" or... something else?
```

### Resolution Rule

FEEL matches names **left-to-right**, preferring the **longest match**.

If `a-b` is an in-scope name, it wins. If not, the parser falls back to `a minus b`.

### Disambiguation

Use parentheses to force expression interpretation:

```
(a) - (b)      // always subtraction, even if "a-b" is in scope
```

### How to Stay Out of Trouble

1. Avoid operator characters in names whenever you can.
2. If the business domain forces your hand (e.g., "Profit and loss"), know the resolution rules cold.
3. When in doubt, wrap each name in parentheses: `(a) - (b)` is always subtraction.
4. Prefer underscores or spaces: `risk_score` or `risk score` over `risk-score`.

---

## 15.5 XML Data in FEEL

If your data arrives as XML (and in enterprise systems, it often does), FEEL can consume it by mapping XML structures to native FEEL values.

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

## 15.6 JSON and FEEL

JSON is the more common case for modern systems, and the mapping is nearly one-to-one:

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

## 15.7 The Full FEEL Grammar

If you ever need to build a parser, write a syntax highlighter, or settle an argument about what counts as valid FEEL, here is the grammar. It has 68 rules in ISO EBNF notation. The key ones:

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

You now know how FEEL works at the deepest level. Chapter 16 puts that knowledge to practical use: deriving comprehensive test cases from decision tables and expressions, measuring coverage, debugging the tricky stuff, and wiring it all into CI/CD. After that, the appendices provide reference material you will reach for daily.

---

[Previous: Chapter 14: S-FEEL, B-FEEL, and the Dialects](chapter-14-feel-dialects.md) | [Next: Chapter 16: Testing FEEL](chapter-16-testing-feel.md)
