# Grader Model Selection & Calibration Guide

> **"Which grading method should I use, and how do I know it's accurate?"**

Choosing the right grading approach is as important as designing good test cases. A miscalibrated grader produces misleading scores — you'll either ship a broken agent or waste cycles fixing an agent that's actually performing well.

**Before you start:** This guide helps you select and calibrate your grading approach. If you already have eval results and need to interpret them, start at [Layer 1: Score Interpretation](README.md#layer-1-score-interpretation--readiness-assessment).

---

## Grading Method Types

There are three fundamental approaches to grading eval results. Most production eval systems use a combination.

| Method | How It Works | Best For | Limitations |
|---|---|---|---|
| **Deterministic (rule-based)** | Exact match, keyword match, regex, or structured output validation | Factual accuracy with known answers, tool call correctness, format compliance | Brittle to valid variations; fails on open-ended responses |
| **LLM-as-judge** | A language model scores the agent's output against criteria or a rubric | Tone, helpfulness, reasoning quality, complex multi-part answers | Non-deterministic; requires calibration; adds cost |
| **Human review** | Domain experts manually score outputs | Ambiguous cases, establishing ground truth, calibrating LLM judges | Expensive, slow, doesn't scale |

### When to Use Each

```
Is there ONE correct answer (or a small set)?
  ├─ YES → Use deterministic grading (keyword match, exact match)
  │         Example: "What is the return policy?" → expected keywords
  │
  └─ NO → Is the quality subjective or multi-dimensional?
           ├─ YES → Use LLM-as-judge with a rubric
           │         Example: "Was the response empathetic and helpful?"
           │
           └─ NO → Is it a binary classification (did/didn't do X)?
                    ├─ YES → Use classification grading (Custom test method)
                    │         Example: "Did the agent refuse the out-of-scope request?"
                    │
                    └─ Use Compare Meaning or LLM-as-judge
                              Example: "Explain the cancellation process" (many valid phrasings)
```

---

## Copilot Studio Grading Methods Mapped

If you're working in Copilot Studio, here's how these general categories map to the platform's built-in test methods:

| Copilot Studio Method | Type | Use When |
|---|---|---|
| **Keyword Match** | Deterministic | Response must contain specific terms (product names, policy phrases, required disclaimers) |
| **Compare Meaning** | LLM-as-judge | Response should convey the same meaning as the expected answer, but exact wording doesn't matter |
| **Capability Use** | Deterministic | Testing whether the agent invoked the right tool, topic, or knowledge source (checks metadata on which topic/action was invoked, not the response text) |
| **Custom** (classification) | LLM-as-judge | Binary or multi-label classification against evaluation instructions (e.g., compliance, safety, brand voice) |

> **Tip:** You can combine methods on the same eval set. Use Keyword Match for factual checks alongside Custom for tone/compliance on the same test cases.

---

## Selecting an LLM Judge Model

When using LLM-as-judge grading (Compare Meaning, Custom, or external eval frameworks), the choice of judge model matters significantly.

### General Principles

1. **Use a stronger model than the agent being evaluated.** A judge should be at least as capable as the system it's grading. If your agent uses GPT-4o-mini, don't judge it with GPT-4o-mini.

2. **Start with the strongest available model, then optimize for cost.** Begin with the highest-quality judge to establish a reliable baseline, then test whether a smaller model produces equivalent verdicts.

3. **Match the judge to the task complexity.** Simple factual checks don't need the most powerful model. Nuanced quality assessments (tone, helpfulness, reasoning) do.

### Model Selection by Task

| Grading Task | Recommended Judge Tier | Rationale |
|---|---|---|
| Factual correctness (semantic match) | Mid-tier | Comparing two statements for semantic equivalence is well within mid-tier capabilities |
| Safety / compliance classification | Top-tier | Safety judgments require nuanced understanding of context and edge cases |
| Tone and empathy assessment | Top-tier | Subjective qualities require sophisticated language understanding |
| Multi-step reasoning quality | Top-tier | Evaluating reasoning chains requires strong reasoning ability in the judge |
| Format / structure compliance | Mid-tier or deterministic | Often checkable with rules; LLM only needed for borderline cases |
| Tool call parameter correctness | Deterministic preferred | Structured outputs are better validated with code than LLM judgment |

> **Note:** "Mid-tier" and "top-tier" refer to capability tiers in your platform's model catalog. Model names and capabilities change frequently — consult your platform's current offerings when selecting a judge model.

### Cost-Quality Tradeoff

Running LLM judges at scale adds up. Here's how to manage the tradeoff:

| Strategy | How | When |
|---|---|---|
| **Tiered judging** | Use a cheap model for easy cases, expensive model for uncertain ones | Large eval sets (100+ test cases) where most cases are straightforward |
| **Confidence thresholds** | If the judge's confidence is low, escalate to human review or a stronger model | When judge accuracy on borderline cases is critical |
| **Periodic calibration** | Run full eval with top-tier judge monthly; use mid-tier for daily runs | Continuous evaluation pipelines |
| **Cache verdicts** | If the agent output hasn't changed, don't re-grade | Iterative development with frequent re-runs |

---

## Calibrating Your Grader

A grader is only useful if it agrees with human judgment. Calibration is the process of verifying and improving that agreement.

### Step 1: Create a Calibration Set

Select 20-30 test cases that represent the range of your eval set:
- 5-8 clear passes (agent responded well)
- 5-8 clear fails (agent responded poorly)
- 10-15 borderline cases (reasonable people might disagree)

Have **two domain experts independently score** each case as pass/fail. Cases where experts disagree are your most valuable calibration data — they reveal where your criteria need tightening.

> **Lightweight alternative:** If two independent reviewers aren't available, one reviewer scoring 15-20 cases still provides useful signal. Prioritize borderline cases over clear pass/fail — those are where grader reliability matters most.

### Step 2: Run Your Grader on the Calibration Set

Run the automated grader on the same 20-30 cases and compare verdicts.

### Step 3: Measure Agreement

Calculate agreement between the grader and human consensus:

| Agreement Rate | Interpretation | Action |
|---|---|---|
| **90%+** | Excellent. Grader is production-ready for this eval set. | Proceed. Spot-check periodically. |
| **80-90%** | Good. Review the disagreements to identify systematic patterns. | Refine rubric or grader prompt, then re-calibrate. |
| **70-80%** | Moderate. Grader has blind spots or your criteria are ambiguous. | Rewrite grader criteria with more specific examples. Consider a stronger judge model. |
| **Below 70%** | Poor. Grader is not reliable for this eval set. | Switch methods (e.g., from LLM judge to deterministic), or fundamentally redesign the rubric. |

### Step 4: Diagnose Disagreements

For each case where the grader disagrees with humans, classify the disagreement:

| Disagreement Type | Pattern | Fix |
|---|---|---|
| **False passes** | Grader says pass, humans say fail | Add negative examples to the rubric: "The following should FAIL: ..." |
| **False fails** | Grader says fail, humans say pass | Broaden acceptance criteria; add positive examples of valid variations |
| **Inconsistent** | Same input, different verdicts across runs | Reduce temperature; add chain-of-thought; use a more deterministic method |
| **Systematic bias** | Grader consistently favors longer/shorter responses, or particular phrasing styles | Add explicit instruction: "Response length should not affect the verdict" |

### Step 5: Re-calibrate After Changes

After adjusting the grader, re-run on the full calibration set. **Don't just check the cases you fixed** — ensure you haven't introduced new disagreements.

---

## Common Grader Failure Patterns

These patterns frequently appear during triage. If your eval scores seem unreliable, check for these first.

### Pattern: Scores Swing Between Runs

**Symptom:** Same agent, same test cases, but pass rates vary by 10%+ across runs.

**Likely causes:**
- LLM judge non-determinism (temperature > 0)
- Ambiguous grading criteria that the judge interprets differently each time
- Agent itself is non-deterministic (varies per run)

**Fix:** Set judge temperature to 0. Add chain-of-thought prompting to the judge. Run 3x and average. If still unstable, switch to deterministic grading for those test cases.

### Pattern: Everything Passes

**Symptom:** 95%+ pass rate, but manual review reveals clear quality issues.

**Likely causes:**
- Grading criteria too lenient ("Is the response relevant?" → almost always yes)
- Judge model is biased toward saying "pass" (positivity bias)
- Expected answers too vague

**Fix:** Add specific failure criteria. Use negative examples in the rubric. Test with intentionally bad responses to verify the grader catches them.

### Pattern: Grader Contradicts Itself

**Symptom:** Very similar responses get different verdicts.

**Likely causes:**
- No chain-of-thought reasoning in the judge prompt
- Judge criteria have conflicting rules
- Judge is sensitive to superficial features (formatting, length)

**Fix:** Add "First, reason step by step about why this should pass or fail. Then give your verdict." Make criteria non-overlapping. Add "Judge only on content, not presentation style."

### Pattern: Grader Is Harsher Than Humans

**Symptom:** Many false fails; humans review and say the response was fine.

**Likely causes:**
- Rubric is aspirational (describes ideal response, not minimum acceptable)
- Expected answer is too specific; valid variations are rejected
- Judge doesn't understand domain-specific conventions

**Fix:** Frame rubric as "minimum acceptable quality" not "ideal response." Add explicit acceptable variations. Include domain context in the judge prompt.

---

## Grader Selection Checklist

Use this before running a new eval set:

- [ ] **Method matches the quality signal.** Factual accuracy → deterministic or semantic match. Subjective quality → LLM judge. Binary classification → Custom method.
- [ ] **Judge model is appropriate for the task.** Simple tasks → mid-tier model. Nuanced judgment → top-tier model.
- [ ] **Grader has been calibrated.** At least 20 cases scored by humans and compared to grader verdicts. Agreement is 80%+.
- [ ] **Rubric includes both positive and negative examples.** The grader knows what "good" looks like AND what "bad" looks like.
- [ ] **Non-determinism is managed.** Temperature set to 0 for judge. Multiple runs averaged if score stability matters.
- [ ] **Cost is sustainable.** Grading cost per eval run is acceptable for your run frequency. Tiered judging considered for large eval sets.

---

## Related Resources

- [Layer 2: Failure Triage — Grader Validation](triage-decision-tree.md#grader-validation) — quick diagnostic for grader reliability issues (this guide provides the deeper treatment)
- [Layer 3: Remediation Mapping](remediation-mapping.md) — fix grader issues (see "Eval Setup Remediation")
- [Layer 4: Pattern Analysis](pattern-analysis.md) — identify systematic grader problems across eval sets
