# Chapter 1: The World Before FEEL

> *"Every enterprise has business rules. Most enterprises have them scattered across spreadsheets, code, policy documents, and the heads of people who are about to retire."*

---

## 1.1 The Problem: Business Rules Are Everywhere

Every software system encodes business logic. An e-commerce platform decides which customers qualify for free shipping. A bank determines who gets approved for a mortgage. An insurance company calculates premiums. A hospital routes patients to the correct ward.

This logic is unavoidable. The question is: where does it live?

In most organizations, the answer is "everywhere and nowhere":

- **Buried in application code.** A Java service with 300 lines of nested if/else statements that implements a discount policy. The business analyst who wrote the original policy has no way to verify that the code matches the rules. The developer who wrote the code has moved to another team.

- **Frozen in spreadsheets.** An Excel workbook with 14 tabs, each containing nested `IF(AND(OR(...)))` formulas that determine customer eligibility. Three people in the organization understand it. One of them is on holiday.

- **Scattered across databases.** Stored procedures that encode pricing rules. Configuration tables with cryptic column names like `RULE_CD` and `THRESH_VAL`. A lookup table that someone added a row to in 2019 and nobody remembers why.

- **Trapped in people's heads.** "Ask Sandra — she knows how the escalation rules work."

The cost of this chaos is enormous:

| Problem | Consequence |
|---------|------------|
| Logic is duplicated | The same rule is implemented differently in 3 systems. When the policy changes, only 2 of them get updated. |
| Logic is opaque | Nobody can answer "what are our current eligibility rules?" without reading code. |
| Logic is untestable | You cannot unit-test a spreadsheet formula. You cannot regression-test a policy stored in someone's memory. |
| Logic is owned by nobody | Business analysts define the rules. Developers implement them. Neither side can independently verify the other. |
| Logic changes slowly | Every rule change requires a development cycle: requirements, coding, testing, deployment. A one-line policy change takes three sprints. |

### Worked Example 1.1 — The Pricing Spreadsheet

Consider a fictional e-commerce company, ShopFast, that offers discounts based on customer tier, order size, and promotional campaigns. Here is how the discount logic actually exists in their systems:

**In the "Pricing Rules" Excel file (maintained by the marketing team):**

```
Cell D4: =IF(AND(B4="Gold",C4>=100), 0.20,
           IF(AND(B4="Gold",C4>=50), 0.15,
             IF(AND(B4="Silver",C4>=100), 0.10,
               IF(B4="Silver", 0.05, 0))))
```

**In the order service (maintained by the backend team):**

```java
public double calculateDiscount(String tier, double amount) {
    if (tier.equals("Gold") && amount >= 100) return 0.20;
    if (tier.equals("Gold") && amount >= 50) return 0.15;
    if (tier.equals("Silver") && amount >= 100) return 0.10;
    if (tier.equals("Silver")) return 0.05;
    // TODO: what about Bronze? Ask marketing.
    return 0;
}
```

**In the email from the VP of Sales (three months ago):**

> "Starting Q4, Gold customers should get 25% off orders over $200. Can someone update the system?"

Nobody updated the spreadsheet. The developer updated the code but missed the email about the $200 threshold. The mobile app still uses the old API. Three systems, three different discount calculations, zero confidence that any of them is correct.

This is not a hypothetical scenario. This is Tuesday at most enterprises.

---

## 1.2 Decision Management: A Discipline

The problem described above was recognised decades ago. The solution is a discipline called **decision management**: the practice of identifying, modelling, automating, and governing business decisions as first-class artefacts, separate from the processes and applications that use them.

### A Brief History

| Era | Approach | Limitation |
|-----|----------|------------|
| **1970s–80s** | Expert systems (MYCIN, XCON) | Expensive, brittle, domain-specific |
| **1990s** | Business rules engines (ILOG, Fair Isaac) | Proprietary languages, vendor lock-in |
| **2000s** | Open-source rules engines (Drools, OpenRules) | Still code-centric, no standard notation |
| **2010s** | Standards emerge: DMN 1.0 (2015), DMN 1.1 (2016) | Early versions limited (no full expression language) |
| **2020s** | DMN matures: 1.3 (2021), 1.4 (2023), 1.5 (2024), 1.6 (2024 beta) | Current state: a robust, multi-vendor standard |

