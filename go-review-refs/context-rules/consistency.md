# Context Rules: Consistency Agent

When trigger seen in diff, load context described. **How**: File Access instructions from agent prompt; search repo when noted.

| Trigger | Load | Why |
|---|---|---|
| Call to func from another package | File w/ func definition | Arg types, return types, error contracts match? |
| Changed exported func signature | All call sites (search repo) | Callers must match new signature |
| Changed/new interface | All implementing types (search repo) | All implementations satisfy new contract? |
| Added/removed/renamed struct field | All files using struct (search repo) | All usages reflect field change? |
| gRPC/protobuf method call | `.proto` file defining service | Go code matches proto contract? |
| Changed error type/sentinel | All callers checking error (search repo) | `errors.Is`/`errors.As` callers handle new type? |
| Changed HTTP handler route/method | Router/mux registration + middleware chain | Route, method, middleware consistent? |
| Changed config struct/env var name | Config loading + all readers | Name matches between definition, loading, usage? |
| File imports both `database/sql` and `net/http` | Full file imports + package name | Mixing persistence + transport? |
| SQL calls in non-storage package | Package structure (search repo for storage/repo packages) | SQL belongs in storage layer |
| Handler imports storage/repository | Import list + call sites | Should go through service layer |
| Setter/constructor taking `[]T`/`map[K]V` assigning to field | Callers | Aliasing: caller mutation leaks into stored state without copy |
| Getter returning `[]T`/`map[K]V` with `Lock`/`RLock` | Full struct def + getter body | Exposes internal collections — bypasses sync without copy |
