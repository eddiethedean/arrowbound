# ArrowBound Capability Audit

## Purpose

ArrowBound should maintain an automated or semi-automated audit against the supported PyArrow datatype surface and the applicable Pydantic declarative constraint surface.

The goal is simple:

> **No unexplained gaps.**

## PyArrow datatype classifications

Every relevant built-in schema-definable PyArrow datatype should be classified as one of:

### DEFAULT

Naturally inferred from an ordinary Python/Pydantic annotation using ArrowBound's documented default mapping.

### SUPPORTED

Fully supported through an explicit ArrowBound declaration such as `Arrow.int32()` or another first-class helper.

### RUNTIME

Can be supplied directly as a `pyarrow.DataType` even when ArrowBound does not yet provide ergonomic first-class syntax.

### UNSUPPORTED

Cannot currently satisfy ArrowBound's contract guarantees and has a documented technical reason.

`UNSUPPORTED` must not simply mean "not implemented yet." Ordinary implementation gaps are incomplete feature work, not intentional unsupported behavior.

## Runtime availability

Availability depends on the installed PyArrow version.

ArrowBound should distinguish `UNAVAILABLE` from `UNSUPPORTED`.

A type may be fully supported by ArrowBound but unavailable in an older PyArrow runtime. In that case, diagnostics should say that the installed runtime lacks the capability and recommend the known required upgrade when possible.

Conceptually:

```text
Installed PyArrow
       │
       ▼
Discover runtime capabilities
       │
       ▼
ArrowBound capability registry
       │
       ├── DEFAULT
       ├── SUPPORTED
       ├── RUNTIME
       ├── UNAVAILABLE
       └── UNSUPPORTED
```

## Capability registry

ArrowBound should maintain lightweight compatibility metadata about known Arrow capabilities.

Conceptual shape:

```python
ArrowCapability(
    name="string_view",
    python_types=(str,),
    default=False,
    arrowbound_factory="string_view",
    pyarrow_factory="string_view",
    introduced_in="known-version-if-available",
)
```

The registry must not reimplement PyArrow datatypes. It exists for:

- Python/Pydantic compatibility checks,
- ergonomic `Arrow.*` helpers,
- error messages,
- documentation generation,
- testing,
- capability auditing.

Runtime capability detection remains authoritative.

## Audit scope

The audit should cover all applicable built-in families available in supported runtimes, including:

- null,
- boolean,
- signed integers,
- unsigned integers,
- floating point,
- binary and fixed-size binary variants,
- binary views,
- strings, large strings, string views,
- decimals,
- dates,
- times,
- timestamps,
- durations,
- intervals,
- lists,
- large lists,
- fixed-size lists,
- list views,
- large list views,
- maps,
- structs,
- sparse unions,
- dense unions,
- dictionary encoding,
- run-end encoding,
- canonical extension types,
- newly introduced built-in PyArrow datatypes.

## Generated documentation

The capability matrix should generate or validate user-facing documentation so code, tests, and docs do not maintain separate definitions of support.

Example:

| Arrow type | Python default | Explicit support | Runtime dependent |
| --- | --- | --- | --- |
| `int64` | `int` | `Arrow.int64()` | no |
| `uint32` | — | `Arrow.uint32()` | no |
| `string` | `str` | `Arrow.string()` | no |
| `large_string` | — | `Arrow.large_string()` | no |
| `string_view` | — | `Arrow.string_view()` | yes |
| `timestamp` | `datetime` | `Arrow.timestamp(...)` | parameters/runtime |

## Constraint capability audit

ArrowBound should apply the same rigor to Pydantic declarative constraints.

Every applicable constraint should be classified as:

### NATIVE

Semantics are already represented directly by Arrow.

### PORTABLE

ArrowBound can completely represent the semantics in its versioned portable metadata format.

### LOCAL

Pydantic can enforce the behavior in Python, but it cannot be represented as a portable declarative ArrowBound constraint.

### UNSUPPORTED

ArrowBound cannot safely interpret or preserve the semantics, with a documented technical reason.

Examples:

```text
Field(ge=0)              → PORTABLE
Field(max_length=100)    → PORTABLE
T | None                 → NATIVE
arbitrary field_validator → LOCAL
```

## Release gate

A stable release should not add or change PyArrow/Pydantic support without updating the audit.

For 1.0, every relevant built-in PyArrow datatype and applicable declarative Pydantic constraint must have a known classification. There should be no accidental gaps and no undocumented fallbacks.
