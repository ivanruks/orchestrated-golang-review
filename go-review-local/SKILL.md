---
name: go-review-local
description: Multi-agent Go code review for uncommitted local changes vs target branch. Use when the user wants to review their staged and unstaged (tracked) changes relative to a target branch (e.g., main) before committing. Analyzes correctness, concurrency, conventions, tests, consistency, transactions, performance, and security using 8 specialized sub-agents. Supports selective review with --only flag.
---

# Go Review (Local Uncommitted)

Multi-agent Go code review for uncommitted changes (staged + unstaged, tracked files only) relative to a target branch. Dispatches 8 specialized sub-agents to analyze diffs and source files, then merges their findings into a unified report.

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

Available agents: `correctness`, `concurrency`, `conventions`, `tests`, `consistency`, `transactions`, `performance`, `security`

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
   - If `--only` not present, use all 8 agents

5. Capture `additional_context`: all remaining text that is not a recognized flag or flag value.

6. Generate paths:
   - `tmp_dir` = `/tmp/go-review/<YYYY-MM-DDTHH-MM>_local-<source_branch>/`
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
