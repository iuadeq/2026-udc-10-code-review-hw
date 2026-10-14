# Review: PR-1

## Findings

### Finding 1
- **Severity**: critical
- **Rule**: #6 — Обмежені списки (bounded lists)
- **Location**: app/src/loans.js:31 (new function `listLoansPage`)
- **Description**: Off-by-one error in pagination. The formula `const start = page * pageSize` uses zero-based page indexing, but the default parameter is `page = 1`, which is 1-based. When a caller asks for page 1 (the default and the natural "first page"), `start = 1 * 20 = 20` — the first 20 items are skipped entirely. Verified empirically: with 7 loans, `listLoansPage(db, iryna, { page: 1, pageSize: 3 })` returns loan ids `[4, 5, 6]` instead of the expected `[1, 2, 3]`. Page 0 returns the actual first page, which contradicts the 1-based default.
- **Suggestion**: Change to `const start = (page - 1) * pageSize;` so that page 1 maps to offset 0.

### Finding 2
- **Severity**: high
- **Rule**: #6 — Обмежені списки (bounded lists)
- **Location**: app/src/loans.js:30 (`pageSize` parameter)
- **Description**: `pageSize` has no upper bound. A caller can pass `pageSize: 1_000_000` (or `Infinity`) and receive the entire dataset in one response, defeating the purpose of pagination and exposing the same performance problem the PR is meant to solve. Team convention 7 and review rule 6 require that page size is bounded from above.
- **Suggestion**: Add a constant such as `const MAX_PAGE_SIZE = 100;` and clamp: `pageSize = Math.min(pageSize, MAX_PAGE_SIZE)`. Also validate that `page >= 1` and `pageSize >= 1`.

### Finding 3
- **Severity**: high
- **Rule**: #5 — CHANGELOG
- **Location**: app/CHANGELOG.md (missing entry)
- **Description**: `listLoansPage` is a new public function exported from `src/loans.js`. Team convention 6 and review rule 5 require that every new or changed public function gets a CHANGELOG entry in the same PR. The diff does not touch `app/CHANGELOG.md`.
- **Suggestion**: Add an entry under a `## 2026-10` section, e.g.: `` `listLoansPage(db, user, { page, pageSize })` — paginated variant of `listLoans`; returns `{ items, page, pageSize, total }`. ``

### Finding 4
- **Severity**: high
- **Rule**: #7 — Конкретні тести (concrete tests)
- **Location**: app/test/loans.test.js:37-41 (new test)
- **Description**: The test "paged list reports the total and the page it returned" asserts only metadata (`total`, `page`, `pageSize`) but never checks `res.items` — the actual paginated data. Because of this, the off-by-one bug in Finding 1 is invisible to the test suite: with `page: 1` and `pageSize: 20`, the buggy formula returns `items: []` (an empty array), yet all three assertions pass. A test that does not verify the core output of the function is not a concrete test per rule 7.
- **Suggestion**: Assert on `res.items` content. For example: `assert.deepEqual(res.items.map(l => l.id), [1, 2, 3, 4, 5, 6, 7]);` for a first page that fits all 7 loans, or use a smaller `pageSize` and verify exact item ids on multiple pages.

### Finding 5
- **Severity**: high
- **Rule**: #8 — Тест на відмову доступу (access denial test)
- **Location**: app/test/loans.test.js (missing test)
- **Description**: `listLoansPage` inherits access control from `listLoans` (which throws when `user` is null and filters loans by role). Review rule 8 requires that a new function with access checking has a test where access is denied. The PR adds no such negative-path test.
- **Suggestion**: Add at least: (1) a test that `listLoansPage(db, null)` throws; (2) a test that a regular member receives only their own loans.

### Finding 6
- **Severity**: medium
- **Rule**: #10 — Опис PR = код (PR description matches code)
- **Location**: review-queue/pr-1/PR.md, app/src/loans.js:29 (JSDoc)
- **Description**: The PR description says "Список позик для персоналу" (loan list for staff) and the JSDoc reads "Paged variant for the staff UI." However, the code does not restrict access to staff — it delegates to `listLoans`, which allows any authenticated user.
- **Suggestion**: Either add `requireRole(user, STAFF)` if this is truly staff-only, or fix the JSDoc and PR description.

## Tests

**Pass** — all 14 tests pass after applying the diff. However, the new test is weak: it does not catch the off-by-one bug because it never asserts on `res.items`.

## Verdict

**REQUEST CHANGES** — six findings, five of which block merge:

1. **Critical off-by-one** in pagination formula (`page * pageSize` should be `(page - 1) * pageSize`).
2. **No upper bound** on `pageSize` — violates rule 6.
3. **Missing CHANGELOG entry** for new public function — violates rule 5.
4. **Test does not check items** — the core output is unverified — violates rule 7.
5. **No access-denial test** — violates rule 8.
6. **PR description / JSDoc says "staff"** but code allows any user — violates rule 10.
