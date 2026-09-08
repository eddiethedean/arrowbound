# ArrowBound Portable Constraints

## Principle

Apache Arrow does **not** provide a general-purpose data constraint system for rules such as numeric bounds, string lengths, regex patterns, or allowed-value sets.

ArrowBound adds that missing contract layer by combining Pydantic validation with portable Arrow metadata.

> **Pydantic enforces at ingress. ArrowBound preserves and communicates the constraint contract. Apache Arrow transports it.**

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

## Boundary enforcement model

When data is instantiated or otherwise validated through an ArrowBound model, Pydantic enforces applicable declared constraints before that data enters the Arrow substrate.

Conceptually:

```text
Python input
    ↓
ArrowBound model
    ↓
Pydantic validation
    ↓
validated values
    ↓
Arrow schema + ArrowBound portable constraint metadata
    ↓
Arrow substrate
```

This means ArrowBound does more than merely document constraints: when the ArrowBound model is used as the ingress boundary, it establishes that the data entering the Arrow substrate satisfied those constraints at validation time.

Arrow itself does not continuously re-check those constraints afterward. If data is later mutated, constructed through another path, or enters the Arrow substrate without ArrowBound model validation, the prior ingress guarantee does not automatically apply.

Downstream ArrowBound-aware systems can read the preserved constraint metadata and choose to revalidate or enforce the same contract at their own boundaries.

## What Arrow does and does not represent

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

Arrow carries this metadata through the substrate; ArrowBound and other aware systems give it contract meaning.

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

This does **not** mean Arrow provides a general constraint-enforcement engine. It means Arrow already carries the relevant schema semantic natively.

### PORTABLE

ArrowBound can completely represent the declarative semantic in versioned, language-neutral metadata and Pydantic can enforce it at an ArrowBound validation boundary where applicable.

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

Pydantic does not naturally express every declarative semantic that may be useful in a portable data contract. ArrowBound may add typed annotations conservatively.

A new ArrowBound-specific constraint should only be added when:

1. the semantic is genuinely portable and declarative,
2. Pydantic has no natural equivalent,
3. it can be serialized deterministically,
4. its validation/enforcement behavior at ArrowBound boundaries is defined,
5. its interaction with relevant Arrow types is well-defined.

## Custom constraint escape hatch

Domain-specific systems should be able to carry declarative semantics ArrowBound does not interpret:

```python
ArrowConstraint(
    namespace="com.example",
    name="quality_code",
    value={"accepted": [1, 2, 3]},
)
```

Custom constraints must be namespaced to prevent collisions. ArrowBound preserves them deterministically without claiming to understand or enforce their domain semantics unless an explicit ArrowBound validator exists for that constraint.

## Validators

Custom Pydantic validators remain useful local behavior. Executable Python validation logic is not portable merely because it lives on an ArrowBound model.

ArrowBound must clearly distinguish:

```text
portable declarative constraint
        vs.
local Pydantic validation
```

A local validator only becomes part of the portable contract if its semantics are separately represented using a supported declarative constraint.

## Future relational constraints

Concepts such as primary keys, foreign keys, composite keys, and relational uniqueness should not be assumed part of the core v1 vocabulary merely because they are useful in databases. They are broader relational-contract concepts and should be evaluated separately to avoid expanding ArrowBound into a database schema system.
