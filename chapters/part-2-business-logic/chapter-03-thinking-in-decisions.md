# Chapter 3: Thinking in Decisions

> *"A decision table is not a piece of software. It is a piece of organisational truth that happens to be executable."*

---

Chapter 2 gave you the vocabulary — the ten analytical primitives that every business rule is built from. This chapter gives you the **structures** for organising those primitives into executable decisions.

The temptation when learning any language — FEEL included — is to dive straight into grammar and built-in functions. Resist that temptation. The hardest part of writing correct FEEL is not the syntax. It is choosing the right structure for the logic: when to use a decision table, when to use a step-by-step computation, and how to connect multiple decisions into a coherent model.

Four structures carry almost all the weight: **decision tables**, **contexts** (step-by-step computations), **Decision Requirements Graphs** (dependency diagrams), and the **mapping table** that connects every primitive from Chapter 2 to its FEEL construct.

---

## 3.1 Decision Tables: The Workhorse

Classification. Threshold Tests. Range Membership. Enumeration Matches. All the primitives you met in Chapter 2 share a common shape: evaluate conditions against inputs, produce a result. A **decision table** is the natural structure for expressing every one of them.

Use a decision table when:

- The decision has a small number of input dimensions (2–5).
- The output is one of a finite set of values (classification) or a computed value.
- There are multiple rules that can be listed and verified row by row.
- Business stakeholders need to review and approve the rules.

### Anatomy of a Decision Table

