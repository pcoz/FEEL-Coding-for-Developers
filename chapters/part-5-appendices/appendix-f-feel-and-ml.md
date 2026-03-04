# Appendix F: FEEL and Machine Learning

---

## Where FEEL Meets ML

Machine learning models excel at pattern recognition — predicting churn, classifying images, scoring credit risk. But an ML prediction alone is rarely a business decision. Between a model's output (`churn_probability: 0.73`) and a business action ("offer a 20% retention discount") lies a layer of **business logic**: thresholds, overrides, compliance checks, and routing rules.

FEEL is purpose-built for this layer. It can consume ML model outputs and apply business rules to turn predictions into decisions — transparently, auditibly, and without requiring the business analyst to understand the model internals.

This appendix explores five integration patterns, each with worked examples.

---

## F.1 Pattern: Pre-Processing — Feature Engineering in FEEL

Before data reaches an ML model, it often needs transformation. FEEL can compute derived features from raw business data, ensuring the feature engineering logic is visible and auditable.

### Worked Example: Customer Features for a Churn Model

**Input data:**

```
{
  Customer: {
    Join Date: @"2019-06-15",
    Orders: [
      { Date: @"2024-01-10", Amount: 120 },
      { Date: @"2024-02-22", Amount: 85 },
      { Date: @"2024-06-01", Amount: 200 },
      { Date: @"2024-11-15", Amount: 45 }
    ],
    Support Tickets: [
      { Date: @"2024-03-01", Severity: "High" },
      { Date: @"2024-09-10", Severity: "Low" }
    ],
    Subscription Tier: "Premium"
  }
}
```

**FEEL feature engineering expression:**

```
{
  tenure months: (years and months duration(Customer.Join Date, today()).years * 12)
                 + years and months duration(Customer.Join Date, today()).months,

  total orders: count(Customer.Orders),

  avg order value: if total orders > 0 then mean(Customer.Orders.Amount) else 0,

  days since last order: if total orders > 0
                         then (today() - max(Customer.Orders.Date)).days
                         else null,

  order frequency: if tenure months > 0
                   then total orders / tenure months
                   else 0,

  high severity tickets: count(Customer.Support Tickets[Severity = "High"]),

  ticket ratio: if total orders > 0
                then count(Customer.Support Tickets) / total orders
                else 0,

  is premium: Customer.Subscription Tier = "Premium",

  features: {
    tenure_months: tenure months,
    total_orders: total orders,
    avg_order_value: decimal(avg order value, 2),
    days_since_last_order: days since last order,
    order_frequency: decimal(order frequency, 4),
    high_severity_tickets: high severity tickets,
    ticket_ratio: decimal(ticket ratio, 4),
    is_premium: is premium
  }
}.features
```

**Result (passed to the ML model):**

```json
{
  "tenure_months": 68,
  "total_orders": 4,
  "avg_order_value": 112.50,
  "days_since_last_order": 109,
  "order_frequency": 0.0588,
  "high_severity_tickets": 1,
  "ticket_ratio": 0.5000,
  "is_premium": true
}
```

The feature definitions are readable by a business analyst. If the business decides that "days since last order" should be capped at 365, or that "order frequency" should only count orders in the last 12 months, the change is a single edit to a FEEL expression — no retraining of the model required.

---

## F.2 Pattern: Post-Processing — Applying Business Rules to Model Outputs

This is the most common integration pattern. The ML model produces a score or classification. FEEL expressions apply business rules to determine the actual business action.

### Worked Example: Churn Prediction to Retention Action

**ML model output (received as input data):**

```
{
  Prediction: {
    churn probability: 0.73,
    confidence: 0.89,
    top factors: ["inactivity", "support_tickets", "declining_order_value"]
  },
  Customer: {
    Tier: "Premium",
    Lifetime Value: 8500,
    Account Age Years: 5,
    Has Active Contract: true
  }
}
```

**FEEL post-processing decision table (hit policy F):**

| F | Churn Probability | Customer Tier | Lifetime Value | Has Active Contract | Retention Action | Discount Offer |
|---|-------------------|---------------|----------------|---------------------|-----------------|----------------|
| 1 | >= 0.8 | "Premium" | > 5000 | - | "Personal outreach" | 25 |
| 2 | >= 0.8 | - | > 5000 | true | "Retention call" | 20 |
| 3 | >= 0.8 | - | - | - | "Email campaign" | 15 |
| 4 | [0.5..0.8) | "Premium" | - | - | "Loyalty reward" | 10 |
| 5 | [0.5..0.8) | - | - | - | "Survey + coupon" | 5 |
| 6 | < 0.5 | - | - | - | "No action" | 0 |

**With a confidence gate (FEEL expression wrapping the table):**

