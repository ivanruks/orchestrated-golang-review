# Report Format Template

Render final report from merged sub-agent JSON reports.

## Orchestrator contract (non-optional)

For **every** finding, pick **exactly one** mode:

**A — Code snippets (default)**
When `code_snippet_unavailable` absent/false:
- Render **Before**/**After** fenced `go` blocks from `code_before`/`code_after`
- If either missing/empty/whitespace: **reconstruct** from Phase 1 (`tmp_dir/diffs/` + full file) at `{file}` around `{line}`
- **Never** publish empty `go` fences under mode A

**B — No snippet (explicit waiver)**
When `code_snippet_unavailable: true`:
- No Before/After fences
- Render: **Code snippet:** not applicable — {code_absence_note}
- Don't reconstruct over waiver

Every finding must be A or B. No compact one-line format dropping both.

Mode B sparingly (cross-cutting, policy-only, missing artifact). **If single line/hunk in Phase 1 diff at finding's `file`/`line` illustrates issue → mode A mandatory.** Sub-agents must not choose B to avoid quoting diff; orchestrator should reconstruct (A) when possible.

---

# Go Code Review | {title}
**{author}** · `{source_branch}` → `{target_branch}`

## MR Overview
**What changed:** {1-3 sentence summary}
**Why:** {1-3 sentence purpose}

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

Rules: 1+ critical → **REJECT**, 0 critical + 1+ major → **REQUEST CHANGES**, else **LGTM**

---

## Critical ({N})

Group by agent category. Per finding:

### {Agent Category Name}

#### [{id}] {title}
- **File:** `{file}:{line}`
- **Category:** {category}
- **Problem:** {problem — include production impact}
- **Mode A** (`code_snippet_unavailable` not true): **Before:**/`go` `{code_before}` — **After:**/`go` `{code_after}` (reconstruct if needed per contract)
- **Mode B** (`code_snippet_unavailable: true`): **Code snippet:** not applicable — {code_absence_note}

---

## Major ({N})

Same A/B branching as Critical, grouped by agent category.

---

## Minor ({N})

Same A/B branching as Critical, grouped by agent category.

---

## Open Questions

Merge `open_questions` from all agents. Dedup by meaning. Cap 10. Only include if exist.
- **[{agent}]** {question}

---

## What Was Done Well

Merge `positive` from all agents. Dedup. Bullet list:
- {positive observation}

---

## Key Production Risks

Only if critical/major findings. Summarize real consequences:
- {risk + estimated impact}

---

Notes:
- `requires_verification: true` → append "(requires verification)" to title
- `--only` → note which agents in Summary
- Agent with 0 findings → no empty section
- Category order: correctness, concurrency, conventions, style, tests, consistency, transactions, performance, security
- **Completeness:** mode A → non-empty Before/After fences (reconstruct if needed). Mode B → `code_absence_note` present. Never finding with neither fences nor waiver.
