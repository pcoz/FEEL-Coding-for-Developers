# Chapter 2: Business Rules and Business Logic

> *"The code did exactly what it was told to do. Unfortunately, what it was told to do was wrong."*

---

What exactly is a business rule? Where do they come from? Why does every organisation have thousands of them? And why is getting one wrong — even slightly — so expensive?

This chapter answers those questions. It is the foundation for everything that follows: before you can write a FEEL expression, you need to understand what you are expressing and why it matters.

---

## 2.1 You Already Know What a Business Rule Is

You encounter business rules every single day. You just do not call them that.

When you buy a coffee with a loyalty card and the cashier says "your tenth coffee is free" — that is a business rule. When your bank texts you "unusual transaction detected, card frozen" — a business rule made that decision. When your insurance company pays 80% of a medical bill and leaves you with 20% — business rules determined those percentages, the eligible procedures, the deductible, and the co-pay. When you get paid on Friday and notice overtime at time-and-a-half — business rules calculated every line on that payslip.

A business rule is a **specific, actionable statement about how an organisation operates**. Not a vague aspiration. Not a strategy document. A concrete rule that determines an outcome:

- **"A customer qualifies for free shipping if their order exceeds $100 and they hold Gold membership."**
- **"A patient presenting with chest pain must be triaged as Priority 1."**
- **"Overtime is paid at 1.5× the hourly rate for hours worked beyond 40 per week."**
- **"A loan application is declined if the applicant's credit score is below 550."**
- **"A claim submitted more than 90 days after the incident is automatically rejected."**

These statements are not code. They are not feature requests. They are not user stories. They are facts about how the business works. Every one of them existed before any software was written to implement it. They come from legislation, industry regulations, actuarial tables, competitive strategy, pricing committees, operational experience, and — critically — from decades of learning what goes wrong when you get them wrong.

---

## 2.2 Why Business Rules Exist

Business rules exist because organisations need to make **consistent decisions at scale**.

A sole trader with three customers can keep the rules in their head. A bank with ten million customers cannot. When a thousand loan applications arrive every day, you cannot have a thousand different loan officers each applying their own personal interpretation of "creditworthy." You need rules. Written down. Unambiguous. Applied the same way every time.

### Consistency

The same input must produce the same output, regardless of who processes it, what time of day it is, or which branch office handles it. A 35-year-old with a credit score of 720 and a stable income must get the same loan decision whether they apply in London or Leeds, on Monday or Friday, via the website or the branch. Business rules make this possible.

### Fairness

Rules protect against bias and favouritism. When the eligibility criteria are explicit — "income at least $30,000 AND employment tenure at least 2 years AND no bankruptcy in the last 7 years" — there is no room for a loan officer to approve their cousin's application while rejecting an equally qualified stranger. The rule applies to everyone.

### Compliance

Regulators do not ask "what does your software do?" They ask "what are your rules?" A bank must demonstrate that its lending rules comply with fair lending laws. An insurer must prove that its claims rules follow the policy contract. A hospital must show that its triage rules meet clinical guidelines. If the rules are buried in code, compliance becomes an archaeology project. If they are expressed in a readable notation, compliance is a document review.

### Speed

A human underwriter can assess perhaps 20 loan applications per day. A set of business rules encoded in a decision engine can assess 20,000 per second. The rules do not get tired, do not have bad days, and do not take lunch breaks. But they also do not exercise judgement — which is why getting the rules right is so critical.

### Auditability

When a customer calls and asks "why was my claim denied?", the organisation needs an answer. Not "the system said no." A *reason*. "Your claim was denied because the incident date was more than 90 days before the submission date, which exceeds the maximum filing period defined in section 4.2 of your policy." Business rules, when properly expressed, are their own audit trail.

---

## 2.3 What Business Rules Represent

This is the part that most developers miss entirely.

When you write `if (age < 25 || age > 70) surcharge *= 1.30` in a codebase, you are not writing a bit of logic. You are encoding:

