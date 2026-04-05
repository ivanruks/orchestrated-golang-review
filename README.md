# Golang Review — Multi-Agent Go Code Review System

## Overview

Golang Review is a Claude Code skill that automates code review for GitLab Merge Requests written in Go. The system dispatches 7 specialized sub-agents (correctness, concurrency, conventions, consistency, transactions, performance, security), each analyzing MR diffs and full files against its own checklist. Agents run in two waves: Wave 1 launches 5 agents in parallel, then Wave 2 launches 2 agents that receive Wave 1 findings as additional context. All results are merged, deduplicated, and rendered into a unified report with a verdict (LGTM / REQUEST CHANGES / REJECT).

---

## Architecture

```
User
    |
    |  <MR_URL> [--only agent1,agent2]
    v
+------------------+
|    SKILL.md      |  Entry point: URL parsing, validation, review_dir generation
+------------------+
    |
    v
+------------------+
|   workflow.md    |  Orchestrator: 5-phase execution
+------------------+
    |
    |  Phase 1: Fetch
    |  Download MR metadata, diffs, full files, discussions via GitLab MCP
    |
    v
+------------------------------------------------------------------+
|  Phase 2: Wave 1 (parallel)                                     |
|                                                                  |
|  +-------------+ +-------------+ +-------------+ +-----------+  |
|  | correctness | | concurrency | | conventions | |performance|  |
|  |   (CORR)    | |   (CONC)    | |   (CONV)    | |  (PERF)   |  |
|  +-------------+ +-------------+ +-------------+ +-----------+  |
|                                                                  |
|  +-------------+                                                 |
|  |  security   |                                                 |
|  |   (SEC)     |                                                 |
|  +-------------+                                                 |
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

## Usage

### Full review (all 7 agents)

```
https://gitlab.com/mygroup/myproject/-/merge_requests/123
```

### Selective review (specific agents only)

```
https://gitlab.com/mygroup/myproject/-/merge_requests/123 --only concurrency,security
```

### Available agents

`correctness`, `concurrency`, `conventions`, `consistency`, `transactions`, `performance`, `security`

When `--only` is used, the final report includes a note indicating a partial review.

---

## File Structure

```
golang-review/
├── SKILL.md                                # Skill entry point: URL parsing, validation, workflow launch
├── workflow.md                             # Orchestrator: 5-phase execution description
├── README.md                               # This file
├── agents/                                 # Sub-agent prompts
│   ├── correctness.md                      # Correctness agent: bugs, resource leaks, panics
│   ├── concurrency.md                      # Concurrency agent: races, deadlocks, goroutine leaks
│   ├── conventions.md                      # Conventions agent: idiomatic Go patterns
│   ├── consistency.md                      # Consistency agent: data flow integrity (Wave 2)
│   ├── transactions.md                     # Transactions agent: atomicity, idempotency (Wave 2)
│   ├── performance.md                      # Performance agent: O(n²), allocations, N+1
│   └── security.md                         # Security agent: injection, auth, cryptography
└── references/                             # Shared reference materials
    ├── agent-output-schema.json            # JSON schema for sub-agent output
    ├── report-format.md                    # Final Markdown report template
    ├── operations.md                       # GitLab MCP operations catalog
    └── context-rules/                      # Context loading trigger rules
        ├── correctness.md                  # Triggers: defer in loop, ignored errors, ...
        ├── concurrency.md                  # Triggers: go func, sync.Mutex, channels, ...
        ├── conventions.md                  # Triggers: interfaces, context.Context, naming
        ├── consistency.md                  # Triggers: changed signatures, interfaces, types
        ├── transactions.md                 # Triggers: tx.Begin, cache invalidation, saga pattern
        ├── performance.md                  # Triggers: loops, append, SQL in loop
        └── security.md                     # Triggers: user input, SQL, exec.Command
