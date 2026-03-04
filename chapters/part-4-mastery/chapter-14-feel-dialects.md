# Chapter 14: S-FEEL, B-FEEL, and the Dialects

> *"Three dialects of the same language, each tuned for a different audience."*

---

## 14.1 Why Dialects?

Not everyone who touches a DMN model is a developer. The DMN spec acknowledges this by defining three FEEL dialects, each tuned for a different level of complexity and a different audience:

| Dialect | Introduced | Audience | Null Handling |
|---------|-----------|----------|---------------|
| **S-FEEL** (Simple FEEL) | DMN 1.0 | Minimal decision tables | Three-valued (like full FEEL) |
| **FEEL** (Full FEEL) | DMN 1.1+ | Developers and analysts | Three-valued (`true`, `false`, `null`) |
| **B-FEEL** (Business FEEL) | DMN 1.6 | Non-technical business users | **Two-valued** (`true`, `false`) |

---

## 14.2 S-FEEL — The Restricted Subset

Think of S-FEEL as FEEL with training wheels bolted on so tightly you can barely steer. It supports:

- Arithmetic: `+`, `-`, `*`, `/`, `**`
- Comparisons: `=`, `!=`, `<`, `<=`, `>`, `>=`
- Simple values: numbers, strings, booleans, dates, times, durations
- Unary tests: for decision table input entries
- Qualified names: `Applicant.Age`

It does **not** support:
- `if / then / else`
- `for / return`
- `some / every ... satisfies`
- Lists (`[1, 2, 3]`)
- Contexts (`{key: value}`)
- Functions
- Built-in function library
- Filters (`list[condition]`)

### When Is S-FEEL Enough?

Almost never. Even the DMN specification admits defeat here:

> "Experience with DMN since its release has shown that few if any complete decision models can be defined using S-FEEL."

Individual decision table cells can get by with S-FEEL, but the moment you need a context, a function call, or a list operation, you are writing full FEEL whether you meant to or not.

**Bottom line:** Always target full FEEL. S-FEEL exists for conformance-level paperwork, not for real-world use.

---

## 14.3 B-FEEL — Business-Friendly FEEL (DMN 1.6)

B-FEEL arrived in DMN 1.6 with a bold promise: same grammar as FEEL, but `null` never leaks into your results. When something goes wrong, you get a safe default instead of a silent null bomb.

### The Core Difference

In FEEL, errors produce `null`, and that `null` silently poisons every downstream expression it touches.

B-FEEL takes the opposite stance: errors produce **type-appropriate defaults**, so nothing ever "fails":

| Return Type | FEEL error result | B-FEEL error result |
|------------|-------------------|---------------------|
| Boolean | `null` | `false` |
| Number | `null` | `0` |
| String | `null` | `""` |
| List | `null` | `[]` |
| Days and time duration | `null` | `@"PT0S"` |
| Years and months duration | `null` | `@"P0M"` |
| Date | `null` | `@"1970-01-01"` |
| Date and time | `null` | `@"1970-01-01T00:00:00"` |
| Time | `null` | `@"00:00:00"` |

### Two-Valued Logic

This is where things get interesting (and potentially dangerous). B-FEEL collapses three-valued logic down to two values. Boolean operations *never* return `null`:

| Expression | FEEL Result | B-FEEL Result |
|-----------|-------------|---------------|
| `"a" = 1` | `null` | `false` |
| `"a" < 1` | `null` | `false` |
| `null >= 1` | `null` | `false` |
| `not("a")` | `null` | `false` |
| `true and "x"` | `null` | `false` |
| `false or "x"` | `null` | `false` |
| `"a" in [1..100]` | `null` | `false` |
| `null between 1 and 100` | `null` | `false` |
| `"a" != 1` | `null` | **`true`** |

Spot the last row: `!=` with incompatible types returns `true` in B-FEEL. The reasoning is straightforward -- a string and a number are, in fact, not equal.

### Numeric Error Handling

Numeric functions that would blow up with `null` in FEEL quietly return `0` in B-FEEL:

| Expression | FEEL | B-FEEL |
|-----------|------|--------|
| `decimal("a", 0)` | `null` | `0` |
| `round up("5.5", 0)` | `null` | `0` |
| `sum([1, "a", 3])` | `null` | `4` (ignores non-numeric!) |
| `mean([1, "a", 3])` | `null` | `2` (ignores non-numeric!) |

Notice the last two rows: B-FEEL's aggregation functions (except `count()`) **silently skip non-numeric elements** instead of failing. Whether that is helpful or terrifying depends on your use case.

### String Error Handling

| Expression | FEEL | B-FEEL |
|-----------|------|--------|
| `substring(null, 1)` | `null` | `""` |
| `upper case(123)` | `null` | `""` |

### When to Use B-FEEL

B-FEEL targets decision models built by **non-IT business users** who find three-valued logic and null-propagation baffling. It trades diagnostic precision for predictability -- nothing ever "breaks."

**But here is the catch:** B-FEEL can mask real errors. When `sum([1, "a", 3])` silently returns `4` instead of screaming that `"a"` is not a number, you lose a critical signal that your input data is garbage. If you are a developer writing production-grade logic, full FEEL's explicit `null` is almost always the safer choice.

### Activating B-FEEL

Flip a single attribute on the DMN definitions element:

```xml
<definitions xmlns="https://www.omg.org/spec/DMN/20191111/MODEL/"
             expressionLanguage="https://www.omg.org/spec/DMN/20240513/B-FEEL/">
  ...
</definitions>
```

> **ENGINE NOTE:** As of DMN 1.6's publication, B-FEEL support has not shipped in any major open-source engine. Camunda and Trisotech have it on their roadmaps; Apache KIE has not announced plans. Check your engine's release notes for current status before relying on B-FEEL in production.

### Worked Example 14.1 — FEEL vs B-FEEL Side by Side

| # | Expression | FEEL | B-FEEL |
|---|-----------|------|--------|
| 1 | `"a" = 1` | `null` | `false` |
| 2 | `"a" != 1` | `null` | `true` |
| 3 | `true and null` | `null` | `false` |
| 4 | `false or null` | `null` | `false` |
| 5 | `not(null)` | `null` | `false` |
| 6 | `1 / 0` | `null` | `0` |
| 7 | `sum([1, "x", 3])` | `null` | `4` |
| 8 | `substring(null, 1)` | `null` | `""` |
| 9 | `before(@"2024-01-01", null)` | `null` | `false` |
| 10 | `@"2024-01-01" + null` | `null` | `@"1970-01-01"` |

---

## 14.4 Choosing the Right Dialect

| Criterion | Use Full FEEL | Use B-FEEL |
|-----------|--------------|-----------|
| Authors are developers | Yes | No |
| Precision matters (financial, regulatory) | Yes | No |
| Need to detect bad input data | Yes | No |
| Authors are business users | Consider | Yes |
| "Fail loud" philosophy | Yes | No |
| "Never crash" philosophy | No | Yes |

If developers write and maintain the FEEL expressions -- which describes most production systems -- **full FEEL is the right choice**. Save B-FEEL for self-service decision modelling tools where business users build their own rules and you would rather show a zero than a stack trace.

---

## Summary

- **S-FEEL**: A minimal subset. Rarely sufficient in practice. Use full FEEL instead.
- **FEEL**: The standard. Three-valued logic. Errors produce `null`. Preferred for developer-authored logic.
- **B-FEEL**: Two-valued logic. Errors produce type-appropriate defaults. New in DMN 1.6. Designed for non-technical users.

---

## What Comes Next

Ready to look under the hood? Chapter 15 digs into the advanced corners of FEEL: the type lattice, implicit conversions, scope resolution, name ambiguity, and how XML data maps into FEEL's world.

---

[Previous: Chapter 13: FEEL in the Wild](chapter-13-engine-integration.md) | [Next: Chapter 15: Advanced Topics](chapter-15-advanced-topics.md)