- **An actuarial finding** based on 20 years of claims data showing that drivers under 25 and over 70 have statistically higher accident rates.
- **A regulatory constraint** — the age boundaries were negotiated with the industry regulator, who mandated that age-based pricing must be justified by statistical evidence.
- **A competitive decision** — the 30% surcharge was chosen because it is high enough to cover the additional risk but low enough to remain competitive with rival insurers.
- **An operational lesson** — the company tried a 50% surcharge in 2015 and lost 40% of their young-driver market share. They learned. They adjusted.

All of that — decades of data, regulatory negotiation, competitive intelligence, and hard-won operational experience — is compressed into one line of code. A line that a junior developer might "refactor" out of existence during a code cleanup because it "looks like a magic number."

**Business rules are the encoded knowledge of the organisation.** They are the DNA of how the business operates. They represent:

| What the rule looks like | What it actually represents |
|--------------------------|---------------------------|
| `if credit_score < 550 then "Decline"` | A risk boundary determined by statistical modelling of 10 years of default data |
| `if claim_amount > 10000 then "Manual Review"` | A fraud-prevention threshold calibrated by the loss-prevention team |
| `discount = 0.15` for Gold customers | A customer retention strategy developed by the marketing department and approved by the CFO |
| `if days_since_incident > 90 then "Reject"` | A contractual obligation defined in the insurance policy wording, reviewed by legal |
| `overtime_rate = base_rate * 1.5` | A legal requirement under the Fair Labor Standards Act (or equivalent national legislation) |

Every one of these rules is **someone's job**. An actuary set the credit score threshold. A fraud analyst set the claim review threshold. A marketing manager set the discount rate. A lawyer set the filing deadline. A legislator set the overtime multiplier. These people do not read code. They should not have to. But their rules run the business — and when the rules are wrong, the business is wrong.

---

## 2.4 What Happens When You Get Business Rules Wrong

Getting a business rule wrong is not like getting a UI colour wrong. It is not a cosmetic issue. It is not a minor inconvenience. When a business rule is wrong, **real people suffer real consequences**.

### The Customer Pays the Price

- A health insurer's rule engine misclassifies a procedure as "elective" instead of "medically necessary." A cancer patient is denied coverage. They either pay out of pocket, delay treatment, or both.
- A bank's lending rule has an off-by-one error in the age range. Applicants who are exactly 18 are rejected. They cannot get a student loan. They do not know why.
- An e-commerce platform's discount rule fires twice due to a race condition. A $500 TV is sold for $50. The company eats the loss — or, worse, cancels the orders and loses customer trust.

### The Organisation Pays the Price

- A payroll rule underpays overtime for six months. When discovered, the company owes back pay, interest, and penalties to 3,000 employees. The finance team spends weeks recalculating. The legal team negotiates with the labour board. The HR team handles the employee relations fallout.
- A compliance rule fails to flag suspicious transactions above the regulatory threshold. The regulator imposes a fine. The remediation project takes two years.
- An insurance company's underwriting rules are too permissive. They approve too many high-risk policies. Claims exceed premiums. The loss ratio climbs. The company's credit rating is downgraded.

### Why Logic Bugs Are the Worst Kind of Bug

When code throws a NullPointerException, the system crashes and you get a stack trace. When code has a logic bug in a business rule, the system does not crash. It runs perfectly. It produces an output. The output is wrong. But nobody notices — because the system did not complain.

Logic bugs are:

- **Silent.** No error. No log. No alert. The system did exactly what it was told to do.
- **Invisible.** The wrong discount was applied. The wrong risk category was assigned. The output looks plausible. It just happens to be incorrect.
- **Cumulative.** A pricing rule that is 2% off does not cause a single dramatic failure. It quietly costs the company 2% of revenue, every day, for months, until someone in finance notices the margin erosion.
- **Expensive to find.** You cannot grep for them. You cannot write a unit test that catches "the business intent was X but the code does Y" — because the test was written by the same developer who misunderstood the rule.
- **Expensive to fix.** Fixing a logic bug requires understanding the business intent, which requires talking to the business expert, who may have left the company.

This is why business rules matter. Not because they are intellectually interesting (although they are). Because **when they are wrong, people get hurt, money is lost, and organisations are damaged**.

---

## 2.5 Where You Encounter Business Rules

Business rules are not confined to any single part of a system. They arise wherever the organisation needs to make a decision:

### Pricing and Discounts

