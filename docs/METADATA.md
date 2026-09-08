# ArrowBound Metadata Specification Plan

## Purpose

Apache Arrow carries the native schema and provides the transport/container for ArrowBound metadata. ArrowBound metadata exists only for portable semantics Arrow does not natively encode.

The metadata layer must remain small, deterministic, language-neutral, and independently versioned from the ArrowBound package and PyArrow runtime.

Important distinction:

> **Apache Arrow carries ArrowBound metadata. Apache Arrow does not interpret or enforce ArrowBound portable constraints.**

## Native Arrow semantics first

ArrowBound must not duplicate native Arrow information.

Do not encode ArrowBound metadata for semantics already represented by:

- `Field.nullable`,
- decimal precision/scale,
- timestamp unit/timezone,
- list element datatypes,
- map key/item datatypes,
- struct fields,
- dictionary encoding,
- union mode/type codes,
- other native datatype parameters.

These are Arrow schema semantics, not ArrowBound portable constraints.

## Metadata dimensions

ArrowBound should distinguish:

### Portable constraints

Declarative rules governing valid values that Arrow itself does not provide as a general constraint system.

Examples:

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
```

ArrowBound normalizes and preserves these rules. Pydantic may enforce them in Python; downstream consumers may enforce them if they understand the ArrowBound metadata specification.

### Descriptive metadata

Semantics that describe values without necessarily making them valid or invalid.

Examples:

```text
unit = degrees
semantic_type = latitude
```

### Custom namespaced metadata

Application/domain-specific semantics that ArrowBound preserves but does not interpret.

## Versioning

Schemas should identify the ArrowBound metadata specification version independently from package versions.

Conceptually:

```text
arrowbound.metadata.version = 1
```

This version is distinct from ArrowBound package version, Pydantic version, PyArrow version, and Arrow format/IPC evolution.

## Serialization

The metadata encoding should be deterministic, language-neutral, serializable through Arrow schema/field metadata, namespace-safe, and stable under IPC round trips.

Canonical JSON is the leading candidate for structured metadata because it is widely implementable across languages. If used, ArrowBound should define canonicalization rules rather than relying on arbitrary JSON serializer output.

Potential conceptual layout:

```json
{
  "version": 1,
  "constraints": {
    "minimum": 0,
    "maximum": 100
  },
  "metadata": {
    "unit": "percent"
  }
}
```

The exact keys and byte-level layout should be frozen only after interoperability tests.

## Determinism requirements

Equivalent contract semantics must produce equivalent serialized metadata.

This requires stable rules for key ordering, numeric serialization, decimal serialization, allowed-value ordering where semantics allow, Unicode normalization policy if needed, nested metadata ordering, and omission of empty/default values.

## Unknown metadata behavior

An Arrow consumer unaware of ArrowBound should still be able to read the ordinary Arrow schema and data while ignoring ArrowBound metadata.

An ArrowBound-aware consumer encountering a newer metadata version or unknown namespaced constraint should preserve what it does not understand where possible and must not pretend to enforce unknown semantics.

## Custom constraint namespaces

Domain-specific custom constraints should be explicitly namespaced:

```python
ArrowConstraint(
    namespace="com.example",
    name="quality_code",
    value={...},
)
```

ArrowBound-reserved names should use an `arrowbound` namespace. Third parties must not write into the reserved namespace.

## Raw Arrow metadata escape hatch

Advanced users may need complete control over field/schema metadata. ArrowBound should expose a low-level mechanism conceptually similar to:

```python
Arrow.metadata(...)
```

Raw metadata must remain distinguishable from ArrowBound's standardized constraint payload so arbitrary user bytes cannot be mistaken for ArrowBound-defined semantics.

## Cross-language goal

By the time the metadata format stabilizes, the specification should be sufficient for an implementation in Rust, Java, Go, or another Arrow ecosystem language to interpret ArrowBound portable constraints without importing Python or Pydantic.

That consumer may choose to validate or enforce those constraints, but enforcement is outside the Arrow metadata transport itself.

That is the test for whether the metadata is truly a portable contract rather than Python implementation detail.
