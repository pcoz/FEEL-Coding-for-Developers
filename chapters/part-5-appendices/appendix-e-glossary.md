# Appendix E: Glossary

---

**Any** — The top type in FEEL's type lattice. Every value conforms to `Any`.

**BKM (Business Knowledge Model)** — A reusable piece of decision logic in DMN. Think of it as a named function that can be invoked by multiple decisions.

**Boxed Expression** — A graphical notation for FEEL expressions in DMN. Types include: literal, context, list, relation, function definition, invocation, decision table, conditional, filter, and iterator.

**BPMN (Business Process Model and Notation)** — An OMG standard for modelling business processes. BPMN models the flow; DMN models the decisions within that flow.

**B-FEEL (Business FEEL)** — A dialect of FEEL introduced in DMN 1.6 that uses two-valued logic and replaces error nulls with type-appropriate defaults (0 for numbers, "" for strings, false for booleans).

**CL1, CL2, CL3 (Conformance Levels)** — The three levels of DMN conformance. CL1: decision tables with simple literals. CL2: decision tables with S-FEEL. CL3: full FEEL.

**Context** — A FEEL data structure consisting of ordered key-value pairs. Entries are evaluated sequentially and can reference earlier entries.

**Decision** — A DRG element that determines an output from given inputs using decision logic (typically a boxed expression).

**Decision Requirements Diagram (DRD)** — A graphical representation of the dependencies between decisions, input data, BKMs, and knowledge sources.

**Decision Requirements Graph (DRG)** — The underlying graph structure that one or more DRDs depict. A DRD is a view of the DRG.

**Decision Service** — A packaged subset of a decision model that can be invoked as a service. Defines input data, output decisions, and encapsulated decisions.

**Decision Table** — A tabular representation of decision logic. Rows are rules; columns are inputs and outputs. Has a hit policy that determines how multiple matching rules are handled.

**DMN (Decision Model and Notation)** — An OMG standard for modelling, documenting, and automating business decisions. Includes graphical notation (DRDs), decision tables, and FEEL.

**DMN TCK (Technology Compatibility Kit)** — A test suite that verifies whether an engine correctly implements the DMN specification.

**DRG (Decision Requirements Graph)** — See Decision Requirements Graph.

**Duration** — A FEEL type representing a span of time. Two kinds: *days and time duration* (`@"P2DT3H"`) and *years and months duration* (`@"P1Y6M"`).

**FEEL (Friendly Enough Expression Language)** — The expression language defined as part of the DMN standard. Side-effect free, three-valued logic, designed for dual readership by business analysts and developers.

**Filter Expression** — A FEEL expression of the form `list[condition]` that selects elements satisfying the condition.

**Hit Policy** — A decision table property that determines how matching rules produce the output. Single-hit: U, A, F, P. Multiple-hit: C, C+, C#, C<, C>, R, O.

**Information Requirement** — A dependency between two DRG elements indicating that one requires information from the other.

**Input Data** — External facts provided to a decision model (e.g., customer data, sensor readings).

**Knowledge Requirement** — A dependency indicating that a decision or BKM requires the logic of a BKM.

**Knowledge Source** — An authority for business knowledge: a policy document, regulation, domain expert, or analytical model.

**Null** — The FEEL value representing "no value", "missing data", or "error occurred". Also the bottom type in the type lattice.

**OMG (Object Management Group)** — The standards body that maintains DMN, BPMN, UML, and FEEL. An international, non-profit technology standards consortium.

**Path Expression** — A FEEL expression of the form `a.b` that navigates into a context or extracts a property from a temporal value.

**Qualified Name (QN)** — A dot-separated name path used to reference nested values: `Applicant.Address.City`.

**Range** — A FEEL type representing a bounded interval. Syntax: `[1..10]`, `(0..1)`, `>= 5`. Endpoints can be inclusive or exclusive.

**Relation** — A boxed expression representing a list of same-shape contexts, displayed as a table.

**S-FEEL (Simple FEEL)** — A restricted subset of FEEL sufficient for simple decision tables. Supports arithmetic, comparisons, and simple values, but not lists, contexts, functions, or iteration.

**Scope** — The set of name bindings available to a FEEL expression, modelled as an ordered list of contexts (context stack).

**Ternary Logic** — A logic system with three truth values: `true`, `false`, and `null`. Used by FEEL and SQL. `true and null` is `null`, not `false`.

**Type Conformance** — The relation `T <: S` meaning "an instance of type T can be used where type S is expected". Determined by the type lattice.

**Type Equivalence** — The relation `T ≡ S` meaning "types T and S are interchangeable in all contexts".

**Unary Test** — A special FEEL expression used in decision table input cells. Tests a condition against an implicit value. Examples: `>= 18`, `[1..10]`, `"Gold","Silver"`, `-`.
