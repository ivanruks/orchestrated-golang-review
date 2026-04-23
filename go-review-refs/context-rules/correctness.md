# Context Rules: Correctness Agent

When trigger seen in diff, load context described. **How**: File Access instructions from agent prompt; search repo when noted.

| Trigger | Load | Why |
|---|---|---|
| `defer` inside `for` loop | Full function w/ loop | defer executes at func exit, not loop iteration |
| Ignored error: `_ = f()` without comment | Calling func + callers | Assess criticality — critical path? |
| `resp.Body` without `defer resp.Body.Close()` | Full HTTP func | All exit paths — early returns may skip Close |
| `*sql.Rows` without `defer rows.Close()` | Full function | All exit paths |
| `*os.File` without `defer f.Close()` | Full function | All exit paths |
| `panic()` in non-test code | Full func + callers | In init/main or leaks into normal flow? |
| `nil` returned where pointer expected | Interface def + callers | Callers handle nil? |
| Type assertion without ok: `x.(T)` | Full function | Panics on wrong type at runtime |
| `context.WithCancel`/`WithTimeout` without `defer cancel()` | Full function | Leaks goroutines + resources until parent cancelled |
| `ctx, cancel := context.WithTimeout`/`WithCancel` inside `if` | Full func w/ `if` | Inner `ctx` shadows outer; deadline never applies after block |
| Exported func declares concrete error pointer return (e.g. `func F() *MyError`) | Full func + callers | Nil `*MyError` = non-nil `error` interface — callers see err != nil |
| `return e` where `e` is concrete pointer error type, func returns `error` | Full function | Typed nil `*T` assigned to `error` becomes non-nil interface |
