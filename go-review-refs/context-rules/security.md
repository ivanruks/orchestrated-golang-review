# Context Rules: Security Agent

When trigger seen in diff, load context described. **How**: File Access instructions from agent prompt; search repo when noted.

| Trigger | Load | Why |
|---|---|---|
| `fmt.Sprintf` near SQL / `db.Query`/`db.Exec` | Full query builder/repo file | User input reaches query via Sprintf? Must parameterize |
| `os/exec.Command` / `exec.CommandContext` | Full handler chain — arg origin | Command injection if unsanitized |
| `filepath.Join`/`os.Open` with variable path | HTTP handler receiving path | Path traversal — `filepath.Clean` + base validation? |
| Hardcoded string looking like token/password/key | N/A (local) | Credentials from env/config, never hardcoded |
| `md5`/`sha1` for password/auth | Full auth module | Must use bcrypt/argon2 |
| `crypto/rand` vs `math/rand` | Full file | Security-sensitive → crypto/rand |
| `http.ListenAndServe` (not TLS) | Main/server setup | TLS at proxy level? |
| Integer arithmetic in money/size/limit | Full function | Overflow can bypass security checks |
| `template.HTML`/`template.JS` | Full handler | XSS — unescaped user input |
| `http.Get`/`Post`/`NewRequest` with variable URL | Full handler chain — URL origin | SSRF if user-controlled |
| `http.Redirect` with variable URL | Full handler — redirect URL origin | Open redirect → phishing |
