# ADR-003 · Books are logically deleted, not physically removed

- **Status:** accepted in the baseline; translated for React/Laravel. Audit proposals below are not implementation evidence or evaluator approval.
- **Source:** [baseline ADR-003](../../spec-.net/adr/adr-003-baja-logica.md), [CA-03.5 and CA-04.3](../../spec-.net/spec.md).
- **Context:** a loan item references a book. Physically deleting referenced catalog data can damage the history and reports.
- **Decision preserved:** a book has `deleted_at`; logical deletion fills it. Catalog access centrally filters `deleted_at IS NULL`. A book with outstanding loan items cannot be logically deleted and returns **409 `BOOK_HAS_OPEN_LOANS`**. ISBN uniqueness uses the existing partial index on active books, allowing the ISBN of a deleted book to be registered again.
- **Alternatives rejected:** physical deletion with cascading history loss; an additional `active` flag that does not record the time of deletion.
- **Consequences:** `loan_item.book_id` retains its `RESTRICT` FK. A direct deletion of a referenced book fails instead of orphaning loan history. The cover binary is the only business data physically deleted, following the reference-clearing order in [data-model.md §7](../data-model.md#7-privacy-and-retention).
- **Verification preserved:** catalog detail for a logically deleted book returns 404; an attempted physical deletion of a referenced book in a disposable test database is rejected by the FK. This describes future verification, not an action performed by this documentation task.

Logical deletion does not change the snapshots in existing loan items. Historical queries must not lose those items because an adapter accidentally applies the catalog's active-book filter to loan history. Reusing an ISBN creates a different book identity; old loan items remain associated with their original `book_id`.

## Audit findings and proposed resolutions

| Priority / type | Baseline evidence | Problem | Scoped resolution and compatibility review |
|---|---|---|---|
| P1 · Definition gap | [CA-03.5](../../spec-.net/spec.md#hu-03--gestionar-libros-admin), [contract calculated statuses](../../spec-.net/api-contract.md#36-préstamos) | “Open loans” may be mistaken for only the literal `open` status and exclude overdue items | Interpret outstanding as any item with `returned_at` unset, including overdue items. Keep `BOOK_HAS_OPEN_LOANS` and the original logical deletion operation; add no state or column |
| P1 · Implementation trap | [ADR-003 decision](../../spec-.net/adr/adr-003-baja-logica.md), [ADR-002](../../spec-.net/adr/adr-002-concurrencia-optimista.md) | Checking pending items and later deleting without concurrency protection can race with lending | Include logical deletion in the proposed book-version protocol and recheck eligibility after a conflict. The mechanism must preserve original behavior and requires evidence, not extra schema objects |
| P2 · Implementation trap | [Model FK policy](../../spec-.net/data-model.md#5-política-de-claves-foráneas) | The book FK is sometimes read as a complete guarantee that history cannot be deleted through SQL | Preserve `RESTRICT` for books and the original loan-to-item `CASCADE`; explain that application immutability does not prevent every privileged SQL operation. No stronger database policy is claimed |
| P2 · Implementation trap | [ADR-003 catalog filter](../../spec-.net/adr/adr-003-baja-logica.md), [ADR-004 snapshots](../../spec-.net/adr/adr-004-reporte-agregado-y-congelado.md) | Framework soft-delete filtering can accidentally hide historical results | Keep the active filter centralized for catalog operations while historical reads use the recorded loan items. Configure the adapter rather than change the business model |
| P2 · Definition gap | [Model retention order](../../spec-.net/data-model.md#7-privacidad-y-retención) | Binary replacement and failed cleanup are not fully specified | Preserve clearing/committing the reference before deletion; define replacement and retry behavior through the existing storage port before implementing it. Any added persistence or public operation would need separate compatibility review and is not included here |

No restoration route, physical book deletion route, additional timestamp or cleanup table is introduced.
