# Context Rules: Correctness Agent

Before analyzing diffs, read this table. When you see a trigger pattern in the diff, load the additional context described.

| Trigger in diff | What to load | How (MCP operation) | Why |
|---|---|---|---|
| `defer` inside a `for` loop | Full function containing the loop | `get_file_contents` for the file (check `files/` first) | Understand scope — defer executes at function exit, not loop iteration |
| Ignored error: `_ = f()` without comment | The calling function and callers of current function | `get_file_contents` for caller files | Assess criticality — is this in a critical path? |
| `resp.Body` or `*http.Response` without `defer resp.Body.Close()` | Full HTTP handler/function | `get_file_contents` for the file | Check ALL exit paths — early returns may skip Close |
| `*sql.Rows` without `defer rows.Close()` | Full function | `get_file_contents` for the file | Same as above — check all exit paths |
| `*os.File` without `defer f.Close()` | Full function | `get_file_contents` for the file | Same — resource leak on error paths |
| `panic()` in non-test code | Full function + callers | `get_file_contents` | Determine if panic is in init/main or leaks into normal flow |
| `nil` returned where pointer is expected | Interface definition and callers | `get_file_contents` for interface file | Check if callers handle nil |
| Type assertion without ok check: `x.(T)` | Full function | `get_file_contents` | Will panic on wrong type at runtime |
