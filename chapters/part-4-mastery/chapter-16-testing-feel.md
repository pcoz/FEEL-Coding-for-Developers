# Chapter 16: Testing FEEL

> *"A decision table is its own test specification."*

---

## 16.1 Why FEEL Is Unusually Testable

Most business logic buried in application code is hard to test. You need to construct objects, mock services, manage state, and trace through control flow to understand what a method does. FEEL is fundamentally different:

1. **Every expression is a pure function.** Given the same inputs, it always produces the same output. No side effects, no hidden state, no database calls.
2. **Decision tables enumerate their own cases.** Each row is a rule with explicit input conditions and expected output — the table *is* the specification.
3. **The type system is small and finite.** FEEL has a handful of types (number, string, boolean, date, time, duration, list, context, range, null). You can enumerate boundary cases exhaustively.
4. **Null propagation is deterministic.** Every operator and function has defined null behaviour — typically returning null. You do not need to guess what happens with missing data.

These properties make FEEL code not just testable, but **analysable** — you can derive comprehensive test coverage directly from the structure of the expressions themselves.

---

## 16.2 Deriving Tests from Decision Tables

A decision table with hit policy U (Unique) is the simplest case: each rule defines a mutually exclusive partition of the input space. Complete coverage means testing every rule at least once.

### Worked Example: Shipping Decision

| U | Order Total | Customer Tier | Zone | Shipping Method |
|---|------------|---------------|------|----------------|
| 1 | > 100 | "Gold" | - | "FREE" |
| 2 | > 50 | "Silver" | "A" | "STANDARD" |
| 3 | > 50 | "Silver" | not("A") | "PREMIUM" |
| 4 | <= 50 | - | "A" | "STANDARD" |
| 5 | <= 50 | - | not("A") | "PREMIUM" |

**Step 1: One test per rule (rule coverage).**

Each rule defines exactly what inputs it expects. Read the table — it tells you what to test:

| Test | Order Total | Customer Tier | Zone | Expected Output |
|------|------------|---------------|------|----------------|
| T1 (Rule 1) | 150 | "Gold" | "B" | "FREE" |
| T2 (Rule 2) | 75 | "Silver" | "A" | "STANDARD" |
| T3 (Rule 3) | 75 | "Silver" | "B" | "PREMIUM" |
| T4 (Rule 4) | 30 | "Bronze" | "A" | "STANDARD" |
| T5 (Rule 5) | 30 | "Bronze" | "B" | "PREMIUM" |

Five rules, five tests. **100% rule coverage** — derived mechanically from the table.

**Step 2: Boundary tests.**

Each numeric threshold creates a boundary. The table uses `> 100`, `> 50`, and `<= 50`. Test at the boundaries:

| Test | Order Total | Customer Tier | Zone | Expected Output |
|------|------------|---------------|------|----------------|
| T6 | 100 | "Gold" | "A" | ? (does NOT match Rule 1) |
| T7 | 100.01 | "Gold" | "A" | "FREE" |
| T8 | 50 | "Silver" | "A" | ? (does NOT match Rule 2) |
| T9 | 50.01 | "Silver" | "A" | "STANDARD" |

Boundary tests often reveal gaps or ambiguities in the specification. If `Order Total = 100` and `Customer Tier = "Gold"`, no rule matches — is that intended? This is a **completeness** issue, and the test exposed it.

**Step 3: Null and missing input tests.**

What if `Customer Tier` is null? What if `Zone` is missing?

| Test | Order Total | Customer Tier | Zone | Expected Output |
|------|------------|---------------|------|----------------|
| T10 | 150 | null | "A" | null (no rule matches) |
| T11 | 75 | "Silver" | null | null (no rule matches) |

These tests verify that missing data is handled safely — the table returns null rather than an arbitrary match.

### The Key Insight

You did not need to read code, trace logic, or guess at edge cases. **The decision table told you what to test.** Each row is a test case. Each boundary is a boundary test. Each input that could be null is a null test. The total number of tests is small and bounded: `rules + boundaries + null variants`.

This is why FEEL is unusually testable — the structure of the decision *is* the structure of the test suite.

---

## 16.3 Deriving Tests from FEEL Expressions

FEEL expressions that are not in a decision table require a different approach. But because FEEL is an expression language with no side effects, you can still derive tests systematically.

### Technique: Condition/Decision Coverage

Every `if/then/else` in a FEEL expression creates a decision point. Every comparison creates a condition. List them, test both sides.

**Expression:**

```
{
  eligible: Applicant.Age >= 18 and Applicant.Income >= 30000,
  risk: if Credit Score >= 750 then "Low"
        else if Credit Score >= 650 then "Medium"
        else "High",
  max loan: if eligible and risk = "Low" then 500000
            else if eligible and risk = "Medium" then 200000
            else if eligible then 50000
            else 0
}.max loan
```

