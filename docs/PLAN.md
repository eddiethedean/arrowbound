# ArrowBound Package Plan

## Product definition

**ArrowBound is a constrained Pydantic model system for defining portable Apache Arrow data contracts.**

ArrowBound combines familiar Python type annotations, Pydantic validation and constraints, deterministic Apache Arrow schemas, portable constraint metadata, and dynamic access to the capabilities of the installed PyArrow runtime.

> **Python and Pydantic for authorship and ingress validation. PyArrow for types. ArrowBound for portable semantics. Apache Arrow for interchange.**

## Core guarantee

Users define contracts by inheriting from `arrowbound.BaseModel`. If ArrowBound accepts a model definition, it can deterministically produce its Apache Arrow schema. ArrowBound never silently degrades, substitutes, or discards requested semantics.

When data is validated through an ArrowBound model, applicable Pydantic constraints are enforced before that data enters the Arrow substrate. ArrowBound then preserves the portable declarative form of those constraints in Arrow metadata so downstream consumers can understand the contract.

This produces boundary validation plus constraint preservation. Arrow itself does not continuously revalidate ArrowBound constraints after ingress; data constructed or mutated through other paths must not be assumed valid merely because the schema carries ArrowBound metadata.

## Basic developer experience

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
import pyarrow as pa
from arrowbound import Arrow

class Exact(BaseModel):
    small: Annotated[int, Arrow.int16()]
    count: Annotated[int, pa.uint32()]
```

`Arrow.*` is not a wrapper type system. Its datatype helpers return actual PyArrow datatypes, and users may freely mix `Arrow.*` and `pa.*`.

## Optional Pandera dataframe validation

ArrowBound's Pydantic models validate individual records and establish validity when data crosses an ArrowBound model boundary. ArrowBound should not grow a competing dataframe validation engine for already-columnar data.

Instead, an optional Pandera integration should expose:

```python
class Customer(BaseModel):
    id: int
    name: Annotated[str, Field(min_length=1)]
    age: Annotated[int, Field(ge=0, le=150)]

schema = Customer.pandera_schema()
validated = schema.validate(table)
```

`pandera_schema()` should return a Pandera schema suitable for validating `pyarrow.Table` data. ArrowBound remains the contract source; Pandera takes over dataframe/table constraint execution.

The compilation path should be:

```text
ArrowBound model
      ↓
normalized ArrowBound contract
      ├────────► pyarrow.Schema + portable metadata
      └────────► Pandera schema/checks
                          ↓
                    table validation
```

The Pandera adapter must compile from ArrowBound's normalized contract rather than independently parsing Pydantic internals. This ensures the Arrow metadata representation and Pandera enforcement are derived from the same semantics.

Pandera remains optional, likely installed with `arrowbound[pandera]`. ArrowBound core must not depend on it.

The adapter must explicitly report any ArrowBound constraint it cannot translate to Pandera. It must never silently drop a constraint and imply that a table was fully validated.

Pandera should own vectorized dataframe checks, whole-table checks, uniqueness scans, lazy error aggregation, and other bulk validation mechanics. ArrowBound should not reimplement those capabilities.

## Design principles

1. Python first.
2. Pydantic first for constraint authoring.
3. Validate applicable constraints at ArrowBound model boundaries.
4. Preserve portable constraint semantics in Arrow metadata.
5. Do not overclaim continuous validity after arbitrary mutation or alternate ingestion.
6. PyArrow is the type system; ArrowBound does not recreate it.
7. Explicit `Arrow.*` and direct `pa.*` are optional first-class escape hatches.
8. Installed PyArrow determines capabilities.
9. Never silently degrade types or constraints.
10. Native Arrow semantics win over duplicate metadata.
11. Portable constraints are declarative; arbitrary Python logic is local unless separately represented.
12. Delegate dataframe validation to mature optional consumers such as Pandera rather than recreating it.
13. Portability over convenience hacks.
14. No unexplained capability gaps.

## Public API direction

Core:

```python
from arrowbound import BaseModel, Field
Model.arrow_schema()
```

Advanced:

```python
from arrowbound import Arrow, Constraints, Metadata
import pyarrow as pa
```

Optional Pandera extra:

```python
Model.pandera_schema()
```

Potential low-level constraint escape hatch: `ArrowConstraint`.

## Scope boundaries

ArrowBound is deliberately not a dataframe library, ETL framework, ORM, query engine, schema registry, database migration system, general Arrow-table constraint engine, or arbitrary-Pydantic-to-Arrow converter.

ArrowBound defines the contract, enforces applicable constraints at model-validation boundaries, and preserves portable semantics. Pandera may optionally enforce applicable contract semantics over Arrow tables.

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
├── exceptions.py
└── integrations/
    └── pandera.py       # optional
```

The Pandera module must import Pandera lazily/optionally and produce a clear install-extra error when unavailable.

## Release philosophy

ArrowBound releases declare minimum Python, supported Pydantic range, minimum PyArrow, and metadata-spec version. Avoid unnecessary PyArrow upper bounds. Optional integrations have their own compatibility tests and should not destabilize the core.

## 1.0 definition

By 1.0, default mappings are deterministic/documented; PyArrow capabilities have known classifications; direct `pa.DataType` and `Arrow.*` authoring are stable; applicable portable constraints are enforced during ArrowBound model validation and preserved deterministically in Arrow metadata; metadata survives serialization; and generated schemas remain ordinary Apache Arrow schemas.

The Pandera integration is an important natural workflow for bulk validation, but the core ArrowBound substrate must remain useful without Pandera.

> **Validate at the boundary. Preserve the contract. Transport with Arrow. Delegate bulk validation.**
