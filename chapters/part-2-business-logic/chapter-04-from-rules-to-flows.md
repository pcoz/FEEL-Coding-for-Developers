# Chapter 4: From Rules to Flows

> *"A business process does not need to know how to calculate a credit score. It only needs to know that a credit score exists, and what to do with it."*

---

Every piece of business-oriented code — every function, every service, every monolith — contains exactly three things: **business rules** that make decisions, **business workflow** that sequences and routes those decisions, and **technical augmentation** that connects it all to the real world. In most codebases, these three layers are tangled together so tightly that nobody can see them. This chapter teaches you to see them, separate them, and implement the separation — with OMG standards like BPMN and DMN if you want them, or with plain code and workflow orchestrators if you do not.

---

## 4.1 The Three Layers of Business Code

Every piece of code that serves a business purpose contains three distinct concerns, whether the developer who wrote it realised it or not:

| Layer | What It Does | Examples |
|-------|-------------|----------|
| **Business Rules** | Makes decisions. Evaluates conditions, applies policies, computes values. | "Critical patients go to ICU." "Gold customers get 15% off orders over $50." "Applicants with a credit score below 600 are ineligible." |
| **Business Workflow** | Sequences and routes. Determines what happens next, in what order, based on what results. | "After triage, verify insurance. If pre-authorisation is needed, request it. Then assign a bed and admit the patient." |
| **Technical Augmentation** | Connects the logic to the real world. The plumbing that makes it actually run. | Database queries, API calls, message queues, authentication, serialisation, error handling, logging, UI rendering. |

These three layers are present in *every* business application. Not most. Every:

- **Hospital admission:** Rules (triage priority, insurance eligibility, bed assignment). Workflow (arrival → triage → insurance check → bed assignment → admission). Augmentation (EMR system integration, insurance API verification, bed management database, nurse paging).

- **Loan origination:** Rules (credit scoring, eligibility checks, risk assessment). Workflow (application → verification → underwriting → approval → disbursement). Augmentation (credit bureau API calls, document storage, email notifications, banking system integration).

- **E-commerce checkout:** Rules (pricing, discounts, tax calculations, fraud detection). Workflow (cart → payment → fulfilment → delivery). Augmentation (payment gateway integration, inventory database updates, shipping label API, tracking notifications).

- **Insurance claims:** Rules (coverage verification, liability assessment, settlement calculation). Workflow (filing → investigation → evaluation → settlement → payment). Augmentation (document scanning, adjuster scheduling, bank transfer processing, regulatory reporting).

The three layers exist in every case. The problem is not that they exist. The problem is that in most codebases, they are inseparably tangled together.

---

## 4.2 The Entanglement Problem

Open any enterprise codebase and you will find the same disease: all three layers — rules, workflow, and augmentation — knotted together so tightly that you cannot change one without risking the others. Here is a patient admission service from a hospital system. See if you can tell where one layer ends and another begins:

```java
public AdmissionResult processArrival(Patient patient, VitalSigns vitals) {

    // ── Triage: determine how urgently the patient needs care ──
    //    (BUSINESS RULE — clinical policy encoded as if/else)
    int priority;
    if (vitals.getHeartRate() > 120 || vitals.getHeartRate() < 40
            || vitals.getSystolicBP() > 180 || vitals.getOxygenSat() < 90) {
        priority = 1;   // Critical — immediate treatment
    } else if (vitals.getTemperature() > 39.0 || vitals.getSystolicBP() > 160
            || vitals.getPainScale() >= 8) {
        priority = 2;   // Urgent — treat within 30 minutes
    } else if (vitals.getTemperature() > 38.0 || vitals.getPainScale() >= 5) {
        priority = 3;   // Semi-urgent — treat within 1 hour
    } else {
        priority = 4;   // Non-urgent — treat when available
    }

    // ── Insurance: verify coverage and decide on pre-authorisation ──
    //    (AUGMENTATION — call to external insurance API)
    InsuranceStatus insurance = insuranceApi.verifyEligibility(
            patient.getInsuranceId(), patient.getPolicyNumber());

    //    (BUSINESS RULE — pre-auth policy encoded as if/else)
    boolean needsPreAuth;
    if (!insurance.isActive()) {
        needsPreAuth = false;          // Self-pay — no auth needed
    } else if (priority <= 2) {
        needsPreAuth = false;          // Emergency — treat first, auth later
    } else if (insurance.getPlanType().equals("Premium")) {
        needsPreAuth = false;          // Premium plans never need pre-auth
    } else {
        needsPreAuth = true;           // Standard plans: must pre-authorise
    }

    // ── Bed assignment: decide which ward and bed type ──
    //    (BUSINESS RULE — bed allocation policy)
    String ward;
    String bedType;
    if (priority == 1) {
        ward = "ICU";
        bedType = "monitored";
    } else if (priority == 2 && patient.getAge() >= 65) {
        ward = "Acute Care - Geriatric";
        bedType = "standard";
    } else if (priority == 2) {
        ward = "Acute Care";
        bedType = "standard";
    } else {
        ward = "General";
        bedType = "standard";
    }

    // ── Find an available bed ──
    //    (AUGMENTATION — query bed management database)
    Bed bed = bedDatabase.findAvailable(ward, bedType);

    //    (WORKFLOW — route based on bed availability)
    if (bed == null) {
        waitlistDb.add(patient, ward, priority);             // AUGMENTATION
        pager.alertBedManager(ward, priority);                // AUGMENTATION
        return AdmissionResult.waitlisted(ward, priority);
    }

    // ── Admit the patient ──
    //    (WORKFLOW — execute the admission sequence)
    if (needsPreAuth) {
        insuranceApi.requestPreAuth(patient, ward);           // AUGMENTATION
    }
    bedDatabase.reserve(bed.getId(), patient.getId());         // AUGMENTATION
    emrSystem.createAdmissionRecord(patient, bed, priority);  // AUGMENTATION
    pager.notifyWardNurse(ward, patient, priority);           // AUGMENTATION

    if (priority <= 2) {
        pager.pageOnCallSpecialist(patient.getChiefComplaint()); // AUGMENTATION
    }

    return AdmissionResult.admitted(bed, ward, priority);
}
```