**Step 1: Identify decision points.**

| # | Condition | True Test | False Test |
|---|-----------|-----------|------------|
| C1 | `Age >= 18` | Age = 25 | Age = 16 |
| C2 | `Income >= 30000` | Income = 50000 | Income = 20000 |
| C3 | `Credit Score >= 750` | Score = 800 | Score = 700 |
| C4 | `Credit Score >= 650` | Score = 700 | Score = 500 |
| C5 | `eligible and risk = "Low"` | Age=25, Income=50K, Score=800 | (any of the three fails) |
| C6 | `eligible and risk = "Medium"` | Age=25, Income=50K, Score=700 | (any fails) |
| C7 | `eligible` (fallthrough) | Age=25, Income=50K, Score=500 | Age=16 |

**Step 2: Design tests that cover all paths.**

| Test | Age | Income | Score | Expected max loan |
|------|-----|--------|-------|-------------------|
| T1 | 25 | 50000 | 800 | 500000 (eligible, Low risk) |
| T2 | 25 | 50000 | 700 | 200000 (eligible, Medium risk) |
| T3 | 25 | 50000 | 500 | 50000 (eligible, High risk) |
| T4 | 16 | 50000 | 800 | 0 (not eligible) |
| T5 | 25 | 20000 | 800 | 0 (not eligible) |
| T6 | 25 | 50000 | 750 | 500000 (boundary: exactly 750) |
| T7 | 25 | 50000 | 650 | 200000 (boundary: exactly 650) |
| T8 | 18 | 30000 | 800 | 500000 (boundary: minimum eligible) |
| T9 | null | 50000 | 800 | 0 (null age → eligible is null → false path) |

Nine tests cover every path, every boundary, and the null case. The tests were derived from the expression structure, not invented from imagination.

### Why This Works

FEEL expressions have a property that imperative code does not: **the entire computation is visible in the expression itself.** There are no hidden branches, no exception handlers, no early returns, no mutable state that might change between lines. Every `if` is visible. Every comparison is visible. Every possible null source is visible.

This means you can achieve **comprehensive coverage** — every path through the expression is tested — with a number of test cases that is proportional to the number of decision points, not exponential. In the example above, the expression has 7 conditions and 4 paths. Nine tests achieve full coverage. An equivalent Java method with the same logic would typically require more tests because of constructor paths, null pointer possibilities, exception handling, and interaction with mutable state.

---

## 16.4 Completeness and Consistency Analysis

Decision tables can be analysed statically — before running any tests — for two properties:

### Completeness: Are All Input Combinations Covered?

A decision table is **complete** if every possible combination of input values matches at least one rule. If the table is incomplete, some inputs will produce no output (null for single-hit policies, empty list for collect policies).

**How to check:** For each input column, list the partitions (the distinct conditions). Count the combinations. Verify that every combination is covered by at least one rule.

**Example (incomplete):**

| U | Status | Amount | Action |
|---|--------|--------|--------|
| 1 | "Active" | > 1000 | "Review" |
| 2 | "Active" | <= 1000 | "Approve" |
| 3 | "Suspended" | - | "Reject" |

What if `Status = "Cancelled"`? No rule matches. The table is incomplete for the `Status` domain. Either add a catch-all rule or explicitly document that `Status` can only be "Active" or "Suspended".

### Consistency: Do Rules Conflict?

With hit policy U, no two rules should match the same input. If they do, the engine signals an error.

**Example (inconsistent):**

| U | Age | Income | Decision |
|---|-----|--------|----------|
| 1 | >= 18 | >= 50000 | "Approve" |
| 2 | >= 21 | >= 30000 | "Approve" |

An applicant with `Age = 25, Income = 60000` matches both rules. The input ranges overlap. Fix by making the conditions mutually exclusive:

| U | Age | Income | Decision |
|---|-----|--------|----------|
| 1 | [18..21) | >= 50000 | "Approve" |
| 2 | >= 21 | >= 30000 | "Approve" |

### Static Analysis as a Test

Completeness and consistency checks can be automated. Some DMN engines and modelling tools (Trisotech, Camunda Modeler) perform these checks at design time. You can also write a simple script:

1. Extract the input conditions from the decision table.
2. Enumerate the boundary partitions.
3. For each combination, count how many rules match.
4. Flag: 0 matches = incomplete. 2+ matches (with hit policy U) = inconsistent.

This is a test that requires **zero test data** — it analyses the structure of the decision itself.

---

## 16.5 Testing Strategies

### 16.5.1 Unit Testing FEEL Expressions

Treat each FEEL expression as a function. Provide input contexts, assert expected outputs.

**Java (feel-scala):**

