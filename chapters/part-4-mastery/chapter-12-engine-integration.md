# Chapter 12: FEEL in the Wild — Engine Integration

> *"FEEL is a standard. Your engine is an implementation."*

---

## 12.1 Camunda 8 (feel-scala)

Camunda 8 is the most widely used open-source platform that embeds FEEL. Its FEEL engine is **feel-scala**, a Scala-based interpreter running on the JVM.

### Where FEEL Appears in Camunda 8

| Location | Purpose | Example |
|----------|---------|---------|
| BPMN Service Task input/output mappings | Transform data | `= Order.Customer.Name` |
| BPMN Gateway conditions | Route flow | `= totalAmount > 1000` |
| DMN Decision Tables | Business rules | Unary tests and output expressions |
| DMN Literal Expressions | Computed values | `= Income - Expenses` |
| Camunda Forms | Dynamic validation | `= Age >= 18` |
| Timer expressions | Schedule | `= @"PT30M"` |

The `=` prefix signals to Camunda that the value is a FEEL expression (not a static string).

### Using feel-scala as a Library

```xml
<!-- Maven dependency -->
<dependency>
  <groupId>org.camunda.feel</groupId>
  <artifactId>feel-engine</artifactId>
  <version>1.17.0</version>
</dependency>
```

```java
import org.camunda.feel.FeelEngine;
import org.camunda.feel.api.*;

var engine = new FeelEngine();
var context = Map.of("x", 10, "y", 20);
var result = engine.evaluateExpression("x + y", context);
// result = Right(30)
```

### Debugging with the FEEL Playground

Camunda Modeler includes an integrated FEEL Playground:

1. Open Camunda Modeler.
2. Navigate to the FEEL expression you want to test.
3. Click the "play" icon to open the playground.
4. Enter test context data as JSON.
5. See the result (or error) immediately.

The standalone web playground is available at `camunda.github.io/feel-scala/docs/playground/`.

---

## 12.2 Apache KIE (Drools)

Apache KIE (formerly Red Hat Drools) provides a full DMN runtime with its own FEEL interpreter, written in Java.

### Key Differences from Camunda

| Feature | Camunda (feel-scala) | KIE (Drools) |
|---------|---------------------|-------------|
| Implementation language | Scala | Java |
| DMN TCK pass rate | ~95% | ~100% |
| FEEL error reporting | Returns `null` + warning | May throw exceptions in some modes |
| `date and time` without TZ | Allowed | May normalise to UTC |
| Custom functions | Via SPI mechanism | Via `DMNRuntime` API |

### Embedding in Spring Boot

```xml
<dependency>
  <groupId>org.kie</groupId>
  <artifactId>kie-dmn-core</artifactId>
  <version>9.44.0.Final</version>
</dependency>
```

```java
import org.kie.dmn.api.*;

KieServices ks = KieServices.Factory.get();
KieContainer kc = ks.getKieClasspathContainer();
DMNRuntime runtime = kc.newKieSession().getKieRuntime(DMNRuntime.class);
DMNModel model = runtime.getModel("namespace", "modelName");
DMNContext ctx = runtime.newContext();
ctx.set("Applicant Age", 35);
ctx.set("Credit Score", 720);
DMNResult result = runtime.evaluateAll(model, ctx);
```

---

## 12.3 JavaScript Implementations

### feelin (by Nico Rehwaldt)

Lightweight, browser-ready FEEL parser and interpreter:

```javascript
import { evaluate } from 'feelin';

const result = evaluate('sum([1, 2, 3]) + 4', {});
// result = 10
```

Useful for: client-side FEEL evaluation, form validation, lightweight DMN tooling.

Limitation: Does not implement the full FEEL specification (partial CL3).

### dmn-eval-js

S-FEEL plus selected FEEL features:

```javascript
const { decisionTable } = require('@hbtgmbh/dmn-eval-js');
const decisions = decisionTable.parseDmnXml(xmlString);
const result = decisionTable.evaluateDecision('MyDecision', decisions, context);
```

---

## 12.4 Python

### pySFeel

Implements S-FEEL (the simple subset):

