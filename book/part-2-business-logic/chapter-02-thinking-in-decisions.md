# Chapter 2: Thinking in Decisions

> *"Before you can write the expression, you must understand the decision. Before you can understand the decision, you must see the logic hiding inside the business rule."*

---

This chapter contains no FEEL syntax. It is pure analysis.

The temptation when learning any language — FEEL included — is to dive straight into grammar and built-in functions. Resist that temptation. The hardest part of writing correct FEEL is not the syntax. It is understanding the business logic clearly enough that the expression writes itself.

This chapter gives you a framework for decomposing any business policy into a small set of analytical primitives. Once you can see these primitives in a policy document, an email, or a conversation with a domain expert, translating them into FEEL becomes mechanical.

---

## 2.1 The Primitives of Business Logic

Every business rule, in every domain, is composed of combinations of these ten logical building blocks:

### Primitive 1: Classification

**What it answers:** "Which category does X belong to?"

A classification maps an input value (or combination of inputs) to one of a finite set of labels.

*Examples:*
- Credit score 720 → "Good"
- Customer spending over $10,000/year → "Gold" tier
- BMI 26.3 → "Overweight"

*Key property:* The categories are mutually exclusive and (ideally) exhaustive — every input maps to exactly one label.

### Primitive 2: Threshold Test

**What it answers:** "Does X exceed / fall below Y?"

A threshold test compares a value against a fixed boundary.

*Examples:*
- Age >= 18 (legal adult)
- Account balance < 0 (overdrawn)
- Temperature > 38.5 (fever)

*Key property:* The result is always true or false (or unknown, if the input is missing).

### Primitive 3: Range Membership

**What it answers:** "Is X within bounds?"

A range test checks whether a value falls within an interval, possibly including or excluding the endpoints.

*Examples:*
- Credit score between 650 and 749 (inclusive)
- Delivery date within the next 5 business days
- Weight in (0 kg, 30 kg] — more than 0, up to and including 30

*Key property:* Ranges have start and end points, and each endpoint is either included or excluded.

### Primitive 4: Enumeration Match

**What it answers:** "Is X one of these specific values?"

An enumeration match checks membership in a finite set of allowed values.

*Examples:*
- Country in {"US", "CA", "MX"}
- Status is "Active" or "Pending"
- Product type is not "Prohibited"

*Key property:* The set of values is finite and known in advance.

### Primitive 5: Conjunction and Disjunction

**What it answers:** "Do conditions A *and* B hold?" / "Does condition A *or* B hold?"

Logical combination of simpler tests.

*Examples:*
- Applicant is employed AND income > 30,000
- Customer is Gold tier OR order amount > 500
- NOT (account is frozen)

*Key property:* Can be nested arbitrarily deep. In practice, more than 3 levels of nesting signals that a decision table would be clearer.

### Primitive 6: Conditional Mapping

**What it answers:** "If A, then X; otherwise Y."

A conditional selects between two (or more) outcomes based on a condition.

*Examples:*
- If VIP customer, apply 20% discount; otherwise 5%
- If hazardous material, require special handling; otherwise standard shipping
- If claim amount < $500, auto-approve; otherwise route to adjuster

*Key property:* Every conditional has an "else" — what happens when the condition is not met. Missing "else" branches are a common source of bugs.

### Primitive 7: Aggregation

**What it answers:** "What is the sum / count / min / max / average of a collection?"

Aggregation reduces a set of values to a single summary value.

*Examples:*
- Total order amount = sum of line item prices
- Number of overdue invoices = count of invoices where due date < today
- Highest claim amount this quarter

*Key property:* Aggregation always operates on a *collection* and produces a *single value*.

### Primitive 8: Iteration with Filter

**What it answers:** "For each item in a set that satisfies a condition, compute something."

Iteration walks through a collection, optionally filtering, and produces a new collection.

*Examples:*
- For each order line where quantity > 10, compute bulk discount
- For each employee in the Engineering department, calculate bonus
- List all invoices older than 90 days

*Key property:* The result is a new collection, not a single value (that would be aggregation applied after iteration).

### Primitive 9: Ordering and Priority

**What it answers:** "Which option ranks highest?"

