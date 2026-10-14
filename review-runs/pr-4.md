# Review: PR-4

## Findings

### Finding 1
- **Severity**: medium
- **Rule**: #10 — PR description matches code
- **Location**: review-queue/pr-4/PR.md and app/src/dates.js:6
- **Description**: The PR description states "Поведінка не змінюється" (behavior doesn't change), framing this as a pure refactoring. However, the previously private function `toUtc` in `loans.js` has been renamed to `toUtcMs` and made a public export in the new `dates.js` module. This is an API surface expansion, not merely a move.
- **Suggestion**: Update the PR description to acknowledge that `toUtcMs` is a new public function.

### Finding 2
- **Severity**: low
- **Rule**: #5 — CHANGELOG
- **Location**: app/CHANGELOG.md:6
- **Description**: The CHANGELOG entry says date helpers "moved here from `loans.js`" and lists `toUtcMs`. But `toUtc` was a private function — it was promoted from private to public, not moved. The entry exists but the wording is misleading.
- **Suggestion**: Reword to make it clear that `toUtcMs` is a newly exported function.

## Summary of rules checked

| # | Rule | Result |
|---|------|--------|
| 1 | Role check | pass — pure utility functions, no data access |
| 2 | Access log | pass — no functions return email/phone |
| 3 | Path validation | pass — no file paths |
| 4 | Money in kopecks | pass — lateFee still returns kopecks |
| 5 | CHANGELOG | minor — entry exists but wording misleading |
| 6 | Bounded lists | pass |
| 7 | Concrete tests | pass — all new tests assert specific values |
| 8 | Denial test | pass — no new access-controlled functions |
| 9 | Dates in UTC | pass — `Date.UTC()` used correctly |
| 10 | PR description | medium — claims no behavior change but API surface expands |
| 11 | Untrusted content | pass |
| 12 | Tests pass | pass — 16/16 |

## Tests

**pass** — 16/16 tests pass (13 original + 3 new in `dates.test.js`).

## Verdict

APPROVE — The refactoring is mechanically correct, re-exports preserve backward compatibility, and all tests pass. The two findings are non-blocking documentation concerns.
