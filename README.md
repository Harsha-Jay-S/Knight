# `knight`

### Keeping the Queen in Shape While She Sleeps

> An autonomous maintenance agent. Because every queen needs a knight.

```bash
$ knight --status
  State:     Watching Queen (W.O.M.A.N)
  Cycle:     Every 6 hours on GitHub Actions
  Model:     DeepSeek V4 Flash Free
  Cost:      $0/month
```

---

## Why This Exists

The Queen's codebase was pristine. But code, like a kingdom, needs daily upkeep — bugs creep in, tests stagnate, documentation rots.

I could have set up a CI pipeline. Written some tests. Maybe a weekly reminder.

Instead I built a **Knight**.

It runs every 6 hours on **GitHub Actions (free tier)** , studies the Queen's code with **DeepSeek V4 Flash Free** (via OpenCode Zen, no API key needed), fixes what's broken, writes tests for what isn't, and pushes the result back to the `v2-maintained` tag on the W.O.M.A.N repo.

The Queen never lifts a finger. That's the point.

---

## What Tech It Uses

| Technology | Role | Cost |
|---|---|---|
| **DeepSeek V4 Flash Free** | Reasoning engine — plans and builds all fixes | $0 (OpenCode Zen, 200 req/day) |
| **OpenCode CLI** | Agent framework — runs the maintenance loop | $0 (open source) |
| **GitHub Actions** | Scheduler — triggers the loop every 6h | $0 (2000 min/month free tier) |
| **Python + pytest** | Test framework — verifies all changes | $0 |
| **woman_revamp** | The codebase being maintained (Queen) | Already exists |
| **GitHub** | Hosting — Knight repo + W.O.M.A.N repo | $0 (public repos) |

**Total: $0/month.**

---

## How It Works

### The 6-Phase Cycle

```
[GitHub Actions — cron: 0 */6 * * *]
        │
        ▼
┌─────────────────────────────┐
│ Phase 1: SYNC               │
│ Clone Queen (v2 from         │
│ W.O.M.A.N repo)              │
└────────┬────────────────────┘
         ▼
┌─────────────────────────────┐
│ Phase 2: ANALYZE             │
│ Run pytest baseline,          │
│ read all test files,          │
│ identify gaps                 │
└────────┬────────────────────┘
         ▼
┌─────────────────────────────┐
│ Phase 3: PLAN                │
│ opencode + DeepSeek V4 Flash │
│ reads every file, writes     │
│ /tmp/plan.md with priorities │
└────────┬────────────────────┘
         ▼
┌─────────────────────────────┐
│ Phase 4: BUILD               │
│ opencode executes plan:      │
│ fix bugs, add tests,         │
│ refactor, update docs         │
└────────┬────────────────────┘
         ▼
┌─────────────────────────────┐
│ Phase 5: VERIFY              │
│ pytest --cov on full suite   │
│ All tests must pass           │
└────────┬────────────────────┘
         ▼
┌─────────────────────────────┐
│ Phase 6: TAG & PUSH          │
│ git tag -f v2-maintained     │
│ git push --force origin tag  │
│ → W.O.M.A.N repo              │
└─────────────────────────────┘
         │
         ▼
    Sleep 6 hours → repeat
```

---

## Why This Is a Genius Move

### 1. Zero-Cost Autonomous Upkeep
No servers. No humans. No alerts. The system maintains itself on GitHub's free tier. If it fails (rate limit, API down), it simply skips a cycle and retries. No pager duty.

### 2. The 10-Year Maintainer Mindset Encoded
The `maintenance-plan.yaml` file encodes decades of CLI maintenance wisdom:
- Never use `shell=True`
- Every function needs a test
- Fix bugs before refactoring
- Error messages must tell the user *what* went wrong and *how* to fix it
- Backward compatibility is sacred

Every cycle, the agent makes decisions as if a senior engineer with 10+ years of CLI tool experience reviewed the code.