One method. Three layers. All fused together. The annotations in the comments reveal the problem: BUSINESS RULE, WORKFLOW, and AUGMENTATION alternate line by line, sometimes within the same block.

| Layer | What It Does Here |
|-------|------------------|
| **Business Rules** | Determines triage priority from vital signs. Decides whether pre-authorisation is needed. Assigns the ward and bed type based on priority and patient age. |
| **Business Workflow** | Sequences the steps: triage → insurance check → bed assignment → admission (or waitlisting). Routes the patient to different outcomes based on bed availability and priority. |
| **Technical Augmentation** | Calls the insurance API, queries the bed management database, writes to the EMR system, sends pages to nurses and specialists. |

That fusion hurts in ways that compound over time:

| Problem | Consequence |
|---------|------------|
| **Rules are invisible.** | The triage thresholds (heart rate > 120, oxygen sat < 90) are buried in `if/else` chains. A clinical director cannot review them without reading Java. |
| **Rules are untestable in isolation.** | To test whether a patient with a heart rate of 130 gets priority 1, you need to mock `insuranceApi`, `bedDatabase`, `emrSystem`, and `pager` — and run the entire method. |
| **Workflow changes require rule changes (and vice versa).** | Adding a "pharmacy consultation" step before admission requires editing the same method that contains triage logic. |
| **Augmentation leaks into everything.** | The triage rules are pure logic — they need nothing but vital signs. Yet they live in a method that cannot run without a live connection to the insurance API and bed database. |
| **All three layers change at different cadences.** | Triage thresholds change when clinical guidelines are updated. The admission workflow changes when the hospital adds a department. The EMR integration changes when the hospital switches vendors. Yet all three live in the same method, the same class, the same deployment unit. |

---

## 4.3 The Separation: Three Layers, Three Homes

The fix is architectural, and it fits in one sentence: **give each layer its own home**.

- **Business rules** move to a **decision model** — FEEL expressions, decision tables, or a dedicated rules module. Pure logic. No side effects. No infrastructure.
- **Business workflow** moves to a **state machine** — a process definition that knows which decisions to invoke and what to do with the results. No opinion about how decisions are made.
- **Technical augmentation** moves to **service adapters** — the infrastructure layer that fetches data, sends data, and handles errors. No opinion about business logic or process flow.

Here is the same patient admission logic, pulled apart into its three natural layers:

### Layer 1: The Business Rules (expressed in FEEL)

**Triage Decision:**
```
{
  critical: Heart Rate > 120 or Heart Rate < 40 or
            BP Systolic > 180 or Oxygen Saturation < 90,
  urgent:   Temperature > 39.0 or BP Systolic > 160 or Pain Scale >= 8,
  semi urgent: Temperature > 38.0 or Pain Scale >= 5,

  priority: if critical then 1
            else if urgent then 2
            else if semi urgent then 3
            else 4
}
```

**Pre-Authorisation Decision (decision table, hit policy F):**

| F | Insurance Active | Priority | Plan Type | Pre-Auth Required |
|---|-----------------|----------|-----------|-------------------|
| 1 | false           | -        | -         | false             |
| 2 | true            | <= 2     | -         | false             |
| 3 | true            | -        | "Premium" | false             |
| 4 | true            | -        | -         | true              |

**Bed Assignment Decision (decision table, hit policy F):**

| F | Priority | Patient Age | Ward                     | Bed Type    |
|---|----------|-------------|--------------------------|-------------|
| 1 | 1        | -           | "ICU"                    | "monitored" |
| 2 | 2        | >= 65       | "Acute Care - Geriatric" | "standard"  |
| 3 | 2        | -           | "Acute Care"             | "standard"  |
| 4 | -        | -           | "General"                | "standard"  |

