---
name: go-review-workflow
description: 5-phase orchestration workflow for GitLab MR Go code review
---

# Go Review Workflow — GitLab MR

## Prerequisites

Orchestrator (SKILL.md) must have resolved:
- `{{project_id}}` — GitLab project path
- `{{merge_request_iid}}` — MR numeric ID
- `{{selected_agents}}` — agents to run (default: all 9)
- `{{discussions_enabled}}` — load MR discussions (default: false)
- `{{additional_context}}` — free text from user (may be empty)
- `{{tmp_dir}}` — temp dir (e.g., `/tmp/go-review/2026-04-03T14-30_mr-456/`)
- `{{output_dir}}` — persistent dir (e.g., `docs/review/2026-04-03T14-30_mr-456/`)

## Constants

```
WAVE_1_AGENTS = [correctness, concurrency, conventions, style, performance, security, tests]
WAVE_2_AGENTS = [consistency, transactions]

AGENT_PREFIXES = {
  correctness: "CORR",
  concurrency: "CONC",
  conventions: "CONV",
  style: "STYL",
  performance: "PERF",
  security: "SEC",
  tests: "TEST",
  consistency: "CONS",
  transactions: "TXN"
}

EXCLUDED_FILE_PATTERNS = ["^vendor/", "_mock\\.go$", "\\.pb\\.go$", "_generated\\.go$", "^testdata/", "\\.gen\\.go$"]
MAX_FILES_TO_FETCH = 20
```

---

## Phase 1: Fetch from GitLab

Save MR data to `{{tmp_dir}}`.

### 1.1 — Create dirs

```bash
mkdir -p {{tmp_dir}}/diffs
mkdir -p {{tmp_dir}}/files
mkdir -p {{output_dir}}/reports
```

### 1.2 — MR metadata

```
CallMcpTool(server: "user-gitlab", toolName: "get_merge_request",
  arguments: {project_id: "{{project_id}}", merge_request_iid: {{merge_request_iid}}})
```

Extract: title, description, author.username, source_branch, target_branch.

### 1.3 — Discussions (optional)

If `{{discussions_enabled}}` true:
```
CallMcpTool(server: "user-gitlab", toolName: "mr_discussions",
  arguments: {project_id: "{{project_id}}", merge_request_iid: {{merge_request_iid}}})
```
Save as `existing_discussions`. If false: `existing_discussions = []`.

### 1.4 — Diffs

```
CallMcpTool(server: "user-gitlab", toolName: "get_merge_request_diffs",
  arguments: {project_id: "{{project_id}}", merge_request_iid: {{merge_request_iid}}})
```

For each diff where `new_path` ends `.go` and NOT matching EXCLUDED_FILE_PATTERNS:
- Save to `{{tmp_dir}}/diffs/<path-with-dashes>.diff` (replace `/` with `-`)
- Track: file path, lines added/removed

No `.go` files → stop: "No Go files changed in this MR."

### 1.5 — Full file contents

Per `.go` file from 1.4, ordered by diff size (largest first), up to MAX_FILES_TO_FETCH:

```
CallMcpTool(server: "user-gitlab", toolName: "get_file_contents",
  arguments: {project_id: "{{project_id}}", file_path: "<file path>", ref: "{{source_branch}}"})
```

Save to `{{tmp_dir}}/files/<original path>` preserving dir structure.

### 1.6 — metadata.json

Save to `{{tmp_dir}}/metadata.json`:
```json
{
  "title": "MR !{{merge_request_iid}}: <title from response>",
  "author": "<author.username from response>",
  "source_branch": "<source_branch from response>",
  "target_branch": "<target_branch from response>",
  "description": "<description from response>",
  "url": "<original MR URL>",
  "fetched_at": "<current ISO timestamp>",
  "existing_discussions": <discussions array or []>,
  "additional_context": "{{additional_context}}"
}
```

### 1.7 — Report

```
Fetched MR !{{merge_request_iid}}: <title>
  Author: <author> · <source_branch> → <target_branch>
  Files: <N> .go files (<lines_added>+ / <lines_removed>-)
  Discussions: <N> existing (or "disabled")
  Agents: {{selected_agents}}
  Launching Wave 1...
```

---

## Phase 2: Wave 1

Run Wave 1 agents parallel. Each analyzes diffs + full files, writes JSON report.

### For each agent in `WAVE_1_AGENTS ∩ {{selected_agents}}`:

Launch all as **parallel Agent** (single message).

Agent prompt:

```
You are the {agent_name} review agent.

{contents of go-review-refs/agents/{agent_name}.md}

## Input

Metadata (including existing discussions — do NOT duplicate): {{tmp_dir}}/metadata.json
Diffs: {{tmp_dir}}/diffs/
Output dir: {{output_dir}}/reports/

## File Access

Full files in {{tmp_dir}}/files/.
If file not there, fetch via GitLab MCP:
  CallMcpTool(server: "user-gitlab", toolName: "get_file_contents",
    arguments: {project_id: "{{project_id}}", file_path: "<path>", ref: "{{source_branch}}"})
To search repo:
  CallMcpTool(server: "user-gitlab", toolName: "get_repository_tree",
    arguments: {project_id: "{{project_id}}", ref: "{{source_branch}}", path: "<dir>", recursive: true})
Load go-review-refs/operations.md for other MCP ops.

## Context Rules

{contents of go-review-refs/context-rules/{agent_name}.md}

## Task Context (from author)

{{additional_context}}

## Output Schema

{contents of go-review-refs/agent-output-schema.json}

## Instructions

1. Read metadata.json for MR context + existing discussions.
2. Read each .diff in diffs/.
3. Read corresponding full file from files/.
4. Before checklist: summarize what changed, why, main execution path affected.
5. Apply checklist to every file.
6. Context-rules trigger → load additional files via File Access.
7. Don't duplicate findings from existing_discussions.
8. **Write JSON report to disk** via Write tool at {{output_dir}}/reports/{agent_name}.json — MUST persist on filesystem.
9. Return JSON content too (orchestrator uses immediately).
```

Wait for ALL Wave 1 before Phase 3.

---

## Phase 3: Wave 2

Run Wave 2 agents parallel. Get Wave 1 findings as extra context.

### For each agent in `WAVE_2_AGENTS ∩ {{selected_agents}}`:

Launch as **parallel Agent**. Same prompt as Phase 2, plus after Input:

```
## Wave 1 Findings

Reports in {{output_dir}}/reports/. Read before starting:
- Avoid duplicating already-reported findings
- Use their context (e.g., correctness resource leak → transaction issue)

Available Wave 1 reports:
{list of reports/*.json files that exist}
```

Wait for ALL Wave 2 before Phase 4.

---

## Phase 4: Merge

Combine all agent JSON reports → deduplicated, sorted findings.

### 4.1 — Load all reports

Read every `{{output_dir}}/reports/*.json`.

### 4.2 — Deduplicate

Same `file` AND `line` from different agents → keep more specialized:
- concurrency > correctness (races)
- transactions > correctness (resource leaks in tx)
- security > correctness (injection)
- consistency > conventions (interface contracts)
- conventions > correctness (error handling style)
- conventions > style (naming/doc — conventions owns API-facing)
- correctness > style (pointer-to-interface foot-guns)
- concurrency > performance (lock contention → deadlock risk)
- concurrency > correctness (ctx cancellation → goroutine lifecycle)
- tests > correctness (missing coverage — tests more specific)
- tests > conventions (test quality — tests owns patterns)
- Neither specialized → keep both

#### Dedup Rules

- **Strip:** Same file:line from lower-priority agent → remove
- **Preserve:** Different failure modes on same line → keep both
- **Positives:** Dedup semantically similar; keep more specific
- **Open questions:** Union all; dedup by meaning; cap 10

### 4.3 — Sort and group

1. Severity: critical → major → minor
2. Agent category: correctness, concurrency, conventions, style, tests, consistency, transactions, performance, security

### 4.4 — Statistics

```json
{
  "files_checked": "<union across agents>",
  "lines_added": "<Phase 1>",
  "lines_removed": "<Phase 1>",
  "critical_count": "<count>",
  "major_count": "<count>",
  "minor_count": "<count>",
  "verdict": "<REJECT if critical > 0, REQUEST CHANGES if major > 0, LGTM otherwise>"
}
```

### 4.5 — Merge positives

Collect `positive` arrays. Dedup semantically; keep specific phrasing.

### 4.6 — Merge open questions

Collect `open_questions`. Dedup by meaning. Cap 10. Prefix with agent name.

---

## Phase 5: Report

### 5.1 — Load template

Read `go-review-refs/report-format.md`.

### 5.2 — Render

Fill with: metadata, statistics, grouped findings, positives, key production risks.

**Snippets:** Follow `## Orchestrator contract` in `report-format.md`. Per finding: `code_snippet_unavailable: true` → render **Code snippet:** not applicable — `code_absence_note` (no fences, don't reconstruct). Otherwise → Before/After `go` fences from `code_before`/`code_after`; if empty → **reconstruct** from `{{tmp_dir}}/diffs/` + `{{tmp_dir}}/files/` at `file`/`line`. No mode A with empty fences. No mode B without `code_absence_note`.

`--only` → add "Partial review: only {agents} were run." in Summary.

### 5.3 — Save

Write `{{output_dir}}/final-report.md`.

### 5.4 — Output

Display full report to user.

---

## Error Handling

- GitLab MCP unavailable → stop, report error
- No `.go` in diff → "No Go files changed in this MR", stop
- Sub-agent fails → log, continue. Note: "Agent {name} failed: {reason}. Category not reviewed."
- All Wave 1 fail → stop. Don't launch Wave 2
- Invalid JSON from agent → skip, note in report

---

## Data Retention

**Don't delete `{{output_dir}}` files.** Reports preserved: `reports/` (agent JSONs) + `final-report.md`. Temp `{{tmp_dir}}` cleaned by OS.

**Sub-agents:** Write reports via Write tool to `{{output_dir}}/reports/{agent_name}.json`. Must persist on disk.
