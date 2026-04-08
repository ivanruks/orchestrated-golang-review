# Multi-Skill Go Code Review System — Design Spec

## Goal

Extend the existing `golang-review` skill (GitLab MR review) into three independent skills sharing a common agent/reference base:

1. **golang-review** — review a GitLab Merge Request
2. **golang-review-branch** — review all commits of current branch vs target branch
3. **golang-review-local** — review uncommitted (staged+unstaged, tracked) changes vs target branch

---

## Repository Structure

```
orchestrated-golang-review/
├── golang-review/                     # Skill: GitLab MR review
│   ├── SKILL.md                       # Entry point: parses MR URL
│   └── workflow.md                    # Phase 1-5, fetch from GitLab
│
├── golang-review-branch/              # Skill: branch vs target review
│   ├── SKILL.md                       # Entry point: parses target branch
│   └── workflow.md                    # Phase 1-5, fetch via git diff target..HEAD
│
├── golang-review-local/               # Skill: uncommitted vs target review
│   ├── SKILL.md                       # Entry point: parses target branch
│   └── workflow.md                    # Phase 1-5, fetch via git diff staged+unstaged
│
├── references/                        # Shared across all skills
│   ├── agents/
│   │   ├── correctness.md
│   │   ├── concurrency.md
│   │   ├── conventions.md
│   │   ├── consistency.md
│   │   ├── transactions.md
│   │   ├── performance.md
│   │   └── security.md
│   ├── context-rules/
│   │   ├── correctness.md
│   │   ├── concurrency.md
│   │   ├── conventions.md
│   │   ├── consistency.md
│   │   ├── transactions.md
│   │   ├── performance.md
│   │   └── security.md
│   ├── agent-output-schema.json
│   ├── report-format.md
│   └── operations.md                  # GitLab MCP catalog (only for golang-review)
│
├── CLAUDE.md
├── README.md
└── LICENSE
```

Each skill folder contains a SKILL.md — cloning this repo into any agent's skills directory makes all three skills immediately discoverable.

---

## Design Decisions

### Monolithic workflows over modular phases

Each workflow.md contains all 5 phases inline. Phases 2-5 are duplicated across the three workflows. This is an intentional trade-off: for LLM-based execution, linear instructions with zero indirection are more reliable than modular composition with file references. Phases 2-5 change rarely; Phase 1 (fetch) is the only part that differs meaningfully.

### No file copying for local skills

Local skills (branch, local) do not copy full files into a working directory. Agents read source files directly from the repository via absolute paths. Only the GitLab skill copies files into tmp_dir/files/ because it fetches them from a remote.

### Temporary vs persistent storage

- `/tmp/golang-review/<timestamp>_<identifier>/` — working data (metadata.json, diffs/, files/). Cleaned up by OS.
- `docs/review/<timestamp>_<identifier>/` — persistent results (agent JSON reports, final-report.md).

### Discussions opt-in for GitLab skill

Existing MR discussions are NOT loaded by default. Pass `--discussions` to enable. This avoids unnecessary API calls and context consumption for most reviews.

### Additional context as free text

No flag required. Any text after recognized arguments is captured as `additional_context` and passed to agents. This gives agents understanding of the intent behind changes.

---

## Skill Invocation

### golang-review

```
/golang-review <MR_URL> [--only agent1,agent2] [--discussions] [free text]
```

Examples:
```
/golang-review https://gitlab.com/group/project/-/merge_requests/123
/golang-review https://gitlab.com/group/project/-/merge_requests/123 --discussions --only concurrency,security
/golang-review https://gitlab.com/group/project/-/merge_requests/123
Тикет PROJ-456: новый эндпоинт регистрации, пишет в users и audit_log.
```

### golang-review-branch

```
/golang-review-branch <target_branch> [--only agent1,agent2] [free text]
```

Examples:
```
/golang-review-branch main
/golang-review-branch main --only correctness,transactions
Рефакторинг слоя репозиториев, переход на sqlc.
```

### golang-review-local

```
/golang-review-local <target_branch> [--only agent1,agent2] [free text]
```

Examples:
```
/golang-review-local main
/golang-review-local main --only concurrency
```

---

## SKILL.md Responsibilities (all three)

1. Parse and validate input arguments
   - golang-review: extract project_id + iid from URL, validate URL format
   - golang-review-branch / local: validate target branch exists (`git rev-parse --verify`)
   - All: validate `--only` agent names against known list
2. Capture `additional_context` (all text not matching known args/flags)
3. Generate paths:
   - `tmp_dir` = `/tmp/golang-review/<YYYY-MM-DDTHH-MM>_<identifier>/`
   - `output_dir` = `docs/review/<YYYY-MM-DDTHH-MM>_<identifier>/`
   - Identifier: `mr-<iid>` / `branch-<source_branch>` / `local-<source_branch>`
4. Load and execute workflow.md with resolved variables

---

## Workflow Phases

### Phase 1: Fetch (differs per skill)

#### golang-review — GitLab fetch

1. `mkdir -p tmp_dir/diffs, tmp_dir/files, output_dir/reports`
2. `get_merge_request` → extract metadata
3. If `--discussions`: `mr_discussions` → `existing_discussions`. Otherwise: `[]`
4. `get_merge_request_diffs` → filter `.go` + EXCLUDED_PATTERNS → `tmp_dir/diffs/`
5. For each .go file (up to 20, sorted by diff size): `get_file_contents` → `tmp_dir/files/`
6. Write `tmp_dir/metadata.json`
7. Output summary to user

