# API Contract — Manager Library

The HTTP contract is **language independent**: React + Laravel must respect it just as a Python or .NET implementation would.
Operation behavior is defined in [`spec.md`](spec.md); stored data is defined in [`data-model.md`](data-model.md).

Sections 1–4 preserve the reference contract in English. Documentation language does not change the product's required Spanish
messages or the fixed Spanish category names. Examples therefore retain those business values. The final audit appendix identifies
remaining gaps and concrete proposed resolutions; those proposals are **not additional acceptance criteria or silently adopted contract changes**.

## 1. Conventions

| Topic | Rule |
|---|---|
| Base | `/api/v1` |
| Format | JSON, UTF-8. Field names use `snake_case` |
| Identifiers | UUID v4 as text |
| Dates | ISO 8601 in UTC, with `Z` (`2026-10-14T14:15:00Z`) |
| Money | None. The system does not handle monetary amounts |
| Authentication | `Authorization: Bearer <token>` everywhere except `POST /auth/login` and `GET /health` |
| Pagination | `?page=1&size=20`. Maximum `size`: 50. Response: `{ "items": [...], "page": 1, "size": 20, "total": 134 }` |
| Language | Error messages in Spanish; fields and codes in English |

### Error body (the same shape for every error)

```json
{
  "code": "BOOK_NOT_AVAILABLE",
  "message": "No quedan ejemplares disponibles de «Cien años de soledad».",
  "details": [ { "field": "items[0].book_id", "issue": "sin ejemplares" } ]
}
```

`details` may be empty. It **never** includes stack traces, SQL, server paths, or the password.

### Status codes

| Status | When |
|---|---|
| 200 / 201 / 204 | Success / created / no content |
| 401 | Missing, invalid, or expired token; invalid credentials |
| 403 | Insufficient role |
| 404 | Does not exist, or has been soft deleted |
| 409 | State conflict: duplicate, no copies available, already returned, member with overdue items, book with open loans |
| 422 | Validation: invalid field, broken input rule, reversed date range |

## 2. Business error codes

| `code` | Status | Source |
|---|---|---|
| `INVALID_CREDENTIALS` | 401 | CA-01.2 |
| `FORBIDDEN_ROLE` | 403 | CA-01.3 |
| `NOT_FOUND` | 404 | Any resource |
| `USERNAME_TAKEN` | 409 | CA-02.2 |
| `ISBN_TAKEN` | 409 | CA-03.2 |
| `DOCUMENT_TAKEN` | 409 | CA-05.1 |
| `BOOK_HAS_OPEN_LOANS` | 409 | CA-03.5 |
| `BOOK_NOT_AVAILABLE` | 409 | CA-06.3 |
| `MEMBER_HAS_OVERDUE` | 409 | CA-06.5 |
| `ITEM_ALREADY_RETURNED` | 409 | CA-07.2 |
| `MEMBER_LIMIT_EXCEEDED` | 422 | CA-06.4 |
| `DUPLICATE_BOOK_IN_LOAN` | 422 | CA-06.2 |
| `VALIDATION_ERROR` | 422 | Any invalid field |
| `INVALID_DATE_RANGE` | 422 | CA-10.2 |

Acceptance-criterion identifiers retain the original `CA` prefix for traceability.

## 3. Endpoints

### 3.1 Authentication and health

| Method | Route | Role | Description |
|---|---|---|---|
| POST | `/auth/login` | public | Returns a token |
| GET | `/health` | public | `{ "status": "ok" }` |

`POST /auth/login` request:

```json
{ "username": "ana", "password": "********" }
```

200 response:

```json
{ "access_token": "eyJ...", "token_type": "Bearer", "expires_in": 28800, "role": "librarian" }
```

### 3.2 Users

| Method | Route | Role | Description |
|---|---|---|---|
| POST | `/users` | admin | Creates a librarian |

Request (`role` only accepts `"librarian"`):

```json
{ "username": "Ana ", "password": "********", "role": "librarian" }
```

201 response, without `password` or `password_hash`:

```json
{ "id": "aaaaaaaa-aaaa-4aaa-8aaa-aaaaaaaaaaa1", "username": "ana", "role": "librarian" }
```

### 3.3 Categories

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/categories` | authenticated | The five categories, ordered by name |

### 3.4 Books

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/books` | authenticated | Search: `q`, `category_id`, `page`, `size` |
| GET | `/books/{id}` | authenticated | Detail |
| POST | `/books` | admin | Creates a book |
| PUT | `/books/{id}` | admin | Replaces title, author, category, and total copies |
| DELETE | `/books/{id}` | admin | Soft deletion |
| PUT | `/books/{id}/cover` | admin | Uploads a cover (`multipart/form-data`, field `file`) |
| DELETE | `/books/{id}/cover` | admin | Removes the cover |
| GET | `/books/{id}/cover` | authenticated | Returns the image binary, or 404 |