No database calls. No API dependencies. No infrastructure. Just pure logic: inputs in, decisions out. A clinical director can read these tables and verify the triage thresholds without touching a line of Java.

### Layer 2: The Business Workflow (a state machine)

```
   ┌─────────┐
   │ Arrival │
   └────┬────┘
        │
        ▼
   ┌─────────┐     ┌───────────────────────────────┐
   │ Triage  │────▶│ Call: Triage Decision          │
   └────┬────┘     └───────────────────────────────┘
        │
        ▼
   ┌───────────┐     ┌───────────────────────────────┐
   │ Insurance │────▶│ Call: Pre-Auth Decision        │
   └────┬──────┘     └───────────────────────────────┘
        │
        ▼
   ┌────────────┐     ┌───────────────────────────────┐
   │ Assign Bed │────▶│ Call: Bed Assignment Decision  │
   └────┬───────┘     └───────────────────────────────┘
        │
    ┌───┴────┐
    │bed     │
    │found?  │
    └───┬────┘
   yes  │  no
   ┌────┘  └────┐
   ▼            ▼
┌──────────┐  ┌────────────┐
│ Admission│  │ Waitlisted │
└──────────┘  └────────────┘
```

No vital-sign thresholds. No insurance plan types. No ward assignment rules. The workflow knows only that decisions exist, what they return, and which state to transition to next.

### Layer 3: The Technical Augmentation (service adapters)

| Adapter | What It Does | When It Is Called |
|---------|-------------|------------------|
| **Insurance adapter** | Calls the insurance API to verify eligibility and fetch plan details | Before the pre-auth decision is invoked |
| **Bed management adapter** | Queries the bed database to check availability in the assigned ward | After the bed assignment decision returns |
| **EMR adapter** | Creates the admission record in the electronic medical record system | During the admission step |
| **Paging adapter** | Sends pages to ward nurses and on-call specialists | During and after the admission step |
| **Waitlist adapter** | Adds the patient to the ward's waitlist if no bed is available | During the waitlisted step |

The augmentation layer knows nothing about triage thresholds or insurance rules. It fetches data, sends data, and handles errors. That is all.

---

## 4.4 The Workflow That Was Hiding in Your Code

Here is the part most developers do not expect. You do not need BPMN. You do not need a workflow engine. You do not even need a new framework. When you extract the business rules from your code, **the remaining native code itself becomes a readable workflow**.

Look at what happens to the patient admission method when we pull the rules out into a decision service — keeping everything else in plain Java:

```java
public AdmissionResult processArrival(Patient patient, VitalSigns vitals) {

    // ── Step 1: TRIAGE ──────────────────────────────────────
    //    Ask the decision service: how urgent is this patient?
    TriageResult triage = decisions.evaluate("Triage", vitals);

    // ── Step 2: INSURANCE VERIFICATION ──────────────────────
    //    Fetch insurance status, then ask: is pre-auth needed?
    InsuranceStatus insurance = insuranceApi.verify(patient);
    PreAuthResult preAuth = decisions.evaluate("PreAuthorisation",
            insurance, triage.priority());

    // ── Step 3: BED ASSIGNMENT ──────────────────────────────
    //    Ask the decision service: which ward and bed type?
    //    Then check the database for an available bed.
    BedAssignment assignment = decisions.evaluate("BedAssignment",
            triage.priority(), patient.getAge());
    Bed bed = bedDatabase.findAvailable(
            assignment.ward(), assignment.bedType());

    if (bed == null) {
        // → WAITLISTED
        waitlistDb.add(patient, assignment.ward(), triage.priority());
        pager.alertBedManager(assignment.ward(), triage.priority());
        return AdmissionResult.waitlisted(assignment.ward(), triage.priority());
    }

    // ── Step 4: ADMISSION ───────────────────────────────────
    //    Execute the admission: pre-auth if needed, reserve bed,
    //    create medical record, notify staff.
    if (preAuth.required()) {
        insuranceApi.requestPreAuth(patient, assignment.ward());
    }
    bedDatabase.reserve(bed.getId(), patient.getId());
    emrSystem.createAdmission(patient, bed, triage.priority());
    pager.notifyWardNurse(assignment.ward(), patient, triage.priority());
    if (triage.priority() <= 2) {
        pager.pageSpecialist(patient.getChiefComplaint());
    }

    // → ADMITTED
    return AdmissionResult.admitted(bed, assignment.ward(), triage.priority());
}
```

Read that method top to bottom. It is a recipe: **triage**, then **insurance verification**, then **bed assignment**, then **admission**. Each step is a heading you could put on a whiteboard. The `if/else` chains that used to obscure the flow — the vital-sign thresholds, the pre-authorisation eligibility matrix, the ward assignment rules — are gone. What remains *is* the workflow, and it reads like a process diagram turned sideways.

