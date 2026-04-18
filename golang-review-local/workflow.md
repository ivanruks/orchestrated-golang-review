---
name: golang-review-local-workflow
description: 5-phase orchestration workflow for uncommitted Go code review
---

# Golang Review Workflow — Local Uncommitted

## Prerequisites

Before executing this workflow, the orchestrator (SKILL.md) must have resolved:
- `{{target_branch}}` — branch to compare against (e.g., `main`)
- `{{source_branch}}` — current branch name
- `{{selected_agents}}` — list of agents to run (default: all 8)
- `{{additional_context}}` — free text context from user (may be empty)
- `{{tmp_dir}}` — path to temporary working directory (e.g., `/tmp/golang-review/2026-04-03T14-30_local-feature-xyz/`)
- `{{output_dir}}` — path to persistent output directory (e.g., `docs/review/2026-04-03T14-30_local-feature-xyz/`)

## Constants

```
WAVE_1_AGENTS = [correctness, concurrency, conventions, performance, security, tests]
WAVE_2_AGENTS = [consistency, transactions]

AGENT_PREFIXES = {
  correctness: "CORR",
  concurrency: "CONC",
  conventions: "CONV",
  performance: "PERF",
  security: "SEC",
  tests: "TEST",
  consistency: "CONS",
  transactions: "TXN"
}

EXCLUDED_FILE_PATTERNS = ["^vendor/", "_mock\\.go$", "\\.pb\\.go$", "_generated\\.go$", "^testdata/", "\\.gen\\.go$"]
```

---

## Phase 1: Fetch Uncommitted Changes

Goal: Generate diffs of uncommitted (staged + unstaged, tracked) changes relative to `{{target_branch}}` and save to `{{tmp_dir}}`.

### Step 1.1 — Create directories

```bash
mkdir -p {{tmp_dir}}/diffs
mkdir -p {{output_dir}}/reports
```

### Step 1.2 — Determine repo root

```bash
repo_root=$(git rev-parse --show-toplevel)
```

### Step 1.3 — Get diffs

Get the combined diff of all tracked changes (both staged and unstaged) relative to target branch:

```bash
git diff {{target_branch}} -- '*.go'
```

This single command produces the combined diff of staged + unstaged changes vs the target branch. It includes all tracked .go files that differ from the target, whether those changes are staged, unstaged, or a mix.

From the output:
- Split into per-file diffs
- Exclude files matching EXCLUDED_FILE_PATTERNS
- Save each file's diff to `{{tmp_dir}}/diffs/<path-with-dashes>.diff`
  (replace `/` with `-` in path)
- Track: file path, lines added, lines removed

If no `.go` files in diff, stop and report: "No uncommitted Go file changes found relative to `{{target_branch}}`."

### Step 1.4 — Get author

```bash
git config user.name
```

### Step 1.5 — Write metadata.json

Save to `{{tmp_dir}}/metadata.json`:
```json
{
  "title": "Uncommitted changes vs {{target_branch}}",
  "author": "<git config user.name>",
  "source_branch": "{{source_branch}}",
  "target_branch": "{{target_branch}}",
  "description": "",
  "url": "",
  "fetched_at": "<current ISO timestamp>",
  "existing_discussions": [],
  "additional_context": "{{additional_context}}"
}
```

### Step 1.6 — Report fetch results

Output to user:
```
Review: uncommitted changes on {{source_branch}} vs {{target_branch}}
  Author: <author>
  Files: <N> .go files (<lines_added>+ / <lines_removed>-)
  Agents: {{selected_agents}}
  Launching Wave 1...
```

---

## Phase 2: Wave 1

Goal: Run Wave 1 agents in parallel. Each agent analyzes diffs + full files and writes a JSON report.

### For each agent in `WAVE_1_AGENTS ∩ {{selected_agents}}`:

Launch as a **parallel Agent** (use the Agent tool with all agents in a single message for true parallelism).

Each agent prompt is constructed as follows:

```
You are the {agent_name} review agent.

{contents of references/agents/{agent_name}.md}

## Input

Metadata: {{tmp_dir}}/metadata.json
Diffs: {{tmp_dir}}/diffs/
Output directory for your report: {{output_dir}}/reports/

## File Access

Read files directly from the repository at: {{repo_root}}/<file_path>
Use the Read tool with absolute paths. For example, to read internal/service/user.go:
  Read {{repo_root}}/internal/service/user.go
To search for files, use the Glob or Grep tools on {{repo_root}}.

## Context Rules

{contents of references/context-rules/{agent_name}.md}

## Task Context (provided by author)

{{additional_context}}

## Output Schema

{contents of references/agent-output-schema.json}

## Instructions

1. Read metadata.json to understand the review context.
2. Read each .diff file in diffs/ directory.
3. For each diff, read the corresponding full file using File Access instructions above.
4. Before applying your checklist, summarize internally: what changed, why, and what is the main execution path affected. This grounds your analysis.
5. Apply your checklist to every file.
6. When your context-rules trigger, load additional files using File Access instructions above.
7. **IMPORTANT: Write your JSON report to disk** using the Write tool at {{output_dir}}/reports/{agent_name}.json
   The file MUST persist on the filesystem after you finish. Do not just return the JSON — it must be saved to disk.
8. Return the JSON report content as well (for the orchestrator to use immediately).
```

