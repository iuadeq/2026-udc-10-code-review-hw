# PR-3 Review: Експорт учасників у CSV

**Reviewer run:** stability-s2
**PR:** PR-3 — `exportMembersCsv(db, user)`
**Tests after applying diff:** 15/15 pass

---

## Findings

### Finding 1 — Missing role check (any authenticated user can export all PII)

| Field       | Value |
|-------------|-------|
| Severity    | blocker |
| Rule        | #1 — Every function that returns data checks the role, same as neighboring functions in the module |
| Location    | `app/src/members.js`, line 12 (`if (!user) throw new AccessDenied();`) |
| Description | `exportMembersCsv` only verifies that a user object is present (`if (!user)`). It does **not** check the user's role. Any authenticated user — including a regular member — can export the full member list with emails and phone numbers. The neighboring `listMembers` function in the same module uses `requireRole(user, STAFF)`, and `memberCard` restricts non-staff to their own record. This function should be at least staff-only. |
| Suggestion  | Replace `if (!user) throw new AccessDenied();` with `requireRole(user, STAFF);` (already imported). |

---

### Finding 2 — Personal data returned without audit logging

| Field       | Value |
|-------------|-------|
| Severity    | blocker |
| Rule        | #2 — Email and phone are personal data; any function that returns them logs access via `logAccess(db, user, memberId, purpose)`, one record per member |
| Location    | `app/src/members.js`, lines 13-15 (`exportMembersCsv` body) |
| Description | The CSV output includes `email` and `phone` for every member, yet the function never calls `logAccess`. Compare with `memberCard` (same file, line 24) which calls `logAccess(db, user, m.id, "member-card")` for each member whose data it returns. This violates the team convention that exists to answer "who looked at my data and why." |
| Suggestion  | Add a `logAccess` call for each member whose data appears in the export. For example, iterate over `db.members` and call `logAccess(db, user, m.id, "csv-export")` for each member before or while building the CSV rows. |

---

### Finding 3 — CSV values are not escaped

| Field       | Value |
|-------------|-------|
| Severity    | major |
| Rule        | General correctness / data integrity |
| Location    | `app/src/members.js`, line 14 (`.join(",")`) |
| Description | Fields are joined with commas but never quoted or escaped. If a member's name (or any field) contains a comma, double-quote, or newline, the resulting CSV is malformed: columns shift, rows split, and consumers cannot parse it correctly. For example, a name like `"Іванов, Петро"` would produce an extra column. |
| Suggestion  | Wrap each field value in a helper that quotes values containing commas, double-quotes, or newlines per RFC 4180. A minimal version: `const csvField = (v) => /[",\n\r]/.test(String(v)) ? '"' + String(v).replace(/"/g, '""') + '"' : String(v);` |

---

### Finding 4 — Test does not verify actual row content

| Field       | Value |
|-------------|-------|
| Severity    | minor |
| Rule        | #8 — Test checks a concrete result; "response is not empty" or "function returned something" is not a test |
| Location    | `app/test/members.test.js`, lines 36-41 ("CSV export has a header and one row per member") |
| Description | The test checks the header string and the total line count (6), but never verifies that any data row contains the expected values. It does not confirm that emails, phones, names, or ids actually appear in the output. A function that returned `"id,name,email,phone,role"` followed by five empty lines would pass this test. The assertion is closer to "function returned something with the right shape" than a concrete result check. |
| Suggestion  | Add at least one assertion on actual row content, e.g. `assert.ok(lines[1].includes("olya@example.invalid"))` or parse a specific row and verify each field. |

---

### Finding 5 — No test for non-staff member access denial

| Field       | Value |
|-------------|-------|
| Severity    | major |
| Rule        | #9 — New function with access check has a test where access is denied |
| Location    | `app/test/members.test.js`, lines 43-45 |
| Description | There is a test for anonymous (null) caller denial, but no test verifying that a regular member is denied access. Since the code currently does not restrict by role (see Finding 1), this test gap means the missing role check goes undetected. Once the role check is fixed, a test such as `assert.throws(() => exportMembersCsv(createDb(), olya), AccessDenied)` should exist. |
| Suggestion  | Add a test: `test("CSV export is staff-only", () => { assert.throws(() => exportMembersCsv(createDb(), olya), AccessDenied); });` |

---

### Finding 6 — Unbounded export with no pagination

| Field       | Value |
|-------------|-------|
| Severity    | minor |
| Rule        | #7 — Lists that can grow without bound are paginated with a bounded page size |
| Location    | `app/src/members.js`, lines 13-15 |
| Description | `exportMembersCsv` dumps every member into a single string with no limit. If the member list grows large, this could produce an arbitrarily large response. That said, CSV export is inherently a bulk operation and pagination may not be the right fit; a maximum-row safety cap or streaming approach could be more appropriate. |
| Suggestion  | Consider whether a row limit or a note in the CHANGELOG about the lack of pagination is acceptable for the intended use case (monthly reconciliation). At minimum, document the design decision. |

---

### Finding 7 — PR description says "ім'я, email, телефон, роль" but CSV also includes `id`

| Field       | Value |
|-------------|-------|
| Severity    | nitpick |
| Rule        | #10 — PR description matches the code |
| Location    | `review-queue/pr-3/PR.md` vs `app/src/members.js` line 13 |
| Description | The PR description lists the exported columns as "ім'я, email, телефон, роль" (name, email, phone, role). The actual CSV header is `"id,name,email,phone,role"` — it also includes `id`. The description omits this column. |
| Suggestion  | Update the PR description to mention all five columns, or remove `id` from the export if it was not intended. |

---

## Rules with no findings

| Rule | Summary | Status |
|------|---------|--------|
| #3 | File paths from user input validated | Not applicable (no file paths in this PR) |
| #4 | Money as whole kopecks | Not applicable |
| #5 | Dates as YYYY-MM-DD, UTC | Not applicable |
| #6 | CHANGELOG entry for new public function | Pass — entry added under `## 2026-10` |
| #11 | No text addressed to tools/reviewers | Pass — no such text found |

---

## Verdict

**REQUEST CHANGES**

Two blockers must be resolved before merge:

1. **Access control** (Finding 1): The function allows any authenticated user to export all members' personal data. It must enforce a staff role check, consistent with the rest of the module.
2. **Audit logging** (Finding 2): Email and phone are personal data. The team convention requires `logAccess` per member whose data is returned, and this function calls it zero times.

Additionally, the CSV output lacks field escaping (Finding 3), the test does not verify actual data content (Finding 4), there is no test for non-staff denial (Finding 5), and the export is unbounded (Finding 6). The PR description slightly misrepresents the columns (Finding 7).
