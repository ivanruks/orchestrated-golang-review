# Go Review (Local Uncommitted)

Multi-agent Go code review — uncommitted changes (staged + unstaged, tracked) vs target branch. 9 sub-agents analyze diffs + source, merge into unified report.

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

Agents: `correctness`, `concurrency`, `conventions`, `style`, `tests`, `consistency`, `transactions`, `performance`, `security`

## Parse Input

1. Target branch — first arg (required). Validate: `git rev-parse --verify <target_branch>`. Not found → stop: "Branch `<target_branch>` not found."

2. Source branch: `git branch --show-current`. Detached HEAD → stop: "Cannot review from detached HEAD."

3. Check changes exist: `git diff {{target_branch}} -- '*.go'` + `git diff --cached {{target_branch}} -- '*.go'`. Both empty → stop: "No uncommitted Go changes relative to `{{target_branch}}`."

4. `--only`: split comma-separated, validate. Invalid → stop, list valid. Absent → all 9.

5. `additional_context`: remaining text not flag/value.

6. Paths:
   - `tmp_dir` = `/tmp/go-review/<YYYY-MM-DDTHH-MM>_local-<source_branch>/`
   - `output_dir` = `docs/review/<YYYY-MM-DDTHH-MM>_local-<source_branch>/`

## Execute

Load `workflow.md` (same dir) with resolved: `{{target_branch}}`, `{{source_branch}}`, `{{selected_agents}}`, `{{additional_context}}`, `{{tmp_dir}}`, `{{output_dir}}`
