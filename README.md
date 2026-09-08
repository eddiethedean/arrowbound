# ArrowBound

**A constrained Pydantic model system for defining portable Apache Arrow data contracts.**

ArrowBound lets developers define Arrow-backed data contracts using familiar Python and Pydantic syntax while preserving deterministic Apache Arrow schemas and portable declarative constraints.

> **Python and Pydantic for authorship. PyArrow for types. ArrowBound for portability. Apache Arrow for interchange.**

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

ArrowBound compiles the model into a deterministic `pyarrow.Schema`:

```python
schema = Customer.arrow_schema()
```

Python types receive documented Arrow defaults. Pydantic constraints are normalized into portable ArrowBound metadata. Developers only reach for explicit Arrow declarations when they need precise control over the underlying Arrow representation:

```python
from arrowbound import Arrow


class Measurement(BaseModel):
    sequence: Annotated[int, Arrow.uint32()]
    value: Annotated[float, Arrow.float32()]
```

## Core guarantees

- ArrowBound models must inherit from `arrowbound.BaseModel`.
- If ArrowBound accepts a model definition, it can deterministically produce its Arrow schema.
- Ordinary Python types use stable, documented Arrow defaults.
- Pydantic constraints describe validity and do not silently change physical Arrow types.
- Explicit `Arrow.*` declarations are optional escape hatches, not the required authoring interface.
- ArrowBound dynamically uses the capabilities of the installed PyArrow runtime.
- Unsupported or unavailable Arrow features fail with actionable errors; ArrowBound never silently degrades a requested representation.
- Native Arrow semantics are used whenever Arrow already represents the concept.
- Portable constraints are stored as versioned, language-neutral Arrow metadata.

## Project scope

ArrowBound defines contracts. It is intentionally **not** a dataframe library, ETL framework, ORM, query engine, schema registry, migration system, or engine-specific integration layer.

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

> **Natural when possible. Explicit when necessary. Portable always.**
