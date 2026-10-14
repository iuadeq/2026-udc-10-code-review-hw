# Review: PR-5

## Findings

### Finding 1
- **Severity**: critical
- **Rule**: #7 — Конкретні тести
- **Location**: app/test/loans.test.js:46
- **Description**: The assertion `assert.equal(lateFee(returnedLate, "2026-10-08"), 1500)` was weakened to `assert.ok(lateFee(returnedLate, "2026-10-08") > 0)`. Rule #7 explicitly prohibits this pattern: "`assert.ok(x > 0)` or 'the answer is not empty' is not a test. If a test blocks a change, understand why instead of weakening the test." The test was checking the exact fee for loan 4 (due 2026-10-04, returned 2026-10-07 = 3 days late = 1500 kopecks). With GRACE_DAYS=1 the new correct value is 1000 kopecks (2 billable late days × 500 kopecks). The test should assert that exact value.
- **Suggestion**: Replace with `assert.equal(lateFee(returnedLate, "2026-10-08"), 1000)`.

### Finding 2
- **Severity**: critical
- **Rule**: #7 — Конкретні тести
- **Location**: app/test/loans.test.js:55
- **Description**: The assertion `assert.equal(rows[0].feeKopecks, 11000)` was weakened to `assert.ok(rows[0].feeKopecks > 0)`. Same violation. The overdue report test for loan 6 (22 days late). With GRACE_DAYS=1, the new correct value is 10500 kopecks (21 billable late days × 500 kopecks).
- **Suggestion**: Replace with `assert.equal(rows[0].feeKopecks, 10500)`.

### Finding 3
- **Severity**: low
- **Rule**: #10 — Опис PR = код
- **Description**: The PR description frames the test weakening as "simplification" — which is the exact anti-pattern Rule #7 forbids. Weakening tests to avoid breakage on future changes is not simplification.
- **Suggestion**: Update tests with correct expected values instead of weakening assertions.

## Tests

**pass** — all 14 tests pass. However, two tests were weakened and no longer verify correctness of the fee calculation.

## Verdict

**REQUEST CHANGES** — The GRACE_DAYS logic itself is correct and the CHANGELOG entry is present. However, the PR weakens two existing test assertions from exact values to vague positivity checks, directly violating rule #7 (blocking). The fix is straightforward: replace the weakened assertions with the correct new expected values (1000 and 10500 kopecks).
