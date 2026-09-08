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

Default mappings must be:

- stable,
- documented,
- deterministic,
- independent of runtime data values.

ArrowBound must not inspect values and opportunistically narrow a type.

For example:

```python
count: int
```

always maps to the documented integer default. It must not become `int8`, `uint16`, or another narrower integer depending on observed values.

## Constraints never change physical types implicitly

```python
quantity: Annotated[int, Field(ge=0)]
```

still maps to `int64` by default. The non-negative rule is a constraint, not a request for `uint64`.

Changing a validation rule must not unexpectedly change an interchange schema.

## Explicit Arrow escape hatch

Users who need exact Arrow semantics may opt in:

```python
from typing import Annotated

from arrowbound import Arrow, BaseModel


class Measurement(BaseModel):
    sequence: Annotated[int, Arrow.uint32()]
    value: Annotated[float, Arrow.float32()]
```

Parameterized examples:

```python
Arrow.timestamp("ns", tz="UTC")
Arrow.decimal128(18, 6)
Arrow.fixed_size_binary(16)
Arrow.large_list(Arrow.string())
Arrow.dictionary(Arrow.int16(), Arrow.string(), ordered=True)
```

The `Arrow.*` surface should be a thin lazy interface over PyArrow factories rather than a competing type implementation.

## Direct PyArrow escape hatch

Advanced users may provide an already-created `pyarrow.DataType`:

```python
import pyarrow as pa


class Record(BaseModel):
    value: Annotated[int, Arrow(pa.int32())]
```

This is especially useful for new PyArrow datatypes that ArrowBound has not yet wrapped ergonomically.

## Python↔Arrow compatibility

Explicit Arrow representations must agree with the Python/Pydantic representation.

Valid:

```python
value: Annotated[int, Arrow.int16()]
```

Invalid:

```python
value: Annotated[str, Arrow.int16()]
```

ArrowBound should maintain compatibility rules at the semantic family level rather than merely comparing exact types.

## Nested models

Nested contract models must inherit from `arrowbound.BaseModel`:

```python
class Coordinates(BaseModel):
    latitude: float
    longitude: float


class Event(BaseModel):
    location: Coordinates
```

This maps naturally to an Arrow struct. Arbitrary nested `pydantic.BaseModel` classes should be rejected because ArrowBound cannot guarantee their complete substrate semantics.

## Complete PyArrow coverage goal

ArrowBound should strive to support every built-in schema-definable datatype exposed by the installed supported PyArrow runtime, including:

- null and boolean,
- all signed and unsigned integer widths,
- all floating-point widths,
- binary, large binary, binary views, fixed-size binary,
- string, large string, string views,
- decimal families,
- dates, times, timestamps, durations, intervals,
- lists, large lists, fixed-size lists, list views and large list views,
- maps and structs,
- sparse and dense unions,
- dictionary encoding,
- run-end encoding,
- canonical extension types,
- future built-in PyArrow datatypes where the runtime exposes them.

Not all types need a Python shorthand. Full coverage can be achieved through a combination of documented defaults, `Arrow.*` helpers, and direct `pyarrow.DataType` escape hatches.

## Ambiguous mappings

ArrowBound should fail rather than guess when no stable default can be defined.

For example, `Decimal` requires an explicit documented policy for precision and scale. If Pydantic metadata cannot deterministically supply enough information, ArrowBound should require an explicit Arrow representation instead of silently inventing precision.

## Stability policy

Changing a default Python→Arrow mapping is a schema compatibility change and should be treated accordingly. Default mappings should be versioned and documented with the same care as a public serialization format.