Every decision table is built from the same parts:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Information Item Name                                         [H] │
├──────────────────┬──────────────────┬────────────────────────────────┤
│  Input Expr 1    │  Input Expr 2    │  Output Label                 │
│  input values    │  input values    │  output values                │
├──────────────────┼──────────────────┼────────────────────────────────┤
│  1  condition    │  condition       │  conclusion                   │
│  2  condition    │  condition       │  conclusion                   │
│  3  condition    │  -               │  conclusion                   │
└──────────────────┴──────────────────┴────────────────────────────────┘
```

- **Input expressions**: The values being tested (column headers). These are FEEL expressions.
- **Input values** (optional): The allowed values for each input. Constrain what can appear in the cells below.
- **Input entries**: The conditions in each rule row. These are *unary tests* — a special sub-language of FEEL.
- **Output label**: The name of the result.
- **Output values** (optional): The allowed output values.
- **Output entries**: The conclusion for each rule row. These are FEEL expressions.
- **Hit policy** (the letter in `[H]`): What happens when multiple rules match.
- **Rules**: Each row is one rule. Read it as: "If input 1 matches AND input 2 matches, then the output is..."

Notice what happened: each input entry is a **Threshold Test**, **Range Membership**, or **Enumeration Match**. Each row is a **Conjunction** of those tests. The table as a whole is a **Classification** or **Conditional Mapping**. Nothing new here — a decision table is just a compact notation for combining primitives you already understand.

### Hit Policies

What happens when the inputs match more than one rule? Or zero rules? The hit policy answers that question — and choosing the right one is one of the most important decisions you will make in DMN.

#### Single-Hit Policies

One input, one answer:

| Policy | Letter | Meaning | When to use |
|--------|--------|---------|------------|
| **Unique** | U | Exactly one rule matches. If zero or more than one match, it is an error in the table design. | When rules are designed to be mutually exclusive. The most common policy. |
| **Any** | A | Multiple rules may match, but they must all produce the same output. | When you have redundant rules for readability. |
| **Priority** | P | Multiple rules may match with different outputs. The output with the highest priority (as defined by the output values list) wins. | When rules overlap and you want the "most important" result. |
| **First** | F | Multiple rules may match. The first match (by row order) wins. | When rules are ordered by specificity — like `if/else if/else`. |

#### Multiple-Hit Policies

Sometimes you want *all* the answers, not just one:

| Policy | Letter | Meaning | When to use |
|--------|--------|---------|------------|
| **Collect** | C | All matching rules fire. Outputs are collected into a list (arbitrary order). | When you need all applicable results (e.g., all fees that apply). |
| **Collect Sum** | C+ | All matching rules fire. Outputs are summed. | Scorecards: partial scores summed into a total. |
| **Collect Count** | C# | All matching rules fire. Returns the count of matches. | "How many rules apply?" |
| **Collect Min** | C< | All matching rules fire. Returns the minimum output. | "What is the lowest applicable rate?" |
| **Collect Max** | C> | All matching rules fire. Returns the maximum output. | "What is the highest applicable fee?" |
| **Rule Order** | R | All matching rules fire. Outputs returned in rule order. | When the order of rules matters for downstream processing. |
| **Output Order** | O | All matching rules fire. Outputs returned in the order defined by output values. | When you need a canonical ordering. |

See the connection back to Chapter 2? **Collect Sum** (C+) is the table-level version of the **Aggregation** primitive. **Priority** (P) and **Output Order** (O) are the table-level versions of **Ordering and Priority**. The hit policy determines *how* the table combines its matched rules — and each combination strategy maps to a primitive you already know.

### Worked Example 3.1 — Shipping Cost Table

**Business policy (prose):**

> Shipping costs depend on package weight, destination zone, and customer membership:
> - Packages up to 1 kg going to Zone A cost $5 ($3 for members).
> - Packages up to 1 kg going to Zone B cost $8 ($6 for members).
> - Packages 1–5 kg going to Zone A cost $10 ($8 for members).
> - Packages 1–5 kg going to Zone B cost $15 ($12 for members).
> - Packages over 5 kg going to Zone A cost $20 ($16 for members).
> - Packages over 5 kg going to Zone B cost $25 ($20 for members).
> - Packages over 5 kg going to Zone C require a quote (regardless of membership).

**Identifying the primitives (from Chapter 2):**

Before writing the table, decompose the policy:

| Component | Primitive |
|-----------|-----------|
| Weight brackets (≤1, 1–5, >5) | **Range Membership** |
| Zone (A, B, C) | **Enumeration Match** |
| Member true/false | **Threshold Test** (boolean) |
| Each row combines weight + zone + member → cost | **Classification** via **Conjunction** |
| Zone C → "quote" | **Conditional Mapping** (special case) |

Now the table writes itself:

**Decision table:**

| U | Weight (kg) | Zone | Member | Shipping Cost |
|---|-------------|------|--------|---------------|
| 1 | <= 1 | "A" | true | 3 |
| 2 | <= 1 | "A" | false | 5 |
| 3 | <= 1 | "B" | true | 6 |
| 4 | <= 1 | "B" | false | 8 |
| 5 | (1..5] | "A" | true | 8 |
| 6 | (1..5] | "A" | false | 10 |
| 7 | (1..5] | "B" | true | 12 |
| 8 | (1..5] | "B" | false | 15 |
| 9 | > 5 | "A" | true | 16 |
| 10 | > 5 | "A" | false | 20 |
| 11 | > 5 | "B" | true | 20 |
| 12 | > 5 | "B" | false | 25 |
| 13 | > 5 | "C" | - | null |

**Why Unique (U)?** Every combination of weight range, zone, and membership matches exactly one rule. The table is both *complete* (covers all cases) and *consistent* (no overlaps). Rule 13 uses `-` (any value) for the Member column because Zone C always requires a quote regardless.

**Why `null` for rule 13?** Returning `null` signals "no standard price available" — the calling process can then route to a manual quoting workflow.

**Completeness check:** The policy mentions 3 weight ranges × 3 zones × 2 membership states = 18 combinations. But Zone C only appears for >5 kg. Are packages ≤5 kg to Zone C covered? The prose does not say. This is exactly the kind of gap that the table makes visible — you can now go back to the business and ask.

---

## 3.2 Beyond Tables: Computed Values and Formulas

Decision tables shine when you are classifying — mapping inputs to one of several predefined outcomes. But what happens when the output is a *computed number* rather than a *classified label*? You need the **Aggregation** and **Temporal Reasoning** primitives, and they call for a different structure: the **context**.

### The Context: A Step-by-Step Computation

A context is a sequence of named values where each value can reference the ones before it. If you have used a spreadsheet column where each cell references the cells above, or a `let` chain in functional programming, you already know how this works.

### Worked Example 3.2 — Monthly Installment Calculation

**Business rule (prose):**

> The monthly installment for a fixed-rate loan is calculated using the present value of annuity formula:
>
> PMT = Amount * (Rate / 12) / (1 - (1 + Rate / 12)^(-Term * 12))
>
> Where Rate is the annual interest rate (decimal), Term is the number of years, and Amount is the loan principal.

**Identifying the primitives:** This is pure **Aggregation** — a mathematical reduction of inputs to a single number. No classification, no ranges, no enumeration. Reaching for a decision table here would be a mistake.

**As a step-by-step computation:**

```
Step 1:  Monthly Rate    = Rate / 12
Step 2:  Num Payments    = Term * 12
Step 3:  PMT             = Amount * Monthly Rate / (1 - (1 + Monthly Rate) ** (- Num Payments))
```

Three named values, each building on the previous ones — that is a context. Here is what it looks like in FEEL (preview — full syntax comes in Chapter 6):

```
{
  Monthly Rate:  Rate / 12,
  Num Payments:  Term * 12,
  PMT:           Amount * Monthly Rate / (1 - (1 + Monthly Rate) ** (- Num Payments))
}
```

The key insight: **name your intermediate values**. Do not write one massive formula. Named steps are readable, debuggable, and self-documenting.

### The Invocation: Reusable Logic

What if the same calculation shows up in multiple decisions? Both the pre-bureau and post-bureau affordability checks need the monthly installment, for instance. Do not copy-paste the formula. Package it as a **Business Knowledge Model (BKM)** — a reusable function that can be invoked with different parameters. Shared computations get factored out and invoked, not duplicated. DRY applies to decision models just as much as it applies to code.

---

## 3.3 The Decision Requirements Graph

No decision lives alone. In any real system, decisions form a dependency graph: the **Aggregation** output of one decision feeds the **Threshold Test** of the next, which feeds the **Classification** of the decision above it. The primitives from Chapter 2 stop being isolated building blocks and become a **system**.

### Worked Example 3.3 — Loan Origination DRG

The DMN specification ships with a loan origination example that is worth studying closely. Here is its structure:

```
                      ┌──────────┐
                      │ Strategy │
                      └────┬─────┘
                           │
               ┌───────────┼───────────┐
               │                       │
        ┌──────┴──────┐         ┌──────┴──────────┐
        │  Eligibility │         │ Bureau Call Type │
        └──────┬──────┘         └──────┬──────────┘
               │                       │
    ┌──────────┼──────────┐            │
    │          │          │    ┌───────┴───────────┐
    │   ┌──────┴──────┐   │   │Pre-Bureau Risk Cat│
    │   │Pre-Bureau   │   │   └───────┬───────────┘
    │   │Affordability│   │           │
    │   └──────┬──────┘   │   ┌───────┴───────────┐
    │          │          │   │App Risk Score      │
    │          │          │   └───────┬───────────┘
    │          │          │           │
    ▼          ▼          ▼           ▼
