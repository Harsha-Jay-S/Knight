# Architecture — How Knight Maintains the Queen

## Overview

Knight is an autonomous maintenance agent that runs as a GitHub Actions workflow. Every 6 hours, it clones the W.O.M.A.N codebase (v2 from the Queen branch), analyzes it, plans improvements, builds them, verifies with tests, and pushes the result to the `v2-maintained` tag.


## The 6-Phase Cycle

### Phase 0: Pre-flight Health Check

```bash
ls *.py
ls tests/
python -c "import woman_revamp"
python -m pytest --version
```

Before any AI runs, the Knight validates repo structure and dependencies. This catches broken checkouts, missing files, and import failures early — so the AI never runs against a broken baseline.

### Phase 1: Analyze (Baseline)

```bash
python -m pytest tests/ -v --tb=short > /tmp/test_baseline.txt
```

Run the existing test suite to establish a baseline. This catches pre-existing failures and provides a reference point for later comparison. The test output is archived.

### Phase 2: Plan (AI Analysis — Read Only)

```bash
opencode run --model opencode/deepseek-v4-flash-free \
  "$(cat maintenance-plan.yaml)

   ANALYSIS ONLY — do NOT make any edits.
   Quickly skim these key files: cli.py, engine.py, ui.py, config.py
   Then check tests/*.py for coverage holes.
   Write a prioritized maintenance plan to plan.md"
```

The AI reads key source files and test files to identify bugs, coverage gaps, and tech debt. It writes `plan.md` — but does **not** modify any code. This phase is capped at 10 minutes.

### Phase 3: Build (AI Fixes)

```bash
opencode run --model opencode/deepseek-v4-flash-free \
  "$(cat maintenance-plan.yaml)

   $(cat plan.md)

   BUILD ONLY — fix ONLY the issues listed in plan.md.
   Execute the maintenance plan at plan.md.
   Fix all identified issues. Add missing tests."
```

The AI executes the plan from Phase 2 — fixing bugs, adding tests, refactoring. Each change respects the rules in `maintenance-plan.yaml`:
- No `shell=True` in subprocess calls
- No bare `except:` clauses
- Every new function gets a test
- Error messages are user-friendly
- Max 5 files and 300 lines changed per cycle

This phase is also capped at 10 minutes.

### Phase 4: Verify + Scope Check

```bash
# Create symlink so 'import woman_revamp' resolves to current directory
mkdir -p /tmp/pylink
ln -sf "$PWD" /tmp/pylink/woman_revamp
PYTHONPATH=/tmp/pylink:$PYTHONPATH \
  python -m pytest tests/ -v --tb=short --cov=.
```

Run the full test suite with coverage. Then check `git diff` to enforce the 5-file / 300-line boundary. Warnings are printed if exceeded — the commit still proceeds so subsequent cycles can fix the overage.

**Why the symlink?** The W.O.M.A.N repo has all `.py` files at the repo root — there is no `woman_revamp/` directory. The package name comes from `pyproject.toml` via `pip install -e .`, but editable install doesn't make it importable as `woman_revamp` at test time. The symlink `/tmp/pylink/woman_revamp → $PWD` solves this.

### Phase 5: Tag & Push

```bash
git add -A
git commit -m "maintenance: 2026-05-20 06:00 UTC — 6 files changed, 502 insertions(+), 13 deletions(-)"
git tag -f v2-maintained
git tag v2-maintained-2026-05-20          # dated tag for rollback
git push --force origin refs/tags/v2-maintained
git push origin refs/tags/v2-maintained-2026-05-20
```

Two tags are pushed:
- `v2-maintained` — moving tag, always points to latest cycle
- `v2-maintained-YYYY-MM-DD` — dated tag for rollback safety

Previous commits are still accessible via git reflog and the dated tags.


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


## Security Model

```text
Fine-grained PAT
├── Repos: Knight, W.O.M.A.N only
├── Permission: Contents (write) + Metadata (read)
└── Stored as: GH_PAT secret in Knight repo

The PAT is encrypted by GitHub, injected at runtime.
The git remote URL uses set +x guards to prevent exposure in debug logs.
```

- Token scoped to exactly 2 repos
- Read/write on Contents only
- No user/admin scope
- Expires automatically (date set at creation)


## Cross-Repo Push

The Knight repo workflow pushes to the W.O.M.A.N repo using the PAT:

```yaml
- name: Push v2-maintained
  run: |
    set +x  # hide token from debug logs
    git remote set-url origin \
      https://x-access-token:${{ secrets.GH_PAT }}@github.com/Harsha-Jay-S/W.O.M.A.N.git
    set -x  # re-enable logging
    git push origin --force refs/tags/v2-maintained
```

The `set +x` / `set -x` guards prevent the PAT from appearing in GitHub Actions debug logs. This is the only cross-repo operation. The Knight repo never needs to read from W.O.M.A.N with elevated permissions — it only needs the clone (public) and the tag push (authenticated).


## Rate Limit Engineering

| Resource | Limit | Our Usage | Safety Margin |
|---|---|---|---|---|
| GitHub Actions | 2000 min/month | 1800 min/month | 10% |
| DeepSeek Zen | 200 req/day | ~80-120 req/day (4 cycles) | 40-60% |

If a rate limit is hit:
1. The step fails with a non-zero exit code
2. The workflow run is marked as failed
3. No commit/tag is pushed
4. The next scheduled run (6 hours later) retries

This is intentional. The system degrades gracefully — maintaining code is never urgent.


## Cost Breakdown

| Component | Cost | Category |
|---|---|---|
| GitHub Actions VM | $0 | Free tier (2000 min/month) |
| DeepSeek V4 Flash Free | $0 | OpenCode Zen free tier |
| OpenCode CLI | $0 | Open source |
| GitHub hosting (2 repos) | $0 | Public repos |
| pytest + Python deps | $0 | Open source |
| **Total** | **$0/month** | **Zero** |