The key insight behind decision management is **separation of concerns**: separate *what* to decide from *how* to execute. The business logic — the rules, the tables, the formulas — should be expressed in a notation that both business analysts and developers can read, test, and maintain. The execution — how those rules are invoked, scaled, and integrated into applications — is the concern of the runtime platform.

### Who Standardises This?

The **Object Management Group (OMG)** is an international, non-profit technology standards consortium — the same body that produced UML, CORBA, and BPMN. The OMG develops and **maintains** the DMN and FEEL specifications through a formal standards process involving multiple vendors, academics, and practitioners.

In 2015, the OMG published version 1.0 of **Decision Model and Notation (DMN)**. The specification has been actively maintained and evolved through versions 1.1 (2016), 1.2 (2019), 1.3 (2021), 1.4 (2023), 1.5 (2024), and the current 1.6 beta (2024). Each revision refines the type system, adds built-in functions, and tightens the specification of FEEL's semantics.

DMN provides:

1. A graphical notation for decision requirements (who needs what to decide what).
2. A tabular notation for decision logic (decision tables).
3. An expression language for everything that does not fit in a table.

That expression language is **FEEL**. Like DMN itself, FEEL is maintained by the OMG — it is not owned by any single vendor. This means that FEEL expressions are portable across conformant engines from different vendors: a decision table written against one CL3 engine should produce the same results on another.

---

## 1.3 DMN in 60 Seconds

Before we dive into FEEL, you need the 30,000-foot view of DMN. Think of DMN as a three-layer cake:

### Layer 1: Decision Requirements

A **Decision Requirements Diagram (DRD)** is a graph. Nodes are:
- **Decisions** (rectangles): things you need to determine. "Is this applicant eligible?"
- **Input Data** (ovals): facts from the outside world. "Applicant's age", "Credit score".
- **Business Knowledge Models** (rectangles with clipped corners): reusable logic. "Affordability calculation".
- **Knowledge Sources** (documents): authority for the rules. "Underwriting policy", "Regulatory guidelines".

Edges show dependencies: "To determine eligibility, I need the risk category and the affordability assessment."

The DRD does not contain any logic — it just shows the structure.

### Layer 2: Decision Logic

Each decision in the DRD has associated logic, expressed as a **boxed expression**. The most common form is a **decision table**: a grid of conditions and conclusions. But boxed expressions also include contexts (step-by-step calculations), invocations (function calls), lists, conditionals, and more.

### Layer 3: Expression Language

Inside every box — every decision table cell, every context entry, every condition — there are **FEEL expressions**. FEEL is the language inside the boxes.

You cannot understand DMN without FEEL. And FEEL without DMN is an expression language without a home. This book focuses on FEEL, but always in the context of the decisions it serves.

---

## 1.4 What Is FEEL?

FEEL stands for **Friendly Enough Expression Language**.

Not "friendly." Not "simple." *Friendly enough.* That qualifier is deliberate. FEEL is designed for a dual audience:

- **Business analysts** should be able to *read* FEEL expressions and verify that they match the intended business rules.
- **Developers** should be able to *write* FEEL expressions and integrate them into production systems.

Neither audience gets everything they want. Analysts might wish it were even simpler. Developers might wish it had stronger typing or familiar syntax. The "enough" in "friendly enough" is the compromise.

### What Kind of Language Is FEEL?

FEEL is an **expression language** — not a programming language. This distinction matters:

| Programming Language | Expression Language (FEEL) |
|---------------------|--------------------------|
| Has statements (`if`, `while`, `for`, `return`, assignment) | Has only expressions — every construct produces a value |
| Has side effects (I/O, mutation, exceptions) | Side-effect free — same inputs always produce the same output |
| Has variables you can reassign | Has names you can bind but never mutate |
| Errors throw exceptions | Errors produce `null` |
| Turing-complete | Deliberately limited (no unbounded loops, no general recursion*) |

*FEEL does support recursion via named function context entries, but it is not Turing-complete in the formal sense because the specification does not guarantee unbounded recursion depth.

### Design Choices That Will Surprise You

