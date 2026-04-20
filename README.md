# Go Review -- Multi-Agent Go Code Review System

## Overview

Go Review is a set of three Claude Code skills that automate code review for Go code. Each skill targets a different source: GitLab Merge Requests, committed branch changes, or uncommitted local changes. All three dispatch 9 specialized sub-agents (correctness, concurrency, conventions, style, tests, consistency, transactions, performance, security), each analyzing diffs and full files against its own checklist. Agents run in two waves: Wave 1 launches 7 agents in parallel, then Wave 2 launches 2 agents that receive Wave 1 findings as additional context. All results are merged, deduplicated, and rendered into a unified report with a verdict (LGTM / REQUEST CHANGES / REJECT).

---

## Architecture

```
User
    |
    |  Three entry points
    |
    +--- GitLab MR URL -------> go-review/SKILL.md
    |                               |
    +--- target branch name --> go-review-branch/SKILL.md
    |                               |
    +--- target branch name --> go-review-local/SKILL.md
                                    |
                                    v
                            +------------------+
                            | <skill>/         |  Each skill has its own
                            | workflow.md      |  monolithic 5-phase workflow
                            +------------------+
                                    |
         +--------------------------+---------------------------+
         |                          |                           |
    Phase 1: Fetch           Phase 2-3: Agents            Phase 4-5: Merge & Report
    (skill-specific)         (shared logic)               (shared logic)
         |                          |                           |
         v                          v                           v
    tmp_dir (/tmp/...)     go-review-refs/agents/*.md         output_dir (docs/review/...)
    - metadata.json        go-review-refs/context-rules/      - reports/*.json
    - diffs/               go-review-refs/agent-output-        - final-report.md
    - files/ (GitLab only)   schema.json

+------------------------------------------------------------------+
|  Phase 2: Wave 1 (parallel)                                     |
|                                                                  |
|  +-------------+ +-------------+ +-------------+ +-----------+  |
|  | correctness | | concurrency | | conventions | |performance|  |
|  |   (CORR)    | |   (CONC)    | |   (CONV)    | |  (PERF)   |  |
|  +-------------+ +-------------+ +-------------+ +-----------+  |
|                                                                  |
|  +-------------+ +-------------+ +-------------+                 |
|  |   style     | |  security   | |    tests    |                 |
|  |   (STYL)    | |   (SEC)     | |   (TEST)    |                 |
|  +-------------+ +-------------+ +-------------+                 |
+------------------------------------------------------------------+
    |
    |  All Wave 1 agents complete, JSON reports saved
    v
+------------------------------------------------------------------+
|  Phase 3: Wave 2 (parallel, with Wave 1 context)                |
|                                                                  |
|  +-------------+ +--------------+                                |
|  | consistency | | transactions |                                |
|  |   (CONS)    | |    (TXN)     |                                |
|  +-------------+ +--------------+                                |
+------------------------------------------------------------------+
    |
    |  Phase 4: Merge
    |  Deduplicate, sort by severity, compute statistics
    |
    |  Phase 5: Report
    |  Render final Markdown report
    |
    v
+------------------+
| final-report.md  |  Final report with verdict
+------------------+
```

---

## Skills

### go-review -- GitLab MR

Reviews a GitLab Merge Request. Fetches diffs and full files from GitLab via MCP.

```
<MR_URL>
<MR_URL> --discussions
<MR_URL> --only agent1,agent2,...
<MR_URL> --only concurrency,security --discussions
<MR_URL> Ticket PROJ-456: new registration endpoint writing to users and audit_log.
```

Flags:
- `--only agent1,agent2,...` -- run only the specified agents (partial review)
- `--discussions` -- load existing MR discussions so agents avoid duplicating them
- Free text after the URL/flags is passed to agents as `additional_context`

### go-review-branch -- Committed Branch Changes

Reviews all commits on the current branch relative to a target branch (`git diff target..HEAD`).

```
main
main --only concurrency,security
develop --only correctness,transactions
main Refactoring the repository layer, switching to sqlc.
```

Flags:
- `--only agent1,agent2,...` -- run only the specified agents
- Free text after the target branch/flags is passed to agents as `additional_context`

### go-review-local -- Uncommitted Local Changes

Reviews uncommitted changes (staged + unstaged, tracked files only) relative to a target branch (`git diff target`).

```
main
main --only concurrency,security
develop --only correctness,transactions
main Adding email validation and username uniqueness check.
```

Flags:
- `--only agent1,agent2,...` -- run only the specified agents
- Free text after the target branch/flags is passed to agents as `additional_context`

