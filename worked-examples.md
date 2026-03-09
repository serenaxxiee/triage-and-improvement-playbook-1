# Worked Examples

Three end-to-end walkthroughs showing how the playbook layers connect in practice. Each journey starts from a different situation and demonstrates a different diagnostic path.

| Journey | Starting Situation | What It Demonstrates |
|---|---|---|
| [Journey 1](#journey-1-first-eval-run) | First eval run — "What do I do with these results?" | Full flow: interpret → prioritize → triage → remediate → verify |
| [Journey 2](#journey-2-score-plateau) | Stuck after 4 iterations — "Nothing is improving" | Pattern analysis, reclassification, platform limitation workaround |
| [Journey 3](#journey-3-post-update-regression) | Scores dropped after a change — "I broke something" | Regression detection, instruction conflict diagnosis, trade-off resolution |

> **About these examples:** These are illustrative scenarios based on common patterns observed in real customer eval runs. The specific numbers, test case IDs, and agent details are representative composites, not verbatim records of a single customer engagement. The diagnostic paths and remediation strategies shown are the same ones used in practice.

---

## Journey 1: First Eval Run

**Situation:** You've just run your eval suite for the first time on a customer support agent. Here are the results:

| Eval Set | Pass Rate |
|---|---|
| Safety & PII | 100% |
| Core Business Q&A | 87% |
| Knowledge Grounding | 71% |
| Tool Invocation | 92% |
| Trigger Routing | 88% |
| Tone & Quality | 83% |
| Escalation | 90% |
| **Overall** | **85%** |

### Step 1: Interpret Scores (Layer 1)

Using the [score interpretation table](README.md#example-threshold-calibration):

| Eval Set | Score | Threshold | Status |
|---|---|---|---|
| Safety & PII | 100% | 95% blocking | PASS |
| Core Business Q&A | 87% | 80% blocking | PASS |
| Knowledge Grounding | 71% | 80% blocking | **BELOW BLOCKING** |
| Tool Invocation | 92% | 85% blocking | PASS |
| Trigger Routing | 88% | 80% blocking | PASS |
| Tone & Quality | 83% | 75% blocking | PASS |
| Escalation | 90% | 85% blocking | PASS |

**Readiness assessment: ITERATE.** Knowledge Grounding is below its blocking threshold. Focus remediation there.

### Step 2: Prioritize Failures (Layer 2, Step 0)

Knowledge Grounding has 7 test cases. 2 are failing: **KG-003** and **KG-005**.

Both are in a core business eval set, so they're Priority 2. Since there are only 2, triage both.

### Step 3: Triage KG-003 (Layer 2, Steps 1-2)

**Test case KG-003:**
- **Sample input:** "What is your return policy?"
- **Expected answer:** "We offer a 30-day return window for all purchases."
- **Agent's response:** "Our return policy allows returns within 15 business days of purchase."
- **Eval method:** Keyword Match
- **Result:** FAIL (expected "30-day", agent said "15 business days")

**Step 1 — Verify the eval:**

| Question | Answer | Result |
|---|---|---|
| 1.1: Is the agent's response actually acceptable? | Need to check the source document | Check source first |
| 1.2: Is the expected answer still current? | **Source doc says "15 business days"** — the policy was updated | **NO — expected answer is outdated** |

**Classification: Eval Setup Issue** — outdated expected answer.

The agent is correct. The eval is wrong.

### Step 4: Triage KG-005 (Layer 2, Steps 1-2)

**Test case KG-005:**
- **Sample input:** "Does the Premium plan include an extended warranty?"
- **Expected answer:** "The Premium plan includes a 2-year standard warranty. Extended warranty options are available for purchase separately."
- **Agent's response:** "Yes, the Premium plan includes a 3-year extended warranty covering all parts and labor."
- **Eval method:** Compare Meaning
- **Result:** FAIL (agent fabricated warranty details)

**Step 1 — Verify the eval:**

| Question | Answer | Result |
|---|---|---|
| 1.1: Is the agent's response acceptable? | No — "3-year extended warranty" is fabricated | Continue |
| 1.2: Is the expected answer current? | Yes — source confirms 2-year standard warranty | Continue |
| 1.3: Is the test case realistic? | Yes — common customer question | Continue |
| 1.4: Could an alternative response be correct? | No — the warranty details are factual | Continue |
| 1.5: Is the eval method appropriate? | Yes — Compare Meaning is right for semantic accuracy | **Eval is valid** |

**Step 2 — Diagnose the agent:**

| Question | Answer |
|---|---|
| 2.3: Is the source content incorrect? | No — source says "2-year standard warranty" |
| 2.5: Did the agent contradict information in the source? | Sort of — the source says "2-year standard", agent said "3-year extended" |
| 2.4: Did the agent answer without using any source? | Likely yes — the "3-year extended warranty covering all parts and labor" detail doesn't exist in any source |

**Classification: Agent Configuration Issue** — hallucination. The agent generated warranty details not grounded in any knowledge source.

### Step 5: Remediate (Layer 3)

**KG-003 (Eval Setup fix):**
- **Change:** Update expected value from "30-day return window" to "15 business days"
- **Re-run:** KG-003 only
- **Expect:** Pass

**KG-005 (Agent Config fix):**
- **Change:** Add grounding instruction to system prompt: "Only answer based on information found in your knowledge sources. If the information is not available, say so."
- **Re-run:** Full Knowledge Grounding eval set (agent config change can have broader effects)
- **Expect:** KG-005 passes; other test cases should not regress

### Step 6: Verify

After both changes, re-run the Knowledge Grounding eval set:

| Before | After |
|---|---|
| 71% (5/7 passing) | 86% (6/7 passing) |

Knowledge Grounding is now above the 80% blocking threshold. One remaining failure (KG-007) is borderline — investigate in the next iteration.

### Step 7: Document (Layer 4)

Record in the [failure log](templates/failure-log-template.md):

| Test Case ID | Root Cause Type | What Went Wrong | What I Changed | Fixed? |
|---|---|---|---|---|
| KG-003 | Eval Setup | Expected answer outdated (policy changed from 30 days to 15 business days) | Updated expected value | Yes |
| KG-005 | Agent Config | Hallucinated warranty details not in any source | Added grounding instruction to system prompt | Yes |

**Pattern note:** Expected values should be verified against source documents before each eval run. Add this to the pre-eval checklist.

**Readiness re-check:** All eval sets now above blocking thresholds. Readiness assessment: **SHIP WITH KNOWN GAPS** (KG-007 documented, monitoring plan in place).

---

## Journey 2: Score Plateau

**Situation:** You've run 4 iterations on a product support agent. Factual accuracy has been stuck at 78% across all 4 runs. You've made prompt changes after each run with no improvement.

### Step 1: Check Patterns (Layer 4)

Review the failure log across all 4 iterations:

| Iteration | Score | Changes Made | Result |
|---|---|---|---|
| 1 | 78% | (baseline) | — |
| 2 | 79% | Added "be precise about product specifications" | No meaningful change |
| 3 | 77% | Reorganized prompt to put accuracy instructions first | No meaningful change |
| 4 | 78% | Added worked examples of correct product answers | No meaningful change |

**Trend: Flat.** Remediation isn't targeting the real root cause.

### Step 2: Analyze Failing Test Cases

Look at the 6 persistent failures across all iterations:

| Test Case | Failing Since | What Goes Wrong |
|---|---|---|
| FA-002 | Iteration 1 | Agent cites FAQ page instead of product manual |
| FA-005 | Iteration 1 | Agent cites FAQ page instead of product manual |
| FA-008 | Iteration 1 | Agent cites FAQ page instead of product manual |
| FA-011 | Iteration 1 | Agent cites FAQ page instead of product manual |
| FA-014 | Iteration 1 | Agent cites FAQ page instead of product manual |
| FA-019 | Iteration 2 | Agent gives partial answer from FAQ, misses manual detail |

**Concentration analysis:** 5 of 6 failures (83%) involve the same root cause — the agent retrieves from the FAQ page instead of the product manual.

### Step 3: Re-Triage (Layer 2)

Originally classified all as "Agent Configuration Issue: wrong source retrieved" (diagnostic question 2.1).

But you've now tried multiple configuration changes (prompt rewording, reordering, examples) with no improvement. Check the platform limitation indicators:

| Indicator | Check |
|---|---|
| Same failure persists across multiple prompt/config variations | **YES** — 4 iterations, no change |
| Retrieval consistently returns wrong documents despite correct source config | **YES** — FAQ always retrieved instead of manual |

**Reclassification:** This is a **Platform Limitation** — retrieval ranking issue. The platform's retrieval consistently prefers the FAQ page over the product manual for these queries, and no prompt change can fix retrieval behavior.

### Step 4: Remediate (Layer 3 — Platform Limitation)

**Workaround strategy:**
1. Restructure the product manual with clearer headings that match the vocabulary users use in queries
2. Duplicate critical product specifications from the manual into the FAQ document, creating redundant retrieval paths
3. Shorten manual sections so each addresses one specific question (better retrieval chunk matching)

**Escalation:**
- Document the limitation: "Queries about [product specs] consistently retrieve the FAQ page (last updated: [date], [N] pages) instead of the product manual ([date], [N] pages), despite the manual containing the authoritative information."
- Include evidence: 5 test case examples with query, expected source, and actual source retrieved
- Submit to platform team

### Step 5: Verify

After restructuring the product manual and adding redundant FAQ entries:

| Before | After |
|---|---|
| 78% (stuck for 4 iterations) | 89% |

Workaround is effective. 1 failure remains (FA-019) — the query is too ambiguous for reliable retrieval even with restructured documents. Documented as a known limitation.

### Step 6: Document

Update failure log:

| Test Case | Root Cause Type | What Went Wrong | What I Changed | Fixed? |
|---|---|---|---|---|
| FA-002, 005, 008, 011, 014 | Platform Limitation | Retrieval ranking prefers FAQ over product manual | Restructured manual headings; duplicated critical specs in FAQ | Yes |
| FA-019 | Platform Limitation | Ambiguous query can't reliably retrieve correct source | Documented as known limitation | No — known gap |

**Key learning:** When scores are flat across multiple iterations of prompt changes, the root cause is likely not in the prompt. Check for infrastructure or platform issues before investing more prompt engineering effort.

---

## Journey 3: Post-Update Regression

**Situation:** You updated your system prompt to improve tone scores. Tone went up, but factual accuracy dropped below its blocking threshold.

### Before the Change

| Eval Set | Score |
|---|---|
| Factual Accuracy | 91% |
| Tone & Quality | 83% |
| All others | Above threshold |

### After the Change

You added: *"Always acknowledge the customer's concern and show empathy before providing your answer. Begin every response by validating the customer's experience."*

| Eval Set | Before | After | Delta |
|---|---|---|---|
| Factual Accuracy | 91% | 76% | **-15%** |
| Tone & Quality | 83% | 91% | +8% |

### Step 1: Interpret (Layer 1)

Factual Accuracy at 76% is now **below the 80% blocking threshold**. Cannot ship. This is a regression.

### Step 2: Check Patterns (Layer 4)

Cross-signal pattern match: **"Accuracy improving but tone degrading"** (inverse of current situation — "tone improving but accuracy degrading").

Indicated root cause: **Instruction conflict** — new accuracy/tone instructions may be competing for the model's attention.

### Step 3: Triage the New Failures (Layer 2)

Review the factual accuracy test cases that newly failed (passed before, fail now):

**Test case FA-007:**
- **Input:** "What's the maximum file upload size?"
- **Expected:** "The maximum file upload size is 25 MB for standard accounts and 100 MB for enterprise accounts."
- **Agent before:** "The maximum file upload size is 25 MB for standard accounts and 100 MB for enterprise accounts."
- **Agent after:** "I completely understand your concern about file upload sizes — it can be really frustrating when you're trying to upload important documents! I want to make sure you have all the information you need. The maximum upload size is 25 MB for standard plans."

**Step 1 — Verify the eval:** Expected answer is correct. Eval is valid. The agent's new response is missing the enterprise account detail.

**Step 2 — Diagnose:** The agent is spending so much of its response on empathy preamble that it truncates the factual content. The new tone instruction ("begin every response by validating the customer's experience") is consuming response space and model attention at the expense of completeness.

**Classification: Agent Configuration Issue** — instruction conflict between tone and accuracy guidance.

### Step 4: Remediate (Layer 3)

The problem isn't that tone instructions are wrong — it's that they compete with accuracy. The fix is to **separate and prioritize**:

**Old instruction (single, competing):**
> "Always acknowledge the customer's concern and show empathy before providing your answer. Begin every response by validating the customer's experience."

**New instruction (separated, prioritized):**
> "Always include the complete factual answer to the customer's question. Do not omit details for brevity. Additionally, when the customer expresses frustration or concern, briefly acknowledge it."

Key changes:
- Accuracy instruction comes first (priority)
- "Complete factual answer" is now explicit
- Empathy is conditional ("when the customer expresses frustration") rather than universal ("every response")
- "Briefly" prevents empathy from consuming the response

### Step 5: Verify

Re-run the **full eval suite** (this was a system prompt change — broad impact possible):

| Eval Set | Before Change | After Regression | After Fix |
|---|---|---|---|
| Factual Accuracy | 91% | 76% | 90% |
| Tone & Quality | 83% | 91% | 89% |
| All others | Above threshold | Above threshold | Above threshold |

Both signals are now above their blocking thresholds. Tone didn't fully return to the 91% peak, but 89% is well above the 75% blocking threshold and represents a net improvement from the original 83%.

### Step 6: Document

| Test Case | Root Cause Type | What Went Wrong | What I Changed | Fixed? |
|---|---|---|---|---|
| FA-007, FA-012, FA-018 (and others) | Agent Config | Tone instruction competed with accuracy; agent prioritized empathy over completeness | Restructured prompt: accuracy first, conditional empathy | Yes |

**Key learning:** System prompt changes should always be verified against the full eval suite, not just the target eval set. Instructions compete for model attention — when adding guidance for one quality signal, check for regression in others.

**Pattern to watch for:** This is an instance of the [instruction budget problem](remediation-mapping.md#the-instruction-budget-problem). As the system prompt grows, instruction conflicts become more likely. Consider consolidation and simplification after each major prompt update.

---

## Common Patterns Across Journeys

Each journey starts from a different situation to demonstrate a different diagnostic path. For a continuous view of one agent through all four layers — score interpretation, triage, remediation, and verification — follow Journey 1, which is the most complete end-to-end example.

| Pattern | Where It Appears | Takeaway |
|---|---|---|
| **Check the eval before the agent** | Journey 1 (KG-003) | The most common time waste is debugging the agent when the eval is wrong |
| **Flat scores = wrong root cause** | Journey 2 | If remediation isn't working, re-classify — you may be fixing the wrong thing |
| **Test the full suite after prompt changes** | Journey 3 | Prompt changes have broad blast radius; always check for regressions |
| **Document everything** | All journeys | The failure log prevents rediscovering the same root causes each iteration |
| **Known gaps are OK** | Journey 1 (KG-007), Journey 2 (FA-019) | Not every failure needs to be fixed before shipping — document and monitor |