### What Changed, What Didn't

| Aspect | Before (Section 4.2) | After |
|--------|----------------------|-------|
| **Lines of code** | ~55 lines | ~30 lines |
| **`if/else` branches encoding business rules** | 11 (triage, insurance policy, ward assignment) | 0 — all delegated to the decision service |
| **`if/else` branches for workflow routing** | 2 (bed availability, specialist paging) | 2 — unchanged, because these *are* workflow |
| **Business rules in the method** | All of them | None |
| **Readability** | Must trace every `if/else` to understand the flow | Flow is immediately obvious: four labelled steps |
| **Test complexity** | Must mock 4 services and construct full objects to test any rule | Rules test with pure input/output pairs; workflow tests with simple stubs |

Notice how the two remaining `if/else` branches — checking bed availability and deciding whether to page a specialist — are *workflow routing*, not business rules. They belong in the method. The distinction is now clear precisely because the rules are gone.

### The Workflow Was Always There

This is not a coincidence. In most enterprise codebases, 60–80% of what looks like "application code" is actually business rules wearing a disguise. Strip them out and what remains is thin, linear, and obviously a workflow — a sequence of steps with clear routing based on decision outcomes.

The workflow was always there. You just could not see it through the rules and the plumbing.

This is true whether you work in Java, Python, C#, JavaScript, or any other language. The three layers exist in every business codebase. Extracting the rules does not require adopting a new framework or a new platform. It requires recognising the three layers and giving each one its own home — even if that home is just a separate module in the same codebase.

### From Here, You Choose

Once the workflow is visible, you have options:

1. **Keep it as plain code.** The cleaned-up method above is already dramatically better. For simple processes, this is enough.
2. **Promote it to a state machine library.** XState, Spring Statemachine, or Stateless give you formal states and transitions, which helps when the workflow gets more complex.
3. **Move it to a workflow orchestrator.** Temporal, AWS Step Functions, or Apache Airflow give you durability, retries, observability, and audit trails.
4. **Model it in BPMN.** For organisations that need formal, visual, vendor-portable process models, BPMN is the OMG standard.

The pattern is the same in every case: **rules in the decision layer, workflow in the process layer, plumbing in the augmentation layer**. The technology varies. The architecture does not.

---

## 4.5 Why This Separation Matters

### 4.5.1 Each Layer Changes Independently

The three layers change at different speeds, for different reasons, driven by different people:

| Layer | Change Cadence | Changed By | Example |
|-------|---------------|-----------|---------|
| **Business Rules** | Frequently (weekly to quarterly) | Business analysts, clinical directors, product owners | "Update triage thresholds per new clinical guidelines" |
| **Business Workflow** | Occasionally (quarterly to yearly) | Process engineers, developers | "Add a pharmacy consultation step before admission" |
| **Technical Augmentation** | When infrastructure changes | Developers, DevOps | "Switch from Cerner to Epic for EMR integration" |

When the three layers are tangled, every change — no matter which layer it belongs to — requires editing the same code, running the full test suite, and rolling a deployment. When separated, a triage threshold change is a one-cell edit in a decision table. A new workflow step is a new state and transition. An EMR vendor swap is a new adapter. Each change stays in its lane.

### 4.5.2 Decisions Are Testable in Isolation

Each decision is a pure function: inputs in, outputs out. No mocks. No service dependencies. No database. No network. Just data.

```
Input:  { Heart Rate: 130, BP Systolic: 140, Oxygen Saturation: 95,
          Temperature: 37.0, Pain Scale: 3 }
Output: { priority: 1 }
```

You can write hundreds of test cases as simple input/output pairs. You can generate them from the decision table itself. You can run them in milliseconds.

### 4.5.3 Decisions Are Reusable Across Processes

The triage decision is needed by the emergency admission process, the ambulance dispatch system, *and* the telemedicine triage app. Without separation, that logic gets copy-pasted three times and drifts three different ways. With separation, all three call the same decision service. One source of truth. Zero drift.

### 4.5.4 Decisions Are Auditable

A regulator asks "why was this patient assigned to the ICU?" You can trace the answer through the decision in seconds:

1. The triage decision was invoked with inputs `{Heart Rate: 135, Oxygen Saturation: 88, ...}`.
2. The condition `Oxygen Saturation < 90` evaluated to `true`.
3. The patient was classified as priority 1 (Critical).
4. The bed assignment decision mapped priority 1 to ward "ICU", bed type "monitored".

The decision tables *are* the audit trail. No code reading required.

### 4.5.5 The Key Insight

If you take only one idea from this chapter, make it this one:

> **A business workflow only needs to know *about* the business rules.** It does not need to implement them, understand them, or contain them. It needs to know:
>
> 1. **What** decisions exist (by name).
> 2. **What** inputs each decision requires.
> 3. **What** outputs each decision produces.
> 4. **What** to do based on the outputs (routing).
>
> Everything else — the actual logic, the tables, the formulas, the edge cases — is encapsulated in the decision model. And the technical augmentation is encapsulated in the service adapters.

