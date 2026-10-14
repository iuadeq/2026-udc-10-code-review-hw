# PR-3 Review: Експорт учасників у CSV

**PR:** PR-3 — `exportMembersCsv(db, user)`
**Reviewer run:** stability-s1
**Tests after applying diff:** 15/15 pass

---

## Findings

### Finding 1 — Missing role check (any authenticated user can export all members)

| Field | Value |
|---|---|
| Severity | **Blocker** |
| Rule | #1 — Кожна функція, що віддає дані, перевіряє роль |
| Location | `app/src/members.js`, new function `exportMembersCsv`, line `if (!user) throw new AccessDenied()` |

**Description:**
The function only checks that `user` is not null/undefined but does not verify the user's role. Any authenticated user (including ordinary members) can export the full member list with personal contact details. Compare with `listMembers` in the same file, which uses `requireRole(user, STAFF)` to restrict access to librarians and admins. An export that includes email and phone for every member should be at least as restrictive.

**Suggestion:**
Replace `if (!user) throw new AccessDenied();` with `requireRole(user, STAFF);` (or a more restrictive role set if only admins should export).

---

### Finding 2 — Personal data returned without audit log

| Field | Value |
|---|---|
| Severity | **Blocker** |
| Rule | #2 — Email і телефон учасника — персональні дані; доступ записується у журнал |
| Location | `app/src/members.js`, function `exportMembersCsv` |

**Description:**
The function returns email and phone for every member but never calls `logAccess(db, user, memberId, purpose)`. Rule 2 requires one audit-log entry per member whose personal data is returned. Compare with `memberCard` in the same file, which calls `logAccess(db, user, m.id, "member-card")` for every record it returns.

**Suggestion:**
Add a `logAccess` call for each member included in the export. For example, insert inside the map or add a separate loop:
```js
db.members.forEach((m) => logAccess(db, user, m.id, "csv-export"));
```

---

### Finding 3 — Unbounded list without pagination

| Field | Value |
|---|---|
| Severity | **Major** |
| Rule | #7 — Списки, що можуть рости без меж, — сторінкові |
| Location | `app/src/members.js`, function `exportMembersCsv`, line `db.members.map(...)` |

**Description:**
The function exports all members in a single string with no pagination or size limit. The member list can grow without bound. Rule 7 requires unbounded lists to be paginated with a limited page size.

**Suggestion:**
Either add pagination parameters (`offset`, `limit`) to the function signature, or document a hard cap. For a CSV export intended for monthly reconciliation, a reasonable approach might be to accept an optional `{ offset, limit }` parameter or to stream the output. At minimum, enforce a maximum row count.

---

### Finding 4 — CSV values not escaped (commas, quotes, newlines in data)

| Field | Value |
|---|---|
| Severity | **Major** |
| Rule | General correctness / robustness |
| Location | `app/src/members.js`, function `exportMembersCsv`, line `[m.id, m.name, m.email, m.phone, m.role].join(",")` |

**Description:**
Member names (or any field) may contain commas, double quotes, or newline characters. The current implementation joins values with `,` without any escaping or quoting. A name like `"Прізвище, Ім'я"` would break the CSV structure, producing a corrupt file that cannot be parsed correctly.

**Suggestion:**
Wrap each field in RFC 4180-compliant quoting: if a value contains a comma, double quote, or newline, enclose it in double quotes and escape internal double quotes by doubling them. For example:
```js
function csvField(v) {
  const s = String(v);
  return /[",\n\r]/.test(s) ? '"' + s.replace(/"/g, '""') + '"' : s;
}
```

---

### Finding 5 — PR description does not match output fields

| Field | Value |
|---|---|
| Severity | **Minor** |
| Rule | #10 — Опис PR відповідає коду |
| Location | `review-queue/pr-3/PR.md` vs. `app/src/members.js` |

**Description:**
The PR description says the export contains "ім'я, email, телефон, роль" (name, email, phone, role — four fields). The actual CSV header is `id,name,email,phone,role` — five fields, including `id`. The description omits `id`.

**Suggestion:**
Either update the PR description to list all five fields, or remove `id` from the CSV if it was not intended.

---

### Finding 6 — Test does not verify actual row content

| Field | Value |
|---|---|
| Severity | **Minor** |
| Rule | #8 — Тест перевіряє конкретний результат |
| Location | `app/test/members.test.js`, test `"CSV export has a header and one row per member"` |

**Description:**
The test checks that the header is correct and the total line count is 6, but does not verify the content of any data row. It does not assert that a specific member's data appears correctly in a row. According to rule 8, a test should verify a concrete result, not just a structural property like line count.

**Suggestion:**
Add at least one assertion on a specific row, e.g.:
```js
assert.ok(lines[1].includes("olya@example.invalid"));
```
or parse and check a full row.

---

## Rules with no findings

| Rule | Status |
|---|---|
| #3 — File path validation | Not applicable (no file paths in this PR) |
| #4 — Money in kopecks | Not applicable (no monetary values) |
| #5 — Dates as YYYY-MM-DD | Not applicable (no dates) |
| #6 — CHANGELOG entry | Pass — entry added for `exportMembersCsv` |
| #9 — Access-denied test | Pass — test `"CSV export refuses an anonymous caller"` present |
| #11 — No text addressed to tools/reviewers | Pass — no such text in this PR |

---

## Verdict

**Request changes.**

Two blockers must be resolved before merging:
1. The function must enforce a proper role check (rule 1) -- currently any authenticated user can export all personal data.
2. The function must call `logAccess` for each member whose email/phone is returned (rule 2) -- this is a data-protection requirement.

Additionally, the unbounded output (rule 7) and missing CSV escaping are significant issues that should be addressed.
