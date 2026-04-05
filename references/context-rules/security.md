# Context Rules: Security Agent

Before analyzing diffs, read this table. When you see a trigger pattern in the diff, load the additional context described.

| Trigger in diff | What to load | How (MCP operation) | Why |
|---|---|---|---|
| `fmt.Sprintf` near SQL query or `db.Query`/`db.Exec` | Full query builder / repository file | `get_file_contents` | Check if user input reaches query via Sprintf — must use parameterized queries |
| `os/exec.Command` or `exec.CommandContext` | Full handler chain — where does the command argument come from? | `get_file_contents` for handler + service | Command injection if user input is unsanitized |
| `filepath.Join` or `os.Open` with variable path | HTTP handler or function receiving the path | `get_file_contents` | Path traversal — check for `filepath.Clean` + base path validation |
| Hardcoded string looking like token/password/key | N/A (local check) | N/A | Credentials must come from env/config, never hardcoded |
| `md5` or `sha1` used for password/auth | Full auth module | `get_file_contents` | Must use bcrypt/argon2 for passwords |
| `crypto/rand` vs `math/rand` | Full file | `get_file_contents` | Security-sensitive randomness must use crypto/rand |
| `http.ListenAndServe` (not TLS) | Main/server setup | `get_file_contents` | Check if TLS termination happens at proxy level |
| Integer arithmetic in money/size/limit calculations | Full function | `get_file_contents` | Integer overflow can bypass security checks |
| `template.HTML` or `template.JS` | Full handler | `get_file_contents` | XSS risk — unescaped user input in templates |
