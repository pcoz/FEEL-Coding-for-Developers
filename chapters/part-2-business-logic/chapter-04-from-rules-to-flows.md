# Chapter 4: From Rules to Flows

> *"A business process does not need to know how to calculate a credit score. It only needs to know that a credit score exists, and what to do with it."*

---

## 4.1 The Entanglement Problem

Open any enterprise codebase and you will find the same disease: business rules and process flow knotted together so tightly that you cannot change one without risking the other. Here is a typical order fulfilment service -- see if this looks familiar:

```java
public OrderResult processOrder(Order order) {
    // Business rule: validate the order
    if (order.getItems().isEmpty()) {
        return OrderResult.rejected("No items");
    }
    for (Item item : order.getItems()) {
        if (item.getQuantity() <= 0) {
            return OrderResult.rejected("Invalid quantity");
        }
        if (!inventoryService.isAvailable(item.getSku(), item.getQuantity())) {
            return OrderResult.rejected("Out of stock: " + item.getSku());
        }
    }

    // Business rule: calculate pricing
    double subtotal = 0;
    for (Item item : order.getItems()) {
        double price = pricingService.getPrice(item.getSku());
        double discount = 0;
        if (order.getCustomer().getTier().equals("Gold") && price > 50) {
            discount = 0.15;
        } else if (order.getCustomer().getTier().equals("Silver")) {
            discount = 0.05;
        }
        subtotal += price * item.getQuantity() * (1 - discount);
    }

    // Business rule: determine shipping
    String shipping;
    if (subtotal > 100 && order.getCustomer().getTier().equals("Gold")) {
        shipping = "FREE";
    } else if (order.getDestination().getZone().equals("A")) {
        shipping = "STANDARD";
    } else {
        shipping = "PREMIUM";
    }

    // Process flow: route based on result
    if (shipping.equals("FREE")) {
        notificationService.sendFreeShippingConfirmation(order);
    }
    paymentService.charge(order.getCustomer(), subtotal);
    warehouseService.dispatch(order, shipping);
    return OrderResult.accepted(subtotal, shipping);
}
```

This single method does at least four different jobs:

1. **Validates** the order (business rule)
2. **Calculates** the price (business rule)
3. **Determines** the shipping method (business rule)
4. **Orchestrates** the process (send notification, charge payment, dispatch)

The business rules and the process flow are *fused*. And that fusion hurts in ways that compound over time:

| Problem | Consequence |
|---------|------------|
| **Rules are invisible.** | The discount policy is buried inside a `for` loop. A business analyst cannot read it without reading Java. |
| **Rules are untestable in isolation.** | To test the discount logic, you need to construct an `Order`, mock `pricingService`, and run the entire method. |
| **Process changes require rule changes (and vice versa).** | Adding a "gift wrap" step to the process requires editing the same method that contains pricing logic. |
| **Rules cannot be reused.** | The mobile app, the batch processing job, and the API gateway all need the same discount logic — so it gets copy-pasted. |
| **Rules change at a different cadence than processes.** | The discount percentages change quarterly. The fulfilment process changes yearly. Yet they live in the same deployment unit. |

---

## 4.2 The Separation: Rules Are Decisions, Flows Are State Machines

The fix is architectural, and it fits in one sentence: **separate what to decide from what to do**.

Strip away the details and every business process is a **state machine**. It moves through states -- order received, validated, priced, shipped, completed -- based on *transitions*. Each transition fires on an event or a condition. The conditions come from *decisions*. And here is the crucial part: the process does not need to know *how* a decision is made. It only needs the answer, so it can pick the next state.

Here is the same order fulfilment logic, pulled apart into its natural halves:

### The Decisions (expressed in FEEL / decision tables)

**Validation Decision:**
```
{
  items valid:  every item in Order.Items satisfies
                  item.Quantity > 0,
  stock available: every item in Order.Items satisfies
                     Inventory[SKU = item.SKU].Available >= item.Quantity,
  valid: items valid and stock available
}
```

**Pricing Decision (decision table, hit policy U):**

| U | Customer Tier | Item Price | Discount |
|---|--------------|-----------|----------|
| 1 | "Gold" | > 50 | 0.15 |
| 2 | "Gold" | <= 50 | 0.10 |
| 3 | "Silver" | - | 0.05 |
| 4 | "Bronze" | - | 0 |

**Shipping Decision (decision table, hit policy F):**

| F | Subtotal | Customer Tier | Zone | Shipping Method |
|---|----------|--------------|------|----------------|
| 1 | > 100 | "Gold" | - | "FREE" |
| 2 | - | - | "A" | "STANDARD" |
| 3 | - | - | - | "PREMIUM" |

