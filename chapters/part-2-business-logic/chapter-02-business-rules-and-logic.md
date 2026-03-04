# Chapter 2: Business Rules and Business Logic

> *"Every organisation is, at its core, a collection of rules. The rules determine who gets what, when, and why. Most of these rules have never been written down."*

---

## 2.1 What Is a Business Rule?

A business rule is a statement that constrains, computes, or classifies some aspect of how an organisation operates. It is not a software requirement. It is not a feature. It is a fact about the business.

Here are five business rules, drawn from five different domains:

1. **"A customer qualifies for free shipping if their order exceeds $100 and they hold Gold membership."** (E-commerce)
2. **"A patient presenting with chest pain must be triaged as Priority 1."** (Healthcare)
3. **"An insurance claim above $10,000 must be reviewed by a senior adjuster."** (Insurance)
4. **"Overtime is paid at 1.5× the hourly rate for hours worked beyond 40 per week."** (Payroll)
5. **"A loan application is declined if the applicant's credit score is below 550."** (Banking)

Every one of these statements existed before any software was written to implement it. They come from company policies, industry regulations, operational handbooks, pricing committees, and — in far too many cases — the head of the person who has worked there the longest.

### Rules Are Not Code

A critical distinction: the rule *"Gold customers get 15% off orders over $50"* is a business fact. This Java code is an **implementation** of that fact:

```java
if (customer.getTier().equals("Gold") && order.getTotal() > 50) {
    discount = 0.15;
}
```

The rule and the code look similar, but they are fundamentally different things. The rule belongs to the business. The code belongs to the codebase. When the business changes the rule — say, from 15% to 20% — someone has to find the code, understand it, modify it, test it, and deploy it. If the rule exists in three different services, someone has to find and update all three.

This gap between the rule (what the business wants) and the code (what the software does) is where errors, inconsistencies, and delays are born.

### Rules Have Structure

Business rules are not random sentences. They follow predictable patterns. Consider:

- **"If X, then Y."** — A conditional.
- **"X must be between A and B."** — A range test.
- **"X must be one of {A, B, C}."** — An enumeration.
- **"The total is the sum of all line items."** — An aggregation.
- **"Classify X into category Y based on criteria Z."** — A classification.

These patterns repeat across every industry, every organisation, every domain. The specifics change — credit scores versus patient symptoms versus shipping weights — but the *shape* of the logic is the same. This is not a coincidence. It is the key insight that makes FEEL possible.

---

## 2.2 What Is Business Logic?

A single business rule is a sentence. **Business logic** is the chapter.

Business logic is the complete, interconnected set of rules that govern how an organisation makes a specific decision. It includes:

- The individual rules themselves.
- The **order** in which they must be evaluated (you cannot check affordability before you know the monthly repayment amount).
- The **interactions** between rules (a Gold customer who is also a high-risk borrower — which rule takes precedence?).
- The **edge cases** that the original policy statement did not address (what happens if the credit score is missing? What if the customer is exactly 18 years old?).
- The **defaults** that apply when no rule matches (if no shipping tier is specified, use Standard).

### Business Logic Is Combinatorial

A single rule with 2 conditions and 3 possible outcomes has 6 combinations. A decision with 5 rules, each with 2–4 conditions, can have hundreds or thousands of valid input combinations. This combinatorial explosion is why business logic is hard — not because any individual rule is complicated, but because the rules *interact*.

Consider a simple pricing decision with three inputs:

| Input | Possible Values |
|-------|----------------|
| Customer tier | Gold, Silver, Bronze |
| Order amount | ≤ $50, $50–$200, > $200 |
| Promo code active | Yes, No |

That is 3 × 3 × 2 = **18 combinations**. Each combination may produce a different discount. A developer writing `if/else` chains has to get all 18 right — and know which ones they have not covered.

A decision table makes all 18 combinations visible in a single structure. This is one reason why expressing business logic in a dedicated notation (rather than general-purpose code) is so powerful: the notation *forces* completeness.

### Business Logic vs. Application Logic

Not all code is business logic. It is worth drawing the boundary clearly:

| Business Logic | Application Logic |
|---------------|-------------------|
| "Gold customers get 15% off" | "Render the discount badge in the UI" |
| "Claims over $10,000 need review" | "Send a Kafka message to the review queue" |
| "Overtime kicks in after 40 hours" | "Store the timesheet in PostgreSQL" |
| "Applicants under 18 are ineligible" | "Return HTTP 422 with a validation error" |

Business logic is **what** the organisation decides. Application logic is **how** the software acts on that decision. This book is about the "what" — and about a language (FEEL) designed specifically to express it.

---

## 2.3 Where Business Rules Hide

If business rules are so important, you might expect them to be neatly documented and carefully managed. In practice, they are scattered across the organisation in ways that make them almost invisible.

### In Code

