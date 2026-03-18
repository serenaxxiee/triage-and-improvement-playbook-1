# Prompt Versioning & Rollback Guide

How to manage prompt changes safely so that eval improvements stick and regressions get caught before they reach users.

**Start here** if you're thinking:
- *"I changed the system prompt and now a different eval set is failing."*
- *"We shipped a prompt update and quality dropped. How do I revert safely?"*
- *"I've made 12 prompt tweaks this week and don't know which ones helped."*
- *"How do I test a prompt change before committing to it?"*

> **Time:** 15-20 minutes to set up versioning practices. Ongoing benefits compound over every prompt change cycle.
>
> **What you need:** Your current agent system prompt, eval results from at least one run, and access to your agent configuration.

---

## Why Prompt Versioning Matters for Eval

Prompt changes are the most common remediation action in the [remediation mapping](remediation-mapping.md) — and also the most common source of regressions. Without versioning:

- You can't tell which change caused a score drop
- You can't revert to a known-good state
- You can't measure whether a change actually helped
- Multiple people editing prompts create conflicts and confusion

**The core principle:** Treat every prompt change as a deployable artifact with a version, a test result, and a rollback path.

---

## Quick-Reference: Prompt Change Checklist

Use this for every prompt modification:

```
PROMPT CHANGE CHECKLIST

BEFORE changing:
  [ ] Record current prompt version (copy full text + timestamp)
  [ ] Run baseline eval and record scores
  [ ] Identify which specific failures this change targets

WHILE changing:
  [ ] Make ONE conceptual change per version
  [ ] Write a change note: what changed and why
  [ ] Keep the diff minimal — don't reorganize while fixing

AFTER changing:
  [ ] Run the SAME eval set against the new prompt
  [ ] Compare: target failures fixed?
  [ ] Compare: any NEW failures introduced?
  [ ] If regression detected: revert, analyze, try a narrower change
  [ ] If improvement confirmed: record as new baseline
```

---

## Versioning Strategy

### Semantic Version Scheme for Prompts

Apply version numbers to prompt changes based on scope of impact:

| Version Bump | When to Use | Example |
|---|---|---|
| **Major** (v1 -> v2) | Fundamental behavior change: new persona, changed scope, restructured instructions | Rewriting the agent's role from "support assistant" to "technical advisor" |
| **Minor** (v1.1 -> v1.2) | Targeted behavior change: new capability, modified handling for a scenario category | Adding instructions for handling refund requests |
| **Patch** (v1.1.1 -> v1.1.2) | Wording refinement: clarification, typo fix, formatting that doesn't change behavior | Changing "respond briefly" to "respond in 2-3 sentences" |

### Version Record Template

For each prompt version, record:

```markdown
## Prompt Version: [agent-name] v1.3.0

**Date:** 2025-03-15
**Author:** [name]
**Parent version:** v1.2.1

### Change summary
Added explicit handling for multi-turn clarification questions.
Users were getting cut off when asking follow-ups.

### Targeted failures
- Test case KC-12: Follow-up question loses context (FAIL -> expected PASS)
- Test case KC-15: Clarification request gets generic response (FAIL -> expected PASS)

### Eval results comparison
| Eval Set | v1.2.1 Score | v1.3.0 Score | Delta |
|---|---|---|---|
| Knowledge Grounding | 78% (14/18) | 83% (15/18) | +5% |
| Conversation Quality | 85% (17/20) | 85% (17/20) | 0% |
| Safety & Compliance | 100% (8/8) | 100% (8/8) | 0% |

### Regression check
No regressions detected. KC-12 and KC-15 now pass.
KC-04 still fails (unrelated — tool invocation issue).

### Decision
ACCEPTED — promoted to current version.
```

---

## Regression Detection

### What Counts as a Regression

A regression is any eval score decrease that was not an intended consequence of the change. Watch for:

| Signal | Severity | Action |
|---|---|---|
| Safety/compliance score drops at all | **Critical** | Immediate revert. Do not ship. |
| Core business eval drops > 5% | **High** | Revert and re-approach with narrower change. |
| Capability eval drops > 10% | **Medium** | Investigate whether the trade-off is acceptable. |
| Single test case flips from pass to fail | **Low** | Check if non-deterministic (re-run 3x). If consistent, investigate. |

### The Regression Test Workflow

```
1. BASELINE
   Run full eval suite → record all scores → this is your "before"

2. CHANGE
   Make ONE prompt change → document what and why

3. VERIFY
   Run full eval suite → record all scores → this is your "after"

4. COMPARE
   For each eval set:
     Score improved?     → Good, expected if targeted
     Score unchanged?    → Good, no regression
     Score decreased?    → REGRESSION — go to step 5

5. DIAGNOSE REGRESSION
   Look at the newly-failing test cases:
     Related to your change?  → Your change has a side effect. Narrow it.
     Unrelated to your change? → Likely non-determinism. Re-run 3x to confirm.
     Flaky (passes sometimes)? → Flag as non-deterministic, not a true regression.

6. DECIDE
   No regressions → Accept new version
   Acceptable trade-off → Accept with documented justification
   Unacceptable regression → Revert to previous version
```

### Non-Determinism vs. Real Regressions

LLM outputs are non-deterministic. A test case that passed before and fails now might just be variance. To distinguish:

- **Run the eval 3 times** on both the old and new prompt versions
- **Compare pass rates**, not single outcomes: "3/3 pass" vs "1/3 pass" is a real regression; "3/3 pass" vs "2/3 pass" might be variance
- **Flag test cases that flip** between runs as "flaky" — these need either more robust grading criteria or acceptance of variance
- **Set a significance threshold**: if a test case passes 2/3+ times on the new version, it's likely not a regression

