# PyArrow Compatibility Plan

## Principle

ArrowBound should dynamically use the capabilities of the installed PyArrow runtime rather than freezing itself to the exact PyArrow feature surface available when an ArrowBound release was published.

> **Installed PyArrow determines available Arrow capabilities. ArrowBound provides the authoring rules, compatibility checks, and diagnostics around them.**

## Minimum supported PyArrow

ArrowBound should declare a minimum supported PyArrow version for maintainability and testability.

Above that floor, ArrowBound should generally avoid unnecessary upper bounds so newer PyArrow releases can expose newer capabilities.

Release metadata should declare:

- minimum Python version,
- supported Pydantic range,
- minimum PyArrow version,
- ArrowBound metadata specification version.

## Capability detection over version branching

Prefer runtime capability checks such as:

```python
factory = getattr(pyarrow, "string_view", None)
```

over widespread logic such as:

```python
if Version(pyarrow.__version__) >= ...:
    ...
```

The presence of the public PyArrow capability is the primary source of truth. Version knowledge is still useful for diagnostics and upgrade guidance.

## Lazy Arrow type specifications

Explicit ArrowBound helpers should resolve lazily.

```python
Arrow.string_view()
```

should create an ArrowBound type specification rather than immediately calling `pyarrow.string_view()`.

This lets ArrowBound control the error path when a capability is missing.

Example error direction:

```text
ArrowBoundUnsupportedTypeError

Arrow type 'string_view' is unavailable in installed PyArrow 15.x.

This capability requires a newer PyArrow release.
Upgrade PyArrow or choose another explicit Arrow representation.
```

Where ArrowBound knows the exact minimum version, it should include it.

## No silent substitution

If a developer asks for an exact type, ArrowBound must honor that exact request or fail.

Examples of forbidden behavior:

- replacing `string_view` with `string`,
- replacing `decimal32` with `decimal128`,
- replacing a requested list-view representation with a normal list,
- changing a requested integer width to another width.

The requested Arrow contract is authoritative.

## Forward compatibility

ArrowBound should distinguish:

1. first-class ergonomic support via `Arrow.*`, and
2. fundamental ability to carry a built-in `pyarrow.DataType`.

When a future PyArrow release adds a datatype before ArrowBound adds a helper, advanced users should be able to supply an already-created `pyarrow.DataType` where compatibility can be validated generically.

ArrowBound can later add:

- ergonomic helper syntax,
- Python compatibility rules,
- documentation,
- test coverage,
- known minimum-version diagnostics.

## Compatibility axes

ArrowBound has three independent compatibility surfaces:

### Pydantic

Determines the Python authoring and local validation semantics.

### PyArrow

Determines which Arrow datatypes and constructors are available in the installed runtime.

### ArrowBound metadata specification

Determines which portable constraints and descriptive semantics ArrowBound-aware consumers can interpret.

These versions should remain intentionally independent.

## CI matrix

CI should include at least:

```text
minimum supported PyArrow
latest stable PyArrow
newer/pre-release/nightly PyArrow where practical (non-blocking initially)
```

The purpose of the newest-runtime job is to identify upcoming API changes and newly available Arrow types early.

## Environment diagnostics

A small diagnostic API may be useful later, for example:

```python
arrowbound.compatibility()
```

returning package/runtime metadata such as ArrowBound, PyArrow, Pydantic, and metadata-spec versions.

This is secondary to the core API and should not be required for normal use.

## Upgrade guidance

Compatibility errors should be actionable. When ArrowBound knows a missing type's first supported PyArrow release, errors should state the requirement rather than merely saying the type is missing.

The capability registry should hold this compatibility knowledge for diagnostics, tests, and generated documentation, while PyArrow runtime introspection remains authoritative for actual availability.
