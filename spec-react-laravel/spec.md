# Specification — Manager Library (React + Laravel)

**Implementation status (2026-10-14):** the baseline and original audit findings below are preserved. The implementation records at the end of [plan.md](plan.md) adopt the relevant resolutions; [tasks.md](tasks.md#executed-implementation-evidence) links executed results. Historical wording such as proposed or pending in the audit is not the current completion register.

**What is being built and why.** A system for a small library to manage its book catalog, members and loans, and see which books are borrowed most often.

This document preserves the product and requirement identifiers from [the .NET reference](../spec-.net/spec.md) and [the Python reference](../spec-python/spec.md). React and PHP/Laravel are required by the [root README](../README.md). No new business feature is introduced. It follows [constitution.md](constitution.md); the physical model belongs in [data-model.md](data-model.md), and HTTP representations belong in [api-contract.md](api-contract.md).

The baseline below is translated into English. Audit notes at the end explain ambiguities and proposed resolutions separately from the original acceptance criteria.

## 1. Actors

| Actor | Responsibilities | Authentication |
|---|---|---|
| Administrator (`admin`) | Maintains the book catalog and members; creates librarians; sees all reports | Username and password → token |
| Librarian (`librarian`) | Records loans and returns; consults catalog, members and overdue items | Username and password → token |
| Member | Person who borrows books; does not access the system and is a business record, not a user | None |

Deployment provisions the `admin` role from environment variables. No runtime operation grants that role. The HTTP contract determines the roles allowed for each operation, including lending and returns.

## 2. Scope

**Included:** catalog with optional covers, five fixed categories, members, loans with multiple items, returns per item, overdue books, most-borrowed report, and login with roles.

**Excluded, without reopening scope:** fines and payments, reservations, renewals, email/SMS, bulk import, inter-branch loans, and additional book attributes such as publisher, year or language.

The six repositories keep their original responsibilities. The operational React application is `app`; `page` is the independent static public presentation. `tool` is the existing API-based demonstration utility, not a new product feature for importing data.

## 3. User stories and acceptance criteria

### HU-01 · Log in

As an internal user, I want to authenticate and operate according to my role.

- **CA-01.1:** valid credentials return a token valid for eight hours.
- **CA-01.2:** invalid credentials return 401 with a message that does not reveal whether the username or password was incorrect.
- **CA-01.3:** every endpoint except login and `/health` requires a token. No token: 401. Insufficient role: 403.

### HU-02 · Create librarians (admin)

- **CA-02.1:** the administrator creates a user with username, password and role `librarian`.
- **CA-02.2:** usernames are trimmed, stored in lowercase and unique. Duplicate: 409.
- **CA-02.3:** requesting role `admin` through the API returns 422. The password is never returned or logged.

### HU-03 · Manage books (admin)

- **CA-03.1:** create a book with title, author, ISBN, category and total copies. Available copies start equal to total copies.
- **CA-03.2:** ISBN contains 13 digits and is unique among active books. Duplicate: 409.
- **CA-03.3:** total copies must be at least one. Changing the total adjusts availability by the same difference and must never leave availability below zero; otherwise return 422.
- **CA-03.4:** upload an optional JPG/PNG cover, maximum 2 MB, and store an opaque key. Without a cover, the field is `null`, never an empty string.
- **CA-03.5:** deletion is a soft deletion: the book disappears from the catalog but remains in past loans. A book with open loans cannot be withdrawn; return 409.

### HU-04 · Search books

- **CA-04.1:** partial text search on title or author, category filter, active books only, title ordering and pagination using `page` and `size`, with maximum size 50.
- **CA-04.2:** the response includes the total number of matching results.
- **CA-04.3:** requesting a nonexistent or soft-deleted book by ID returns 404.

### HU-05 · Manage members

- **CA-05.1:** administrators and librarians create members with full name and document number. The document number is unique; duplicate: 409.
- **CA-05.2:** members are listed and searched by name or document number, with pagination.
- **CA-05.3:** members are not deleted.

### HU-06 · Record a loan

- **CA-06.1:** a loan belongs to one member, is recorded by the authenticated user and contains one to three items.
- **CA-06.2:** the same book cannot appear twice in a loan; return 422.
- **CA-06.3:** each item subtracts one available copy in the same operation. If a book has no available copies, return 409 and record nothing.
- **CA-06.4:** a member cannot have more than three unreturned books in total. Exceeding the limit returns 422 with the current count.
- **CA-06.5:** a member with any overdue book cannot borrow another; return 409.
- **CA-06.6:** the due instant is `loaned_at + 14 days`. The item freezes the title, ISBN and category name at the lending instant.
- **CA-06.7:** a recorded loan is not edited or deleted.

### HU-07 · Record a return

- **CA-07.1:** return one loan item at a time. Set `returned_at` and add one to available copies.
- **CA-07.2:** returning an already returned item produces 409. Availability is never incremented twice for that item.
- **CA-07.3:** availability never exceeds total copies; the database guarantees the range.
- **CA-07.4:** once every item is returned, the loan is shown as closed.

### HU-08 · View and list loans

- **CA-08.1:** loan detail includes its items and calculated status (`open`, `overdue`, `closed`).
- **CA-08.2:** list by date range, status and member, ordered by descending date and paginated.
- **CA-08.3:** status is calculated, not stored.

### HU-09 · Overdue books

- **CA-09.1:** list unreturned items whose `due_at` is earlier than now, including book, member and days late.
- **CA-09.2:** order by greatest delay first. Only administrators and librarians have access.

### HU-10 · Most-borrowed books report

- **CA-10.1:** for a date range, show one row per book and frozen category name, with the number of times borrowed, ordered from greatest to smallest count.
- **CA-10.2:** an end date before the start returns 422. A range without loans returns an empty list, not an error.
- **CA-10.3:** the report is not broken down by librarian or member.
- **CA-10.4:** a report for a closed period does not change when a book or its category is subsequently renamed.
- **CA-10.5:** only administrators have access.

## 4. Business rules

| ID | Preserved rule |
|---|---|
| RN-1 | Available copies always stay between zero and the total |
| RN-2 | Loan duration is 14 days; maximum three unreturned books per member |
| RN-3 | A member with overdue items cannot borrow more until returning them |
| RN-4 | Frozen title, ISBN and category do not follow the live catalog |
| RN-5 | Loans and returns are not deleted; books only use soft deletion |
| RN-6 | Dates are UTC instants; user-facing messages are Spanish; code and columns are English |

English documentation is the owner's explicit instruction for this adaptation; it does not change RN-6's product-message requirement.

## 5. Non-functional requirements

| ID | Preserved requirement | Original evidence expectation |
|---|---|---|
| RNF-1 | Book search responds in less than 300 ms with 5,000 books | Load test using the `tool` seeder |
| RNF-2 | Two librarians simultaneously lend the last copy: one succeeds and the other receives 409 | Concurrent test |
| RNF-3 | Passwords never travel or persist in clear text and never appear in logs | Review logs and responses |
| RNF-4 | Every HTTP error uses the same body, defined by the contract | Contract test |
| RNF-5 | The system starts with one command and the database initially starts empty | `infra` README |

No benchmark percentile, extra deployment target or new pass threshold is added to these requirements. The missing measurement details are identified below for clarification.

## Audit findings and proposed resolutions

| ID / priority / classification | Evidence and issue | Proposed resolution within the existing scope |
|---|---|---|
| S-01 / High / Conditional contradiction | CA-06.6–6.7, [original T-21](../spec-.net/tasks.md) and the original loan request provide no way to produce overdue items immediately on a fresh database with real time | Use the already specified `Clock` through an isolated demonstration/test configuration, with historical creation followed by normal time, if that environment procedure is accepted. Keep 14 days, immutable loans, API-only seeding and the unchanged request body. Never backdate by SQL or add a loan-date input field |
| S-02 / High / Concurrency gap | CA-06.4 is global across a member's loans; the original book version token does not coordinate different books | Coordinate member eligibility across the entire lending transaction, for example by protecting that member's row and rereading pending items. This implements the existing limit without a new entity or stored counter; see ADR-002's proposal |
| S-03 / High / Return concurrency gap | CA-07.2 requires one increment even when two requests initially observe an unreturned item | Apply an atomic pending-to-returned transition with the inventory update and revalidate after conflict. Keep the existing duplicate-return code; distinguish returns of different items of the same book |
| S-04 / High / Implementation trap | CA-03.1, CA-03.3, CA-06.3 and CA-07.1 imply more than a valid numerical range | Check the derived property `available = total - unreturned items of that book`. This is an explanation of existing operations, not a new stored field, table or database constraint |
| S-05 / Medium / Wording ambiguity | CA-03.5 says open loans, while the contract distinguishes literal `open` from `overdue` | Clarify that any unreturned item of the book blocks soft deletion, including overdue items; do not filter only on the literal status `open` |
| S-06 / Medium / Missing precision | CA-09.1 does not specify rounding of days late | Recommend complete elapsed UTC days while preserving strict `due_at < now`; record the selected rounding before contract tests. This may display zero whole days for an item that became overdue seconds ago |
| S-07 / Medium / Measurement gap | RNF-1 supplies a time limit and dataset size but no workload or environment | Record workload, concurrency, environment and reported statistic before implementation. Preserve the 300 ms/5,000-book requirement; do not substitute the 40-book demonstration or assert a new evaluator threshold |
| S-08 / Medium / Evidence gap | RNF-3 prohibits clear-text transport, but checking responses/logs does not establish TLS | Identify where HTTPS terminates and include transport evidence alongside hashing/log inspection; do not silently permit clear-text credentials as a local-environment exception |
| S-09 / Low / Already resolved by available sources | CA-10.1 is less precise than the original contract and ADR-004 | Use the already specified grouping by frozen `book_id`, `book_title`, `category_name`. A title change can create separate rows for the same book; this is not a new report feature |
| S-10 / Medium / Implementation trap | CA-06.4 counts unreturned books, while HTTP uses the potentially misleading name `active_loans` | Preserve that field name and clarify that it counts pending items across all loans, including overdue ones, not loan headers |
| S-11 / Low / Scope trap | A 13-digit ISBN is required; no checksum rule is specified | Validate the required format and active-book uniqueness only; do not add a stricter ISBN acceptance condition |

The proposed checks are evidence candidates for existing requirements, not new user stories or additional T-numbered tasks. None has been executed in this documentation repository.
