# ArrowBound

**A constrained Pydantic model system for defining portable Apache Arrow data contracts.**

ArrowBound lets developers define Arrow-backed data contracts using familiar Python and Pydantic syntax while preserving deterministic Apache Arrow schemas and portable declarative semantics across system boundaries.

> **Python and Pydantic for authorship and ingress validation. PyArrow for types. ArrowBound for portable semantics. Apache Arrow for interchange.**

## Core idea

Ordinary contracts should look like ordinary Pydantic models:

```python
from typing import Annotated

from arrowbound import BaseModel, Field


class Customer(BaseModel):
    id: int
    name: str
    age: Annotated[int, Field(ge=0, le=150)]
```

When data is validated through an ArrowBound model, Pydantic enforces the declared constraints before that data enters the Arrow substrate.

ArrowBound then compiles the model into a deterministic `pyarrow.Schema`:

```python
schema = Customer.arrow_schema()
```

Python types receive documented Arrow defaults. Pydantic declarative constraints that Apache Arrow does not natively represent are normalized into versioned **ArrowBound portable constraint metadata** attached to Arrow fields or schemas.

This gives ArrowBound two related roles:

1. **Ingress enforcement** — applicable constraints are enforced by Pydantic when data is validated through ArrowBound models.
2. **Constraint communication** — those portable declarative constraints are preserved in Arrow metadata so downstream systems can understand the contract.

Apache Arrow transports that metadata but does not continuously re-enforce ArrowBound constraints after data has entered the Arrow substrate. A downstream ArrowBound-aware consumer may choose to interpret or enforce them again.

Developers only reach for explicit Arrow declarations when they need precise control over the underlying Arrow representation:

```python
from typing import Annotated
import pyarrow as pa
from arrowbound import Arrow, BaseModel


class Measurement(BaseModel):
    sequence: Annotated[int, Arrow.uint32()]
    value: Annotated[float, pa.float32()]
```

`Arrow.*` is a convenience facade over PyArrow datatypes, not a second type system. Direct `pa.*` datatypes are equally first-class.

## Three parts of an ArrowBound contract

- **Arrow schema semantics** — datatype, nullability, nested structure, decimal precision/scale, timestamp parameters, and other semantics Apache Arrow represents natively.
- **ArrowBound portable constraints** — declarative validity rules such as numeric bounds or string lengths that Arrow itself does not natively encode as a general constraint system.
- **ArrowBound metadata** — descriptive or domain semantics such as units or semantic labels that are not necessarily validity rules.

## Core guarantees

- ArrowBound models must inherit from `arrowbound.BaseModel`.
- If ArrowBound accepts a model definition, it can deterministically produce its Arrow schema.
- Data validated through ArrowBound models is checked against applicable Pydantic constraints before entering the Arrow substrate.
- Ordinary Python types use stable, documented Arrow defaults.
- Pydantic constraints describe validity and do not silently change physical Arrow types.
- `Arrow.*` and direct `pyarrow.DataType` declarations are optional escape hatches, not the required authoring interface.
- ArrowBound dynamically uses the capabilities of the installed PyArrow runtime.
- Unsupported or unavailable Arrow features fail clearly; ArrowBound never silently degrades a requested representation.
- Native Arrow schema semantics are used whenever Arrow already represents the concept.
- ArrowBound portable constraints are stored as versioned, language-neutral Arrow metadata.
- Arrow carries ArrowBound constraint metadata downstream; downstream systems may enforce it, but Arrow itself does not act as a general constraint engine.

## Project scope

ArrowBound defines, enforces at model-validation boundaries, and preserves portable contracts. It is intentionally **not** a dataframe library, ETL framework, ORM, query engine, schema registry, migration system, general Arrow table constraint-enforcement engine, or engine-specific integration layer.

## Planning documents

- [Package Plan](docs/PLAN.md)
- [Architecture](docs/ARCHITECTURE.md)
- [Type System](docs/TYPES.md)
- [Constraints](docs/CONSTRAINTS.md)
- [Metadata Specification](docs/METADATA.md)
- [PyArrow Compatibility](docs/COMPATIBILITY.md)
- [Capability Audit](docs/CAPABILITY_AUDIT.md)
- [Testing Strategy](docs/TESTING.md)
- [Roadmap](docs/ROADMAP.md)

## Guiding principle

> **Validate at the boundary. Preserve the contract. Transport with Arrow.**
