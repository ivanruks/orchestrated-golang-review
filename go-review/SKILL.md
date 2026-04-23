---
name: go-review
description: Multi-agent Go code review for GitLab merge requests. Use when user shares GitLab MR URL and wants comprehensive Go review. Analyzes correctness, concurrency, conventions, style, tests, consistency, transactions, performance, security using 9 specialized sub-agents. Supports selective review with --only flag and MR discussions loading with --discussions flag.
---

# Go Review (GitLab MR)

Multi-agent Go code review for GitLab MRs. 9 sub-agents analyze diffs + full files, merge into unified report.

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

Agents: `correctness`, `concurrency`, `conventions`, `style`, `tests`, `consistency`, `transactions`, `performance`, `security`

## Parse Input

1. Extract MR URL. Format: `https://gitlab.com/<namespace>/<project>/-/merge_requests/<iid>`
   - `project_id` = path before `/-/merge_requests/`, URL-decoded
   - `merge_request_iid` = numeric ID after `merge_requests/`

2. Malformed URL → stop: "Expected GitLab MR URL: `https://gitlab.com/<project>/-/merge_requests/<iid>`"

3. `--only`: split comma-separated, validate against agents. Invalid → stop, list valid. Absent → all 9.

4. `--discussions`: present → `discussions_enabled = true`, else `false`.

5. `additional_context`: remaining text not URL/flag/value.

6. Paths:
   - `tmp_dir` = `/tmp/go-review/<YYYY-MM-DDTHH-MM>_mr-<iid>/`
   - `output_dir` = `docs/review/<YYYY-MM-DDTHH-MM>_mr-<iid>/`

## Execute

Load `workflow.md` (same dir) with resolved: `{{project_id}}`, `{{merge_request_iid}}`, `{{source_branch}}` (Phase 1), `{{selected_agents}}`, `{{discussions_enabled}}`, `{{additional_context}}`, `{{tmp_dir}}`, `{{output_dir}}`

## GitLab MCP

Uses MCP server `user-gitlab`. Load `go-review-refs/operations.md` for op reference.