Every organisation that sells something has pricing rules. Base prices, volume discounts, loyalty discounts, promotional pricing, bundle pricing, surge pricing, time-of-day pricing. A single e-commerce site might have hundreds of pricing rules, interacting in complex ways. "Gold customers get 15% off, but not on sale items, unless the order exceeds $200, in which case the discount applies to everything except clearance items." That is four rules in one sentence.

### Eligibility and Qualification

Can this customer open an account? Is this patient eligible for this treatment? Does this applicant qualify for this loan? Does this employee qualify for this benefit? Eligibility rules are gatekeepers — they produce a yes/no decision, but the logic behind that decision can be extraordinarily complex.

### Risk Assessment and Scoring

Insurance underwriting, credit scoring, fraud detection, anti-money laundering. These rules assign a risk level to an entity based on multiple factors. They are often expressed as scorecards — weighted combinations of individual risk indicators that produce an aggregate score.

### Routing and Assignment

Which department handles this claim? Which warehouse fulfils this order? Which support tier handles this ticket? Which adjuster reviews this case? Routing rules direct work to the right place based on attributes of the work item.

### Compliance and Validation

Is this transaction within regulatory limits? Does this product labelling meet the requirements? Is this employee's working time within legal limits? Has the mandatory cooling-off period elapsed? Compliance rules are non-negotiable — they come from legislation, regulation, or contract.

### Calculation and Computation

Tax calculation, interest accrual, premium computation, commission calculation, penalty assessment. These are not classifications or yes/no decisions — they are formulas. But the formulas embed business decisions: *which* tax rate applies, *how* interest compounds, *when* penalties kick in.

### Every Domain, Every Industry

| Domain | Example Rules |
|--------|---------------|
| **Healthcare** | Triage priority, drug interaction checks, treatment eligibility, insurance pre-authorisation |
| **Banking** | Loan eligibility, interest rate determination, transaction fraud scoring, regulatory capital calculation |
| **Insurance** | Underwriting acceptance, premium calculation, claims adjudication, policy renewal |
| **Retail** | Pricing, promotions, shipping tier, return eligibility, loyalty points |
| **Logistics** | Route optimisation, carrier selection, customs classification, hazmat handling |
| **HR / Payroll** | Overtime calculation, leave entitlement, benefits eligibility, tax withholding |
| **Telecommunications** | Plan eligibility, usage rating, roaming charges, service throttling |
| **Government** | Tax assessment, benefit eligibility, permit approval, regulatory compliance |

Business rules are not a niche concern. They are the operational core of every organisation. They are also — and this is the uncomfortable truth — the least well-managed part of most software systems.

---

## 2.6 Where Business Rules Hide

If business rules are so critical, you would expect them to be carefully documented, version-controlled, and centrally managed. In practice, they are scattered in ways that make them nearly invisible.

### In Code

The most common hiding place. Business rules live as `if/else` blocks, `switch` statements, magic numbers, and anonymous functions buried inside application code:

```python
# Why 0.15? Who decided this? When does it change? Who approved it?
if customer.tier == "Gold" and order.total > 50:
    discount = 0.15
```

The rule is there, but it is entangled with database calls, UI rendering, error handling, and logging. A developer can see *that* a rule exists, but cannot tell *who owns it*, *why it has that value*, or *when it last changed*.

### In Spreadsheets

Business analysts maintain rules in Excel — pricing sheets, scoring matrices, commission tables. These spreadsheets are often remarkably accurate. But they are disconnected from the code. A developer translates the spreadsheet into Java or Python, and from that moment the two copies diverge. The analyst updates the spreadsheet. The developer does not know. The code is now wrong.

### In People's Heads

The most dangerous hiding place. "Ask Sarah, she knows how the eligibility rules work." This is **tribal knowledge** — rules that exist only in the memory of specific individuals. When Sarah retires, goes on holiday, or simply forgets, the rules go with her.

### In Documents

Policy manuals, regulatory filings, contract terms, Board minutes. The rules are written in natural language, often ambiguously: "applicants should generally meet the minimum criteria." Different developers read this differently. Different interpretations produce different code.

### In Configuration

Feature flags, property files, environment variables, database lookup tables. `MAX_CLAIM_AUTO_APPROVE=5000`. Is that inclusive? What currency? Does it apply to all claim types? Who set it? When? Why?

