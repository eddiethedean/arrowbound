# ArrowBound Architecture

## Architectural position

ArrowBound is a contract-definition and validation-boundary layer, not a general Arrow execution engine.

```text
Application input
        │
        ▼
ArrowBound model
        │
        ├── Python/Pydantic authoring
        ├── Pydantic ingress validation
        ├── native PyArrow datatype resolution
        └── portable constraint/metadata compilation
        │
        ▼
validated values + pyarrow.Schema
        │
        ▼
Apache Arrow interchange
        │
 ┌──────┼────────┬────────┐
 ▼      ▼        ▼        ▼
Polars DuckDB   Spark   other systems
```

ArrowBound establishes contract validity where data crosses an ArrowBound model-validation boundary, then preserves the portable declarative contract for downstream systems.

## Model hierarchy

ArrowBound owns its model tree: `pydantic.BaseModel` → `arrowbound.BaseModel` → user contract models. An arbitrary `pydantic.BaseModel` is not a valid ArrowBound contract. Nested models must also inherit from `arrowbound.BaseModel` so the entire schema tree satisfies ArrowBound's deterministic representation guarantee.

## Compilation and validation pipeline

```text
User input
   │
   ▼
arrowbound.BaseModel
   │
   ├── inspect Python annotations
   ├── inspect Pydantic field semantics
   ├── validate values through Pydantic
   ├── resolve documented default Arrow types
   ├── accept explicit Arrow.* or pa.* DataTypes
   ├── validate Python↔Arrow compatibility
   ├── normalize portable constraints
   └── collect descriptive/custom metadata
   │
   ├──────────────► validated model values
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

The important invariant is temporal: data that successfully passes through ArrowBound model validation satisfied the applicable declared constraints **at that boundary**.

Arrow does not continuously revalidate those constraints after the fact. If data is mutated or introduced by another path, the previous validation guarantee does not automatically carry forward.

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

The physical/logical representation belongs to Apache Arrow and is represented by `pyarrow.DataType`, field nullability, nested structure, and other native schema parameters.

### ArrowBound portable constraints

Declarative statements about valid values that Arrow does not provide as a general constraint system. Pydantic enforces applicable ones at ArrowBound validation boundaries; ArrowBound serializes their portable meaning into versioned metadata.

### ArrowBound metadata

Descriptive semantics such as unit, semantic type, or domain annotations. Metadata is not automatically treated as validation.

These must remain distinct.

## Constraint communication model

```text
constraint authored in Pydantic
        │
        ├── enforcement at ArrowBound validation boundary
        │
        └── normalization to ArrowBound portable metadata
                            │
                            ▼
                      Arrow schema
                            │
                            ▼
                    downstream consumer
                            │
                  may inspect/re-enforce
```

This is the core value of ArrowBound beyond ordinary Arrow schema definition: the validity rules that were actually applied at ingress can also be communicated with the Arrow substrate instead of being lost at the Python boundary.

## Native Arrow semantics first

ArrowBound never duplicates information already represented by Arrow. `T | None` maps to field nullability; decimal precision/scale stay in the decimal datatype; timestamp timezone/unit stay in the timestamp datatype; list item types stay in the list datatype; dictionary encoding stays in the dictionary datatype. ArrowBound metadata fills only semantic gaps.

## Early validation

ArrowBound should reject invalid contract definitions as early as practical, ideally during model class construction. This includes unsupported arbitrary Python types, arbitrary nested Pydantic models, incompatible Python and explicit PyArrow types, conflicting annotations, unsupported unions, and malformed custom constraint metadata.

Value validation occurs through normal Pydantic model validation. A datatype factory that does not exist in the installed PyArrow may fail at the point the user calls it. ArrowBound should not introduce lazy wrapper datatypes solely to intercept this. Separate capability helpers may provide preflight diagnostics.

## Failure behavior

ArrowBound-controlled errors should identify the model, full nested field path, Python annotation, requested Arrow representation, installed PyArrow version when relevant, unsupported/unavailable capability, and recommended remediation. ArrowBound must never silently substitute a different representation.

## Forward compatibility

A future PyArrow type may not yet have an `Arrow.some_new_type()` convenience helper. Users should still be able to pass the actual `pa.some_new_type(...)` result directly where ArrowBound can safely preserve and validate it.

This direct PyArrow path is a foundational forward-compatibility mechanism.

## Architectural non-goals

Do not add dataframe execution, query compilation, persistence, database migration, schema registries, engine-specific adapters, general Arrow-table validation, or plugin systems to the core without a separate project-level decision.

ArrowBound validates at its own model boundaries and communicates the contract. It does not promise that arbitrary Arrow data remains valid forever or that every downstream engine automatically enforces ArrowBound metadata.