```
if Prediction.confidence < 0.7 then {
  action: "Flag for review",
  discount: 0,
  reason: "Low model confidence (" + string(decimal(Prediction.confidence * 100, 1)) + "%)"
}
else Retention Action Decision Table
```

For the input above (`churn_probability: 0.73`, `Tier: "Premium"`), Rule 4 matches → `{action: "Loyalty reward", discount: 10}`.

The business analyst can adjust thresholds, add tiers, or change actions without touching the ML model. The data scientist can retrain the model without touching the business rules. Each side evolves independently.

---

## F.3 Pattern: PMML Integration

**PMML (Predictive Model Markup Language)** is an XML-based format for serialising ML models (logistic regression, decision trees, neural networks, etc.). DMN supports invoking PMML models directly from a boxed expression using an **external function**.

### How It Works

1. Train a model in Python (scikit-learn), R, or any ML framework.
2. Export the model to PMML format (using libraries like `sklearn2pmml`, `r2pmml`, or `nyoka`).
3. Reference the PMML file from a DMN Business Knowledge Model (BKM).
4. Invoke the BKM from a FEEL expression — the engine passes inputs to the PMML model and receives the prediction.

### Worked Example: Credit Score Prediction via PMML

**Step 1: The PMML model** (simplified)

```xml
<PMML xmlns="http://www.dmg.org/PMML-4_4" version="4.4">
  <RegressionModel modelName="CreditScoreModel"
                   functionName="regression"
                   algorithmName="logisticRegression">
    <MiningSchema>
      <MiningField name="income" usageType="active"/>
      <MiningField name="debt_ratio" usageType="active"/>
      <MiningField name="years_employed" usageType="active"/>
      <MiningField name="credit_score" usageType="predicted"/>
    </MiningSchema>
    <!-- model coefficients here -->
  </RegressionModel>
</PMML>
```

**Step 2: DMN BKM referencing the PMML model**

The BKM "Predicted Credit Score" is defined as an external PMML function with parameters `income`, `debt_ratio`, `years_employed`.

**Step 3: FEEL expression that invokes the model and applies business rules**

```
{
  predicted score: Predicted Credit Score(
    income: Applicant.Annual Income,
    debt_ratio: Applicant.Total Debt / Applicant.Annual Income,
    years_employed: Applicant.Employment Years
  ),

  risk band: if predicted score >= 750 then "Excellent"
             else if predicted score >= 650 then "Good"
             else if predicted score >= 550 then "Fair"
             else "Poor",

  max approved amount: if risk band = "Excellent" then 500000
                       else if risk band = "Good" then 250000
                       else if risk band = "Fair" then 100000
                       else 0,

  override: if Applicant.Is Existing Customer and risk band = "Fair"
            then { risk band: "Good", max approved amount: 150000,
                   note: "Existing customer override" }
            else null,

  result: if override != null then override
          else { risk band: risk band, max approved amount: max approved amount,
                 note: "Standard assessment" }
}.result
```

The ML model provides the prediction. FEEL provides the business interpretation. The override rule ("existing customers get bumped up one band") is visible, testable, and changeable without retraining.

### Engine Support for PMML

| Engine | PMML Support |
|--------|-------------|
| Apache KIE (Drools) | Yes — built-in PMML evaluator (kie-pmml) |
| Trisotech | Yes — cloud-based PMML execution |
| Camunda 8 | No — invoke via external task worker |
| feelin / pySFeel | No |

---

## F.4 Pattern: ONNX Integration

**ONNX (Open Neural Network Exchange)** is a format for serialising neural networks and other ML models. DMN 1.5+ supports ONNX as an external function type, similar to PMML.

### Worked Example: Image Classification Result to Business Decision

An ONNX model classifies product images for quality control. FEEL applies the business rules:

```
{
  classification: Quality Model(image: Product.Image),

  label: classification.predicted class,
  confidence: classification.probability,

  disposition: if label = "defective" and confidence >= 0.95 then "Reject"
               else if label = "defective" and confidence >= 0.80 then "Manual inspection"
               else if label = "marginal" then "Manual inspection"
               else if label = "acceptable" and confidence >= 0.90 then "Accept"
               else "Manual inspection",

  result: {
    action: disposition,
    model label: label,
    model confidence: decimal(confidence, 3),
    requires human: disposition = "Manual inspection"
  }
}.result
```

The quality thresholds (`0.95` for auto-reject, `0.90` for auto-accept) are business parameters that can be tuned in the decision table without retraining the neural network.

---

## F.5 Pattern: Hybrid Decision — Combining Rules, Tables, and ML

The most powerful pattern combines all three: FEEL expressions for computation, decision tables for classification, and ML models for prediction — composed in a single decision model.