┌──────┐  ┌──────┐  ┌──────────┐  ┌──────────────┐
│Appl. │  │Req'd │  │Applicant │  │ Applicant    │
│ Data │  │ Mon  │  │  Data    │  │   Data       │
│      │  │Inst. │  │          │  │              │
└──────┘  └──────┘  └──────────┘  └──────────────┘
```

Reading this graph from bottom to top:

1. **Input Data** (ovals at the bottom): Applicant Data, Requested Product, Bureau Data.
2. **Intermediate decisions**: Application Risk Score, Pre-Bureau Risk Category, Required Monthly Installment, Pre-Bureau Affordability.
3. **Top-level decisions**: Eligibility, Bureau Call Type, Strategy.

Each arrow says: "This decision *requires* that information." And like any dependency graph worth its salt, it must be acyclic — no circular dependencies allowed.

**The primitives at work:** Application Risk Score is a **Classification** (via a C+ scorecard table — using the **Aggregation** primitive to sum partial scores). Pre-Bureau Risk Category is another **Classification** that consumes the score via a **Range Membership** test. Pre-Bureau Affordability performs **Aggregation** (the installment formula from Section 3.2) followed by a **Threshold Test** (can the applicant afford it?). Eligibility is a **Conjunction** of multiple conditions. The DRG makes these dependencies explicit — you can see which primitives feed which.

### Why This Matters

Draw the DRG *before* you write a single FEEL expression. You get four things for free:

1. **Clarity** -- you know exactly what each decision needs and produces.
2. **Testability** -- you can test each decision in isolation by feeding it mock inputs.
3. **Reusability** -- shared dependencies (like Application Risk Score) are computed once and used by multiple downstream decisions.
4. **Communication** -- the graph is readable by business stakeholders, testers, and developers alike. Everyone points at the same picture.

---

## 3.4 From Analysis to FEEL: The Bridge

You now have the analytical framework (Chapter 2) and the organisational structures (this chapter). Time to connect them. Here is the mapping table that links every business logic primitive to the FEEL constructs you will learn in Part III:

| Analytical Primitive | FEEL Construct | Chapter |
|---------------------|----------------|---------|
| Classification | Decision table with enumerated output values | 10 |
| Threshold test | Comparison operators (`>=`, `<`, etc.) or unary tests | 6 |
| Range membership | `x in [a..b]`, interval expressions, unary tests | 6, 7 |
| Enumeration match | `x in ("Gold", "Silver", "Bronze")` | 6 |
| Conjunction / Disjunction | `and` / `or` with ternary logic | 5, 6 |
| Conditional mapping | `if ... then ... else ...` | 6 |
| Aggregation | `sum()`, `count()`, `min()`, `max()`, `mean()` | 9 |
| Iteration with filter | `for ... in ... return ...`, list filters `[condition]` | 7, 8 |
| Ordering / Priority | `sort()`, Priority hit policy, `list[1]` | 9, 10 |
| Temporal reasoning | Date/time arithmetic, `duration()`, temporal comparisons | 5, 6 |

Bookmark this table. When a business requirement lands on your desk and you are not sure how to express it in FEEL, start here: identify the primitive (Chapter 2), look up the corresponding FEEL construct, and flip to the referenced chapter.

---

## 3.5 A Complete Decomposition Walkthrough

Time to put it all together. One more example, end-to-end: primitive analysis from Chapter 2 meets the structures from this chapter.

### The Policy: Insurance Premium Calculation

> **Auto Insurance Premium Rules**
>
> Base premium is determined by vehicle category:
> - Sedan: $600/year
> - SUV: $800/year
> - Sports car: $1,200/year
> - Truck: $700/year
>
> Adjustments:
> - Drivers under 25 or over 70: +30%
> - Drivers with 0 claims in last 5 years: -15%
> - Drivers with 3+ claims in last 5 years: +50%
> - Multi-car discount (2+ vehicles on same policy): -10%
> - Annual mileage over 20,000 km: +20%
>
> The final premium is the base premium multiplied by the product of all adjustment factors.

### Step 1: Identify the Primitives (Chapter 2 Analysis)

| Component | Primitive | Structure |
|-----------|-----------|-----------|
| Vehicle category → base premium | **Classification** | 4-row lookup table |
| Driver age check (under 25 or over 70) | **Disjunction** of two **threshold tests** | `Age < 25 or Age > 70` |
| Claims count in last 5 years | **Iteration with filter** + **aggregation** | Count claims where date > today - 5 years |
| Claims = 0 discount | **Threshold test** | `Claims Count = 0` |
| Claims >= 3 surcharge | **Threshold test** | `Claims Count >= 3` |
| Multi-car discount | **Threshold test** on **aggregation** | `count(Vehicles) >= 2` |
| Mileage surcharge | **Threshold test** | `Annual Mileage > 20000` |
| Final premium | **Aggregation** (product of factors) | Multiply base by all applicable factors |

### Step 2: Choose the Structures (This Chapter)

Each component gets matched to the structure that fits it best:

- **Base Premium**: Pure **Classification** → use a **decision table** (4 values in, 1 value out).
- **Age Factor, Claims Factor, Multi-Car Factor, Mileage Factor**: Each is a **Conditional Mapping** that produces a numeric multiplier → use a **context** (step-by-step computation), because the factors interact (Claims Factor depends on Claims Count, which is itself computed).
- **Final Premium**: **Aggregation** of all factors → the final entry in the context.
- **The whole thing**: A **DRG** with Base Premium as a separate decision feeding into the Premium Calculation context.

```
Final Premium
  ├── Base Premium (decision table: Vehicle Category → amount)
  ├── Age Factor (conditional: Age → factor)
  ├── Claims Factor (conditional: Claims Count → factor)
  │     └── Claims Count (aggregation: filter + count)
  ├── Multi-Car Factor (conditional: Vehicle Count → factor)
  └── Mileage Factor (conditional: Annual Mileage → factor)
