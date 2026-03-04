# Appendix D: Engine Compatibility Matrix

---

## FEEL Engine Comparison

| Feature | Camunda 8 (feel-scala) | Apache KIE (Drools) | Trisotech | feelin (JS) | pySFeel (Python) |
|---------|----------------------|--------------------|-----------|-----------|----|
| **Language** | Scala/JVM | Java | Cloud | JavaScript | Python |
| **License** | Apache 2.0 | Apache 2.0 | Commercial | MIT | MIT |
| **DMN Conformance Level** | CL3 | CL3 | CL3 | Partial CL3 | CL2 (S-FEEL) |
| **DMN TCK Pass Rate** | ~95% | ~100% | ~100% | ~70% | N/A |

---

## Feature Support

| FEEL Feature | Camunda 8 | KIE | Trisotech | feelin |
|-------------|-----------|-----|-----------|--------|
| Numbers (Decimal128) | Yes | Yes | Yes | Yes (JS floats) |
| Strings | Yes | Yes | Yes | Yes |
| Booleans (3VL) | Yes | Yes | Yes | Yes |
| Date/Time/Duration | Yes | Yes | Yes | Yes |
| `@"..."` literal syntax | Yes | Yes | Yes | Yes |
| Lists | Yes | Yes | Yes | Yes |
| Contexts | Yes | Yes | Yes | Yes |
| Ranges | Yes | Yes | Yes | Yes |
| `for ... return` | Yes | Yes | Yes | Yes |
| `some/every ... satisfies` | Yes | Yes | Yes | Yes |
| User-defined functions | Yes | Yes | Yes | Partial |
| Recursive functions | Yes | Yes | Yes | No |
| External Java functions | Yes | Yes | N/A | N/A |
| External PMML | No | Yes | Yes | No |
| External ONNX | No | No | Yes | No |
| `context put()` | Yes | Yes | Yes | No |
| `context merge()` | Yes | Yes | Yes | No |
| `string join()` | Yes | Yes | Yes | Partial |
| `list replace()` | Yes | Yes | Yes | No |
| `round half up/down()` | Yes | Yes | Yes | No |
| `now()` / `today()` | Yes | Yes | Yes | Yes |
| `assert()` | Yes | No | No | No |
| `get or else()` | Yes | No | No | No |
| B-FEEL (DMN 1.6) | Planned | No | Planned | No |
| Java-style comments | Yes | Yes | Yes | No |
| `partial` in `for` loops | Yes | Yes | Yes | No |
| Descendant expression (`...`) | Yes | Yes | Yes | No |

---

## Known Behavioural Differences

| Scenario | Camunda 8 | KIE |
|----------|-----------|-----|
| `date and time` without timezone | Treated as local | May normalise to UTC |
| Division by zero | Returns `null` | Returns `null` |
| Recursive function depth | Stack-dependent | Stack-dependent |
| Unrecognised function name | Returns `null` | May throw exception |
| FEEL error reporting | `null` + optional warning | `null` or exception (mode-dependent) |
| `is defined()` built-in | Supported | Not supported |
| `get or else()` built-in | Supported | Not supported |

---

## DMN Technology Compatibility Kit (TCK)

- Repository: `github.com/dmn-tck/tck`
- Contains test cases for all DMN conformance levels
- Each test case specifies inputs, the decision to evaluate, and expected outputs
- Running the TCK against your engine is the most objective measure of conformance

### Running the TCK

```bash
git clone https://github.com/dmn-tck/tck.git
cd tck
# Follow engine-specific instructions in the runner directory
```

---

## Choosing an Engine

| If you need... | Consider |
|---------------|----------|
| Full BPMN+DMN workflow platform | Camunda 8 |
| Maximum spec conformance | Apache KIE (Drools) |
| Cloud-native, GUI modelling | Trisotech |
| Browser-side FEEL evaluation | feelin |
| Python integration | REST API to JVM engine, or pySFeel for simple cases |
| Embedded library in Java/Scala | feel-scala or kie-dmn-core |
