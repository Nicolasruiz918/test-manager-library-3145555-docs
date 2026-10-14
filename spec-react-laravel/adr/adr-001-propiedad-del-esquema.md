# ADR-001 · The application migrations own the schema

- **Status:** accepted in the baseline; translated to Laravel. Additional audit resolutions below remain proposals, not evaluator approval or implementation evidence.
- **Source:** [baseline ADR-001](../../spec-.net/adr/adr-001-propiedad-del-esquema.md), [constitution V](../constitution.md) and the [README](../../README.md).
- **Context:** `infra` starts a database engine. Putting business DDL both in container startup scripts and application migrations creates competing schema owners.
- **Decision preserved:** all business DDL belongs to the application's versioned **Laravel migrations**, including the baseline tables, indexes, constraints and fixed category seed. `infra` provides the empty engine, user and volume. Replacing EF Core with Laravel changes the tool, not the business schema.
- **Alternatives rejected:** DDL in `infra/db/init/`, which creates another source of truth; ORM-driven automatic table creation at startup, which bypasses explicit migration history.
- **Consequences:** if a PostgreSQL extension or index is actually required by an approved design, its application migration owns it. This principle does not authorize any new object beyond [data-model.md](../data-model.md). A manual schema change is a discrepancy to investigate and resolve through the controlled design, not an automatically valid variant.
- **Verification:** `infra` contains no business table DDL; migration results match the read-only inspection in [data-model.md §10](../data-model.md#10-verification-against-the-database). Verification remains pending until executable migrations and evidence exist.

Laravel convention-based migrations must not add plural business tables, incrementing business identifiers, generic timestamps or framework example tables. The six-table model remains unchanged.

## Audit findings and proposed resolutions

| Priority / type | Baseline evidence | Problem | Scoped resolution and compatibility review |
|---|---|---|---|
| P2 · Implementation trap | [ADR-001 decision](../../spec-.net/adr/adr-001-propiedad-del-esquema.md), [model §3](../../spec-.net/data-model.md#3-modelo-físico) | Framework defaults can silently create an incompatible schema | Adapt migration definitions to the exact baseline; review generated schema before declaring T-02 complete. No new tables, columns, indexes or constraints are authorized |
| P2 · Definition gap | [Model: six entities](../../spec-.net/data-model.md#2-entidades-y-relaciones), [README: Laravel](../../README.md#el-reto-se-desarrolla-con-react-y-php-laravel) | The baseline does not define Laravel migration bookkeeping | Document the migrator's requirements and obtain an explicit compatibility decision before introducing technical schema objects. No bookkeeping table or additional schema is specified in this ADR |
| P2 · Definition gap | [ADR-001 consequences](../../spec-.net/adr/adr-001-propiedad-del-esquema.md), [constitution X](../../spec-.net/constitution.md#artículo-x--gana-el-motor) | Manual database changes are defects in one source but may appear authoritative under “the engine wins” | Treat inspection as evidence of actual state; determine whether the implementation or document deviates from the expected design before correcting it. Preserve migration ownership and business rules |

This file contains no migration, DDL, seed implementation or claim that the database has been created.
