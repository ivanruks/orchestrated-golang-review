# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Claude Code skill (not a Go application) that performs multi-agent code review on GitLab Merge Requests containing Go code. There is no build, test, or lint step — the repository is pure Markdown and JSON that defines agent prompts and orchestration logic.

## How It Works

The skill is invoked via `/golang-review` or by pasting a GitLab MR URL. Entry point is `SKILL.md`, which parses the URL and delegates to `workflow.md`.

**Execution flow:** 5 phases run sequentially:
1. **Fetch** — downloads MR metadata, diffs (.go only), full files, and existing discussions from GitLab via `user-gitlab` MCP server
2. **Wave 1** — launches 5 agents in parallel: correctness (CORR), concurrency (CONC), conventions (CONV), performance (PERF), security (SEC)
3. **Wave 2** — launches 2 agents in parallel that receive Wave 1 results: consistency (CONS), transactions (TXN)
4. **Merge** — deduplicates findings (specialized agent wins on same file+line), sorts by severity, computes verdict
5. **Report** — renders final Markdown using `references/report-format.md` template

**Verdict rules:** 1+ critical = REJECT, 1+ major = REQUEST CHANGES, otherwise LGTM.

## Key File Roles

- `SKILL.md` — skill entry point; parses MR URL and `--only` flag, generates review_dir path, launches workflow
- `workflow.md` — orchestrator; defines the 5-phase execution with exact MCP calls and agent prompt templates
- `agents/*.md` — each file is a complete sub-agent prompt with role, checklist, output rules, and scope
- `references/context-rules/*.md` — trigger patterns that tell agents when to load additional files beyond the diff
- `references/agent-output-schema.json` — JSON schema all sub-agents must conform to
- `references/report-format.md` — Markdown template for the final report
- `references/operations.md` — catalog of available GitLab MCP operations

## Architecture Constraints

- Wave 2 agents (consistency, transactions) depend on Wave 1 output — they must not launch until all Wave 1 agents complete
- Sub-agents must write JSON reports to disk at `{review_dir}/reports/{agent_name}.json` using the Write tool, not just return them in-memory
- All review artifacts persist in `docs/review/<timestamp>_mr-<iid>/` — never delete review directories
- File exclusion patterns: `vendor/`, `*_mock.go`, `*.pb.go`, `*_generated.go`, `testdata/`, `*.gen.go`
- Max 20 full files fetched per review, sorted by diff size (largest first)

## When Editing Agent Prompts

Each agent prompt in `agents/` follows a strict structure: Role, ID Prefix, Checklist (with checkboxes), Output rules, Scope, Context Loading. Wave 2 agents also have a Wave section. Changes to one agent may require updating `workflow.md` (if the prompt template changes), `references/agent-output-schema.json` (if output format changes), or `references/report-format.md` (if report rendering changes).
