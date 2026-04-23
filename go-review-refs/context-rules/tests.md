# Context Rules: Tests Agent

When trigger seen in diff, load context described. **How**: File Access instructions from agent prompt; search repo when noted.

| Trigger | Load | Why |
|---|---|---|
| Changed function in `internal/`/`pkg/` | Corresponding `*_test.go` in same package | Tests exist + match new behavior? |
| New/modified error return | Test file + grep func name in `*_test.go` (search repo) | Error path tested? |
| New `go` keyword or goroutine | Test file for package | Concurrent/race coverage? |
| New `tx.Begin` or transaction code | Test file for package | Rollback/failure path tested? |
| Changed function signature | All `*_test.go` calling func (search repo) | Test call sites match new signature? |
| New HTTP handler or route change | Handler `*_test.go` in same package | Request/response coverage? |
| Modified struct used in tests | Test files using struct (search repo) | Test data may use outdated field values |
| `*_test.go` with changed assertions | Implementation file being tested | Assertions match actual behavior? |