The last row makes explicit the read operation already required in the reference contract's text below; it does not add an endpoint.

Book response:

```json
{
  "id": "bbbbbbbb-bbbb-4bbb-8bbb-bbbbbbbbbbb1",
  "title": "Cien años de soledad",
  "author": "Gabriel García Márquez",
  "isbn": "9780307474728",
  "category": { "id": "11111111-1111-4111-8111-111111111111", "name": "Novela" },
  "total_copies": 4,
  "available_copies": 3,
  "cover_key": null
}
```

`POST /books` request: `title`, `author`, `isbn`, `category_id`, `total_copies`. The server calculates available copies.
`PUT /books/{id}` replaces the four attributes listed in the table; ISBN is not listed as an editable attribute.
`cover_key` is an opaque key: display the cover using `GET /books/{id}/cover`, which returns the binary or 404.

### 3.5 Members

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/members` | admin, librarian | List and search: `q`, `page`, `size` |
| GET | `/members/{id}` | admin, librarian | Detail with `active_loans` (count of unreturned items) |
| POST | `/members` | admin, librarian | Creates: `full_name`, `document_number` |

The business limit concerns unreturned books across all of a member's loans. In that context, `active_loans` denotes the count of
unreturned **loan items**, including overdue items, not the number of loan headers; its existing field name is retained.

### 3.6 Loans

| Method | Route | Role | Description |
|---|---|---|---|
| POST | `/loans` | admin, librarian | Records a loan |
| GET | `/loans` | admin, librarian | List: `from`, `to`, `status`, `member_id`, `page`, `size` |
| GET | `/loans/{id}` | admin, librarian | Detail with items and status |
| POST | `/loans/{loan_id}/items/{item_id}/return` | admin, librarian | Returns one item |
| GET | `/loans/overdue` | admin, librarian | Overdue items |

`POST /loans` request: 1 to 3 items, without repeated books. This example uses one item in both the request and response,
correcting the reference example's mismatch without changing its shape or the permitted item count.

```json
{
  "member_id": "cccccccc-cccc-4ccc-8ccc-ccccccccccc1",
  "items": [ { "book_id": "bbbbbbbb-bbbb-4bbb-8bbb-bbbbbbbbbbb1" } ]
}
```

201 response:

```json
{
  "id": "dddddddd-dddd-4ddd-8ddd-ddddddddddd1",
  "member_id": "cccccccc-cccc-4ccc-8ccc-ccccccccccc1",
  "loaned_at": "2026-10-14T14:15:00Z",
  "status": "open",
  "items": [
    {
      "id": "eeeeeeee-eeee-4eee-8eee-eeeeeeeeeee1",
      "book_id": "bbbbbbbb-bbbb-4bbb-8bbb-bbbbbbbbbbb1",
      "book_title": "Cien años de soledad",
      "isbn": "9780307474728",
      "category_name": "Novela",
      "due_at": "2026-10-28T14:15:00Z",
      "returned_at": null
    }
  ]
}
```

`status` is calculated: `open` (there are unreturned items and none is overdue), `overdue` (at least one unreturned item is overdue),
or `closed` (all items have been returned).

`POST /loans/{loan_id}/items/{item_id}/return` → 200 with the item, with `returned_at` populated.

`GET /loans/overdue` returns `items` whose entries have `loan_id`, `item_id`, `book_title`, `member_name`, `due_at`, and `days_late`.
This operational list includes the member's name; **the report in §3.7 does not**.

### 3.7 Report

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/reports/most-borrowed` | admin | `from` and `to` required (dates, both inclusive) |

Response:

```json
{
  "from": "2026-10-01",
  "to": "2026-10-31",
  "rows": [
    {
      "book_id": "bbbbbbbb-bbbb-4bbb-8bbb-bbbbbbbbbbb1",
      "book_title": "Cien años de soledad",
      "category_name": "Novela",
      "times_borrowed": 12
    }
  ]
}
```

One row per frozen `book_id + book_title + category_name`. Order: `times_borrowed` descending, then title.
No members or librarians are included.

## 4. Security

| Aspect | Rule |
|---|---|
| Token | Signed JWT, 8 hours, with `sub` and `role`. No personal details in the payload beyond the required subject identifier |
| Passwords | Only the hash is stored; the hash never appears in a response |
| Authorization | Evaluated on every endpoint by role; a 403 does not reveal whether the resource exists |
| CORS | Only the frontend origin configured through the environment |
| Cover upload | Only `image/jpeg` and `image/png`, maximum 2 MB; validate the type from content, not the extension |

