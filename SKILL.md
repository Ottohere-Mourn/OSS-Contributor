---
name: /oss-contribute
description: Discover, evaluate, and execute high-quality open-source contribution opportunities in a target domain. Search GitHub for repos, scan good-first-issues, generate human-reviewable reports, create PRs with worktree isolation, and follow up on submitted PRs. Use when the user wants to find OSS projects to contribute to, discover PR opportunities, do a full contribution pipeline from domain keywords to submitted PR, or check on previously submitted PRs.
argument-hint: "<domain> [--topk N] [--type docs,test,bugfix] [--language Python] [--min-stars 1000] | followup --session <id>"
allowed-tools: Read, Write, Edit, Bash, Agent, Grep, Glob, WebSearch, AskUserQuestion
user-invocable: true
disable-model-invocation: false
---

# OSS-Contributor

Full pipeline: domain keywords → repo discovery → opportunity evaluation → human review → PR execution → contribution report.
Also supports: `/oss-contribute followup --session <id>` to check and update previously submitted PRs.

---

## Command Routing

Check `$ARGUMENTS` first. If it starts with `followup`, jump to **Phase 6: Followup**. Otherwise, proceed with the full pipeline below.

---

## Phase 0: Setup

### 0.1 Prerequisite Check

Run these checks and abort with a clear message if any fails:

```bash
gh auth status 2>&1
```

If not authenticated, output:
```
❌ GitHub CLI (gh) is not authenticated.
   Run: gh auth login
   Then retry /oss-contribute.
```

```bash
git --version
```

### 0.2 Rate Limit Check

Before spawning any sub-agents, check the current GitHub API rate limit:

```bash
gh api /rate_limit --jq '.rate | "\(.remaining)/\(.limit) reset at \(.reset)"'
```

Determine strategy based on remaining quota:

| Remaining | Strategy |
|-----------|----------|
| > 1000 | Full parallelism: Phase 1 spawns all dimension agents, Phase 2 parallel per repo |
| 200–1000 | Reduced parallelism: Phase 1 limited to 2 agents, Phase 2 sequential |
| 50–200 | Sequential: all phases serial, 3s delay between API calls |
| < 50 | Warn user: `⚠️ GitHub API quota low ({remaining}/{limit}). Reset at {reset_time}. Consider waiting or using a different token.` Then abort. |

All `gh api` calls throughout the pipeline should use `--cache 30s` to reduce redundant quota consumption.

### 0.3 Load Configuration

Read `config.yaml` alongside this SKILL.md if it exists. Fall back to these defaults for any missing keys:

```
topk: 5
min_stars: 500
max_stars: 0
languages: []
types: ["bugfix", "docs", "test"]
exclude_repos: []
```

### 0.4 Parse Arguments

From `$ARGUMENTS`, extract:

| Argument | Default | Description |
|----------|---------|-------------|
| `<domain>` | **required** | Domain keyword(s), e.g. "LLM inference", "vector database" |
| `--topk` | config or 5 | Number of repos to evaluate deeply |
| `--type` | config or "bugfix,docs,test" | Contribution types: docs, test, bugfix, feature |
| `--language` | config or none | Filter by programming language |
| `--min-stars` | config or 500 | Minimum GitHub stars |
| `--max-stars` | config or 0 (no limit) | Maximum GitHub stars — filters out mega-repos |
| `--exclude` | config or none | Comma-separated repos/orgs to exclude |

If `<domain>` is missing, output the usage and abort:
```
Usage: /oss-contribute <domain> [options]
       /oss-contribute followup --session <session_id>

Examples:
  /oss-contribute "LLM inference optimization" --max-stars 8000
  /oss-contribute "rust web framework" --topk 3 --type bugfix
  /oss-contribute "vector database" --language Python --min-stars 1000
  /oss-contribute followup --session 20260530-143000

Options:
  --topk N        Number of repos to evaluate (default: 5)
  --type TYPES    Contribution types: docs,test,bugfix,feature (default: bugfix,docs,test)
  --language LANG Filter by programming language
  --min-stars N   Minimum GitHub stars (default: 500)
  --max-stars N   Maximum GitHub stars — filter out mega-repos (default: no limit)
  --exclude REPOS Comma-separated repos/orgs to exclude
```

### 0.5 Create Working Directory

```bash
session_id=$(date +%Y%m%d-%H%M%S)
workdir=".oss-contributor/$session_id"
mkdir -p "$workdir"
```

All intermediate artifacts (repos.json, opportunities.md, selection.json, pr-links.json, contribution-report.md) go into `$workdir`.

---

## Phase 1: Discover

