# Context Rules: Correctness Agent

Before analyzing diffs, read this table. When you see a trigger pattern in the diff, load the additional context described.

| Trigger in diff | What to load | How | Why |
|---|---|---|---|
| `defer` inside a `for` loop | Full function containing the loop | Use File Access instructions from your prompt | Understand scope — defer executes at function exit, not loop iteration |
| Ignored error: `_ = f()` without comment | The calling function and callers of current function | Use File Access instructions from your prompt | Assess criticality — is this in a critical path? |
| `resp.Body` or `*http.Response` without `defer resp.Body.Close()` | Full HTTP handler/function | Use File Access instructions from your prompt | Check ALL exit paths — early returns may skip Close |
| `*sql.Rows` without `defer rows.Close()` | Full function | Use File Access instructions from your prompt | Same as above — check all exit paths |
| `*os.File` without `defer f.Close()` | Full function | Use File Access instructions from your prompt | Same — resource leak on error paths |
| `panic()` in non-test code | Full function + callers | Use File Access instructions from your prompt | Determine if panic is in init/main or leaks into normal flow |
| `nil` returned where pointer is expected | Interface definition and callers | Use File Access instructions from your prompt | Check if callers handle nil |
| Type assertion without ok check: `x.(T)` | Full function | Use File Access instructions from your prompt | Will panic on wrong type at runtime |
