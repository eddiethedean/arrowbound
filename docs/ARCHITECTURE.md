# ArrowBound Architecture

## Architectural position

ArrowBound is a contract-definition and preservation layer, not an execution engine.

```text
Application developer
        │
        ▼
ArrowBound model
        │
        ├── Python/Pydantic authoring
        ├── native PyArrow datatype resolution
        └── ArrowBound portable constraints/metadata
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

Apache Arrow carries the native schema and ArrowBound metadata. Apache Arrow does not interpret or enforce ArrowBound portable constraints.

Pydantic may enforce applicable constraints when ArrowBound model instances are validated. Downstream ArrowBound-aware consumers may choose to interpret or enforce the portable constraint metadata.

The core package ends at contract definition and preservation.

## Model hierarchy

ArrowBound owns its model tree: `pydantic.BaseModel` → `arrowbound.BaseModel` → user contract models. An arbitrary `pydantic.BaseModel` is not a valid ArrowBound contract. Nested models must also inherit from `arrowbound.BaseModel` so the entire schema tree satisfies ArrowBound's deterministic representation guarantee.

## Compilation pipeline

```text
User model
   │
   ▼
arrowbound.BaseModel
   │
   ├── inspect Python annotations
   ├── inspect Pydantic field semantics
   ├── resolve documented default Arrow types
   ├── accept explicit Arrow.* or pa.* DataTypes
   ├── validate Python↔Arrow compatibility
   ├── normalize portable declarative constraints
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

## PyArrow is the type system

ArrowBound does not define wrapper datatypes. The physical/logical type substrate is `pyarrow.DataType`.

`Arrow.*` is a convenience namespace over PyArrow factories and returns actual PyArrow datatypes. Direct `pa.*` datatypes are equally first class.

```python
Arrow.int32() == pa.int32()
```

A model may freely mix the two forms. This prevents ArrowBound from becoming a second Arrow type system and gives users immediate access to newly introduced PyArrow datatypes when ArrowBound can validate them generically.

## Separation of concerns

ArrowBound has three contract dimensions:

### Arrow schema semantics

The physical/logical representation belongs to Apache Arrow and is represented by `pyarrow.DataType`, field nullability, and other native schema properties.

### ArrowBound portable constraints

Declarative validity rules such as minimums, maximums, lengths, patterns, and allowed-value sets that Apache Arrow does not provide as a general-purpose constraint system. ArrowBound normalizes these rules into a language-neutral metadata vocabulary.

### ArrowBound metadata

Descriptive semantics such as unit, semantic type, or domain annotations. Metadata is not automatically treated as validation.

These must remain distinct.

## Native Arrow semantics first

ArrowBound never duplicates information already represented by Arrow. `T | None` maps to field nullability; decimal precision/scale stay in the decimal datatype; timestamp timezone/unit stay in the timestamp datatype; list item types stay in the list datatype; dictionary encoding stays in the dictionary datatype. ArrowBound metadata fills only semantic gaps.

## Definition versus enforcement

ArrowBound defines and preserves portable constraints. It does not make Apache Arrow enforce them.

The enforcement model is intentionally layered:

```text
ArrowBound model definition
        │
        ├── Pydantic may enforce constraints locally
        │
        ▼
Arrow schema + ArrowBound metadata
        │
        ▼
Apache Arrow transports the contract
        │
        ▼
Downstream consumer may enforce if ArrowBound-aware
```

A future dedicated validator could consume ArrowBound schemas, but that would be a separate concern from the core contract-definition layer unless explicitly added later.

## Early validation

ArrowBound should reject invalid contract definitions as early as practical, ideally during model class construction. This includes unsupported arbitrary Python types, arbitrary nested Pydantic models, incompatible Python and explicit PyArrow types, conflicting annotations, unsupported unions, and malformed custom constraint metadata.

A datatype factory that does not exist in the installed PyArrow may fail at the point the user calls it. ArrowBound should not introduce lazy wrapper datatypes solely to intercept this. Separate capability helpers may provide preflight diagnostics.

## Failure behavior

ArrowBound-controlled errors should identify the model, full nested field path, Python annotation, requested Arrow representation, installed PyArrow version when relevant, unsupported/unavailable capability, and recommended remediation. ArrowBound must never silently substitute a different representation.

## Forward compatibility

A future PyArrow type may not yet have an `Arrow.some_new_type()` convenience helper. Users should still be able to pass the actual `pa.some_new_type(...)` result directly where ArrowBound can safely preserve and validate it.

This direct PyArrow path is a foundational forward-compatibility mechanism.

## Architectural non-goals

Do not add dataframe execution, query compilation, persistence, database migration, schema registries, engine-specific adapters, general Arrow-table constraint enforcement, or plugin systems to the core without a separate project-level decision. These concerns may consume ArrowBound schemas externally.