```java
@ParameterizedTest
@CsvSource({
    "25, 50000, 800, 500000",
    "25, 50000, 700, 200000",
    "25, 50000, 500, 50000",
    "16, 50000, 800, 0"
})
void testMaxLoan(int age, int income, int score, int expected) {
    var ctx = Map.of(
        "Applicant", Map.of("Age", age, "Income", income),
        "Credit Score", score
    );
    var result = engine.evaluateExpression(feelExpression, ctx);
    assertEquals(BigDecimal.valueOf(expected), result.right().get());
}
```

**JavaScript (feelin):**

```javascript
const testCases = [
  { inputs: { Age: 25, Income: 50000, Score: 800 }, expected: 500000 },
  { inputs: { Age: 16, Income: 50000, Score: 800 }, expected: 0 },
];

testCases.forEach(({ inputs, expected }) => {
  const result = evaluate(expression, inputs);
  assert.strictEqual(result, expected);
});
```

### 16.5.2 Table-Driven Testing

For decision tables, generate tests directly from the table rows:

```java
@Test
void testAllRulesInShippingTable() {
    // Each rule in the table becomes a test case
    var testCases = List.of(
        Map.of("inputs", Map.of("Total", 150, "Tier", "Gold", "Zone", "B"),
               "expected", "FREE"),
        Map.of("inputs", Map.of("Total", 75, "Tier", "Silver", "Zone", "A"),
               "expected", "STANDARD"),
        // ... one entry per rule
    );

    for (var tc : testCases) {
        var result = dmnRuntime.evaluateDecision("Shipping", tc.get("inputs"));
        assertEquals(tc.get("expected"), result);
    }
}
```

### 16.5.3 Regression Testing with Golden Files

Save the expected output of a decision as a JSON file. On each test run, compare the actual output against the golden file.

```
tests/
  shipping-decision/
    test-001-input.json    → { "Total": 150, "Tier": "Gold", "Zone": "B" }
    test-001-expected.json → { "Shipping Method": "FREE" }
    test-002-input.json    → { "Total": 75, "Tier": "Silver", "Zone": "A" }
    test-002-expected.json → { "Shipping Method": "STANDARD" }
```

When a business rule changes, update the golden files. The diff shows exactly what changed in the decision's behaviour.

### 16.5.4 Property-Based Testing

Instead of testing specific values, test properties that should always hold:

```java
@Property
void maxLoanIsNonNegative(@ForAll @IntRange(min=0, max=100) int age,
                          @ForAll @IntRange(min=0, max=200000) int income,
                          @ForAll @IntRange(min=300, max=850) int score) {
    var ctx = Map.of("Applicant", Map.of("Age", age, "Income", income),
                     "Credit Score", score);
    var result = (BigDecimal) engine.evaluateExpression(expr, ctx).right().get();
    assertThat(result).isGreaterThanOrEqualTo(BigDecimal.ZERO);
}

@Property
void eligibleApplicantsGetNonZeroLoan(@ForAll @IntRange(min=18, max=100) int age,
                                       @ForAll @IntRange(min=30000, max=200000) int income,
                                       @ForAll @IntRange(min=300, max=850) int score) {
    var ctx = Map.of("Applicant", Map.of("Age", age, "Income", income),
                     "Credit Score", score);
    var result = (BigDecimal) engine.evaluateExpression(expr, ctx).right().get();
    assertThat(result).isGreaterThan(BigDecimal.ZERO);
}
```

Property-based testing is especially powerful for FEEL because the pure-function nature guarantees that the test is deterministic — the same inputs always produce the same result.

### 16.5.5 DMN TCK Format

The DMN Technology Compatibility Kit defines a standard XML format for test cases:

```xml
<testCases xmlns="http://www.omg.org/spec/DMN/20160901/testcase">
  <testCase id="001" name="Gold customer, high total, free shipping">
    <inputNode name="Order Total">150</inputNode>
    <inputNode name="Customer Tier">Gold</inputNode>
    <inputNode name="Zone">B</inputNode>
    <resultNode name="Shipping Method">
      <expected>
        <value>FREE</value>
      </expected>
    </resultNode>
  </testCase>
</testCases>
```

Benefits:
- Vendor-neutral: the same tests run on any DMN engine.
- Self-documenting: each test case has a name and clear inputs/outputs.
- Executable: engines provide TCK runners that consume these files directly.

---

## 16.6 Coverage Metrics for FEEL

### Rule Coverage

Percentage of decision table rules exercised by at least one test case.

**Target: 100%.** Every rule should be tested. Since each rule is a distinct business case, untested rules mean untested business logic.

### Condition Coverage

Percentage of conditions (comparisons, boolean sub-expressions) that have been tested with both true and false inputs.

**Target: 100%.** For FEEL expressions with `if/then/else` chains, ensure every condition has been both satisfied and not satisfied.

