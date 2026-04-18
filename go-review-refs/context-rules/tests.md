# Context Rules: Tests Agent

Before analyzing diffs, read this table. When you see a trigger pattern in the diff, load the additional context described.

| Trigger in diff | What to load | How | Why |
|---|---|---|---|
| Changed function in `internal/` or `pkg/` | Corresponding `*_test.go` file in the same package | Use File Access instructions from your prompt | Verify tests exist and match new behavior |
| New or modified error return | Test file + grep for function name in `*_test.go` files | Search repo for function name in test files, then use File Access instructions | Check if error path is tested |
| New `go` keyword or goroutine | Test file for the package | Use File Access instructions from your prompt | Check for concurrent/race test coverage |
| New `tx.Begin` or transaction code | Test file for the package | Use File Access instructions from your prompt | Check for rollback/failure path tests |
| Changed function signature (new param, changed return) | All `*_test.go` files calling that function | Search repo for function name in test files, then use File Access instructions | Verify test call sites match new signature |
| New HTTP handler or route change | Handler test file (`*_test.go` in same package) | Use File Access instructions from your prompt | Check for request/response test coverage |
| Modified struct used in tests | Test files using that struct | Search repo for struct name in test files, then use File Access instructions | Test data may use outdated field values |
| `*_test.go` file in diff with changed assertions | Implementation file being tested | Use File Access instructions from your prompt | Verify assertions match actual implementation behavior |