### Worked Example: Insurance Premium Calculation

**Decision architecture:**

```
                        ┌──────────────────┐
                        │ Premium Decision │
                        └────────┬─────────┘
                                 │
                    ┌────────────┼────────────┐
                    │            │            │
            ┌───────▼──────┐ ┌──▼────────┐ ┌─▼──────────────┐
            │ Base Rate    │ │ Risk      │ │ Claims         │
            │ (table)      │ │ Prediction│ │ History Score  │
            │              │ │ (PMML)    │ │ (FEEL)         │
            └──────────────┘ └───────────┘ └────────────────┘
```

**Base Rate — decision table (hit policy U):**

| U | Vehicle Type | Driver Age | Region | Base Rate |
|---|-------------|-----------|--------|-----------|
| 1 | "Sedan" | [18..25) | "Urban" | 2400 |
| 2 | "Sedan" | [25..65) | "Urban" | 1200 |
| 3 | "Sedan" | [25..65) | "Rural" | 900 |
| 4 | "SUV" | [18..25) | - | 3000 |
| 5 | "SUV" | [25..65) | - | 1500 |
| 6 | "Motorcycle" | - | - | 1800 |
| 7 | - | >= 65 | - | 1600 |

**Risk Prediction — PMML model** (returns `risk_score` between 0 and 1)

**Claims History Score — FEEL expression:**

```
{
  claims last 3y: Driver.Claims[Date >= today() - @"P3Y"],
  claim count: count(claims last 3y),
  total claimed: sum(claims last 3y.Amount),
  at fault count: count(claims last 3y[At Fault = true]),

  score: 1 + (claim count * 0.15)
           + (at fault count * 0.25)
           + (if total claimed > 10000 then 0.3 else 0)
}.score
```

**Premium Decision — FEEL composition:**

```
{
  base: Base Rate,
  risk: Risk Prediction(
    age: Driver.Age,
    years_licensed: Driver.Years Licensed,
    violations: count(Driver.Violations),
    zip_code: Driver.Address.Zip
  ).risk_score,
  claims: Claims History Score,

  risk multiplier: 1 + (risk * 0.5),
  claims multiplier: claims,
  loyalty discount: if Driver.Years With Us > 5 then 0.90
                    else if Driver.Years With Us > 3 then 0.95
                    else 1.0,

  raw premium: base * risk multiplier * claims multiplier * loyalty discount,
  final premium: max([round half up(raw premium, 0), 300]),

  breakdown: {
    base rate: base,
    risk adjustment: decimal((risk multiplier - 1) * 100, 1),
    claims adjustment: decimal((claims multiplier - 1) * 100, 1),
    loyalty discount: decimal((1 - loyalty discount) * 100, 1),
    annual premium: final premium,
    monthly premium: round half up(final premium / 12, 2)
  }
}.breakdown
```

**Result:**

```json
{
  "base_rate": 1200,
  "risk_adjustment": 18.5,
  "claims_adjustment": 15.0,
  "loyalty_discount": 5.0,
  "annual_premium": 1463,
  "monthly_premium": 121.92
}
```

Every component of the premium is traceable:
- The base rate came from the decision table (rule 2).
- The risk adjustment came from the ML model (37% risk score × 0.5 multiplier).
- The claims adjustment came from the FEEL expression (1 claim, no at-fault → 1.15).
- The loyalty discount came from a simple FEEL conditional (4 years → 5%).

A regulator can audit each component independently. A data scientist can retrain the risk model without affecting the base rates. A business analyst can adjust the loyalty discount tiers without touching the model.

---

## F.6 Taking the Noise Out of ML

One of the most underappreciated benefits of separating business rules from ML models is **noise reduction** — in the data, in the model, and in the process.

### The Noise Problem

When business rules are baked into the training data or the model itself, the model learns things it should not need to learn:

**Example: A fraud detection model trained on raw transaction data.**

The training data includes transactions that were flagged as fraudulent *because of a business rule* — e.g., "any transaction over $10,000 from a new account is automatically flagged." The model learns this threshold as a pattern. But the threshold is not a statistical signal — it is a policy decision. If the policy changes from $10,000 to $15,000, the model must be retrained, even though the underlying fraud patterns have not changed.

**With FEEL separation:**

```
{
  // Business rule: applied BEFORE the model
  auto flag: Transaction.Amount > Policy.Auto Flag Threshold
             and Account.Age Days < Policy.New Account Days,

  // Only transactions NOT caught by the rule go to the model
  needs model: not(auto flag),

  // Model only predicts on ambiguous cases
  model prediction: if needs model
                    then Fraud Model(
                      amount: Transaction.Amount,
                      merchant category: Transaction.MCC,
                      distance from home: Transaction.Distance,
                      time of day: Transaction.Time.hour
                    ).fraud probability
                    else null,

  result: {
    flagged: auto flag or (model prediction != null and model prediction >= 0.7),
    source: if auto flag then "rule" else "model",
    confidence: if auto flag then 1.0 else model prediction
  }
}.result
```

