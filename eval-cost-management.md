# Evaluation Cost Management Guide

> Practical strategies for managing the cost of evaluating AI agents — so you can maintain quality without runaway spending.

## Why Eval Costs Matter

Evaluating AI agents is significantly more expensive than testing traditional software. A single LLM-as-judge call costs orders of magnitude more than a unit test assertion. Teams that don't plan for eval costs often face one of two failure modes:

1. **Cost shock** — an unconstrained eval run generates a five-figure bill overnight
2. **Quality erosion** — teams skip evaluations to save money, then ship regressions

This guide helps you find the middle ground: rigorous evaluation within a planned budget.

## Cost Drivers in Agent Evaluation

Before optimizing, understand where the money goes:

| Cost Driver | Impact | Example |
|---|---|---|
| **Judge model tier** | High | GPT-4 costs ~20x more than GPT-3.5-turbo per token |
| **Test case volume** | High | 100 vs 1,000 test cases = 10x cost difference |
| **Eval frequency** | Medium | Running on every commit vs weekly batches |
| **Multi-turn depth** | Medium | 3-turn conversations cost 3x single-turn |
| **Grader prompt length** | Low-Medium | Detailed rubrics consume more input tokens |
| **Retry/re-run rate** | Variable | Flaky evals that need 3 runs waste 2/3 of spend |

### Quick Cost Estimation

For a rough budget estimate:

```
Monthly eval cost ≈ (test cases) × (avg tokens per eval) × (price per token) × (runs per month)
```

**Example**: 200 test cases × 2,000 tokens × $0.01/1K tokens (GPT-4o) × 20 runs/month = **$80/month**

The same setup with GPT-4o-mini at $0.00015/1K tokens: **$1.20/month**

## The Eval Cost Pyramid

Structure your evaluation like a testing pyramid — cheap checks at the base, expensive checks at the top:

```
        /  Human  \          ← Highest quality, highest cost
       /  Review   \            5-10% of cases
      /─────────────\
     / LLM-as-Judge  \      ← Flexible, moderate cost
    /  (Full Model)    \        20-30% of cases
   /───────────────────\
  / LLM-as-Judge (Small) \  ← Good enough for most checks
 /   or Classification     \    40-50% of cases
/───────────────────────────\
/  Deterministic Checks      \ ← Near-zero cost
/ (regex, keyword, format)     \   100% of cases
─────────────────────────────────
```

**Apply every layer that fits your scenario.** Don't use an LLM judge to check whether the response is valid JSON — a regex check costs nothing and is more reliable.

## Five Strategies for Cost-Effective Evaluation

### Strategy 1: Right-Size Your Judge Model

Not every evaluation needs your most expensive model. Match the judge to the task:

| Eval Task | Recommended Tier | Examples |
|---|---|---|
| Format validation, keyword presence | **Deterministic** (no LLM) | JSON structure, required disclaimers |
| Binary classification (pass/fail) | **Small model** (GPT-4o-mini, Haiku) | Safety checks, topic relevance |
| Nuanced quality scoring | **Standard model** (GPT-4o, Sonnet) | Helpfulness, completeness |
| Complex reasoning judgment | **Premium model** (GPT-4-turbo, Opus) | Multi-step accuracy, subtle errors |

**Cost impact**: Switching from GPT-4 to GPT-4o-mini for binary classification tasks can reduce those eval costs by **95%+** with minimal quality loss (Pearson correlation >0.85 with human judgments for straightforward tasks).

**In Copilot Studio**: Use the built-in **Generative Answer** evaluator for standard quality checks before adding expensive Custom evaluators. The Custom test method with classification labels is cheaper than open-ended LLM scoring because it constrains the output space.

### Strategy 2: Tiered Test Case Selection

Don't run every test case through every evaluator:

1. **Always run** (100%): Deterministic checks on all test cases — fast and free
2. **Regular run** (full set): Small-model classification on all cases — cheap
3. **Targeted run** (subset): Full LLM judge on cases that *failed* cheaper checks or are flagged as high-risk
4. **Periodic audit** (sample): Human review on a rotating 5-10% sample

**Implementation pattern**:
```
For each test case:
  1. Run deterministic checks → if FAIL, log and skip expensive eval
  2. Run small-model classifier → if PASS with high confidence, done
  3. If UNCERTAIN or marginal → escalate to full LLM judge
  4. Weekly: sample 10% of "PASS" cases for human spot-check
```

This cascading approach typically reduces LLM judge invocations by **40-60%** compared to running every case through the full pipeline.

### Strategy 3: Smart Scheduling and Caching

**Batch evaluations strategically:**
- **On every commit**: Only run deterministic checks + critical safety evals
- **Nightly**: Run the full small-model eval suite
- **Weekly**: Run full LLM-as-judge evaluation + human review sample
- **Pre-release**: Run the complete evaluation matrix

