# ArrowBound Package Plan

## Product definition

**ArrowBound is a constrained Pydantic model system for defining portable Apache Arrow data contracts.**

ArrowBound combines familiar Python type annotations, Pydantic validation and constraints, deterministic Apache Arrow schemas, versioned ArrowBound portable constraint metadata, and dynamic access to the capabilities of the installed PyArrow runtime.

> **Python and Pydantic for authorship. PyArrow for types. ArrowBound for portable semantics. Apache Arrow for interchange.**

## Core guarantee

Users define contracts by inheriting from `arrowbound.BaseModel`. If ArrowBound accepts a model definition, it can deterministically produce its Apache Arrow schema. ArrowBound never silently degrades, substitutes, or discards requested semantics.

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

Apache Arrow does not provide a general-purpose constraint system for rules such as `minimum`, `maximum`, `pattern`, or string length. ArrowBound defines and preserves those portable declarative constraints in Arrow metadata. Apache Arrow transports that metadata but does not interpret or enforce it.

Pydantic enforces applicable constraints when ArrowBound model instances are validated. Downstream ArrowBound-aware consumers may choose to interpret or enforce the same portable metadata. ArrowBound's core responsibility is definition and preservation, not general Arrow-table constraint execution.

## Contract dimensions

ArrowBound keeps three concepts separate:

1. **Arrow schema semantics** — datatypes, nullability declarations, nested structure, decimal precision/scale, timestamp parameters, and other semantics represented natively by Apache Arrow.
2. **ArrowBound portable constraints** — declarative validity rules that Arrow does not natively represent as a general constraint system.
3. **ArrowBound metadata** — descriptive or domain semantics that are not necessarily validity rules.

Native Arrow schema semantics always take precedence over duplicating the same information in ArrowBound metadata.

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
3. **PyArrow is the type system.** ArrowBound must not recreate or wrap Apache Arrow datatypes unnecessarily.
4. **Explicit Arrow is optional.** Developers use `Arrow.*` or `pa.*` only when they want control beyond documented defaults.
5. **`Arrow.*` preserves PyArrow identity.** Successful datatype helpers return real `pyarrow.DataType` values equivalent to their `pa.*` counterparts.
6. **Direct PyArrow is first-class.** Compatible `pa.DataType` instances require no ArrowBound wrapper.
7. **Installed PyArrow determines capabilities.** Runtime capability detection should be preferred over unnecessary release locking.
8. **Never silently degrade.** If a requested representation cannot be honored, fail clearly.
9. **Native Arrow semantics win.** Do not duplicate information in ArrowBound metadata that Arrow already represents.
10. **ArrowBound portable constraints are not Arrow constraints.** They are ArrowBound-defined semantics carried through Arrow metadata.
11. **Portable constraints are declarative.** Arbitrary Python validation logic is not a portable contract.
12. **Definition is separate from enforcement.** ArrowBound defines and preserves portable constraints; Pydantic or downstream consumers may enforce them.
13. **Escape hatches remain available.** Advanced users can access PyArrow types, namespaced custom constraints, and raw metadata.
14. **ArrowBound defines; it does not execute.** Dataframes, databases, query engines, registries, and ETL systems remain outside the package.
15. **Portability over convenience hacks.** Never make schema construction easier by making meaning ambiguous.
16. **No unexplained gaps.** Every relevant PyArrow type and applicable Pydantic constraint should have a known support classification.

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

ArrowBound is deliberately not a dataframe library, ETL framework, ORM, query engine, schema registry, database migration system, Arrow table constraint-enforcement engine, engine integration layer, or arbitrary-Pydantic-to-Arrow converter. ArrowBound defines and preserves contracts; other systems consume them through Apache Arrow.

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

- `model.py`: constrained `BaseModel` and definition enforcement.
- `schema.py`: Arrow schema compiler.
- `types.py`: Python defaults, `Arrow.*` convenience facade, Python↔PyArrow compatibility.
- `compatibility.py`: installed PyArrow capability detection and diagnostics.
- `constraints.py`: Pydantic constraint extraction and normalization into ArrowBound portable constraints.
- `metadata.py`: metadata specification, canonical serialization, custom namespaces.
- `exceptions.py`: public error hierarchy.

No plugin architecture without demonstrated need.

## Release philosophy

ArrowBound releases declare minimum Python, supported Pydantic range, minimum PyArrow, and ArrowBound metadata-spec version. Avoid upper-bounding PyArrow unnecessarily. Newer compatible PyArrow versions should normally expose more capabilities rather than forcing an ArrowBound upgrade.

## 1.0 definition

By 1.0, default Python mappings are deterministic/documented; every built-in schema-definable PyArrow type has a known capability classification; direct `pa.DataType` and `Arrow.*` authoring are stable and equivalent where helpers overlap; runtime capability detection is reliable; ArrowBound portable declarative constraints are versioned/deterministic; metadata survives Arrow serialization round trips; equivalent models generate equivalent schemas/metadata; and generated schemas remain ordinary Apache Arrow schemas.

> **Natural when possible. Explicit when necessary. Portable always.**
