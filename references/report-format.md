# Report Format Template

Use this template to render the final review report from merged sub-agent JSON reports.

---

# Go Code Review | MR !{iid}: {title}
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
- **Before:**
```go
{code_before}
```
- **After:**
```go
{code_after}
```

---

## Major ({N})

Same structure as Critical, grouped by agent category.

---

## Minor ({N})

Compact format, one entry per finding:

- **[{id}]** `{file}:{line}` — {title}. {problem} → Fix: {code_after summary}

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
