# Layer 2: Failure Triage

> **"Why did this fail and who needs to act?"**

This is the core diagnostic engine of the playbook. For each failing test case, work through the steps below to classify the root cause, identify the owner, and determine the fix.

**Before you start:** Make sure you've reviewed [Layer 1: Score Interpretation](README.md#layer-1-score-interpretation--readiness-assessment) to understand which eval sets need attention and which failures to prioritize.

---

## Pre-Triage Check: Verify Infrastructure Was Healthy

Before entering the triage process, confirm that all dependencies were functioning during the eval run. Infrastructure failures (API timeouts, knowledge source indexing failures, connector outages) produce eval results that **look like** agent failures but have nothing to do with the agent or the eval design.

**Quick checks:**
- [ ] Were all knowledge sources accessible and fully indexed during the run?
- [ ] Did any API backends or connectors return errors, timeouts, or rate-limiting responses?
- [ ] Were authentication tokens valid throughout the run?
- [ ] Did the eval environment match the intended configuration (e.g., correct version of agent published)?

> If any infrastructure dependency was unhealthy, **re-run the eval after fixing the dependency** before triaging failures. Triaging results from a broken infrastructure run wastes time on phantom failures.

---

## Step 0: Prioritize Your Failures

Before triaging individual failures, decide where to focus:

| Priority | Triage First | Rationale |
|---|---|---|
| 1 | Safety & compliance failures | Highest consequence; must be resolved before ship |
| 2 | Core business scenario failures (highest-priority test cases) | Direct impact on agent value proposition |
| 3 | Failures in the eval set with the lowest pass rate | Likely systemic issue — fixing the root cause may resolve multiple failures |
| 4 | Recurring failures (same test case failing across multiple runs) | Consistent failures are most diagnosable |
| 5 | Capability scenario failures | Important but lower blast radius |

> **If you have many failures (15+):** Don't triage every one individually. Start with the lowest-scoring eval set. Manually review 3-5 failures from that set. If they share a root cause, fix that root cause and re-run — many individual failures will likely resolve together.

---

## Identifying Which Quality Signal a Failure Tests

If your eval results show "test case X failed" without an obvious quality signal mapping, use the test case's eval set and method to identify the signal:

| The test case is in this eval set... | And uses this method... | It tests this quality signal |
|---|---|---|
| Core Business Q&A | Keyword Match + Compare Meaning | Factual accuracy |
| Knowledge Grounding | Capability Use + Compare Meaning | Source retrieval / hallucination prevention |
| Tool Invocation | Capability Use | Tool call correctness |
| Triage & Routing | Capability Use | Trigger routing accuracy |
| Safety & PII | Compare Meaning + General Quality | Boundary enforcement / PII protection |
| Tone & Quality | General Quality + Compare Meaning | Tone, empathy, response structure |
| Escalation & Failure | Capability Use + General Quality | Escalation judgment / fallback behavior |

> This table maps to the eval sets in the [scenario library](../). If your eval set names differ, match by method type and what the expected value is checking for.

---

## Step 1: Verify the Eval ("Is my test valid?")

**Always start here.** Before investigating the agent, verify the eval setup is sound.

For each failure, manually review the agent's actual response alongside the expected value and the grading method. Ask these questions in order:

| # | Diagnostic Question | If YES | If NO |
|---|---|---|---|
| 1.1 | Is the agent's actual response acceptable (i.e., would a real user be satisfied with this answer), even though it failed the eval? | **Eval Setup Issue:** grader or expected value is wrong | Continue to 1.2 |
| 1.2 | Is the expected answer still current and accurate against the source? | Continue to 1.3 | **Eval Setup Issue:** expected answer is outdated or wrong |
| 1.3 | Does the test case represent a realistic user input? | Continue to 1.4 | **Eval Setup Issue:** test case is unrealistic |
| 1.4 | Could a reasonable alternative response also be correct, but the grader doesn't allow for it? | **Eval Setup Issue:** grader too rigid, doesn't account for valid variations | Continue to 1.5 |
| 1.5 | Is the eval method appropriate for what you're trying to test? | **Eval is valid** — proceed to [Step 2](#step-2-diagnose-the-agent-whats-misconfigured) | **Eval Setup Issue:** wrong eval method for this quality signal |

