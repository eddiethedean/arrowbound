# ArrowBound Type System

## Principle

ArrowBound should feel Python-first. Most users should define contracts with ordinary Python annotations and never need to spell Arrow types directly.

Explicit Arrow typing exists as an escape hatch for precise control.

> **Use Python types by default. Use explicit Arrow types only when the physical/logical Arrow representation matters.**

## Stable default mappings

ArrowBound defines deterministic defaults up front and documents them as part of the public compatibility contract.

Initial direction:

| Python annotation | Default Arrow representation |
| --- | --- |
| `bool` | `pa.bool_()` |
| `int` | `pa.int64()` |
| `float` | `pa.float64()` |
| `str` | `pa.string()` |
| `bytes` | `pa.binary()` |
| `date` | `pa.date32()` |
| `datetime` | `pa.timestamp("us")` |
| `timedelta` | `pa.duration("us")` |
| `list[T]` | `pa.list_(T)` |
| nested ArrowBound model | `pa.struct(...)` |
| `T | None` | same datatype with field `nullable=True` |

Additional deterministic defaults should be defined for `time`, `Decimal`, `UUID`, enums, `Literal`, mappings, and other standard Python/Pydantic constructs where the mapping is unambiguous.

## Defaults are schema behavior

Default mappings must be stable, documented, deterministic, and independent of runtime data values. ArrowBound must not inspect values and opportunistically narrow a type.

For example, `count: int` always maps to the documented integer default. It must not become `int8`, `uint16`, or another narrower integer depending on observed values.

## Constraints never change physical types implicitly

`quantity: Annotated[int, Field(ge=0)]` still maps to `int64` by default. The non-negative rule is a constraint, not a request for `uint64`. Changing a validation rule must not unexpectedly change an interchange schema.

## Explicit Arrow types are first-class PyArrow datatypes

Users who need exact Arrow semantics may opt in with either ArrowBound's `Arrow.*` convenience namespace or PyArrow directly.

```python
from typing import Annotated
import pyarrow as pa
from arrowbound import Arrow, BaseModel

class Measurement(BaseModel):
    sequence: Annotated[int, Arrow.uint32()]
    value: Annotated[float, pa.float32()]
```

`Arrow.*` must not define wrapper datatype objects. When a corresponding PyArrow factory exists, it returns the actual `pyarrow.DataType` produced by that factory.

```python
Arrow.int32() == pa.int32()
# True
```

Conceptually, `Arrow.int32` is a convenience alias/facade over `pa.int32`, not an ArrowBound implementation of `int32`.

Parameterized helpers follow the same rule:

```python
Arrow.timestamp("ns", tz="UTC") == pa.timestamp("ns", tz="UTC")
Arrow.decimal128(18, 6) == pa.decimal128(18, 6)
Arrow.large_list(pa.string()) == pa.large_list(pa.string())
```

Users may freely mix `Arrow.*` and `pa.*` types within the same model.

## Direct PyArrow support

Any compatible built-in `pyarrow.DataType` is a first-class explicit type annotation. It does not need to be wrapped in `Arrow(...)`.

```python
import pyarrow as pa

class Record(BaseModel):
    small: Annotated[int, pa.int16()]
    count: Annotated[int, Arrow.uint64()]
    payload: Annotated[bytes, pa.large_binary()]
```

This direct path is also the primary forward-compatibility escape hatch for newly introduced PyArrow datatypes that ArrowBound has not yet exposed through `Arrow.*`.

## Role of the Arrow namespace

`Arrow.*` exists for discoverability, a stable ArrowBound-facing convenience surface, documentation, and optional compatibility helpers. It must not become a second type system.

If an `Arrow.*` datatype helper is available, its successful result should be the same PyArrow datatype the corresponding `pa.*` call would produce.

This means datatype construction itself is not lazy. If a type factory does not exist in the installed PyArrow version, direct `pa.*` usage may naturally fail before ArrowBound can provide model-level diagnostics. ArrowBound may provide separate capability helpers such as `Arrow.supports(...)` or `Arrow.require(...)` if useful, without changing datatype identity.

## Python↔Arrow compatibility

Explicit Arrow representations must agree with the Python/Pydantic representation regardless of whether they came from `Arrow.*` or `pa.*`.

Valid:

```python
value: Annotated[int, Arrow.int16()]
value2: Annotated[int, pa.int16()]
```

Invalid:

```python
value: Annotated[str, pa.int16()]
```

ArrowBound should maintain compatibility rules at the semantic family level rather than merely comparing exact types.

## Nested models

Nested contract models must inherit from `arrowbound.BaseModel`. They map naturally to Arrow structs. Arbitrary nested `pydantic.BaseModel` classes should be rejected because ArrowBound cannot guarantee their complete substrate semantics.

## Complete PyArrow coverage goal

ArrowBound should strive to support every built-in schema-definable datatype exposed by the installed supported PyArrow runtime, including null/boolean, all integer and floating widths, binary/string variants and views, decimals, temporal types, lists and list views, maps, structs, unions, dictionary encoding, run-end encoding, canonical extension types, and future built-in PyArrow datatypes.

Not all types need a Python shorthand or an `Arrow.*` helper immediately. Full coverage can be achieved through documented Python defaults plus first-class direct `pyarrow.DataType` support.

## Ambiguous mappings

ArrowBound should fail rather than guess when no stable Python default can be defined. For example, `Decimal` requires an explicit documented policy for precision and scale. If Pydantic metadata cannot deterministically supply enough information, ArrowBound should require an explicit Arrow representation instead of silently inventing precision.

## Stability policy

Changing a default Python→Arrow mapping is a schema compatibility change and should be treated accordingly. Default mappings should be versioned and documented with the same care as a public serialization format.
