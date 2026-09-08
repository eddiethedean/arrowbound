# ArrowBound Roadmap

## V0.1 — Foundation

Prove the core architecture with constrained `BaseModel`, documented Python→Arrow defaults, real PyArrow-backed `Arrow.*` convenience helpers, direct `pyarrow.DataType` support, runtime PyArrow capability detection, `arrow_schema()`, nested ArrowBound models, natural Pydantic constraint extraction/enforcement at model boundaries, versioned ArrowBound constraint metadata, metadata escape hatches, clear errors, and initial capability audit infrastructure.

## V0.2 — Complete built-in type coverage

Systematically close the PyArrow capability matrix across numeric, binary/string, decimal, temporal, collection, map, struct, union, dictionary, run-end encoded, canonical extension, and other built-in schema-definable types available in supported runtimes.

Every type must have a known capability classification and no unexplained gaps.

## V0.3 — Complete declarative constraint coverage

Audit Pydantic's applicable declarative constraint surface. Every semantic must be classified as `ARROW_NATIVE`, `PORTABLE`, `LOCAL`, or `UNSUPPORTED` with a technical reason. Complete the ArrowBound portable constraint vocabulary and stabilize canonical serialization.

## V0.4 — Advanced Arrow semantics

Harden interoperability for difficult Arrow representations including unions, dictionary encoding, run-end encoding, list views, nested advanced types, and canonical extension types. Define clear Python/Pydantic compatibility rules for each.

## V0.5 — Portability specification

Extract the ArrowBound metadata representation into an implementation-independent specification covering namespace, versioning, canonical serialization, constraint vocabulary, descriptive metadata, custom namespaces, unknown semantics, and forward compatibility.

At this stage, another Arrow ecosystem implementation should theoretically be able to read ArrowBound semantics without Python or Pydantic.

## V0.6 — Optional Pandera dataframe validation

Add an optional Pandera integration that lets Pandera take over efficient dataframe/table validation while ArrowBound remains the single source of truth for the contract.

The primary API should be:

```python
class Customer(BaseModel):
    id: int
    age: Annotated[int, Field(ge=0, le=150)]

pandera_schema = Customer.pandera_schema()
validated = pandera_schema.validate(table)
```

`pandera_schema()` should compile the ArrowBound model into Pandera's underlying schema/check representation for the PyArrow backend. It should not require users to maintain a separate Pandera `DataFrameModel` class.

The intended division of responsibility is:

```text
ArrowBound model
      │
      ├── Pydantic record validation at ingress
      ├── pyarrow.Schema + portable constraint metadata
      └── pandera_schema()
                │
                ▼
             Pandera
                │
                ▼
       dataframe/table validation
```

ArrowBound continues to define and transport the contract. Pandera becomes an optional execution backend for validating already-columnar data against the applicable portions of that contract.

The integration should compile from ArrowBound's normalized constraint representation rather than independently reinterpreting Pydantic fields. This keeps one source of truth:

```text
Pydantic declaration
        ↓
ArrowBound normalized contract
        ├────────► Arrow metadata
        └────────► Pandera checks
```

Pandera should remain an optional dependency, likely through an extra such as `arrowbound[pandera]`. Core ArrowBound must not require Pandera.

The adapter should clearly classify semantics that Pandera can enforce, cannot enforce, or that require custom checks. Unsupported translations must fail or report explicitly rather than silently dropping constraints.

ArrowBound should not duplicate Pandera's dataframe-validation capabilities. Features such as vectorized column checks, dataframe-wide checks, uniqueness scans, lazy aggregation of validation failures, and other bulk validation behavior should remain Pandera's responsibility.

## V0.9 — Release audit

Before 1.0, audit every relevant PyArrow datatype and applicable Pydantic declarative constraint; verify default mappings, metadata IPC round trips, boundary validation, minimum/latest PyArrow CI, diagnostics, and intentional unsupported cases; eliminate unexplained gaps. The Pandera integration may mature independently and must not block the core substrate if its dependency/API stability requires more time.

## V1.0 — Stable substrate

1.0 means the core data-contract substrate is stable: deterministic defaults, complete capability classifications, stable explicit PyArrow authoring, reliable runtime compatibility, enforced Pydantic ingress constraints, stable portable metadata, deterministic schemas, and ordinary Apache Arrow interoperability.

The optional Pandera integration provides a natural dataframe-validation path but does not redefine ArrowBound's core responsibility.

## Post-1.0 discipline

ArrowBound should resist feature expansion outside contract definition and boundary validation. Execution features should normally be delegated to mature consumers such as Pandera rather than recreated in ArrowBound.

The core product remains a constrained Pydantic model system for defining validated, portable Apache Arrow data contracts.

> **Validate at the boundary. Preserve the contract. Transport with Arrow.**
