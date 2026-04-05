# Context Rules: Consistency Agent

Before analyzing diffs, read this table. When you see a trigger pattern in the diff, load the additional context described.

| Trigger in diff | What to load | How (MCP operation) | Why |
|---|---|---|---|
| Call to function from another package | File containing the function definition | `get_file_contents` (check `files/` first, then MCP) | Verify argument types, return types, and error contracts match |
| Changed signature of exported function | All call sites in the repository | Search repo for function name, then `get_file_contents` | Callers must be updated to match new signature |
| Changed or new interface | All types implementing the interface | Search repo for interface name, then `get_file_contents` | All implementations must satisfy the new contract |
| Added/removed/renamed struct field | All files using that struct | Search repo for struct name, then `get_file_contents` | All usages must reflect the field change |
| gRPC/protobuf method call | `.proto` file defining the service | `get_file_contents` for the .proto | Verify Go code matches proto contract |
| Changed error type or sentinel error | All callers checking for that error | Search for error name, then `get_file_contents` | Callers using errors.Is/errors.As must handle new error type |
| Changed HTTP handler route or method | Router/mux registration and middleware chain | `get_file_contents` for router file | Verify route, method, and middleware are consistent |
| Changed config struct or env var name | Config loading code + all readers | `get_file_contents` | Config name must match between definition, loading, and usage |
