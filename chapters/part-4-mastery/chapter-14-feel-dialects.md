# Chapter 14: S-FEEL, B-FEEL, and the Dialects

> *"Three dialects of the same language, each tuned for a different audience."*

---

## 14.1 Why Dialects?

The DMN specification defines three variants of FEEL to serve different conformance levels and user needs:

| Dialect | Introduced | Audience | Null Handling |
|---------|-----------|----------|---------------|
| **S-FEEL** (Simple FEEL) | DMN 1.0 | Minimal decision tables | Three-valued (like full FEEL) |
| **FEEL** (Full FEEL) | DMN 1.1+ | Developers and analysts | Three-valued (`true`, `false`, `null`) |
| **B-FEEL** (Business FEEL) | DMN 1.6 | Non-technical business users | **Two-valued** (`true`, `false`) |

---

## 14.2 S-FEEL — The Restricted Subset

S-FEEL is FEEL with training wheels. It supports:

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

Almost never. The DMN specification itself notes:

> "Experience with DMN since its release has shown that few if any complete decision models can be defined using S-FEEL."

Individual decision tables can use S-FEEL for their input and output entries, but any surrounding logic (contexts, function calls, list operations) requires full FEEL.

**Recommendation:** Always target full FEEL. S-FEEL exists for conformance level documentation, not for practical use.

---

## 14.3 B-FEEL — Business-Friendly FEEL (DMN 1.6)

B-FEEL is new in DMN 1.6. It shares the same grammar as FEEL but changes the semantics to eliminate `null` as an error value.

### The Core Difference

In FEEL, when something goes wrong, the result is `null`. `null` then propagates silently through further operations.

In B-FEEL, errors produce **type-appropriate defaults** instead of `null`:

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

The most significant change: B-FEEL uses **two-valued logic** instead of three-valued logic. Boolean operations never return `null`:

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

Note the last row: `!=` with incompatible types returns `true` in B-FEEL (they are indeed not equal).

### Numeric Error Handling

Functions that would return `null` in FEEL return `0` in B-FEEL:

| Expression | FEEL | B-FEEL |
|-----------|------|--------|
| `decimal("a", 0)` | `null` | `0` |
| `round up("5.5", 0)` | `null` | `0` |
| `sum([1, "a", 3])` | `null` | `4` (ignores non-numeric!) |
| `mean([1, "a", 3])` | `null` | `2` (ignores non-numeric!) |

Notice that B-FEEL list aggregation functions (except `count()`) **ignore non-numeric elements** rather than failing.

### String Error Handling

| Expression | FEEL | B-FEEL |
|-----------|------|--------|
| `substring(null, 1)` | `null` | `""` |
| `upper case(123)` | `null` | `""` |

### When to Use B-FEEL

B-FEEL is designed for decision models authored primarily by **non-IT business users** who find null-propagation confusing and error-prone. It trades correctness for predictability — no expression ever "fails" silently.

**Trade-off warning:** B-FEEL can mask real errors. When `sum([1, "a", 3])` silently returns `4` instead of flagging that `"a"` is not a number, you lose a valuable signal that something is wrong with the input data. For developer-authored, production-grade logic, full FEEL's explicit `null` is usually preferable.

### Activating B-FEEL

Set the expression language on the DMN definitions element:

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

For most production systems where developers write and maintain the FEEL expressions, **full FEEL is the right choice**. B-FEEL is appropriate for self-service decision modelling tools where business users create their own rules.

---

## Summary

- **S-FEEL**: A minimal subset. Rarely sufficient in practice. Use full FEEL instead.
- **FEEL**: The standard. Three-valued logic. Errors produce `null`. Preferred for developer-authored logic.
- **B-FEEL**: Two-valued logic. Errors produce type-appropriate defaults. New in DMN 1.6. Designed for non-technical users.

---

## What Comes Next

Chapter 15 covers the advanced corners of FEEL: the type lattice, implicit conversions, scope and context resolution, name ambiguity, and XML data mapping.

---

[Previous: Chapter 13: FEEL in the Wild](chapter-13-engine-integration.md) | [Next: Chapter 15: Advanced Topics](chapter-15-advanced-topics.md)
