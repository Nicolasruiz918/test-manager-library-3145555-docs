# Tasks — Manager Library (React + Laravel)

Each task is complete when its **evidence** exists and can be executed (constitution article XII). Every task initially remains **pending**.

The table below preserves the original T-01 through T-23, their dependencies, scope and evidence from [the reference](../spec-.net/tasks.md). Only language-specific implementation and test-tool terminology is translated to PHP/Laravel. Identified defects and proposed corrections are separated below: they do not silently replace the evaluator's task list.

| # | Task | Depends on | Evidence |
|---|---|---|---|
| T-01 | Project skeleton and `GET /health` | — | `GET /health` returns `{"status":"ok"}`; the container starts with one command |
| T-02 | Initial migration: six tables, foreign keys, unique indexes and the five seeded categories | T-01 | The checks in `data-model.md` §10 match the document; no column has a `DEFAULT` |
| T-03 | Domain: `Isbn`, `Quantity`, `DateRange` value objects | T-01 | Unit tests: 13-digit ISBN, quantity > 0, end ≥ start |
| T-04 | Domain: `Book` (`Withdraw`, `Restock`, soft deletion) | T-03 | Tests: availability never < 0 and never > total |
| T-05 | Domain: `Loan` (`AddItem`, `ReturnItem`, calculated status, 14 days, maximum 3) | T-04 | Tests with a fake `Clock`: `open`, `overdue`, `closed` |
| T-06 | Domain: `Member` and `User` (username normalization, valid role) | T-03 | `"Ana "` becomes `ana`; requesting role `admin` through the API is rejected |
| T-07 | Authentication ports and use cases (`LoginService`, `CreateLibrarianService`) with fakes | T-06 | HU-01 and HU-02 pass against fakes |
| T-08 | Book ports and use cases (create, search, edit, soft delete, cover) with fakes | T-04 | HU-03 and HU-04 pass against fakes |
| T-09 | Lending use case (`LendBooksService`) with fakes | T-05, T-08 | Complete HU-06: limits, overdue items and unavailable copies |
| T-10 | Return and overdue-items use cases | T-09 | HU-07 and HU-09; returning twice produces `ITEM_ALREADY_RETURNED` |
| T-11 | Most-borrowed report use case and its read port | T-09 | HU-10; the old title remains after renaming (ADR-004) |
| T-12 | Persistence adapter: book, member and user repositories | T-02, T-08 | Integration tests against real PostgreSQL |
| T-13 | Persistence adapter: loan repository and version token | T-12, T-09 | Two attempts to lend the last copy: one 201, the other 409 (RNF-2) |
| T-14 | Security adapter: password hashing and JWT | T-07 | The password never appears in logs or responses (RNF-3) |
| T-15 | Cover storage adapter (disk) | T-08 | A 2 MB PNG uploads successfully; 3 MB produces 422; no cover means `cover_key` is null |
| T-16 | HTTP adapter: [contract routes](api-contract.md), uniform error body and role authorization | T-10, T-11, T-14, T-15 | Contract test covers all routes and all business codes in §2 |
| T-17 | Composition and startup (initial admin from environment) | T-16 | System starts with an empty database and admin can log in |
| T-18 | Enforce the original domain-only invariants in the database: `ck_book_total_positive`, `ck_user_role` | T-12 | The source's negative database-write test rejects values that violate those constraints; `data-model.md` §4 is updated |
| T-19 | Architecture tests for articles I, II and III, using PHP-compatible tooling | T-17 | Architecture checks pass and fail when a forbidden dependency is introduced |
| T-20 | `infra`: `docker compose up` starts the empty engine, API and frontend | T-17 | One command; the database has no application tables until migrations run |
| T-21 | `tool`: demonstration seeder through the API (at least 40 books, 10 members, 15 loans and 3 overdue items) | T-17 | After execution, `/loans/overdue` returns 3 items and the report has rows |
| T-22 | `app`: login, catalog, members, lending, returns, overdue items and report | T-16 | Manual end-to-end walkthrough at the presentation |
| T-23 | `page`: static public presentation site | — | Opens without the API; works on mobile and desktop |

T-18 describes an existing future implementation criterion from the reference; this documentation task does not create that test, issue a database write, add a constraint or mark it completed. T-19 changes the language-specific test tool, not the architecture principle or required outcome.

## Definition of done for the technical test

Reaching T-17 with evidence means the **core** is done. T-18 through T-23 distinguish a good submission from a more complete one, as stated in the reference. SDD proficiency is assessed, not task count: one evidenced task is worth more than several half-finished tasks.

The six forks, publication deadline and small commits remain those of the [root README](../README.md). This adaptation does not add a new core gate, score, task identifier, dependency or mandatory framework version. React preparation is part of its existing application work, not a new T-01 condition.

## Audit findings and proposed resolutions — separate from the baseline