If you are coming from JavaScript, Python, Java, or C#, FEEL will feel unfamiliar in several ways. This is not a bug — these choices are deliberate:

| Surprise | Why? |
|----------|------|
| **Names can contain spaces and hyphens**: `Applicant Age`, `risk-score` | Business analysts write in natural language. `applicantAge` is developer jargon. |
| **1-based indexing**: `list[1]` is the first element | Business users count from 1. |
| **Three-valued logic**: `true`, `false`, `null` | Like SQL, missing data must be modelled explicitly. |
| **Built-in function names use spaces**: `string length()`, `years and months duration()` | Readability for non-developers. |
| **No exceptions**: errors produce `null` | Simpler mental model — but harder to debug. |
| **No type coercion**: `"1" + 1` is `null`, not `"11"` or `2` | Explicit is better than implicit (for a rule language). |

Each of these will be covered in depth in later chapters. For now, just know that the unfamiliarity is intentional, and there is a rationale behind every choice.

### What FEEL Looks Like

Here are some FEEL expressions to give you a taste. Do not worry about understanding every detail — each construct will be explained fully in later chapters.

**A simple condition:**
```
Applicant.Age >= 18 and Applicant.Age <= 65
```

**A conditional expression:**
```
if Credit Score >= 750 then "Excellent"
else if Credit Score >= 650 then "Good"
else "Review Required"
```

**A list filter:**
```
Invoices[Status = "Overdue" and Amount > 1000]
```

**A step-by-step calculation (context):**
```
{
  Monthly Income: Applicant.Income,
  Monthly Obligations: Applicant.Expenses + Applicant.Repayments,
  Disposable: Monthly Income - Monthly Obligations,
  Affordable: Disposable >= Required Installment
}
```

**A loop over a list:**
```
for item in Order.Lines return item.Quantity * item.Unit Price
```

If these look more like English than code — that is the point.

---

## 1.5 The FEEL Ecosystem

FEEL is a standard, not a product. Multiple vendors and open-source projects implement it. Knowing the landscape will help you choose tools and understand why the same expression might behave slightly differently across engines.

### Engines and Implementations

| Engine | Vendor/Project | Language | Conformance | Notes |
|--------|---------------|----------|-------------|-------|
| **feel-scala** | Camunda | Scala/JVM | Near-full CL3 | Most widely used open-source FEEL engine |
| **Drools DMN** | Apache KIE (Red Hat) | Java | Full CL3 | Claims 100% spec conformance |
| **Trisotech** | Trisotech | Cloud | Full CL3 | Strong DMN TCK contributor |
| **IBM ODM** | IBM | Java | CL3 | Enterprise-focused |
| **SAP Signavio** | SAP | Cloud | Partial (DMN 1.0) | Limited FEEL support |
| **feelin** | Nico Rehwaldt (bpmn.io) | JavaScript | Partial CL3 | Lightweight, MIT licensed |
| **dmn-eval-js** | HBT GmbH | JavaScript | S-FEEL + partial CL3 | Useful for browser evaluation |
| **pySFeel** | Community | Python | S-FEEL only | Limited to simple expressions |

### Conformance Levels

The DMN specification defines three conformance levels:

- **CL1**: Decision tables with simple literal expressions. No FEEL beyond basic values.
- **CL2**: Decision tables with S-FEEL (Simple FEEL) — a restricted subset with arithmetic, comparisons, and simple values.
- **CL3**: Full FEEL — contexts, lists, functions, iteration, quantification, the complete built-in function library.

This book targets **CL3**. If you are building anything beyond trivial decision tables, you need full FEEL.

### Playgrounds and Tools

You do not need to install anything to start experimenting with FEEL:

| Tool | What it is |
|------|-----------|
| **Camunda FEEL Playground** | Browser-based evaluator using feel-scala. The most complete online tool. |
| **Nikku's FEEL Playground** | Lightweight browser-based evaluator using feelin (JavaScript). |
| **feel-scala REPL** | Command-line REPL for local evaluation. |
| **bpmn.io DMN Editor** | Browser-based DMN modeller with FEEL support. |

### The DMN Technology Compatibility Kit (TCK)

