# Layer 4: Pattern Analysis & Continuous Improvement

> **"What systemic issues do my failures reveal?"**

Individual failures are tactical; patterns are strategic. After triaging and classifying individual failures, step back and look at what they reveal in aggregate. This layer helps you identify systemic issues, track trends across iterations, and build institutional knowledge.

**Before you start:** You should have classified individual failures using the [triage decision tree (Layer 2)](triage-decision-tree.md) and identified remediations using [Layer 3](remediation-mapping.md). Pattern analysis is most valuable after you've triaged at least 5+ failures.

---

## Concentration Analysis

After classifying individual failures, look for concentrations across the full set:

| Pattern | What It Indicates | Recommended Action |
|---|---|---|
| **80%+ failures are eval setup issues** | The eval suite needs a calibration overhaul — not agent changes | Pause agent iteration. Audit and fix eval quality first. Then re-run to get clean signal. |
| **80%+ failures are agent config issues in one area** (e.g., all knowledge-related) | Systemic agent configuration gap | Focus remediation on that dimension. Likely an architectural issue (e.g., knowledge source structure) rather than individual test case fixes. |
| **80%+ failures are platform limitations** | The agent is pushing against platform boundaries | Reassess agent scope. Escalate to platform team. Adjust expectations and thresholds for affected signals. |
| **Failures spread evenly across root cause types** | No systemic issue — isolated problems | Work through failures case by case using the [remediation mapping](remediation-mapping.md). |

### How to Do Concentration Analysis

1. Tally your classified failures by root cause type:
   - Eval Setup Issues: ___
   - Agent Configuration Issues: ___
   - Platform Limitations: ___
   - Unclassified: ___
2. Calculate the percentage for each type
3. If any single type is 80%+, that's your systemic issue — fix the category, not individual cases
4. If agent config issues concentrate in one quality signal (e.g., 5 of 6 are knowledge grounding), that points to an architectural root cause

---

## Cross-Signal Patterns

When failures span multiple eval sets, they often point to a shared root cause. Look for these patterns:

