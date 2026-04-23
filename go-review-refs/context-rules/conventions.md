# Context Rules: Conventions Agent

When trigger seen in diff, load context described. **How**: File Access instructions from agent prompt; search repo when noted.

| Trigger | Load | Why |
|---|---|---|
| New exported type/function | Usage sites + callers (search repo) | Naming follows Go conventions? Godoc present? |
| Interface definition | Consuming packages | Interface at consumer side, not implementer |
| `context.Context` stored in struct | All methods of struct | Must pass as first arg, never store |
| `fmt.Errorf` without `%w` | Callers checking error | Without %w, can't `errors.Is`/`errors.As` |
| Error string uppercase or ends with period | N/A (local) | Go: lowercase, no trailing punctuation |
| `init()` function | Full file | Only for package-level registration |
| Function >4 params | N/A (local) | Consider options struct |
| Exported func returning unexported type | Full file | Exported API can't use unexported types |