> **Not sure if the response is "acceptable"?** Here are some practical signals to help you decide (questions 1.1 and 1.4):
> - The agent's response contains the same key facts as the expected answer but uses different wording → **usually acceptable** (the grader may be too rigid)
> - The agent's response is missing critical information that is in the expected answer → **usually NOT acceptable**
> - Reasonable people could disagree on whether the response is good enough → the acceptance criteria may be ambiguous (flag for 1.4)
> - When in doubt, compare the agent's response to the original source document, not just the expected answer — the source is the ground truth
>
> These are signals to inform your judgment, not substitutes for it.

### Common Eval Setup Failure Sub-Types

| Sub-Type | Description | Example |
|---|---|---|
| **Outdated expected answer** | Source content changed but expected value wasn't updated | Policy updated to 15 days; eval still expects "30-day return window" |
| **Overly rigid grader** | Keyword match fails on a valid synonym or rephrasing | Expected "cold water"; agent said "cool water, 30 degrees C" — semantically correct |
| **Unrealistic test case** | Test scenario doesn't match actual user behavior | Testing a 4-paragraph query when real users type 5-10 words |
| **Wrong eval method** | Eval method doesn't match what you're actually testing | Using Keyword Match (All) for a synthesis question where Compare Meaning is appropriate |
| **Grader factual error** | LLM-as-judge invents a failure reason that isn't real (isolated error) | LLM grader says "response doesn't mention the return policy" when it clearly does |
| **Grader systematic bias** | LLM-as-judge applies an inconsistent standard across test cases (calibration problem) | Grader passes short responses but fails longer ones for the same quality signal, regardless of content |
| **Ambiguous acceptance criteria** | Expected value can be interpreted multiple ways | "Should include pricing information" — monthly? annual? per-user? All of the above? |

### Grader Validation

Grader reliability is a prerequisite for trustworthy triage. If the grader itself is unreliable, you'll misdiagnose every failure it touches.

**How to validate your graders:**
1. Take 5-10 test cases where you **know** the correct verdict (pass or fail) from manual review
2. Run the eval and check whether the grader agrees with your manual verdict
3. If the grader disagrees on > 20% of cases, the grader needs recalibration before any agent debugging

**Signs your grader needs attention:**
- The same test case produces different verdicts across runs (high variance)
- Failures cluster around test cases using LLM-as-judge while deterministic methods (Keyword Match, Capability Use) pass
- The grader flags issues you can't reproduce when you read the agent's response

**Grader recalibration options:**
- Switch from LLM-as-judge to a deterministic method where possible
- Add explicit "this is acceptable" and "this is not acceptable" examples to the rubric
- Broaden keyword sets to include synonyms and valid rephrasings
- Use Compare Meaning instead of Keyword Match (All) for semantic equivalence checks

---

## Step 2: Diagnose the Agent ("What's misconfigured?")

The eval is valid and the agent genuinely produced a bad response. Now diagnose what went wrong in the agent's configuration.

> **Observability note:** Many of the diagnostic questions below require visibility into what the agent did internally — which knowledge source it retrieved from, which tool it called, which topic triggered. Consult your platform's trace logs, conversation transcripts, or test analytics to find this information. If your platform does not expose this level of detail, you may need to infer from the agent's response (e.g., if the response contains information only available in Source A, it likely retrieved from Source A).

### Factual Accuracy / Knowledge Grounding Failures

| # | Question | If YES → Root Cause |
|---|---|---|
| 2.1 | Did the agent retrieve from the wrong knowledge source? | Knowledge source configuration — wrong source indexed or prioritized |
| 2.2 | Did the agent retrieve the right source but extract the wrong information? | Prompt/instruction gap — model needs extraction guidance |
| 2.3 | Is the source content itself incorrect or outdated? | Knowledge source content — update the source document |
| 2.4 | Did the agent answer without using any knowledge source (made up an answer)? | Source accessibility — source not indexed, or query phrasing doesn't match source vocabulary |
| 2.5 | Did the agent contradict information that is in the source? | Hallucination — add explicit grounding instruction |

### Tool Invocation Failures

