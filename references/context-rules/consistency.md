# Context Rules: Consistency Agent

Before analyzing diffs, read this table. When you see a trigger pattern in the diff, load the additional context described.

| Trigger in diff | What to load | How | Why |
|---|---|---|---|
| Call to function from another package | File containing the function definition | Use File Access instructions from your prompt | Verify argument types, return types, and error contracts match |
| Changed signature of exported function | All call sites in the repository | Search repo for function name, then use File Access instructions | Callers must be updated to match new signature |
| Changed or new interface | All types implementing the interface | Search repo for interface name, then use File Access instructions | All implementations must satisfy the new contract |
| Added/removed/renamed struct field | All files using that struct | Search repo for struct name, then use File Access instructions | All usages must reflect the field change |
| gRPC/protobuf method call | `.proto` file defining the service | Use File Access instructions from your prompt | Verify Go code matches proto contract |
| Changed error type or sentinel error | All callers checking for that error | Search for error name, then use File Access instructions | Callers using errors.Is/errors.As must handle new error type |
| Changed HTTP handler route or method | Router/mux registration and middleware chain | Use File Access instructions from your prompt | Verify route, method, and middleware are consistent |
| Changed config struct or env var name | Config loading code + all readers | Use File Access instructions from your prompt | Config name must match between definition, loading, and usage |
