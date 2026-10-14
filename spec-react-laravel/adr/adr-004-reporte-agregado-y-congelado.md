# ADR-004 · The database aggregates reports using snapshot values

- **Status:** accepted in the baseline; translated for React/Laravel. Proposed clarifications below are not implementation evidence or evaluator approval.
- **Source:** [baseline ADR-004](../../spec-.net/adr/adr-004-reporte-agregado-y-congelado.md), [HU-10](../../spec-.net/spec.md#hu-10--reporte-de-libros-más-prestados) and [baseline HTTP report](../../spec-.net/api-contract.md#37-reporte).
- **Context:** the most-borrowed report joins loans and items over a date range. Reading titles or categories from the current catalog would change historical labels after an edit.
- **Decision preserved:** the report is not persisted. A read port exposes a single database aggregation implemented in the Laravel persistence adapter. It groups by **`book_id`, `book_title`, `category_name`** captured in the loan item. `loan_item.category_name` deliberately has no FK. If snapshot values differ within the range, the report retains separate groups rather than choosing a winning value.
- **Alternatives rejected:** joining the live catalog for labels; selecting the latest value in the range; materializing a report table as another source of truth.
- **Consequences:** the report can contain more than one row for a book identity when its captured title or category differs. It includes no breakdown or personal data by member or operator. React displays the server's result; it does not replace database aggregation with client-side counting.
- **Verification preserved:** lend a book, rename it and request the same historical range; the report retains the title captured in the original item. Evidence remains pending until the use case, adapter and checks exist.

The existing contract defines inclusive `from` and `to` dates, ordering by count descending and then title, an empty result for an empty range, and 422 for an inverted range. These external rules are preserved. Returns and logical deletion do not rewrite the captured loan event; the report counts recorded loan items rather than current availability.

## Audit findings and proposed resolutions

| Priority / type | Baseline evidence | Problem | Scoped resolution and compatibility review |
|---|---|---|---|
| P2 · Apparent contradiction resolved by existing evidence | [CA-10.1](../../spec-.net/spec.md#hu-10--reporte-de-libros-más-prestados), [contract §3.7](../../spec-.net/api-contract.md#37-reporte), [ADR-004 decision](../../spec-.net/adr/adr-004-reporte-agregado-y-congelado.md) | The shorter criterion mentions book/category, while the contract and ADR explicitly include the captured title | Preserve the three-key grouping already specified. Multiple rows for one book are expected when snapshots differ; no new grouping or report column is added |
| P2 · Implementation trap | [ADR-004 decision](../../spec-.net/adr/adr-004-reporte-agregado-y-congelado.md), [model snapshots](../../spec-.net/data-model.md#3-modelo-físico) | An ORM relationship or catalog join can silently replace historical labels or exclude logically deleted books | Aggregate `loan` and `loan_item` using their recorded values; retain historical items. Verify rename, category change and logical deletion without altering the schema |
| P2 · Definition gap | [Contract inclusive report dates](../../spec-.net/api-contract.md#37-reporte), [RN-6 UTC](../../spec-.net/spec.md#4-reglas-de-negocio) | The physical interpretation of inclusive date boundaries is not spelled out | Recommend a UTC interval from the start of `from` through, but excluding, the start of the day after `to`; verify fractional-second boundaries. This is a proposed realization of existing inclusive dates, not a new public filter or response |
| P2 · Implementation trap | [CA-10.4](../../spec-.net/spec.md#hu-10--reporte-de-libros-más-prestados) | “Closed period” may be interpreted as requiring a new persisted accounting closure | Read the guarantee in its stated context: later catalog changes do not rewrite past labels. No period-closing operation, table or immutable report snapshot is introduced |

Any proposal changing report fields, grouping, authorization or date semantics requires compatibility review against the original contract. This document makes none of those external changes.
