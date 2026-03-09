# Triage & Improvement Playbook

A diagnostic and remediation framework for turning eval results into action. Use this playbook to interpret scores, diagnose why test cases fail, identify who needs to act, and map to specific fixes.

**Start here** if you're thinking:
- *"My Knowledge Grounding eval set is at 71%. Is that bad? What do I fix?"*
- *"3 out of 7 tool invocation tests are failing. I don't know why."*
- *"We're at 88% overall. Is that good enough to ship?"*
- *"I have 15 failing test cases across 4 eval sets. Where do I even start?"*

> **Time:** 10-15 minutes for score interpretation (this page). Budget additional time per failure for triage and remediation in the later layers.
>
> **What you need:** Pass/fail results for each test case across one or more eval sets.

---

## Before You Start

You should have eval results in hand — a pass/fail outcome for each test case. This playbook will help you interpret scores, diagnose failures, identify who acts, and map to specific fixes. If you don't yet have results, run your eval sets first, then return here.

---

## Quick-Reference Cheat Sheet

Use this during active triage sessions. It's self-contained — you don't need to read the full playbook to use it.

```
TRIAGE CHEAT SHEET

1. SCORES OK?
   Safety/Compliance < 95%  → BLOCK     → Go to Layer 2 to triage safety failures
   Core business < 80%      → ITERATE   → Go to Layer 2 to triage the lowest-scoring eval set
   Capabilities < threshold → CONDITIONAL SHIP → Document gaps, then go to Layer 2 for each
   All above threshold      → SHIP ✓

2. FOR EACH FAILURE — ask in order:
   □ Is the agent's response actually acceptable?     → YES = Fix the eval
   □ Is the expected answer still correct?            → NO  = Fix the eval
   □ Can I identify a specific config to change?      → YES = Fix the agent
   □ Does the fix persist after config change?         → NO  = Platform limit

3. AFTER TRIAGE:
   □ 80%+ same root cause? = Systemic issue, fix the category
   □ Scores flat after fix? = Wrong root cause, re-triage
   □ One score up, another down? = Instruction conflict
```

---

## Playbook Structure