### The Process (a state machine)

```
   ┌──────────┐
   │ Received │
   └────┬─────┘
        │
        ▼
   ┌──────────┐     ┌──────────────────────────────┐
   │ Validate │────▶│ Call: Validation Decision     │
   └────┬─────┘     └──────────────────────────────┘
        │
    ┌───┴───┐
    │valid? │
    └───┬───┘
   yes  │  no
   ┌────┘  └────┐
   ▼            ▼
┌──────┐    ┌──────────┐
│Price │    │ Rejected │
│      │    └──────────┘
│      │────▶┌──────────────────────────────┐
│      │     │ Call: Pricing Decision        │
└──┬───┘     └──────────────────────────────┘
   │
   ▼
┌──────────┐     ┌──────────────────────────────┐
│Determine │────▶│ Call: Shipping Decision       │
│ Shipping │     └──────────────────────────────┘
└────┬─────┘
     │
     ▼
┌──────────┐
│ Dispatch │
└────┬─────┘
     │
     ▼
┌───────────┐
│ Completed │
└───────────┘
```

Notice what the state machine does *not* know. It has zero awareness of discount percentages, price thresholds, or zone classifications. All it knows is:

1. There is a decision called "Validation" that returns `{valid: true/false}`.
2. There is a decision called "Pricing" that returns `{subtotal: number}`.
3. There is a decision called "Shipping" that returns `{method: string}`.
4. Based on the results, it transitions between states.

---

## 4.3 Why This Separation Matters

### 4.3.1 Rules Change Independently of Flows

Imagine the discount policy changes from 15% to 20% for Gold customers. In the separated architecture:

- **What changes:** One cell in a decision table.
- **What does not change:** The state machine. The code. The deployment pipeline.
- **Who can make the change:** A business analyst, using a DMN modelling tool, without touching code.
- **How it is tested:** Re-run the decision tests with the new expected outputs.

In the entangled architecture? That same one-number change means modifying Java code, running the full test suite, surviving code review, and rolling a deployment. For a percentage.

### 4.3.2 Flows Change Independently of Rules

Now flip it around. The business wants to add a "gift wrapping" step between pricing and shipping. In the separated architecture:

- **What changes:** The state machine gains a new state and transition.
- **What does not change:** The pricing decision. The shipping decision. The validation decision.
- **Who can make the change:** A process engineer, using a BPMN modelling tool.

### 4.3.3 Decisions Are Testable in Isolation

Each decision is a pure function: inputs in, outputs out. No mocks. No service dependencies. No database. No network. Just data.

```
Input:  { Customer Tier: "Gold", Item Price: 75 }
Output: { Discount: 0.15 }
```

You can write hundreds of test cases as simple input/output pairs. You can generate them from the decision table itself. You can run them in milliseconds.

### 4.3.4 Decisions Are Reusable Across Processes

The pricing decision is needed by the order fulfilment process, the invoice generation batch job, *and* the mobile app's cart preview. Without separation, that logic gets copy-pasted three times and drifts three different ways. With separation, all three call the same decision service. One source of truth. Zero drift.

### 4.3.5 Decisions Are Auditable

A regulator asks "why was this customer charged $47.50?" You can trace the answer through the decision in seconds:

1. The pricing decision was invoked with inputs `{Tier: "Silver", Items: [...]}`.
2. Rule 3 matched (Silver tier, any price, 5% discount).
3. The subtotal was computed as 50.00 * 0.95 = 47.50.

The decision table *is* the audit trail. No code reading required.

---

## 4.4 The Architecture in Practice

### 4.4.1 BPMN + DMN: The Standard Stack

This separation is not a novel idea -- it is exactly what the OMG designed BPMN and DMN to do. They were built as two halves of a whole:

- **BPMN** models the process flow — the state machine.
- **DMN** models the decisions — the business rules.
- BPMN **Business Rule Tasks** invoke DMN **Decision Services**.

```
BPMN Process                         DMN Decision Model
┌─────────────────┐                  ┌──────────────────────┐
│                 │                  │                      │
│  ┌───────────┐  │    invokes       │  ┌────────────────┐  │
│  │ Business  │──┼─────────────────▶│  │ Decision       │  │
│  │ Rule Task │  │                  │  │ Service        │  │
│  └───────────┘  │    returns       │  │                │  │
│       │         │◀─────────────────┼──│  Inputs:       │  │
│       ▼         │   {result: ...}  │  │   Applicant    │  │
│  ┌───────────┐  │                  │  │   Product      │  │
│  │ Gateway   │  │                  │  │  Outputs:      │  │
│  │ (decides  │  │                  │  │   Eligibility  │  │
│  │  routing) │  │                  │  │   Risk         │  │
│  └───────────┘  │                  │  └────────────────┘  │
│                 │                  │                      │
└─────────────────┘                  └──────────────────────┘
```