**Goal**: Find candidate repos matching the domain via multi-dimensional divergent search, confirmed by the user before execution.

### 1.1 Propose Search Dimensions (Human-in-the-Loop)

From the user's `<domain>`, generate search dimensions across these categories. Think like a researcher mapping out a field:

- **Core**: the domain itself — exact and near-exact matches
- **Sub-areas (IMPORTANT)**: break the domain into 3-4 concrete sub-topics. Don't be vague — name real techniques, algorithms, and problem classes. Examples:
  - "embodied intelligence" → VLA (vision-language-action), world-action models, sim-to-real transfer, dexterous manipulation, imitation learning for robotics, embodied navigation
  - "LLM inference" → KV-cache optimization, speculative decoding, model quantization, continuous batching, tensor parallelism
  - "agent architecture" → memory-augmented agents, hierarchical planning, tool-use frameworks, multi-agent coordination, ReAct pattern
- **Upstream**: technologies the domain depends on (e.g., simulators, physics engines, perception backbones)
- **Downstream**: applications built on top of the domain (e.g., robot platforms, industrial automation, home robotics SDKs)
- **Ecosystem**: awesome lists, topic hubs, benchmark suites, community-curated collections

Present the proposed dimensions to the user:

```
🔍 Proposed search dimensions for "{domain}":

  [1] Core:        "{core_keywords}"
  [2] Sub-areas:   "{sub_area_1}", "{sub_area_2}", "{sub_area_3}", ...
  [3] Upstream:    "{upstream_keywords}"
  [4] Downstream:  "{downstream_keywords}"
  [5] Ecosystem:   "{ecosystem_keywords}"

  Each dimension will be searched independently by a dedicated agent.
  Sub-areas are where the best contribution opportunities usually hide.

  Proceed with all, or adjust?
```

Use `AskUserQuestion` to let the user approve or adjust. Options:

```
question: "Proceed with these search dimensions? You can remove irrelevant ones."
header: "Search dimensions"
multiSelect: true
options:
  - label: "[1] Core: {core_keywords}"
    description: "Direct matches in the domain"
  - label: "[2] Sub-areas: {sub_areas_short}"
    description: "Specific sub-topics — usually where the best opportunities hide. Recommended to keep."
  - label: "[3] Upstream: {upstream_keywords}"
    description: "Technologies and libraries the domain depends on"
  - label: "[4] Downstream: {downstream_keywords}"
    description: "Applications and frameworks built on the domain"
  - label: "[5] Ecosystem: {ecosystem_keywords}"
    description: "Awesome lists, topic pages, community collections"
```

The user can deselect dimensions they find irrelevant. Proceed only with approved dimensions.

### 1.2 Parallel Search (Explorer Agents)

Spawn one **Explorer** sub-agent per approved search dimension. Each agent searches deeply within its one dimension — this yields more diverse and higher-quality results than having a single agent run the same keyword repeatedly.

Read `prompts/discover.md` for each agent's instructions. Interpolate variables:

- `{dimension_label}`: the dimension name (e.g., "Core", "Upstream")
- `{dimension_keywords}`: the search query for this dimension
- `{min_stars}`, `{max_stars}`, `{language}`, `{exclude_repos}`: from config/arguments

Each agent searches 2-3 different query strategies within its dimension and returns structured JSON.

### 1.3 Merge, Cross-Validate, and Rank

After all Explorer agents return:

