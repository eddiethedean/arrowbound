# ArrowBound Package Plan

## Product definition

**ArrowBound is a constrained Pydantic model system for defining portable Apache Arrow data contracts.**

ArrowBound combines familiar Python type annotations, familiar Pydantic validation and constraints, deterministic Apache Arrow schemas, portable constraint metadata, and dynamic access to the capabilities of the installed PyArrow runtime.

The core philosophy is:

> **Python and Pydantic for authorship. PyArrow for types. ArrowBound for portability. Apache Arrow for interchange.**

## Core guarantee

ArrowBound owns its model hierarchy:

```python
from arrowbound import BaseModel


class Customer(BaseModel):
    id: int
    name: str
    active: bool
```

The fundamental invariant is:

> **If ArrowBound accepts a model definition, ArrowBound can deterministically produce its Apache Arrow schema.**

ArrowBound must never silently degrade, substitute, or discard unsupported type semantics. Unsupported definitions fail with actionable errors.

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

```python
schema = Measurement.arrow_schema()
```

The result is a normal `pyarrow.Schema` containing Arrow-native types, Arrow-native nullability and structure, plus ArrowBound metadata only where Arrow itself does not represent the required portable semantics.

## Design principles

1. **Python first.** Ordinary Python annotations should handle ordinary contracts.
2. **Pydantic first for constraints.** Do not invent new syntax when Pydantic already expresses the concept naturally.
3. **Arrow is authoritative.** ArrowBound must not recreate Apache Arrow's type system.
4. **Explicit Arrow is an escape hatch.** Developers use `Arrow.*` only when they want control beyond documented defaults.
5. **Installed PyArrow determines capabilities.** Runtime capability detection should be preferred over unnecessary release locking.
6. **Never silently degrade.** If a requested representation cannot be honored, fail clearly.
7. **Native Arrow semantics win.** Do not duplicate information in ArrowBound metadata that Arrow already represents.
8. **Portable constraints are declarative.** Arbitrary Python validation logic is not a portable contract.
9. **Escape hatches remain available.** Advanced users can access explicit Arrow types, namespaced custom constraints, and raw metadata.
10. **ArrowBound defines; it does not execute.** Dataframes, databases, query engines, registries, and ETL systems remain outside the package.
11. **Portability over convenience hacks.** Never make schema construction easier by making meaning ambiguous.
12. **No unexplained gaps.** Every relevant PyArrow type and applicable Pydantic constraint should have a known support classification.

## Public API direction

The common API should remain small:

```python
from arrowbound import BaseModel, Field
```

Advanced users additionally reach for:

```python
from arrowbound import Arrow, Constraints, Metadata
```

Potential low-level escape hatch:

```python
from arrowbound import ArrowConstraint
```

The primary model method is:

```python
Model.arrow_schema()
```

## Scope boundaries

ArrowBound is deliberately not:

- a dataframe library,
- an ETL framework,
- an ORM,
- a query engine,
- a schema registry,
- a database migration system,
- an Arrow table validation engine,
- a Polars integration layer,
- a DuckDB integration layer,
- a Spark integration layer,
- a general arbitrary-Pydantic-to-Arrow converter.

ArrowBound defines contracts. Other systems consume those contracts through Apache Arrow.

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

Responsibilities:

- `model.py`: constrained `BaseModel` and model-definition enforcement.
- `schema.py`: Arrow schema compiler.
- `types.py`: Python defaults, lazy Arrow type specifications, Python↔Arrow compatibility.
- `compatibility.py`: installed PyArrow capability detection and diagnostics.
- `constraints.py`: Pydantic constraint extraction and ArrowBound normalization.
- `metadata.py`: metadata specification, canonical serialization, and custom namespaces.
- `exceptions.py`: public error hierarchy.

No plugin architecture should be introduced without demonstrated need.

## Release philosophy

ArrowBound releases declare:

- minimum Python version,
- supported Pydantic range,
- minimum PyArrow version,
- ArrowBound metadata specification version.

ArrowBound should avoid upper-bounding PyArrow unnecessarily. Newer compatible PyArrow versions should normally expose more available capabilities rather than forcing an ArrowBound upgrade.

## 1.0 definition

ArrowBound 1.0 means the substrate is stable, not that the project has accumulated integrations. By 1.0:

- default Python mappings are deterministic and documented,
- every built-in schema-definable PyArrow type has a known capability classification,
- explicit Arrow escape hatches are stable,
- runtime capability detection is reliable,
- portable declarative constraints are versioned and deterministic,
- metadata survives Arrow serialization round trips,
- equivalent model definitions generate equivalent Arrow schemas and metadata,
- ArrowBound-generated schemas remain ordinary Apache Arrow schemas.

> **Natural when possible. Explicit when necessary. Portable always.**