---

## Rollback Procedures

### When to Roll Back

| Situation | Rollback? | Reasoning |
|---|---|---|
| Safety score dropped | **Yes, immediately** | Safety regressions are never acceptable trade-offs |
| Core eval dropped, cause unknown | **Yes** | Don't debug in production; revert first, investigate second |
| Core eval dropped, cause understood, fix is quick | **Maybe** | If you can fix forward in <1 hour, consider it |
| Capability score dropped, trade-off documented | **No** | If the trade-off was intentional and accepted, this is expected |
| Scores unchanged, but latency/cost increased | **Investigate** | Not a rollback trigger by default, but may warrant revert if cost is excessive |

### Rollback Process

```
ROLLBACK STEPS

1. REVERT the system prompt to the last known-good version
   (This is why you recorded the full text at each version)

2. RE-RUN the full eval suite to confirm scores return to baseline
   - If scores DON'T return to baseline, the regression may not be prompt-related
   - Check: did the model version change? Did knowledge sources update?

3. RECORD the rollback in your version history
   - Version that was reverted and why
   - Eval scores before and after rollback
   - Root cause hypothesis

4. PLAN a narrower re-attempt
   - What part of the change caused the regression?
   - Can you split it into smaller changes?
   - Do you need different test cases to validate the approach?
```

### Rollback Doesn't Fix Scores

If reverting the prompt doesn't restore the original scores, the regression has a different cause:

| Symptom | Likely Cause | Investigation |
|---|---|---|
| Scores don't recover after prompt revert | Model version changed | Check if the underlying LLM was updated |
| Scores don't recover, model is the same | Knowledge source changed | Check if documents, URLs, or data sources were updated |
| Scores partially recover | Multiple changes were made | Identify which non-prompt changes also happened |
| Scores are worse than original baseline | Non-deterministic drift | Run baseline prompt 3x to establish true current performance |

---

## Prompt Change Patterns

### Patterns That Commonly Cause Regressions

| Change Pattern | Risk | Why It Regresses |
|---|---|---|
| Adding length to system prompt | Medium | Instruction dilution — important rules get less attention |
| Contradicting existing instructions | High | "Be concise" + "Provide thorough explanations" creates unpredictable behavior |
| Reordering sections | Medium | Position bias — instructions at the beginning and end get more weight |
| Adding exception handling | Medium | "If X, do Y; otherwise do Z" can fire on unintended inputs |
| Removing instructions | High | Behaviors you thought were implicit may have depended on the removed text |
| Copy-pasting from another agent | High | Instructions tuned for one context rarely transfer cleanly |

### Safe Change Practices

1. **One change, one purpose**: Each prompt version change should target exactly one behavior. If you need to fix three things, make three versions and test each.

2. **Additive before subtractive**: When possible, add clarifying instructions rather than removing existing ones. Removal is harder to predict.

3. **Test the exact failure first**: Before running the full eval suite, test the specific cases you're trying to fix. If they don't improve, no need to check for regressions.

4. **Keep a prompt changelog**: A running log of what changed and why. This is invaluable when debugging regressions weeks later.

---

## Prompt Changelog Template

Maintain this alongside your agent configuration:

```markdown
# [Agent Name] Prompt Changelog

## v1.3.0 — 2025-03-15
**Change:** Added multi-turn clarification handling
**Why:** KC-12 and KC-15 failing — follow-up questions losing context
**Result:** +5% knowledge grounding, no regressions
**Status:** Active

## v1.2.1 — 2025-03-10
**Change:** Clarified response length expectation ("2-3 sentences" instead of "brief")
**Why:** CQ-08 failing — responses too short for complex questions
**Result:** +5% conversation quality, no regressions
**Status:** Superseded by v1.3.0

## v1.2.0 — 2025-03-05
**Change:** Added refund handling instructions
**Why:** New business requirement — refund flow launched
**Result:** New eval set (Refund Handling) at 90%, no regressions on existing sets
**Status:** Superseded by v1.2.1

## v1.1.0 — 2025-02-28 [ROLLED BACK]
**Change:** Restructured system prompt to group instructions by topic
**Why:** Prompt was getting hard to maintain
**Result:** -8% knowledge grounding regression, cause: reordering moved grounding rules to middle
**Status:** Rolled back to v1.0.2 on 2025-03-01
```

---

## Integration with the Triage Playbook

Prompt versioning connects to the triage flow at several points:

| Triage Stage | How Versioning Helps |
|---|---|
| **Layer 1: Score Interpretation** | Version history shows whether a score drop is new or pre-existing |
| **Layer 2: Failure Triage** | Version diffs help identify whether a failure was caused by a recent change |
| **Layer 3: Remediation** | Version records show which remediation approaches have already been tried |
| **Layer 4: Pattern Analysis** | Changelog reveals patterns: "every time we modify section X, eval Y regresses" |

### Triage Question: "Did a prompt change cause this?"

When investigating a failure, check the version history:

1. **When did this test case last pass?** Find the version where it was passing.
2. **What changed between then and now?** Diff the prompt versions.
3. **Is the change related to the failure?** If the change was about refunds and the failure is about grounding, the change is likely not the cause.
4. **Can you reproduce?** Run the failing test case against the last-known-good version. If it passes there, the prompt change is the cause.

---

## Further Reading

- [Remediation Mapping](remediation-mapping.md) — specific fixes for each root cause type
- [Pattern Analysis](pattern-analysis.md) — identifying systemic issues across failures
- [Eval Cost Management](eval-cost-management.md) — budgeting for regression test runs
- [Worked Examples](worked-examples.md) — see prompt versioning in context of full triage flows