```

---

## Sub-Agents

| Agent | Prefix | Wave | Focus Area | Key Checks |
|-------|--------|------|-----------|------------|
| **correctness** | `CORR` | 1 | Bugs, resource leaks, logic errors | Nil dereference, panics, `http.Response.Body`/`sql.Rows`/`os.File` leaks, ignored errors, `errors.Is` vs `==` |
| **concurrency** | `CONC` | 1 | Data races, goroutines, mutexes | Goroutines without shutdown mechanism, map writes from multiple goroutines, lock held during I/O, channels without `ctx.Done()` |
| **conventions** | `CONV` | 1 | Idiomatic Go patterns | `fmt.Errorf` without `%w`, context in struct field, functions >60 lines, naming, godoc |
| **performance** | `PERF` | 1 | Performance and allocations | O(n²) in hot paths, `append` without pre-allocation, SQL in loop (N+1), `regexp.Compile` inside function body |
| **security** | `SEC` | 1 | Security vulnerabilities | SQL/command injection, path traversal, hardcoded credentials, `math/rand` for secrets, PII leaks in logs |
| **consistency** | `CONS` | 2 | Data flow integrity across layers | Type mismatches between layers, broken interface contracts, incomplete renames, API incompatibility |
| **transactions** | `TXN` | 2 | Atomicity, transactions, idempotency | `tx.Begin` without `defer tx.Rollback`, partial failure, missing idempotency, cache invalidation before commit |

**Wave 2 agents** (`consistency`, `transactions`) receive Wave 1 reports as additional context. This allows them to:
- Avoid duplicating already-reported findings
- Use Wave 1 findings as signals (e.g., a resource leak found by the correctness agent may indicate a transaction issue)

---

## Workflow Phases

### Phase 1: Fetch

Download MR data from GitLab via MCP:

1. Create review directory (`docs/review/<timestamp>_mr-<iid>/`)
2. Get MR metadata (`get_merge_request`)
3. Get diffs (`get_merge_request_diffs`) with filtering: `.go` files only, excluding `vendor/`, `*_mock.go`, `*.pb.go`, `*_generated.go`, `testdata/`, `*.gen.go`
4. Download full files (`get_file_contents`) — up to 20 files, sorted by diff size (largest first)
5. Get existing discussions (`mr_discussions`) — agents must not duplicate existing comments
6. Output a summary of fetched data to the user

### Phase 2: Wave 1

Parallel launch of 5 agents: `correctness`, `concurrency`, `conventions`, `performance`, `security`. Each agent:
- Reads metadata and existing discussions
- Analyzes each diff file and its corresponding full file
- Loads additional files when context-rules triggers fire
- Writes a JSON report to `reports/<agent_name>.json`

### Phase 3: Wave 2

Parallel launch of 2 agents: `consistency`, `transactions`. They receive the same input as Wave 1, plus Wave 1 agent reports for cross-referencing and deduplication.

### Phase 4: Merge

Combine all agent results:
1. Load all JSON reports
2. Deduplicate: when two agents flag the same file+line, the more specialized agent wins (`concurrency` > `correctness` for races, `transactions` > `correctness` for leaks in tx scope, `security` > `correctness` for injection, `consistency` > `conventions` for interface contracts)
3. Group: by severity (critical → major → minor), then by agent category
4. Compute statistics and determine verdict
5. Merge positive findings

### Phase 5: Report

Render the final Markdown report using the template from `references/report-format.md`, save to `final-report.md`, display to user.

---

## Inter-Agent JSON Schema

Each sub-agent returns JSON in the following format (`references/agent-output-schema.json`):

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
      "problem": "resp.Body not closed on error path — leaks TCP connection. At 100 RPS, connection pool exhausted in ~30s.",
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
| `agent` | string | Agent name (one of 7 values) |
| `files_checked` | integer | Number of files analyzed |
| `findings` | array | List of found issues |
| `positive` | array (min 1) | What was done well — at least one entry required |

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
| `code_before` | string | Current (problematic) code |
| `code_after` | string | Suggested fix |
| `requires_verification` | boolean | Whether manual verification is needed |

---

## Final Report Format

The report is rendered using the template from `references/report-format.md` and includes:

1. **Header** — MR number, title, author, branches
2. **MR Overview** — brief description of changes and their purpose
3. **Summary** — metrics table (files, lines, critical/major/minor counts) and verdict
4. **Critical** — detailed critical findings grouped by agent, with `code_before`/`code_after` blocks
5. **Major** — same structure for major findings
6. **Minor** — compact list, one line per finding
7. **What Was Done Well** — merged positive observations from all agents
8. **Key Production Risks** — production risk summary (only if critical/major findings exist)

### Verdict Rules

| Condition | Verdict |
|-----------|---------|
| 1+ critical | **REJECT** |
| 0 critical, 1+ major | **REQUEST CHANGES** |
| Only minor or clean | **LGTM** |

Findings with `requires_verification: true` are marked with "(requires verification)" in the title.

---

## Context Rules

Each agent has its own set of triggers in `references/context-rules/<agent>.md`. A trigger is a pattern in the diff that, when detected, tells the agent to load additional files for thorough analysis.

### Loading algorithm

1. Agent detects a trigger in the diff (e.g., `go func` for the concurrency agent)
2. First checks the local `files/` directory — the file may have been pre-fetched in Phase 1
3. If the file is not in `files/`, fetches it from GitLab via MCP
4. For finding related files, can use `get_repository_tree` + `get_file_contents`

### Trigger examples

| Agent | Trigger | What it loads |
|-------|---------|---------------|
| correctness | `defer` inside `for` | Full function containing the loop |
| concurrency | `go func` | Entire file to analyze goroutine lifecycle |
| concurrency | `sync.Mutex` in struct | All struct methods to check lock ordering |
| consistency | Changed exported function signature | All call sites in the repository |
| transactions | `tx.Begin()` | Full function and calling code |
| security | User input → `db.Query` | Full path from handler to DB |

---

## Review Directory (docs/review/)

An isolated directory is created for each review:

```
docs/review/2026-04-03T14-30_mr-456/
├── metadata.json          # MR metadata: project, iid, title, author, branches, discussions
├── diffs/                 # Diffs of .go files (/ replaced with -)
│   ├── internal-service-user.go.diff
│   └── pkg-api-handler.go.diff
├── files/                 # Full files from source branch, preserving directory structure
│   ├── internal/
│   │   └── service/
│   │       └── user.go
│   └── pkg/
│       └── api/
│           └── handler.go
├── reports/               # Sub-agent JSON reports
│   ├── correctness.json
│   ├── concurrency.json
│   ├── conventions.json
│   ├── performance.json
│   ├── security.json
│   ├── consistency.json
│   └── transactions.json
└── final-report.md        # Final merged report
```

### Directory name format

`docs/review/<YYYY-MM-DDTHH-MM>_mr-<iid>/`

- Timestamp in 24-hour format, dashes instead of colons
- iid — numeric MR identifier in GitLab

---

## GitLab MCP

All GitLab integration is handled through the MCP server named `user-gitlab`. The operations catalog is in `references/operations.md`.

### Operations used

| Phase | Operation | Purpose |
|-------|-----------|---------|
| Phase 1 | `get_merge_request` | Get MR metadata (title, author, branches) |
| Phase 1 | `get_merge_request_diffs` | Get diffs with file pattern filtering |
| Phase 1 | `get_file_contents` | Download full files from source branch |
| Phase 1 | `mr_discussions` | Get existing discussions (for deduplication) |
| Phase 2-3 | `get_file_contents` | Load additional context per context-rules triggers |
| Phase 2-3 | `get_repository_tree` | Search for files in repository (for consistency/transactions) |

### Excluded file patterns

```
^vendor/          — vendored dependencies
_mock\.go$        — mocks
\.pb\.go$         — generated protobuf code
_generated\.go$   — other generated code
^testdata/        — test data
\.gen\.go$        — generated code
```

### Error handling

- If GitLab MCP is unavailable during Phase 1, the review stops with an error message
- If a sub-agent fails (timeout, error), the review continues with remaining agents and a note is added to the report
- If the MR contains no `.go` files, the review stops with "No Go files changed in this MR"
