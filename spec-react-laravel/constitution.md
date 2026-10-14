# Constitution — Manager Library (React + Laravel)

This folder documents the same product as [spec-.net](../spec-.net/) and [spec-python](../spec-python/), adapted to the React and PHP/Laravel stack required by the [root README](../README.md). The objective remains demonstrated proficiency in Spec-Driven Development, not the amount of code produced.

## Scope and document status

- Preserve the original eight main Markdown files and four ADR files, their responsibilities, and the six project repositories.
- Preserve all actors, stories, acceptance criteria, business rules, exclusions, API operations, business error codes and the physical business model. Do not add entities, columns, indexes or constraints as an undocumented correction.
- Preserve T-01 through T-23, their baseline dependencies, evidence and definition of the core. Findings about that plan appear separately; they are not new evaluation tasks or requirements.
- Documentation preceded implementation. This repository contains specifications and evidence links only; application code, migrations and deployment remain in their designated sibling repositories. The owner subsequently authorized implementation and direct task commits to main, as recorded in [plan.md](plan.md).
- Write the new documentation in **English**, as explicitly instructed by the project owner. This is a documented exception to the reference's documentation-language rule. It does not translate fixed category data or change the original requirement for Spanish user-facing messages.

Each document separates its **preserved baseline** from **audit findings and proposed resolutions**. A proposed technical solution explains how an existing requirement could be met; it is not evidence of implementation or permission to change the product. A decision that would change a stated schema or contract requirement must not be adopted silently.

Read these files in the order given by the root README: constitution; [specification](spec.md); [plan](plan.md), [architecture](architecture.md) and [overview](arquitectura-panoramica.md); [data model](data-model.md) and [HTTP contract](api-contract.md); [tasks](tasks.md); and the four ADRs. The source versions and root README remain unchanged.

## Article I — The domain depends on nothing

The domain contains business entities, value objects and invariants in ordinary PHP. It must not depend on other project layers, Laravel, Eloquent or third-party libraries. PHP standard types can represent values and instants.

**Reason:** invariants must be testable without a database, web server or container, and replacing persistence must not change business rules.

**Verification:** architecture checks appropriate to PHP reject dependencies from domain to application, adapters or composition. These replace the reference's language-specific tooling while preserving the same rule.

## Article II — The core declares its ports; external components implement them

The application layer knows the domain. Repositories, clock, password hashing, cover storage and token generation are expressed through ports owned by the application. Framework and ORM types do not appear in port signatures.

**Verification:** application must not depend on adapters. Application tests use fakes without loading real adapters. The domain receives time as a value; the application-owned `Clock` does not become a dependency of the domain.

## Article III — A single composition point

One module binds application ports to adapters. Laravel's provider mechanism can host that composition; its location must be documented before implementation. Infrastructure must not be instantiated throughout controllers or business services.

**Verification:** an architecture check detects product adapter instantiation or binding outside the designated composition module. Framework bootstrap registration is distinguished from product port bindings.

## Article IV — SOLID is concrete

| Principle | Application | Verification |
|---|---|---|
| SRP | One application service represents one use case; lending has one responsibility | Separate services with different reasons to change |
| OCP | Changing cover storage from disk to S3 does not change domain or application | Replace the adapter through composition |
| LSP | A fake and a real adapter satisfy the same port expectations | The same behavioral test passes against both the fake and the real adapter |
| ISP | Small ports such as `Clock`, `PasswordHasher`, `TokenIssuer`, `CoverStorage` | No unused placeholder implementations |
| DIP | The consumer owns its port | Ports belong to application |

## Article V — Migrations own the schema

The `infra` repository starts an **empty database engine**. Application migrations own the DDL, indexes, constraints, extensions and fixed category seed already required by the reference. In this stack, the migration mechanism is Laravel's. See [ADR-001](adr/adr-001-propiedad-del-esquema.md).

**Verification:** infrastructure contains no business-table creation statements. The future migration result must match the preserved model. Describing this responsibility does not create or execute a migration during the documentation phase.

## Article VI — Database-enforceable invariants belong in the database