### The Scatter Problem

The deepest damage is not that rules hide in any one of these places. It is that **the same rule exists in multiple places simultaneously, and the copies disagree**.

The pricing rule is in the Java service. It is also in the mobile app (JavaScript). It is also in Sarah's spreadsheet. It is also in the policy document from 2019. The Java service says 15%. The mobile app says 10%. The spreadsheet says 20%. The policy document says "competitive rates at management's discretion."

Which one is right? **Nobody knows.** There is no single source of truth. Changes propagate inconsistently. Testing is incomplete because nobody knows all the places a rule lives. When a customer calls to ask "why was I charged $47.50?", the answer is scattered across four systems, three teams, and a spreadsheet on a laptop that was decommissioned last year.

---

## 2.7 The Hidden Value of Business Rules

Most organisations treat business rules as an implementation detail — something to be coded and forgotten. This is a profound mistake. Business rules are among the most valuable intellectual assets an organisation possesses.

### Rules ARE the Business

Consider what a bank actually *is*. Strip away the marble lobby, the website, the mobile app, the core banking platform. What remains? **Rules.** Rules about who qualifies for a loan. Rules about how interest is calculated. Rules about when a transaction is flagged as suspicious. Rules about capital reserves and regulatory compliance.

The technology is replaceable. The brand is fungible. The rules — refined over decades of regulatory negotiation, competitive learning, and operational experience — are the core intellectual property. An insurance company's underwriting rules represent centuries of accumulated actuarial wisdom. A retailer's pricing rules encode hard-won knowledge about customer behaviour, competitive positioning, and margin management.

When an organisation says "we are migrating to a new platform," the rules are the hardest part to migrate — because they are the part that nobody fully understands.

### Rules Change Faster Than Anything Else

Application code changes with feature releases — quarterly. Infrastructure changes with platform migrations — yearly. Business rules change with the market — weekly, daily, sometimes hourly.

- A competitor drops their price. Your pricing rules must respond *today*.
- A regulator issues emergency guidance. Your compliance rules must be updated *this week*.
- A product manager launches a flash sale. Your discount rules change *for the next four hours*.
- An actuary reviews the loss ratio. Your underwriting rules are tightened *immediately*.

If every rule change requires a code change, a code review, a test cycle, a staging deployment, and a production release, the organisation cannot respond at the speed the business demands. The business moves at the speed of its rules. If its rules are locked inside code, the business moves at the speed of its deployment pipeline.

### Rules Are the Most Duplicated Code

In most enterprise codebases, the same business rule is implemented in multiple places — the server, the batch job, the mobile app, the reporting system, the data warehouse ETL. Each copy was written by a different developer, at a different time, with a different understanding of the rule. They are all slightly different. They all need updating when the rule changes. They almost never are.

---

## 2.8 The Primitives of Business Logic

Here is a claim that will save you enormous amounts of time: **every business rule, in every domain, is composed of combinations of the same ten logical building blocks.** These are the primitives of business logic.

This is not academic. It is a practical tool. Once you can recognise these primitives in a policy document, an email, or a conversation with a domain expert, the path from business rule to executable expression becomes mechanical.

### Primitive 1: Classification

**"Which category does X belong to?"**

A classification maps inputs to one of a finite set of labels. Credit score 720 → "Good". BMI 26.3 → "Overweight". Customer spending over $10,000/year → "Gold" tier.

*Key property:* The categories are mutually exclusive and (ideally) exhaustive — every input maps to exactly one label.

### Primitive 2: Threshold Test

**"Does X exceed / fall below Y?"**

A comparison against a fixed boundary. Age >= 18 (legal adult). Account balance < 0 (overdrawn). Temperature > 38.5 (fever).

*Key property:* The result is always true or false (or unknown, if the input is missing).

### Primitive 3: Range Membership

**"Is X within bounds?"**

A check whether a value falls within an interval. Credit score between 650 and 749 (inclusive). Weight in (0 kg, 30 kg] — more than 0, up to and including 30.

*Key property:* Ranges have start and end points, and each endpoint is either included or excluded.

### Primitive 4: Enumeration Match

**"Is X one of these specific values?"**