Ordering sorts a set of values or selects the best according to some criterion.

*Examples:*
- Select the cheapest shipping method
- Apply the highest-priority matching discount
- Rank candidates by interview score

*Key property:* Requires a comparison function or a predefined priority list.

### Primitive 10: Temporal Reasoning

**What it answers:** "Is date/time A before/after/within a period of date/time B?"

Temporal reasoning involves dates, times, durations, and their relationships.

*Examples:*
- Insurance policy expires within 30 days
- Employee tenure is at least 2 years
- Appointment is between 09:00 and 17:00

*Key property:* Requires both point-in-time values (dates, timestamps) and durations (periods of time).

---

## 2.2 Worked Example: Decomposing Loan Eligibility

Let us take a realistic business policy and identify every primitive lurking inside it.

### The Policy (as stated by the business)

> **Loan Eligibility Rules**
>
> 1. The applicant must be at least 18 years old and no more than 70.
> 2. The applicant's employment status must be "Employed", "Self-employed", or "Retired with pension".
> 3. The applicant must not have declared bankruptcy in the last 7 years.
> 4. The risk category (Low, Medium, High, Decline) is determined by credit score:
>    - 750–850: Low
>    - 650–749: Medium
>    - 550–649: High
>    - Below 550: Decline
> 5. If the risk category is "Decline", the applicant is ineligible.
> 6. Monthly repayment must not exceed 30% of monthly income minus monthly expenses.
> 7. If the applicant is an existing customer with a Low risk category, the bureau check is waived.
> 8. All required documents must be signed.

### The Decomposition

| Rule | Primitive(s) | Analytical Structure |
|------|-------------|---------------------|
| 1. Age 18–70 | **Range membership** | `Age in [18..70]` |
| 2. Employment status | **Enumeration match** | `Status in {"Employed", "Self-employed", "Retired with pension"}` |
| 3. No recent bankruptcy | **Temporal reasoning** + **Threshold test** | `today() - Bankruptcy Date > 7 years` (or bankruptcy date is absent) |
| 4. Risk from credit score | **Classification** (via ranges) | Four range memberships combined into a classification table |
| 5. Decline = ineligible | **Threshold test** (on a classified value) | `Risk != "Decline"` |
| 6. Affordability | **Aggregation** + **Threshold test** | Compute: `(Income - Expenses) * 0.30 >= Repayment` |
| 7. Waive bureau check | **Conjunction** + **Conditional mapping** | `if Existing Customer and Risk = "Low" then waive else require` |
| 8. All documents signed | **Iteration with filter** (universal quantifier) | `every doc in Documents satisfies doc.Signed = true` |

Notice that even a short policy of 8 rules uses 8 of our 10 primitives. This is typical. Business rules are *combinatorial* — they combine simple building blocks in ways that can become complex quickly.

The value of this decomposition is that each primitive maps directly to a FEEL construct. When you reach Part III, you will learn each construct and be able to trace it back to the analytical primitive it serves.

---

## 2.3 Decision Tables: The Workhorse

Many business rules are best expressed as tables. A decision table is the most natural representation when:

- The decision has a small number of input dimensions (2–5).
- The output is one of a finite set of values (classification) or a computed value.
- There are multiple rules that can be listed and verified row by row.
- Business stakeholders need to review and approve the rules.

### Anatomy of a Decision Table

A decision table has these components:

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

### Hit Policies

The hit policy determines what happens when the inputs match zero, one, or more than one rule. This is one of the most important concepts in DMN.

#### Single-Hit Policies

These policies produce a single output value:

| Policy | Letter | Meaning | When to use |
|--------|--------|---------|------------|
| **Unique** | U | Exactly one rule matches. If zero or more than one match, it is an error in the table design. | When rules are designed to be mutually exclusive. The most common policy. |
| **Any** | A | Multiple rules may match, but they must all produce the same output. | When you have redundant rules for readability. |
| **Priority** | P | Multiple rules may match with different outputs. The output with the highest priority (as defined by the output values list) wins. | When rules overlap and you want the "most important" result. |
| **First** | F | Multiple rules may match. The first match (by row order) wins. | When rules are ordered by specificity — like `if/else if/else`. |

