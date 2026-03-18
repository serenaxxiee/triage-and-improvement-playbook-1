# Evaluation Tooling Guide

> **"Which evaluation tools should I use, and when?"**

This guide helps you choose the right evaluation tools for your agent's maturity stage and your team's specific needs. It complements the [triage decision tree (Layer 2)](triage-decision-tree.md) and [remediation mapping (Layer 3)](remediation-mapping.md) by connecting diagnostic workflows to the tools that support them.

**Before you start:** You should have a working evaluation setup and understand which quality signals matter for your agent. If you haven't run an evaluation yet, start with the [scenario library](https://github.com/microsoft/ai-agent-eval-scenario-library) to design your first eval set.

---

## How This Guide Is Organized

1. [Tool Selection by Evaluation Stage](#tool-selection-by-evaluation-stage) — match tools to your workflow phase
2. [Platform Comparison Matrix](#platform-comparison-matrix) — feature-by-feature comparison
3. [Decision Flowchart](#decision-flowchart) — quick path to a recommendation
4. [Tool Profiles](#tool-profiles) — detailed analysis of each platform
5. [Integration Patterns](#integration-patterns) — how tools fit into your pipeline
6. [Cost Considerations](#cost-considerations) — budget planning guidance

---

## Tool Selection by Evaluation Stage

Different tools excel at different stages of the evaluation lifecycle. Most teams need at least two tools — one for development-time evaluation and one for production monitoring.

| Evaluation Stage | What You Need | Recommended Tools |
|---|---|---|
| **Early development** — writing first evals, iterating on prompts | Fast feedback loops, easy metric definition, low setup cost | DeepEval, RAGAS (for RAG), Braintrust |
| **CI/CD integration** — blocking bad deploys | Pytest-compatible assertions, GitHub Actions integration, regression detection | DeepEval, Braintrust |
| **Pre-production simulation** — stress-testing before launch | Multi-scenario simulation, synthetic user generation, edge case discovery | Maxim AI, Braintrust |
| **Production monitoring** — tracking live quality | Tracing, alerting, drift detection, low-latency scoring | Arize Phoenix, Langfuse, Fiddler, LangSmith |
| **Human review** — expert annotation of agent outputs | Annotation queues, inter-rater agreement tracking, labeling workflows | LangSmith, Truesight, Braintrust |
| **Compliance & audit** — regulatory documentation | Audit trails, reproducibility logs, data residency controls | Fiddler, W&B Weave, Langfuse (self-hosted) |

---

## Platform Comparison Matrix

### Core Capabilities

| Tool | Dev Eval | CI/CD | Production Monitoring | Human Review | Self-Hostable | Agent-Specific |
|---|---|---|---|---|---|---|
| **Arize Phoenix** | ★★★☆ | ★★☆☆ | ★★★★★ | ★★☆☆ | ✅ Yes | ★★★★☆ |
| **Braintrust** | ★★★★☆ | ★★★★★ | ★★★★☆ | ★★★☆ | ❌ No | ★★★★☆ |
| **Comet Opik** | ★★★☆ | ★★★☆ | ★★★★☆ | ★★☆☆ | ✅ Yes | ★★★☆ |
| **DeepEval** | ★★★★★ | ★★★★★ | ★★☆☆ | ★☆☆☆ | ✅ Yes | ★★★☆ |
| **Fiddler** | ★★☆☆ | ★★☆☆ | ★★★★★ | ★★★☆ | ❌ No | ★★★★☆ |
| **Galileo** | ★★★☆ | ★★★☆ | ★★★★☆ | ★★☆☆ | ❌ No | ★★★☆ |
| **Langfuse** | ★★★☆ | ★★★☆ | ★★★★☆ | ★★★☆ | ✅ Yes | ★★★☆ |
| **LangSmith** | ★★★★☆ | ★★★☆ | ★★★★☆ | ★★★★★ | ❌ No | ★★★★☆ |
| **Maxim AI** | ★★★★☆ | ★★★☆ | ★★★★☆ | ★★★☆ | ❌ No | ★★★★★ |
| **RAGAS** | ★★★★★ | ★★☆☆ | ★☆☆☆ | ★☆☆☆ | ✅ Yes | ★★☆☆ |
| **Truesight** | ★★★☆ | ★★☆☆ | ★★★☆ | ★★★★★ | ❌ No | ★★★☆ |
| **W&B Weave** | ★★★★☆ | ★★★☆ | ★★★☆ | ★★☆☆ | ❌ No | ★★★☆ |

### Pricing Overview (as of March 2026)

| Tool | Free Tier | Paid Starting At | Open Source |
|---|---|---|---|
| **Arize Phoenix** | Self-hosted unlimited | $50/mo (cloud) | ✅ (Elastic License 2.0) |
| **Braintrust** | 1M spans/mo | $249/mo (Pro) | ❌ |
| **Comet Opik** | Self-hosted unlimited | $19/mo (cloud) | ✅ (Apache 2.0) |
| **DeepEval** | Unlimited (library) | $19.99/user/mo (Confident AI cloud) | ✅ (Apache 2.0) |
| **Fiddler** | Guardrails tier | Enterprise (custom) | ❌ |
| **Galileo** | 5K traces/mo | ~$100/mo (Pro) | ❌ |
| **Langfuse** | Self-hosted unlimited | $59/mo (cloud) | ✅ (MIT) |
| **LangSmith** | 5K traces/mo | $39/seat/mo (Plus) | ❌ |
| **Maxim AI** | 10K traces | $9–49/seat/mo | ❌ |
| **RAGAS** | Unlimited | N/A | ✅ (Apache 2.0) |
| **Truesight** | Limited | $19/mo | ❌ |
| **W&B Weave** | Limited | $60/mo | ❌ |

> **Note:** Pricing changes frequently. Verify current pricing on each vendor's website before making purchasing decisions.

---

## Decision Flowchart

Use these questions to narrow your selection:

### 1. What is your primary evaluation need?

- **"I need to test during development and in CI/CD"** → Start with **DeepEval** (pytest-native) or **Braintrust** (experiment comparison UI)
- **"I need to monitor production quality"** → Start with **Arize Phoenix** (OSS, OTel-native) or **Langfuse** (OSS, self-hostable)
- **"I need domain experts to review outputs"** → Start with **LangSmith** (annotation queues) or **Truesight** (no-code criteria)
- **"I need end-to-end agent simulation"** → Start with **Maxim AI** (multi-scenario simulation)

### 2. What are your constraints?

- **Data must stay on-premise** → Langfuse, Arize Phoenix, Comet Opik, or DeepEval (all self-hostable)
- **Budget is near zero** → RAGAS + DeepEval (both Apache 2.0) + Arize Phoenix (self-hosted)
- **Team is non-technical** → Truesight (no-code) or Braintrust (polished UI)
- **Already using LangChain/LangGraph** → LangSmith (native integration)
- **Need audit trails for compliance** → Fiddler (Trust Service) or W&B Weave (local scorers)

### 3. What scale are you operating at?

- **< 1K evals/month** → Any free tier works; optimize for developer experience
- **1K–100K evals/month** → Consider Braintrust, LangSmith, or Langfuse cloud tiers
- **> 100K evals/month** → Self-hosted options (Langfuse, Arize, Opik) or enterprise tiers (Braintrust, Fiddler)
- **> 1M traces/day** → Comet Opik (built for 40M traces/day) or Arize enterprise

---

## Tool Profiles

### DeepEval — Best for Engineering-Led Evaluation

**What it does:** Testing-first evaluation framework that mirrors pytest patterns. Write evaluation criteria as test assertions, run them in CI, block deploys on regressions.

**Key strengths:**
- Unmatched pytest integration — `deepeval test run` feels native to engineering workflows
- 50+ built-in metrics including DAG metrics that map to execution graphs
- G-Eval enables natural language evaluation criteria definition
- CI/CD deployment gating with regression detection

**Key limitations:**
- Library-focused — no built-in production monitoring or tracing
- Every evaluation requires LLM calls (cost scales linearly with dataset size)
- Metric implementation can be opaque; debugging why a metric scored a certain way is not always straightforward

**Best for:** Engineering teams who want evaluation-as-code integrated into existing test workflows.

---

### Braintrust — Best for Experiment Management

**What it does:** Full-lifecycle evaluation platform with side-by-side experiment comparison, dataset versioning, and online evaluation scoring for production traffic.

**Key strengths:**
- Experiment comparison UI makes regression detection intuitive for technical and non-technical stakeholders
- "Loop AI" automates evaluation cycles
- GitHub Actions integration for deployment gating
- Online evaluation bridges offline testing to production monitoring
- Timeline replay shows multi-agent execution sequences

**Key limitations:**
- Cloud-only (no self-hosting) — data residency concerns for some organizations
- Pricing scales with usage volume
- Vendor lock-in risk

**Best for:** Teams needing polished experiment workflows with cross-functional stakeholder review.

---

### Arize Phoenix — Best Open-Source Observability

**What it does:** Trace-first evaluation platform built on OpenTelemetry standards. Deep observability alongside evaluation with embedding-based retrieval analysis.

**Key strengths:**
- OpenTelemetry-native — vendor-agnostic trace collection
- Free self-hosting with unlimited traces
- Excellent retriever analysis through embedding visualization
- Session-level tracking for multi-turn conversations
- Dedicated agent evaluators for tool-use and multi-step evaluation

**Key limitations:**
- Lacks native pytest integration and CI/CD workflow support
- Exploratory strength doesn't translate well to automated testing
- Elastic License 2.0 (not true OSS) — may affect some organizations

**Best for:** Teams wanting deep production observability with zero vendor lock-in.

---

### Langfuse — Best Self-Hosted Alternative

**What it does:** Open-source LLM engineering platform covering tracing, prompt management, dataset management, and evaluation scoring.

**Key strengths:**
- Fully self-hostable (MIT license) — complete data sovereignty
- OpenTelemetry compatible
- Prompt management alongside evaluation
- Growing community and ecosystem
- Dataset management with version control

**Key limitations:**
- Smaller feature set than commercial alternatives
- Self-hosting requires infrastructure management
- Evaluation features less mature than dedicated eval platforms

**Best for:** Infrastructure-savvy teams needing full control over data and privacy.

---

### LangSmith — Best for Human Annotation

**What it does:** Integrated tracing and evaluation platform with native LangChain/LangGraph support and comprehensive human annotation workflows.

**Key strengths:**
- Automatic tracing for LangChain applications
- Best-in-class human annotation queue for large-scale labeling
- Multi-turn agent evaluation with step-level scoring
- Production example promotion to test cases
- Timeline view with per-operation timing

**Key limitations:**
- Evaluation features feel secondary to observability focus
- Weaker integration for non-LangChain applications
- Cloud-only (no self-hosting)

**Best for:** Teams already in the LangChain ecosystem; those needing robust human-in-the-loop annotation workflows.

---

### Maxim AI — Best for Agent Simulation

**What it does:** Comprehensive agent evaluation platform with high-fidelity simulation, synthetic user generation, and visual execution graphs.

**Key strengths:**
- Agent simulation engine tests across thousands of scenarios
- Visual trace view shows agent interactions step-by-step
- Unified offline-to-online evaluation pipeline
- VPC deployment available for data security

**Key limitations:**
- Newer platform — smaller community and ecosystem
- Simulation quality depends on scenario design
- Pricing can scale with simulation volume

**Best for:** Teams building complex multi-step agents who need pre-production stress testing.

---

### RAGAS — Best for RAG Evaluation

**What it does:** Purpose-built evaluation library for Retrieval-Augmented Generation pipelines with specialized metrics.

**Key strengths:**
- Specialized metrics: faithfulness, answer relevancy, context precision/recall
- Lightweight — no platform dependency
- Zero vendor lock-in; fully Apache 2.0
- Low barrier to entry

**Key limitations:**
- Library-only — no dashboard, no dataset versioning, no tracing
- Custom metrics feel bolted on versus core RAG focus
- Every evaluation requires LLM calls (scaling cost concern)

**Best for:** Teams building RAG pipelines who want lightweight, focused evaluation without platform overhead.

---

## Integration Patterns

### Pattern 1: Development + Production (Most Common)

Use separate tools for development-time and production evaluation:

```
Development                         Production
┌─────────────────┐                ┌─────────────────┐
│ DeepEval         │  ──deploy──►  │ Arize Phoenix    │
│ or Braintrust    │               │ or Langfuse      │
│                  │               │                  │
│ • Unit evals     │               │ • Trace analysis │
│ • CI/CD gates    │               │ • Drift alerts   │
│ • Regression     │               │ • Live scoring   │
└─────────────────┘                └─────────────────┘
```

**When to use:** Most teams. Development tools optimize for fast iteration; production tools optimize for scale and observability.

### Pattern 2: Unified Platform

Use a single platform across the lifecycle:

```
┌──────────────────────────────────────────┐
│ Braintrust or LangSmith                  │
│                                          │
│ Dev evals → CI gates → Online scoring    │
│ Experiment tracking across all stages    │
└──────────────────────────────────────────┘
```

**When to use:** Teams wanting simplicity over best-of-breed. Reduces integration overhead at the cost of some capability depth.

### Pattern 3: OSS Stack (Budget-Optimized)

Combine open-source tools for zero licensing cost:

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ DeepEval     │  │ RAGAS       │  │ Langfuse     │
│ (CI/CD)      │  │ (RAG evals) │  │ (Production) │
└─────────────┘  └─────────────┘  └─────────────┘
```

**When to use:** Budget-constrained teams willing to invest integration effort. All components are self-hostable with permissive licenses.

### Pattern 4: Compliance-First

Prioritize audit trails and data residency:

```
┌─────────────────┐  ┌─────────────────┐
│ W&B Weave        │  │ Fiddler          │
│ (Local scorers)  │  │ (Trust Service)  │
│ Data stays local │  │ Audit trails     │
└─────────────────┘  └─────────────────┘
```

**When to use:** Regulated industries (healthcare, finance, government) where data residency and auditability are non-negotiable.

---

## Cost Considerations

### Hidden Costs to Account For

1. **LLM judge costs** — Most eval tools use LLM-as-judge under the hood. At scale, judge API calls can exceed platform licensing costs. Budget $0.01–0.10 per evaluation depending on the judge model.

2. **Self-hosting infrastructure** — Open-source tools avoid licensing fees but require compute, storage, and maintenance. Budget 10–20 hours/month for operational overhead.

3. **Integration engineering** — Multi-tool stacks require glue code. Budget 1–2 weeks for initial setup, plus ongoing maintenance.

4. **Data egress** — Cloud platforms may charge for data export. Verify egress policies before committing large trace volumes.

### Cost Optimization Strategies

- **Start with free tiers** — Most platforms offer generous free tiers. Validate fit before committing budget.
- **Use smaller judge models for screening** — Route only ambiguous cases to expensive judges (see [cost-efficiency evaluation patterns](https://github.com/microsoft/ai-agent-eval-scenario-library) for details).
- **Cache evaluation results** — Many evaluations are deterministic given the same inputs. Cache aggressively.
- **Sample production traces** — You don't need to evaluate every production interaction. Statistical sampling (1–10%) often provides sufficient signal.

---

## Connecting to the Triage Workflow

When using the [triage decision tree](triage-decision-tree.md), your tooling affects how you diagnose failures:

| Triage Step | What Your Tool Should Provide |
|---|---|
| **Identify failing test cases** | Dashboard or CLI showing pass/fail by test case with filtering |
| **Examine agent response** | Full trace with intermediate steps, tool calls, and retrieval results |
| **Compare against expected** | Side-by-side view of expected vs actual, with diff highlighting |
| **Check for systematic patterns** | Aggregation views showing failure concentration by signal, topic, or test type |
| **Verify after remediation** | Before/after comparison with statistical significance indicators |
| **Track iteration history** | Version-controlled datasets and evaluation run history |

If your current tooling doesn't provide visibility into one of these steps, that's a signal to either upgrade your tool or add a complementary one.

---

## Next Steps

- **New to evaluation?** Start with [DeepEval](https://github.com/confident-ai/deepeval) (free, pytest-native) and the [scenario library](https://github.com/microsoft/ai-agent-eval-scenario-library) for eval design guidance
- **Ready for production monitoring?** Add [Langfuse](https://langfuse.com) (self-hosted) or [Arize Phoenix](https://phoenix.arize.com/) alongside your dev eval tool
- **Need to choose between platforms?** Run a 2-week proof-of-concept with your top 2 candidates using the same eval set
- **Have failures to diagnose?** Return to [Layer 2: Failure Triage](triage-decision-tree.md) with your tooling context
