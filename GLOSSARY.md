# Glossary — Everything Explained

Every term, tool, and concept in the Knight project, explained in plain English. For when you encounter jargon you don't recognize, or want to understand why something works the way it does.


## A-Z Jargon

### Agent
**What it is:** A program that makes decisions and takes actions autonomously (without human input at each step).

**Why it matters here:** Knight is an agent. Every 6 hours, it decides what to fix, how to fix it, whether tests pass, and whether to push. No human reviews each decision.

**Example:** `opencode` acts as Knight's agent — it reads code, plans fixes, edits files, and runs tests, all without you watching.

### Autonomous
**What it is:** Self-governing. A system that operates independently once configured.

**Why it matters here:** Knight is fully autonomous. You set it up once, and it runs maintenance cycles forever (until the PAT expires or GitHub Actions changes free tier).

### Cron
**What it is:** A time-based job scheduler. Pronounced "kron." Runs commands at specified intervals.

**Why it matters here:** `cron: '0 */6 * * *'` in the workflow file means "run every 6 hours." The five fields are: minute, hour, day-of-month, month, day-of-week.

**Example:** `0 */6 * * *` = at minute 0 of every 6th hour (midnight, 6 AM, noon, 6 PM).

### DeepSeek V4 Flash Free
**What it is:** A large language model (MoE architecture, 284B total / 13B active parameters) with a 1M-token context window, optimized for speed and coding tasks. The "Flash" variant prioritizes speed; "Free" means it's available at no cost through OpenCode's Zen tier.

**Why it matters here:** This is the brain behind Knight. It analyzes code, plans fixes, writes tests, and refactors. At 79% SWE-bench, it competes with paid models while costing $0.

**Context:** Released April 2026 by DeepSeek. Available through OpenCode Zen at `https://opencode.ai/zen/v1/chat/completions` with model ID `deepseek-v4-flash-free`.

### Fine-grained PAT
**What it is:** A Personal Access Token (PAT) limited to specific repositories and permissions. Unlike classic tokens (which have access to all repos you can access), fine-grained tokens can be scoped to exactly one or two repos.

**Why it matters here:** Knight's PAT has write access to exactly 2 repos (Knight and W.O.M.A.N) and read access to nothing else. If it leaks, only those 2 repos are at risk.

### GitHub Actions
**What it is:** GitHub's built-in CI/CD platform. You define workflows in YAML files; GitHub runs them on virtual machines (Ubuntu, Windows, macOS) for free up to 2000 minutes per month.

**Why it matters here:** Knight is a GitHub Actions workflow. GitHub provides the VM, the scheduler, the secrets storage, and the logs — all for free.

### Heuristic Engine
**What it is:** A rule-based scoring system that evaluates candidate commands without AI or ML. Uses hand-crafted rules (token overlap, synonym matching, intent detection) to rank results.

**Why it matters here:** woman_revamp's heuristic engine is the first-pass scorer. It's fast, deterministic, and works offline. It handles 80% of queries accurately before the ML reranker even runs.

### LLM
**What it is:** Large Language Model. A neural network trained on vast amounts of text that can generate human-like responses, write code, analyze text, etc.

**Why it matters here:** DeepSeek V4 Flash is the LLM that powers Knight's planning and building phases.

### LogisticRegression
**What it is:** A machine learning algorithm for binary classification (yes/no decisions). Despite the name, it's used for classification, not regression. It learns a linear decision boundary between two classes.

**Why it matters here:** woman_revamp's reranker uses LogisticRegression trained on pairwise data to predict whether a command-query pair is a good match (label=1) or not (label=0).

### Maintenance Plan
**What it is:** A YAML file (`maintenance-plan.yaml`) that encodes rules and priorities for an AI agent to follow during maintenance. It acts as a "constitution" — a set of constraints the agent must obey.

**Why it matters here:** Knight's `maintenance-plan.yaml` encodes 10+ years of CLI tool maintenance experience. Every action the agent takes is governed by these rules.

### MoE (Mixture of Experts)
**What it is:** A neural network architecture where different "expert" subnetworks handle different types of input. Only a subset of experts is activated for each input, making the model more efficient.

**Why it matters here:** DeepSeek V4 Flash has 284B total parameters but only activates 13B per query. This makes it fast and cost-effective while maintaining high quality.