#### Multiple-Hit Policies

These policies produce a list of output values:

| Policy | Letter | Meaning | When to use |
|--------|--------|---------|------------|
| **Collect** | C | All matching rules fire. Outputs are collected into a list (arbitrary order). | When you need all applicable results (e.g., all fees that apply). |
| **Collect Sum** | C+ | All matching rules fire. Outputs are summed. | Scorecards: partial scores summed into a total. |
| **Collect Count** | C# | All matching rules fire. Returns the count of matches. | "How many rules apply?" |
| **Collect Min** | C< | All matching rules fire. Returns the minimum output. | "What is the lowest applicable rate?" |
| **Collect Max** | C> | All matching rules fire. Returns the maximum output. | "What is the highest applicable fee?" |
| **Rule Order** | R | All matching rules fire. Outputs returned in rule order. | When the order of rules matters for downstream processing. |
| **Output Order** | O | All matching rules fire. Outputs returned in the order defined by output values. | When you need a canonical ordering. |

### Worked Example 2.2 — Shipping Cost Table

**Business policy (prose):**

> Shipping costs depend on package weight, destination zone, and customer membership:
> - Packages up to 1 kg going to Zone A cost $5 ($3 for members).
> - Packages up to 1 kg going to Zone B cost $8 ($6 for members).
> - Packages 1–5 kg going to Zone A cost $10 ($8 for members).
> - Packages 1–5 kg going to Zone B cost $15 ($12 for members).
> - Packages over 5 kg going to Zone A cost $20 ($16 for members).
> - Packages over 5 kg going to Zone B cost $25 ($20 for members).
> - Packages over 5 kg going to Zone C require a quote (regardless of membership).

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

---

## 2.4 Beyond Tables: Computed Values and Formulas

Not all business logic fits in a table. When the output is a *computed value* rather than a *classified label*, you need formulas.

### The Context: A Step-by-Step Computation

A context is a sequence of named values, where each value can reference the ones before it. Think of it as a spreadsheet column, or as a `let` chain in functional programming.

### Worked Example 2.3 — Monthly Installment Calculation

**Business rule (prose):**

> The monthly installment for a fixed-rate loan is calculated using the present value of annuity formula:
>
> PMT = Amount * (Rate / 12) / (1 - (1 + Rate / 12)^(-Term * 12))
>
> Where Rate is the annual interest rate (decimal), Term is the number of years, and Amount is the loan principal.

**As a step-by-step computation:**

```
Step 1:  Monthly Rate    = Rate / 12
Step 2:  Num Payments    = Term * 12
Step 3:  PMT             = Amount * Monthly Rate / (1 - (1 + Monthly Rate) ** (- Num Payments))
```

This is a context: three named values, each building on the previous ones. In FEEL (preview — we will cover the syntax in Chapter 5):

```
{
  Monthly Rate:  Rate / 12,
  Num Payments:  Term * 12,
  PMT:           Amount * Monthly Rate / (1 - (1 + Monthly Rate) ** (- Num Payments))
}
```

The key insight: **name your intermediate values**. Do not write one massive formula. Named steps are readable, debuggable, and self-documenting.

### The Invocation: Reusable Logic

When the same calculation is used in multiple decisions (e.g., both the pre-bureau and post-bureau affordability checks need the monthly installment), package it as a **Business Knowledge Model (BKM)** — a reusable function that can be invoked with different parameters.

---

## 2.5 The Decision Requirements Graph

Individual decisions do not exist in isolation. They form a dependency graph.

### Worked Example 2.4 — Loan Origination DRG

The DMN specification includes a comprehensive loan origination example. Here is its structure:

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

Each arrow says: "This decision *requires* that information." The graph must be acyclic — no circular dependencies.

### Why This Matters

Drawing the DRG *before* writing any FEEL expressions gives you:

1. **Clarity**: You know exactly what each decision needs and produces.
2. **Testability**: You can test each decision in isolation by providing its direct inputs.
3. **Reusability**: Shared dependencies (like Application Risk Score) are computed once and used by multiple downstream decisions.
4. **Communication**: The graph is readable by business stakeholders, testers, and developers alike.

---

