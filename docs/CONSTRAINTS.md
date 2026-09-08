# ArrowBound Constraints

## Principle

Constraints should feel natural to Pydantic users.

ArrowBound should reuse ordinary Pydantic declarations whenever they already express the required semantics and only introduce ArrowBound-specific syntax for genuinely missing portable concepts.

> **Pydantic first for constraints. ArrowBound-specific constraints only where necessary.**

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

## Normalization

Pydantic syntax is the Python authoring interface, but Pydantic-specific names should not leak into the portable format.

For example:

```python
Field(ge=0, lt=100)
```

should normalize to language-neutral semantics such as:

```text
minimum: 0
exclusive_maximum: 100
```

Candidate normalized vocabulary includes:

- `minimum`
- `maximum`
- `exclusive_minimum`
- `exclusive_maximum`
- `multiple_of`
- `min_length`
- `max_length`
- `pattern`
- `allowed_values`
- `min_items`
- `max_items`

The final vocabulary should be formalized in the metadata specification.

## Native, portable, local, unsupported

Every applicable declarative Pydantic constraint should receive a capability classification:

### NATIVE

The semantic already exists directly in Arrow and requires no ArrowBound metadata.

Example: optionality maps to Arrow field nullability.

### PORTABLE

ArrowBound can completely represent the declarative semantic in versioned portable metadata.

Examples: numeric bounds, string length, regex patterns, `multiple_of`, allowed values.

### LOCAL

Pydantic can enforce the rule in Python, but the behavior cannot be represented as a language-neutral ArrowBound contract.

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

Pydantic does not naturally express every declarative semantic that may be useful in an Arrow contract. ArrowBound may add typed annotations conservatively:

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

Custom constraints must be namespaced to prevent collisions. ArrowBound should preserve them deterministically without claiming to validate their domain semantics.

## Validators

Custom Pydantic validators remain useful local behavior:

```python
@field_validator("name")
def validate_name(cls, value):
    ...
```

However, executable Python validation logic is not portable merely because it lives on an ArrowBound model.

ArrowBound must clearly distinguish:

```text
portable declarative constraint
        vs.
local Pydantic validation
```

A local validator only becomes part of the portable contract if its semantics are separately represented using a supported declarative constraint.

## Future relational constraints

Concepts such as primary keys, foreign keys, composite keys, and relational uniqueness should not be assumed part of the core v1 constraint vocabulary merely because they are useful in databases.

They are broader relational-contract concepts and should be evaluated separately to avoid expanding ArrowBound into a database schema system.
