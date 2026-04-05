# GitLab MCP Operations

Available operations on MCP server `user-gitlab`.

## Files and Assets

- `get_file_contents` - Read the full contents of a file or list a directory from a repository.
- `get_repository_tree` - List files and directories in a repository, optionally recursively.

## Commits and Diffs

- `get_branch_diffs` - Compare two branches or commits and return the diff.
- `get_commit` - Get metadata for a commit, optionally including stats.
- `get_commit_diff` - Get the diff for a single commit.
- `list_commits` - List commits with filters such as branch, author, path, or date range.

## Merge Requests

- `get_merge_request` - Get one merge request by IID or source branch.
- `get_merge_request_diffs` - Get the diff for a merge request.
- `mr_discussions` - List merge request discussions and review threads.
- `list_merge_requests` - Search and list merge requests.

## Projects and Groups

- `get_project` - Get metadata for a project.
- `search_repositories` - Search GitLab projects by text query.
- `list_group_projects` - List projects in a group.

## Selection Guide

- Need a full file: use `get_file_contents`.
- Need MR diffs: use `get_merge_request_diffs`.
- Need MR metadata + discussions: use `get_merge_request` and `mr_discussions`.
- Need to search for a function across the repo: use `get_repository_tree` then `get_file_contents`.