### Available Agents

`correctness`, `concurrency`, `conventions`, `style`, `tests`, `consistency`, `transactions`, `performance`, `security`

When `--only` is used, the final report includes a note indicating a partial review.

---

## File Structure

```
orchestrated-go-review/
├── go-review/                     # Skill: GitLab MR review
│   ├── SKILL.md                       #   Entry point: URL parsing, flag parsing, path generation
│   └── workflow.md                    #   Monolithic 5-phase workflow (fetches from GitLab MCP)
├── go-review-branch/              # Skill: committed branch changes review
│   ├── SKILL.md                       #   Entry point: target branch validation, path generation
│   └── workflow.md                    #   Monolithic 5-phase workflow (diffs from git)
├── go-review-local/               # Skill: uncommitted local changes review
│   ├── SKILL.md                       #   Entry point: target branch validation, uncommitted check
│   └── workflow.md                    #   Monolithic 5-phase workflow (diffs from git working tree)
├── go-review-refs/                        # Shared resources used by all three skills
│   ├── agents/                        #   Sub-agent prompts
│   │   ├── correctness.md             #     Bugs, resource leaks, panics
│   │   ├── concurrency.md             #     Races, deadlocks, goroutine leaks
│   │   ├── conventions.md             #     Idiomatic Go patterns
│   │   ├── style.md                   #     Naming, imports, routing idioms, interface style
│   │   ├── tests.md                   #     Test adequacy, missing coverage
│   │   ├── consistency.md             #     Data flow integrity (Wave 2)
│   │   ├── transactions.md            #     Atomicity, idempotency (Wave 2)
│   │   ├── performance.md             #     O(n^2), allocations, N+1
│   │   └── security.md               #     Injection, auth, cryptography
│   ├── context-rules/                 #   Source-agnostic trigger rules for loading extra files
│   │   ├── correctness.md
│   │   ├── concurrency.md
│   │   ├── conventions.md
│   │   ├── style.md
│   │   ├── tests.md
│   │   ├── consistency.md
│   │   ├── transactions.md
│   │   ├── performance.md
│   │   └── security.md
│   ├── agent-output-schema.json       #   JSON schema for sub-agent output
│   ├── report-format.md               #   Final Markdown report template
│   └── operations.md                  #   GitLab MCP operations catalog
├── docs/
│   └── review/                        #   Persistent review output (final reports + agent JSON)
├── CLAUDE.md                          #   Guidance for Claude Code
├── README.md                          #   This file
└── LICENSE                            #   MIT license
```

---

## Sub-Agents

| Agent | Prefix | Wave | Focus Area | Key Checks |
|-------|--------|------|-----------|------------|
| **correctness** | `CORR` | 1 | Bugs, resource leaks, logic errors | Nil dereference, panics, `http.Response.Body`/`sql.Rows`/`os.File` leaks, ignored errors, `errors.Is` vs `==` |
| **concurrency** | `CONC` | 1 | Data races, goroutines, mutexes | Goroutines without shutdown mechanism, map writes from multiple goroutines, lock held during I/O, channels without `ctx.Done()` |
| **conventions** | `CONV` | 1 | Idiomatic Go patterns | `fmt.Errorf` without `%w`, context in struct field, functions >60 lines, naming, godoc |
| **style** | `STYL` | 1 | Idioms and presentation | Package/file naming, import grouping, `http.ServeMux` patterns, pointer-to-interface, nil vs empty slices |
| **performance** | `PERF` | 1 | Performance and allocations | O(n^2) in hot paths, `append` without pre-allocation, SQL in loop (N+1), `regexp.Compile` inside function body |
| **security** | `SEC` | 1 | Security vulnerabilities | SQL/command injection, path traversal, hardcoded credentials, `math/rand` for secrets, PII leaks in logs |
| **tests** | `TEST` | 1 | Test adequacy and coverage gaps | Missing tests for new error paths, outdated assertions after behavior change, untested concurrent/tx paths |
| **consistency** | `CONS` | 2 | Data flow integrity across layers | Type mismatches between layers, broken interface contracts, incomplete renames, API incompatibility |
| **transactions** | `TXN` | 2 | Atomicity, transactions, idempotency | `tx.Begin` without `defer tx.Rollback`, partial failure, missing idempotency, cache invalidation before commit |

**Wave 2 agents** (`consistency`, `transactions`) receive Wave 1 reports as additional context. This allows them to:
- Avoid duplicating already-reported findings
- Use Wave 1 findings as signals (e.g., a resource leak found by the correctness agent may indicate a transaction issue)