The most common hiding place. Business rules live as `if/else` blocks, `switch` statements, magic numbers, and anonymous functions buried inside application code. They are interleaved with database calls, UI rendering, error handling, and logging. A developer reading the code can see *that* a rule exists, but often cannot tell *where it came from* or *who owns it*.

```python
# Why 0.15? Who decided this? When does it change?
if customer.tier == "Gold" and order.total > 50:
    discount = 0.15
```

### In Spreadsheets

Business analysts often maintain rules in Excel spreadsheets. A pricing sheet, a risk scoring matrix, a commission table. These spreadsheets are remarkably accurate — the analyst who maintains them usually knows the rules intimately. But they are disconnected from the code. A developer translates the spreadsheet into code manually, and from that moment the two copies diverge.

### In People's Heads

The most dangerous hiding place. "Ask Sarah, she knows how the eligibility rules work." This is **tribal knowledge** — rules that exist only in the memory of specific individuals. When those individuals leave, go on holiday, or simply forget, the rules leave with them.

### In Documents

Policy manuals, regulatory filings, contract terms, Board minutes. The rules are written in natural language, often ambiguously ("applicants should generally meet the minimum criteria"). The gap between the document and the code is bridged by interpretation — and different developers interpret differently.

### In Configuration

Feature flags, property files, environment variables, database lookup tables. Rules are encoded as data — which is better than hardcoding them, but still lacks structure, documentation, and testability. What does `MAX_CLAIM_AUTO_APPROVE=5000` mean? Is it inclusive? What currency? Who set it?

### The Scatter Problem

The real damage is not that rules hide in any one of these places. It is that **the same rule hides in multiple places simultaneously**, and the copies disagree.

The pricing rule is in the Java service. It is also in the mobile app (JavaScript). It is also in Sarah's spreadsheet. It is also in the policy document from 2019. The Java service says 15%. The mobile app says 10%. The spreadsheet says 20%. The policy document says "competitive rates at management's discretion." Which one is correct?

When rules are scattered, there is no single source of truth. Changes propagate inconsistently. Testing is incomplete because nobody knows all the places a rule lives. Auditors cannot answer "why was this customer charged $47.50?" because the answer is spread across four systems and a retirement spreadsheet.

---

## 2.4 The Hidden Value of Business Rules

Most organisations treat business rules as an implementation detail — something to be coded and forgotten. This is a mistake. Business rules are among the most valuable intellectual assets an organisation possesses.

### Rules ARE the Business

Consider what a bank actually *is*. Strip away the marble lobby, the website, the mobile app, the core banking platform. What remains? **Rules.** Rules about who qualifies for a loan. Rules about how interest is calculated. Rules about when a transaction is flagged as suspicious. Rules about capital reserves and regulatory compliance.

The technology is replaceable. The brand is important but fungible. The rules — refined over decades of regulatory negotiation, competitive learning, and operational experience — are the core intellectual property. An insurance company's underwriting rules represent centuries of accumulated actuarial wisdom. A retailer's pricing rules encode hard-won knowledge about customer behaviour.

When an organisation says "we are migrating to a new platform," the rules are the hardest part to migrate — because they are the part that nobody fully understands.

### Rules Encode Institutional Memory

When a developer writes `if (age < 25 || age > 70) surcharge *= 1.30`, that number — 1.30 — encodes a decision made by an actuary based on claims data spanning 20 years. The `if` condition encodes a regulatory boundary established by the industry regulator. The variable name `surcharge` encodes a pricing model chosen by the product team in 2018.

All of this context is lost in the code. The rule becomes a magic number inside a conditional, comprehensible only to the person who wrote it — and only for as long as they remember why.

When business rules are expressed in a dedicated notation — like a decision table with named inputs, defined ranges, and documented output values — the context is preserved. The rule is self-documenting. It carries its own explanation.

### Rules Change Faster Than Anything Else

Application code changes with feature releases — maybe quarterly. Infrastructure changes with platform migrations — maybe yearly. Business rules change with the market — weekly, daily, sometimes hourly.

- A competitor drops their price. Your pricing rules need to respond.
- A regulator issues new guidance. Your compliance rules must be updated.
- A product manager launches a promotion. Your discount rules change for the weekend.
- An underwriter reviews loss ratios. Your risk scoring rules are adjusted.

If every rule change requires a code change, a test cycle, and a deployment, the organisation cannot respond at the speed the business demands. This is why separating business rules from application code is not just an architectural preference — it is an operational necessity.

### Rules Are the Most Duplicated Code

In most enterprise codebases, the same business rule is implemented in multiple places:

- The main application server.
- The batch processing job.
- The mobile app.
- The reporting system.
- The data warehouse ETL pipeline.

Each copy was written by a different developer, at a different time, with a different understanding of the rule. They are all slightly different. They all need to be updated when the rule changes. They almost never are.

A study by the Business Rules Group estimates that in a typical enterprise application, **40% or more of the code** is devoted to implementing business rules. And most of it is duplicated.