### What Changes

| Without FEEL (rules baked into ML) | With FEEL (rules separated) |
|-------------------------------------|----------------------------|
| Model learns business thresholds as patterns | Model learns only statistical patterns |
| Threshold change → retrain model | Threshold change → edit FEEL expression |
| Model accuracy includes "easy" rule-based cases, inflating metrics | Model accuracy reflects only genuinely ambiguous cases |
| Audit question: "why was this flagged?" → opaque model explanation | "why?" → "rule: amount > $10K from new account" or "model: 83% fraud probability" |
| Training data contains policy artefacts | Training data is cleaner: only cases the model should learn from |

### The Signal-to-Noise Ratio

By extracting known business rules into FEEL, you improve the **signal-to-noise ratio** of your ML pipeline at every stage:

1. **Training data is cleaner.** Cases that are deterministic (decided by rules) are removed from the training set. The model only sees cases where statistical learning adds value.

2. **The model is simpler.** It does not need to learn obvious thresholds or policy-driven classifications. It focuses on the genuinely uncertain cases — the ones where ML actually helps.

3. **Predictions are more meaningful.** A fraud score of 0.8 now means "the model thinks this is likely fraud based on behavioural patterns," not "the transaction is over $10,000."

4. **Explanations are precise.** When a decision is made by a rule, the explanation is exact ("amount exceeded threshold"). When it is made by the model, the explanation is a model interpretation (SHAP values, feature importance). The two never get conflated.

5. **Changes are targeted.** Policy changes touch FEEL. Model improvements touch the ML pipeline. Neither forces a change in the other.

### Worked Example: Loan Approval with Clean ML

**Without separation (noisy):**

An ML model is trained to predict loan approval. The training data includes historical decisions that were made partly by policy ("reject all applicants under 18") and partly by judgement. The model learns that `age < 18 → reject` as a strong feature, even though this is not a statistical pattern — it is a legal requirement.

**With separation (clean):**

```
{
  // Hard rules — FEEL (not learned, just enforced)
  meets age requirement: Applicant.Age >= 18,
  meets income floor: Applicant.Annual Income >= 12000,
  not sanctioned: not(list contains(Sanctions List, Applicant.ID)),

  eligible for model: meets age requirement and meets income floor and not sanctioned,

  // Soft assessment — ML model (only for eligible applicants)
  model assessment: if eligible for model
                    then Credit Model(
                      income: Applicant.Annual Income,
                      employment years: Applicant.Employment Years,
                      debt ratio: Applicant.Debt / Applicant.Annual Income,
                      payment history score: Applicant.Payment History Score
                    )
                    else null,

  // Business rules on top of model output — FEEL
  decision: if not(eligible for model) then {
    approved: false,
    reason: if not(meets age requirement) then "Below minimum age"
            else if not(meets income floor) then "Below income floor"
            else "Sanctions match"
  }
  else if model assessment.approval probability >= 0.75 then {
    approved: true,
    reason: "Auto-approved by model",
    rate: model assessment.suggested rate
  }
  else if model assessment.approval probability >= 0.50 then {
    approved: null,
    reason: "Referred for manual review",
    model score: model assessment.approval probability
  }
  else {
    approved: false,
    reason: "Model assessment below threshold"
  }
}.decision
```

The model never sees applicants under 18 or below the income floor — those are rule-based rejections. The model is trained only on applicants who pass the eligibility rules, so it learns the patterns that matter: income stability, debt management, payment behaviour. The result is a cleaner model, a faster training cycle, and a transparent decision trail.

---

## F.7 Anti-Patterns: When NOT to Use FEEL for ML

| Scenario | Why Not FEEL |
|----------|-------------|
| Training an ML model | FEEL is an expression language, not a training framework. Use Python/R/Julia. |
| Real-time inference on millions of rows | FEEL engines are not optimised for batch scoring. Use the model runtime directly. |
| Complex tensor operations | FEEL has no matrix/tensor types. Use NumPy/PyTorch. |
| Feature engineering requiring database joins | Use SQL or a feature store. FEEL operates on in-memory data. |
| Model selection and hyperparameter tuning | Use MLOps tools (MLflow, Kubeflow). |

FEEL's sweet spot in the ML pipeline is the **last mile**: taking a model's output and applying the business rules that turn a prediction into a decision. That last mile is where transparency, auditability, and business ownership matter most — and where FEEL excels.

---

[Previous: Appendix E: Glossary](appendix-e-glossary.md)