The DMN TCK is a suite of test cases that verify whether an engine correctly implements the specification. It is hosted at `github.com/dmn-tck/tck`. When evaluating engines, check their TCK pass rate — it is the most objective measure of conformance.

---

## 1.6 Who This Book Is For

This book is written for **working software developers** — people who write code for a living in languages like Java, JavaScript, Python, C#, or similar. You do not need prior experience with DMN, decision management, or business rules engines.

You **will** benefit from this book if you:

- Build enterprise applications that contain business logic (pricing, eligibility, risk assessment, routing, compliance).
- Want to separate business rules from application code — and understand why that matters.
- Need to integrate with a DMN engine (Camunda, Drools/KIE, Trisotech, or others) and want to understand the language inside the boxes.
- Are a technical architect evaluating whether DMN and FEEL are the right fit for your organisation.
- Are a business analyst with programming experience who wants to write and maintain FEEL expressions directly.

You will **not** benefit from this book if you are looking for a general introduction to BPMN, a guide to a specific vendor's product, or a book on machine learning. This book is narrowly focused on one thing: teaching you to read, write, and think in FEEL.

---

## 1.7 How to Use This Book

### Structure

This book is organised in five parts, progressing from easy to hard:

1. **Part I** (this chapter) explains *why* FEEL exists and what the ecosystem looks like.
2. **Part II** teaches you to *think in decisions* — how to decompose business logic into its analytical primitives before writing any FEEL.
3. **Part III** teaches FEEL itself, from atomic values to complex expressions, layer by layer.
4. **Part IV** covers real-world application: patterns, engine integration, dialect variants, and advanced topics.
5. **Part V** provides reference material: a quick reference card, exercise solutions, a cross-language Rosetta Stone, and a glossary.

### Conventions

Throughout the book, you will encounter several recurring elements:

**FEEL expressions** are shown in code blocks:
```
sum(Order.Items.Price)
```

**Rosetta Stone boxes** compare FEEL to JavaScript, Python, and SQL, so you can map unfamiliar constructs to ones you already know:

| FEEL | JavaScript | Python | SQL |
|------|-----------|--------|-----|
| `list[1]` | `list[0]` | `list[0]` | First row |
| `count(list)` | `list.length` | `len(list)` | `COUNT(*)` |

**Gotcha boxes** highlight common traps that catch developers:

> **GOTCHA:** `"1" + 1` evaluates to `null` in FEEL — not `"11"` (JavaScript) or `2` (Python). FEEL does not perform implicit type coercion. Use `string(1)` or `number("1")` to convert explicitly.

**Engine Note boxes** flag behavioural differences between implementations:

> **ENGINE NOTE (Camunda vs KIE):** Camunda's feel-scala allows `date and time` values without a timezone. KIE's Drools engine may normalise these to UTC. Test your expressions in the target engine.

### Reference Engine

The examples in this book are tested against **Camunda 8** (feel-scala) as the primary reference engine. Where Drools/KIE or Trisotech behave differently, this is noted. The companion repository contains all examples as executable `.feel` and `.dmn` files.

### Exercises

Every chapter (from Chapter 3 onward) ends with exercises. Solutions are in Appendix B. The exercises follow the same progressive structure as the chapters — start with the first few, and attempt the later ones only after you are comfortable with the material.

---

## Summary

- Business rules are everywhere; the problem is that they are scattered, duplicated, opaque, and untestable.
- Decision management separates *what to decide* from *how to execute*.
- DMN is the OMG standard for modelling decisions. It has three layers: requirements, logic, and expressions.
- FEEL is the expression language inside DMN — designed for dual readership by analysts and developers.
- FEEL is an expression language, not a programming language: no statements, no side effects, no exceptions.
- Multiple engines implement FEEL with varying levels of conformance. The DMN TCK measures conformance.
- This book teaches FEEL progressively — from the light to the heavy — with worked examples, Rosetta Stone comparisons, and gotcha warnings at every step.

---

## What Comes Next

In Chapter 2, we will step back from syntax entirely and learn to *think in decisions*. Before writing a single FEEL expression, you will learn to decompose any business policy into a small set of analytical building blocks — and then see how each building block maps to FEEL.