The BPMN process does not contain any business rules. It contains:
- Tasks that invoke decisions
- Gateways that route based on decision outputs
- Events that trigger state transitions

The DMN model does not contain any process logic. It contains:
- Decisions with their inputs and outputs
- Decision tables and FEEL expressions
- Dependencies between decisions

### 4.4.2 Microservices and Decision Services

Not using BPMN? The pattern still holds. A decision service is simply a microservice that:

1. Accepts a JSON payload with the decision inputs.
2. Evaluates the DMN model.
3. Returns a JSON payload with the decision outputs.

The caller -- whether it is an API gateway, a workflow engine, an event handler, or a batch job -- treats the decision as a black box. Send the facts, get the verdict, act on it.

```
┌──────────────┐     POST /decisions/eligibility     ┌──────────────────┐
│              │     { applicant: {...},              │                  │
│  Order       │        product: {...} }              │  Decision        │
│  Service     │ ───────────────────────────────────▶ │  Service         │
│              │                                      │  (DMN engine)    │
│  (state      │     { eligible: true,                │                  │
│   machine)   │ ◀─────────────────────────────────── │  Evaluates FEEL  │
│              │       risk: "Low" }                   │  expressions     │
└──────────────┘                                      └──────────────────┘
```

### 4.4.3 Event-Driven Architectures

In an event-driven system, the state machine reacts to events. Some of those events carry data that needs a judgment call before the process can continue. The pattern looks like this:

1. Event arrives (e.g., `OrderPlaced`).
2. State machine transitions to a "deciding" state.
3. Decision service is invoked with the event payload.
4. Decision result triggers the next state transition.
5. State machine emits a new event (e.g., `OrderApproved` or `OrderRejected`).

The decision service remains blissfully ignorant of events, states, and transitions. It only knows about inputs and outputs. FEEL expressions inside it are pure functions -- no side effects, no awareness of the world outside their input data.

---

## 4.5 The Process Only Needs to Know About the Decisions

If you take only one idea from this chapter, make it this one:

> **A business process flow only needs to know about the business rules.** It does not need to implement them, understand them, or contain them. It needs to know:
>
> 1. **What** decisions exist (by name).
> 2. **What** inputs each decision requires.
> 3. **What** outputs each decision produces.
> 4. **What** to do based on the outputs (routing).
>
> Everything else — the actual logic, the tables, the formulas, the edge cases — is encapsulated in the decision model.

If this sounds familiar, it should -- it is the same principle behind every well-designed service interface. The consumer knows the contract (inputs, outputs, guarantees), not the implementation.

### The Benefits Compound

When you adopt this separation, each piece becomes simpler:

| Component | Complexity | Who Owns It |
|-----------|-----------|-------------|
| **State machine** | Simple: states, transitions, decision invocations | Process engineers / developers |
| **Decision model** | Moderate: tables, expressions, dependencies | Business analysts + developers |
| **FEEL expressions** | Focused: pure logic, no side effects, independently testable | Developers (with analyst review) |

And the system as a whole gains properties that you simply cannot get when rules and flows are tangled together, no matter how clever the code:

- **Rules are visible** — expressed in a standard notation, not buried in code.
- **Rules are versionable** — a decision table is a document that can be versioned, diff'd, and rolled back.
- **Rules are deployable independently** — change a discount percentage without redeploying the order service.
- **Rules are explainable** — "why was this claim rejected?" can be answered by replaying the decision with the same inputs.
- **Processes are lean** — the state machine is simple enough to draw on a whiteboard.

---

## 4.6 From Monolith to State Machine: A Migration Path

Good news: you do not need to stop the world and rewrite everything. This migration works best when you do it one decision at a time.

### Step 1: Identify the Decisions

Go on a hunting expedition through your codebase. Every `if/else` block, every `switch` statement, every lookup table that encodes a business rule is a candidate. Tag them. You will be surprised how many there are.

### Step 2: Extract One Decision

Pick the simplest, most self-contained business rule -- the low-hanging fruit. Express it as a FEEL expression or decision table. Wrap it in a decision service (even if that "service" is just a function call to a local DMN engine). Get a win on the board.

### Step 3: Replace the Code with a Call