If this sounds familiar, it should — it is the same principle behind every well-designed service interface. The consumer knows the contract (inputs, outputs, guarantees), not the implementation.

When you adopt this separation, each layer becomes simpler:

| Layer | Complexity | Who Owns It |
|-------|-----------|-------------|
| **Business Workflow** (state machine) | Simple: states, transitions, decision invocations | Process engineers / developers |
| **Business Rules** (decision model) | Moderate: tables, expressions, dependencies | Business analysts + developers |
| **Technical Augmentation** (adapters) | Variable: API integrations, data access, error handling | Developers / DevOps |
| **FEEL expressions** | Focused: pure logic, no side effects, independently testable | Developers (with analyst review) |

---

## 4.6 Implementation: With OMG Standards

The OMG (Object Management Group) designed two standards specifically for this separation. If you choose to use them, you get a battle-tested architecture with vendor support and interoperability.

### BPMN for the Workflow Layer

**BPMN (Business Process Model and Notation)** is the OMG standard for modelling workflows. It gives you a visual notation for states, transitions, gateways, events, and task invocations. Every major workflow engine — Camunda, IBM BAW, Oracle BPM, SAP Signavio — implements BPMN.

A BPMN process is a formal state machine. You draw it in a modelling tool. The engine executes it. The process has no opinion about business logic — it only knows which decisions to invoke and what to do with the results.

### DMN for the Rules Layer

**DMN (Decision Model and Notation)** is the OMG standard for modelling decisions. It gives you decision tables, FEEL expressions, and Decision Requirements Graphs. The decisions are authored separately from the process, version-controlled separately, and deployed separately.

### The Integration Point

BPMN and DMN were designed to work together. A BPMN **Business Rule Task** invokes a DMN **Decision Service**:

```
BPMN Process                         DMN Decision Model
┌─────────────────┐                  ┌──────────────────────┐
│                 │                  │                      │
│  ┌───────────┐  │    invokes       │  ┌────────────────┐  │
│  │ Business  │──┼─────────────────▶│  │ Decision       │  │
│  │ Rule Task │  │                  │  │ Service        │  │
│  └───────────┘  │    returns       │  │                │  │
│       │         │◀─────────────────┼──│  Inputs:       │  │
│       ▼         │   {result: ...}  │  │   Patient      │  │
│  ┌───────────┐  │                  │  │   Vitals       │  │
│  │ Gateway   │  │                  │  │  Outputs:      │  │
│  │ (decides  │  │                  │  │   Priority     │  │
│  │  routing) │  │                  │  │   Ward         │  │
│  └───────────┘  │                  │  └────────────────┘  │
│                 │                  │                      │
└─────────────────┘                  └──────────────────────┘
```

The BPMN process (workflow layer) contains no business rules. The DMN model (rules layer) contains no process logic. The technical augmentation is handled by service tasks in the BPMN engine that call external systems.

### When to Choose the OMG Stack

The full BPMN + DMN stack makes sense when:

- Your organisation already uses a BPMN-capable workflow engine (Camunda, KIE, IBM, etc.).
- You need **vendor-neutral portability** — BPMN and DMN models can (in theory) move between engines.
- Business analysts need to **read and modify** both the process and the decision logic directly.
- Compliance or regulatory requirements demand a **formal, auditable** process definition.

---

## 4.7 Implementation: Without OMG Standards

You do not need BPMN or DMN to achieve the three-layer separation. The pattern — workflow in one layer, rules in another, augmentation in a third — works with any technology. Here are four ways to implement it without touching an OMG standard.

### 4.7.1 Plain Code State Machine

The simplest approach: write the workflow as an explicit state machine in your application language. The decisions live in a separate module — functions, classes, or a lightweight rules engine. The augmentation lives in adapter classes or service wrappers.

This is exactly what we showed in Section 4.4: the cleaned-up Java method with `decisions.evaluate()` calls. For more complex processes with branching, retries, and parallel paths, use a state machine library:

| Language | Library | Notes |
|----------|---------|-------|
| Java | Spring Statemachine, Stateless4j | Explicit states and transitions |
| Python | transitions, pytransitions | Lightweight, decorator-based |
| JavaScript | XState | Full statecharts with visualisation |
| C# | Stateless | Fluent API, widely used |
| Go | looplab/fsm | Minimal, idiomatic |

### 4.7.2 Workflow Orchestrators

Modern workflow orchestrators give you a durable, observable state machine without writing state management code yourself. The orchestrator handles retries, timeouts, compensation, and audit trails. You write the steps — and keep the business rules in a separate decision service.

**Temporal (Python):**