| ID / priority / classification | Original evidence and issue | Proposed resolution without changing product scope |
|---|---|---|
| T-A01 / High / Dependency gap | T-17 → T-16 does not transitively include T-02, T-12 or T-13, although real startup and lending need persistence | Propose recording persistence/concurrency evidence as prerequisites of the core and correcting those dependency edges when the plan is reconciled. The table above preserves the original graph; no claim is made that it already guarantees those prerequisites |
| T-A02 / Medium / Traceability gap | HU-05 and HU-08 are required, but no application task explicitly owns all their use cases | Map member and loan-query behavior to existing work and its evidence, rather than dropping the stories or creating new product scope. Resolve the task ownership explicitly before implementation; do not add T-24 or redefine the core |
| T-A03 / Medium / Evidence-layer mismatch | T-06 is a domain task but its evidence mentions an API rejection; T-13 similarly mentions HTTP statuses before T-16 | Keep the required outcomes. Distinguish domain/application behavior from its later HTTP verification; record both under the existing tasks without making the domain depend on HTTP |
| T-A04 / Medium / Architecture evidence gap | Principles I–III are mandatory, but their named verification task T-19 follows core completion | Clarify which architectural evidence accompanies core work and which is consolidated in T-19. The principle remains mandatory; the distinction is not permission to break architecture until later |
| T-A05 / High / Conditional demonstration contradiction | T-21 requires overdue items immediately, while loans created through the public API expire 14 days later | Propose an isolated configuration of the existing business `Clock`: create three one-item loans at a historical instant, restore real time, then create the remaining loans for eligible members. Keep every business write behind the unchanged API, keep JWT time real and leave the database untouched by the tool. See the plan's audit notes |
| T-A06 / Medium / Implementation trap | T-03 defines positive `Quantity`, but available copies may legitimately be zero | Use that value object only where positivity is required; represent zero availability without rejecting a valid book. No model change is required |
| T-A07 / Medium / Evidence coverage gap | The last-copy test in T-13 does not cover the member limit or duplicate returns | Add candidate scenarios to the evidence of the already required invariants, not as new tasks: different books for the same member; simultaneous returns of the same item; distinct items of one book; complete rollback of a failed multi-item loan |
| T-A08 / Medium / HTTP and storage trap | T-15 expects 422 for 3 MB, but PHP/proxy limits can reject before the adapter; the contract uses actual PUT multipart | Verify the chosen runtime/parser and upstream limits against the existing route and response. Do not substitute another HTTP method or claim a changed upload size is an original criterion |
| T-A09 / Medium / Measurement gap | RNF-1 requires 5,000 books; T-21's minimum is only 40 | Use the existing tool for the larger measurement dataset; record workload/environment/statistic for review. Preserve both original thresholds and distinguish demonstration from performance evidence |
| T-A10 / Medium / Incomplete deployment evidence | T-17 demonstrates admin login, T-20 a full startup; transport and restart coordination are underspecified | Propose a sequence of migrations, admin provisioning and readiness, with repeat-start behavior and TLS evidence for RNF-3. Do not add infrastructure-owned DDL or change the schema |

## Candidate verification scenarios for existing requirements

These are proposed ways to verify the baseline, not executed results or additions to its task table.

| Existing requirement | Scenario and expected invariant |
|---|---|
| RNF-2 / CA-06.3 | Two eligible members request the final copy: one succeeds and one receives 409; no extra item is recorded |
| CA-06.4 | A member with two non-overdue pending items requests two different active available books concurrently: no confirmed result leaves four pending items |
| CA-07.2 | Two requests return the same item: one inventory increment; the repeated return uses the existing 409 business error |
| CA-07.1 / RN-1 | Two distinct items of the same book are returned: each confirmed return restores one copy, without losing an update |
| CA-06.3 | One book in a multi-item request is unavailable: no header, item or inventory decrement from that failed loan remains |
| CA-03.3 / CA-03.5 | Total editing or soft deletion competes with lending/returning: every confirmed state still meets the original invariants |
| CA-06.6 / CA-09.1 | Evaluate at exactly `due_at`: the item is not overdue until the instant is strictly later |
| CA-10.4 / ADR-004 | Rename between loans and query history: the original frozen values remain and the grouping follows the existing contract |

## Documentation-first review

Before implementation, reconcile the proposed resolutions against the root README, both reference versions, the unchanged business schema and the original HTTP contract. A resolution is acceptable only if it explains how to satisfy an existing requirement without narrowing or expanding the product.

Record any remaining decision explicitly instead of presenting an assumption as a confirmed defect or an approved requirement. This file supplies no application code, migration, seed SQL or database-write instruction. All implementation evidence remains pending.

## Executed implementation evidence

The baseline table above remains unchanged. Completion is tracked separately and only when the corresponding execution exists.

| Task | Evidence | Status |
|---|---|---|
| T-01 | [API commit 92d121d](https://github.com/Nicolasruiz918/test-manager-library-3145555-api/commit/92d121d5c08283b163878e5fd885ac8868becff2), including `evidence/T-01.md`: healthy container, two real HTTP 200 responses, three passing Laravel tests | Verified and published on `main` |
| T-02–T-23 | No completion claimed in this record | Pending |
