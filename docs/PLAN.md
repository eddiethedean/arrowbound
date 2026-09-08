# ArrowBound Package Plan

## Product definition

**ArrowBound is a constrained Pydantic model system for defining portable Apache Arrow data contracts.**

ArrowBound combines familiar Python type annotations, Pydantic validation and constraints, deterministic Apache Arrow schemas, portable constraint metadata, and dynamic access to the capabilities of the installed PyArrow runtime.

> **Python and Pydantic for authorship and ingress validation. PyArrow for types. ArrowBound for portable semantics. Apache Arrow for interchange.**

## Core guarantee

Users define contracts by inheriting from `arrowbound.BaseModel`. If ArrowBound accepts a model definition, it can deterministically produce its Apache Arrow schema. ArrowBound never silently degrades, substitutes, or discards requested semantics.

When data is validated through an ArrowBound model, applicable Pydantic constraints are enforced before that data enters the Arrow substrate. ArrowBound then preserves the portable declarative form of those constraints in Arrow metadata so downstream consumers can understand the contract.

This produces two complementary guarantees:

1. **Boundary validation:** values that enter through ArrowBound model validation satisfied the declared applicable constraints at that boundary.
2. **Constraint preservation:** the corresponding portable declarative contract can travel with the Arrow schema downstream.

Arrow itself does not continuously revalidate ArrowBound constraints after ingress. Data constructed or mutated through other paths must not be assumed valid merely because the schema carries ArrowBound metadata.

## Basic developer experience

Ordinary models should look like ordinary Pydantic models:

```python
from datetime import datetime
from typing import Annotated
from arrowbound import BaseModel, Field

class Measurement(BaseModel):
    sensor_id: str
    sequence: Annotated[int, Field(ge=0)]
    value: float
    timestamp: datetime
```

`Measurement.arrow_schema()` returns a normal `pyarrow.Schema` containing Arrow-native types, nullability and structure, plus ArrowBound metadata only where Arrow itself lacks the portable semantic.

## Explicit type control

Most users rely on documented Python→Arrow defaults. When exact Arrow representation matters, both ArrowBound's convenience namespace and PyArrow itself are first-class:

```python
from typing import Annotated
import pyarrow as pa
from arrowbound import Arrow, BaseModel

class Exact(BaseModel):
    small: Annotated[int, Arrow.int16()]
    count: Annotated[int, pa.uint32()]
```

`Arrow.*` is not a wrapper type system. Its datatype helpers return the actual PyArrow datatypes produced by the corresponding factories:

```python
Arrow.int32() == pa.int32()
```

Users may freely mix `Arrow.*` and `pa.*`. Direct PyArrow datatypes are also the forward-compatibility escape hatch for newly introduced types before ArrowBound adds convenience helpers.

## Design principles

1. **Python first.** Ordinary Python annotations should handle ordinary contracts.
2. **Pydantic first for constraints.** Do not invent new syntax when Pydantic already expresses the concept naturally.
3. **Validate at ArrowBound boundaries.** Applicable portable constraints should be enforced when data is validated through ArrowBound models.
4. **Preserve the contract.** Portable constraint semantics should survive into Arrow metadata for downstream consumers.
5. **Do not overclaim continuous validity.** ArrowBound validation establishes validity at a boundary, not forever after arbitrary mutation or alternate ingestion paths.
6. **PyArrow is the type system.** ArrowBound must not recreate or wrap Apache Arrow datatypes unnecessarily.
7. **Explicit Arrow is optional.** Developers use `Arrow.*` or `pa.*` only when they want control beyond documented defaults.
8. **`Arrow.*` preserves PyArrow identity.** Successful datatype helpers return real `pyarrow.DataType` values equivalent to their `pa.*` counterparts.
9. **Direct PyArrow is first-class.** Compatible `pa.DataType` instances require no ArrowBound wrapper.
10. **Installed PyArrow determines capabilities.** Runtime capability detection should be preferred over unnecessary release locking.
11. **Never silently degrade.** If a requested representation cannot be honored, fail clearly.
12. **Native Arrow semantics win.** Do not duplicate information in ArrowBound metadata that Arrow already represents.
13. **Portable constraints are declarative.** Arbitrary Python validation logic is not automatically a portable contract.
14. **Escape hatches remain available.** Advanced users can access PyArrow types, namespaced custom constraints, and raw metadata.
15. **ArrowBound defines and validates boundaries; it does not become an Arrow execution engine.** Dataframes, databases, query engines, registries, and ETL systems remain outside the package.
16. **Portability over convenience hacks.** Never make schema construction easier by making meaning ambiguous.
17. **No unexplained gaps.** Every relevant PyArrow type and applicable Pydantic constraint should have a known support classification.

## Public API direction

Common API:

```python
from arrowbound import BaseModel, Field
```

Advanced convenience API:

```python
from arrowbound import Arrow, Constraints, Metadata
```

PyArrow itself remains part of the supported advanced authoring surface:

```python
import pyarrow as pa
```

Potential low-level constraint escape hatch: `ArrowConstraint`. Primary model method: `Model.arrow_schema()`.

## Scope boundaries

ArrowBound is deliberately not a dataframe library, ETL framework, ORM, query engine, schema registry, database migration system, general Arrow-table constraint engine, engine integration layer, or arbitrary-Pydantic-to-Arrow converter.

ArrowBound defines the contract, enforces applicable constraints when data crosses an ArrowBound model-validation boundary, and preserves portable constraint semantics for downstream Arrow consumers.

## Internal package shape

```text
arrowbound/
├── __init__.py
├── model.py
├── schema.py
├── types.py
├── compatibility.py
├── constraints.py
├── metadata.py
└── exceptions.py
```

- `model.py`: constrained `BaseModel`, definition enforcement, and boundary validation behavior inherited from Pydantic.
- `schema.py`: Arrow schema compiler.
- `types.py`: Python defaults, `Arrow.*` convenience facade, Python↔PyArrow compatibility.
- `compatibility.py`: installed PyArrow capability detection and diagnostics.
- `constraints.py`: Pydantic constraint extraction, portability classification, and normalization.
- `metadata.py`: metadata specification, canonical serialization, custom namespaces.
- `exceptions.py`: public error hierarchy.

No plugin architecture without demonstrated need.

## Release philosophy

ArrowBound releases declare minimum Python, supported Pydantic range, minimum PyArrow, and ArrowBound metadata-spec version. Avoid upper-bounding PyArrow unnecessarily. Newer compatible PyArrow versions should normally expose more capabilities rather than forcing an ArrowBound upgrade.

## 1.0 definition

By 1.0, default Python mappings are deterministic/documented; every built-in schema-definable PyArrow type has a known capability classification; direct `pa.DataType` and `Arrow.*` authoring are stable and equivalent where helpers overlap; runtime capability detection is reliable; applicable portable constraints are actually enforced during ArrowBound model validation; those constraints are versioned and deterministic in Arrow metadata; metadata survives Arrow serialization round trips; equivalent models generate equivalent schemas/metadata; and generated schemas remain ordinary Apache Arrow schemas.

> **Validate at the boundary. Preserve the contract. Transport with Arrow.**