```python
@workflow.defn
class PatientAdmissionWorkflow:
    @workflow.run
    async def run(self, patient_data, vitals_data):
        # Step 1: Triage — pure decision, no infrastructure
        triage = await workflow.execute_activity(
            evaluate_triage, vitals_data,
            start_to_close_timeout=timedelta(seconds=30))

        # Step 2: Insurance verification — augmentation + decision
        insurance = await workflow.execute_activity(
            verify_insurance, patient_data,
            start_to_close_timeout=timedelta(seconds=30))
        pre_auth = await workflow.execute_activity(
            evaluate_pre_auth, {"insurance": insurance, "triage": triage},
            start_to_close_timeout=timedelta(seconds=30))

        # Step 3: Bed assignment — decision + augmentation
        assignment = await workflow.execute_activity(
            evaluate_bed_assignment, {"triage": triage, "patient": patient_data},
            start_to_close_timeout=timedelta(seconds=30))
        bed = await workflow.execute_activity(
            find_available_bed, assignment,
            start_to_close_timeout=timedelta(seconds=30))

        if bed is None:
            return {"status": "waitlisted", "ward": assignment["ward"]}

        # Step 4: Admission — augmentation
        await workflow.execute_activity(
            admit_patient, {"patient": patient_data, "bed": bed,
                            "triage": triage, "pre_auth": pre_auth},
            start_to_close_timeout=timedelta(seconds=60))

        return {"status": "admitted", "ward": assignment["ward"],
                "bed": bed["id"], "priority": triage["priority"]}
```

Each activity is either a decision evaluation (rules layer) or a system interaction (augmentation layer). The workflow definition is the workflow layer. Temporal handles persistence, retries, and visibility. You get a production-grade workflow engine without BPMN.

**AWS Step Functions (JSON state machine):**

```json
{
  "StartAt": "Triage",
  "States": {
    "Triage": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123:function:evaluate-triage",
      "Next": "VerifyInsurance"
    },
    "VerifyInsurance": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123:function:verify-insurance",
      "Next": "AssignBed"
    },
    "AssignBed": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123:function:assign-bed",
      "Next": "CheckBedAvailability"
    },
    "CheckBedAvailability": {
      "Type": "Choice",
      "Choices": [{
        "Variable": "$.bed_found",
        "BooleanEquals": true,
        "Next": "AdmitPatient"
      }],
      "Default": "Waitlisted"
    },
    "AdmitPatient": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123:function:admit-patient",
      "Next": "Admitted"
    },
    "Admitted": { "Type": "Succeed" },
    "Waitlisted": { "Type": "Succeed" }
  }
}
```

The Step Function definition *is* the workflow layer — states, transitions, branching. Each Lambda function contains either decision logic (rules layer) or system integration (augmentation layer). The separation is architecturally enforced: you literally cannot put triage thresholds inside the state machine definition.

**Other workflow orchestrators:**

| Orchestrator | Platform | Notes |
|-------------|----------|-------|
| **Temporal** | Any | Open-source, polyglot, durable execution |
| **AWS Step Functions** | AWS | Serverless, JSON state machine definition |
| **Azure Durable Functions** | Azure | Code-first orchestration |
| **Google Cloud Workflows** | GCP | YAML-based, serverless |
| **Apache Airflow** | Any | Python-based, strong in data pipelines |
| **Prefect** | Any | Python-native, modern API |

### 4.7.3 Microservice with Decision Service

Even without a formal workflow engine, the pattern works as separate services: one for the process flow, one (or more) for the decisions, and infrastructure adapters in each.

```
┌──────────────┐     POST /decisions/triage            ┌──────────────────┐
│              │     { heart_rate: 130,                 │                  │
│  Admission   │       oxygen_sat: 88, ... }           │  Decision        │
│  Service     │ ───────────────────────────────────▶   │  Service         │
│              │                                       │  (DMN engine     │
│  (workflow   │     { priority: 1,                     │   or plain code) │
│   layer)     │ ◀───────────────────────────────────   │                  │
│              │       ward: "ICU" }                    │  Returns JSON    │
└──────────────┘                                       └──────────────────┘
```

The decision service can use a DMN engine internally (Camunda feel-scala, Apache KIE) or it can be plain code with a clean API. The calling service does not know or care. It sends facts, gets a verdict, and routes accordingly. You can start with plain code and migrate to a DMN engine later without changing the caller.

### 4.7.4 The Hybrid: DMN Decisions Without BPMN Processes

In practice, the most common architecture is a **hybrid**: use DMN and FEEL for the rules layer (because decision tables and FEEL expressions are genuinely better than `if/else` chains for expressing business rules), but use whatever workflow technology you already have for the process layer — Temporal, Step Functions, a plain code state machine, or even a simple controller method.

This is the approach most of this book assumes. You will learn FEEL not because you are required to adopt the full OMG stack, but because FEEL is the best language available for expressing business logic clearly, testably, and portably. What you use for the workflow and augmentation layers is your choice.

### Choosing Your Approach