### Rules Are Where Bugs Live

When software gets the business logic wrong, the consequences are directly felt by the customer:

- Charged the wrong price.
- Denied a claim they should have received.
- Approved a loan they should not have been offered.
- Missed a compliance deadline.

These are not technical bugs (null pointer exceptions, timeout errors). They are **logic bugs** — the code does exactly what it was told to do, but what it was told to do was wrong. Logic bugs are the hardest to find (the system does not crash), the most expensive to fix (they often require business expert review), and the most damaging to the organisation (regulatory fines, customer churn, reputational harm).

---

## 2.5 The Primitives of Business Logic

Here is the central claim of this chapter: **every business rule, in every domain, is composed of combinations of the same ten logical building blocks.** These are the primitives of business logic.

This is not an abstraction for academics. It is a practical tool. Once you can see these primitives in a policy document, an email, or a conversation with a domain expert, translating them into executable expressions becomes mechanical.

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

## 2.6 Worked Example: Decomposing Loan Eligibility

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

---

## 2.7 Why This Decomposition Matters

The ten primitives are not just an analytical exercise. They unlock three practical capabilities:

### 1. Conversations With Domain Experts Become Structured

When a business analyst says "we need to add a new eligibility rule," you can now ask targeted questions:

- "Is this a **classification** (mapping inputs to categories) or a **threshold test** (yes/no check)?"
- "What are the **ranges**? Are the boundaries inclusive or exclusive?"
- "Is this an **and** condition (all must be true) or an **or** condition (any suffices)?"
- "What happens in the **else** case? What if the data is **missing**?"

These questions are not about technology. They are about the logic itself. They uncover ambiguities that would otherwise surface as bugs in production.

### 2. The Path to Code Becomes Mechanical

Each primitive maps directly to a FEEL construct. You will learn every mapping in Chapter 3. But the principle is simple: once you have identified the primitives in a business rule, the expression almost writes itself. The hard part is the analysis, not the syntax.

### 3. Completeness Becomes Verifiable

When you know a rule is a **classification** with 4 categories, you can verify that all 4 categories are covered. When you know a rule involves **range membership**, you can check that the ranges are contiguous and non-overlapping. When you know a rule is a **conjunction** of 3 conditions, you can enumerate the 2³ = 8 possible combinations and verify that each one has a defined outcome.

This is why decision tables are so powerful — they make the combinatorial structure visible and checkable. We will explore this in detail in Chapter 3.

---

## Summary

- A **business rule** is a statement that constrains, computes, or classifies some aspect of how an organisation operates. It is a business fact, not a piece of code.
- **Business logic** is the complete, interconnected set of rules governing a specific decision. It is combinatorial: simple rules interact to produce complex behaviour.
- Business rules hide in code, spreadsheets, people's heads, documents, and configuration. The real problem is that the same rule hides in multiple places, and the copies disagree.
- Business rules are among the most valuable assets an organisation possesses: they encode institutional memory, change faster than anything else, and are where logic bugs live.
- Every business rule is built from **ten analytical primitives**: classification, threshold test, range membership, enumeration match, conjunction/disjunction, conditional mapping, aggregation, iteration with filter, ordering/priority, and temporal reasoning.
- Decomposing rules into primitives structures conversations with domain experts, makes the path to code mechanical, and makes completeness verifiable.

---

## Exercises

**Exercise 2.1:** A hospital triage policy states: "Patients with chest pain are Priority 1. Patients with breathing difficulty are Priority 1. Patients with fractures are Priority 2. All others are Priority 3. If the patient is under 5 or over 80, increase priority by one level (Priority 3 becomes 2, Priority 2 becomes 1, Priority 1 stays 1)." Identify every analytical primitive in this policy.

**Exercise 2.2:** An airline's upgrade policy states: "If a passenger is a Platinum frequent flyer, offer a complimentary upgrade to the next available cabin class. If Gold, offer upgrade at 50% off. If Silver, offer upgrade at 25% off. Upgrades are only available if the higher cabin has at least 3 empty seats." Identify every primitive (do not sketch a decision table yet — that comes in Chapter 3).

**Exercise 2.3:** A payroll rule states: "Overtime is paid at 1.5x the hourly rate for hours worked beyond 40 per week, and at 2x for hours beyond 60. Weekend hours always count as overtime." Identify the primitives and write out the rules as structured English, labelling each primitive.

---

## What Comes Next

Now that you understand what business rules are, where they hide, and why they matter, Chapter 3 shows you the **structures** for organising them: decision tables, step-by-step computations (contexts), dependency graphs, and the mapping from each analytical primitive to its FEEL construct. You will see how the primitives from this chapter become executable decisions.

---

[Previous: Chapter 1: The World Before FEEL](../part-1-foundations/chapter-01-the-world-before-feel.md) | [Next: Chapter 3: Thinking in Decisions](chapter-03-thinking-in-decisions.md)