```python
from pySFeel import SFeelParser
parser = SFeelParser()
result = parser.sFeelParse("3 + 4")
# result = (True, 7)
```

For full FEEL, use a JVM-based engine via subprocess or REST API.

### REST API Pattern

Most engines expose DMN evaluation via HTTP:

```python
import requests

response = requests.post(
    "http://localhost:8080/engine/decisions/evaluate",
    json={
        "decision_id": "risk-assessment",
        "variables": {
            "age": 35,
            "income": 75000,
            "credit_score": 720
        }
    }
)
result = response.json()
# {"risk_category": "Low", "approved": True}
```

---

## 12.5 Returning FEEL Contexts as Business State

When integrating FEEL with a host language, decision results are not limited to simple scalars. The most powerful pattern is to **return a FEEL context** — a structured result that represents the complete business state after evaluation. This context maps directly to the native data structures of every mainstream language.

### FEEL Context to Host Language Mapping

| FEEL Type | Java | JavaScript | Python | C# |
|-----------|------|-----------|--------|-----|
| Context `{a: 1, b: "x"}` | `Map<String, Object>` | `Object` / `{}` | `dict` | `Dictionary<string, object>` |
| List `[1, 2, 3]` | `List<Object>` | `Array` | `list` | `List<object>` |
| Number `42` | `BigDecimal` | `number` | `Decimal` or `float` | `decimal` |
| String `"hello"` | `String` | `string` | `str` | `string` |
| Boolean `true` | `Boolean` | `boolean` | `bool` | `bool` |
| Date `@"2024-03-15"` | `LocalDate` | `Date` or string | `date` | `DateTime` |
| Null `null` | `null` | `null` | `None` | `null` |

### Java: Receiving a FEEL Context

```java
// The FEEL expression returns a context (business state)
String feelExpression = """
    {
      eligible: Applicant.Age >= 18 and Credit Score >= 600,
      risk: if Credit Score >= 750 then "Low"
            else if Credit Score >= 650 then "Medium"
            else "High",
      max loan: if risk = "Low" then 500000
                else if risk = "Medium" then 200000
                else 50000,
      decision: {
        approved: eligible and risk != "High",
        category: risk,
        limit: max loan,
        conditions: if risk = "Medium" then ["requires-guarantor"]
                    else []
      }
    }.decision
    """;

var engine = new FeelEngine();
var context = Map.of(
    "Applicant", Map.of("Age", 35, "Name", "Alice"),
    "Credit Score", 720
);

// Result is a Map representing the FEEL context
Map<String, Object> result = (Map<String, Object>) engine.evaluateExpression(
    feelExpression, context
).right().get();

// Access fields like any Map
boolean approved = (Boolean) result.get("approved");   // true
String category = (String) result.get("category");      // "Medium"
BigDecimal limit = (BigDecimal) result.get("limit");    // 200000
List<String> conditions = (List<String>) result.get("conditions"); // ["requires-guarantor"]

// Pass the business state to the next step in the process
if (approved) {
    loanService.createOffer(applicantId, limit, conditions);
} else {
    notificationService.sendRejection(applicantId, category);
}
```

### JavaScript: Receiving a FEEL Context

```javascript
import { evaluate } from 'feelin';

const context = {
  Order: {
    Items: [
      { SKU: "A100", Quantity: 2, "Unit Price": 25.00 },
      { SKU: "B200", Quantity: 1, "Unit Price": 75.00 }
    ]
  },
  Customer: { Tier: "Gold" }
};

// FEEL returns a context — JS receives a plain object
const result = evaluate(`{
  subtotal: sum(for item in Order.Items return item.Quantity * item["Unit Price"]),
  discount: if Customer.Tier = "Gold" then 0.15 else 0,
  total: subtotal * (1 - discount),
  summary: {
    amount: total,
    items: count(Order.Items),
    discount applied: discount > 0
  }
}.summary`, context);

// result is a plain JavaScript object
console.log(result);
// { amount: 106.25, items: 2, "discount applied": true }

// Use directly in the application
updateCartDisplay(result.amount, result["discount applied"]);
```

### Python: Receiving a FEEL Context via REST