```

### Step 3: Express in FEEL (Preview)

**Base Premium (decision table, hit policy U):**

| U | Vehicle Category | Base Premium |
|---|-----------------|-------------|
| 1 | "Sedan" | 600 |
| 2 | "SUV" | 800 |
| 3 | "Sports car" | 1200 |
| 4 | "Truck" | 700 |

This table is the **Classification** primitive in its purest form: one input, one output, mutually exclusive categories.

**Premium Calculation (context):**

```
{
  Base:          Base Premium,
  Age Factor:    if Driver.Age < 25 or Driver.Age > 70 then 1.30 else 1.00,
  Claims Count:  count(Driver.Claims[Date > today() - @"P5Y"]),
  Claims Factor: if Claims Count = 0 then 0.85
                 else if Claims Count >= 3 then 1.50
                 else 1.00,
  Multi Car:     if count(Policy.Vehicles) >= 2 then 0.90 else 1.00,
  Mileage:       if Driver.Annual Mileage > 20000 then 1.20 else 1.00,
  Premium:       Base * Age Factor * Claims Factor * Multi Car * Mileage
}
```

You can read this even if you have never seen a line of FEEL before. Each line maps straight to a Chapter 2 primitive:

- `Age Factor` → **Disjunction** of **Threshold Tests**, wrapped in a **Conditional Mapping**
- `Claims Count` → **Iteration with Filter** + **Aggregation**
- `Claims Factor` → chained **Threshold Tests** in a **Conditional Mapping**
- `Multi Car` → **Aggregation** + **Threshold Test** in a **Conditional Mapping**
- `Premium` → **Aggregation** (product)

Decompose first, code second. That is the pattern, and it works every time.

---

## Summary

- **Decision tables** are the go-to structure for **Classification**, **Threshold**, **Range**, and **Enumeration** primitives — any rule where conditions map to outcomes.
- **Contexts** (step-by-step computations) are the go-to structure for **Aggregation** and **Conditional Mapping** primitives — any rule where the output is computed, not classified.
- **Decision Requirements Graphs** make dependencies between decisions explicit. The primitives of one decision feed the primitives of the next. Draw the graph before writing any expressions.
- Hit policies determine how a decision table handles multiple matching rules. **Unique** (U) is the most common. **Collect Sum** (C+) embodies **Aggregation**. **Priority** (P) embodies **Ordering**.
- Every analytical primitive from Chapter 2 maps to specific FEEL constructs. The mapping table in Section 3.4 is your bridge from analysis to code.

---

## Exercises

**Exercise 3.1:** A ride-sharing app determines surge pricing based on three inputs: time of day (peak/off-peak), weather (clear/rain/snow), and current demand ratio (low/medium/high). Sketch a decision table with hit policy U. How many rules do you need? Are there any combinations that should be excluded?

**Exercise 3.2:** An e-commerce platform calculates a customer's loyalty score as the sum of: 10 points per order in the last 12 months, 5 bonus points if average order value exceeds $75, and 20 bonus points if the customer has referred at least 2 new customers. Identify the primitives, then decide: should this be a decision table, a context, or both? Write it out as a step-by-step computation.

**Exercise 3.3:** Draw a Decision Requirements Graph for the following: a university admissions decision depends on an Academic Score (computed from GPA and test scores), an Extracurricular Score (computed from activities and leadership positions), and a Financial Aid eligibility check (based on household income and family size). The final decision produces both an Admit/Reject verdict and a scholarship amount.

---

## What Comes Next

Once business rules are separated from process logic, something interesting happens: the process collapses into a simple state machine. Chapter 4 shows how FEEL contexts serve as the boundary between decisions and the processes that consume them — and why this separation transforms the way systems are built and maintained.

---

[Previous: Chapter 2: Business Rules and Business Logic](chapter-02-business-rules-and-logic.md) | [Next: Chapter 4: From Rules to Flows](chapter-04-from-rules-to-flows.md)
