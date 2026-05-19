# Build Your Own Agentic Orchestrator

This guide teaches you how to build an autonomous maintenance system like Knight from scratch. The same pattern works for any codebase, any language, any maintenance task.

---

## Prerequisites

- A GitHub repository with code you want to maintain (like W.O.M.A.N)
- A free GitHub account (for Actions + repo hosting)
- A free OpenCode Zen account (for DeepSeek V4 Flash Free)
- Basic familiarity with YAML and git

---

## Step 1: Define Your Maintenance Plan

Create a `maintenance-plan.yaml` that encodes the rules your agent will follow. This is the most important file — it determines the quality of every action.

```yaml
# maintenance-plan.yaml

always:
  - Read existing code before making changes
  - Never break backward compatibility
  - Keep functions under 50 lines

before_build:
  - Run the test suite first to get a baseline

testing_rules:
  - Every new function needs a test
  - Every bug fix needs a regression test
  - Test error paths explicitly

code_review_rules:
  - No shell=True in subprocess calls
  - Use Path over os.path
  - Annotate all function signatures

refactoring_priority:
  1. Fix bugs
  2. Add tests
  3. Improve error messages
  4. Reduce complexity
  5. Update docs
```

**What to customize:**
- Language-specific rules (import conventions, typing requirements)
- Security rules for your domain
- Testing requirements (minimum coverage, mocking rules)
- Documentation standards

---

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
  "provider": {
    "zen": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "OpenCode Zen",
      "options": {
        "baseURL": "https://opencode.ai/zen/v1/chat/completions"
      },
      "models": {
        "deepseek-v4-flash-free": {
          "name": "DeepSeek V4 Flash Free",
          "limit": {
            "context": 1048576,
            "output": 262144
          },
          "options": {
            "reasoningEffort": "max",
            "thinking": {
              "type": "enabled"
            }
          }
        }
      }
    }
  },
  "model": "zen/deepseek-v4-flash-free",
  "autoCompact": true
}
```

---

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
        run: npm install -g @opencode-ai/plugin

      - name: Configure provider
        run: |
          mkdir -p ~/.config/opencode
          # Copy or create your opencode-config.json here

      - name: Analyze & Plan
        working-directory: codebase
        run: |
          opencode \
            --model zen/deepseek-v4-flash-free \
            -p ../maintenance-plan.yaml \
            "Analyze the codebase. Write a plan to /tmp/plan.md"

      - name: Build
        working-directory: codebase
        run: |
          opencode \
            --model zen/deepseek-v4-flash-free \
            -p ../maintenance-plan.yaml \
            -f /tmp/plan.md \
            "Execute the maintenance plan."

      - name: Test
        working-directory: codebase
        run: |
          python -m pytest || npm test || echo "No test command"

      - name: Tag & Push
        working-directory: codebase
        run: |
          git config user.name "Maintenance Agent"
          git config user.email "agent@your-org.com"
          git add -A
          git diff --quiet || git commit -m "maintenance: $(date)"
          git tag -f maintained
          git remote set-url origin \
            https://x-access-token:${{ secrets.GH_PAT }}@github.com/your-org/your-repo.git
          git push origin --force refs/tags/maintained
```

---

## Step 4: The Moving Tag Pattern

Instead of creating branches (which grow forever), use a moving tag:

```bash
# Create/update the tag
git tag -f maintained

# Force push (replaces the old tag)
git push --force origin refs/tags/maintained
```

### Why Tags Over Branches

| Aspect | Branch | Moving Tag |
|---|---|---|
| Git history | Grows forever | Single point that moves |
| GitHub display | Listed under branches | Listed under tags |
| Default checkout | Easy | Need to specify |
| CI trigger | Automatic on push | Manual workflow_dispatch |
| Use case | Long-lived development | Ephemeral snapshots |

---

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

---

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

---

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

---

## Step 8: Evolve Your Plan

The maintenance plan isn't static. As you learn what works:

1. Add rules for common bugs you find
2. Tighten testing requirements
3. Add documentation patterns
4. Remove rules that cause more harm than good

The system maintains itself — the plan file is part of the repo, so future cycles can improve it.

---

## Template: Minimum Viable Maintenance Plan

If you want to start simple, here's a 10-line plan:

```yaml
always:
  - Keep backward compatibility
  - Run tests after changes

testing_rules:
  - Every new function needs a test

code_review_rules:
  - No shell=True
  - Handle all errors
  - Write clear error messages
```

Even this minimal plan will catch 80% of common issues.

---

## What You Will Have Built

After following these steps, you'll have an autonomous system that:
1. Runs for free, forever
2. Fixes bugs before you know they exist
3. Writes tests for untested code
4. Keeps documentation current
5. Improves its own improvement process

> "The best maintenance system is the one that runs itself."
