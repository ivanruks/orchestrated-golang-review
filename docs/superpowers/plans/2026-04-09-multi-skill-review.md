# Multi-Skill Go Code Review — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restructure the monolithic golang-review skill into three independent skills (gitlab-mr, branch, local-uncommitted) sharing common agents and references.

**Architecture:** Three skill folders at the repo root, each with SKILL.md + workflow.md. Shared agents, context-rules, schema, and report template live in references/. Each workflow is monolithic (all 5 phases inline) for maximum LLM reliability.

**Tech Stack:** Claude Code skills (Markdown prompts), GitLab MCP, local git CLI.

**Spec:** `docs/superpowers/specs/2026-04-09-multi-skill-review-design.md`

---

### Task 1: Create directory structure and move unchanged files

**Files:**
- Create: `golang-review/` directory
- Create: `golang-review-branch/` directory
- Create: `golang-review-local/` directory
- Move: `agents/` → `references/agents/`
- Keep: `references/context-rules/`, `references/agent-output-schema.json`, `references/operations.md`, `references/report-format.md` (already in place)

- [ ] **Step 1: Create the three skill directories**

```bash
mkdir -p golang-review golang-review-branch golang-review-local
```

- [ ] **Step 2: Move agents/ into references/**

```bash
git mv agents/ references/agents/
```

Verify: `ls references/agents/` should list 7 files: correctness.md, concurrency.md, conventions.md, consistency.md, transactions.md, performance.md, security.md.

- [ ] **Step 3: Commit**

```bash
git add -A
git commit -m "chore: create skill directories and move agents to references/"
```

---

### Task 2: Update context-rules to be source-agnostic

Currently the "How" column references MCP operations and `files/` directory. Agents will receive `file_access_instructions` in their prompt from the workflow, so context-rules should say "Use File Access instructions" instead of hard-coding MCP.

**Files:**
- Modify: `references/context-rules/correctness.md`
- Modify: `references/context-rules/concurrency.md`
- Modify: `references/context-rules/conventions.md`
- Modify: `references/context-rules/consistency.md`
- Modify: `references/context-rules/transactions.md`
- Modify: `references/context-rules/performance.md`
- Modify: `references/context-rules/security.md`

- [ ] **Step 1: Update all 7 context-rules files**

Replace the column header `How (MCP operation)` with `How` in every file.

Replace all cell values in that column:
- `get_file_contents` (check `files/` first) → `Use File Access instructions from your prompt`
- `get_file_contents` (check `files/` first, then MCP) → `Use File Access instructions from your prompt`
- `get_file_contents` for the file → `Use File Access instructions from your prompt`
- `get_file_contents` for caller files → `Use File Access instructions from your prompt`
- `get_file_contents` for callers → `Use File Access instructions from your prompt`
- `get_file_contents` for interface file → `Use File Access instructions from your prompt`
- `get_file_contents` for the file (check `files/` first) → `Use File Access instructions from your prompt`
- `get_file_contents` for consumer files → `Use File Access instructions from your prompt`
- `get_file_contents` for handler + service → `Use File Access instructions from your prompt`
- `get_file_contents` for router file → `Use File Access instructions from your prompt`
- `get_file_contents` for the .proto → `Use File Access instructions from your prompt`
- `get_file_contents` for go.mod → `Use File Access instructions from your prompt`
- `get_file_contents` → `Use File Access instructions from your prompt`
- Search repo for function name, then `get_file_contents` → `Search repo for function name, then use File Access instructions`
- Search repo for interface name, then `get_file_contents` → `Search repo for interface name, then use File Access instructions`
- Search repo for struct name, then `get_file_contents` → `Search repo for struct name, then use File Access instructions`
- Search for error name, then `get_file_contents` → `Search for error name, then use File Access instructions`
- Search repo for table name, then `get_file_contents` → `Search for table name, then use File Access instructions`
- Search for table name, check migration files → `Search for table name, check migration files via File Access instructions`
- Search repo or `get_repository_tree` + `get_file_contents` → `Search repo, then use File Access instructions`
- `N/A` → keep as `N/A`

Read each file, make the replacements, verify the table still renders correctly (4 columns, all rows aligned).

- [ ] **Step 2: Verify all 7 files**

Read each updated file and confirm:
- Header is `| Trigger in diff | What to load | How | Why |`
- No remaining references to `get_file_contents`, `MCP`, or `files/` in the How column
- N/A entries are preserved

- [ ] **Step 3: Commit**

```bash
git add references/context-rules/
git commit -m "refactor: make context-rules source-agnostic for multi-skill support"
```

---

### Task 3: Update report-format.md for non-MR reviews

The current header assumes a GitLab MR. It needs to support branch and local reviews too.

**Files:**
- Modify: `references/report-format.md`

- [ ] **Step 1: Update the report header**

Replace:
```markdown
# Go Code Review | MR !{iid}: {title}
**{author}** · `{source_branch}` → `{target_branch}`
```

With:
```markdown
# Go Code Review | {title}
**{author}** · `{source_branch}` → `{target_branch}`
```

This works for all three skills:
- GitLab MR: title = "MR !123: Add user endpoint"
- Branch: title = "Branch feature/xyz vs main"
- Local: title = "Uncommitted changes vs main"

The title field in metadata.json is set by each skill's Phase 1 with the appropriate format.

- [ ] **Step 2: Commit**

```bash
git add references/report-format.md
git commit -m "refactor: generalize report header for non-MR review sources"
```

---

### Task 4: Write golang-review/SKILL.md

Refactored from the existing root-level SKILL.md. Adds `--discussions` flag, `additional_context` capture, and references workflow.md in the same directory.

**Files:**
- Create: `golang-review/SKILL.md`

- [ ] **Step 1: Write golang-review/SKILL.md**

```markdown
---
name: golang-review
description: Multi-agent Go code review for GitLab merge requests. Use when the user shares a GitLab MR URL and wants a comprehensive Go review. Analyzes correctness, concurrency, conventions, consistency, transactions, performance, and security using 7 specialized sub-agents. Supports selective review with --only flag and MR discussions loading with --discussions flag.
---

# Golang Review (GitLab MR)

Multi-agent Go code review system for GitLab Merge Requests. Dispatches 7 specialized sub-agents to analyze diffs and full files, then merges their findings into a unified report.

## Usage

```
<MR_URL> [--only agent1,agent2,...] [--discussions] [free text context]
```

Examples:
```
https://gitlab.com/mygroup/myproject/-/merge_requests/123
https://gitlab.com/mygroup/myproject/-/merge_requests/123 --discussions
https://gitlab.com/mygroup/myproject/-/merge_requests/123 --only concurrency,security
https://gitlab.com/mygroup/myproject/-/merge_requests/123 --only correctness,transactions
Тикет PROJ-456: новый эндпоинт регистрации, пишет в users и audit_log.
```

Available agents: `correctness`, `concurrency`, `conventions`, `consistency`, `transactions`, `performance`, `security`

## Parse Input

1. Extract MR URL from user input. Expected format:
   `https://gitlab.com/<namespace>/<project>/-/merge_requests/<iid>`
   - `project_id` = full path before `/-/merge_requests/` (e.g., `group/subgroup/project`), URL-decoded
   - `merge_request_iid` = numeric ID after `merge_requests/`

2. If URL is malformed or not a GitLab MR URL, stop and say:
   "Expected a GitLab MR URL in format: `https://gitlab.com/<project>/-/merge_requests/<iid>`"

3. Parse `--only` flag if present:
   - Split comma-separated values
   - Validate each against known agents list
   - If invalid agent name, stop and list valid names
   - If `--only` not present, use all 7 agents

4. Parse `--discussions` flag: if present, set `discussions_enabled = true`. Otherwise `false`.

5. Capture `additional_context`: all remaining text that is not a recognized URL, flag, or flag value.

6. Generate paths:
   - `tmp_dir` = `/tmp/golang-review/<YYYY-MM-DDTHH-MM>_mr-<iid>/`
   - `output_dir` = `docs/review/<YYYY-MM-DDTHH-MM>_mr-<iid>/`
   - Timestamp: current time, 24h format, dashes instead of colons

## Execute

Load and follow `workflow.md` (in the same directory as this SKILL.md) with the resolved variables:
- `{{project_id}}`
- `{{merge_request_iid}}`
- `{{source_branch}}` (resolved in Phase 1)
- `{{selected_agents}}`
- `{{discussions_enabled}}`
- `{{additional_context}}`
- `{{tmp_dir}}`
- `{{output_dir}}`

## GitLab MCP

This skill uses GitLab MCP server `user-gitlab` for all GitLab API operations.
Load `references/operations.md` when you need to find the right MCP operation.
```

- [ ] **Step 2: Commit**

```bash
git add golang-review/SKILL.md
git commit -m "feat: create golang-review skill entry point with --discussions flag"
```

---

### Task 5: Write golang-review/workflow.md

Full monolithic workflow for GitLab MR review. Phase 1 fetches from GitLab MCP. Phases 2-5 are the standard analysis/merge/report pipeline. Uses `tmp_dir` for working data and `output_dir` for persistent reports.

**Files:**
- Create: `golang-review/workflow.md`

- [ ] **Step 1: Write golang-review/workflow.md**

```markdown
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
4. Apply your checklist to every file.
5. When your context-rules trigger, load additional files using File Access instructions above.
6. Do NOT duplicate findings from existing_discussions in metadata.json.
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
  - If neither is more specialized, keep both (different perspectives are valuable)

### Step 4.3 — Sort and group

Group findings:
1. First level: severity (critical → major → minor)
2. Second level: agent category (in order: correctness, concurrency, conventions, consistency, transactions, performance, security)

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

Collect all `positive` arrays from all agents. Deduplicate similar observations. Keep as bullet list.

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
```

- [ ] **Step 2: Commit**

```bash
git add golang-review/
git commit -m "feat: create golang-review skill with monolithic GitLab MR workflow"
```

---

### Task 6: Write golang-review-branch/SKILL.md

Entry point for reviewing all commits on current branch vs a target branch.

**Files:**
- Create: `golang-review-branch/SKILL.md`

- [ ] **Step 1: Write golang-review-branch/SKILL.md**

```markdown
---
name: golang-review-branch
description: Multi-agent Go code review for local branch changes vs target branch. Use when the user wants to review all committed changes on their current branch relative to a target branch (e.g., main). Analyzes correctness, concurrency, conventions, consistency, transactions, performance, and security using 7 specialized sub-agents. Supports selective review with --only flag.
---

# Golang Review (Branch)

Multi-agent Go code review for all commits on the current branch relative to a target branch. Dispatches 7 specialized sub-agents to analyze diffs and source files, then merges their findings into a unified report.

## Usage

```
<target_branch> [--only agent1,agent2,...] [free text context]
```

Examples:
```
main
main --only concurrency,security
develop --only correctness,transactions
main
Рефакторинг слоя репозиториев, переход на sqlc.
```

Available agents: `correctness`, `concurrency`, `conventions`, `consistency`, `transactions`, `performance`, `security`

## Parse Input

1. Extract target branch name — the first argument (required).
   - Validate it exists: `git rev-parse --verify <target_branch>`
   - If it does not exist, stop and say: "Branch `<target_branch>` not found."

2. Determine source branch: `git branch --show-current`
   - If detached HEAD, stop and say: "Cannot review from detached HEAD. Please check out a branch."

3. Parse `--only` flag if present:
   - Split comma-separated values
   - Validate each against known agents list
   - If invalid agent name, stop and list valid names
   - If `--only` not present, use all 7 agents

4. Capture `additional_context`: all remaining text that is not a recognized flag or flag value.

5. Generate paths:
   - `tmp_dir` = `/tmp/golang-review/<YYYY-MM-DDTHH-MM>_branch-<source_branch>/`
   - `output_dir` = `docs/review/<YYYY-MM-DDTHH-MM>_branch-<source_branch>/`
   - Timestamp: current time, 24h format, dashes instead of colons

## Execute

Load and follow `workflow.md` (in the same directory as this SKILL.md) with the resolved variables:
- `{{target_branch}}`
- `{{source_branch}}`
- `{{selected_agents}}`
- `{{additional_context}}`
- `{{tmp_dir}}`
- `{{output_dir}}`
```

- [ ] **Step 2: Commit**

```bash
git add golang-review-branch/SKILL.md
git commit -m "feat: create golang-review-branch skill entry point"
```

---

### Task 7: Write golang-review-branch/workflow.md

Full monolithic workflow for branch review. Phase 1 fetches diffs via `git diff target..HEAD`. Phases 2-5 are identical to golang-review except `file_access_instructions` tells agents to read files from repo root.

**Files:**
- Create: `golang-review-branch/workflow.md`

- [ ] **Step 1: Write golang-review-branch/workflow.md**

```markdown
---
name: golang-review-branch-workflow
description: 5-phase orchestration workflow for local branch Go code review
---

# Golang Review Workflow — Branch

## Prerequisites

Before executing this workflow, the orchestrator (SKILL.md) must have resolved:
- `{{target_branch}}` — branch to compare against (e.g., `main`)
- `{{source_branch}}` — current branch name
- `{{selected_agents}}` — list of agents to run (default: all 7)
- `{{additional_context}}` — free text context from user (may be empty)
- `{{tmp_dir}}` — path to temporary working directory (e.g., `/tmp/golang-review/2026-04-03T14-30_branch-feature-xyz/`)
- `{{output_dir}}` — path to persistent output directory (e.g., `docs/review/2026-04-03T14-30_branch-feature-xyz/`)

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
```

---

## Phase 1: Fetch from Local Git

Goal: Generate diffs from local git and save to `{{tmp_dir}}`.

### Step 1.1 — Create directories

```bash
mkdir -p {{tmp_dir}}/diffs
mkdir -p {{output_dir}}/reports
```

### Step 1.2 — Determine repo root

```bash
repo_root=$(git rev-parse --show-toplevel)
```

### Step 1.3 — Get commit log for description

```bash
git log {{target_branch}}..HEAD --format="%h %s" --no-merges
```

Save output as `commit_log` for metadata.

### Step 1.4 — Get diffs

```bash
git diff {{target_branch}}..HEAD -- '*.go'
```

From the combined diff output:
- Split into per-file diffs
- Exclude files matching EXCLUDED_FILE_PATTERNS
- Save each file's diff to `{{tmp_dir}}/diffs/<path-with-dashes>.diff`
  (replace `/` with `-` in path)
- Track: file path, lines added, lines removed

If no `.go` files in diff, stop and report: "No Go files changed between `{{source_branch}}` and `{{target_branch}}`."

### Step 1.5 — Get author

```bash
git config user.name
```

### Step 1.6 — Write metadata.json

Save to `{{tmp_dir}}/metadata.json`:
```json
{
  "title": "Branch {{source_branch}} vs {{target_branch}}",
  "author": "<git config user.name>",
  "source_branch": "{{source_branch}}",
  "target_branch": "{{target_branch}}",
  "description": "<commit_log from step 1.3>",
  "url": "",
  "fetched_at": "<current ISO timestamp>",
  "existing_discussions": [],
  "additional_context": "{{additional_context}}"
}
```

### Step 1.7 — Report fetch results

Output to user:
```
Review: {{source_branch}} vs {{target_branch}}
  Author: <author>
  Files: <N> .go files (<lines_added>+ / <lines_removed>-)
  Commits: <N>
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
4. Apply your checklist to every file.
5. When your context-rules trigger, load additional files using File Access instructions above.
6. **IMPORTANT: Write your JSON report to disk** using the Write tool at {{output_dir}}/reports/{agent_name}.json
   The file MUST persist on the filesystem after you finish. Do not just return the JSON — it must be saved to disk.
7. Return the JSON report content as well (for the orchestrator to use immediately).
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
  - If neither is more specialized, keep both (different perspectives are valuable)

### Step 4.3 — Sort and group

Group findings:
1. First level: severity (critical → major → minor)
2. Second level: agent category (in order: correctness, concurrency, conventions, consistency, transactions, performance, security)

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

Collect all `positive` arrays from all agents. Deduplicate similar observations. Keep as bullet list.

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
- If no `.go` files in the diff, report "No Go files changed between {{source_branch}} and {{target_branch}}" and stop.
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
```

- [ ] **Step 2: Commit**

```bash
git add golang-review-branch/
git commit -m "feat: create golang-review-branch skill with local git workflow"
```

---

### Task 8: Write golang-review-local/SKILL.md

Entry point for reviewing uncommitted (staged + unstaged, tracked only) changes vs a target branch.

**Files:**
- Create: `golang-review-local/SKILL.md`

- [ ] **Step 1: Write golang-review-local/SKILL.md**

```markdown
---
name: golang-review-local
description: Multi-agent Go code review for uncommitted local changes vs target branch. Use when the user wants to review their staged and unstaged (tracked) changes relative to a target branch (e.g., main) before committing. Analyzes correctness, concurrency, conventions, consistency, transactions, performance, and security using 7 specialized sub-agents. Supports selective review with --only flag.
---

# Golang Review (Local Uncommitted)

Multi-agent Go code review for uncommitted changes (staged + unstaged, tracked files only) relative to a target branch. Dispatches 7 specialized sub-agents to analyze diffs and source files, then merges their findings into a unified report.

## Usage

```
<target_branch> [--only agent1,agent2,...] [free text context]
```

Examples:
```
main
main --only concurrency,security
develop --only correctness,transactions
main
Добавляю валидацию email и проверку уникальности username.
```

Available agents: `correctness`, `concurrency`, `conventions`, `consistency`, `transactions`, `performance`, `security`

## Parse Input

1. Extract target branch name — the first argument (required).
   - Validate it exists: `git rev-parse --verify <target_branch>`
   - If it does not exist, stop and say: "Branch `<target_branch>` not found."

2. Determine source branch: `git branch --show-current`
   - If detached HEAD, stop and say: "Cannot review from detached HEAD. Please check out a branch."

3. Check that there are uncommitted changes:
   - Run `git diff {{target_branch}} -- '*.go'` and `git diff --cached {{target_branch}} -- '*.go'`
   - If both are empty, stop and say: "No uncommitted Go file changes found relative to `{{target_branch}}`."

4. Parse `--only` flag if present:
   - Split comma-separated values
   - Validate each against known agents list
   - If invalid agent name, stop and list valid names
   - If `--only` not present, use all 7 agents

5. Capture `additional_context`: all remaining text that is not a recognized flag or flag value.

6. Generate paths:
   - `tmp_dir` = `/tmp/golang-review/<YYYY-MM-DDTHH-MM>_local-<source_branch>/`
   - `output_dir` = `docs/review/<YYYY-MM-DDTHH-MM>_local-<source_branch>/`
   - Timestamp: current time, 24h format, dashes instead of colons

## Execute

Load and follow `workflow.md` (in the same directory as this SKILL.md) with the resolved variables:
- `{{target_branch}}`
- `{{source_branch}}`
- `{{selected_agents}}`
- `{{additional_context}}`
- `{{tmp_dir}}`
- `{{output_dir}}`
```

- [ ] **Step 2: Commit**

```bash
git add golang-review-local/SKILL.md
git commit -m "feat: create golang-review-local skill entry point"
```

---

### Task 9: Write golang-review-local/workflow.md

Full monolithic workflow for uncommitted changes review. Phase 1 fetches diffs via `git diff` (staged + unstaged vs target). Phases 2-5 identical to golang-review-branch.

**Files:**
- Create: `golang-review-local/workflow.md`

- [ ] **Step 1: Write golang-review-local/workflow.md**

```markdown
---
name: golang-review-local-workflow
description: 5-phase orchestration workflow for uncommitted Go code review
---

# Golang Review Workflow — Local Uncommitted

## Prerequisites

Before executing this workflow, the orchestrator (SKILL.md) must have resolved:
- `{{target_branch}}` — branch to compare against (e.g., `main`)
- `{{source_branch}}` — current branch name
- `{{selected_agents}}` — list of agents to run (default: all 7)
- `{{additional_context}}` — free text context from user (may be empty)
- `{{tmp_dir}}` — path to temporary working directory (e.g., `/tmp/golang-review/2026-04-03T14-30_local-feature-xyz/`)
- `{{output_dir}}` — path to persistent output directory (e.g., `docs/review/2026-04-03T14-30_local-feature-xyz/`)

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
4. Apply your checklist to every file.
5. When your context-rules trigger, load additional files using File Access instructions above.
6. **IMPORTANT: Write your JSON report to disk** using the Write tool at {{output_dir}}/reports/{agent_name}.json
   The file MUST persist on the filesystem after you finish. Do not just return the JSON — it must be saved to disk.
7. Return the JSON report content as well (for the orchestrator to use immediately).
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
  - If neither is more specialized, keep both (different perspectives are valuable)

### Step 4.3 — Sort and group

Group findings:
1. First level: severity (critical → major → minor)
2. Second level: agent category (in order: correctness, concurrency, conventions, consistency, transactions, performance, security)

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

Collect all `positive` arrays from all agents. Deduplicate similar observations. Keep as bullet list.

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
```

- [ ] **Step 2: Commit**

```bash
git add golang-review-local/
git commit -m "feat: create golang-review-local skill with uncommitted changes workflow"
```

---

### Task 10: Remove old root-level files and update docs

Remove the original SKILL.md and workflow.md from the root (replaced by skill-specific versions). Update README.md and CLAUDE.md to reflect the new structure.

**Files:**
- Delete: `SKILL.md` (root)
- Delete: `workflow.md` (root)
- Modify: `README.md`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Remove old files**

```bash
git rm SKILL.md workflow.md
```

- [ ] **Step 2: Update CLAUDE.md**

Rewrite CLAUDE.md to reflect the new multi-skill structure. It should describe:
- Three skills and their invocation format
- The shared references directory
- The monolithic workflow approach (Phase 1 differs, Phases 2-5 same across skills)
- tmp_dir vs output_dir split
- That agents, context-rules, schema, report template are shared in references/

Read the current CLAUDE.md first, then rewrite it.

- [ ] **Step 3: Update README.md**

Rewrite README.md to document:
- Overview of the three skills
- Updated file structure showing 3 skill folders + references/
- Usage examples for all three skills (with --only, --discussions, additional context)
- Updated architecture diagram showing the three entry points
- Sub-agent table (unchanged)
- Updated workflow phases description (tmp_dir/output_dir split, three Phase 1 variants)
- Context-rules description (source-agnostic)
- Updated review directory structure (tmp vs docs)
- Error handling table

Read the current README.md first, then rewrite it to match the new structure while preserving the style and level of detail.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "chore: remove old root-level files, update README and CLAUDE.md for multi-skill structure"
```