## 2.6 From Analysis to FEEL: The Bridge

We have now established the analytical framework. Here is the mapping table that connects every business logic primitive to the FEEL constructs you will learn in Part III:

| Analytical Primitive | FEEL Construct | Chapter |
|---------------------|----------------|---------|
| Classification | Decision table with enumerated output values | 8 |
| Threshold test | Comparison operators (`>=`, `<`, etc.) or unary tests | 4 |
| Range membership | `x in [a..b]`, interval expressions, unary tests | 4, 5 |
| Enumeration match | `x in ("Gold", "Silver", "Bronze")` | 4 |
| Conjunction / Disjunction | `and` / `or` with ternary logic | 3, 4 |
| Conditional mapping | `if ... then ... else ...` | 4 |
| Aggregation | `sum()`, `count()`, `min()`, `max()`, `mean()` | 7 |
| Iteration with filter | `for ... in ... return ...`, list filters `[condition]` | 5, 6 |
| Ordering / Priority | `sort()`, Priority hit policy, `list[1]` | 7, 8 |
| Temporal reasoning | Date/time arithmetic, `duration()`, temporal comparisons | 3, 4 |

Keep this table bookmarked. When you encounter a business requirement and are unsure how to express it in FEEL, start here: identify the primitive, look up the corresponding FEEL construct, and turn to the referenced chapter.

---

## 2.7 A Complete Decomposition Walkthrough

Let us work through one more example end-to-end, combining multiple primitives.

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

### Step 1: Identify the Primitives

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

### Step 2: Structure as Decisions

```
Final Premium
  ├── Base Premium (decision table: Vehicle Category → amount)
  ├── Age Factor (conditional: Age → factor)
  ├── Claims Factor (conditional: Claims Count → factor)
  │     └── Claims Count (aggregation: filter + count)
  ├── Multi-Car Factor (conditional: Vehicle Count → factor)
  └── Mileage Factor (conditional: Annual Mileage → factor)
```

### Step 3: Preview the FEEL (Full Details in Later Chapters)

**Base Premium (decision table, hit policy U):**

| U | Vehicle Category | Base Premium |
|---|-----------------|-------------|
| 1 | "Sedan" | 600 |
| 2 | "SUV" | 800 |
| 3 | "Sports car" | 1200 |
| 4 | "Truck" | 700 |

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

Even without knowing FEEL syntax, you can read this. That is the power of decomposing first and coding second.

---

## Summary

- Every business rule is built from a small set of analytical primitives: classification, threshold, range, enumeration, conjunction/disjunction, conditional, aggregation, iteration, ordering, and temporal reasoning.
- Decision tables are the workhorse for classification-style rules. Hit policies determine what happens when multiple rules match.
- Computed values and formulas use contexts — step-by-step named calculations.
- Decision Requirements Graphs show how decisions depend on each other. Draw the graph before writing any expressions.
- Each analytical primitive maps to specific FEEL constructs. The mapping table in Section 2.6 is your bridge from analysis to code.

---

## Exercises

**Exercise 2.1:** A hospital triage policy states: "Patients with chest pain are Priority 1. Patients with breathing difficulty are Priority 1. Patients with fractures are Priority 2. All others are Priority 3. If the patient is under 5 or over 80, increase priority by one level (Priority 3 becomes 2, Priority 2 becomes 1, Priority 1 stays 1)." Identify every analytical primitive in this policy.

**Exercise 2.2:** An airline's upgrade policy states: "If a passenger is a Platinum frequent flyer, offer a complimentary upgrade to the next available cabin class. If Gold, offer upgrade at 50% off. If Silver, offer upgrade at 25% off. Upgrades are only available if the higher cabin has at least 3 empty seats." Decompose this into primitives and sketch a decision table.

**Exercise 2.3:** A payroll rule states: "Overtime is paid at 1.5x the hourly rate for hours worked beyond 40 per week, and at 2x for hours beyond 60. Weekend hours always count as overtime." Identify the primitives and sketch the computation as named steps.

---

## What Comes Next

In Chapter 3, we begin learning FEEL itself — starting with the atoms: numbers, strings, booleans, dates, times, durations, and the ever-present `null`.