### Moving Tag
**What it is:** A git tag that is force-updated to point to different commits over time. Unlike a branch (which moves forward as commits are added), a moving tag overwrites its previous position.

**Why it matters here:** Knight uses `git tag -f v2-maintained` each cycle. The tag always points to the latest maintenance commit. Previous commits are accessible via reflog, but the tag itself "moves."

### OpenCode
**What it is:** An open-source AI coding agent (by anomalyco). It runs in the terminal, supports custom AI providers, and can autonomously read, edit, and create files in your codebase.

**Why it matters here:** OpenCode is the agent framework that Knight uses. It handles the AI interaction (connecting to DeepSeek), the file editing, and the shell command execution.

### OpenCode Zen
**What it is:** OpenCode's built-in provider system. You get an API key (or skip it for free models) and can access models like DeepSeek V4 Flash Free, MiniMax M2.5 Free, etc. without any external API setup.

**Why it matters here:** We use Zen's free tier so Knight can run without any API keys, subscriptions, or billing setup.

### Orchestrator
**What it is:** A system that coordinates multiple components to achieve a goal. In this context, it schedules, monitors, and manages the maintenance pipeline.

**Why it matters here:** Knight is an orchestrator. It coordinates: git operations (clone/push), Python testing (pytest), AI analysis (opencode + DeepSeek), and GitHub's infrastructure (Actions).

### Pairwise Data
**What it is:** Training data organized as pairs: (query, candidate_command, label). Each pair represents "given this query, is this command a good match?" (label=1) or not (label=0).

**Why it matters here:** The dataset `processed_pairwise_reranker_data.csv` contains 300 such pairs. The ML reranker trains on these to learn what makes a good command-query match.

### Reranker
**What it is:** A second-stage scoring system that takes a list of candidates from a first-pass system and re-orders them for better results.

**Why it matters here:** woman_revamp's ML reranker takes the top 20 candidates from the heuristic engine and reranks them using the trained model. The reranker is optional — if the model file is missing, heuristics alone are used.

### Reasoning Effort
**What it is:** A parameter that controls how much "thinking" an AI model does before responding. Higher values produce more thorough analysis at the cost of speed.

**Why it matters here:** Knight uses `reasoningEffort: "max"` for DeepSeek V4 Flash Free, ensuring the agent thinks deeply about code before making changes.

### SWE-bench
**What it is:** A benchmark that tests AI models on real-world software engineering tasks (bug fixes, feature additions from GitHub issues). Scores represent the percentage of tasks solved correctly.

**Why it matters here:** DeepSeek V4 Flash scores 79% on SWE-bench. For comparison, GPT-4 scores around 50-60% and top models score 80%+. Flash's 79% puts it in the top tier while being free.

### Token
**What it is:** The basic unit of text that an LLM processes. A token is roughly 3-4 characters in English. "Hello world!" might be 3 tokens: ["Hello", " world", "!"].

**Why it matters here:** Rate limits (200 req/day), context windows (1M tokens), and model capacity (284B parameters) all refer to tokens. Token counts determine how much code the model can analyze in one session.

### v2-maintained
**What it is:** The moving git tag that Knight pushes to after each maintenance cycle. It lives on the W.O.M.A.N repo and points to the latest maintained version of the codebase.

**Why it matters here:** This is the output of the entire Knight system. Check this tag to see what Knight changed.

### Woman / W.O.M.A.N
**What it is:** The codebase being maintained. Stands for "Working Omniscient Manager of Actual Needs" — an AI-powered CLI tool that converts natural language to shell commands.

**Why it matters here:** Woman is the "Queen" in the Knight/Queen metaphor. It's the valuable codebase that Knight protects and improves.

### Zen
**What it is:** OpenCode's provider platform. Offers free and paid AI models through an OpenAI-compatible API endpoint at `https://opencode.ai/zen/v1/chat/completions`.

**Why it matters here:** Knight's `opencode-config.json` points to Zen's endpoint with the `deepseek-v4-flash-free` model. No API key needed for free models.


## Project-Specific Terms

### Knight
**What it is:** The autonomous maintenance agent. Also the name of this GitHub repo.

**In context:** Knight is the guardian that watches over the Queen (W.O.M.A.N). It runs in GitHub Actions, uses opencode + DeepSeek to maintain the code, and pushes results to the `v2-maintained` tag.

### Queen
**What it is:** The W.O.M.A.N codebase at the `Queen` branch. This is the frozen v2 source of truth.