| Approach | Best When |
|----------|-----------|
| **BPMN + DMN** (full OMG) | You need formal process models, vendor portability, and business analyst authoring |
| **Workflow orchestrator + DMN** | You want durable execution and observability with clean decision logic |
| **Plain code state machine + DMN** | You want simplicity and full control with clean decision logic |
| **Workflow orchestrator + plain code rules** | You are not ready for DMN yet but want the three-layer separation |
| **Plain code everywhere** | You are starting small and want to migrate incrementally |

The pattern is the same in every case. The technology varies. The architecture does not.

---

## 4.8 From Monolith to Three Layers: A Migration Path

Good news: you do not need to stop the world and rewrite everything. This migration works best when you do it one decision at a time.

### Step 1: See the Three Layers

Go on a hunting expedition through your codebase. Annotate (even just in comments) which lines are **rules**, which are **workflow**, and which are **augmentation** — exactly as we did in Section 4.2. You will be surprised at the ratio: most of the code is rules and plumbing, with the actual workflow barely visible beneath them.

### Step 2: Extract One Decision

Pick the simplest, most self-contained business rule — the low-hanging fruit. Express it as a FEEL expression or decision table. Wrap it in a decision service (even if that "service" is just a function call to a local DMN engine). Get a win on the board.

### Step 3: Replace the Code with a Call

Rip out the original `if/else` block and replace it with a call to your decision service. The calling code still receives the result and acts on it — but it no longer computes it. That distinction matters more than it seems.

### Step 4: Test Side-by-Side

Run both the old code path and the new decision service with the same inputs. Verify that the outputs match. Fix any discrepancies — and there *will* be discrepancies. This is the step where you discover that the old code had bugs nobody knew about.

### Step 5: Repeat and Watch the Workflow Emerge

Extract the next decision. And the next. With each extraction, the remaining code sheds complexity and becomes increasingly process-oriented. Something interesting happens around the third or fourth extraction: you look at the remaining code and realise you can read the workflow from top to bottom like a recipe. The state machine has emerged on its own — not because you designed it, but because it was always there, buried under the rules.

### Step 6: Formalise (Optional)

Once enough decisions have been pulled out, what remains is simple enough to express as a formal state machine — in BPMN, in a workflow orchestrator, or in a state machine library in your language of choice. Or you can leave it as clean, readable code. Either way, the three layers now have their own homes.

### The Result

```
Before:                              After:
┌──────────────────────────┐         ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│                          │         │              │  │              │  │              │
│   Monolithic Service     │         │  Workflow    │  │  Decision    │  │  Adapters    │
│                          │         │  Layer       │  │  Layer       │  │  Layer       │
│   - triage rules         │         │              │  │              │  │              │
│   - insurance rules      │         │  - states    │  │  - FEEL      │  │  - EMR API   │
│   - bed assignment rules │         │  - events    │  │  - tables    │  │  - insurance │
│   - process flow         │         │  - routing   │  │  - formulas  │  │  - bed mgmt  │
│   - EMR integration      │         │  - decisions │  │  - logic     │  │  - paging    │
│   - insurance API        │         │              │  │              │  │              │
│   - paging               │         └──────────────┘  └──────────────┘  └──────────────┘
│   - error handling       │
│                          │         Each layer changes independently.
└──────────────────────────┘         Each layer is testable in isolation.
                                     Each layer is owned by the right team.
```

---

## 4.9 FEEL Contexts as Business State

A FEEL context looks like a data structure, but think of it as something more powerful: a **business state representation**. When a decision service finishes evaluating, the result is a context that captures the complete outcome — what was decided, what the computed values are, and what classification was assigned.

This context *is* the contract between the rules layer and the workflow layer. The state machine does not care how the values were computed. It receives a structured result and routes accordingly.

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

That context *is* the **business state** of the claim after evaluation. The workflow receives it and uses its fields to pick the next state transition:

- If `next action = "auto-approve"` → transition to the Approve state.
- If `next action = "manual-review"` → transition to the Review state.
- If `next action = "decline"` → transition to the Decline state.

The process never inspects `approved amount` or `reason codes` to make its routing decision — those fields exist for downstream consumers (notifications, audit logs, the customer-facing portal). The process only reads the fields it needs for routing.

### Why Contexts, Not Scalars

Sure, a decision *could* return a single scalar — just `"approve"`. But a rich context pays for itself immediately:

| Approach | Limitation |
|----------|-----------|
| Return a single string (`"approve"`) | Downstream systems need additional calls to get the details |
| Return a tuple of values | Position-dependent, fragile, unreadable |
| Return a FEEL context | Self-documenting, extensible, directly serialisable to JSON |

A FEEL context maps naturally to a JSON object, a Java `Map`, a Python `dict`, or a C# `Dictionary`. That makes it the ideal **boundary type** between your decision engine and whatever host application calls it.

### Example: Patient Triage with Business State

**The FEEL decision (returns a context):**

