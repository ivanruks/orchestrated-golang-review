---
name: golang-review-workflow
description: 5-phase orchestration workflow for multi-agent Go code review
---

# Golang Review Workflow

## Prerequisites

Before executing this workflow, the orchestrator (SKILL.md) must have resolved:
- `{{project_id}}` — GitLab project path (e.g., `group/subgroup/project`)
- `{{merge_request_iid}}` — MR numeric ID
- `{{selected_agents}}` — list of agents to run (default: all 7)
- `{{review_dir}}` — path to review working directory (e.g., `docs/review/2026-04-03T14-30_mr-456/`)

## Constants

```
WAVE_1_AGENTS = [correctness, concurrency, conventions, performance, security]
WAVE_2_AGENTS = [consistency, transactions]

AGENT_PREFIXES = {
  correctness: "CORR",
  concurrency: "CONC",
  conventions: "CONV",
  performance: "PERF",
  security: "SEC",
  consistency: "CONS",
  transactions: "TXN"
}

EXCLUDED_FILE_PATTERNS = ["^vendor/", "_mock\\.go$", "\\.pb\\.go$", "_generated\\.go$", "^testdata/", "\\.gen\\.go$"]
MAX_FILES_TO_FETCH = 20
```

---

## Phase 1: Fetch

Goal: Download MR data from GitLab and save to `{{review_dir}}`.

### Step 1.1 — Create review directory

```
mkdir -p {{review_dir}}/diffs
mkdir -p {{review_dir}}/files
mkdir -p {{review_dir}}/reports
```

**Critical:** All files created in this directory MUST persist after the review completes. Use the Write tool (not temporary variables) to save every file to disk. Sub-agents must also write their reports to disk, not just return them in-memory.

### Step 1.2 — Get MR metadata

```
CallMcpTool(server: "user-gitlab", toolName: "get_merge_request",
  arguments: {project_id: "{{project_id}}", merge_request_iid: {{merge_request_iid}}})
```

Save to `{{review_dir}}/metadata.json`:
```json
{
  "project_id": "{{project_id}}",
  "merge_request_iid": {{merge_request_iid}},
  "title": "<from response>",
  "description": "<from response>",
  "author": "<from response: author.username>",
  "source_branch": "<from response>",
  "target_branch": "<from response>",
  "url": "<original MR URL>",
  "fetched_at": "<current ISO timestamp>",
  "existing_discussions": []
}
```

### Step 1.3 — Get diffs

```
CallMcpTool(server: "user-gitlab", toolName: "get_merge_request_diffs",
  arguments: {project_id: "{{project_id}}", merge_request_iid: {{merge_request_iid}},
    excluded_file_patterns: EXCLUDED_FILE_PATTERNS})
```

For each diff entry where `new_path` ends with `.go`:
- Save diff content to `{{review_dir}}/diffs/<path-with-dashes>.diff`
  (replace `/` with `-` in path, e.g., `internal/service/user.go` → `internal-service-user.go.diff`)
- Track: file path, lines added, lines removed

### Step 1.4 — Get full file contents

For each `.go` file from step 1.3, ordered by diff size (largest first), up to MAX_FILES_TO_FETCH:

```
CallMcpTool(server: "user-gitlab", toolName: "get_file_contents",
  arguments: {project_id: "{{project_id}}", file_path: "<file path>", ref: "{{source_branch}}"})
```

Save to `{{review_dir}}/files/<original path>` preserving directory structure.

### Step 1.5 — Get existing discussions

```
CallMcpTool(server: "user-gitlab", toolName: "mr_discussions",
  arguments: {project_id: "{{project_id}}", merge_request_iid: {{merge_request_iid}}})
```

Update `metadata.json` field `existing_discussions` with the response. Sub-agents must NOT duplicate these.

### Step 1.6 — Report fetch results

