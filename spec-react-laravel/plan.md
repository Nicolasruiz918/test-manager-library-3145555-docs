# Technical plan — Manager Library (React + Laravel)

How the product in [spec.md](spec.md) and the principles in [constitution.md](constitution.md) translate to the stack required by the [base README](../README.md). The schema remains in [data-model.md](data-model.md); the system structure is in [architecture.md](architecture.md).

This plan follows the sections and decision categories of the [.NET reference](../spec-.net/plan.md). It does not prescribe new product behavior, modify the six-table model or certify an existing implementation. Open points below distinguish definition gaps and implementation risks from confirmed failures. Documentation is English; the original Spanish user-message requirement remains in force.

## 1. Technical decisions

| # | Decision | React/Laravel translation | Reason and scope |
|---|---|---|---|
| D-01 | Language and runtime | PHP compatible with the selected Laravel release | Required backend language. The README does not fix a PHP or Laravel version; select and record a compatible combination before implementation. |
| D-02 | HTTP framework | Laravel as the HTTP adapter | Implements [api-contract.md](api-contract.md), with thin controllers and framework-independent use cases. |
| D-03 | Persistence | Eloquent/SQL adapters and Laravel migrations over PostgreSQL 16 | Schema ownership remains with application migrations under [ADR-001](adr/adr-001-propiedad-del-esquema.md). ORM defaults cannot redefine the physical model. |
| D-04 | Concurrency | Optimistic book version checking plus the inventory range constraint | Preserve [ADR-002](adr/adr-002-concurrencia-optimista.md) and RNF-2. Token representation, member coordination and return conflicts need the clarifications below. |
| D-05 | Time | UTC instants, standard PHP immutable date values and an application-owned `Clock` port | Enables time-based tests without waiting fourteen days. The domain receives an instant rather than importing the application port. |
| D-06 | Reporting | An aggregated query behind a read port | Preserve frozen report values and [ADR-004](adr/adr-004-reporte-agregado-y-congelado.md). |
| D-07 | Value objects | `Isbn`, `Quantity`, `DocumentNumber`, `DateRange`, stored within their owner's existing row | No identity or additional tables. Positive `Quantity` must not be used for available copies, which may equal zero. |
| D-08 | Covers | External binary storage: disk locally, S3 in production; only `cover_key` in the database | Preserve the original storage distinction and opaque-key model. |
| D-09 | Passwords | Hashing behind `PasswordHasher` | The .NET reference selects BCrypt factor 12; the Python reference selects Argon2id. Record the Laravel-compatible choice as a technology translation, not a new business requirement. |
| D-10 | Startup | The application provisions the initial admin from `ADMIN_USERNAME` and `ADMIN_PASSWORD` | The application produces the hash; `infra` starts an empty database rather than seeding a password hash. |
| D-11 | Authentication | Signed JWT, HS256 and eight-hour validity, secret from `JWT_SECRET` | Preserve the reference plan and HTTP contract. A different default Laravel authentication mechanism does not automatically satisfy JWT. |
| D-12 | Tests | PHPUnit-compatible tests, port fakes and a real database for integration | Translation of the reference test strategy; versions and any architecture-test tooling are implementation choices, not extra evaluation criteria. |

React is the frontend required by the README and T-22. Its release, build tooling and test tooling must be compatible and recorded when the `app` project is created; this document does not impose React, PHP, Laravel or build-tool versions absent from the README. Frontend integration uses the existing HTTP contract and role restrictions, without relocating the React application into the backend repository.

**Definition gaps requiring a recorded implementation decision:**