```python
import requests

# Send decision inputs
response = requests.post(
    "http://localhost:8080/engine/decisions/evaluate",
    json={
        "decision_id": "claim-assessment",
        "variables": {
            "Claim": {
                "Amount": 12000,
                "Type": "Property",
                "Date": "2024-03-15"
            },
            "Policy": {
                "Coverage Limit": 50000,
                "Deductible": 500,
                "Status": "Active"
            }
        }
    }
)

# The FEEL decision returned a context — Python receives a dict
result = response.json()
# {
#   "valid": True,
#   "net_amount": 11500,
#   "within_limit": True,
#   "risk_flag": "none",
#   "action": "auto-approve",
#   "audit_trail": ["policy-active", "under-limit", "standard-claim"]
# }

# Use the business state to drive the process
if result["action"] == "auto-approve":
    approve_claim(claim_id, result["net_amount"])
elif result["action"] == "manual-review":
    queue_for_review(claim_id, result["risk_flag"])
else:
    reject_claim(claim_id, result["audit_trail"])
```

### The Key Insight

The FEEL context is the **contract** between the decision layer and the application layer. It serves the same role as a response DTO or an API schema — but it is defined by the business rules, not by the application code. When the business rules change (e.g., a new risk category is added), the context shape may expand, but existing consumers that only read the fields they need are unaffected.

This is why FEEL contexts are the natural boundary type for decision services: they are structured, self-documenting, and map to native data structures in every mainstream language without transformation.

---

## 12.6 Testing FEEL Expressions

### Unit Testing Individual Expressions

Treat each FEEL expression as a pure function. Test with input/output pairs:

```java
@Test
void testEligibility() {
    var engine = new FeelEngine();

    // Eligible case
    var ctx1 = Map.of("Age", 30, "Income", 50000, "CreditScore", 720);
    assertEquals("Eligible", engine.evaluate("eligibility.feel", ctx1));

    // Ineligible case
    var ctx2 = Map.of("Age", 16, "Income", 50000, "CreditScore", 720);
    assertEquals("Not Eligible", engine.evaluate("eligibility.feel", ctx2));

    // Null input
    var ctx3 = Map.of("Age", null, "Income", 50000, "CreditScore", 720);
    assertNull(engine.evaluate("eligibility.feel", ctx3));
}
```

### Integration Testing Decision Services

Test the full DMN model end-to-end:

```java
@Test
void testLoanOrigination() {
    var result = dmnRuntime.evaluateDecisionService(
        "RoutingDecisionService",
        Map.of(
            "Applicant", applicantData,
            "RequestedProduct", productData,
            "BureauData", bureauData
        )
    );
    assertEquals("Accept", result.get("Routing"));
}
```

### DMN TCK Test Format

The DMN Technology Compatibility Kit provides tests as XML files:

```xml
<testCases xmlns="http://www.omg.org/spec/DMN/20160901/testcase">
  <testCase id="001">
    <inputNode name="Age">25</inputNode>
    <inputNode name="CreditScore">720</inputNode>
    <resultNode name="Eligibility">
      <expected>
        <value>Eligible</value>
      </expected>
    </resultNode>
  </testCase>
</testCases>
```

### CI/CD Integration

1. Store `.dmn` files and test cases in version control alongside application code.
2. Run FEEL/DMN tests as part of the build pipeline.
3. Fail the build if any test case produces an unexpected result.
4. Track decision coverage: are all rules in each decision table exercised by at least one test?

---

## Summary

| Engine | Language | Best For |
|--------|----------|----------|
| **Camunda 8 (feel-scala)** | Scala/JVM | Production BPMN+DMN workflows |
| **Apache KIE (Drools)** | Java | Full spec conformance, embedded rule engines |
| **feelin** | JavaScript | Browser-based evaluation, lightweight tooling |
| **pySFeel** | Python | Simple FEEL evaluation in Python |
| **REST API** | Any | Language-agnostic integration |

---

## What Comes Next

Chapter 13 covers the FEEL dialects: S-FEEL (the restricted subset), B-FEEL (the business-friendly variant), and how they relate to full FEEL.
