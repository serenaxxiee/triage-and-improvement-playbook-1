# Failure Log Template

Use this template to record failure analysis from your triage sessions. Documenting failures builds institutional knowledge — the second time a knowledge grounding issue appears, you already know what to check first.

Choose the version that fits your team:
- [Lightweight version](#lightweight-failure-log) — for small teams iterating on a single agent
- [Detailed version](#detailed-failure-log) — for larger teams or when building institutional knowledge across agents

A CSV version is also available: [failure-log-template.csv](failure-log-template.csv)

---

## Lightweight Failure Log

Copy this table and fill it in during triage sessions. One row per failure.

| Test Case ID | Root Cause Type | What Went Wrong | What I Changed | Fixed? |
|---|---|---|---|---|
| ___ | Eval Setup / Agent Config / Platform Limitation | ___ | ___ | Yes / No / Partial |
| ___ | ___ | ___ | ___ | ___ |
| ___ | ___ | ___ | ___ | ___ |

### Example (filled in)

| Test Case ID | Root Cause Type | What Went Wrong | What I Changed | Fixed? |
|---|---|---|---|---|
| KG-003 | Eval Setup | Expected answer outdated (old return policy — 30 days; current policy is 15 business days) | Updated expected value to "15 business days" | Yes |
| KG-005 | Agent Config | Agent hallucinated warranty details not in any knowledge source | Added grounding instruction: "Only answer from knowledge sources" | Yes |
| TI-002 | Platform Limitation | Retrieval ranking ignores exact document title; FAQ always retrieved instead of product manual | Restructured doc headings as workaround; escalated to platform team | Partial |
| FA-019 | Platform Limitation | Ambiguous query can't reliably retrieve correct source | Documented as known limitation; monitoring in production | No — known gap |

---

## Detailed Failure Log

For teams that need to share findings, track status across sprints, or build institutional knowledge across multiple agents.

### Per-Failure Record

| Field | Value |
|---|---|
| **Test Case ID** | _(from the eval set, e.g., KG-003)_ |
| **Eval Set** | _(which eval set this belongs to)_ |
| **Quality Signal** | _(factual accuracy, knowledge grounding, tool invocation, etc.)_ |
| **Root Cause Type** | _(Eval Setup / Agent Config / Platform Limitation / Tool-Integration / Unclassified)_ |
| **Root Cause Detail** | _(specific sub-type, e.g., "outdated expected answer", "tool description ambiguity")_ |
| **What Went Wrong** | _(what the agent did vs. what it should have done)_ |
| **Diagnostic Path** | _(which triage questions led to this classification, e.g., "Step 1, Q1.2 — expected answer outdated")_ |
| **Remediation Action** | _(what was changed — specific enough to reproduce)_ |
| **Status** | _(Open / In Progress / Resolved / Won't Fix)_ |
| **Won't Fix Rationale** | _(if Won't Fix: why, and what monitoring is in place)_ |
| **Verification** | _(re-run result: pass/fail, date, iteration number)_ |
| **Date Triaged** | ___ |
| **Triaged By** | ___ |

### Example (filled in)

| Field | Value |
|---|---|
| **Test Case ID** | KG-005 |
| **Eval Set** | Knowledge Grounding |
| **Quality Signal** | Knowledge grounding / hallucination prevention |
| **Root Cause Type** | Agent Config |
| **Root Cause Detail** | Hallucination — agent generated content not in any knowledge source |
| **What Went Wrong** | Agent claimed "3-year extended warranty covering all parts and labor" when source says "2-year standard warranty" |
| **Diagnostic Path** | Step 1 passed (eval valid) → Step 2, Q2.4 (answered without source) + Q2.5 (contradicted source) |
| **Remediation Action** | Added to system prompt: "Only answer based on information found in your knowledge sources. If the information is not available, say so." |
| **Status** | Resolved |
| **Won't Fix Rationale** | N/A |
| **Verification** | Pass — iteration 2, Feb 15 |
| **Date Triaged** | Feb 14 |
| **Triaged By** | [name] |

---

## Iteration Summary Log

Track scores and changes across iterations to enable [trend analysis](../pattern-analysis.md#trend-analysis-across-iterations).

| Iteration | Date | Change Made | Eval Set Affected | Score Before | Score After | Delta | Notes |
|---|---|---|---|---|---|---|---|
| 1 | ___ | Baseline (no changes) | All | — | ___% | — | Initial run |
| 2 | ___ | ___ | ___ | ___% | ___% | ___ | ___ |
| 3 | ___ | ___ | ___ | ___% | ___% | ___ | ___ |

---

## Concentration Summary

After each triage session, tally your root cause types to check for [concentration patterns](../pattern-analysis.md#concentration-analysis).

| Root Cause Type | Count | % of Total | Systemic? |
|---|---|---|---|
| Eval Setup | ___ | ___% | _(80%+ = pause agent work, fix evals first)_ |
| Agent Config | ___ | ___% | _(80%+ in one area = architectural issue)_ |
| Platform Limitation | ___ | ___% | _(80%+ = reassess scope, escalate)_ |
| Tool/Integration | ___ | ___% | _(fix backend, not agent)_ |
| Unclassified | ___ | ___% | _(monitor; may become classifiable with more data)_ |
| **Total** | ___ | 100% | |

---

## Tips for Maintaining the Log

- **Update in real time** during triage — don't batch updates after the session
- **Record negative results** too ("tried X, didn't help") — these prevent re-trying failed approaches
- **Review before each iteration** — check for patterns before diving into individual failures
- **Share across the team** — the log's value multiplies when everyone can see prior findings
- **Archive, don't delete** — keep resolved entries for pattern analysis; move to an archive section if the active log gets long
