# ArrowBound Testing Strategy

## Goal

ArrowBound's value depends on deterministic schema generation, predictable compatibility behavior, and metadata preservation. The test suite should therefore be systematic and matrix-driven rather than relying mainly on example-based happy paths.

## 1. Default mapping tests

Every documented Python default must have exact schema assertions.

For each mapping, test:

- resulting `pyarrow.DataType`,
- field nullability,
- nested structure where applicable,
- default behavior independent of runtime values,
- stable output across equivalent model definitions.

Examples include:

```text
bool -> bool
int -> int64
float -> float64
str -> string
bytes -> binary
date -> date32
datetime -> timestamp[us]
timedelta -> duration[us]
list[T] -> list<T>
T | None -> nullable field
nested ArrowBound model -> struct
```

## 2. Explicit Arrow type tests

Every first-class `Arrow.*` helper should be tested for:

- lazy construction,
- correct PyArrow resolution,
- valid Python/Pydantic compatibility,
- invalid compatibility rejection,
- parameter preservation,
- missing-runtime capability diagnostics,
- no silent type substitution.

## 3. Direct `pyarrow.DataType` tests

The raw PyArrow escape hatch should verify that:

- supported built-in datatypes are accepted,
- unknown/new built-in datatypes can flow through generically where safe,
- incompatible Python annotations fail clearly,
- unsupported custom extension behavior is explicit,
- direct types preserve their exact parameters in the resulting schema.

## 4. Constraint tests

Every supported Pydantic declarative constraint should test the full pipeline:

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

## 5. Native semantic tests

Verify that ArrowBound does not duplicate native Arrow semantics into metadata.

Examples:

- optionality is represented through Arrow nullability,
- decimal precision/scale remain datatype parameters,
- timestamp timezone/unit remain datatype parameters,
- list element types remain datatype structure,
- dictionary semantics remain datatype structure.

## 6. Nested and advanced type tests

Systematically cover:

- nested structs,
- deeply nested lists/structs/maps,
- fixed-size collections,
- list views,
- dense and sparse unions,
- dictionary encoding,
- run-end encoding,
- decimal families,
- temporal variants,
- canonical extension types,
- combinations of advanced types.

Nested failure errors must report full field paths.

## 7. Runtime compatibility tests

CI should test at least:

```text
minimum supported PyArrow
latest stable PyArrow
pre-release/nightly PyArrow where practical
```

The nightly/pre-release job may begin non-blocking, but should surface API drift and new datatype capabilities early.

Version-specific tests should verify:

- capabilities present in the installed runtime are usable,
- capabilities missing in older runtimes produce useful upgrade errors,
- ArrowBound does not rely solely on version-number comparisons when capability introspection is available.

## 8. Capability audit tests

The capability registry and audit should be testable artifacts.

Tests should detect:

- built-in PyArrow datatypes with no ArrowBound classification,
- documented `Arrow.*` helpers missing tests,
- registry entries whose runtime factories no longer exist,
- default mappings not represented in docs/tests,
- Pydantic declarative constraints lacking portability classification.

The audit should become a release gate.

## 9. Determinism tests

Equivalent model definitions must produce equivalent Arrow schemas and canonical metadata.

Test:

- metadata key ordering,
- canonical JSON output,
- enum/allowed-value ordering policy,
- decimal constraint serialization,
- nested metadata,
- repeated schema generation,
- process-independent output where practical.

## 10. IPC round-trip tests

Every metadata-bearing feature should be tested through actual Arrow serialization/IPC rather than only inspecting in-memory metadata dictionaries.

The contract after deserialization should match the contract before serialization.

## 11. Negative tests

Explicitly test failure for:

- arbitrary unsupported Python types,
- arbitrary nested `pydantic.BaseModel` classes,
- incompatible Python and Arrow types,
- unsupported unions,
- unavailable runtime capabilities,
- malformed custom constraint payloads,
- malformed custom metadata,
- conflicting annotations,
- unsupported declarative constraints,
- invalid nested combinations.

## 12. Error quality tests

Important errors should assert actionable content, including where applicable:

- model name,
- full field path,
- Python annotation,
- requested Arrow type,
- installed PyArrow version,
- required PyArrow version when known,
- recommended remediation.

## 13. Regression policy

Every bug in schema interpretation, runtime compatibility, constraint serialization, or type compatibility should receive a regression test that would have failed before the fix.

## 14. Coverage philosophy

High line coverage is useful, but the primary quality metric should be **semantic matrix coverage**: every supported type family, default mapping, constraint class, escape hatch, and compatibility state has explicit expected behavior.