Output to user:
```
Fetched MR !{{merge_request_iid}}: {{title}}
  Author: {{author}} · {{source_branch}} → {{target_branch}}
  Files: {{N}} .go files ({{lines_added}}+ / {{lines_removed}}-)
  Discussions: {{N}} existing
  Review dir: {{review_dir}}
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

{contents of agents/{agent_name}.md}

## Input

Review directory: {{review_dir}}

Diffs are in: {{review_dir}}/diffs/
Full files are in: {{review_dir}}/files/
MR metadata (including existing discussions to NOT duplicate): {{review_dir}}/metadata.json

## Context Rules

{contents of references/context-rules/{agent_name}.md}

## Output Schema

{contents of references/agent-output-schema.json}

## Instructions

1. Read metadata.json to understand the MR context and existing discussions.
2. Read each .diff file in diffs/ directory.
3. For each diff, read the corresponding full file from files/ directory.
4. Apply your checklist to every file.
5. When your context-rules trigger, load additional files:
   - First check files/ directory
   - If not there, use GitLab MCP (server: "user-gitlab", project: "{{project_id}}", ref: "{{source_branch}}")
   - Load references/operations.md if you need to find the right MCP operation
6. Do NOT duplicate findings from existing_discussions in metadata.json.
7. **IMPORTANT: Write your JSON report to disk** using the Write tool at {{review_dir}}/reports/{agent_name}.json
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

The following agents have already completed their analysis. Their reports are in {{review_dir}}/reports/.
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

Read every `{{review_dir}}/reports/*.json` file.

### Step 4.2 — Deduplicate

For each pair of findings from different agents:
- If `file` AND `line` match (same location):
  - Keep the finding from the more specialized agent:
    - concurrency > correctness (for race conditions)
    - transactions > correctness (for resource leaks in tx scope)
    - security > correctness (for injection issues)
    - consistency > conventions (for interface contract issues)
  - If neither is more specialized, keep both (different perspectives are valuable)

### Step 4.3 — Sort and group

Group findings:
1. First level: severity (critical → major → minor)
2. Second level: agent category (in order: correctness, concurrency, conventions, consistency, transactions, performance, security)

### Step 4.4 — Compute statistics

```json
{
  "files_checked": <union of files_checked across all agents>,
  "lines_added": <from Phase 1>,
  "lines_removed": <from Phase 1>,
  "critical_count": <count>,
  "major_count": <count>,
  "minor_count": <count>,
  "verdict": "<REJECT if critical > 0, REQUEST CHANGES if major > 0, LGTM otherwise>"
}
```

### Step 4.5 — Merge positive findings

Collect all `positive` arrays from all agents. Deduplicate similar observations. Keep as bullet list.

---

## Phase 5: Report

Goal: Render the final markdown report and present to user.

### Step 5.1 — Load template

Read `references/report-format.md`.

### Step 5.2 — Render

Fill the template with:
- MR metadata from `metadata.json`
- Statistics from Phase 4
- Grouped findings from Phase 4
- Merged positive findings
- Key production risks (summarize critical + major findings impact)

If `--only` was used, add a note in Summary: "Partial review: only {agents} were run."

### Step 5.3 — Save

Write to `{{review_dir}}/final-report.md`.

### Step 5.4 — Output

Display the full final report to the user.

---

## Data Retention

**Do NOT delete any files from `{{review_dir}}` after the review.**

All data must be preserved for future reference:
- `metadata.json` — MR context
- `diffs/` — original diffs for re-analysis
- `files/` — full source files as they were at review time
- `reports/` — individual agent JSON reports for debugging/comparison
- `final-report.md` — the rendered report

This allows the user to:
- Re-run individual agents on the same data (`--only` with existing review dir)
- Compare findings across multiple reviews of the same MR
- Debug agent behavior by inspecting their JSON reports
- Reference the exact code state at review time

**Important for sub-agents:** When writing reports to `reports/`, use the Write tool to create the file. Do NOT use temporary storage — the JSON report must persist on disk at `{{review_dir}}/reports/{agent_name}.json`.

---

## Error Handling

- If a sub-agent fails (timeout, error), log the failure and continue with remaining agents.
  Add a note to the final report: "Agent {name} failed: {reason}. Its category was not reviewed."
- If GitLab MCP is unavailable during Phase 1, stop and report the error to the user.
- If no `.go` files in the diff, report "No Go files changed in this MR" and stop.