Membership in a finite set. Country in {"US", "CA", "MX"}. Status is "Active" or "Pending". Product type is not "Prohibited".

*Key property:* The set of values is finite and known in advance.

### Primitive 5: Conjunction and Disjunction

**"Do A *and* B both hold?" / "Does A *or* B hold?"**

Logical combination. Applicant is employed AND income > 30,000. Customer is Gold tier OR order amount > 500.

*Key property:* Can be nested arbitrarily deep. More than 3 levels of nesting usually signals that a decision table would be clearer.

### Primitive 6: Conditional Mapping

**"If A, then X; otherwise Y."**

A conditional that selects between outcomes. If VIP customer, apply 20% discount; otherwise 5%. If claim amount < $500, auto-approve; otherwise route to adjuster.

*Key property:* Every conditional has an "else". Missing "else" branches are one of the most common sources of business rule bugs.

### Primitive 7: Aggregation

**"What is the sum / count / min / max / average?"**

Reducing a collection to a single value. Total order amount = sum of line item prices. Number of overdue invoices = count where due date < today.

*Key property:* Aggregation operates on a *collection* and produces a *single value*.

### Primitive 8: Iteration with Filter

**"For each item that satisfies a condition, compute something."**

Walking through a collection, optionally filtering, producing a new collection. For each order line where quantity > 10, compute bulk discount. List all invoices older than 90 days.

*Key property:* The result is a new collection, not a single value (that would be aggregation after iteration).

### Primitive 9: Ordering and Priority

**"Which option ranks highest?"**

Sorting or selecting the best according to some criterion. Select the cheapest shipping method. Apply the highest-priority matching discount. Rank candidates by interview score.

*Key property:* Requires a comparison function or a predefined priority list.

### Primitive 10: Temporal Reasoning

**"Is date/time A before/after/within a period of B?"**

Dates, times, durations, and their relationships. Insurance policy expires within 30 days. Employee tenure is at least 2 years. Appointment is between 09:00 and 17:00.

*Key property:* Requires both point-in-time values (dates, timestamps) and durations (periods of time).

---

## 2.9 Worked Example: Decomposing Loan Eligibility

Let us take a realistic business policy and identify every primitive lurking inside it. This is the skill you need — the ability to look at a policy and see the primitives that compose it.

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

| Rule | Primitive(s) | What it encodes |
|------|-------------|-----------------|
| 1. Age 18–70 | **Range membership** | A legal constraint (minimum lending age) and a risk boundary (maximum lending age) |
| 2. Employment status | **Enumeration match** | The underwriting department's list of acceptable income sources |
| 3. No recent bankruptcy | **Temporal reasoning** + **Threshold test** | A regulatory requirement — the Insolvency Act specifies a rehabilitation period |
| 4. Risk from credit score | **Classification** (via ranges) | An actuarial model calibrated on historical default data |
| 5. Decline = ineligible | **Threshold test** (on a classified value) | A risk appetite decision made by the Chief Risk Officer |
| 6. Affordability | **Aggregation** + **Threshold test** | A regulatory affordability test — the 30% ratio is mandated by the financial conduct authority |
| 7. Waive bureau check | **Conjunction** + **Conditional mapping** | A cost-saving measure — bureau checks cost money, and low-risk existing customers are unlikely to surprise |
| 8. All documents signed | **Iteration with filter** (universal quantifier) | A legal requirement — unsigned loan agreements are not enforceable |

Notice: even a short policy of 8 rules uses 8 of our 10 primitives. Also notice — and this is the important part — that **every single rule has a reason**. It comes from somewhere: a regulation, a risk model, a cost decision, a legal requirement. When these rules are buried in `if/else` blocks, those reasons are invisible. When they are expressed in a structured notation, the reasons can be documented alongside the rules.

---

## 2.10 Why This Decomposition Matters

The ten primitives are not an academic exercise. They are a practical tool that unlocks three capabilities:

### 1. Conversations With Domain Experts Become Structured

When a business analyst says "we need to add a new eligibility rule," you can now ask targeted questions:

- "Is this a **classification** (mapping inputs to categories) or a **threshold test** (yes/no)?"
- "What are the **ranges**? Are the boundaries inclusive or exclusive?"
- "Is this an **and** condition (all must be true) or an **or** condition (any suffices)?"
- "What happens in the **else** case? What if the data is **missing**?"

These questions are not about technology. They are about the logic itself. They uncover ambiguities that would otherwise surface as bugs in production — the kind of bugs from Section 2.4 that deny cancer patients coverage or underpay employees for six months.

### 2. The Path to Code Becomes Mechanical

Each primitive maps directly to a FEEL construct. You will learn every mapping in Chapter 3. But the principle is simple: once you have identified the primitives in a business rule, the expression almost writes itself. The hard part is the analysis, not the syntax.

### 3. Completeness Becomes Verifiable

When you know a rule is a **classification** with 4 categories, you can verify that all 4 are covered. When you know it involves **range membership**, you can check that the ranges are contiguous and non-overlapping. When you know it is a **conjunction** of 3 conditions, you can enumerate the 2³ = 8 possible combinations and verify that each has a defined outcome.

This is why business rules deserve a dedicated notation — not because `if/else` chains cannot express the logic, but because `if/else` chains cannot make the logic *visible, verifiable, and auditable*. Decision tables can. FEEL can. That is what the rest of this book teaches you.

---

## Summary

- A **business rule** is a specific, actionable statement about how an organisation operates — a fact about the business, not a piece of code.
- Business rules exist to ensure **consistency**, **fairness**, **compliance**, **speed**, and **auditability** in organisational decision-making.
- Business rules represent the **encoded knowledge of the organisation** — actuarial findings, regulatory constraints, competitive decisions, and operational lessons compressed into executable logic.
- Getting business rules wrong has **real consequences**: customers are harmed, money is lost, regulators intervene, and organisations are damaged. Logic bugs are the worst kind of bug — silent, invisible, cumulative, and expensive.
- Business rules appear in **every domain**: pricing, eligibility, risk assessment, routing, compliance, and computation. They are the operational core of every organisation.
- Business rules **hide** in code, spreadsheets, people's heads, documents, and configuration — often duplicated across multiple systems with conflicting values.
- Every business rule is built from **ten analytical primitives**: classification, threshold test, range membership, enumeration match, conjunction/disjunction, conditional mapping, aggregation, iteration with filter, ordering/priority, and temporal reasoning.
- Decomposing rules into primitives **structures conversations**, makes the path to code **mechanical**, and makes completeness **verifiable**.

---

## Exercises

**Exercise 2.1:** A hospital triage policy states: "Patients with chest pain are Priority 1. Patients with breathing difficulty are Priority 1. Patients with fractures are Priority 2. All others are Priority 3. If the patient is under 5 or over 80, increase priority by one level (Priority 3 becomes 2, Priority 2 becomes 1, Priority 1 stays 1)." Identify every analytical primitive. For each rule, state what it likely represents (clinical guideline? hospital policy? regulatory requirement?).

**Exercise 2.2:** An airline's upgrade policy states: "If a passenger is a Platinum frequent flyer, offer a complimentary upgrade to the next available cabin class. If Gold, offer upgrade at 50% off. If Silver, offer upgrade at 25% off. Upgrades are only available if the higher cabin has at least 3 empty seats." Identify every primitive. What business concerns are these rules balancing? (Hint: customer retention vs. revenue vs. operational constraints.)

**Exercise 2.3:** A payroll rule states: "Overtime is paid at 1.5x the hourly rate for hours worked beyond 40 per week, and at 2x for hours beyond 60. Weekend hours always count as overtime." Identify the primitives. Which of these rules come from legislation, and which from company policy? What happens if you get the 1.5x vs 2x boundary wrong?

---

## What Comes Next

Now that you understand what business rules are, where they live, what they represent, and why getting them right matters so profoundly, Chapter 3 shows you the **structures** for organising them: decision tables, step-by-step computations, dependency graphs, and the mapping from each analytical primitive to its FEEL construct. You will see how the primitives from this chapter become executable, testable, auditable decisions.

---

[Previous: Chapter 1: The World Before FEEL](../part-1-foundations/chapter-01-the-world-before-feel.md) | [Next: Chapter 3: Thinking in Decisions](chapter-03-thinking-in-decisions.md)
