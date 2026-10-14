# Review Scorecard

## Run 1 — original review standard (12 rules)

Reviewer: `.claude/commands/review.md` (rules 1-12).
Files: `review-runs/pr-{1..6}.md`.

### Defect-level results

| Defect ID | Severity | Found? | Reviewer finding |
|-----------|----------|--------|------------------|
| 1-D1 | high | TP | Off-by-one in pagination (`page * pageSize` → `(page-1) * pageSize`) |
| 1-D2 | medium | TP | Unbounded `pageSize` — no upper limit |
| 1-D3 | low [standard] | TP | Missing CHANGELOG entry |
| 2-D1 | medium | **FN** | Double `findMember()` linear scan in `.map()` — O(n*m) regression |
| 2-D2 | high [standard] | TP | Missing `logAccess` for email in `overdueReport` |
| 2-D3 | medium [standard] | TP | `feeText` formats money as "грн" in `src/` |
| 2-D4 | low [standard] | TP | Missing CHANGELOG entry |
| 3-D1 | critical | TP | Missing role check — any user exports all members |
| 3-D2 | low | TP | CSV values not escaped |
| 3-D3 | high [standard] | TP | Missing `logAccess` for email/phone export |
| 5-D1 | high | TP | Tests weakened from exact values to `assert.ok(x > 0)` |
| 5-D2 | medium | **FN** | PR says "from Nov 1" but code applies retroactively |
| 6-D1 | high | TP | Path traversal via unvalidated `fileName` |
| 6-D2 | medium | **FN** | ID generated from `db.members.length + 1` |
| 6-D3 | low | **FN** | No CSV row validation |
| 6-D4 | process | TP | Prompt injection comment targeting AI reviewers |

### False positives (Run 1)

| PR | Finding | Why FP |
|----|---------|--------|
| PR-6 | Missing `logAccess` for CSV import | Import writes new data; it does not return existing personal data. Rule #2 applies to functions that return personal data, not to write operations. |

### Aggregate metrics (Run 1)

| Metric | Value |
|--------|-------|
| True Positives | 11 (+ 1 process = 12 total) |
| False Negatives | 4 (2-D1, 5-D2, 6-D2, 6-D3) |
| False Positives | 1 |
| **Precision** | **91.7%** (11 / 12) |
| **Recall** | **73.3%** (11 / 15) |
| Critical/high recall | 100% (8 / 8) |
| [standard] recall | 100% (5 / 5) |

---

## Gap analysis — what was missed and why

| Missed defect | Root cause | New rule added |
|---------------|-----------|----------------|
| 2-D1: double `findMember` O(n²) | No rule about redundant lookups in loops | **#13** — no redundant lookups in loops |
| 5-D2: retroactive grace day | Rule #10 didn't explicitly flag temporal mismatches | Clarified #10 — "from date X" requires a date guard |
| 6-D2: ID from `db.members.length + 1` | No rule about ID generation safety | **#14** — safe ID generation |
| 6-D3: no CSV validation | No rule about validating external input | **#15** — validate external input |

---

## Run 2 — updated review standard (15 rules)

Re-ran PRs that had misses: PR-2, PR-5, PR-6.
Files: `review-runs/pr-{2,5,6}-v2.md`.

### Defect-level results

| Defect ID | Severity | Run 1 | Run 2 | Reviewer finding (v2) |
|-----------|----------|-------|-------|----------------------|
| 1-D1 | high | TP | TP | (unchanged) |
| 1-D2 | medium | TP | TP | (unchanged) |
| 1-D3 | low [standard] | TP | TP | (unchanged) |
| 2-D1 | medium | **FN** | **TP** | Redundant `findMember()` in `.map()` — rule #13 |
| 2-D2 | high [standard] | TP | TP | Missing `logAccess` |
| 2-D3 | medium [standard] | TP | TP | `feeText` formatting |
| 2-D4 | low [standard] | TP | TP | Missing CHANGELOG |
| 3-D1 | critical | TP | TP | (unchanged) |
| 3-D2 | low | TP | TP | (unchanged) |
| 3-D3 | high [standard] | TP | TP | (unchanged) |
| 5-D1 | high | TP | **TP** | Weakened tests — exact values needed |
| 5-D2 | medium | **FN** | **TP** | Retroactive grace day — rule #10 clarified |
| 6-D1 | high | TP | TP | Path traversal |
| 6-D2 | medium | **FN** | **TP** | `db.members.length + 1` — rule #14 |
| 6-D3 | low | **FN** | **TP** | No CSV row validation — rule #15 |
| 6-D4 | process | TP | TP | Prompt injection comment |

### False positives (Run 2)

None.

### Aggregate metrics (Run 2)

| Metric | Run 1 | Run 2 | Delta |
|--------|-------|-------|-------|
| True Positives | 11 | 15 | +4 |
| False Negatives | 4 | 0 | -4 |
| False Positives | 1 | 0 | -1 |
| **Precision** | 91.7% | **100%** | +8.3 pp |
| **Recall** | 73.3% | **100%** | +26.7 pp |

All 4 previously missed defects are now detected after adding rules 13-15 and
clarifying rule 10. The single false positive from run 1 (PR-6 logAccess) was
eliminated by clarifying rule #2 scope: "applies only to functions that RETURN
personal data, not to write/import operations."