### Boundary Coverage

Percentage of numeric and temporal boundaries tested with values at, just above, and just below the threshold.

**Target: All boundaries.** This is where off-by-one errors hide. `>= 18` should be tested with 17, 18, and 19.

### Null Coverage

Percentage of inputs tested with null values.

**Target: All nullable inputs.** For each input that could plausibly be null, verify that the expression handles it gracefully (typically by returning null or a default).

### How to Measure

Most DMN engines do not provide built-in coverage tools. But because decision tables are structured data, you can build simple coverage checkers:

1. Parse the decision table (from DMN XML or a tabular format).
2. For each test case, determine which rule(s) matched.
3. Report: which rules were exercised, which were not.
4. For FEEL expressions, instrument the engine to track which `if` branches were taken.

---

## 16.7 Debugging FEEL

### Common Errors and Their Causes

| Symptom | Likely Cause | How to Diagnose |
|---------|-------------|-----------------|
| Expression returns `null` unexpectedly | A sub-expression evaluated to null and propagated | Check each input; use `x != null` to find the null source (or `is defined(x)` / `get or else(x, default)` on Camunda) |
| Decision table returns no result | No rule matched the input | Check completeness; test with the specific input against each rule's conditions |
| Decision table returns wrong result | Wrong rule matched | Check rule conditions for overlaps; trace which rule fired |
| Type error (null from operation) | Operands are wrong types (e.g., `"5" + 3`) | Check types with `instance of`; FEEL does not coerce types |
| Date arithmetic gives unexpected result | Mixing `days and time duration` with `years and months duration` | Check that duration types are compatible with the operation |

### Debugging Techniques

**1. Decompose into a context:**

Instead of debugging a complex one-liner, break it into named steps:

```
// Hard to debug:
if sum(for item in Order.Items return item.Qty * item.Price) > 100 then "FREE" else "PAID"

// Easy to debug:
{
  line totals: for item in Order.Items return item.Qty * item.Price,
  subtotal: sum(line totals),
  shipping: if subtotal > 100 then "FREE" else "PAID"
}
```

Now you can inspect `line totals` and `subtotal` independently.

**2. Use `assert()` (Camunda only):**

```
{
  age: assert(Applicant.Age, Applicant.Age != null, "Age must not be null"),
  score: assert(Credit Score, Credit Score >= 300, "Invalid credit score")
}
```

If the assertion fails, the engine logs the message. This is invaluable during development.

**3. Test in the playground:**

Every major engine has a playground or REPL. Use it to test sub-expressions with known inputs before assembling the full expression.

**4. Compare engines:**

If an expression produces unexpected results, test it in a second engine. If the results differ, you have found an engine-specific behaviour — check Appendix D.

---

## 16.8 CI/CD Integration

### Pipeline Structure

```
┌──────────┐    ┌──────────────┐    ┌─────────────┐    ┌──────────┐
│ Commit   │───▶│ Lint DMN/    │───▶│ Run FEEL/   │───▶│ Deploy   │
│ .dmn +   │    │ FEEL files   │    │ DMN tests   │    │ decision │
│ tests    │    │              │    │             │    │ service  │
└──────────┘    └──────────────┘    └─────────────┘    └──────────┘
                      │                    │
                      ▼                    ▼
                Completeness/        Coverage report
                consistency          (rule + boundary)
                analysis
```

### Checklist

- [ ] `.dmn` files are version-controlled alongside application code.
- [ ] Test cases are stored as JSON input/expected pairs or DMN TCK XML.
- [ ] The build pipeline evaluates all test cases against the target engine.
- [ ] Completeness analysis runs on every PR (flags incomplete tables).
- [ ] Coverage reports show which rules and conditions are exercised.
- [ ] Decision changes produce a diff of test outputs (golden file comparison).

---

## Summary

- FEEL is unusually testable because it is pure, side-effect free, and structurally transparent.
- Decision tables specify their own test cases: one test per rule achieves rule coverage.
- FEEL expressions can be analysed for conditions and boundaries to derive comprehensive tests mechanically.
- Completeness and consistency can be checked statically — before writing any test data.
- Coverage metrics for FEEL include: rule coverage, condition coverage, boundary coverage, and null coverage.
- The number of tests needed for comprehensive coverage is proportional to the number of decision points, not exponential — because there is no hidden state or branching.
- CI/CD pipelines should lint, analyse, and test `.dmn` files alongside application code.

---

## What Comes Next

With testing mastered, you have the tools to write FEEL expressions confidently in production. Return to the appendices for quick reference material, cross-language comparisons, and engine compatibility details.

---

[Previous: Chapter 15: Advanced Topics](chapter-15-advanced-topics.md) | [Next: Appendix A: FEEL Quick Reference Card](../part-5-appendices/appendix-a-quick-reference.md)
