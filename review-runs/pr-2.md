# Review: PR-2

## Findings

### Finding 1
- **Severity**: critical
- **Rule**: #2 — Журнал доступу до персональних даних
- **Location**: app/src/loans.js:54 (after patch)
- **Description**: The function `overdueReport` now returns `email` for every overdue member, but never calls `logAccess(db, user, memberId, purpose)`. Email is personal data per team conventions. Compare with `memberCard` in `src/members.js`, which calls `logAccess` before returning email/phone. Here the report can expose emails for multiple members in a single call, and none of those accesses are recorded. This breaks the team's ability to answer "who looked at my data and why."
- **Suggestion**: Import `logAccess` from `./audit.js` and call it once per member whose email is returned.

### Finding 2
- **Severity**: critical
- **Rule**: #4 — Гроші лише копійками
- **Location**: app/src/loans.js:58 (after patch)
- **Description**: The new field `feeText: (lateFee(l, todayIso) / 100).toFixed(2).replace(".", ",") + " грн"` formats money as a human-readable string inside `src/`. Team convention #4 explicitly forbids this: "Formatting for humans (`15,00 грн`) is done by the client, not `src/`." The API already returns `feeKopecks` in integer kopecks.
- **Suggestion**: Remove the `feeText` field entirely. The client should format `feeKopecks` for display.

### Finding 3
- **Severity**: high
- **Rule**: #5 — CHANGELOG
- **Location**: app/CHANGELOG.md (missing entry)
- **Description**: The PR adds two new fields (`email`, `feeText`) to the return shape of the public function `overdueReport`. This changes what the function returns, which requires a CHANGELOG entry per team convention #6. No entry was added.
- **Suggestion**: Add an entry to `app/CHANGELOG.md` under a new `## 2026-10` section documenting the new fields.

### Finding 4
- **Severity**: high
- **Rule**: #10 — Опис PR = код
- **Location**: review-queue/pr-2/PR.md
- **Description**: The PR description says the change adds email to the overdue report. However, the code also adds `feeText` — a formatted fee string — which is not mentioned anywhere in the PR description. Undocumented changes are suspicious per rule #10.
- **Suggestion**: Either remove the undocumented `feeText` field (which also violates rule #4), or update the PR description to cover all changes.

## Tests

**Pass** — all 13 tests pass after applying the diff. Rule #12 is satisfied.

## Verdict

**REQUEST CHANGES** — 4 violations found, 2 critical blockers:

1. **(Critical, #2)** Missing `logAccess` calls — email is personal data and every access must be logged.
2. **(Critical, #4)** Human-readable money formatting (`feeText` with "грн") in server code — must be removed.
3. **(High, #5)** Missing CHANGELOG entry for new return fields.
4. **(High, #10)** PR description does not mention the `feeText` addition — undocumented change.