Wait for ALL Wave 1 agents to complete before proceeding to Phase 3.

---

## Phase 3: Wave 2

Goal: Run Wave 2 agents in parallel. They receive Wave 1 findings as additional context.

### For each agent in `WAVE_2_AGENTS ∩ {{selected_agents}}`:

Launch as a **parallel Agent**. Same prompt structure as Phase 2, with this addition after the Input section:

```
## Wave 1 Findings

The following agents have already completed their analysis. Their reports are in {{output_dir}}/reports/.
Read these reports before starting your analysis — use them to:
- Avoid duplicating findings already reported
- Use their context to inform your analysis (e.g., a correctness resource leak may indicate a transaction issue)

Available Wave 1 reports:
{list of reports/*.json files that exist}
```

Wait for ALL Wave 2 agents to complete before proceeding to Phase 4.

---

## Phase 4: Merge

Goal: Combine all agent JSON reports into a single deduplicated, sorted findings list.

### Step 4.1 — Load all reports

Read every `{{output_dir}}/reports/*.json` file.

### Step 4.2 — Deduplicate

For each pair of findings from different agents:
- If `file` AND `line` match (same location):
  - Keep the finding from the more specialized agent:
    - concurrency > correctness (for race conditions)
    - transactions > correctness (for resource leaks in tx scope)
    - security > correctness (for injection issues)
    - consistency > conventions (for interface contract issues)
    - conventions > correctness (for error handling style — missing %w, error string format)
    - concurrency > performance (for lock contention — deadlock risk outweighs perf concern)
    - concurrency > correctness (for context cancellation leaks — goroutine lifecycle is root cause)
    - tests > correctness (for missing test coverage — tests agent is more specific about what's missing)
    - tests > conventions (for test quality — tests agent owns test patterns)
  - If neither is more specialized, keep both (different perspectives are valuable)

#### Dedup Rules (Strip / Preserve)

- **Strip:** Duplicate finding on same file:line from lower-priority agent — remove the lower-priority one.
- **Preserve:** Keep both findings if they describe different failure modes on the same line, even from different agents.
- **Positive merge:** Deduplicate semantically similar positives; keep the more specific phrasing.
- **Open questions merge:** Union all; deduplicate by meaning; cap at 10 total.

### Step 4.3 — Sort and group

Group findings:
1. First level: severity (critical → major → minor)
2. Second level: agent category (in order: correctness, concurrency, conventions, tests, consistency, transactions, performance, security)

### Step 4.4 — Compute statistics

```json
{
  "files_checked": "<union of files_checked across all agents>",
  "lines_added": "<from Phase 1>",
  "lines_removed": "<from Phase 1>",
  "critical_count": "<count>",
  "major_count": "<count>",
  "minor_count": "<count>",
  "verdict": "<REJECT if critical > 0, REQUEST CHANGES if major > 0, LGTM otherwise>"
}
```

### Step 4.5 — Merge positive findings

Collect all `positive` arrays from all agents. Deduplicate semantically similar observations; keep the more specific phrasing. Keep as bullet list.

### Step 4.6 — Merge open questions

Collect all `open_questions` arrays from all agents. Deduplicate by meaning. Cap at 10 total. Prefix each with the agent name.

---

## Phase 5: Report

Goal: Render the final markdown report and present to user.

### Step 5.1 — Load template

Read `references/report-format.md`.

### Step 5.2 — Render

Fill the template with:
- Metadata from `{{tmp_dir}}/metadata.json`
- Statistics from Phase 4
- Grouped findings from Phase 4
- Merged positive findings
- Key production risks (summarize critical + major findings impact)

If `--only` was used, add a note in Summary: "Partial review: only {agents} were run."

### Step 5.3 — Save

Write to `{{output_dir}}/final-report.md`.

### Step 5.4 — Output

Display the full final report to the user.

---

## Error Handling

- If target branch does not exist, stop and report the error to the user (should be caught by SKILL.md).
- If no `.go` files in the diff, report "No uncommitted Go file changes found relative to {{target_branch}}" and stop.
- If a sub-agent fails (timeout, error), log the failure and continue with remaining agents.
  Add a note to the final report: "Agent {name} failed: {reason}. Its category was not reviewed."
- If all Wave 1 agents fail, stop and report the error. Do not launch Wave 2.
- If a sub-agent returns invalid JSON, skip it and note in the report.

---

## Data Retention

**Do NOT delete any files from `{{output_dir}}` after the review.**

All reports must be preserved:
- `reports/` — individual agent JSON reports for debugging/comparison
- `final-report.md` — the rendered report

Temporary files in `{{tmp_dir}}` (metadata.json, diffs/) will be cleaned up by the OS.

**Important for sub-agents:** When writing reports to `reports/`, use the Write tool to create the file. Do NOT use temporary storage — the JSON report must persist on disk at `{{output_dir}}/reports/{agent_name}.json`.
