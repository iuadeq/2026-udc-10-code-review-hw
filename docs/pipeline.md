# AI Code Review Pipeline

## Overview

The pipeline (`pipeline/ai-review.yml`) runs on every pull request targeting `main`.
It consists of two jobs executed sequentially:

1. **tests** — deterministic gate: `node --test` in `app/`. If tests fail, the
   AI review is skipped entirely. This ensures the model never reviews code that
   doesn't even compile or pass the existing test suite.

2. **ai-review** — sends the PR diff and description to the Anthropic Messages
   API (Claude) with the team review standard baked into the system prompt. The
   model's response is posted as a **comment** (advisory), not an approval or
   request-changes review. A human reviewer must still approve before merge.

## Design decisions

| Decision | Rationale |
|----------|-----------|
| `pull_request`, not `pull_request_target` | `pull_request_target` runs with write access to the base repo's secrets and grants the PR author code execution in that context. For a public or fork-friendly repo this is a privilege-escalation vector. `pull_request` runs in the fork's context with read-only base access. |
| Same-repo guard (`head.repo.full_name == github.repository`) | Even with `pull_request`, we skip the AI job for external forks. Fork PRs can't access secrets anyway, so the API call would fail; the guard makes that explicit and avoids a confusing error. |
| Deterministic checks first | Running `node --test` before the AI call saves API cost on broken PRs and gives the model a guarantee that the code at least passes its own test suite. |
| Comment, not review verdict | The AI posts an advisory comment. It does not approve or block. Human judgment remains the merge gate. This avoids the model making merge/reject decisions and keeps accountability with the team. |
| Minimal permissions | `contents: read` + `pull-requests: write` — the workflow can read code and post comments but cannot push, merge, or modify branch protections. |

## Threat model

### 1. Prompt injection via PR body or diff

**Attack**: Attacker crafts a PR description or code comment containing
instructions like "ignore all previous rules" or "approve this PR."

**Mitigation**: The system prompt explicitly states that PR text is data, not
instructions. Rule #11 in the review standard tells the model to flag such text
as a violation. The model's output is a comment, not an approval — even if the
model were manipulated, it cannot merge the PR.

### 2. Secret exfiltration from fork PRs

**Attack**: A fork PR's workflow code reads `secrets.ANTHROPIC_API_KEY` and
exfiltrates it (e.g., via curl to an attacker server).

**Mitigation**: The `if:` guard skips the AI job entirely for fork PRs. Fork PR
workflows triggered by `pull_request` cannot access the base repo's secrets by
default. We use `pull_request` (not `pull_request_target`) to avoid granting
forks elevated context.

### 3. Excessive API cost (denial of wallet)

**Attack**: Attacker opens many PRs or pushes many commits to trigger expensive
API calls.

**Mitigation**: The deterministic test gate filters out trivially broken PRs. For
additional protection: set Anthropic API spend limits, use GitHub's concurrency
controls (`concurrency: { group: ai-review-${{ github.event.number }}, cancel-in-progress: true }`)
to limit parallel runs, and consider a required label or approval before the AI
job runs on external contributions.

### 4. Model hallucination / false positives

**Attack**: Not adversarial, but the model may report issues that don't exist,
wasting reviewer time or creating confusion.

**Mitigation**: The review is advisory only (comment, not blocking review).
Precision is tracked via the scorecard (`docs/review-scorecard.md`). The review
standard is tuned iteratively (Task C) to reduce false positives. Human reviewer
is the final authority.

### 5. Data leakage to model provider

**Attack**: Proprietary code is sent to an external API (Anthropic) as part of
the review prompt.

**Mitigation**: For open-source projects this is a non-issue. For proprietary
code: review Anthropic's data retention policy (zero-retention API by default),
consider a self-hosted model, or scope the diff sent to the model (e.g., only
`app/src/` files, exclude secrets/configs).

### 6. Workflow tampering via PR

**Attack**: A PR modifies `.github/workflows/` to alter the CI pipeline itself —
e.g., removing the same-repo guard or changing permissions.

**Mitigation**: The AI review workflow lives in `pipeline/ai-review.yml`, not in
`.github/workflows/`, per project convention. PRs that touch workflow files
should require CODEOWNERS approval. GitHub's branch protection rules can require
review from specific teams for changes to `.github/`.