**file_access_instructions:**
```
Full files are in {{tmp_dir}}/files/.
If a file is not there, fetch via GitLab MCP (server: "user-gitlab", project: "{{project_id}}", ref: "{{source_branch}}")
```

#### golang-review-branch — git diff target..HEAD

1. `mkdir -p tmp_dir/diffs, output_dir/reports`
2. `repo_root` = `git rev-parse --show-toplevel`
3. `git log target..HEAD` → collect commit messages for description
4. `git diff target..HEAD -- '*.go'` → filter EXCLUDED_PATTERNS → split per file → `tmp_dir/diffs/`
5. Write `tmp_dir/metadata.json`
6. Output summary to user

**file_access_instructions:**
```
Read files directly from the repository: {{repo_root}}/<file_path>
```

#### golang-review-local — git diff staged+unstaged

1. `mkdir -p tmp_dir/diffs, output_dir/reports`
2. `repo_root` = `git rev-parse --show-toplevel`
3. `git diff target -- '*.go'` combined with `git diff --cached target -- '*.go'` → filter EXCLUDED_PATTERNS → split per file → `tmp_dir/diffs/`
4. Write `tmp_dir/metadata.json`
5. Output summary to user

**file_access_instructions:**
```
Read files directly from the repository: {{repo_root}}/<file_path>
```

#### EXCLUDED_PATTERNS (all skills)

```
^vendor/
_mock\.go$
\.pb\.go$
_generated\.go$
^testdata/
\.gen\.go$
```

#### metadata.json schema (all skills)

```json
{
  "title": "...",
  "author": "...",
  "source_branch": "...",
  "target_branch": "...",
  "description": "...",
  "url": "..." ,
  "fetched_at": "...",
  "existing_discussions": [],
  "additional_context": "..."
}
```

For local skills: `url` is empty, `existing_discussions` is `[]`, `description` is commit messages (branch) or empty (local).

#### Phase 1 output contract

After any fetch phase completes:
- `tmp_dir/metadata.json` exists with all fields
- `tmp_dir/diffs/*.diff` — one file per changed .go file
- `output_dir/reports/` — empty directory exists

---

### Phase 2: Wave 1 (same in all workflows)

Launch agents in parallel via Agent tool (single message for true parallelism).

Agents: `correctness`, `concurrency`, `conventions`, `performance`, `security` — intersected with `selected_agents`.

Each agent prompt is assembled inline:

```markdown
You are the {agent_name} review agent.

{contents of references/agents/{agent_name}.md}

## Input

Metadata: {{tmp_dir}}/metadata.json
Diffs: {{tmp_dir}}/diffs/
Output directory: {{output_dir}}/reports/

## File Access

{{file_access_instructions}}

## Context Rules

{contents of references/context-rules/{agent_name}.md}

## Task Context (provided by author)

{{additional_context}}

## Output Schema

{contents of references/agent-output-schema.json}

## Instructions

1. Read metadata.json — check existing_discussions, do NOT duplicate them.
2. Read each .diff file in diffs/.
3. For each diff, read the corresponding full file using File Access instructions above.
4. Apply your checklist to every file.
5. When context-rules trigger, load additional files using File Access instructions.
6. Write your JSON report to {{output_dir}}/reports/{agent_name}.json
7. Return the JSON report content as well.
```

Wait for ALL Wave 1 agents to complete before Phase 3.

---

### Phase 3: Wave 2 (same in all workflows)

Agents: `consistency`, `transactions` — intersected with `selected_agents`.

Same prompt structure as Wave 1, with added section after Input:

```markdown
## Wave 1 Findings

The following agents have completed their analysis. Read their reports before starting —
avoid duplicating their findings and use their context to inform your analysis.

Reports: {{output_dir}}/reports/
```

Wait for ALL Wave 2 agents to complete before Phase 4.

---

### Phase 4: Merge (same in all workflows)

1. Read all `output_dir/reports/*.json`
2. Deduplicate by file+line — specialized agent wins:
   - concurrency > correctness (race conditions)
   - transactions > correctness (resource leaks in tx scope)
   - security > correctness (injection issues)
   - consistency > conventions (interface contracts)
3. Group: severity (critical → major → minor), then by agent category
4. Compute statistics: files_checked, lines_added/removed, counts per severity
5. Verdict: 1+ critical → REJECT, 1+ major → REQUEST CHANGES, else LGTM
6. Merge positive findings from all agents, deduplicate

---

### Phase 5: Report (same in all workflows)

1. Read `references/report-format.md`
2. Read `tmp_dir/metadata.json` for report header
3. Fill template with Phase 4 data
4. If `--only` was used: add "Partial review: only {agents} were run"
5. Write to `output_dir/final-report.md`
6. Display full report to user

---

## Error Handling

| Situation | Behavior |
|-----------|----------|
| GitLab MCP unavailable (Phase 1, golang-review only) | Stop, report error to user |
| Target branch does not exist (Phase 1, local skills) | Stop, report error to user |
| No .go files in diff | Stop: "No Go files changed" |
| Agent failed (timeout, error) | Continue with remaining agents, note in report |
| All Wave 1 agents failed | Stop, do not launch Wave 2 |
| Invalid JSON from agent | Skip agent, note in report |

---

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

MAX_FILES_TO_FETCH = 20  (GitLab skill only)
```