The reference technical plan selects HS256; see [`plan.md`](plan.md). Translating the implementation to Laravel does not by itself
replace JWT with session cookies or opaque tokens. Neither the password nor its hash belongs in logs. The token's subject is the
identifier needed for authentication; names, documents, and other personal details are not added to it.

## Audit findings and proposed resolutions

These notes distinguish clear requirements from missing definitions and implementation pitfalls. They are proposals to resolve
before independent frontend/backend implementation, **not new mandatory behavior**. They do not add entities, database columns,
business endpoints, business error codes, or an evaluation rubric. References below are to this adaptation and its unchanged
counterparts in `spec-.net/` and `spec-python/`.

### A. Incomplete successful responses

**Evidence:** §3 gives concrete shapes for login, users, books, loan creation, return, and report, but does not fully specify successful
statuses and bodies for every operation. §1 defines a general pagination envelope, while categories and overdue responses lack a
complete standalone schema. This is a definition gap, not evidence that any endpoint is missing from an implementation.

**Proposed resolution:** agree reusable response descriptions based solely on fields already present in the contract/model, then
record the missing status/body choices together. The following matrix is a proposal for that review; it does not replace the baseline above.

| Proposed response description | Existing fields and meaning |
|---|---|
| `Book` | The complete book example in §3.4, with nullable `cover_key` and nested category |
| `Member` | `id`, `full_name`, `document_number` from the existing member model |
| `MemberDetail` | `Member` plus the already specified `active_loans` count |
| `LoanItem` | `id`, `book_id`, `book_title`, `isbn`, `category_name`, `due_at`, nullable `returned_at`, as in §3.6 |
| `Loan` | `id`, `member_id`, `loaned_at`, calculated `status`, and `items`, as in §3.6 |
| Category entry | Existing `id` and `name` fields |
| Overdue entry | The six existing fields listed in §3.6 |

| Operation group | Proposed completion, pending agreement |
|---|---|
| Create book or member | 201 with `Book` or `Member`; user and loan creation already show 201 |
| Book/member/loan detail | 200 with `Book`, `MemberDetail`, or `Loan` |
| Update book or upload cover | 200 with the resulting `Book` |
| Delete book or cover | 204 without a response body; separately decide the result of deleting an already absent cover |
| List books, members, loans, overdue items | 200 with the §1 pagination envelope and the appropriate entry type |
| List categories | 200 with `{ "items": [...] }`, without pagination |
| Report | 200 with the existing `from`, `to`, `rows` shape; an empty range result has `rows: []` |

This proposal does not expose new columns or require changing the physical model. The references already require a loan's items
and calculated status; using the same `Loan` shape in list/detail avoids inventing separate resource fields.

### B. Member counts, validation, and missing error definitions

**Evidence:** CA-06.4 requires the current unreturned-book count in `MEMBER_LIMIT_EXCEEDED`, but §1–2 does not say where to put it.
The universal error-body requirement also covers technical failures for which the original 14-code catalogue has no assigned code.
UUIDs, required input fields, reference lookup failures, and invalid pagination are not fully mapped to errors.

**Proposed resolution:** express the current count in the existing Spanish `message` and, if machine-readable access is required,
agree a decimal-string representation in the existing `details[].issue` for `field: "member_id"`. Do not add an unapproved top-level
field. Retain `active_loans` as the pending-item count clarified in §3.5, including overdue items.

Agree whether malformed UUIDs use 422 `VALIDATION_ERROR` and well-formed but missing resources/references use 404 `NOT_FOUND`.
Also agree how to handle a return whose item belongs to a different loan. Record handling of absent/expired tokens, malformed JSON,
unsupported methods/media types, and unexpected failures before publishing technical error identifiers; this document does not invent
those identifiers or change the original status catalogue. Preserve the common error shape in Laravel's exception and validation adapters.

The original specifies ISBN length and digits, not checksum validation; do not silently add a checksum rule. Similarly, do not impose
an arbitrary new password minimum or unknown-field rejection policy and present it as a requirement from the reference.

### C. Temporal filters, pagination, and ordering

**Evidence:** §3.7 explicitly makes report dates inclusive; §3.6 does not give equivalent type/boundary rules for loan-list `from`/`to`.
CA-09.1 defines overdue as `due_at` earlier than now but does not define the rounding of `days_late`. §1 gives page/size examples and
maximum size without specifying rejection versus adjustment of invalid values. Several sort orders lack deterministic tie-breakers.

