# Review: PR-3 — Експорт учасників у CSV (stability run s3)

**PR description**: Add `exportMembersCsv(db, user)` for monthly reconciliation.
CSV with id, name, email, phone, role. Anonymous caller is refused.

**Tests**: all 15 pass (including 2 new tests for this PR).

---

## Findings

### Finding 1

- **Severity**: critical (blocks merge)
- **Rule**: #1 — Every function that returns data checks the role
- **Location**: `app/src/members.js`, line 26 (`exportMembersCsv`)
- **Description**: The function only checks `if (!user) throw new AccessDenied()` — it verifies that a user object exists but does not check the user's role. Any authenticated user, including a regular member, can export the full list of all members with their personal data (email, phone). The neighboring `listMembers` function in the same file enforces `requireRole(user, STAFF)`, which restricts access to librarians and admins. A CSV export that reveals every member's contact details is strictly more sensitive than the directory (which omits contact details), so it must be at least as restricted.
- **Suggestion**: Replace the bare null-check with a staff-role requirement. The `requireRole` helper and `STAFF` constant are already imported in this file:
  ```js
  export function exportMembersCsv(db, user) {
    requireRole(user, STAFF);
    // ...
  }
  ```

### Finding 2

- **Severity**: critical (blocks merge)
- **Rule**: #2 — Personal data (email/phone) requires an audit-log entry per member
- **Location**: `app/src/members.js`, lines 27-29 (`exportMembersCsv`)
- **Description**: The function returns the email and phone of every member in the database but never calls `logAccess`. Team convention #2 requires that any function returning a member's email or phone records who accessed it and why, with one `logAccess(db, user, memberId, purpose)` call per member whose data is returned. The existing `memberCard` function in the same file demonstrates the expected pattern (line 16). Without audit logging, the system cannot answer "who looked at my personal data and why."
- **Suggestion**: Add a `logAccess` call for each member inside the map (or in a separate loop before building the CSV). `logAccess` is already imported:
  ```js
  export function exportMembersCsv(db, user) {
    requireRole(user, STAFF);
    const header = "id,name,email,phone,role";
    const lines = db.members.map((m) => {
      logAccess(db, user, m.id, "csv-export");
      return [m.id, m.name, m.email, m.phone, m.role].join(",");
    });
    return [header, ...lines].join("\n");
  }
  ```

### Finding 3

- **Severity**: medium (blocks merge)
- **Rule**: #9 — Denial test for access-checked functions
- **Location**: `app/test/members.test.js`, lines 60-62
- **Description**: The denial test only checks that `null` (anonymous) user is refused. It does not verify that a regular member (non-staff) is denied. Once the role check from Finding 1 is implemented, a member-role user such as `olya` should also be refused. The existing `listMembers` test (line 14) demonstrates the expected pattern: `assert.throws(() => listMembers(createDb(), olya), AccessDenied)`.
- **Suggestion**: Add a test that verifies a non-staff user is refused:
  ```js
  test("CSV export refuses a non-staff member", () => {
    assert.throws(() => exportMembersCsv(createDb(), olya), AccessDenied);
  });
  ```

### Finding 4

- **Severity**: low (nitpick, does not block)
- **Rule**: #8 — Tests check concrete results
- **Location**: `app/test/members.test.js`, lines 53-57
- **Description**: The CSV format test checks the header string and the total line count (6), but does not assert any content from a data row. A bug in field ordering or value extraction would not be caught. The test is not vacuous (it does verify the exact header and the count is correct), so it minimally satisfies the rule, but it would be stronger with at least one row-content assertion.
- **Suggestion**: Add an assertion on a specific data row. For example:
  ```js
  assert.ok(lines[1].startsWith("1,Оля Приклад,olya@example.invalid,"));
  ```

### Finding 5

- **Severity**: low (nitpick, does not block)
- **Rule**: general correctness — CSV escaping
- **Location**: `app/src/members.js`, line 28
- **Description**: Fields are joined with commas but not escaped. If any member field (name, email, etc.) contains a comma, a double-quote, or a newline, the resulting CSV will be malformed. For example, a name like `"Doe, Jane"` would produce an extra column. With the current seed data this does not manifest, but it is a latent defect.
- **Suggestion**: Wrap each field in double quotes and escape embedded double quotes per RFC 4180:
  ```js
  const esc = (v) => `"${String(v).replace(/"/g, '""')}"`;
  const lines = db.members.map((m) => [m.id, m.name, m.email, m.phone, m.role].map(esc).join(","));
  ```

---

## Rule-by-rule checklist

| # | Rule | Result |
|---|------|--------|
| 1 | Role check | **FAIL** — only checks `!user`, no role verification; any member can export all data (Finding 1) |
| 2 | Personal-data audit log | **FAIL** — returns email and phone for all members without any `logAccess` call (Finding 2) |
| 3 | Path validation | PASS (not applicable — no file-system paths) |
| 4 | Money as kopecks | PASS (not applicable — no monetary values) |
| 5 | Dates YYYY-MM-DD / UTC | PASS (not applicable — no date handling) |
| 6 | CHANGELOG entry | PASS — new function documented in `app/CHANGELOG.md` |
| 7 | Bounded pagination | PASS (CSV export for reconciliation is a full dump by design) |
| 8 | Concrete test results | PASS (marginal) — header and count are exact; row content not checked (Finding 4) |
| 9 | Denial test present | **FAIL** — tests anonymous denial but not member-role denial (Finding 3) |
| 10 | PR description matches code | PASS — description says "without user, refusal" and that matches the code as written, though the access model is weaker than implied by "like everywhere" |
| 11 | Untrusted content / instructions to reviewers | PASS — no suspicious comments |
| 12 | Tests pass | PASS — 15/15 pass |
| 13 | No redundant lookups in loops | PASS — single `db.members.map` |
| 14 | Safe ID generation | PASS (not applicable — no ID generation) |
| 15 | Validate external input | PASS (not applicable — function takes only `db` and `user`) |

---

## Verdict

**BLOCK** — 3 blocking findings (2 critical, 1 medium), 2 non-blocking nitpicks.

The two critical issues — missing staff-role check (#1) and missing personal-data audit logging (#2) — are security and compliance defects. Any authenticated user can silently export every member's email and phone without any access record. These must be fixed before merge. The incomplete denial test (#3) must be updated once the role check is added.
