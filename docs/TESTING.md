# ArrowBound Testing Strategy

## Goal

ArrowBound's value depends on deterministic schema generation, predictable compatibility behavior, boundary validation, and metadata preservation. The test suite should therefore be systematic and matrix-driven rather than relying mainly on example-based happy paths.

## 1. Default mapping tests

Every documented Python default must have exact schema assertions.

For each mapping, test resulting `pyarrow.DataType`, field nullability, nested structure where applicable, default behavior independent of runtime values, and stable output across equivalent model definitions.

## 2. Explicit Arrow type tests

Every first-class `Arrow.*` helper should be tested for:

- returning the same effective `pyarrow.DataType` as the corresponding `pa.*` factory,
- valid Python/Pydantic compatibility,
- invalid compatibility rejection,
- parameter preservation,
- no silent type substitution.

Examples should assert equivalence such as:

```python
Arrow.int32() == pa.int32()
Arrow.timestamp("us") == pa.timestamp("us")
```

## 3. Direct `pyarrow.DataType` tests

Direct PyArrow annotations should verify that:

- supported built-in datatypes are accepted without ArrowBound wrapping,
- `Arrow.*` and `pa.*` forms produce equivalent schemas,
- new built-in datatypes can flow through generically where safe,
- incompatible Python annotations fail clearly,
- unsupported custom extension behavior is explicit,
- direct types preserve their exact parameters in the resulting schema.

## 4. Ingress validation tests

Every portable Pydantic constraint should be tested as an actual ArrowBound validation boundary, not only as metadata extraction.

For each applicable constraint, verify:

```text
invalid Python input
        ↓
ArrowBound model validation
        ↓
validation failure
```

and:

```text
valid Python input
        ↓
ArrowBound model validation
        ↓
validated model value
        ↓
constraint preserved in ArrowBound metadata
```

These tests establish the key promise that data entering through ArrowBound model validation satisfies the declared applicable constraints at that boundary.

Tests must also document the limit of that guarantee: arbitrary Arrow arrays/tables created or mutated outside ArrowBound model validation are not assumed valid merely because they use an ArrowBound-bearing schema.

## 5. Constraint preservation tests

Every supported Pydantic declarative constraint should test the full representation pipeline:

```text
Pydantic declaration
        ↓
constraint extraction
        ↓
normalized ArrowBound semantic
        ↓
canonical metadata serialization
        ↓
Arrow field/schema
        ↓
IPC round trip
        ↓
identical portable metadata
```

Test combinations and interactions, not only individual constraints.

## 6. Arrow-native semantic tests

Verify that ArrowBound does not duplicate native Arrow schema semantics into ArrowBound constraint metadata.

Examples include optionality through Arrow nullability, decimal precision/scale in datatype parameters, timestamp timezone/unit in datatype parameters, list element types in datatype structure, and dictionary semantics in datatype structure.

## 7. Nested and advanced type tests

Systematically cover nested structs, deeply nested lists/structs/maps, fixed-size collections, list views, dense/sparse unions, dictionary encoding, run-end encoding, decimal families, temporal variants, canonical extension types, and combinations of advanced types.

Nested failure errors must report full field paths.

## 8. Runtime compatibility tests

CI should test at least the minimum supported PyArrow, latest stable PyArrow, and pre-release/nightly PyArrow where practical.

Version-specific tests should verify capabilities present in the installed runtime are usable; capabilities missing in older runtimes produce useful diagnostics when ArrowBound detects them; and ArrowBound does not rely solely on version-number comparisons when capability introspection is available.

## 9. Capability audit tests

The capability registry and audit should be testable artifacts. Tests should detect built-in PyArrow datatypes with no ArrowBound classification, documented `Arrow.*` helpers missing tests, registry entries whose runtime factories no longer exist, default mappings not represented in docs/tests, and Pydantic declarative constraints lacking portability classification.

The audit should become a release gate.

## 10. Determinism tests

Equivalent model definitions must produce equivalent Arrow schemas and canonical metadata. Test metadata key ordering, canonical JSON output, enum/allowed-value ordering policy, decimal constraint serialization, nested metadata, repeated schema generation, and process-independent output where practical.

## 11. IPC round-trip tests

Every metadata-bearing feature should be tested through actual Arrow serialization/IPC rather than only inspecting in-memory metadata dictionaries. The contract after deserialization should match the contract before serialization.

## 12. Negative tests

Explicitly test failure for arbitrary unsupported Python types, arbitrary nested `pydantic.BaseModel` classes, incompatible Python and Arrow types, unsupported unions, malformed custom constraint payloads, malformed custom metadata, conflicting annotations, unsupported declarative constraints, invalid nested combinations, and invalid values rejected by ArrowBound/Pydantic boundary validation.

## 13. Error quality tests

Important errors should assert actionable content, including where applicable model name, full field path, Python annotation, requested Arrow type, installed PyArrow version, required PyArrow version when known, and recommended remediation.

## 14. Regression policy

Every bug in schema interpretation, boundary validation, runtime compatibility, constraint serialization, or type compatibility should receive a regression test that would have failed before the fix.

## 15. Coverage philosophy

High line coverage is useful, but the primary quality metric should be **semantic matrix coverage**: every supported type family, default mapping, constraint class, validation boundary, escape hatch, and compatibility state has explicit expected behavior.