| # | Question | If YES → Root Cause |
|---|---|---|
| 2.6 | Did the wrong tool fire? | Tool description ambiguity — descriptions overlap between tools |
| 2.7 | Did the right tool fire with wrong parameters? | Parameter definition — schema or description unclear |
| 2.8 | Did the tool not fire at all? | Trigger condition — input doesn't meet invocation criteria |
| 2.9 | Did the tool fire when it shouldn't have? | Negative guardrail missing — no instruction for when NOT to call the tool |
| 2.10 | Did the tool fire correctly but the response misused the output? | Response instruction — agent needs guidance on formatting tool outputs |
| 2.11 | Did the tool fire correctly but the tool itself failed (error, timeout, bad data)? | **Tool/integration issue** — the failure is in the backend system, not the agent. Fix the tool, not the agent. |

### Trigger Routing Failures

| # | Question | If YES → Root Cause |
|---|---|---|
| 2.12 | Did the wrong topic fire? | Topic trigger overlap — triggers are ambiguous between topics |
| 2.13 | Did no topic fire (hit fallback)? | Topic coverage gap — no topic handles this input type |
| 2.14 | Did multiple topics match with wrong disambiguation? | Disambiguation logic — priority or clarification flow misconfigured |

### Tone & Response Quality Failures

| # | Question | If YES → Root Cause |
|---|---|---|
| 2.15 | Is the agent's tone inconsistent with system prompt guidance? | Tone instruction gap — guidance missing or contradictory |
| 2.16 | Is the response too verbose or too terse for the question? | Format instruction — add length/structure guidance |
| 2.17 | Does the agent lack empathy in sensitive contexts? | Empathy instruction gap — add explicit guidance for emotional inputs |
| 2.18 | Is the response structurally poor (wall of text, no steps)? | Format instruction — add formatting requirements |

### Safety & Boundary Failures

| # | Question | If YES → Root Cause |
|---|---|---|
| 2.19 | Did the agent reveal system information? | System prompt protection — add "do not reveal" instructions |
| 2.20 | Did the agent go out of scope? | Scope definition gap — boundaries not clearly defined |
| 2.21 | Did the agent comply with prompt injection? | Safety instructions — add adversarial resistance guidance |
| 2.22 | Did the agent handle PII incorrectly? | PII handling rules — add data protection instructions |

### Escalation & Graceful Failure

| # | Question | If YES → Root Cause |
|---|---|---|
| 2.23 | Did the agent fail to escalate when it should have? | Escalation trigger — criteria not defined or too narrow |
| 2.24 | Did the agent escalate prematurely? | Escalation threshold — criteria too sensitive |
| 2.25 | Did escalation lose conversation context? | Handoff configuration — context preservation not set up |
| 2.26 | Did the agent loop instead of acknowledging failure? | Fallback logic — retry limit or fallback behavior not configured |

**After diagnosing:** Go to [Remediation Mapping (Layer 3)](remediation-mapping.md) for the specific fix corresponding to your root cause.

---

## Step 3: Identify Platform Limitations ("Is this something I can't fix?")

If you've verified the eval is correct AND tried reasonable agent configuration fixes without improvement, the issue may be a platform limitation.

**Indicators of a platform limitation:**

| Indicator | What It Suggests |
|---|---|
| Same failure persists across multiple prompt/config variations | Not a configuration problem |
| Retrieval consistently returns wrong documents despite correct source config | Retrieval ranking limitation |
| Agent cannot perform the required reasoning despite clear instructions | Model capability boundary |
| The required orchestration pattern isn't supported by any configuration option | Orchestration logic constraint |
| LLM-as-judge grader consistently misclassifies despite rubric tuning | Grader model limitation |

**Action path for platform limitations:**
1. Document the limitation clearly (what fails, what you've tried, evidence it's not configuration)
2. Design a workaround if possible (e.g., restructure the source document to improve retrieval)
3. Adjust the eval threshold or flag the test case as "known limitation" — don't let it block unrelated improvements
4. Escalate to the platform team with documentation
5. Track in the [failure log](templates/failure-log-template.md) for re-evaluation when platform capabilities update

