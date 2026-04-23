---
name: go-review-branch
description: Multi-agent Go code review for local branch changes vs target branch. Use when the user wants to review all committed changes on their current branch relative to a target branch (e.g., main). Analyzes correctness, concurrency, conventions, style, tests, consistency, transactions, performance, and security using 9 specialized sub-agents. Supports selective review with --only flag.
---

# Go Review (Branch)

Multi-agent Go code review — committed changes on current branch vs target. 9 sub-agents analyze diffs + source, merge into unified report.

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

Agents: `correctness`, `concurrency`, `conventions`, `style`, `tests`, `consistency`, `transactions`, `performance`, `security`

## Parse Input

1. Target branch — first arg (required). Validate: `git rev-parse --verify <target_branch>`. Not found → stop: "Branch `<target_branch>` not found."

2. Source branch: `git branch --show-current`. Detached HEAD → stop: "Cannot review from detached HEAD."

3. `--only`: split comma-separated, validate. Invalid → stop, list valid. Absent → all 9.

4. `additional_context`: remaining text not flag/value.

5. Paths:
   - `tmp_dir` = `/tmp/go-review/<YYYY-MM-DDTHH-MM>_branch-<source_branch>/`
   - `output_dir` = `docs/review/<YYYY-MM-DDTHH-MM>_branch-<source_branch>/`

## Execute

Load `workflow.md` (same dir) with resolved: `{{target_branch}}`, `{{source_branch}}`, `{{selected_agents}}`, `{{additional_context}}`, `{{tmp_dir}}`, `{{output_dir}}`
