# Architecture — How Knight Maintains the Queen

## Overview

Knight is an autonomous maintenance agent that runs as a GitHub Actions workflow. Every 6 hours, it clones the W.O.M.A.N codebase (v2 from the Queen branch), analyzes it, plans improvements, builds them, verifies with tests, and pushes the result to the `v2-maintained` tag.

---

## The 6-Phase Cycle

### Phase 1: Sync

```
git clone --branch Queen Harsha-Jay-S/W.O.M.A.N
```

The Knight workflow checks out the W.O.M.A.N repo at the `Queen` branch. This is the frozen v2 codebase — never modified in-place. The clone is always fresh (GitHub Actions gives us a clean VM each time), ensuring every cycle starts from the exact same baseline.

### Phase 2: Analyze

```bash
python -m pytest tests/ -v --tb=short
```

Before making any changes, the Knight runs the existing test suite to establish a baseline. This catches pre-existing failures and provides a reference point. The test output is saved for comparison.

### Phase 3: Plan

The Knight invokes opencode with the `maintenance-plan.yaml` rules and the DeepSeek V4 Flash Free model:

```bash
opencode --model zen/deepseek-v4-flash-free \
  -p maintenance-plan.yaml \
  "Analyze woman_revamp thoroughly. Read all test files.
   Check coverage. Identify bugs, tech debt, docs gaps.
   Write plan to /tmp/plan.md"
```

DeepSeek V4 Flash Free (284B total / 13B activated parameters) reads:
- All Python source files in `woman_revamp/`
- All test files in `tests/`
- The `pyproject.toml` for dependency info
- The latest test results

It then writes `/tmp/plan.md` — a prioritized, actionable maintenance plan.

### Phase 4: Build

The Knight executes the plan:

```bash
opencode --model zen/deepseek-v4-flash-free \
  -p maintenance-plan.yaml \
  -f /tmp/plan.md \
  "Execute the plan. Fix issues, add tests, improve docs."
```

DeepSeek makes all file edits — fixing bugs, adding test cases, refactoring functions, updating docstrings. Each change respects the rules in `maintenance-plan.yaml`:
- No `shell=True` in subprocess calls
- No bare `except:` clauses
- Every new function gets a test
- Error messages are user-friendly

### Phase 5: Verify

```bash
python -m pytest tests/ -v --tb=short --cov=woman_revamp
```

Full test suite + coverage report. If tests fail, the cycle still completes (the commit preserves the failed state for debugging), but subsequent cycles will attempt to fix the failures.

### Phase 6: Tag & Push

```bash
git add -A
git commit -m "maintenance: 2026-05-20 06:00 UTC"
git tag -f v2-maintained
git push --force origin refs/tags/v2-maintained
```

The `v2-maintained` tag is force-pushed to the W.O.M.A.N repo. This is a **moving tag** — it always points to the latest maintenance commit. Previous commits are still accessible via git reflog.

---

## Why GitHub Actions (Not Docker)

| Approach | Pros | Cons |
|---|---|---|
| **Docker (local machine)** | Full control, persistent state | Requires 24/7 uptime, uses electricity |
| **GitHub Actions** | Free, no uptime needed, serverless | Ephemeral (fresh VM each cycle), 2000 min/month limit |

GitHub Actions wins because:
1. Your laptop can sleep
2. It's truly free (2000 min/month)
3. No infrastructure to maintain
4. The ephemeral nature is a feature — every cycle starts clean

---

## The Moving Tag Strategy

```text
Queen branch:  A---B---C---D---E  (v2, frozen)

v2-maintained tag:
               E---M1---M2---M3
                       ↑
              git tag -f v2-maintained
              (moves each cycle)
```

- **Queen** stays frozen as v2. Never modified.
- **v2-maintained** is a lightweight tag that moves (force-pushed) each cycle.
- Each cycle adds one commit on top of the previous state.
- The tag history is linear — you can always see what the Knight changed.

---

## Security Model

```text
Fine-grained PAT
├── Repos: Knight, W.O.M.A.N only
├── Permission: Contents (write) + Metadata (read)
└── Stored as: GH_PAT secret in Knight repo

The PAT is encrypted by GitHub, injected at runtime,
and never logged or exposed.
```

- Token scoped to exactly 2 repos
- Read/write on Contents only
- No user/admin scope
- Expires automatically (date set at creation)

---

## Cross-Repo Push

The Knight repo workflow pushes to the W.O.M.A.N repo using the PAT:

```yaml
- name: Push v2-maintained
  run: |
    git remote set-url origin \
      https://x-access-token:${{ secrets.GH_PAT }}@github.com/Harsha-Jay-S/W.O.M.A.N.git
    git push origin --force refs/tags/v2-maintained
```

This is the only cross-repo operation. The Knight repo never needs to read from W.O.M.A.N with elevated permissions — it only needs the clone (public) and the tag push (authenticated).

---

## Rate Limit Engineering

| Resource | Limit | Our Usage | Safety Margin |
|---|---|---|---|
| GitHub Actions | 2000 min/month | 1800 min/month | 10% |
| DeepSeek Zen | 200 req/day | 100 req/day | 50% |

If a rate limit is hit:
1. The step fails with a non-zero exit code
2. The workflow run is marked as failed
3. No commit/tag is pushed
4. The next scheduled run (6 hours later) retries

This is intentional. The system degrades gracefully — maintaining code is never urgent.

---

## Cost Breakdown

| Component | Cost | Category |
|---|---|---|
| GitHub Actions VM | $0 | Free tier (2000 min/month) |
| DeepSeek V4 Flash Free | $0 | OpenCode Zen free tier |
| OpenCode CLI | $0 | Open source |
| GitHub hosting (2 repos) | $0 | Public repos |
| pytest + Python deps | $0 | Open source |
| **Total** | **$0/month** | **Zero** |
