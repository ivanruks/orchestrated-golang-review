# Context Rules: Performance Agent

Before analyzing diffs, read this table. When you see a trigger pattern in the diff, load the additional context described.

| Trigger in diff | What to load | How | Why |
|---|---|---|---|
| `append` in a loop without pre-allocation | Caller of the function to estimate data size | Use File Access instructions from your prompt | If N is large, missing `make([]T, 0, N)` causes repeated allocations |
| Nested loop over collections | Full function + type definitions of collections | Use File Access instructions from your prompt | Check if O(n²) can be reduced to O(n) with a map |
| String concatenation with `+` inside loop | Full function | Use File Access instructions from your prompt | Use `strings.Builder` — each `+` allocates a new string |
| SQL query in a loop | Full function + DB schema/migration files | Use File Access instructions from your prompt | N+1 query problem — consider batch query or JOIN |
| `json.Marshal`/`json.Unmarshal` in hot path | Full function + callers | Use File Access instructions from your prompt | Consider `json.Encoder`/`json.Decoder` for streams, or code generation |
| Large struct copied by value (>5 fields) | Struct definition + call sites | Use File Access instructions from your prompt | Consider pointer receiver/parameter to avoid copy cost |
| `regexp.Compile` inside function (not package-level) | Full file | Use File Access instructions from your prompt | Compile once at package level, not per call |
| `map` creation without size hint | Full function to estimate entry count | Use File Access instructions from your prompt | `make(map[K]V, n)` avoids rehashing |
| `defer` inside `for` loop | Full function to estimate iteration count | Use File Access instructions from your prompt | ~50ns overhead per defer, significant at millions of iterations |
| `reflect.` in non-test code | Full function + callers to assess hot path | Use File Access instructions from your prompt | Reflection is 10-100x slower than direct access |
