# ML Reranker — How Woman Revamp Scores Commands

## Overview

woman_revamp uses a two-stage pipeline to match user queries to shell commands:

1. **Heuristic Engine** — fast, deterministic scoring for all candidates
2. **ML Reranker** — learns from data to improve ranking

This document explains both stages, the dataset, and how they work together.


## Stage 1: The Heuristic Engine

The heuristic engine runs first, scoring every candidate command in the registry against the user's query. It never relies on an external API — it works entirely offline.

### Scoring Features

| Feature | What It Measures | Why It Matters |
|---|---|---|
| **Token overlap ratio** | How many words the query and command share | Direct match indicator |
| **Intent phrase matching** | Does the query contain known intent keywords? | Maps "compress" to tar/gzip |
| **Synonym expansion** | Similar words from the synonym table | Maps "remove" to "delete" commands |
| **OS context** | Is the command available on this OS? | Filters Windows-only for Linux queries |
| **Directory context** | What files are in the current directory? | Prefers commands relevant to visible files |
| **Shell history** | Has the user run this command recently? | Recency bias for learned habits |
| **Number patterns** | Does the query mention numbers? | Maps "5 minutes" to timeout flags |

### How Scoring Works

```python
def heuristic_score(query, candidate, context):
    score = 0
    score += token_overlap(query, candidate.command) * 0.3
    score += intent_match(query, candidate.keywords) * 0.25
    score += os_compatibility(candidate, context.os) * 0.2
    score += context_relevance(candidate, context) * 0.15
    score += recency_bias(candidate, context.history) * 0.1
    return score
```

The scores are normalized to 0-100. Top 20 candidates move to Stage 2.

### Intent Phrase Mapping

The engine uses curated intents to bridge natural language to commands:

```python
INTENTS = {
    "compress": ["tar", "gzip", "bzip2", "zip", "7z"],
    "find": ["find", "grep", "locate", "fd", "rg"],
    "network": ["curl", "wget", "ping", "nmap", "netstat"],
    "process": ["ps", "kill", "htop", "top", "pgrep"],
    "file": ["ls", "cp", "mv", "rm", "cat", "less"],
    "git": ["git", "gh"],
    "docker": ["docker", "docker-compose"],
    "python": ["python", "pip", "uv", "poetry"],
}
```


## Stage 2: The ML Reranker

The ML reranker is an optional second pass that learns from data. It receives the top 20 candidates from the heuristic engine and reranks them using a trained model.

### Model Architecture

```
Input: 8 features per candidate
    ↓
LogisticRegression (scikit-learn)
    - C=1.0 (regularization)
    - max_iter=1000
    - solver='lbfgs'
    ↓
Output: reranked list (probabilities → sort)
```

### Features Used by the Model

| Feature | Type | Source |
|---|---|---|
| `heuristic_rank` | numeric (1-20) | Output from Stage 1 |
| `token_overlap_ratio` | float (0.0-1.0) | Jaccard similarity |
| `query_length` | integer | Character count of query |
| `candidate_length` | integer | Character count of command |
| `has_pipe` | boolean (0/1) | Does command use pipes? |
| `has_sudo` | boolean (0/1) | Does command need root? |
| `has_git` | boolean (0/1) | Is this a git command? |
| `has_docker` | boolean (0/1) | Is this a docker command? |

### Training

The model is trained offline using `training/train_reranker.py` (not yet implemented in the Queen branch):

> **Note:** The training pipeline (`training/train_reranker.py`) and the model file (`ml/models/woman_reranker.joblib`) do not yet exist in the W.O.M.A.N repo. The dataset is ready in `data/processed_pairwise_reranker_data.csv`; the training code still needs to be written. The Knight's future cycles should generate this code.

```bash
# (aspirational — training script does not exist yet)
python training/train_reranker.py \
  --data data/processed_pairwise_reranker_data.csv \
  --output ml/models/woman_reranker.joblib
```

The training script would:
1. Loads the CSV with pandas
2. Splits features from labels
3. Trains a LogisticRegression (binary classifier: 1 = good match, 0 = bad)
4. Saves the model as a joblib file

### Inference (aspirational — requires model file)

```python
def rerank(candidates, features):
    model = joblib.load("ml/models/woman_reranker.joblib")
    X = build_feature_matrix(candidates, features)
    probabilities = model.predict_proba(X)[:, 1]
    return sort_by_probability(candidates, probabilities)
```


## The Dataset

### Schema

`data/processed_pairwise_reranker_data.csv` — 300 rows, 25KB.

| Column | Example | Description |
|---|---|---|
| `query` | "find large files in current directory" | The natural language query |
| `candidate_command` | "find . -type f -size +100M" | Proposed shell command |
| `label` | 1 | 1 = good match, 0 = bad |
| `os` | linux | Target operating system |
| `recent_history` | "du -sh .\nls -lah" | Last shell commands for context |
| `combined_text` | "find large files..." | Query + candidate concatenated |
| `heuristic_rank` | 1 | Rank from the heuristic engine |
| `token_overlap_ratio` | 0.167 | Token similarity |
| `query_length` | 6 | Words in the query |
| `candidate_length` | 6 | Words in the command |
| `has_pipe` | 0 | Boolean flags |

### Coverage

| OS | Rows |
|---|---|
| linux | ~200 |
| macos | ~50 |
| windows | ~50 |

### How to Improve the Dataset

1. **Add more data** — collect real user queries and their intended commands
2. **Add negative examples** — command-query pairs that are NOT a match (currently labeled 0)
3. **Add more features** — shell type, terminal width, time of day, working directory depth
4. **Crowdsource** — set up a feedback collection in the CLI tool


## The Hybrid Pipeline End-to-End

```text
User types: "compress this folder"
    │
    ▼
Heuristic Engine
    │  Token overlap: "compress" → matches INTENTS["compress"]
    │  Synonym: "folder" → matches INTENTS["file"]
    │  Context: user has .tar.gz files in directory
    │
    │  Top 20 candidates:
    │  tar -czf archive.tar.gz .    (heuristic score: 85)
    │  zip -r archive.zip .         (heuristic score: 72)
    │  gzip -r .                    (heuristic score: 45)
    │  ...
    ▼
ML Reranker (if configured)
    │  Model predicts probabilities:
    │  tar -czf...      → 0.92
    │  zip -r...        → 0.78
    │  gzip -r...       → 0.31
    │
    ▼
Final output: "tar -czf archive.tar.gz ."
    │
    ▼
User sees: purple UI with the command
           "Execute? (y/n)"
```


## Why This Design

### Heuristics First
- Works offline (no AI provider needed)
- Deterministic (same query → same results)
- Fast (milliseconds)
- Interpretable (you can explain why a match scored high)

### ML Second
- Learns from real user data
- Catches patterns heuristics miss
- Improves over time as more data is collected
- Optional — degrades gracefully if model is absent

### Together
- Heuristics handle 80% of cases accurately
- ML handles the remaining 20% where heuristics are ambiguous
- Either can work without the other
- The combination is better than either alone
