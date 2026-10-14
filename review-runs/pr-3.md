# Review: PR-3

## Findings

### Finding 1
- **Severity**: critical
- **Rule**: #1 — Role check
- **Location**: app/src/members.js:10 (new function `exportMembersCsv`)
- **Description**: `exportMembersCsv` only checks `if (!user)`, which means any authenticated user — including a regular `"member"` — can export the full member directory with all personal data. The neighboring `listMembers` function in the same module uses `requireRole(user, STAFF)`, establishing the convention that bulk member data is staff-only. This PR bypasses that protection entirely.
- **Suggestion**: Replace `if (!user) throw new AccessDenied();` with `requireRole(user, STAFF);`, consistent with `listMembers`.

### Finding 2
- **Severity**: critical
- **Rule**: #2 — Access log for personal data
- **Location**: app/src/members.js:12-13
- **Description**: The function returns the email and phone of every member in the database, but never calls `logAccess`. The team convention requires `logAccess(db, user, memberId, purpose)` to be called once per member whose personal data is disclosed. Compare with `memberCard` (same file, line 17) which calls `logAccess(db, user, m.id, "member-card")`.
- **Suggestion**: Iterate over the members and call `logAccess(db, user, m.id, "export-members-csv")` for each member.

### Finding 3
- **Severity**: high
- **Rule**: #8 — Test for access denied
- **Location**: app/test/members.test.js:41-42
- **Description**: The only access-denied test passes `null` as the user. This covers the anonymous-caller path but does not test denial by role. Since the function should restrict access to staff (see Finding 1), there must be a test proving that a regular member is denied.
- **Suggestion**: Add: `assert.throws(() => exportMembersCsv(createDb(), olya), AccessDenied);`

### Finding 4
- **Severity**: medium
- **Rule**: #6 — Bounded lists
- **Location**: app/src/members.js:12
- **Description**: The function dumps every member into a single CSV string with no pagination or size limit. As the member list grows, this produces an unbounded response.
- **Suggestion**: Accept optional pagination parameters or document the expected size constraint.

### Finding 5
- **Severity**: medium
- **Rule**: #10 — PR description matches code
- **Location**: review-queue/pr-3/PR.md
- **Description**: The PR description states "Bez korystuvacha — vidmova, yak i skriz" implying access control equivalent to the rest of the module, but it is not: `listMembers` and `overdueReport` use `requireRole(user, STAFF)`, while this function only checks for a null user.
- **Suggestion**: Fix the access control to match the rest of the module.

### Finding 6
- **Severity**: medium
- **Rule**: (security best practice)
- **Location**: app/src/members.js:13
- **Description**: Fields are concatenated with commas and no quoting or escaping. Commas in names break the CSV. Values starting with `=`, `+`, `-`, `@` can trigger formula injection in spreadsheets.
- **Suggestion**: Wrap each field in double quotes, escaping internal double-quotes (RFC 4180).

## Tests

**pass** — all 17 tests green. However, the test suite is incomplete: no role-based access denial test.

## Verdict

**REQUEST CHANGES** — two critical blockers: (1) missing role check (any user exports all data), (2) missing logAccess for personal data. Plus weak test coverage, unbounded list, and CSV injection.