An invariant expressible through the engine's `CHECK`, `UNIQUE` or foreign keys belongs there. The reference explicitly records unapplied guarantees as pending in [data-model.md](data-model.md). Preserve those classifications rather than claiming they are already implemented.

**Verification:** the model's verification procedure is compared with the actual schema when an implementation exists. Missing classifications are documented as findings; this adaptation does not insert additional constraints into the baseline model.

## Article VII — Derived values are calculated

Loan status (`open`, `overdue`, `closed`) and overdue-day counts are calculated, not stored. No `status` or `days_late` column is added.

**Verification:** those columns are absent from the physical model and future persistence.

## Article VIII — Frozen values do not follow the catalog

A loan item stores the title, ISBN and category name from the lending instant. Reports use those stored snapshots rather than current catalog labels.

**Verification:** renaming a book after lending does not change the old report title. See [ADR-004](adr/adr-004-reporte-agregado-y-congelado.md).

## Article IX — No secret enters the repository or a log

Passwords, precomputed hashes and token signing keys come from the environment and are not committed or logged. The real `.env` is excluded; only a non-secret `.env.example` is versioned. Neither `password` nor `password_hash` is returned by the API.

**Verification:** secret checks and response/log inspection, as required by the reference. Transport protection remains an existing requirement of RNF-3, not something demonstrated by hashing alone.

## Article X — The engine wins

If the data document and the database disagree, the document is broken and must be corrected. This is the preserved reference wording; its ambiguous relationship with migration ownership is recorded in C-01 below, together with a proposed resolution. It is not an instruction to edit a database during this documentation task.

## Article XI — Language

Code, columns and routes remain English. User-facing messages remain Spanish, as specified by the unchanged product contract. **Documentation in this adaptation is English under the owner's explicit instruction.** Fixed category names remain exactly as specified in the seed; translating prose must not alter business data.

## Article XII — Nothing is complete without evidence

A task in [tasks.md](tasks.md) is complete when its evidence exists and can be executed, not merely when code has been written. The original state was pending. The separate executed-evidence register now links actual results from each repository; a proposal alone still proves no result.

The original definition of the core at T-17 and the distinction of T-18 through T-23 are retained. No new scoring system, deadline, mandatory screen or implementation milestone is introduced.

## Audit findings and proposed resolutions

The sources below refer to available documents, not an inspected application or database.

| ID / classification | Evidence and issue | Scope-preserving resolution |
|---|---|---|
| C-01 / Authority gap | [Source article X](../spec-.net/constitution.md#artículo-x--gana-el-motor), model introduction and ADR-001 leave unclear whether the engine describes observed state or dictates expected design | Treat engine inspection as evidence of what exists and requirements/migrations as the expected state. Correct the mistaken document or implementation after comparison; do not legitimize drift by changing the model blindly |
| C-02 / Explicit documentation-language exception | Source article XI requires Spanish documentation; the owner's current instruction requires English documentation | Record the explicit owner instruction as the exception for these files. Preserve Spanish product messages and category values; do not call the original two specifications internally inconsistent on language |
| C-03 / Enforcement classification gap | Source article VI requires enforceable invariants to be recorded, while the model omits the enforcement classification of some stated value rules | Record the missing classification and proposed enforcement for review in the model's audit notes. Do not add schema objects or claim constraints exist without evidence |
| C-04 / Evidence gap | Architecture principles are mandatory, but [source T-19](../spec-.net/tasks.md) follows the declared core | Confirm that mandatory principles apply from the start and clarify the evidence expected at the core. Keep the baseline task table unchanged; distinguish a suggested sequencing correction from the evaluator's task list |
| C-05 / Implementation trap | Source T-05 mentions a fake `Clock`, while article I prohibits domain dependencies on application | Pass an explicit instant into domain behavior. Test time-dependent behavior without importing the application port into domain |
| C-06 / Framework trap | Laravel defaults can introduce persistence dependencies into domain entities or framework types into ports | Keep the same dependency direction while adapting the framework boundary. Eloquent models belong to persistence adapters, not automatically to the business domain |

The original audit identifies risks in documentation; it does not establish that traps were deliberate or that code contains them. Subsequent implementation choices and evidence are recorded separately in the plan and task register.
