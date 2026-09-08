# PyArrow Compatibility Plan

## Principle

ArrowBound dynamically uses the capabilities of the installed PyArrow runtime rather than freezing itself to the exact PyArrow feature surface available when an ArrowBound release was published.

> **Installed PyArrow determines available Arrow capabilities. ArrowBound provides authoring rules, compatibility checks, and diagnostics around them.**

## Minimum supported PyArrow

ArrowBound declares a minimum supported PyArrow version for maintainability and testability. Above that floor, it should generally avoid unnecessary upper bounds so newer PyArrow releases can expose newer capabilities.

Release metadata should declare minimum Python, supported Pydantic range, minimum PyArrow, and ArrowBound metadata-spec version.

## PyArrow datatypes are first class

ArrowBound must accept compatible `pyarrow.DataType` instances directly in `Annotated` metadata. Users do not need to wrap them in an ArrowBound object.

```python
value: Annotated[int, pa.int32()]
```

This direct PyArrow path is a core part of ArrowBound's compatibility strategy, not merely an undocumented escape hatch.

## Arrow.* preserves PyArrow datatype identity

`Arrow.*` datatype helpers are a convenience facade over PyArrow factories. They return actual PyArrow datatypes rather than lazy ArrowBound datatype wrappers.

```python
Arrow.int32() == pa.int32()
Arrow.timestamp("us") == pa.timestamp("us")
```

Users may freely mix `Arrow.*` and `pa.*` declarations. ArrowBound applies the same Python↔Arrow compatibility validation to both.

## Capability detection

ArrowBound should still prefer runtime capability detection over broad version branching when inspecting what the installed PyArrow supports.

```python
factory = getattr(pyarrow, "string_view", None)
```

The presence of the public PyArrow capability is the primary source of truth. Version knowledge remains useful for diagnostics, generated documentation, and upgrade guidance.

Because `Arrow.*` returns real PyArrow datatypes, datatype construction is intentionally not lazy. A direct call to a factory absent from the installed PyArrow may fail before model construction. ArrowBound should not sacrifice datatype identity merely to intercept that error path.

If better preflight diagnostics are useful, add separate helpers such as:

```python
Arrow.supports("string_view")
Arrow.require("string_view")
```

These helpers may provide version-aware guidance without changing what `Arrow.string_view()` returns when available.

## No silent substitution

If a developer asks for an exact type, ArrowBound must honor that exact request or fail. It must never replace string views with strings, one decimal width with another, list views with normal lists, or requested integer widths with other widths.

## Forward compatibility

ArrowBound distinguishes ergonomic first-class support via `Arrow.*` from fundamental support for a `pyarrow.DataType`.

When a future PyArrow release adds a datatype before ArrowBound adds a corresponding helper, users should be able to use the PyArrow datatype directly where ArrowBound can validate its Python representation generically.

A later ArrowBound release can add convenience aliases, compatibility knowledge, documentation, and dedicated tests without having blocked early adopters.

## Compatibility axes

ArrowBound has three independent compatibility surfaces:

- **Pydantic:** Python authoring and local validation semantics.
- **PyArrow:** available Arrow datatypes and constructors in the installed runtime.
- **ArrowBound metadata specification:** portable constraints and descriptive semantics understood by ArrowBound-aware consumers.

These versions remain intentionally independent.

## CI matrix

CI should include the minimum supported PyArrow, latest stable PyArrow, and a newer/pre-release/nightly PyArrow where practical (initially non-blocking). Tests must cover both `Arrow.*` and direct `pa.*` annotations and assert equivalent resulting schemas.

## Environment diagnostics

A small diagnostic API may later expose ArrowBound, PyArrow, Pydantic, and metadata-spec versions. This remains secondary to the core contract-definition API.

## Upgrade guidance

When ArrowBound itself detects an unavailable capability and knows its first supported PyArrow release, errors should state the requirement. The capability registry holds this compatibility knowledge for diagnostics, tests, and generated documentation, while runtime PyArrow introspection remains authoritative for actual availability.
