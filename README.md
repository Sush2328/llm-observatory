# LLM Cost Attribution and Routing Observatory

An AI operations observability platform for tracking inference spend, attributing costs across teams and workloads, and making model routing decisions transparent and auditable.

Built to explore a problem I kept running into: as enterprise AI adoption accelerates, the operational and economic controls haven't kept pace. Most teams know they're spending on LLMs. Very few know *where*, *why*, or *whether the spend is justified*.

---

## Problem Statement

Enterprise AI adoption introduces a class of operational challenges that most observability tooling doesn't address well:

- **Uncontrolled inference spend** — API costs accumulate across teams with no attribution or chargeback framework
- **Weak workload routing** — premium reasoning models handle tasks that don't require them, inflating cost-per-inference
- **Prompt inefficiency** — duplicate retrieval chunks, oversized system prompts, and excessive context retention waste significant token budget
- **Provider concentration risk** — over-reliance on a single provider creates operational exposure during outages or pricing changes
- **Routing opacity** — model selection decisions are implicit, making cost optimisation and audit trails difficult

---

## Why Existing Tooling Falls Short

Most LLM observability platforms (LangSmith, Helicone, etc.) focus on tracing and debugging. They answer: *what happened?*

This platform focuses on: *why did it cost this much, was the routing decision correct, and where is spend leaking?*

The gap is the combination of:
- routing logic with explainable decisions
- inference economics and unit cost tracking
- governance controls and anomaly detection
- operational simulation (what happens when a provider goes down?)

---

## System Design

```
┌─────────────────────────────────────────────────────────┐
│                    API Usage Ingestion                   │
│         (team, model, tokens_in, tokens_out,            │
│          use_case, timestamp, latency)                   │
└──────────────────────┬──────────────────────────────────┘
                       │
          ┌────────────▼────────────┐
          │    Provider Abstraction  │
          │  OpenAI / Anthropic /   │
          │  Google / Mistral        │
          └────────────┬────────────┘
                       │
     ┌─────────────────▼──────────────────┐
     │         Routing Engine             │
     │  Weighted multi-factor scoring:    │
     │                                    │
     │  Score = 0.35 × WorkloadComplexity │
     │        + 0.25 × ContextRequirement │
     │        + 0.20 × LatencySensitivity │
     │        + 0.20 × CostSensitivity    │
     │                                    │
     │  Produces: selected model,         │
     │  rejected alternatives, projected  │
     │  savings, confidence score         │
     └─────────────────┬──────────────────┘
                       │
     ┌─────────────────▼──────────────────┐
     │       Attribution Engine           │
     │  Cost by team / use-case / model   │
     │  Chargeback allocation             │
     │  Committed-use utilisation         │
     │  Week-over-week trend              │
     └─────────────────┬──────────────────┘
                       │
     ┌─────────────────▼──────────────────┐
     │     Prompt Efficiency Analysis     │
     │  Duplicate retrieval detection     │
     │  System prompt caching opportunity │
     │  Context window utilisation        │
     │  Over-spec detection               │
     └─────────────────┬──────────────────┘
                       │
     ┌─────────────────▼──────────────────┐
     │      Governance Dashboard          │
     │  Anomaly alerts                    │
     │  API key audit                     │
     │  Model proliferation tracking      │
     │  Incident simulation + failover    │
     └────────────────────────────────────┘
```

---

## Routing Logic

Routing decisions are scored across four weighted factors:

| Factor | Weight | Rationale |
|---|---|---|
| Estimated workload complexity | 35% | Primary driver — reasoning-heavy tasks justify premium models |
| Context requirement | 25% | Large contexts favour providers with wider windows and better caching |
| Latency sensitivity | 20% | SLA-critical workloads require faster inference paths |
| Cost sensitivity | 20% | Budget-constrained workloads route to cheaper models when quality is equivalent |

Every routing decision logs: selected model, score, rejected alternatives with scores, projected cost delta, and confidence level. This makes routing auditable and reversible — important for governance.