**In context:** Queen is never modified by Knight. It stays as v2 forever. Knight clones Queen, makes improvements, and saves the result as a new tag (`v2-maintained`).

### The 6-Phase Cycle
**What it is:** Knight's internal workflow — Pre-flight, Analyze (baseline), Plan (AI read-only), Build (AI fixes), Verify + Scope Check, Tag & Push. Each phase has a specific purpose and output.

**In context:** Every 6 hours, Knight runs through these 6 phases. The output of each phase feeds the next, creating a reliable maintenance pipeline.

### The 200-Request Math
**What it is:** The calculation that ensures the DeepSeek free tier (200 req/day) can handle Knight's workload (~80-120 req/day).

**In context:** Phases 0, 1, 4, and 5 use zero AI requests (they're shell/Python commands). Phase 2 (Plan) and Phase 3 (Build) each use ~10-15 opencode requests. At 4 cycles/day, that's ~80-120 requests/day — well within the 200 req/day free limit. This gives a 40-60% safety margin.

### The "10-Year Maintainer" Mindset
**What it is:** A set of software engineering principles encoded in `maintenance-plan.yaml` that reflect decades of experience maintaining production CLI tools.

**In context:** Rules like "never use shell=True," "every bug fix needs a regression test," "error messages must say what and how," and "never break backward compatibility" come from hard-earned experience. Knight applies them consistently every cycle.


## Why Reading and Learning from This Project Is Useful

### 1. It's a Case Study in Real Zero-Cost Infrastructure

Most tutorials show you how to build something assuming you have a budget. Knight shows you how to build something assuming you have *nothing*. It proves that free tiers, strategically combined, can run a production-grade system indefinitely.

**What you learn:** How to stack free tiers (GitHub Actions + OpenCode Zen + public repos) into a system that would cost $50-200/month with paid alternatives.

### 2. It Demonstrates Agentic Patterns You Can Reuse

The Plan → Build → Verify → Deploy loop generalizes to any automated task:
- Automated code review
- Security auditing
- Documentation generation
- Dependency upgrades
- Migration tooling
- Test generation

**What you learn:** A template for building autonomous agents. Change the maintenance-plan.yaml rules, change the codebase path, and you have a different agent.

### 3. It Shows How to Encode Expertise

The most valuable knowledge in any organization lives in senior engineers' heads. Knight's `maintenance-plan.yaml` is a mechanism for serializing that knowledge into rules that an AI can apply at scale.

**What you learn:** How to distill years of experience into machine-readable rules. This is the same pattern used by:
- Style guides (automated linting)
- Security policies (automated scanning)
- Architecture decisions (automated review)

### 4. It Teaches ML-in-Production Patterns

woman_revamp's hybrid heuristics + ML reranker is industry-standard production ML:
- Heuristics handle the common path (fast, deterministic, interpretable)
- ML handles edge cases (learns from data, improves over time)
- Either can work without the other (graceful degradation)

**What you learn:** A production ML pipeline from data collection through training through deployment, all in ~300 lines of Python and a 25KB CSV file.

### 5. It's a Working Example of Multi-Repo Automation

Knight lives in one repo but pushes to another. This cross-repo pattern requires understanding:
- Fine-grained PATs and secret management
- GitHub Actions cross-repo authentication
- Moving tag strategies for cross-repo publishing

**What you learn:** How to build systems that span multiple GitHub repos — useful for monorepo splits, artifact publishing, and federated workflows.

### 6. It's a Self-Improving System

Knight maintains the Queen, but the Knight repo itself is subject to maintenance. As the agent learns better patterns, it improves its own configuration.

**What you learn:** The bootstrap paradox — a system that improves its own improvement process — solved safely through separation of concerns (Queen vs. Knight vs. v2-maintained).

### 7. It Was Built for Free, By an LLM, On a CLI

This entire project — all the code, all the docs, all the architecture — was designed and built by an AI coding agent running on a CLI with a free model. That's a proof point that the tools and techniques in this repo are accessible to anyone with a command line and an internet connection.

**What you learn:** The barrier to building sophisticated autonomous systems is lower than ever. The limiting factor isn't budget or infrastructure — it's understanding the patterns.


> "The best way to learn is to read a working system, understand why every decision was made, and then build your own."

This project is designed for that. Every decision has a rationale. Every file has a purpose. Every term is defined here.

Read. Understand. Build something better.
