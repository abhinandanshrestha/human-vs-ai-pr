# AI-Agent vs Human PR Dependency Vulnerability Pipeline

## Table of Contents

1. [Quick Start](#1-quick-start)
2. [The Research Question](#2-the-research-question)
3. [Which Notebook Should I Use?](#3-which-notebook-should-i-use)
4. [Background: What is AIDev?](#4-background-what-is-aidev)
5. [Background: The Tools](#5-background-the-tools)
6. [End-to-End Flowchart](#6-end-to-end-flowchart)
7. [Step-by-Step Breakdown](#7-step-by-step-breakdown)
8. [Data Artifacts Produced](#8-data-artifacts-produced)
9. [Rate-Limit & Resilience Strategy](#9-rate-limit--resilience-strategy)
10. [Why Each Design Choice Matters](#10-why-each-design-choice-matters)
11. [Expected Outputs](#11-expected-outputs)
12. [Caveats](#12-caveats)

---

## 1. Quick Start

```bash
cd C:\Users\abhinandan.shrestha\Documents\Test\AIAgentVulnPipeline

# 1. Create .env with your GitHub Personal Access Token (public_repo read scope)
echo "GITHUB_TOKEN=ghp_your_actual_token" > .env

# 2. Launch Jupyter and pick ONE notebook:
jupyter notebook AI_Agent_Vuln_Pipeline_Fast.ipynb   # RECOMMENDED — parallelized, rate-limit safe

# 3. Cell -> Run All
# 4. Inspect data_fast/final_analysis_dataset.csv when done
```

**Runtime estimate for the Fast notebook on all 9,509 repos**: 10-15 hours wall-clock (resumable).

---

## 2. The Research Question

> **When an AI coding agent modifies a project's dependencies, does it introduce known-vulnerable packages at a different rate — and with different severity — than a human developer performing the same task?**

This matters because:
- AI agents (Copilot, Cursor, Devin, Claude Code, OpenAI Codex) now open and merge real PRs in production repositories
- They're often rubber-stamped by reviewers who trust the agent's output
- If they introduce vulnerabilities at a higher rate, we need to know

The paper's answer: **AI-agent PRs introduce vulnerabilities at ~2× the odds of human PRs** (17.17% vs. 9.14%), and the ones they introduce skew **more severe** (36.8% critical vs. 20.5% for humans).

---

## 3. Which Notebook Should I Use?

There are **two notebooks** in this folder. Pick based on scope and time budget:

| Feature | `AI_Agent_Vuln_Pipeline.ipynb` (original) | `AI_Agent_Vuln_Pipeline_Fast.ipynb` ⚡ (recommended) |
|---|---|---|
| **Scope default** | Top 500 repos (`N_REPOS = 500`) | **All 9,509 repos** (`N_REPOS = None`) |
| **Human mining** | Sequential (one repo at a time) | **Parallel** (`ThreadPoolExecutor`, 8 workers) |
| **Semgrep scan** | Yes (slow, ~2h) | **Skipped** (paper doesn't use Semgrep) |
| **Rate-limit strategy** | Basic | **Pre-flight check + quota floor + soft throttle + secondary-limit retry** |
| **Progress output** | Every repo | Every 25 repos (flood control) |
| **CSV writes** | Append-mode | Append-mode (thread-safe via `csv_lock`) |
| **Writes to** | `data/` | `data_fast/` (separate folder, won't collide) |
| **Fast-path reuse** | No | **Yes** — reuses `ai_manifest_touching_prs.csv` and `osv-scanner.exe` from `data/` if present |
| **Full-scope runtime** | 10+ hours (slow single-thread) | **10-15 hours** (dominated by rate limits, not code) |

**Default choice**: use the Fast notebook. The original is kept as the canonical reference implementation.

---

## 4. Background: What is AIDev?

**AIDev** (and its curated subset **AIDev-Pop**) is a research dataset published on HuggingFace by **Hao Li et al.** at [`hao-li/AIDev`](https://huggingface.co/datasets/hao-li/AIDev).

### What's in it

| File | Contents | Columns we use |
|---|---|---|
| `pull_request.parquet` | Every PR authored by an AI agent on a public GitHub repo (>100★) | `id`, `agent`, `repo_id`, `user`, `created_at`, `merged_at` |
| `pr_commit_details.parquet` | One row per file changed in each PR (~12M rows, ~2 GB) | `pr_id`, `filename` |
| `repository.parquet` | Metadata for every repo in the dataset | `id`, `full_name` |

### How AIDev identifies AI-authored PRs

Li et al. identified PRs by:
- GitHub user accounts for known agents (`cursor[bot]`, `devin-ai-integration[bot]`, `openai-codex`, etc.)
- Commit metadata and PR descriptions indicating AI authorship (Copilot "co-authored by" trailers)
- Repository integrations that flag the agent

### AIDev-Pop subset we use

- **9,509 total repositories** containing AI-agent manifest-touching PRs
- **57,976 unique AI PRs** across 6 agents:

| Agent | PRs in dataset |
|---|---|
| Claude Code | 172,597 |
| Copilot | 98,384 |
| OpenAI Codex | 62,731 |
| Cursor | 12,908 |
| Devin | 11,219 |
| Google Jules | 3,457 |

(these are *total* PR counts; after filtering to manifest-touching + merged-only the final AI sample is ~1,200 PRs)

### Repo ranking — most important first

Repo counts are highly skewed:

| Position | Cumulative AI PR coverage |
|---|---|
| Top 100 | 25.7% |
| Top 500 | **50.0%** |
| Top 1,000 | 63.5% |
| Top 2,000 | 77.2% |
| Top 5,000 | 92.2% |
| All 9,509 | 100% |

The Fast notebook **ranks repos by AI PR count descending** and processes them in that order, so if you interrupt early (or hit a long quota pause) you already have the high-signal data.

Top 5 repos (processed first):
1. `openclaw/openclaw` — 706 AI PRs
2. `vm0-ai/vm0` — 509 AI PRs
3. `github/gh-aw` — 476 AI PRs
4. `ceedaragents/cyrus` — 363 AI PRs
5. `nomadkaraoke/karaoke-gen` — 355 AI PRs

### Why we trust it

AIDev is a **published**, **versioned**, **openly-downloadable** dataset — and our pipeline mines the matched human-authored baseline **fresh** from GitHub each run, so the human side is never stale.

---

## 5. Background: The Tools

### 5.1 OSV-Scanner

- **What it is**: Google's open-source vulnerability scanner
- **What it does**: Reads a dependency lockfile (e.g. `package-lock.json`), resolves every package version, and queries [OSV.dev](https://osv.dev) — an aggregated database of known vulnerabilities covering CVEs, GHSAs, GSD, RustSec, etc.
- **Output**: JSON listing every vulnerable package with its CVE/GHSA ID and severity (LOW / MODERATE / HIGH / CRITICAL)
- **Why it matters**: It scans **declared dependencies**, so it finds supply-chain risks — the kind introduced when an AI agent bumps `lodash` to a version with a known prototype-pollution bug
- **What it can't do**: It doesn't look at source code. A SQL injection the author wrote by hand in `app.py` is invisible to OSV

### 5.2 Semgrep (original notebook only)

- **What it is**: A static-analysis tool that pattern-matches **source code** against vulnerability rules
- **What it does**: Runs a ruleset (`--config auto`) over `.py`, `.js`, `.ts`, `.go`, `.java`, etc. — flagging hardcoded secrets, SQL injection, insecure deserialization, XSS patterns, weak cryptography
- **Status**: **Skipped in the Fast notebook**. The paper doesn't use Semgrep, and it was the single slowest step. If you want Semgrep findings back, run them selectively on the subset of PRs that OSV already flagged.

### 5.3 GitHub GraphQL API

- Used for **90% of data fetching** in this pipeline
- Reasons we prefer it over REST:
  1. **Batching with aliases** — one request fetches 50 PRs' worth of data instead of 50 requests
  2. **Field selection** — we only pay (in rate-limit quota) for fields we actually need
  3. **`rateLimit` field** embedded in every response — live quota tracking for free
- Rate limit: **5,000 "points" per hour per token** (points ≈ cost of each field requested)
- Falls back to REST for files >512 KB (a GraphQL blob text limit)

---

## 6. End-to-End Flowchart

```mermaid
flowchart TD
    Start([Start]) --> Setup[Step 1-3: Setup<br/>imports, scanner, .env token]
    Setup --> Check{ai_manifest_touching_prs.csv<br/>exists?}
    Check -- Yes --> FastPath[Step 4-5 FAST PATH:<br/>load CSV directly<br/>skip 2 GB parquet download]
    Check -- No --> SlowPath[Step 4-5 FULL PATH:<br/>load 3 parquets from HuggingFace<br/>classify 12M rows]
    FastPath --> Rank
    SlowPath --> Rank
    Rank[Step 6: Rank 9,509 repos<br/>by AI PR count DESC<br/>most important FIRST]

    Rank --> MineHuman[Step 7: Mine HUMAN PRs IN PARALLEL<br/>8 workers, ThreadPoolExecutor<br/>Quota-aware, secondary-limit safe<br/>APPEND to CSV per repo]

    MineHuman --> Combine[Step 8: Combine AI + Human<br/>into one PR list]
    Combine --> Numbers[Step 9: Backfill PR numbers<br/>for AI side]
    Numbers --> SHAs[Step 10: Batched SHA fetch<br/>50 PRs per GraphQL call]

    SHAs --> Filter[Step 11: Filter to scannable files<br/>group rows by PR]

    Filter --> OSV[Step 12: OSV-Scanner loop<br/>Fetch base + after content<br/>REST fallback if >512KB<br/>Compute introduced / fixed]

    OSV --> Review[Step 14: Fetch review metadata<br/>review_count, comment_count,<br/>state, merged_at]

    Review --> Final[Step 15: Assemble final CSV<br/>filter to MERGED only<br/>compute hours_to_merge]
    Final --> RQ1[Step 16: RQ1 quick-look<br/>chi-square + Fisher + OR]
    RQ1 --> End([Done:<br/>final_analysis_dataset.csv])

    style FastPath fill:#dfd,stroke:#333,stroke-width:2px
    style MineHuman fill:#ffd,stroke:#333,stroke-width:2px
    style OSV fill:#fdd,stroke:#333,stroke-width:2px
    style Final fill:#ddf,stroke:#333,stroke-width:2px
```

---

## 7. Step-by-Step Breakdown

### Step 1 — Imports and workspace paths
- Standard library + `pandas`, `requests`, `threading`, `concurrent.futures`
- `WORK_DIR` points to `data_fast/` for the Fast notebook (so it doesn't clobber the original's `data/`)
- **Reuses OSV binary** from `../data/` if present (saves 57 MB download)
- **Reuses AI manifest CSV** from `../data/` if present (saves 49 MB copy + HF trip)
- All artifact filenames declared up front

### Step 2 — Install OSV-Scanner (Semgrep skipped in Fast)
- Downloads `osv-scanner_windows_amd64.exe` → `data_fast/osv-scanner.exe`
- Only re-downloads if not already present or copied from `data/`
- Verifies with `--version`

### Step 3 — Load GitHub token from `.env`
- Reads `GITHUB_TOKEN=...` from `.env` in the notebook's folder
- Falls back to `GITHUB_TOKEN` environment variable
- Prints masked preview + remaining rate limit

### Step 4 — Load AIDev parquets (fast-path aware)
- **Fast path**: if `ai_manifest_touching_prs.csv` already exists, **skip the 2 GB `commit_details` download** — only fetch `pull_request.parquet` (needed for `bot_exclusions` in Step 6)
- **Full path**: load all 3 parquets (~30s + 2 GB network)
- Fast path is ~3s instead of ~90s + heavy network

### Step 5 — Classify dependency files by ecosystem (fast-path aware)
- Regex-matches filenames against 4 ecosystems: `python` / `typescript` / `go` / `csharp`
- **Fast path**: `pd.read_csv(AI_PRS_FILE)` — instant
- **Full path**: classify all 12M rows, join with PR + repo metadata, save

### Step 6 — Rank repos + build sampling frame
- **Rank all 9,509 repos by unique AI PR count** (descending)
- `N_REPOS = None` keeps all repos; integer limit keeps top-N
- `repo_paths` is an ordered dict `{full_name: [manifest_paths]}` preserving rank
- Build `bot_exclusions` (every AI-agent login + dependabot + renovate + github-actions)

### Step 7 — Parallel human-PR mining ⚡ **SLOWEST STEP**
See [§9 Rate-Limit & Resilience Strategy](#9-rate-limit--resilience-strategy) for the full concurrency/safety breakdown.

Per repo (in its own thread):
1. Paginate merged PRs (newest first, 50 per page)
2. Stop paging when `createdAt < global_since`
3. Skip authors in `bot_exclusions`
4. Intersect each PR's file list with `paths_of_interest` (manifest files AI touched)
5. Buffer matching rows

**Append-only CSV writes** (thread-safe via `csv_lock`):
```python
pd.DataFrame(repo_rows, columns=CSV_COLUMNS).to_csv(
    HUMAN_PRS_FILE, mode="a", header=False, index=False
)
```

Flood-controlled progress: full progress line only every 25 completions (or on any "big" repo with ≥50 rows).

### Step 8 — Combine AI + human sides
- Union both sides into `all_prs_needing_sha` — one row per PR with `pr_id`, `full_name`, `number`, `side`
- Human side already has PR numbers; AI side needs them backfilled next

### Step 9 — Backfill AI-side PR numbers
- AIDev stores `databaseId` but not the repo-scoped `number`
- Per-repo paginate merged PRs, match by `databaseId`, record `number`
- Save as `ai_pr_numbers.csv`

### Step 10 — Batched SHA fetch
For each PR we need 3 commit SHAs:
- `baseRefOid` — "before" state
- `mergeCommit.oid` — "after" state (primary)
- `headRefOid` — "after" fallback when mergeCommit is null

**Batched 50 PRs per GraphQL call** via field aliases:
```graphql
query {
  repository(owner: "x", name: "y") {
    pr0: pullRequest(number: 1) { baseRefOid headRefOid mergeCommit { oid } }
    pr1: pullRequest(number: 2) { baseRefOid headRefOid mergeCommit { oid } }
    ...
  }
}
```
→ ~50× fewer API calls.

### Step 11 — Filter to scannable files + group by PR
Only these manifests are scannable by OSV:
- `poetry.lock`, `requirements*.txt`
- `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`
- `go.mod`
- `packages.lock.json` (C# — rare)

`package.json` / `pyproject.toml` alone use unpinned version ranges (`^1.2.0`). OSV needs concrete versions, which only lockfiles provide.

Group row-by-row manifest touches → **one row per PR** — the atomic scan unit.

### Step 12 — OSV-Scanner loop ⚠️ **HEAVY STEP**
For each PR, for each manifest file:
1. GraphQL-fetch content at `base_sha`
2. GraphQL-fetch content at `after_sha` (`merge_sha` or `head_sha` fallback)
3. Write both snapshots to temp files
4. Run `osv-scanner scan --format json` on each
5. Compute `introduced = after - before`, `fixed = before - after` (set difference on vuln IDs)

**REST fallback** for files >512 KB (GraphQL blob text limit).
**Checkpointed every 25 PRs** → `osv_scan_results.csv`.

### Step 13 — Semgrep scan (SKIPPED in Fast notebook)
Original notebook only. Scans `.py` / `.js` / `.ts` source files for code-pattern vulnerabilities (SQLi, XSS, hardcoded secrets). Omitted from Fast variant because the paper doesn't use Semgrep and it was the slowest step.

### Step 14 — Review + merge-time metadata
Per-PR needs:
- `state` → to filter MERGED only
- `created_at` / `merged_at` → `hours_to_merge`
- `review_count` → number of code reviews
- `comment_count` → number of top-level PR comments

Batched 50 PRs per GraphQL call.

### Step 15 — Final assembly
```python
final = osv_df
         .merge(review_df, on="pr_id", how="left")
         .query("state == 'MERGED'")
final["hours_to_merge"] = (merged_at - created_at).hours
```

Prints side breakdown (AI vs Human counts), per-agent breakdown, and intro-vuln rates.

**Output**: `final_analysis_dataset.csv` — the analysis-ready file.

### Step 16 — RQ1 quick-look
2×2 proportion test on `(side, has_introduced_vuln)`:
- **Chi-square** with Yates' correction
- **Fisher's exact**
- **Odds ratio** with 95% CI

---

## 8. Data Artifacts Produced

Every file lives in `data_fast/` (or `data/` for the original notebook):

| Filename | Produced by | Write mode | Purpose |
|---|---|---|---|
| `ai_manifest_touching_prs.csv` | Step 5 | overwrite (or **reused**) | AI side raw (49 MB) |
| `human_manifest_touching_prs.csv` | Step 7 | **append** (one block per repo) | Human side raw |
| `human_mining_done_repos.txt` | Step 7 | **append** (one line per repo) | Resume checkpoint |
| `ai_pr_numbers.csv` | Step 9 | overwrite | AI PR number lookup |
| `all_pr_shas.csv` | Step 10 | overwrite | Commit SHAs for both sides |
| `osv_scan_results.csv` | Step 12 | overwrite every 25 PRs | Dependency vulns per PR |
| `osv_scan_done_ids.txt` | Step 12 | overwrite | Resume checkpoint |
| `review_metadata.csv` | Step 14 | overwrite per batch | Review/merge metadata |
| **`final_analysis_dataset.csv`** | Step 15 | overwrite | **THE OUTPUT** |
| `osv-scanner.exe` | Step 1-2 | downloaded / reused | OSV binary (57 MB) |
| `mining_failures.log` | Step 7 | append | Repos that failed GraphQL |
| `osv_scan_failures.log` | Step 12 | append | PRs OSV couldn't scan |

---

## 9. Rate-Limit & Resilience Strategy

The Fast notebook is designed for a **10-15 hour unsupervised run** against GitHub. Every layer of rate-limit protection stacks:

### Settings (top of Step 7 cell)
```python
MAX_WORKERS       = 8      # conservative; stability > throughput for long run
QUOTA_FLOOR       = 500    # pause pool at this threshold
QUOTA_WARN        = 1500   # soft-throttle zone
SLOW_SLEEP        = 1.5    # seconds between requests when in warn zone
LOG_EVERY         = 25     # progress line every N completions (flood control)
```

### Protection layers

| Mechanism | Trigger | Action |
|---|---|---|
| **Pre-flight check** | Before pool starts | Checks `/rate_limit`; sleeps until reset if starting quota < 500 |
| **Live quota tracking** | Every GraphQL response (free, embedded in query) | Updates shared `quota["remaining"]` counter under `state_lock` |
| **Soft throttle (warn)** | `remaining ≤ 1500` | Each worker sleeps 1.5s before next request |
| **Hard pause (floor)** | `remaining ≤ 500` | **All workers pause** via `threading.Event`, wait for quota reset + 10s buffer, then resume |
| **Secondary rate limit** | HTTP 403/429 with `Retry-After` header | Sleep the full `Retry-After` value (min 60s), then retry same request |
| **Per-request retries** | Transient network / 5xx | 5 retries with exponential backoff |
| **Failed-repo logging** | Worker exception | Logged to `mining_failures.log`, doesn't crash pool |

### Thread safety

| Lock | Guards |
|---|---|
| `csv_lock` | CSV appends, done-file writes, progress prints |
| `state_lock` | `quota` dict and `rate_counter` dict |
| `pause_event` | Collective pause: `.clear()` pauses all workers, `.set()` resumes |

Each thread uses its own `requests.Session` for connection pooling.

### Resume behavior

The checkpoint files make the whole thing idempotent:
- `human_mining_done_repos.txt` → repos to skip
- `human_manifest_touching_prs.csv` → already-mined data
- On re-run, `done_now` set is reloaded from disk; only incomplete repos are submitted to the pool
- In-flight repos at interruption time simply re-run — mild wasted work, no corruption

---

## 10. Why Each Design Choice Matters

### Why **rank repos by AI PR count** and process from most → least important?
Repos with 1-2 AI PRs cost the same GraphQL quota to mine as repos with hundreds, but contribute almost no statistical signal. Ranking them first means:
- The first ~1,000 repos (biggest ones) cover 63.5% of all AI PRs — ~2 hours of work
- The last ~4,500 repos cover only ~8% — they can be truncated if time is short
- Interrupt-and-resume gives you a usable partial dataset at any point

### Why **matched sampling** (same repos, same window)?
If we compared AI PRs in bleeding-edge React repos against human PRs in sleepy enterprise Java repos, differences would be driven by ecosystem/repo culture — not AI vs. human. Matching eliminates this confound.

### Why **merged-only**?
An unmerged PR introduces nothing — its vulnerabilities are hypothetical. We only count what actually landed in production codebases.

### Why **both base and after**, not just after?
A PR that updates `lodash` from a vulnerable version to a safe one would look "bad" if we only scanned the after-state (lodash is still there). Measuring `after - before` correctly attributes it as **fixing** a vulnerability, not introducing one.

### Why **append-mode CSV writes** in Step 7?
- The naive approach (`df.to_csv(path)` after each repo) rewrites the entire file every time. As the CSV grows, each checkpoint takes longer — O(N²) in total I/O
- Append-mode (`mode="a", header=False`) only serializes **this repo's rows** — flat O(N) cost
- Memory stays flat: only the current repo's rows held in RAM
- On resume, we don't `read_csv()` the whole file — just count lines

### Why **parallel workers** (ThreadPoolExecutor) instead of asyncio?
- Threading is dead simple for I/O-bound work and the stdlib `concurrent.futures` API is easy to reason about
- `requests.Session` works cleanly per-thread
- `pandas` is not asyncio-friendly, so we'd be mixing paradigms
- With `MAX_WORKERS=8`, the GIL isn't a bottleneck — every worker spends >99% of its time in `requests.post` (released GIL)

### Why **skip Semgrep** in the Fast variant?
- The paper doesn't use Semgrep at all — it's an addition from the earlier exploratory notebook
- Semgrep takes 2-5 seconds per source file × 10 files × every PR × 2 snapshots = **multiple seconds per PR**, ~2 hours total
- If you need code-level findings, run Semgrep offline on just the PRs OSV flagged (tiny fraction)

### Why **batched GraphQL (50 per call)**?
GitHub's 5,000-points-per-hour budget would last ~90 seconds against ~150K PR-detail requests if each got its own call. Batching 50:1 stretches that to ~75 minutes of quota per cycle.

### Why **no helper functions in notebook cells**?
Every operation is inline so you can read each cell top-to-bottom, pause mid-loop, inspect intermediate state via `df.head()`, and understand exactly what's happening without jumping between cells. The only exception is `_mine_one_repo` in Step 7, which is unavoidable because `ThreadPoolExecutor.submit` requires a callable — but it lives **inside the cell** (not at module scope).

---

## 11. Expected Outputs

### Final CSV structure
```
pr_id      side   agent   full_name         ecosystems   introduced_count  review_count  hours_to_merge
123456     ai     devin   openclaw/openclaw python       2                 3             16.4
123457     human  human   openclaw/openclaw python       0                 1             4.2
123458     ai     copilot github/gh-aw      typescript   0                 5             28.9
...
```

### Approximate sizes

| Scope | Repos | AI PRs | Human PRs (est.) | Step 7 runtime |
|---|---|---|---|---|
| Top 500 | 500 | ~29,000 | ~8,000-15,000 | ~30-60 min (slow NB) / ~5-10 min (Fast NB) |
| **Full 9,509** | **9,509** | **~58,000** | **~16,000+** | **~10-15 hours (Fast NB)** |

### Headline statistics the paper reports (full-scope reference)
- Pooled AI intro rate: **17.17%**
- Pooled human intro rate: **9.14%**
- Pooled OR (AI vs human): **2.06** [95% CI 1.76-2.41], p = 3.8×10⁻¹⁷
- Python ecosystem: AI 23.33% vs Human 8.34%, OR 3.34
- TypeScript: AI 17.79% vs Human 9.60%, OR 2.04
- Severity: 36.8% of AI-introduced vulns are CRITICAL vs 20.5% for human
- AI PRs receive **more** reviews and **take longer** to merge — the gap persists despite scrutiny

Direction should replicate but confidence intervals will be **wider** than the paper's because our time window is a snapshot of AIDev as of today, not the exact window Li et al. published.

---

## 12. Caveats

- **Observational, not causal**: AI vs. human assignment is not randomized. We report associations.
- **C# excluded from vulnerability analysis**: OSV needs pinned lockfiles. NuGet's `packages.lock.json` is opt-in and rare — only ~3% of C# PRs are scannable. C# rows are retained in the raw dataset for transparency.
- **Go stratum underpowered**: ~4 positive AI cases. Reported as inconclusive, not reversed.
- **Semgrep ruleset variance**: `--config auto` pulls community rules whose set changes over time. Not a problem for *relative* AI vs. human comparison (both scanned the same way). Only in the original notebook — Fast notebook skips Semgrep entirely.
- **Present-day vulnerability status**: We measure vulnerabilities as *currently known*, not *as of the PR's merge date*. This measures realized risk exposure — a design choice stated explicitly.

---

## References

- **AIDev dataset**: Li et al. — https://huggingface.co/datasets/hao-li/AIDev
- **OSV-Scanner**: Google — https://github.com/google/osv-scanner
- **OSV.dev vulnerability database**: https://osv.dev
- **Semgrep**: https://semgrep.dev
- **GitHub GraphQL API**: https://docs.github.com/en/graphql
