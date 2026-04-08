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
