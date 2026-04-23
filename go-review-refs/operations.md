# GitLab MCP Operations

Available ops on MCP server `user-gitlab`.

## Files and Assets

- `get_file_contents` - Read full contents of file or list directory from repo.
- `get_repository_tree` - List files and directories in repo, optionally recursively.

## Commits and Diffs

- `get_branch_diffs` - Compare two branches or commits, return diff.
- `get_commit` - Get metadata for commit, optionally including stats.
- `get_commit_diff` - Get diff for single commit.
- `list_commits` - List commits with filters: branch, author, path, date range.

## Merge Requests

- `get_merge_request` - Get one merge request by IID or source branch.
- `get_merge_request_diffs` - Get diff for merge request.
- `mr_discussions` - List MR discussions and review threads.
- `list_merge_requests` - Search and list merge requests.

## Projects and Groups

- `get_project` - Get metadata for project.
- `search_repositories` - Search GitLab projects by text query.
- `list_group_projects` - List projects in group.

## Selection Guide

- Need full file: use `get_file_contents`.
- Need MR diffs: use `get_merge_request_diffs`.
- Need MR metadata + discussions: use `get_merge_request` and `mr_discussions`.
- Need to search for function across repo: use `get_repository_tree` then `get_file_contents`.