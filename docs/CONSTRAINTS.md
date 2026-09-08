# ArrowBound Portable Constraints

## Principle

Apache Arrow does **not** provide a general-purpose data constraint system for rules such as numeric bounds, string lengths, regex patterns, or allowed-value sets.

ArrowBound therefore defines its own **portable declarative constraint vocabulary** and stores those semantics in Arrow field/schema metadata when Arrow has no native representation for them.

> **Pydantic first for authoring. ArrowBound for portable constraint representation. Apache Arrow for transport.**

ArrowBound should reuse ordinary Pydantic declarations whenever they already express the required semantics and only introduce ArrowBound-specific syntax for genuinely missing portable concepts.

## Natural authoring

```python
from typing import Annotated

from arrowbound import BaseModel, Field


class Product(BaseModel):
    sku: Annotated[str, Field(min_length=3, max_length=32)]
    quantity: Annotated[int, Field(ge=0)]
    score: Annotated[float, Field(gt=0, le=1)]
```

Users should not have to repeat the same semantics in a second ArrowBound constraint API.

## What Arrow does and does not enforce

Apache Arrow represents native schema semantics such as datatype structure, field nullability declarations, decimal precision/scale, timestamp parameters, nested types, dictionary encodings, and other datatype parameters.

Those are **Arrow schema semantics**, not ArrowBound portable constraints.

ArrowBound portable constraints include declarative rules such as:

```text
minimum
maximum
exclusive_minimum
exclusive_maximum
multiple_of
min_length
max_length
pattern
allowed_values
min_items
max_items
```

Apache Arrow transports ArrowBound metadata but does not interpret or enforce these rules.

Pydantic enforces applicable rules when ArrowBound model instances are validated. A downstream ArrowBound-aware consumer may also choose to interpret or enforce them.

ArrowBound itself is primarily responsible for **definition, normalization, and preservation**, not for becoming a general Arrow-table constraint execution engine.

## Normalization

Pydantic syntax is the Python authoring interface, but Pydantic-specific names should not leak into the portable format.

For example:

```python
Field(ge=0, lt=100)
```

normalizes to language-neutral semantics such as:

```text
minimum: 0
exclusive_maximum: 100
```

The final vocabulary should be formalized in the metadata specification.

## Constraint capability classifications

Every applicable Pydantic semantic should receive a clear classification:

### ARROW_NATIVE

The semantic is already represented by the Arrow schema/type system and therefore does not require ArrowBound constraint metadata.

Examples include field nullability and datatype parameters such as decimal precision/scale.

This classification does **not** mean Arrow provides a general constraint-enforcement engine. It means Arrow already carries the relevant schema semantic natively.

### PORTABLE

ArrowBound can completely represent the declarative semantic in versioned, language-neutral metadata.

Examples include numeric bounds, string length, regex patterns, `multiple_of`, and allowed values.

### LOCAL

Pydantic can enforce the behavior in Python, but the behavior cannot be represented as a language-neutral ArrowBound contract.

Example: arbitrary `@field_validator` Python logic.

### UNSUPPORTED

ArrowBound cannot safely interpret or preserve the declared semantics. This must have a documented technical reason.

## Constraints do not choose physical types

A constraint must not silently alter the Arrow datatype.

```python
quantity: Annotated[int, Field(ge=0)]
```

remains the documented `int` default (`int64`) unless the user explicitly requests an unsigned type.

This keeps validity semantics independent from storage semantics.

## ArrowBound-specific constraints

Pydantic does not naturally express every declarative semantic that may be useful in a portable data contract. ArrowBound may add typed annotations conservatively:

```python
from arrowbound import Constraints


class Record(BaseModel):
    code: Annotated[
        str,
        Field(min_length=3),
        Constraints.some_portable_constraint(...),
    ]
```

A new ArrowBound-specific constraint should only be added when:

1. the semantic is genuinely portable and declarative,
2. Pydantic has no natural equivalent,
3. it can be serialized deterministically,
4. its interaction with relevant Arrow types is well-defined.

## Custom constraint escape hatch

Domain-specific systems should be able to carry declarative semantics ArrowBound does not interpret:

```python
ArrowConstraint(
    namespace="com.example",
    name="quality_code",
    value={"accepted": [1, 2, 3]},
)
```

Custom constraints must be namespaced to prevent collisions. ArrowBound preserves them deterministically without claiming to understand or enforce their domain semantics.

## Validators

Custom Pydantic validators remain useful local behavior:

```python
@field_validator("name")
def validate_name(cls, value):
    ...
```

Executable Python validation logic is not portable merely because it lives on an ArrowBound model.

ArrowBound must clearly distinguish:

```text
portable declarative constraint
        vs.
local Pydantic validation
```

A local validator only becomes part of the portable contract if its semantics are separately represented using a supported declarative constraint.

## Future relational constraints

Concepts such as primary keys, foreign keys, composite keys, and relational uniqueness should not be assumed part of the core v1 vocabulary merely because they are useful in databases.

They are broader relational-contract concepts and should be evaluated separately to avoid expanding ArrowBound into a database schema system.
