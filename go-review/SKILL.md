---
name: go-review
description: Multi-agent Go code review for GitLab merge requests. Use when the user shares a GitLab MR URL and wants a comprehensive Go review. Analyzes correctness, concurrency, conventions, style, tests, consistency, transactions, performance, and security using 9 specialized sub-agents. Supports selective review with --only flag and MR discussions loading with --discussions flag.
---

# Go Review (GitLab MR)

Multi-agent Go code review system for GitLab Merge Requests. Dispatches 9 specialized sub-agents to analyze diffs and full files, then merges their findings into a unified report.

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

Available agents: `correctness`, `concurrency`, `conventions`, `style`, `tests`, `consistency`, `transactions`, `performance`, `security`

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
   - If `--only` not present, use all 9 agents

4. Parse `--discussions` flag: if present, set `discussions_enabled = true`. Otherwise `false`.

5. Capture `additional_context`: all remaining text that is not a recognized URL, flag, or flag value.

6. Generate paths:
   - `tmp_dir` = `/tmp/go-review/<YYYY-MM-DDTHH-MM>_mr-<iid>/`
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
Load `go-review-refs/operations.md` when you need to find the right MCP operation.