---

## Workflow Phases

Each skill runs 5 phases sequentially. Phase 1 is skill-specific; Phases 2-5 are identical across all three skills.

### Phase 1: Fetch (skill-specific)

| Skill | Source | What it creates |
|-------|--------|----------------|
| **go-review** | GitLab MCP (`get_merge_request`, `get_merge_request_diffs`, `get_file_contents`, optionally `mr_discussions`) | `tmp_dir/metadata.json`, `tmp_dir/diffs/`, `tmp_dir/files/` |
| **go-review-branch** | `git diff target..HEAD -- '*.go'` | `tmp_dir/metadata.json`, `tmp_dir/diffs/` |
| **go-review-local** | `git diff target -- '*.go'` | `tmp_dir/metadata.json`, `tmp_dir/diffs/` |

Working data goes to `tmp_dir` (`/tmp/go-review/...`), which is ephemeral. The `output_dir` (`docs/review/...`) is created for persistent results only.

File filtering applies to all skills: `.go` files only, excluding `vendor/`, `*_mock.go`, `*.pb.go`, `*_generated.go`, `testdata/`, `*.gen.go`. The GitLab skill fetches up to 20 full files sorted by diff size (largest first).

### Phase 2: Wave 1

Parallel launch of 7 agents: `correctness`, `concurrency`, `conventions`, `style`, `performance`, `security`, `tests`. Each agent:
- Reads metadata and diffs from `tmp_dir`
- Accesses full source files (GitLab skill: from `tmp_dir/files/` or via MCP; local skills: directly from the repo)
- Loads additional files when context-rules triggers fire
- Writes a JSON report to `output_dir/reports/<agent_name>.json`

### Phase 3: Wave 2

Parallel launch of 2 agents: `consistency`, `transactions`. They receive the same input as Wave 1, plus Wave 1 agent reports from `output_dir/reports/` for cross-referencing and deduplication.

### Phase 4: Merge

Combine all agent results:
1. Load all JSON reports from `output_dir/reports/`
2. Deduplicate: when two agents flag the same file+line, the more specialized agent wins (`concurrency` > `correctness` for races, `transactions` > `correctness` for leaks in tx scope, `security` > `correctness` for injection, `consistency` > `conventions` for interface contracts, `conventions` > `style` for naming overlap, `correctness` > `style` for pointer-to-interface)
3. Group: by severity (critical -> major -> minor), then by agent category
4. Compute statistics and determine verdict
5. Merge positive findings

### Phase 5: Report

Render the final Markdown report using the template from `go-review-refs/report-format.md`, save to `output_dir/final-report.md`, display to user.

---

## Directory Layout at Runtime

### tmp_dir (ephemeral: /tmp/go-review/...)

```
/tmp/go-review/2026-04-03T14-30_mr-456/       # GitLab MR
/tmp/go-review/2026-04-03T14-30_branch-feat/   # Branch review
/tmp/go-review/2026-04-03T14-30_local-feat/    # Local review
    ├── metadata.json          # Review metadata: title, author, branches, discussions, additional_context
    ├── diffs/                 # Per-file diffs (.go files, / replaced with -)
    │   ├── internal-service-user.go.diff
    │   └── pkg-api-handler.go.diff
    └── files/                 # Full files from source branch (GitLab skill only)
        ├── internal/
        │   └── service/
        │       └── user.go
        └── pkg/
            └── api/
                └── handler.go
```

### output_dir (persistent: docs/review/...)

```
docs/review/2026-04-03T14-30_mr-456/
    ├── reports/               # Sub-agent JSON reports
    │   ├── correctness.json
    │   ├── concurrency.json
    │   ├── conventions.json
    │   ├── tests.json
    │   ├── performance.json
    │   ├── security.json
    │   ├── consistency.json
    │   └── transactions.json
    └── final-report.md        # Final merged report with verdict
```

---

## Inter-Agent JSON Schema

Each sub-agent returns JSON in the following format (`go-review-refs/agent-output-schema.json`):