**Example output:**

```
Request routed to: claude-sonnet-4
Confidence: 91%
Projected cost: $0.0038/call

Factors:
  Workload complexity:   62/100  (medium — below Opus threshold of 80)
  Context requirement:   58/100  (180K — exceeds GPT-4.1 efficient window)
  Latency sensitivity:   75/100  (within 2s SLA)
  Cost sensitivity:      88/100  (budget-constrained workload)

Rejected alternatives:
  claude-opus-4   — complexity score below selection threshold; 31% cost premium unwarranted
  gpt-4.1         — context requirement exceeded efficient window (128K limit)
```

---

## Simulated Case Study: Compliance Screening Workload

**Input:**
- 420 calls/month, avg 680 tokens in / 180 tokens out
- Currently routed to: `claude-opus-4`
- Use case: compliance document screening

**Observations:**
- Workload complexity score: 44/100 — below Opus-4 selection threshold (80)
- Output token ratio low (0.26) — short, structured responses, not reasoning-intensive
- No latency SLA requirement — latency-tolerant workload

**Routing recommendation:** `claude-sonnet-4`

**Projected result:**
- Inference cost reduction: ~83%
- Monthly saving: $7,350
- Quality impact: negligible for structured classification tasks

---

## Prompt Efficiency Observations

Running the platform against simulated enterprise workloads surfaced patterns I'd expect to find in real deployments:

- **Duplicate retrieval chunks** — retrieval pipelines injecting the same document segments across multiple calls. Common when chunking is done without deduplication. Estimated waste: 18% of token budget on affected workloads.
- **Static system prompt repetition** — teams sending 3,000+ token system prompts on every request with 0% cache utilisation. Prompt caching at 90% discount would recover significant spend.
- **Context over-retention** — conversation history replayed in full rather than summarised. Average context: 28K tokens on workloads where the last 3 turns carry >80% of reasoning load.
- **Premium model over-specification** — high-cost models returning <200 output tokens on classification tasks. The reasoning capacity is not being used; the cost is.

---

## Governance Features

- **Team-level chargeback attribution** — spend broken down by cost centre with per-call unit economics
- **API key audit** — flags stale keys (0 calls in 30+ days) that represent unnecessary credential exposure
- **Model proliferation tracking** — monitors number of distinct models in active use; high proliferation increases vendor management overhead
- **Anomaly detection** — flags week-over-week spend spikes, unplanned batch jobs, and budget threshold breaches
- **Incident simulation** — models provider outage, latency degradation, budget breach, and context overflow scenarios with failover routing and estimated business impact

---

## Assumptions and Limitations

This is an operational prototype, not a production system. Important caveats:

- **Simulated workloads** — usage data is synthetically generated to represent realistic enterprise patterns; no real API keys or production data used
- **Pricing snapshots** — model pricing accurate as of May 2026; rates change frequently and the platform does not auto-update
- **Routing heuristics are generalised** — the weighting model is a reasonable approximation; real deployments would calibrate weights per organisation and workload type
- **No authentication layer** — intended as a prototype/demo; production deployment would require auth, role-based access, and audit logging
- **Frontend-only** — attribution and routing logic runs client-side on simulated data; a production version would ingest real API logs via a backend pipeline

---

## What I'd Build Next

- Real API log ingestion (webhook or CSV pipeline from actual provider dashboards)
- Per-organisation routing weight calibration
- Prompt caching simulation with provider-specific cache hit rate modelling
- Committed-use optimisation recommendations (when to renegotiate vs switch provider)
- Slack/email alerting on governance threshold breaches

---

## Stack

HTML, CSS, JavaScript, Chart.js, Anthropic API (claude-sonnet-4 for governance report generation)

Deployed on GitHub Pages.

---

*Built by Bhavani Susmitha Iragaraju — [LinkedIn](https://linkedin.com/in/bhavanisusmitha) · [Portfolio](https://sush2328.github.io/portfolio)*
