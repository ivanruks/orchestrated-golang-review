---
name: golang-review-workflow
description: 5-phase orchestration workflow for GitLab MR Go code review
---

# Golang Review Workflow — GitLab MR

## Prerequisites

Before executing this workflow, the orchestrator (SKILL.md) must have resolved:
- `{{project_id}}` — GitLab project path (e.g., `group/subgroup/project`)
- `{{merge_request_iid}}` — MR numeric ID
- `{{selected_agents}}` — list of agents to run (default: all 7)
- `{{discussions_enabled}}` — whether to load MR discussions (default: false)
- `{{additional_context}}` — free text context from user (may be empty)
- `{{tmp_dir}}` — path to temporary working directory (e.g., `/tmp/golang-review/2026-04-03T14-30_mr-456/`)
- `{{output_dir}}` — path to persistent output directory (e.g., `docs/review/2026-04-03T14-30_mr-456/`)

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
MAX_FILES_TO_FETCH = 20
```

---

## Phase 1: Fetch from GitLab

Goal: Download MR data from GitLab and save to `{{tmp_dir}}`.

### Step 1.1 — Create directories

```bash
mkdir -p {{tmp_dir}}/diffs
mkdir -p {{tmp_dir}}/files
mkdir -p {{output_dir}}/reports
```

### Step 1.2 — Get MR metadata

```
CallMcpTool(server: "user-gitlab", toolName: "get_merge_request",
  arguments: {project_id: "{{project_id}}", merge_request_iid: {{merge_request_iid}}})
```

Extract from response: title, description, author.username, source_branch, target_branch.

### Step 1.3 — Get discussions (optional)

If `{{discussions_enabled}}` is true:
```
CallMcpTool(server: "user-gitlab", toolName: "mr_discussions",
  arguments: {project_id: "{{project_id}}", merge_request_iid: {{merge_request_iid}}})
```
Save response as `existing_discussions`.

If `{{discussions_enabled}}` is false: set `existing_discussions = []`.

### Step 1.4 — Get diffs

```
CallMcpTool(server: "user-gitlab", toolName: "get_merge_request_diffs",
  arguments: {project_id: "{{project_id}}", merge_request_iid: {{merge_request_iid}}})
```

For each diff entry where `new_path` ends with `.go` and does NOT match any EXCLUDED_FILE_PATTERNS:
- Save diff content to `{{tmp_dir}}/diffs/<path-with-dashes>.diff`
  (replace `/` with `-` in path, e.g., `internal/service/user.go` → `internal-service-user.go.diff`)
- Track: file path, lines added, lines removed

If no `.go` files in diff, stop and report: "No Go files changed in this MR."

### Step 1.5 — Get full file contents

For each `.go` file from step 1.4, ordered by diff size (largest first), up to MAX_FILES_TO_FETCH:

```
CallMcpTool(server: "user-gitlab", toolName: "get_file_contents",
  arguments: {project_id: "{{project_id}}", file_path: "<file path>", ref: "{{source_branch}}"})
```

Save to `{{tmp_dir}}/files/<original path>` preserving directory structure.

### Step 1.6 — Write metadata.json

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

### Step 1.7 — Report fetch results

Output to user:
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

Goal: Run Wave 1 agents in parallel. Each agent analyzes diffs + full files and writes a JSON report.

### For each agent in `WAVE_1_AGENTS ∩ {{selected_agents}}`:

Launch as a **parallel Agent** (use the Agent tool with all agents in a single message for true parallelism).

Each agent prompt is constructed as follows:

```
You are the {agent_name} review agent.

{contents of references/agents/{agent_name}.md}

## Input

Metadata (including existing discussions to NOT duplicate): {{tmp_dir}}/metadata.json
Diffs: {{tmp_dir}}/diffs/
Output directory for your report: {{output_dir}}/reports/

## File Access

Full files are in {{tmp_dir}}/files/.
If a file you need is not there, fetch it via GitLab MCP:
  CallMcpTool(server: "user-gitlab", toolName: "get_file_contents",
    arguments: {project_id: "{{project_id}}", file_path: "<path>", ref: "{{source_branch}}"})
To search for files in the repository:
  CallMcpTool(server: "user-gitlab", toolName: "get_repository_tree",
    arguments: {project_id: "{{project_id}}", ref: "{{source_branch}}", path: "<dir>", recursive: true})
Load references/operations.md if you need to find other MCP operations.

## Context Rules

{contents of references/context-rules/{agent_name}.md}

## Task Context (provided by author)

{{additional_context}}

## Output Schema

{contents of references/agent-output-schema.json}

## Instructions

1. Read metadata.json to understand the MR context and existing discussions.
2. Read each .diff file in diffs/ directory.
3. For each diff, read the corresponding full file from files/ directory.
4. Before applying your checklist, summarize internally: what changed, why, and what is the main execution path affected. This grounds your analysis.
5. Apply your checklist to every file.
6. When your context-rules trigger, load additional files using File Access instructions above.
7. Do NOT duplicate findings from existing_discussions in metadata.json.
8. **IMPORTANT: Write your JSON report to disk** using the Write tool at {{output_dir}}/reports/{agent_name}.json
   The file MUST persist on the filesystem after you finish. Do not just return the JSON — it must be saved to disk.
9. Return the JSON report content as well (for the orchestrator to use immediately).
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
- MR metadata from `{{tmp_dir}}/metadata.json`
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

- If GitLab MCP is unavailable during Phase 1, stop and report the error to the user.
- If no `.go` files in the diff, report "No Go files changed in this MR" and stop.
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

Temporary files in `{{tmp_dir}}` (metadata.json, diffs/, files/) will be cleaned up by the OS.

**Important for sub-agents:** When writing reports to `reports/`, use the Write tool to create the file. Do NOT use temporary storage — the JSON report must persist on disk at `{{output_dir}}/reports/{agent_name}.json`.
