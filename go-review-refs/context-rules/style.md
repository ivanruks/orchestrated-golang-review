# Context Rules: Style Agent

When trigger seen in diff, load context described. **How**: File Access instructions from agent prompt.

| Trigger | Load | Why |
|---|---|---|
| `package util`/`common`/`helpers`/`misc` | Other files in package | Generic name hides domain; assess rename impact |
| New/renamed `consts.go` | Rest of package | Constants may belong next to owning types |
| >1 import group without std/non-std blank line | Full import block | Verify grouping matches Go conventions |
| `http.NewServeMux` or `Handle`/`HandleFunc` | Full file w/ all route registrations | Overlapping patterns? Missing `$` anchors? |
| Route pattern with `{` (Go 1.22+ wildcard) | Adjacent registrations in same mux | Prefix vs full-path conflicts |
| `*` before interface name (`*io.Reader`, `*any`) | Full func signature + call sites | Pointer-to-interface almost always wrong |
| Map value `any`/`interface{}` storing structs used w/ interface methods | Type def; value vs pointer receivers | Value in `any` can bind wrong method set |
| Struct literal built field-by-field after `var` | Full func building struct | Prefer keyed composite literal |
| `return []T{}`/`return make([]T, 0)` in API func | Other return paths in func/package | Compare with idiomatic `nil` slice |