**After classifying:** Go to [Remediation Mapping (Layer 3)](remediation-mapping.md#platform-limitation-response) for workaround and escalation guidance.

---

## When a Failure Doesn't Fit the Framework

Not every failure will classify cleanly into the three root cause types. If you've worked through Steps 1-3 and cannot classify the failure, this is expected. Common "unclassifiable" scenarios:

- **Backend data quality issues** — the knowledge source content is technically correct but ambiguously written, so neither the agent nor the eval is "wrong"
- **Intermittent infrastructure failures** — network timeouts, API rate limiting, connector hiccups that don't reproduce consistently
- **Model version changes** — agent behavior changed after a platform model update that you didn't initiate
- **Ambiguous test cases** — the scenario is genuinely ambiguous and reasonable people disagree on the correct answer

**What to do:** Document what you observed (the failure, the agent's response, what you checked). Flag it as "unclassified" in the [failure log](templates/failure-log-template.md). If it recurs, it will likely become classifiable with more data. If it's a one-off, it may be noise. Do not spend unbounded time forcing a classification — move on to failures you can diagnose.

---

## Handling Compound Causes

A single failure can have multiple contributing root causes. For example:
- A factual accuracy failure where the expected answer is slightly outdated (eval setup) AND the knowledge source is also incomplete (agent config)
- A tool invocation failure where the tool description is ambiguous (agent config) AND the orchestration doesn't support conditional tool calls (platform limitation)

**Approach:** Complete the full triage for each failure. If multiple root cause types apply, address them in priority order:

1. **Fix the eval first** — so you get clean signal on whether the agent change actually helps
2. **Fix the agent config** — so you can determine if the remaining failure is truly a platform issue
3. **Document the platform limitation** — only after 1 and 2 are addressed

After each fix, re-run the affected test cases before proceeding to the next root cause layer.

---

## Multi-Turn Conversation Failures

Many agent deployments involve multi-turn conversations where failures are contextual — they only emerge across turns, not within a single turn. The triage framework above applies to multi-turn failures with additional diagnostic considerations.

### When to Suspect a Multi-Turn Issue

- The agent answers correctly in early turns but contradicts itself later
- Context from a previous tool call or knowledge retrieval is lost in a later turn
- Escalation timing only makes sense when considering the full conversation history
- Agent tone degrades progressively as the conversation lengthens
- The agent asks for information the user already provided

### Key Diagnostic Distinction

**Turn of failure vs. turn of root cause.** A failure may manifest in turn 5 but the root cause is in turn 2 (e.g., the agent misclassified the issue in turn 2, which causes wrong tool invocation in turn 5). When triaging multi-turn failures, trace back to identify which turn introduced the error.

### Additional Diagnostic Questions

| # | Question | If YES → Root Cause |
|---|---|---|
| M.1 | Did the failure depend on information from a previous turn that was lost? | Context management — conversation state not preserved across turns |
| M.2 | Did the agent contradict something it said in an earlier turn? | Consistency — no instruction to maintain coherence across turns |
| M.3 | Did the agent re-ask for information the user already provided? | Context retrieval — agent not referencing earlier conversation turns |
| M.4 | Did the failure only appear after many turns (5+)? | Context window — conversation may be exceeding effective context length |

### Remediation for Multi-Turn Issues

- **Context loss** → Check conversation state configuration; ensure tool outputs and key facts are persisted across turns
- **Contradictions** → Add consistency instruction: "Maintain consistency with your previous responses in this conversation"
- **Re-asking** → Verify the platform's conversation memory configuration
- **Long-conversation degradation** → Consider conversation summarization or context pruning strategies

---

## Validating Passing Test Cases (False Positive Check)

The triage framework above focuses on failures (test cases that didn't pass). But **a test case that passes incorrectly** — the grader said PASS but the agent's response was actually bad — is a more dangerous problem because it creates invisible quality gaps.

**Recommended practice:** Manually review 5-10% of passing test cases per eval run, especially for:
- Test cases using LLM-as-judge grading (highest risk of false positives)
- Quality signals where the grader's judgment is subjective (tone, helpfulness)
- Test cases that previously failed and now pass after a change (verify the pass is genuine, not a grader artifact)

If you find false positives, the grader needs recalibration — see [Grader Validation](#grader-validation) above.

---

## What's Next

- **You've classified your failures?** Go to [Remediation Mapping (Layer 3)](remediation-mapping.md) for specific fix actions.
- **You want to look for patterns across failures?** Go to [Pattern Analysis (Layer 4)](pattern-analysis.md).
- **You want to see this flow in action?** Go to [Worked Examples](worked-examples.md).