| Pattern | What It Likely Indicates | What to Investigate |
|---|---|---|
| **Factual accuracy AND knowledge grounding both failing** | Knowledge source issue — source is wrong, missing, or inaccessible | Check knowledge source configuration, indexing status, and content freshness |
| **Tool invocation AND trigger routing both failing** | Orchestration configuration issue — topics and tools aren't properly connected | Review how topics route to tools; check for disconnected or misconfigured flows |
| **Tone failing but accuracy passing** | Agent gets the right answer but delivers it poorly | Focus on prompt style instructions; accuracy infrastructure is sound |
| **Safety passing but accuracy failing** | Agent may be over-constrained — too cautious, refuses to answer when it should | Review safety instructions for overly broad restrictions that block legitimate answers |
| **Everything passing except edge cases** | Core behavior is solid | Focus on expanding robustness at the margins; this is a good sign |
| **Accuracy improving but tone degrading** | Instruction conflict — new accuracy instructions may be crowding out tone guidance | Review recent prompt changes; see [The Instruction Budget Problem](remediation-mapping.md#the-instruction-budget-problem) |
| **Multiple eval sets all degrading simultaneously** | Likely a single root cause with broad impact | Check for recent system prompt changes, knowledge source updates, or platform model updates |

### What to Do With Cross-Signal Patterns

1. **Identify the shared root cause** — if two signals fail together, they likely share a dependency (knowledge source, prompt section, tool configuration)
2. **Fix the shared dependency** — rather than fixing each signal independently
3. **Re-run both eval sets** after the fix to confirm both improve
4. **If only one improves**, the signals don't actually share a root cause — triage the remaining failures independently

---

## Trend Analysis Across Iterations

Track how scores change across your iteration cycles to understand whether your remediation strategy is working:

| Trend | Interpretation | Action |
|---|---|---|
| **Scores improving across iterations** | Remediation is working | Continue until thresholds met |
| **Scores flat despite changes** | Remediation isn't targeting the real root cause | Re-triage; the root cause classification may be wrong |
| **Scores degrading after a change** | Regression — the change broke something | Roll back the change; investigate what regressed and why |
| **One eval set improving, another degrading** | Trade-off — fixing one dimension hurt another | Investigate the coupling; likely an instruction conflict (see [Journey 3](worked-examples.md#journey-3-post-update-regression)) |
| **Scores fluctuating between runs (> +/-10% variance)** | Grader instability or agent non-determinism | Validate grader reliability first (see [Grader Validation](triage-decision-tree.md#grader-validation)); run minimum 3x per iteration |

### Building a Trend View

After each iteration, record:

| Iteration | Date | Change Made | Eval Set | Score Before | Score After | Delta |
|---|---|---|---|---|---|---|
| 1 | ___ | ___ | ___ | ___% | ___% | ___ |
| 2 | ___ | ___ | ___ | ___% | ___% | ___ |
| 3 | ___ | ___ | ___ | ___% | ___% | ___ |

This table lets you:
- See whether you're converging toward thresholds
- Identify when a change caused a regression
- Detect plateau patterns early (see [Journey 2: Score Plateau](worked-examples.md#journey-2-score-plateau))

---

## Failure Documentation

Structured failure records build institutional knowledge across the iteration loop. The second time a knowledge grounding issue appears, you already know what to check first. Without documentation, you rediscover the same root causes every iteration.

### Why Document Failures

- **Speed up future triage:** Known failure patterns are immediately recognizable
- **Build escalation evidence:** Accumulated platform limitation records make stronger cases to the platform team
- **Enable team learning:** When multiple people work on the same agent, the log prevents duplicate investigation
- **Track known gaps:** Failures classified as "won't fix" or "known limitation" need to be tracked, not forgotten

### Using the Template

See the [Failure Log Template](templates/failure-log-template.md) for both a lightweight version (small teams, single agent) and a detailed version (larger teams, institutional knowledge).

### What to Record

At minimum, capture for each triaged failure:
1. **Which test case** failed
2. **What root cause type** you classified it as
3. **What went wrong** specifically
4. **What you changed** to fix it
5. **Whether the fix worked**

For unresolved failures, also record:
- What you've tried so far
- Why it remains unresolved
- When to re-evaluate (e.g., "after platform update X")

---

## Continuous Improvement Workflow

After completing a full triage-and-remediation cycle, use this checklist to confirm you've captured the value:

### Post-Iteration Checklist

- [ ] All triaged failures are recorded in the failure log
- [ ] Root cause concentrations have been identified and noted
- [ ] Cross-signal patterns have been checked
- [ ] Scores have been recorded for trend tracking
- [ ] Known limitations are documented with workarounds
- [ ] Next iteration priorities are identified based on remaining failures
- [ ] Re-run schedule is set (which eval sets, when)

### When to Stop Iterating

Refer back to [Layer 1: When Are You Done Iterating?](README.md#when-are-you-done-iterating) for the full criteria. In summary:

**Stop when:**
- All eval sets are above thresholds
- Known gaps are documented
- Scores are consistent (< 5% variance)
- No open agent config issues for blocking signals

**Don't stop when:**
- You haven't investigated persistent failures
- You removed hard test cases to hit thresholds
- Platform limitations aren't documented

---

## What's Next

- **Want to see pattern analysis in action?** See [Journey 2: Score Plateau](worked-examples.md#journey-2-score-plateau) and [Journey 3: Post-Update Regression](worked-examples.md#journey-3-post-update-regression).
- **Need the failure log template?** Go to [templates/failure-log-template.md](templates/failure-log-template.md).
- **Starting fresh with a new eval run?** Go back to [Layer 1: Score Interpretation](README.md#layer-1-score-interpretation--readiness-assessment).
