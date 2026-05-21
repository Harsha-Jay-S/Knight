# Build Your Own Agentic Orchestrator

This guide teaches you how to build an autonomous maintenance system like Knight from scratch. The same pattern works for any codebase, any language, any maintenance task.


## Prerequisites

- A GitHub repository with code you want to maintain (like W.O.M.A.N)
- A free GitHub account (for Actions + repo hosting)
- A free OpenCode Zen account (for DeepSeek V4 Flash Free)
- Basic familiarity with YAML and git


## Step 1: Define Your Maintenance Plan

Create a `maintenance-plan.yaml` that encodes the rules your agent will follow. This is the most important file — it determines the quality of every action.

```yaml
# maintenance-plan.yaml

persona:
  role: "Senior CLI tool maintainer with 15+ years experience"
  mindset: "Defensive coding, test-first, backward compatibility sacred"
  decision_framework: "If a change breaks existing behavior, don't make it"

always:
  - Read existing code before making changes
  - Never break backward compatibility
  - Keep functions under 50 lines
  - Use dataclasses over dicts; use Path over os.path; use f-strings

before_build:
  - Review Phase 1 test baseline results (already captured — do not re-run)
  - Check for subprocess.run(shell=True), hardcoded secrets, absolute paths
  - Identify functions with no tests — they get tests first

testing_rules:
  - Every new function needs a test
  - Every bug fix needs a regression test
  - Test error paths explicitly
  - Mock external calls — never hit real APIs in tests

code_review_rules:
  - Subprocess calls: MUST use list args, NEVER shell=True
  - File paths: sanitize, use Path.resolve(), guard against traversal
  - Error messages: tell the user WHAT went wrong and HOW to fix it
  - Annotate all function signatures with types

refactoring_priority:
  - Fix bugs
  - Add missing tests
  - Improve error messages
  - Fix security issues
  - Reduce complexity
  - Update docs
  - Modernize syntax

boundaries:
  - Do NOT reformat whitespace or change import style
  - Do NOT extract functions that are only called once
  - Do NOT change the public CLI interface
  - Limit changes to 5 files and 300 lines per cycle max
```

**What to customize:**
- Language-specific rules (import conventions, typing requirements)
- Security rules for your domain
- Testing requirements (minimum coverage, mocking rules)
- Documentation standards


## Step 2: Choose Your LLM Provider

Knight uses DeepSeek V4 Flash Free via OpenCode Zen — $0, no API key needed.

### Provider Options (All Free or Near-Free)

| Provider | Cost | Rate Limit | Quality |
|---|---|---|---|
| **DeepSeek V4 Flash Free** (Zen) | **$0** | 200 req/day | Excellent (79% SWE-bench) |
| OpenRouter DeepSeek V4 Flash | $0.07-0.28/M tokens | Varies by provider | Same model |
| Direct DeepSeek API | $0.07-0.28/M tokens | Dynamic | Same model |
| MiniMax M2.5 Free (Zen) | $0 | 200 req/day | Good for simpler tasks |

### Configure Your OpenCode Provider

Create `opencode-config.json`:

```json
{
  "model": "opencode/deepseek-v4-flash-free",
  "provider": {
    "opencode": {
      "models": {
        "deepseek-v4-flash-free": {
          "options": {
            "reasoningEffort": "max"
          }
        }
      }
    }
  },
  "agent": {
    "plan": {
      "steps": 20,
      "temperature": 0.1
    },
    "build": {
      "steps": 30,
      "temperature": 0.3
    }
  }
}
```

That's it. The `opencode/` prefix uses OpenCode's built-in provider — no custom provider config, no API keys, no SDK dependencies. `reasoningEffort: "max"` tells the model to think deeply before responding. The `agent.plan` and `agent.build` sections set per-agent step limits and temperature for deterministic analysis vs. creative building.


## Step 3: Set Up the GitHub Actions Scheduler

Create `.github/workflows/maintain.yml`:

```yaml
name: Autonomous Maintenance

on:
  schedule:
    - cron: '0 */6 * * *'   # Every 6 hours
  workflow_dispatch:          # Manual trigger

jobs:
  maintain:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: your-org/your-repo
          ref: main
          path: codebase

      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install opencode
        run: npm install -g opencode-ai

      - name: Configure opencode
        run: |
          mkdir -p ~/.config/opencode
          cp your-opencode-config.json ~/.config/opencode/opencode.json

      - name: Phase 0 — Pre-flight
        working-directory: codebase
        run: |
          ls *.py && python -m pytest --version

      - name: Phase 1 — Baseline
        working-directory: codebase
        run: |
          python -m pytest tests/ -v --tb=short > /tmp/test_baseline.txt 2>&1 || true

      - name: Phase 2 — Plan (AI, read only)
        working-directory: codebase
        run: |
          opencode run --model opencode/deepseek-v4-flash-free --variant max --agent plan \
            "$(cat maintenance-plan.yaml)

            Quickly skim source files for bugs and gaps.
            Write a prioritized plan to plan.md"

      - name: Phase 3 — Build (AI fixes)
        working-directory: codebase
        run: |
          opencode run --model opencode/deepseek-v4-flash-free --variant max \
            "$(cat maintenance-plan.yaml)

            $(cat plan.md 2>/dev/null || echo '(no plan file — proceed with maintenance-plan.yaml rules)')

            BUILD ONLY — fix ONLY what's in plan.md.
            Execute the plan. Add missing tests."

      - name: Phase 4 — Verify + Coverage
        working-directory: codebase
        run: |
          set -o pipefail
          python -m pytest tests/ -v --tb=short --cov=. 2>&1 | tee /tmp/test_final.txt
          grep -E "^TOTAL|^---" /tmp/test_final.txt || echo "(no coverage line found)"

      - name: Phase 4 — Scope check
        working-directory: codebase
        run: |
          CHANGED=$(git diff --name-only | wc -l)
          LINES=$(git diff --stat | tail -1 | grep -oP '\d+(?= insertions?(?!\.))' || echo 0)
          echo "Files changed: $CHANGED  |  Lines added: $LINES"
          if [ "$CHANGED" -gt 5 ]; then
            echo ":warning: WARNING: Changed $CHANGED files (max per boundaries)"
          fi

      - name: Phase 5 — Tag & Push
        working-directory: codebase
        run: |
          git config user.name "Maintenance Agent"
          git config user.email "agent@your-org.com"
          if ! git diff --quiet; then
            git add -A
            SUMMARY=$(git diff --cached --shortstat)
            git commit -m "maintenance: $(date -u '+%Y-%m-%d') [run:${{ github.run_id }}] — $SUMMARY"
          fi
          git tag -f maintained
          set +x
          git remote set-url origin \
            https://x-access-token:${{ secrets.GH_PAT }}@github.com/your-org/your-repo.git
          set -x
          git push origin --force refs/tags/maintained
```


## Step 4: The Moving Tag Pattern

Instead of creating branches (which grow forever), use a moving tag:

```bash
# Create/update the tag
git tag -f maintained

# Force push (replaces the old tag)
git push --force origin refs/tags/maintained
```

### Rollback via Reflog

Every force-push overwrites the tag, but the previous commit isn't lost:

```bash
git reflog show refs/tags/maintained
git checkout <old-sha>
```

### Why Tags Over Branches

| Aspect | Branch | Moving Tag |
|---|---|---|
| Git history | Grows forever | Single point that moves |
| GitHub display | Listed under branches | Listed under tags |
| Default checkout | Easy | Need to specify |
| CI trigger | Automatic on push | Manual workflow_dispatch |
| Use case | Long-lived development | Ephemeral snapshots |


## Step 5: The Cross-Repo Authentication

When your workflow is in Repo A but pushes to Repo B, you need a PAT:

```yaml
- name: Push to Repo B
  run: |
    git remote set-url origin \
      https://x-access-token:${{ secrets.GH_PAT }}@github.com/owner/repo-b.git
    git push origin --force refs/tags/maintained
```

### Token Setup
1. GitHub -> Settings -> Developer settings -> Fine-grained tokens
2. Scope: Only the target repo (Repo B)
3. Permission: Contents (Write)
4. Store as: GH_PAT secret in Repo A


## Step 6: Handle Failures Gracefully

Add retry logic and backoff:

```yaml
- name: Plan (with retry)
  working-directory: codebase
  run: |
    for i in 1 2 3; do
      opencode ... && break
      echo "Attempt $i failed, waiting 30s..."
      sleep 30
    done
```

Or simpler — just let it fail and retry next cycle. Knight uses the latter approach. Maintenance is never urgent.


## Step 7: Monitor and Iterate

### Check the Logs
1. Go to your GitHub Actions tab
2. Click on the latest workflow run
3. Each step shows its output

### Common Issues and Fixes

| Symptom | Likely Cause | Fix |
|---|---|---|
| Workflow never starts | Cron not on default branch | Merge to main/default |
| Push fails | PAT expired or scoped wrong | Regenerate token |
| OpenCode fails silently | Model name wrong | Check opencode-config.json |
| Tests fail after build | Plan didn't account for test patterns | Update maintenance-plan.yaml |
| Rate limited | Too many cycles | Increase from 6h to 12h |


## Step 8: Evolve Your Plan

The maintenance plan isn't static. As you learn what works:

1. Add rules for common bugs you find
2. Tighten testing requirements
3. Add documentation patterns
4. Remove rules that cause more harm than good

The system maintains itself — the plan file is part of the repo, so future cycles can improve it.


## Template: Minimum Viable Maintenance Plan

If you want to start simple, here's a 10-line plan:

```yaml
persona:
  role: "Senior maintainer"
  mindset: "Defensive, backward-compatible, test-first"

always:
  - Keep backward compatibility
  - Run tests after changes

testing_rules:
  - Every new function needs a test

code_review_rules:
  - No shell=True
  - Handle all errors
  - Write clear error messages

boundaries:
  - Do NOT change the public API
  - Limit to 5 files and 300 lines per cycle
```

Even this minimal plan will catch 80% of common issues.


## What You Will Have Built

After following these steps, you'll have an autonomous system that:
1. Runs for free, forever
2. Fixes bugs before you know they exist
3. Writes tests for untested code
4. Keeps documentation current
5. Improves its own improvement process

> "The best maintenance system is the one that runs itself."