**Proposed resolution:** agree `YYYY-MM-DD` UTC filters on `loaned_at`; implement inclusive date ranges as
`[from at 00:00 UTC, the day after to at 00:00 UTC)`. Decide whether each loan-list bound is independently optional; keep both report
bounds required. A concrete candidate for `days_late` is `floor((now - due_at) / 86400 seconds)`, which yields 0 for an item overdue by
less than one day. Retain the already specified strict `due_at < now` comparison and use one reference instant per query.

Agree rejection with 422 rather than silent clamping for zero, negative, fractional, empty, or oversized pagination parameters;
also agree defaults, empty-page behavior, text-search case handling, and stable tie-breakers. These details are proposals to make tests
repeatable, not additional acceptance criteria silently imposed by this translation.

### D. Concurrency and return semantics

**Evidence:** CA-06.4 limits the member across all loans; ADR-002 describes a version token on each book. Different books can be
borrowed concurrently by the same member without a book-version conflict. CA-07.2 forbids a double return, while ADR-002's explanation
that `returned_at` is already filled does not specify the simultaneous-read case. Its loan conflict code does not define every return/edit conflict.

**Proposed resolution:** document coordination of member eligibility separately from book inventory; compare candidate transaction
isolation/locking strategies in [ADR-002](adr/adr-002-concurrencia-optimista.md). Specify how a complete operation is rolled back,
re-read, and revalidated after a version conflict. Reuse `ITEM_ALREADY_RETURNED` only when that same item is confirmed returned;
two different returned items of one book are not a duplicate return. Preserve the required last-copy result: one 201 and one 409
`BOOK_NOT_AVAILABLE`. Agree any remaining conflict-response policy before adding codes.

The book version is an internal persistence concern, not an automatic new field in requests or responses. Resolve its representation
with `data-model.md` and ADR-002 rather than assuming the framework supplies it. For soft deletion, clarify that “open loans” means
any unreturned item of the book, including items whose loan's calculated status is `overdue`.

### E. Covers, CORS, and Laravel transport behavior

**Evidence:** §3.4 already requires authenticated binary cover reads; §4 restricts CORS origins and uploads to JPEG/PNG up to 2 MB.
T-15 expects a 3 MB upload to return 422. The references do not settle the exact byte meaning of MB, all success bodies, or middleware behavior.

**Proposed resolution:** agree whether the existing 2 MB limit means 2,097,152 bytes; this is a candidate interpretation, not a new
silently binding unit. Configure PHP and the web server so the specified 3 MB verification reaches a consistent 422 error rather than
an HTML or differently shaped upstream rejection. Verify content-based image validation and the lifecycle of replacements and failed cleanup.

React can fetch the cover with the Bearer header and display a local Blob URL, releasing it after use. That preserves authentication
without putting the token in a query string or making the endpoint public. Handle browser `OPTIONS` preflight in the CORS layer
without requiring the actual request's Bearer token; allow only the configured origin and required methods/headers. This is transport
handling, not an additional public business endpoint. Enforce role authorization before resource binding whenever needed to preserve
the existing requirement that 403 must not reveal resource existence.

Verify the selected Laravel/PHP versions' handling of `multipart/form-data` on the already specified **PUT** cover route; do not assume
file parsing works identically to POST or silently change the route. Inspect `TrimStrings` and `ConvertEmptyStringsToNull` middleware:
automatic trimming/null conversion must not alter passwords, the documented username normalization, or nullable/nonempty cover-key
semantics. Limit or replace those transformations where they would change the contract, and verify with representative requests.
No framework/library version is mandated by these notes.

### F. Demonstration time and security

**Evidence:** T-21 asks the API-based demonstration tool to create overdue items, while CA-06.6 uses `loaned_at + 14 days`,
CA-06.7 prohibits editing loans, and the creation request has no date fields. The plan's `Clock` port enables controlled tests but
does not by itself specify a demonstration configuration. On a fresh database with real time, newly created loans are not immediately overdue.

**Proposed resolution:** agree an environment-restricted business-clock setup for demonstration/testing, or revise the demonstration
evidence. Describe it in [`plan.md`](plan.md) and the relevant notes in [`tasks.md`](tasks.md). Keep JWT issuance/expiry tied to real
time so a business-clock simulation does not change authentication validity. Do not add historical-date request fields, a public clock
control, or a new demonstration endpoint. Retain the original API-only tool requirement.

RNF-3 also requires passwords not to travel in clear text. Specify and verify TLS termination in deployment; checking hashes, logs,
and response bodies alone does not demonstrate encrypted transport. This closes evidence for the existing requirement rather than
introducing a new business feature.
