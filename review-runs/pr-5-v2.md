# Review: PR-5 (v2 — після Task C)

## Findings

### Finding 1
- **Severity**: critical
- **Rule**: #10 — PR description matches code
- **Description**: The PR description says "Нова політика бібліотеки з 1 листопада" (new library policy from November 1), the CHANGELOG says "library policy from 1 November", and the code comment says "політика з 1 листопада". However, the code unconditionally subtracts `GRACE_DAYS` from the late-day count in `lateFee()` for ALL loans, regardless of when they were taken, due, or returned. There is no date condition such as `if (loan.dueAt >= "2026-11-01")`. A loan that was due and returned in August 2026 — months before the November policy — would also receive the grace day reduction. This is a retroactive behavior change that contradicts the PR's stated temporal scope. Rule 10 explicitly requires: "if the description says 'from date X', the code must have a date condition — applying the change retroactively is a mismatch."
- **Suggestion**: Add a date condition to `lateFee` so that `GRACE_DAYS` is only subtracted for loans whose due date falls on or after 2026-11-01. For example: `const grace = loan.dueAt >= "2026-11-01" ? GRACE_DAYS : 0;` and then `const late = daysBetween(loan.dueAt, loan.returnedAt ?? onIso) - grace;`.

### Finding 2
- **Severity**: critical
- **Rule**: #7 — Concrete tests
- **Description**: The late-fee test (`app/test/loans.test.js`, line 40) was changed from `assert.equal(lateFee(returnedLate, "2026-10-08"), 1500)` to `assert.ok(lateFee(returnedLate, "2026-10-08") > 0)`. This is a weakened assertion: `assert.ok(x > 0)` does not verify the exact fee amount. It would pass for any positive value — whether the formula is correct, partially correct, or completely wrong. The PR author acknowledges weakening: "спростив їх до перевірки, що штраф нараховано, щоб наступна зміна тарифу знову все не ламала." The correct response to a test that fails after a formula change is to update the expected value to match the new formula, not to weaken the test.
- **Suggestion**: Update the expected value to match the new formula. With `GRACE_DAYS = 1`, loan 4 (3 days late minus 1 grace = 2 chargeable days) should yield `2 * 500 = 1000`. Write: `assert.equal(lateFee(returnedLate, "2026-10-08"), 1000)`.

### Finding 3
- **Severity**: critical
- **Rule**: #7 — Concrete tests
- **Description**: The overdue-report test (`app/test/loans.test.js`, line 49) was changed from `assert.equal(rows[0].feeKopecks, 11000)` to `assert.ok(rows[0].feeKopecks > 0)`. Same weakening pattern as Finding 2. The test no longer verifies the actual computed fee for the most overdue loan in the report. Any positive fee passes, which defeats the purpose of regression testing.
- **Suggestion**: Update the expected value. With `GRACE_DAYS = 1`, loan 6 (22 days late minus 1 grace = 21 chargeable days) should yield `21 * 500 = 10500`. Write: `assert.equal(rows[0].feeKopecks, 10500)`.

### Finding 4
- **Severity**: low
- **Rule**: #12 — Tests pass
- **Description**: All 13 existing tests pass after applying the diff. However, the tests only pass because two exact-value assertions were weakened to `assert.ok(x > 0)` (see Findings 2 and 3). This masks potential calculation errors rather than validating the new behavior.
- **Suggestion**: Once the exact expected values are restored (Findings 2–3), re-run the tests to confirm they pass with the corrected formula.

## Verdict

**BLOCK** — REQUEST CHANGES.

Three blocking violations found:

1. **Rule #10** (critical): The PR description claims the grace-day policy applies "from November 1", but the code applies `GRACE_DAYS` retroactively to all loans with no date condition. This mismatch between stated scope and actual behavior must be resolved — either add a date guard in the code, or rewrite the PR description and CHANGELOG to accurately state the change is retroactive.
2. **Rule #7** (critical, x2): Two test assertions were weakened from exact value checks to `assert.ok(x > 0)`. Tests must verify concrete expected results. The correct fix is to compute the new expected values under the updated formula and assert those exact amounts.