**Cache aggressively:**
- Cache eval results for test cases where *both* the agent response and the grader prompt haven't changed
- For regression testing, only re-evaluate cases where the agent's response actually changed
- Store evaluation results with content hashes to detect when re-evaluation is truly needed

### Strategy 4: Budget Controls with Graduated Response

Set explicit spend limits with escalating actions — don't let a runaway eval drain your budget:

| Budget Threshold | Action |
|---|---|
| **50%** consumed | Alert the team, review remaining schedule |
| **80%** consumed | Throttle: switch to small-model judges only |
| **90%** consumed | Downgrade: deterministic checks only, queue LLM evals for next period |
| **100%** consumed | Pause eval runs, notify team lead |

**Set budgets at multiple levels:**
- Per eval run (prevent any single run from exceeding a cap)
- Per day/week (smooth out spending over time)
- Per month (align with financial planning)

### Strategy 5: Invest in Deterministic Checks First

Before spending on LLM judges, maximize what you can check for free:

| Check Type | What It Catches | Cost |
|---|---|---|
| Response length bounds | Runaway generation, empty responses | Free |
| Required keyword/phrase | Missing disclaimers, safety language | Free |
| JSON/XML schema validation | Structured output errors | Free |
| Regex pattern matching | Format violations, PII leakage | Free |
| Response latency threshold | Performance regressions | Free |
| Tool call validation | Incorrect function signatures | Free |

A well-designed deterministic check suite catches **30-50% of failures** without any LLM cost. Build these first, then layer LLM judges on top for what deterministic checks can't cover.

## Budget Planning Template

Use this template to plan your evaluation budget:

### Step 1: Inventory Your Eval Needs

| Scenario Category | Test Cases | Eval Method | Estimated Monthly Runs |
|---|---|---|---|
| Safety & compliance | ___ | ___ | ___ |
| Core task accuracy | ___ | ___ | ___ |
| Edge cases | ___ | ___ | ___ |
| Regression suite | ___ | ___ | ___ |

### Step 2: Estimate Costs per Method

| Method | Cost per Eval | Cases | Monthly Runs | Monthly Cost |
|---|---|---|---|---|
| Deterministic | ~$0 | ___ | ___ | $0 |
| Small LLM judge | ~$0.001 | ___ | ___ | $___ |
| Full LLM judge | ~$0.01-0.10 | ___ | ___ | $___ |
| Human review | ~$1-5 | ___ | ___ | $___ |
| **Total** | | | | **$___** |

### Step 3: Set Your Budget Allocation

A common split for teams getting started:

- **60%** — Regular automated evaluation (small + full LLM judges)
- **20%** — Human review and calibration
- **15%** — Ad-hoc investigation of failures
- **5%** — Buffer for unexpected spikes

## Common Cost Mistakes

| Mistake | Why It Happens | Fix |
|---|---|---|
| Using GPT-4 for every check | "We want the best quality" | Right-size: most checks don't need premium models |
| Running full suite on every commit | Fear of missing regressions | Tiered schedule: deterministic on commit, full suite nightly |
| No caching of eval results | Eval pipeline rebuilt from scratch each run | Hash-based caching of unchanged case + grader combinations |
| Evaluating unchanged outputs | Full re-run when only 5% of cases changed | Diff-based evaluation: only re-eval changed responses |
| Ignoring deterministic options | Over-reliance on LLM judges | Build deterministic checks first, LLM judges for the rest |
| No spend limits | "We'll monitor it manually" | Automated budget controls with graduated response |
| Overly verbose grader prompts | Detailed rubrics with extensive examples | Optimize grader prompts for token efficiency (keep examples concise) |

## Connecting to the Triage Playbook

When the [triage decision tree](triage-decision-tree.md) identifies a problem, you need to re-evaluate after applying a fix. Use cost-aware re-evaluation:

1. **Targeted re-eval**: Only re-run the eval scenarios relevant to the specific failure pattern
2. **Cascading verification**: Start with deterministic checks to confirm the fix didn't break basics, then escalate to LLM judges for the specific quality dimension that failed
3. **Regression sampling**: After a fix, run a random 20% sample of the full suite (not 100%) to check for unintended side effects — expand to 100% only if the sample shows issues

This keeps the fix-verify cycle fast and affordable, especially when iterating through multiple remediation attempts from the [remediation mapping](remediation-mapping.md).

## Key Takeaways

1. **Structure evals like a pyramid** — cheap deterministic checks at the base, expensive LLM judges at the top
2. **Right-size the judge model** — use small models for classification, reserve premium models for complex judgment
3. **Cascade evaluation** — let cheap checks filter before expensive ones run
4. **Set budget controls early** — graduated response prevents cost surprises
5. **Cache and diff** — never pay to re-evaluate unchanged outputs
6. **Start with deterministic checks** — they catch more than you'd expect, at zero marginal cost
