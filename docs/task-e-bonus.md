# Task E (Bonus): Stability Test

**Path chosen**: Option 2 — Stability (run the same PR three times, compare).

**Target PR**: PR-3 (`exportMembersCsv`).

**Reason for choosing PR-3**: It has a mix of critical (role check, logAccess),
medium (CSV escaping), and low-severity defects, making it a good test of whether
the reviewer consistently finds issues at all severity levels.

## Session results

| Finding | Answer key | S1 | S2 | S3 | Stability |
|---------|------------|----|----|-----|-----------|
| 3-D1: Missing role check (critical) | critical | Found | Found | Found | **3/3** |
| 3-D2: CSV not escaped (low) | low | Found | Found | Found | **3/3** |
| 3-D3: Missing logAccess (high) | high [standard] | Found | Found | Found | **3/3** |

All three answer-key defects were found in all three sessions.

## Supplementary findings (beyond answer key)

| Supplementary finding | S1 | S2 | S3 | Stability |
|----------------------|----|----|-----|-----------|
| Unbounded export (no pagination) | Found | Found | — | 2/3 |
| Incomplete denial test (role-based) | — | Found | Found | 2/3 |
| Test doesn't check row content | Found | Found | Found | 3/3 |
| PR description mismatch (columns) | Found | Found | — | 2/3 |

## Observations

1. **Critical/high defects are fully stable**: The three answer-key defects
   (3-D1, 3-D2, 3-D3) were found in 100% of sessions with consistent severity
   classification. This is the most important result — the reviewer reliably
   catches the defects that matter most.

2. **Supplementary findings vary at the margin**: Findings beyond the answer key
   (unbounded export, denial test gap, PR description mismatch) appeared in 2-3
   out of 3 sessions. These are valid observations but not always surfaced — they
   depend on model attention allocation on a given run.

3. **Severity labeling is consistent for blockers**: All sessions agreed that the
   role-check and logAccess defects are blockers/critical. The CSV escaping issue
   was consistently rated medium-to-low.

4. **Verdict is stable**: All three sessions recommended "Request Changes" with
   the same two blockers cited as the primary reason.

## Conclusion

The reviewer demonstrates **high stability** for critical and high-severity
defects (3/3 sessions). Marginal findings (medium/low/advisory) show natural
variance but do not affect the overall verdict. For a team relying on this
reviewer as a first pass before human review, the consistency of blocker
detection is the key metric — and it scores 100%.

## Raw outputs

- `review-runs/pr-3-stability-s1.md`
- `review-runs/pr-3-stability-s2.md`
- `review-runs/pr-3-stability-s3.md`
