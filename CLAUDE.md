# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Three Claude Code skills (not a Go application) for multi-agent Go code review. The repository is pure Markdown and JSON that defines agent prompts and orchestration logic. There is no build, test, or lint step.

## Skills

| Skill | Directory | Invocation | Source |
|-------|-----------|------------|--------|
| **golang-review** | `golang-review/` | Paste a GitLab MR URL | GitLab MR diffs + full files via MCP |
| **golang-review-branch** | `golang-review-branch/` | Target branch name (e.g., `main`) | `git diff target..HEAD` (committed changes) |
| **golang-review-local** | `golang-review-local/` | Target branch name (e.g., `main`) | `git diff target` (uncommitted changes) |

Each skill has its own `SKILL.md` (entry point, input parsing) and `workflow.md` (monolithic 5-phase execution). All three share the resources in `references/`.

## Shared References

```
references/
  agents/           # 7 sub-agent prompts (correctness, concurrency, conventions, consistency, transactions, performance, security)
  context-rules/    # Per-agent trigger patterns for loading additional files beyond the diff
  agent-output-schema.json  # JSON schema all sub-agents must conform to
  report-format.md          # Markdown template for the final report
  operations.md             # Catalog of available GitLab MCP operations
```

## Architecture

- **Monolithic workflows:** Each skill's `workflow.md` contains all 5 phases inline (Fetch, Wave 1, Wave 2, Merge, Report). This is intentional for LLM reliability -- a single document the orchestrator follows top to bottom.
- **Two-wave agent execution:** Wave 1 launches 5 agents in parallel (correctness, concurrency, conventions, performance, security). Wave 2 launches 2 agents (consistency, transactions) that receive Wave 1 output as context. All 7 agents share the same prompts and schema.
- **tmp_dir vs output_dir:** Working data (metadata.json, diffs, full files) goes to `/tmp/golang-review/...` (ephemeral). Persistent results (agent JSON reports, final-report.md) go to `docs/review/...` (committed to repo).
- **GitLab skill specifics:** `--discussions` flag (opt-in for loading MR discussions), `--only` flag (subset of agents), `additional_context` as free text. Agents fetch additional files from GitLab MCP.
- **Local skill specifics:** Agents read files directly from the repo (no copying). No MCP dependency. `--only` flag and `additional_context` supported.
- **Verdict rules:** 1+ critical = REJECT, 1+ major = REQUEST CHANGES, otherwise LGTM.

## Key File Roles

- `<skill>/SKILL.md` -- skill entry point; parses input, resolves variables, launches workflow
- `<skill>/workflow.md` -- orchestrator; defines the 5-phase execution with agent prompt templates
- `references/agents/*.md` -- sub-agent prompts with role, checklist, output rules, scope
- `references/context-rules/*.md` -- source-agnostic trigger patterns (use "File Access instructions" from the agent prompt)
- `references/agent-output-schema.json` -- JSON schema all sub-agents must conform to
- `references/report-format.md` -- Markdown template for the final report
- `references/operations.md` -- catalog of available GitLab MCP operations

## When Editing

Changes can ripple across files. Here is where to look:

| What you change | Also check |
|-----------------|------------|
| Agent prompt (`references/agents/*.md`) | All three `workflow.md` files (if the prompt template section changes), `agent-output-schema.json` (if output format changes) |
| Context rules (`references/context-rules/*.md`) | Nothing -- these are self-contained and source-agnostic |
| Output schema (`references/agent-output-schema.json`) | `references/report-format.md`, agent prompts (if field names change) |
| Report template (`references/report-format.md`) | Phase 5 in all three `workflow.md` files |
| Phase 1 (Fetch) in one skill's `workflow.md` | Only that skill -- Phase 1 is the only skill-specific phase |
| Phases 2-5 in one skill's `workflow.md` | The same phases in the other two `workflow.md` files -- they should stay in sync |
| A skill's `SKILL.md` | Only that skill's `workflow.md` (if variables change) |

## Architecture Constraints

- Wave 2 agents depend on Wave 1 output -- they must not launch until all Wave 1 agents complete
- Sub-agents must write JSON reports to `{{output_dir}}/reports/{agent_name}.json` using the Write tool, not just return them in-memory
- All review artifacts persist in `docs/review/` -- never delete review directories
- File exclusion patterns: `vendor/`, `*_mock.go`, `*.pb.go`, `*_generated.go`, `testdata/`, `*.gen.go`
- Max 20 full files fetched per review (GitLab skill only), sorted by diff size (largest first)