```
{
  critical: Heart Rate > 120 or Heart Rate < 40 or
            BP Systolic > 180 or Oxygen Saturation < 90,
  urgent:   Temperature > 39.0 or BP Systolic > 160 or Pain Scale >= 8,
  semi urgent: Temperature > 38.0 or Pain Scale >= 5,

  priority: if critical then 1
            else if urgent then 2
            else if semi urgent then 3
            else 4,

  ward: if priority = 1 then "ICU"
        else if priority = 2 and Patient Age >= 65 then "Acute Care - Geriatric"
        else if priority = 2 then "Acute Care"
        else "General",

  bed type: if priority = 1 then "monitored" else "standard",

  triage state: {
    priority: priority,
    ward: ward,
    bed type: bed type,
    specialist needed: priority <= 2,
    chief complaint: Patient.Chief Complaint,
    triage timestamp: now()
  }
}.triage state
```

The final `.triage state` projection strips away the working variables and returns only the business-relevant summary — a clean context that the workflow can consume without knowing a thing about how triage priorities, ward assignments, or bed types were determined.

**The workflow receives:**

```json
{
  "priority": 1,
  "ward": "ICU",
  "bed_type": "monitored",
  "specialist_needed": true,
  "chief_complaint": "chest pain",
  "triage_timestamp": "2024-03-15T14:23:00"
}
```

The workflow reads `priority` and `ward` to route the patient. The augmentation layer reads `specialist_needed` to decide whether to page the on-call doctor. The EMR adapter reads the full context to create the admission record. Each consumer reads only the fields it cares about.

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

**The workflow receives:**

```json
{
  "ready": false,
  "missing_documents": ["Tax Form"],
  "equipment_request": "full-workstation",
  "assigned_office": "Building B"
}
```

If `ready` is `false`, the workflow transitions to "Awaiting Documents" and fires off a notification listing `missing_documents`. If `ready` is `true`, it transitions to "Provision Equipment" using `equipment_request` and "Assign Desk" using `assigned_office`. The decision encapsulates *all* the business logic. The workflow just reads and routes — exactly as it should.

---

## 4.10 Real-World Example: Loan Origination (Revisited)

Let's bring this full circle with the loan origination process from Chapter 3. Watch how the three layers separate cleanly:

**The Workflow Layer (BPMN state machine):**

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

**The Rules Layer (DMN decisions):**

- Strategy → decision table (2 inputs, 1 output)
- Bureau Call Type → decision table (1 input, 1 output)
- Eligibility → decision table (3 inputs, 1 output)
- Risk Category → decision table (2 inputs, 1 output)
- Affordability → FEEL context (5 inputs, 1 output)
- Application Risk Score → scorecard decision table (3 inputs, 1 output, C+ hit policy)
- Routing → decision table (4 inputs, 1 output)

**The Augmentation Layer:**

- Credit bureau API integration (fetch applicant credit data)
- Document management service (store and retrieve application documents)
- Notification service (send approval/decline/referral emails)
- Core banking system integration (create loan account on approval)
- Regulatory reporting adapter (submit required disclosures)

The workflow has **7 states** and **6 transitions**. The rules layer has **10 decisions** with complex interrelated logic. The augmentation layer has **5 integrations** with external systems. Yet all three are completely separate. Change the risk scoring model? The workflow does not notice. Add a "verify identity" step to the workflow? The decision logic does not care. Switch notification providers? Neither the workflow nor the rules are affected.

This is the architecture that FEEL enables. The rest of this book teaches you the language that powers the rules layer of that split.

---

## Summary

- Every piece of business-oriented code contains three layers: **business rules** (decisions), **business workflow** (sequencing and routing), and **technical augmentation** (the plumbing that connects logic to the real world).
- In most codebases, these three layers are tangled together, making rules invisible, untestable, and hard to change.
- The solution is to give each layer its own home. The workflow only needs to know *what* decisions exist, *what* they need, and *what* they return.
- When you extract business rules from code, the remaining native code itself becomes a readable workflow — even without adopting a framework or standard. The workflow was always there; you just could not see it through the rules.
- BPMN models the workflow layer. DMN models the rules layer. FEEL is the language inside the rules.
- You do not need OMG standards to achieve this separation: plain code state machines, workflow orchestrators, and microservice architectures all implement the same pattern.
- FEEL contexts serve as **business state representations** — structured results that form the contract between the rules layer and the workflow layer.
- Migration is incremental: extract decisions one at a time, replace code with decision service calls, and watch the workflow emerge.

---

## What Comes Next

You now know *why* FEEL exists (Chapter 1), *what* business rules and business logic are (Chapter 2), *how* to think in decisions (Chapter 3), and *where* FEEL fits in the three-layer architecture (this chapter). The stage is set. Chapter 5 begins with the atoms of FEEL itself: values and types.

---

[Previous: Chapter 3: Thinking in Decisions](chapter-03-thinking-in-decisions.md) | [Next: Chapter 5: Values and Types](../part-3-feel-ground-up/chapter-05-values-and-types.md)
