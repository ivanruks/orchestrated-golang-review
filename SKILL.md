---
name: golang-review
description: Multi-agent Go code review for GitLab merge requests. Use when the user shares a GitLab MR URL and wants a comprehensive Go review. Analyzes correctness, concurrency, conventions, consistency, transactions, performance, and security using 7 specialized sub-agents. Supports selective review with --only flag.
---

# Golang Review

Multi-agent Go code review system. Dispatches 7 specialized sub-agents to analyze a GitLab merge request, then merges their findings into a unified report.

## Usage

```
<MR_URL>
<MR_URL> --only agent1,agent2,...
```

Examples:
```
https://gitlab.com/mygroup/myproject/-/merge_requests/123
https://gitlab.com/mygroup/myproject/-/merge_requests/123 --only concurrency,security
```

Available agents: `correctness`, `concurrency`, `conventions`, `consistency`, `transactions`, `performance`, `security`

## Parse Input

1. Extract MR URL from user input. Expected format:
   `https://gitlab.com/<namespace>/<project>/-/merge_requests/<iid>`
   - `project_id` = full path before `/-/merge_requests/` (e.g., `group/subgroup/project`)
   - `merge_request_iid` = numeric ID after `merge_requests/`

2. If URL is malformed or not a GitLab MR URL, stop and say:
   "Expected a GitLab MR URL in format: `https://gitlab.com/<project>/-/merge_requests/<iid>`"

3. Parse `--only` flag if present:
   - Split comma-separated values
   - Validate each against known agents list
   - If invalid agent name, stop and list valid names
   - If `--only` not present, use all 7 agents

4. Generate `review_dir`:
   - Format: `docs/review/<timestamp>_mr-<iid>/`
   - Timestamp format: `YYYY-MM-DDTHH-MM` (current time, 24h, dashes instead of colons)
   - Example: `docs/review/2026-04-03T14-30_mr-456/`

## Execute

Load and follow `workflow.md` with the resolved variables:
- `{{project_id}}`
- `{{merge_request_iid}}`
- `{{selected_agents}}`
- `{{review_dir}}`

## GitLab MCP

This skill uses GitLab MCP server `user-gitlab` for all GitLab API operations.
Load `references/operations.md` when you need to find the right MCP operation.