1. Merge all `candidates` arrays
2. Deduplicate by `full_name`
3. **Cross-validation bonus**: repos that appear in results from 2+ different dimensions get +10 to their `contribution_friendliness` score (they're being discovered from multiple angles, suggesting genuine relevance)
4. Re-sort by `contribution_friendliness` descending
5. Take top `--topk` entries

Save to `$workdir/repos.json`.

### 1.4 Present Discovery Summary

Output a table to the user:

```
🔍 Discovered {total} repos across {N} dimensions, evaluated top {topk}:

| # | Repo | Stars | Lang | Good First Issues | Friendliness | Cross-Hits |
|---|------|-------|------|-------------------|-------------|------------|
| 1 | owner/repo-a | 12.3k | Python | 5 | 85 | 3 dimensions |
| 2 | owner/repo-b | 8.1k | Rust | 3 | 72 | 2 dimensions |
| 3 | owner/repo-c | 5.6k | Python | 12 | 68 | 1 dimension |
| ... | ... | ... | ... | ... | ... | ... |

Proceeding to deep evaluation...
```

---

## Phase 2: Evaluate

**Goal**: For each top-K repo, perform a deep code-level evaluation of contribution opportunities.

### 2.1 Parallel Deep Evaluation (Explorer Agents)

Spawn one Explorer agent per repo from the top-K list. Adjust parallelism based on Phase 0.2 rate limit strategy.

Read `prompts/evaluate.md` for each agent's instructions. Interpolate variables:

- `{full_name}`: the repo to analyze
- `{types}`: from config/arguments
- `{languages}`: from config/arguments

Each agent will:
1. Read contribution documentation
2. Scan for actionable issues by type
3. **Read actual code** (shallow clone, sample 2-3 recent PR diffs and target files) to verify difficulty assessments
4. Assess PR merge patterns and compute merge_rate
5. Identify red flags
6. Return structured JSON with up to 5 ranked opportunities

### 2.2 Merge Evaluations

Collect all agent results. Sort opportunities across all repos by:
1. Merge probability (High > Medium > Low)
2. Impact (High > Medium > Low)
3. Difficulty (Trivial > Easy > Medium > Hard — easier first for confidence)

Save to `$workdir/opportunities.md` as a formatted markdown report.

---

## Phase 3: Review (Human Gate)

### 3.1 Present Opportunities

Output a formatted summary to the user:

```
📋 Contribution Opportunities Report
════════════════════════════════════════

Repo 1: owner/repo-a (⭐ 12.3k, PR merge rate: 78%)

  [1] #1234 — Fix typo in quantization docstring
      Type: docs | Difficulty: Trivial | Impact: Low | Time: 1-2h
      → One-line fix, active maintainer merges doc PRs quickly

  [2] #1456 — Add missing parameter docs in API reference
      Type: docs | Difficulty: Easy | Impact: Medium | Time: 2-4h
      → 3 users asked about this, clear acceptance criteria

  [3] #1890 — Fix edge case in batched inference with empty input
      Type: bugfix | Difficulty: Medium | Impact: High | Time: 4-8h
      → Reproduced by 2 users, maintainer confirmed bug

Repo 2: owner/repo-b (⭐ 8.1k, PR merge rate: 62%)

  [4] #567 — Add unit tests for attention mask utilities
      Type: test | Difficulty: Easy | Impact: Medium | Time: 2-4h
      → Core module with 40% coverage gap, clear test surface

  [5] #892 — Fix memory leak warning in worker process shutdown
      Type: bugfix | Difficulty: Medium | Impact: Medium | Time: 4-8h
      → Long-standing issue with debugging hints from maintainer
```

### 3.2 Interactive Selection

Use `AskUserQuestion` to let the user choose:

```
question: "Which opportunities do you want to pursue?"
header: "Select targets"
multiSelect: true
options:
  - label: "#1 — owner/repo-a: Fix docstring typo (Trivial, 1-2h)"
    description: "Recommended: quick win, very high merge probability, builds confidence"
  - label: "#2 — owner/repo-a: Add missing API docs (Easy, 2-4h)"
    description: "Moderate effort, clear acceptance criteria, medium visibility"
  - label: "#3 — owner/repo-a: Fix batched inference edge case (Medium, 4-8h)"
    description: "Higher risk but high impact, maintainer confirmed"
  - label: "#4 — owner/repo-b: Add attention mask tests (Easy, 2-4h)"
    description: "Test contribution, low risk of rejection"
```

### 3.3 Save Selection

Save selected targets to `$workdir/selection.json`. Proceed only with confirmed selections.

**If user selects nothing**: ask if they want to broaden search or adjust parameters, then re-run from Phase 1 with adjusted inputs.

---

## Phase 4: Execute

**Goal**: For each selected opportunity, implement the fix in an isolated worktree, test it, and submit a PR. Failed targets are retried with fallback opportunities from the same repo.

### 4.1 First Attempt (Parallel Execution)

For each selected target, spawn a **General-purpose** agent with `isolation: "worktree"`. Read `prompts/execute.md` for each agent's instructions. Interpolate:

- `{full_name}`, `{issue_number}`, `{issue_title}`, `{url}`, `{type}`

**Important**: For the first agent on a given repo, no special context. For subsequent agents on the SAME repo (see retry), inject `{shared_lessons}` from prior attempts.

### 4.2 Collect Results and Handle Failures

After all first-attempt agents complete, collect their results.

If any agent returned `status: "failed"`:

1. **Extract lessons**: collect `lessons_learned` from all failed agents for the same repo
2. **Find fallback**: look in `$workdir/opportunities.md` for another open opportunity in the same repo that was NOT already attempted
3. **Spawn retry agent**: same as 4.1, but with `{shared_lessons}` injected into the prompt:
   ```
   ## Prior Attempt on This Repo

   A previous agent attempted a different issue on this repo and reported:
   {lessons_learned}

   Use this information to avoid repeating the same mistakes.
   ```
4. **Retry limits**:
   - Max 2 retries per target repo
   - Max 3 cross-repo retries total (across all repos)
   - If retries exhausted, log the failure and continue

### 4.3 Collect All Results

After retries are exhausted, collect all results (successes + final failures) into `$workdir/pr-links.json`.

### 4.4 Progress Update

Output a live summary:

```
🚀 Executing PRs...

  [✓] owner/repo-a #1234 — https://github.com/owner/repo-a/pull/5678
  [↻] owner/repo-a #1890 — failed, retrying with issue #2001...
  [✓] owner/repo-b #567  — https://github.com/owner/repo-b/pull/901

All PRs processed! 2 submitted, 0 failed after retry.
```

---

## Phase 5: Report

**Goal**: Generate a comprehensive, interview-ready contribution report.

Read all artifacts from `$workdir/`:

- `repos.json` — discovery results
- `opportunities.md` — evaluation report
- `selection.json` — human selections
- `pr-links.json` — execution results (including retry history)

Synthesize into `$workdir/contribution-report.md` using the template from `templates/report-template.md`. Read `prompts/report.md` for detailed instructions.

### Final Output

```
✅ OSS-Contributor run complete!

   Domain: {domain}
   Repos evaluated: {total}
   PRs submitted: {count}
   Retries used: {N}

   📄 Full report: $workdir/contribution-report.md

   PR Links:
   - {pr_url_1}
   - {pr_url_2}

   Next steps:
   1. Review the PRs on GitHub — respond to maintainer comments
   2. Check back later with: /oss-contribute followup --session {session_id}
   3. Share contribution-report.md with your network / in interviews
```

---

## Phase 6: Followup

**Trigger**: `/oss-contribute followup --session {session_id}`

This phase checks the status of previously submitted PRs and handles review feedback.

### 6.1 Load Session

Read `$workdir/pr-links.json` from `.oss-contributor/{session_id}/`.

If the file doesn't exist:
```
❌ Session {session_id} not found.
   Check .oss-contributor/ for available sessions:
   {list of existing session directories}
```

### 6.2 Check and Handle Each PR

Read `prompts/followup.md` for the detailed followup agent instructions. Spawn one **General-purpose** agent for the session. The agent will:

1. Load all PRs from pr-links.json
2. Check each PR's status (merged / review-comments / no-response / closed)
3. Handle simple review feedback automatically (fix + push + reply)
4. Escalate significant rework requests to human
5. Generate a followup section in the report

**Critical rules (enforced in followup.md)**:
- NEVER comment on a PR unattended
- NEVER close a PR
- NEVER ping maintainers
- Simple "thanks, addressed!" replies are the only automatic comments

### 6.3 Present Followup Summary

After the agent completes, present a summary:

```
📬 Followup complete — Session {session_id}

   PRs checked: 3
   ✅ Merged: 1
   🔄 Updated (feedback addressed): 1
   ⏳ Waiting (no response yet): 1

   Items needing your attention:
   - owner/repo #N: Maintainer requests architectural change. See report for details.

   📄 Updated report: .oss-contributor/{session_id}/contribution-report.md
```

---

## Sub-Agent Reference

| Phase | File | Agent Type | Count |
|-------|------|-----------|-------|
| 1. Discover | `prompts/discover.md` | Explorer | 3-5 parallel (one per approved dimension) |
| 2. Evaluate | `prompts/evaluate.md` | Explorer | topk parallel (rate-limit adjusted) |
| 4. Execute | `prompts/execute.md` | General + worktree | selected_count + retries |
| 6. Followup | `prompts/followup.md` | General | 1 per session |

When spawning agents, read the corresponding prompt file and interpolate `{variables}` with actual values. Each prompt file is self-contained — the agent needs no conversation context beyond the prompt.

## Error Recovery

- **Rate limit low (Phase 0)**: Warn user, abort, suggest retry after reset time.
- **Phase 1 fails entirely**: Retry with broader dimensions (remove language filter, lower min-stars). If still no results, suggest the user rephrase the domain.
- **Phase 2 fails for a repo**: Mark it with a red flag, continue with other repos. Don't block the pipeline.
- **Phase 4 fails for all targets**: Log all failures in pr-links.json. Report honestly in Phase 5. Failed attempts are still valuable learning data.
- **Phase 4 retry fails**: Move on — don't retry infinite times. The failure report is a useful artifact.
- **gh CLI rate limited mid-pipeline**: Pause, tell user remaining quota and reset time, ask whether to continue sequentially or abort.
