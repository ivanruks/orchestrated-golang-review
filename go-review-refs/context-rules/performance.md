# Context Rules: Performance Agent

When trigger seen in diff, load context described. **How**: File Access instructions from agent prompt.

| Trigger | Load | Why |
|---|---|---|
| `append` in loop without pre-alloc | Caller (estimate data size) | Large N → repeated allocations without `make([]T, 0, N)` |
| Nested loop over collections | Full func + collection type defs | O(n²) reducible to O(n) with map? |
| String `+` inside loop | Full function | Use `strings.Builder` — each `+` allocates |
| SQL query in loop | Full func + DB schema/migrations | N+1 — batch query or JOIN |
| `json.Marshal`/`Unmarshal` in hot path | Full func + callers | Consider streaming encoder/decoder or codegen |
| Large struct (>5 fields) copied by value | Struct def + call sites | Pointer avoids copy cost |
| `regexp.Compile` inside func | Full file | Compile once at package level |
| `map` creation without size hint | Full func (estimate entry count) | `make(map[K]V, n)` avoids rehashing |
| `defer` inside `for` loop | Full func (estimate iteration count) | ~50ns per defer, significant at millions |
| `reflect.` in non-test code | Full func + callers (hot path?) | 10-100x slower than direct access |