| Layer | Question It Answers | Go To |
|---|---|---|
| **Layer 1: Score Interpretation** | "What do my results mean? Am I ready to ship?" | [Below](#layer-1-score-interpretation--readiness-assessment) |
| **Layer 2: Failure Triage** | "Why did this fail and who needs to act?" | [triage-decision-tree.md](triage-decision-tree.md) |
| **Layer 3: Remediation Mapping** | "What specifically should I change?" | [remediation-mapping.md](remediation-mapping.md) |
| **Layer 4: Pattern Analysis** | "What systemic issues do my failures reveal?" | [pattern-analysis.md](pattern-analysis.md) |
| **Worked Examples** | "Show me the full flow end to end" | [worked-examples.md](worked-examples.md) |
| **Failure Log Template** | "How do I track what I've found?" | [templates/failure-log-template.md](templates/failure-log-template.md) |

> **Looking for a definition of "done"?** See [When Are You Done Iterating?](#when-are-you-done-iterating)

### The Three Root Cause Types

Every eval failure has one of three root cause types, defined by who needs to act:

| Root Cause Type | Who Acts | What It Means |
|---|---|---|
| **Eval Setup Issue** | Eval author | The test case, expected answer, or grader is wrong. The agent may actually be performing correctly. |
| **Agent Configuration Issue** | Agent builder | The agent genuinely produced a bad response, and you can fix it through configuration changes. |
| **Platform Limitation** | Platform team | The failure is caused by underlying platform behavior you cannot fix through configuration. |

### Design Principles

| Principle | What It Means in Practice |
|---|---|
| **Start from eval results, not theory** | Enter the playbook with actual pass rates and failing test cases — not abstract concepts. |
| **Eliminate wrong work first** | Always verify the eval setup before investigating the agent. This prevents the most common time waste. |
| **Root cause → owner → action** | Every diagnostic path ends with a specific person doing a specific thing. No dead ends. |
| **Verify your classification** | After remediating, re-run the eval. If the failure persists, the classification was wrong — re-triage. |
| **Reference, don't duplicate** | Quality signals and scenarios are defined in the [scenario library](../). The playbook references them, not restates them. |
| **Compound causes are real** | A single failure can have multiple contributing root causes. The triage process handles mixed cases. |
| **Non-determinism is expected** | LLM-based agents and graders produce variable results. The playbook accounts for this with re-run verification guidance. |

---

## Layer 1: Score Interpretation & Readiness Assessment

### Reading Your Results

Interpret pass rates at two levels:

- **Per eval set:** What does X% mean for this specific eval set?
- **Per quality signal:** What does X% mean for this quality signal across all eval sets that test it?

A low pass rate doesn't automatically mean the agent is broken. It means something needs investigation — it could be the agent, the eval, or the platform.

### Setting Your Own Thresholds

Rather than applying fixed numbers, derive thresholds from your agent's risk profile:

| Factor | Questions to Ask | Impact on Threshold |
|---|---|---|
| **Consequence of failure** | What happens if the agent gets this wrong? Inconvenience? Financial loss? Safety risk? | Higher consequence → higher threshold |
| **Frequency of query type** | How often will users trigger this quality signal? | Higher frequency → higher threshold (more exposure) |
| **Fallback availability** | If the agent fails, is there a human backup? How fast? | No fallback → higher threshold |
| **Audience** | Internal employees? External customers? Regulated industry? | External/regulated → higher threshold |

**Translating factors into starting thresholds — example risk profiles:**

| Risk Profile | Description | Safety/Compliance | Core Business | Capabilities |
|---|---|---|---|---|
| Low-risk internal tool | Internal-only, human review on all outputs, low-stakes domain | 90%+ | 75%+ | 65%+ |
| Medium-risk customer-facing agent | External users, some automation, recoverable errors | 95%+ | 85%+ | 75%+ |
| High-risk regulated/financial agent | External users, consequential decisions, regulatory exposure | 98%+ | 92%+ | 85%+ |
| Safety-critical agent | Health, legal, or financial advice with limited human oversight | 99%+ | 95%+ | 90%+ |

These profiles are illustrative starting points only — not universal standards. Use them as anchors, then calibrate up or down based on your specific answers to the factor questions above.

### Example Threshold Calibration

The following numbers are **one example calibration** for a medium-risk, customer-facing support agent. They are not universal standards. Your thresholds should reflect your specific risk profile using the framework above.

| Quality Signal Category | Example Starting Threshold | Example Blocking Threshold | Rationale |
|---|---|---|---|
| Safety & PII protection | 95-100% | < 95% blocks ship | Any safety failure is high-risk |
| Compliance & verbatim content | 95-100% | < 95% blocks ship | Regulatory/legal exposure |
| Factual accuracy (core business) | 85-95% | < 80% blocks ship | Core value proposition |
| Knowledge grounding | 85-95% | < 80% blocks ship | Foundation for accuracy |
| Tool invocation correctness | 90-95% | < 85% blocks ship | Task execution reliability |
| Trigger routing accuracy | 85-95% | < 80% blocks ship | Conversation flow correctness |
| Escalation & graceful failure | 90-95% | < 85% blocks ship | User experience safety net |
| Tone & response quality | 80-90% | < 75% blocks ship | Subjective; inherent grader variability |

> **Calibrate, don't copy.** A low-risk internal FAQ bot might set factual accuracy at 80%. A healthcare triage agent might require 98%. The right threshold is the one where a failure at that rate is an acceptable risk for your use case.

### Readiness Assessment

Use this decision framework when asking "should I ship or keep iterating?"

```
                    ALL safety/compliance eval sets
                    above blocking threshold?
                           │
                    NO ──► BLOCK: Fix safety issues before
                           │      anything else. Do not ship.
                    YES
                           │
                    ALL core business eval sets
                    above their thresholds?
                           │
                    NO ──► ITERATE: Focus remediation on
                           │        the lowest-scoring core eval set.
                    YES
                           │
                    Capability eval sets
                    above their thresholds?
                           │
                    NO ──► SHIP WITH KNOWN GAPS:
                           │ Document which capability gaps
                           │ exist. Set monitoring plan.
                           │ Continue iteration post-ship.
                    YES
                           │
                    SHIP ✓
```

> **"Ship with known gaps"** is a legitimate outcome. Not every capability gap needs to be resolved before launch — some are acceptable with monitoring. The key is to **document what you're accepting** and have a plan to address it. Use the [failure log template](templates/failure-log-template.md) to track known gaps.

### When Are You Done Iterating?

**You're done when:**
- All eval sets are above their thresholds (including thresholds you've consciously adjusted)
- Known gaps are documented with owners and timelines
- Re-runs produce consistent scores (< 5% variance between runs)
- The failure log has no open items classified as "Agent Configuration Issue" for blocking signals

**You are NOT done when:**
- You've hit your threshold but haven't investigated why some test cases still fail
- Scores are above threshold only because you removed hard test cases
- Platform limitations are being silently absorbed into "acceptable" pass rates without documentation

### Handling Non-Determinism in Scores

LLM-based agents and graders produce variable outputs. Practical guidance:

- **Establish baselines:** Run the full eval set at least 3 times before treating any score as a baseline. Use the average as your working score. With fewer than 3 runs, you cannot distinguish real signal from noise.
- **Score variance between runs:** +/-5% variance between runs is normal for LLM-based graders. If variance exceeds +/-10%, investigate grader reliability before diagnosing agent issues — see [Grader Validation](triage-decision-tree.md#grader-validation) in Layer 2.
- **Interpreting score changes after remediation:** For eval sets with fewer than 30 test cases, a single test case flipping from fail to pass changes the score by 3%+. Do not over-interpret small movements. For eval sets with 50+ test cases, a 5%+ change is likely meaningful. When in doubt, re-run 3 times and compare the average to your baseline average.
- **Flaky test cases** (pass sometimes, fail others): A test case that passes 2 out of 3 runs is borderline. Investigate: is the expected value too rigid (eval setup), or is the agent genuinely inconsistent (agent config)? If the agent produces two different but both-acceptable responses, the eval is too rigid.

---

## What's Next

Once you've interpreted your scores and identified where to focus:

1. **If you have failing test cases to investigate:** Go to [Failure Triage (Layer 2)](triage-decision-tree.md)
2. **If you've already diagnosed failures and need fixes:** Go to [Remediation Mapping (Layer 3)](remediation-mapping.md)
3. **If you want to see the full flow end to end:** Go to [Worked Examples](worked-examples.md)
4. **If you're looking for systemic patterns across failures:** Go to [Pattern Analysis (Layer 4)](pattern-analysis.md)

If you have more than 10 failing test cases, consider starting with [Pattern Analysis (Layer 4)](pattern-analysis.md) to identify systemic issues before triaging individually in Layer 2.

---

## Related Resources

### Evaluation Scenario Library

The [scenario library](../) defines the quality signals and scenarios that produce the eval results this playbook helps you interpret.

| The Scenario Library Gives You | This Playbook Uses It For |
|---|---|
| Quality signal definitions (factual accuracy, knowledge grounding, etc.) | The vocabulary for classifying and triaging failures |
| Evaluation patterns (per scenario) | Context for understanding what was being tested when a failure occurs |
| Practical examples (test cases) | The test cases whose failures you're triaging |
| Recommended test methods per scenario | Input for diagnosing "wrong eval method" issues in [Layer 2](triage-decision-tree.md) |
| Pass thresholds (per eval set) | Baseline for score interpretation (Layer 1, above) |

### Eval Set Architecture

The quality of your triage results depends on the quality of your eval set structure:
- **Well-structured eval sets** (organized by quality signal or scenario category) → each set's pass rate is interpretable and this playbook's triage works well
- **Poorly structured eval sets** (mixed quality signals in one set, no clear category boundaries) → scores are noisy and triage produces ambiguous results

If your eval set scores are hard to interpret because test cases testing different things are mixed together, consider restructuring your eval sets before triaging individual failures.