Rip out the original `if/else` block and replace it with a call to your decision service. The calling code still receives the result and acts on it -- but it no longer computes it. That distinction matters more than it seems.

### Step 4: Test Side-by-Side

Run both the old code path and the new decision service with the same inputs. Verify that the outputs match. Fix any discrepancies -- and there *will* be discrepancies. This is the step where you discover that the old code had bugs nobody knew about.

### Step 5: Repeat

Extract the next decision. And the next. With each extraction, the remaining code sheds complexity and becomes increasingly process-oriented. Something interesting happens: a state machine starts to emerge on its own.

### Step 6: Formalise the State Machine

Once enough decisions have been pulled out, what remains is simple enough to express as a formal state machine -- in BPMN, in a workflow engine, or in a state machine library in your language of choice.

### The Result

```
Before:                           After:
┌──────────────────────────┐      ┌──────────────┐   ┌──────────────┐
│                          │      │              │   │              │
│   Monolithic Service     │      │  State       │   │  Decision    │
│                          │      │  Machine     │──▶│  Service     │
│   - validation rules     │      │              │   │              │
│   - pricing rules        │      │  - states    │   │  - FEEL      │
│   - shipping rules       │      │  - events    │   │  - tables    │
│   - process flow         │      │  - routing   │   │  - formulas  │
│   - notifications        │      │              │   │              │
│   - error handling       │      └──────────────┘   └──────────────┘
│                          │
└──────────────────────────┘
```

The monolith has been split along its natural fault line: rules on one side, flow on the other. Each half is simpler, more testable, and more maintainable than the tangled whole ever was.

---

## 4.7 FEEL Contexts as Business State

A FEEL context looks like a data structure, but think of it as something more powerful: a **business state representation**. When a decision service finishes evaluating, the result is a context that captures the complete outcome -- what was decided, what the computed values are, and what classification was assigned.

This context *is* the contract between the decision layer and the process layer. The state machine does not care how the values were computed. It receives a structured result and routes accordingly.

### The Pattern: Decision Returns Business State

Take an insurance claim assessment. The decision model evaluates several rules and packs everything into a single context:

```
{
  claim valid: true,
  risk category: "Medium",
  approved amount: 4500,
  requires manual review: false,
  reason codes: ["within-policy-limit", "no-prior-claims"],
  next action: "auto-approve"
}
```

That context *is* the **business state** of the claim after evaluation. The process flow receives it and uses its fields to pick the next state transition:

- If `next action = "auto-approve"` → transition to the Approve state.
- If `next action = "manual-review"` → transition to the Review state.
- If `next action = "decline"` → transition to the Decline state.

The process never inspects `approved amount` or `reason codes` to make its routing decision — those fields exist for downstream consumers (notifications, audit logs, the customer-facing portal). The process only reads the fields it needs for routing.

### Why Contexts, Not Scalars

Sure, a decision *could* return a single scalar -- just `"approve"`. But a rich context pays for itself immediately:

| Approach | Limitation |
|----------|-----------|
| Return a single string (`"approve"`) | Downstream systems need additional calls to get the details |
| Return a tuple of values | Position-dependent, fragile, unreadable |
| Return a FEEL context | Self-documenting, extensible, directly serialisable to JSON |

A FEEL context maps naturally to a JSON object, a Java `Map`, a Python `dict`, or a C# `Dictionary`. That makes it the ideal **boundary type** between your decision engine and whatever host application calls it.

### Example: Order Assessment with Business State

**The FEEL decision (returns a context):**

```
{
  line totals: for item in Order.Items
                 return { sku: item.SKU,
                          qty: item.Quantity,
                          subtotal: item.Quantity * item.Unit Price },
  order subtotal: sum(line totals.subtotal),
  discount rate: if Customer.Tier = "Gold" and order subtotal > 100 then 0.15
                 else if Customer.Tier = "Silver" then 0.05
                 else 0,
  discount amount: order subtotal * discount rate,
  total: order subtotal - discount amount,
  shipping method: if total > 100 and Customer.Tier = "Gold" then "FREE"
                   else if Customer.Zone = "A" then "STANDARD"
                   else "PREMIUM",
  order state: {
    status: "assessed",
    total: total,
    shipping: shipping method,
    discount applied: discount rate > 0,
    discount percentage: discount rate * 100
  }
}.order state
```

The final `.order state` projection strips away the working variables and returns only the business-relevant summary -- a clean context that the process can consume without knowing a thing about how discounts, shipping methods, or totals were calculated.

**The process receives:**

