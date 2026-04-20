# Context Rules: Style Agent

Before analyzing diffs, read this table. When you see a trigger pattern in the diff, load the additional context described.

| Trigger in diff | What to load | How | Why |
|---|---|---|---|
| Package clause `package util`, `package common`, `package helpers`, or `package misc` | Other files in the package | Use File Access instructions from your prompt | Generic package names hide domain; assess rename impact |
| New or renamed file `consts.go` | Rest of package for related types | Use File Access instructions from your prompt | Constants may belong next to owning types |
| More than one import group without blank line between std and non-std | Full file import block | Use File Access instructions from your prompt | Verify grouping matches Go conventions |
| `http.NewServeMux` or method on `ServeMux` with `Handle` / `HandleFunc` | Full file with all route registrations | Use File Access instructions from your prompt | Detect overlapping patterns and missing `$` anchors |
| Route pattern string containing `{` (Go 1.22+ wildcard) | Adjacent registrations in same mux | Use File Access instructions from your prompt | Prefix vs full-path conflicts |
| Parameter or field type `*` immediately before interface name (e.g. `*io.Reader`, `*any`) | Full function signature and call sites | Use File Access instructions from your prompt | Pointer-to-interface is almost always wrong |
| Map value type `any` or `interface{}` and stored values are structs (or other concrete types) later used with interface method calls | Type definition; list methods with value vs pointer receivers | Use File Access instructions from your prompt | Value stored in `any`/`interface{}` can bind to wrong method set vs pointer receiver |
| Struct literal with many fields set in separate statements after `var` | Full function building the struct | Use File Access instructions from your prompt | Prefer keyed composite literal when safety/readability at stake |
| `return []T{}` or `return make([]T, 0)` in API-style function | Other return paths in same function / package | Use File Access instructions from your prompt | Compare with idiomatic `nil` slice returns |
