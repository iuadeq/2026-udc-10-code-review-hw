# Review: PR-2 (v2 — після Task C)

## Findings

### Finding 1
- **Severity**: critical
- **Rule**: #2 — Personal-data audit log
- **Description**: `overdueReport` now returns `email` for each member in the report (line 54: `email: db.findMember(l.memberId)?.email ?? null`). Per rule #2, any function whose return value contains a member's email must call `logAccess(db, user, memberId, purpose)` once per member whose data is returned. There is no `logAccess` call anywhere in the function. The existing `memberCard` function in `members.js` demonstrates the expected pattern — it calls `logAccess(db, user, m.id, "member-card")` before returning email/phone.
- **Suggestion**: Before the `.map()`, collect the set of overdue member IDs from the filtered loans, then call `logAccess(db, user, memberId, "overdue-report")` for each unique member ID. Import `logAccess` from `./audit.js`.

### Finding 2
- **Severity**: critical
- **Rule**: #4 — Money as kopecks only
- **Description**: Line 58 adds `feeText: (lateFee(l, todayIso) / 100).toFixed(2).replace(".", ",") + " грн"` inside `src/loans.js`. Rule #4 states that human-readable money formatting (`"15,00 грн"`) belongs on the client, not in `src/`. The API should expose only integer kopecks via `feeKopecks`, which it already does.
- **Suggestion**: Remove the `feeText` field entirely. Let the client format `feeKopecks` for display.

### Finding 3
- **Severity**: critical
- **Rule**: #13 — No redundant lookups in loops
- **Description**: The diff removes the pre-built `names` Map and replaces it with `db.findMember(l.memberId)` calls inside `.map()` (lines 53-54). `db.findMember()` performs `members.find((m) => m.id === id)` — a linear scan. It is called twice per loan in the loop (once for `.name`, once for `.email`), making member lookups O(n*m) where n = overdue loans and m = members. The original code used a pre-built Map for O(1) name lookups; this PR regresses that. Additionally, `db.findBook(l.bookId)` on line 55 is also a linear search inside the loop (pre-existing, but the PR had an opportunity to fix it).
- **Suggestion**: Pre-build Maps before the loop: `const memberMap = new Map(db.members.map(m => [m.id, m]));` and `const bookMap = new Map(db.books.map(b => [b.id, b]));`. Then use `memberMap.get(l.memberId)?.name`, `memberMap.get(l.memberId)?.email`, and `bookMap.get(l.bookId)?.title` inside `.map()`.

### Finding 4
- **Severity**: high
- **Rule**: #5 — CHANGELOG
- **Description**: The PR adds two new fields to `overdueReport`'s return value (`email` and `feeText`) and changes how `member` is resolved — this is a change to what a public function in `src/` returns. There is no corresponding entry in `app/CHANGELOG.md`.
- **Suggestion**: Add a CHANGELOG entry under a new `## 2026-10` section documenting the added `email` field in `overdueReport` (and `feeText` if it survives review — though per Finding 2 it should not).

### Finding 5
- **Severity**: high
- **Rule**: #10 — PR description matches code
- **Description**: The PR description says the change adds email to the overdue report ("для кожного рядка беремо учасника і віддаємо ім'я та email"). However, the code also adds `feeText` — a formatted money string — which is not mentioned in the description. An undocumented behavior change is a mismatch between description and code.
- **Suggestion**: Either remove `feeText` (recommended — it also violates rule #4), or update the PR description to explain why `feeText` is included.

## Verdict

**BLOCK** — request changes.

The PR has five findings, three of which are critical:
- Missing `logAccess` for personal data (rule #2)
- Money formatting in `src/` (rule #4)
- Redundant `findMember`/`findBook` linear searches inside `.map()` — a regression from the original pre-built Map (rule #13)

Additionally: missing CHANGELOG entry (rule #5) and undocumented `feeText` field in the PR description (rule #10).

Tests pass (rule #12: 13/13 green). No issues found for rules #1, #3, #6, #7, #8, #9, #11, #14, #15.
