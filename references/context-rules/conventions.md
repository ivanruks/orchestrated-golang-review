# Context Rules: Conventions Agent

Before analyzing diffs, read this table. When you see a trigger pattern in the diff, load the additional context described.

| Trigger in diff | What to load | How | Why |
|---|---|---|---|
| New exported type or function | Usage sites in the same package and callers | Search repo, then use File Access instructions | Verify naming follows Go conventions, godoc present |
| Interface definition | Packages that consume this interface | Use File Access instructions from your prompt | Interface should be defined on consumer side, not implementer side |
| `context.Context` stored in struct field | All methods of that struct | Use File Access instructions from your prompt | Context must be passed as first argument, never stored in struct |
| Error returned with `fmt.Errorf` without `%w` | Callers that check the error | Use File Access instructions from your prompt | Without %w, callers cannot use errors.Is/errors.As |
| Error string starting with uppercase or ending with period | N/A (local check) | N/A | Go convention: error strings are lowercase, no trailing punctuation |
| `init()` function | Full file | Use File Access instructions from your prompt | init() should only be used for package-level registration, never business logic |
| Function with >4 parameters | N/A (local check) | N/A | Consider options struct pattern |
| Exported function returning unexported type | Full file | Use File Access instructions from your prompt | Exported API cannot use unexported types |