```json
{
  "agent": "correctness",
  "files_checked": 5,
  "findings": [
    {
      "id": "CORR-1",
      "severity": "critical",
      "title": "HTTP response body not closed",
      "file": "internal/client/api.go",
      "line": 42,
      "category": "Resource Leak",
      "problem": "resp.Body not closed on error path -- leaks TCP connection. At 100 RPS, connection pool exhausted in ~30s.",
      "code_before": "resp, err := client.Do(req)\nif err != nil {\n    return nil, err\n}",
      "code_after": "resp, err := client.Do(req)\nif err != nil {\n    return nil, err\n}\ndefer resp.Body.Close()",
      "requires_verification": false
    }
  ],
  "positive": [
    "Correct use of errgroup for goroutine lifecycle management",
    "All SQL queries use parameterized arguments"
  ]
}
```

### Required fields

| Field | Type | Description |
|-------|------|-------------|
| `agent` | string | Agent name (one of 8 values) |
| `files_checked` | integer | Number of files analyzed |
| `findings` | array | List of found issues |
| `positive` | array (min 1) | What was done well -- at least one entry required |
| `open_questions` | array (max 5, optional) | Unresolved questions or risk areas the agent could not fully verify |

### Finding fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Format: `PREFIX-N` (e.g., `CONC-3`, `SEC-1`) |
| `severity` | string | `critical`, `major`, or `minor` |
| `title` | string | Short issue title |
| `file` | string | File path |
| `line` | integer | Line number |
| `category` | string | Subcategory (e.g., "Data Race", "Resource Leak") |
| `problem` | string | Issue description with production impact |
| `code_before` | string | Current (problematic) code for the report Before block (or `""` when waiver applies) |
| `code_after` | string | Suggested fix for the After block (or `""` when waiver applies) |
| `code_snippet_unavailable` | boolean (optional) | When `true`, no Before/After; use empty `code_before`/`code_after` and required `code_absence_note` |
| `code_absence_note` | string (optional) | Required with waiver: why no code snippet (English, min length per schema) |
| `requires_verification` | boolean | Whether manual verification is needed |

---

## Final Report Format

The report is rendered using the template from `go-review-refs/report-format.md` and includes:

1. **Header** -- title, author, branches
2. **Overview** -- brief description of changes and their purpose
3. **Summary** -- metrics table (files, lines, critical/major/minor counts) and verdict
4. **Critical** -- detailed critical findings grouped by agent, with `code_before`/`code_after` blocks
5. **Major** -- same structure for major findings
6. **Minor** -- same as critical/major: default is Before/After Go fences; if `code_snippet_unavailable`, a **Code snippet: not applicable** line with `code_absence_note` instead of fences
7. **Open Questions** -- unresolved questions and residual risks from all agents (only if present)
8. **What Was Done Well** -- merged positive observations from all agents
9. **Key Production Risks** -- production risk summary (only if critical/major findings exist)

### Verdict Rules

| Condition | Verdict |
|-----------|---------|
| 1+ critical | **REJECT** |
| 0 critical, 1+ major | **REQUEST CHANGES** |
| Only minor or clean | **LGTM** |

Findings with `requires_verification: true` are marked with "(requires verification)" in the title.

---

## Context Rules

Each agent has its own set of triggers in `go-review-refs/context-rules/<agent>.md`. A trigger is a pattern in the diff that, when detected, tells the agent to load additional files for thorough analysis. Context rules are source-agnostic -- they reference "File Access instructions from your prompt" rather than hardcoding GitLab MCP or local file reads.

### Loading algorithm

1. Agent detects a trigger in the diff (e.g., `go func` for the concurrency agent)
2. Uses the file access method from its prompt (GitLab skill: check `files/` then fetch via MCP; local skills: read directly from repo)
3. For finding related files, uses the discovery method from its prompt (GitLab skill: `get_repository_tree` via MCP; local skills: Glob/Grep tools)

### Trigger examples

| Agent | Trigger | What it loads |
|-------|---------|---------------|
| correctness | `defer` inside `for` | Full function containing the loop |
| concurrency | `go func` | Entire file to analyze goroutine lifecycle |
| concurrency | `sync.Mutex` in struct | All struct methods to check lock ordering |
| consistency | Changed exported function signature | All call sites in the repository |
| transactions | `tx.Begin()` | Full function and calling code |
| security | User input -> `db.Query` | Full path from handler to DB |

---

## Error Handling

- **GitLab MCP unavailable** (go-review only): review stops with an error message
- **Target branch not found** (branch/local skills): review stops with an error message
- **No `.go` files** in the diff: review stops with an appropriate message
- **Sub-agent failure** (timeout, error): review continues with remaining agents; a note is added to the report
- **All Wave 1 agents fail**: review stops; Wave 2 is not launched
- **Invalid sub-agent JSON**: agent is skipped; noted in the report

---

## License

[MIT](LICENSE)