```json
{
  "status": "assessed",
  "total": 85.00,
  "shipping": "STANDARD",
  "discount_applied": true,
  "discount_percentage": 5
}
```

The state machine reads `status` and transitions. The order service reads `total` and `shipping` to dispatch the order. The notification service reads `discount_applied` to decide whether to mention the discount in the confirmation email. Each consumer reads only the fields it cares about.

### Example: Employee Onboarding Routing

```
{
  department valid: list contains(["Engineering", "Sales", "Finance", "HR"], Employee.Department),
  has required docs: every doc in ["ID", "Tax Form", "Contract"]
                       satisfies list contains(Employee.Submitted Documents, doc),
  missing docs: ["ID", "Tax Form", "Contract"][not(list contains(Employee.Submitted Documents, item))],
  equipment tier: if Employee.Role = "Developer" then "full-workstation"
                  else if Employee.Role = "Manager" then "laptop-only"
                  else "shared-terminal",
  onboarding state: {
    ready: department valid and has required docs,
    missing documents: missing docs,
    equipment request: equipment tier,
    assigned office: if Employee.Department = "Engineering" then "Building B"
                     else "Building A"
  }
}.onboarding state
```

**The process receives:**

```json
{
  "ready": false,
  "missing_documents": ["Tax Form"],
  "equipment_request": "full-workstation",
  "assigned_office": "Building B"
}
```

If `ready` is `false`, the state machine transitions to "Awaiting Documents" and fires off a notification listing `missing_documents`. If `ready` is `true`, it transitions to "Provision Equipment" using `equipment_request` and "Assign Desk" using `assigned_office`. The decision encapsulates *all* the business logic. The process just reads and routes -- exactly as it should.

---

## 4.8 Real-World Example: Loan Origination (Revisited)

Let's bring this full circle with the loan origination process from Chapter 3. The DMN specification models it as:

**The process (BPMN state machine):**

```
Collect Application Data
    │
    ▼
Decide Bureau Strategy  ──▶  [DMN Decision Service: Strategy + Bureau Call Type]
    │
    ├── Strategy = "Bureau" ──▶ Collect Bureau Data ──▶ Decide Routing
    ├── Strategy = "Through" ──▶ Decide Routing
    └── Strategy = "Decline" ──▶ Decline Application
                                        │
                            ┌───────────┴───────────┐
                    Routing = "Accept"        Routing = "Refer"
                            │                       │
                            ▼                       ▼
                    Accept Application      Review Application
                                                    │
                                            ┌───────┴───────┐
                                     Accepted           Declined
```

**The decisions (DMN):**

- Strategy → decision table (2 inputs, 1 output)
- Bureau Call Type → decision table (1 input, 1 output)
- Eligibility → decision table (3 inputs, 1 output)
- Risk Category → decision table (2 inputs, 1 output)
- Affordability → FEEL context (5 inputs, 1 output)
- Application Risk Score → scorecard decision table (3 inputs, 1 output, C+ hit policy)
- Routing → decision table (4 inputs, 1 output)

The process has **7 states** and **6 transitions**. The decision model has **10 decisions** with complex interrelated logic. Yet they are completely separate. Change the risk scoring model? The process does not notice. Add a new "verify identity" step to the process? The decision logic does not care.

This is the architecture that FEEL enables. The rest of this book teaches you the language that powers the decision side of that split.

---

## Summary

- In most codebases, business rules and process flow are tangled together. This makes rules invisible, untestable, and hard to change.
- The solution is to separate rules (decisions) from flow (state machine). The process only needs to know *what* decisions exist, *what* they need, and *what* they return.
- BPMN models the process. DMN models the decisions. FEEL is the language inside the decisions.
- FEEL contexts serve as **business state representations** — structured results that map directly to JSON, Java Maps, Python dicts, and other host-language data structures. The decision returns a context; the process reads it and routes.
- This separation enables: independent change cadence, isolated testing, multi-channel reuse, auditability, and simpler process logic.
- Migration is incremental: extract decisions one at a time, replace code with decision service calls, and watch the state machine emerge.

---

## What Comes Next

You now know *why* FEEL exists (Chapter 1), *what* business rules and business logic are (Chapter 2), *how* to think in decisions (Chapter 3), and *where* FEEL fits in the architecture (this chapter). The stage is set. Chapter 5 begins with the atoms of FEEL itself: values and types.

---

[Previous: Chapter 3: Thinking in Decisions](chapter-03-thinking-in-decisions.md) | [Next: Chapter 5: Values and Types](../part-3-feel-ground-up/chapter-05-values-and-types.md)
