# ArrowBound Roadmap

## V0.1 — Foundation

Prove the core architecture with:

- constrained `BaseModel`,
- documented Python→Arrow defaults,
- `Arrow.*` lazy type specifications,
- direct `pyarrow.DataType` escape hatch,
- runtime PyArrow capability detection,
- `arrow_schema()`,
- nested ArrowBound models,
- nullability,
- core collections,
- natural Pydantic constraint extraction,
- versioned ArrowBound constraint metadata,
- descriptive metadata escape hatch,
- clear unsupported/unavailable errors,
- initial capability registry and audit infrastructure.

The goal is not complete type coverage yet; it is to prove the invariant that accepted models compile deterministically.

## V0.2 — Complete built-in type coverage

Systematically close the PyArrow capability matrix across:

- all numeric types,
- all binary/string variants,
- all decimal types,
- all temporal types,
- all list variants,
- maps,
- structs,
- dense/sparse unions,
- dictionary encoding,
- run-end encoding,
- canonical extension types,
- other built-in schema-definable types available in supported runtimes.

Every type must be classified as `DEFAULT`, `SUPPORTED`, `RUNTIME`, `UNAVAILABLE`, or `UNSUPPORTED` with a technical reason.

## V0.3 — Complete declarative constraint coverage

Audit Pydantic's applicable declarative constraint surface.

Every constraint must be classified as:

- `NATIVE`,
- `PORTABLE`,
- `LOCAL`,
- `UNSUPPORTED` with a technical reason.

Complete the ArrowBound constraint vocabulary and stabilize canonical serialization.

Add ArrowBound-native constraints only where Pydantic lacks a natural way to express a genuinely portable semantic.

## V0.4 — Advanced Arrow semantics

Harden interoperability for difficult Arrow representations, including:

- dense and sparse unions,
- dictionary encoding,
- run-end encoding,
- list views,
- nested combinations of advanced types,
- canonical extension types.

Define clear Python/Pydantic representations and compatibility rules for each.

## V0.5 — Portability specification

Extract the ArrowBound metadata representation into an implementation-independent specification covering:

- metadata namespace,
- metadata versioning,
- canonical serialization,
- constraint vocabulary,
- descriptive metadata,
- custom namespaces,
- unknown constraint behavior,
- forward compatibility expectations.

At this stage, another Arrow ecosystem implementation should theoretically be able to read ArrowBound semantics without Python or Pydantic.

## V0.9 — Release audit

Before 1.0:

- audit every relevant PyArrow datatype,
- audit every applicable Pydantic declarative constraint,
- verify all default mappings are documented and tested,
- verify all metadata-bearing features survive IPC round trips,
- verify minimum/latest PyArrow CI,
- verify error diagnostics for missing runtime capabilities,
- document intentional unsupported cases,
- eliminate unexplained support gaps.

## V1.0 — Stable substrate

1.0 means the core data-contract substrate is stable.

### Types

Every built-in schema-definable datatype available in supported PyArrow environments has a known capability classification. No accidental support gaps remain.

### Defaults

Every supported ordinary Python annotation has a deterministic documented Arrow default covered by compatibility tests.

### Escape hatches

Users can request exact Arrow representations when Python defaults are insufficient, and direct `pyarrow.DataType` use remains possible where safe.

### Runtime compatibility

ArrowBound dynamically detects installed PyArrow capabilities. Missing capabilities produce actionable errors rather than silent substitutions.

### Constraints

Every applicable Pydantic declarative constraint has a known portability classification. Portable constraints use stable versioned metadata.

### Determinism

Equivalent ArrowBound definitions generate equivalent Arrow schemas and metadata.

### Portability

Generated schemas remain ordinary Apache Arrow schemas. Consumers unaware of ArrowBound can still understand native Arrow structure; ArrowBound-aware consumers can additionally interpret portable constraints and metadata.

## Post-1.0 discipline

ArrowBound should resist feature expansion outside contract definition. Potential future features should be rejected or split into separate packages when they belong to execution engines, registries, storage systems, migrations, or framework integrations.

The core product definition remains:

> **A constrained Pydantic model system for defining portable Apache Arrow data contracts.**

And the guiding rule remains:

> **Natural when possible. Explicit when necessary. Portable always.**
