# Context Rules: Performance Agent

Before analyzing diffs, read this table. When you see a trigger pattern in the diff, load the additional context described.

| Trigger in diff | What to load | How (MCP operation) | Why |
|---|---|---|---|
| `append` in a loop without pre-allocation | Caller of the function to estimate data size | `get_file_contents` for callers | If N is large, missing `make([]T, 0, N)` causes repeated allocations |
| Nested loop over collections | Full function + type definitions of collections | `get_file_contents` | Check if O(n²) can be reduced to O(n) with a map |
| String concatenation with `+` inside loop | Full function | `get_file_contents` | Use `strings.Builder` — each `+` allocates a new string |
| SQL query in a loop | Full function + DB schema/migration files | `get_file_contents` | N+1 query problem — consider batch query or JOIN |
| `json.Marshal`/`json.Unmarshal` in hot path | Full function + callers | `get_file_contents` | Consider `json.Encoder`/`json.Decoder` for streams, or code generation |
| Large struct copied by value (>5 fields) | Struct definition + call sites | `get_file_contents` | Consider pointer receiver/parameter to avoid copy cost |
| `regexp.Compile` inside function (not package-level) | Full file | `get_file_contents` | Compile once at package level, not per call |
| `map` creation without size hint | Full function to estimate entry count | `get_file_contents` | `make(map[K]V, n)` avoids rehashing |
