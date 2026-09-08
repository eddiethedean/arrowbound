# ArrowBound Architecture

## Architectural position

ArrowBound is a contract-definition layer, not an execution engine.

```text
Application developer
        │
        ▼
ArrowBound model
        │
        ├── Python/Pydantic authoring
        ├── Arrow type resolution
        └── portable constraints/metadata
        │
        ▼
pyarrow.Schema
        │
        ▼
Apache Arrow interchange
        │
 ┌──────┼────────┬────────┐
 ▼      ▼        ▼        ▼
Polars DuckDB   Spark   other systems
```

The package ends where the Arrow contract begins.

## Model hierarchy

ArrowBound must own its model tree:

```text
pydantic.BaseModel
        ↑
arrowbound.BaseModel
        ↑
user contract models
```

An arbitrary `pydantic.BaseModel` is not a valid ArrowBound contract. Nested models must also inherit from `arrowbound.BaseModel` so the entire schema tree satisfies ArrowBound's deterministic representation guarantee.

## Compilation pipeline

```text
User model
   │
   ▼
arrowbound.BaseModel
   │
   ├── inspect Python annotations
   ├── inspect Pydantic field semantics
   ├── resolve explicit Arrow escape hatches
   ├── validate Python↔Arrow compatibility
   ├── normalize portable constraints
   └── collect descriptive/custom metadata
   │
   ▼
Arrow schema compiler
   │
   ├── native pyarrow.DataType
   ├── native field nullability
   ├── native nested structure
   └── ArrowBound metadata only for missing semantics
   │
   ▼
pyarrow.Schema
```

## Separation of concerns

ArrowBound has three contract dimensions:

### Type

The physical/logical representation belongs to Apache Arrow and is represented by `pyarrow.DataType`.

### Constraint

A declarative statement about valid values. ArrowBound normalizes portable Pydantic constraints into a language-neutral metadata vocabulary when Arrow has no native equivalent.

### Metadata

Descriptive semantics such as unit, semantic type, or domain annotations. Metadata is not automatically treated as validation.

These must remain distinct.

## Native Arrow semantics first

ArrowBound should never duplicate information already represented by Arrow.

Examples:

- `T | None` maps to Arrow field nullability.
- decimal precision and scale belong in the decimal datatype.
- timestamp timezone and unit belong in the timestamp datatype.
- list item types belong in the list datatype.
- dictionary encoding belongs in the dictionary datatype.

ArrowBound metadata should only fill semantic gaps.

## Early validation

ArrowBound should reject invalid contract definitions as early as practical, ideally during model class construction.

Examples of errors that should be caught early:

- unsupported arbitrary Python types,
- arbitrary nested Pydantic models,
- incompatible Python and explicit Arrow types,
- invalid or conflicting Arrow annotations,
- unsupported unions,
- malformed custom constraint metadata.

Runtime PyArrow capability availability may also be resolved during model construction where safe, or deferred to schema resolution when lazy behavior is needed for import-time ergonomics.

## Failure behavior

Errors should identify:

- model,
- full nested field path,
- Python annotation,
- requested Arrow representation,
- installed PyArrow version when relevant,
- unsupported or unavailable capability,
- recommended remediation.

ArrowBound must never silently substitute a different Arrow representation.

## Forward compatibility

ArrowBound should distinguish ergonomic support from fundamental runtime support.

A future PyArrow type may not yet have an `Arrow.some_new_type()` helper in ArrowBound. Advanced users should still be able to pass an already-created `pyarrow.DataType` where ArrowBound can safely preserve and validate it.

This keeps ArrowBound thin over PyArrow instead of turning it into a second Arrow type system.

## Architectural non-goals

Do not add the following to the core architecture without a separate project-level decision:

- dataframe execution,
- query compilation,
- persistence,
- database schema migration,
- schema registry/network service,
- engine-specific adapters,
- arbitrary runtime data validation against Arrow tables,
- plugin systems.

These concerns may consume ArrowBound schemas externally.