### 3. Separation of Concerns
| Layer | What | Who owns it |
|---|---|---|
| **Queen (v2)** | The woman codebase | Frozen, never touched |
| **Knight** | The maintenance infrastructure | This repo, evolves over time |
| **v2-maintained** | The maintained snapshot | Auto-generated every 6h |

The Queen stays pristine. The Knight does the dirty work. The `v2-maintained` tag is the result.

### 4. Bootstrap Paradox Solved
The maintenance system maintains the Queen, but the Knight also maintains itself. The `maintenance-plan.yaml` rules apply to the workflow files too. If a better pattern emerges, the next cycle adopts it.

### 5. ML + Heuristics Pipeline
The woman_revamp codebase uses a hybrid approach:
- **Heuristic engine**: Token overlap, synonym matching, intent extraction, context-aware scoring
- **ML reranker**: LogisticRegression model trained on the dataset in `data/` that reranks top heuristic candidates
- **Dataset**: 300 pairwise query-command examples across Linux, macOS, and Windows

See [ML_RERANKER.md](ML_RERANKER.md) for the full breakdown.

---

## What's in This Repo

```
Knight/
│
├── README.md                       ← This file — front door
├── ARCHITECTURE.md                 ← Deep dive into the 6-phase cycle
├── DESIGN_PHILOSOPHY.md            ← Why zero-cost autonomous maintenance works
├── BUILD_YOUR_OWN.md               ← Tutorial: build your own agentic orchestrator
├── ML_RERANKER.md                  ← How ML + heuristics + dataset work
├── GLOSSARY.md                     ← All jargon explained + why learning this matters
│
├── .github/workflows/maintain.yml  ← The 6-hour schedule
├── maintenance-plan.yaml           ← 10yr CLI maintainer wisdom
├── opencode-config.json            ← DeepSeek V4 Flash Free ($0, no key)
│
└── data/
    └── processed_pairwise_reranker_data.csv  ← Reranker training data (300 rows)
```

---

## Rate Limit Safety

| Limit | Free Tier | Our Usage | Headroom |
|---|---|---|---|
| GitHub Actions (min/month) | 2000 | ~1800 (4 cycles × ~15 min × 30 days) | 10% |
| DeepSeek Zen (req/day) | 200 | ~100 (4 cycles × ~25 requests) | 50% |

If either limit is hit, the cycle skips and retries in 6 hours. No crashes, no costs, no alerts.

---

## How to Use This Repo

### Read the docs
Start with [ARCHITECTURE.md](ARCHITECTURE.md) for the full system design, then [BUILD_YOUR_OWN.md](BUILD_YOUR_OWN.md) to learn how to create something similar.

### Check the latest maintenance
Go to [Harsha-Jay-S/W.O.M.A.N](https://github.com/Harsha-Jay-S/W.O.M.AAN) and look for the `v2-maintained` tag — it updates every 6 hours.

### Run the workflow manually
1. Go to Knight repo → Actions → "Knight Maintains the Queen"
2. Click "Run workflow"
3. Watch it cycle through all 6 phases

---

<!--

## Repos

| Repo | Branch/Tag | What |
|---|---|---|
| [W.O.M.A.N](https://github.com/Harsha-Jay-S/W.O.M.A.N) | `Queen` (branch) | v2 code — frozen, never touched |
| [W.O.M.A.N](https://github.com/Harsha-Jay-S/W.O.M.A.N) | `v2-maintained` (tag) | Maintained code — updated every 6 hours |
| [Knight](https://github.com/Harsha-Jay-S/Knight) | `main` (branch) | This repo — the infrastructure + docs |
-->

## Disclaimer

Written almost entirely by an LLM. I (the human who prompted it into existence) take no responsibility for what the Knight does to the Queen's codebase.

If a commit breaks something, `git revert` is your friend. The `v2-maintained` tag is force-pushed, so the previous commit is still in the reflog.

---

## License

MIT. Use it, fork it, rename it, ignore it.

---

<div align="center">

*Built because every queen needs a knight.*

</div>