- **Version token:** ADR-002 requires a book version token, but the physical model has no corresponding user column. An adapter-only PostgreSQL `xmin` token is one possible solution without altering the model; it remains a proposal here. Its scope, comparison and lifecycle must be specified before relying on it. PostgreSQL transaction identifiers are not permanent version identifiers. [System columns](https://www.postgresql.org/docs/16/ddl-system-columns.html).
- **Member limit and returns:** per-book version checks do not protect a member's aggregate limit when requests use different books. Coordinating on the member, then rereading pending lines, is a proposed solution. Returns also need atomicity and revalidation so a duplicate cannot restock twice. Row locking is not intrinsically a lock on the entire library; the reference ADR's contrary explanation is technically incorrect. [PostgreSQL row locks](https://www.postgresql.org/docs/16/explicit-locking.html#LOCKING-ROWS).
- **Inventory and retries:** the range constraint does not establish that availability equals total minus unreturned lines. Any chosen retry strategy must preserve the complete loan/return operation rather than retrying an isolated update with stale state. The reference leaves some conflicts involving returns, edits and logical deletion undefined; resolve their mapping using an explicit contract clarification, not an undocumented new code.
- **Startup:** migrations must precede application use of the tables. The initial-admin procedure needs an explicit decision for repeated or simultaneous startup and for an existing username. Idempotent provisioning is a possible implementation approach; this document does not add a role-promotion operation or a new account policy.
- **Covers:** the original removal order is reference removal and commit before binary deletion. Replacement, storage failure, concurrent updates and failed cleanup require a consistent procedure. Storing under a fresh key and compensating an uncommitted upload is a possible approach. The local disk evidence does not prove S3 works; deployment configuration and production evidence remain to be specified without creating a new examination milestone.

**Laravel implementation risks, not observed code defects:**

- Eloquent conventionally uses plural table names, integer identities and timestamps. Configure persistence to match the existing singular tables, UUID identities and absence of `created_at`/`updated_at`. Framework migration metadata must be distinguished from the six business tables and its placement clarified; no metadata table is adopted as part of this model. [Eloquent conventions](https://laravel.com/framework/docs/13.x/eloquent).
- The contract requires UUID v4. Some Laravel UUID helpers/traits generate a different version by default; explicitly select and verify v4 generation in an adapter, without importing framework utilities into the domain. [Laravel UUID helpers](https://laravel.com/framework/docs/13.x/strings#method-str-uuid).
- Sanctum personal tokens or cookie sessions do not by themselves implement the specified signed JWT. Keep password hashing and JWT behind the existing ports and verify the contract. Do not introduce a new token table, refresh endpoint or logout contract as a framework convention. [Sanctum authentication modes](https://laravel.com/framework/docs/13.x/sanctum).
- Laravel's automatic string trimming and empty-string-to-null conversion can alter request semantics. Align normalization with the specified username/title rules, optional fields and query behavior, and preserve password bytes. Do not silently turn unspecified empty-value policies into new business rules. [Input normalization](https://laravel.com/framework/docs/13.x/requests#input-trimming-and-normalization).
- The cover operation is a real multipart `PUT`. Parser support depends on the selected PHP/framework combination; verify it instead of replacing the contract with POST and method spoofing. PHP introduced `request_parse_body()` in 8.4, and Symfony documents non-POST multipart support in 7.4; these are compatibility references, not newly mandated versions. [PHP parsing](https://www.php.net/request-parse-body), [Symfony request support](https://symfony.com/blog/new-in-symfony-7-4-request-class-improvements).
- BCrypt's PHP implementation truncates input beyond 72 bytes. If BCrypt is selected, resolve how the adapter and contract avoid silent truncation before implementation; the reference does not define a new password-length policy. Argon2id is already a technological alternative in the Python plan. [PHP password hashing](https://www.php.net/manual/en/function.password-hash.php).

## 2. Environment configuration

These are the five configuration entries from the reference plan. Secrets come from the environment, never committed values. Additional framework configuration, if necessary, must be documented in the implementation without being mistaken for new business fields or changes to this baseline contract.

| Variable | Required | Example / meaning |
|---|---|---|
| `DATABASE_URL` | Yes | PostgreSQL connection string supplied by the environment |
| `JWT_SECRET` | Yes | A long, random secret; never committed or logged |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` | Yes | Initial administrator credentials from the environment |
| `COVERS_DIR` | No | `./var/covers` for the local disk adapter |
| `CORS_ORIGIN` | Yes | The configured React origin, such as `http://localhost:5173` during local development |

The S3 production adapter needs deployment-specific storage configuration that the reference does not enumerate. That is an open configuration detail, not evidence that S3 has been implemented or an authorization to add storage data to the model.

RNF-3 requires protected credential transport as well as hashing and log hygiene. Specify where TLS terminates and verify the client-to-API connection; inspecting responses alone does not establish transport protection. A local frontend-origin example does not prove that the API transports credentials over HTTPS. A development exception must not be silently presented as compliance with RNF-3.

PHP and the web server can reject uploads before the controller runs. Configure and test the selected stack against the original 2 MB limit and T-15's 3 MB rejection, preserving the specified HTTP error format. The precise byte interpretation of “2 MB” and handling of pre-controller errors are clarifications to agree; this document does not introduce a new unit or error code. CORS preflight also needs transport-level handling without converting protected business operations into public endpoints.

## 3. Test strategy

| Level | What it tests | Against what |
|---|---|---|
| Domain | Limits, fourteen-day due dates, inventory range and state calculation | Plain PHP objects with explicit instants; no framework or database |
| Application | Lending, returns, reporting and the other specified use cases | Fakes of the application-owned ports |
| Integration | Repositories, migrations and report query | Real PostgreSQL |
| Contract | Existing routes, roles, response shapes, status codes and uniform error body | The running API |
| Architecture | Articles I, II and III | An automated dependency/composition check appropriate for PHP |
| Concurrency | Two simultaneous loans of the last copy | Independent operations against real PostgreSQL |

The table preserves the reference strategy. The audit identifies further scenarios needed to substantiate its existing rules: one member requesting different books concurrently, duplicate returns, separate lines returning the same book, failed multi-line loans, and inventory edits or logical deletion overlapping with loans. These explain the original invariants; they do not create new numbered tasks or change the evaluation's completion threshold.

Verify HU-05 and HU-08 explicitly when implementing their existing operations; their limited visibility in the original application-task breakdown is a coverage gap, not permission to omit members or loan queries. Review the existing constraint tasks and data-model verification queries against the actual schema when it becomes available. A document alone is not evidence that a constraint exists.

RNF-1 remains less than 300 ms with 5,000 books. The reference does not define a workload, concurrency, percentile or measurement environment. Record and resolve those details before claiming a reproducible result; do not silently replace the original threshold with a percentile or invent a new pass criterion. The smaller demonstration catalogue is not the specified performance dataset.

**Overdue demonstration gap:** T-21 requires three overdue lines after API-based seeding, while new loans are due fourteen days later and cannot be backdated through the contract. An immediate run against a new database and a real clock cannot satisfy both requirements. The existing `Clock` port offers a possible solution: coordinate an isolated historical seeding phase and a subsequent real-time phase. For example, three one-line loans to three members with a business clock fifteen days earlier, followed by twelve current loans distributed among the other seven members, can respect the original counts and member limit.

That is a **proposal to resolve the demonstration gap**, not an adopted environment variable, new endpoint or implemented procedure. Its isolation, restoration of real time, failure handling and separation from JWT time must be agreed and documented in the implementation. The `tool` seeder must still use only the API; it cannot bypass the contract through SQL. No existing command or passing demonstration is claimed here.

## 4. Construction order

Skeleton and health → initial migration and category seed → domain → use cases with fakes → adapters → API → contract and concurrency tests.

The original numbered work, dependencies, evidence and completion definition remain in [tasks.md](tasks.md). The audit notes identify where the original dependency graph or wording needs clarification; this plan does not silently rewrite that graph or add gates. The README's six forks, small incremental commits, delivery deadline and SDD evaluation focus remain authoritative.
