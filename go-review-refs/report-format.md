# Report Format Template

Use this template to render the final review report from merged sub-agent JSON reports.

## Orchestrator contract (non-optional)

For **every** finding (all severities), pick **exactly one** presentation:

**A — Code snippets (default)**  
Use when `code_snippet_unavailable` is absent, `false`, or not set:

- Render **Before** and **After** fenced `go` blocks from `code_before` and `code_after`.
- If either string is missing, empty, or only whitespace: **reconstruct** minimal snippets from Phase 1 (`tmp_dir/diffs/` plus full file: `tmp_dir/files/` for GitLab, or repo path for branch/local) at `{file}` around `{line}`, then render those inside the fences.
- **Never** publish empty fenced `go` blocks under mode A.

**B — No applicable snippet (explicit waiver)**  
Use only when merged JSON has `code_snippet_unavailable: true`:

- Render **no** Before/After code fences for that finding.
- Render a line: **Code snippet:** not applicable — {code_absence_note}
- Do **not** reconstruct snippets over a waiver; the agent asserted no honest single-location pair exists.

**Do not** use a compact one-line format that drops both A and B (every finding must be either A or B).

Use mode B sparingly (cross-cutting design, policy-only, missing artifact not in diff, multi-file contract). **If a single line or small hunk in the Phase 1 diff at this finding's `file`/`line` would illustrate the issue, mode A is mandatory** — sub-agents must not choose mode B to avoid quoting the diff; the orchestrator should prefer reconstruction (mode A) when diff + file text supply such a line.

---

# Go Code Review | {title}
**{author}** · `{source_branch}` → `{target_branch}`

## MR Overview
**What changed:** {1-3 sentence summary of what the MR introduces or modifies}
**Why:** {1-3 sentence explanation of the goal/purpose}

## Summary

| Metric | Value |
|--------|-------|
| Files checked | {N} |
| Lines added | +{N} |
| Lines removed | -{N} |
| Critical | {N} |
| Major | {N} |
| Minor | {N} |

**Verdict:** {REJECT / REQUEST CHANGES / LGTM}

Verdict rules:
- 1+ critical → **REJECT**
- 0 critical, 1+ major → **REQUEST CHANGES**
- only minor or clean → **LGTM**

---

## Critical ({N})

Group findings by agent category. For each finding:

### {Agent Category Name}

#### [{id}] {title}
- **File:** `{file}:{line}`
- **Category:** {category}
- **Problem:** {problem — must include production impact}
- **If mode A** (`code_snippet_unavailable` not true): **Before:** / fenced `go` `{code_before}` — **After:** / fenced `go` `{code_after}` (after reconstruction if needed; see Orchestrator contract).
- **If mode B** (`code_snippet_unavailable: true`): **Code snippet:** not applicable — {code_absence_note}

---

## Major ({N})

Same branching (mode A vs B) as Critical, grouped by agent category.

---

## Minor ({N})

Same branching (mode A vs B) as Critical, grouped by agent category.

### {Agent Category Name}

#### [{id}] {title}
- **File:** `{file}:{line}`
- **Category:** {category}
- **Problem:** {problem}
- **If mode A** (`code_snippet_unavailable` not true): **Before:** / fenced `go` `{code_before}` — **After:** / fenced `go` `{code_after}` (after reconstruction if needed; see Orchestrator contract).
- **If mode B** (`code_snippet_unavailable: true`): **Code snippet:** not applicable — {code_absence_note}

---

## Open Questions

Merge `open_questions` arrays from all agent reports. Deduplicate by meaning. Cap at 10 total. Only include this section if there are open questions.

For each question, prefix with the agent name:

- **[{agent}]** {question or residual risk note}

---

## What Was Done Well

Merge `positive` arrays from all agent reports. Deduplicate. Present as bullet list:

- {positive observation from agent}

---

## Key Production Risks

Only include if critical or major findings exist. Summarize real production consequences:

- {risk description with estimated impact}

---

Notes:
- Findings with `requires_verification: true` — append "(requires verification)" to the title
- If `--only` was used, note which agents were included in the Summary section
- If an agent returned 0 findings, do not create an empty section for it
- Agent category order for grouping: correctness, concurrency, conventions, style, tests, consistency, transactions, performance, security
- **Completeness:** under mode A, every finding must end with non-empty Before and After fences (after reconstruction if needed). Under mode B, `code_absence_note` must be present and substantive — never a finding with neither fences nor a waiver line
