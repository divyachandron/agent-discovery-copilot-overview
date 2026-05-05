# Agent Discovery Copilot

**Agent Discovery Copilot** is a multi-agent pre-kickoff research and hypothesis tool for AI implementation teams, including Solutions PMs, implementation leads, forward-deployed engineers, and customer-facing product teams.

You give it a **company URL** and optional context, and it researches the company, identifies the key workflows, recommends the strongest automation starting point, defines success metrics, and generates a **kickoff-ready hypothesis deck**. An evaluator scores every output before it reaches the PM.

---
## Demo

🎥 **Watch the demo:** [Agent Discovery Copilot Demo](https://youtu.be/17EqHSJaISk)

In this demo, I show how the system takes a company URL and sales context, researches the company, maps operational workflows, identifies the strongest automation wedge, generates success metrics, creates a kickoff-ready hypothesis deck, and evaluates the output before it reaches the PM.

---

## Why this exists

Enterprise AI discovery often starts with fragmented notes, generic research, and too much hand-waving.

A strong Agent PM should be able to answer:

- What does this company likely do operationally?
- Which workflows matter most?
- Where is the friction?
- What is the best wedge to automate first?
- How will we know if it is working?
- What assumptions need to be validated in the room?

This project turns that preparation process into an explicit, structured **multi-agent workflow**.

---

## Who this is for

- Agent PMs
- Solutions PMs
- Implementation leads
- Forward deployed engineers
- Customer-facing product and deployment teams

Used before a customer discovery workshop or implementation kickoff.

---

## What it does

### Input
- Company URL
- What was sold
- Optional notes / context

### Output
A workshop-ready brief and hypothesis deck with:

1. **Working Hypothesis** — one-sentence wedge hypothesis with scope and signal basis
2. **Ranked Workflow Hypotheses** — Confirmed workflows ranked by composite score with confidence labels
3. **Top Automation Wedge** — recommended wedge, all six dimension scores with evidence sources, why alternatives ranked lower, explicit evidence gaps
4. **Strategic Fit** — why this wedge is relevant, timely, and feasible for this company now
5. **Success Metrics** — primary KPI, supporting indicators, guardrails, and confidence labels on every number
6. **Validation Questions and Next Steps** — what to confirm in the workshop, what would invalidate the wedge, and the fallback path

---

## Multi-agent workflow

### Agent 1 — Planner
Fetches the company homepage via Jina, extracts 6–8 investigation targets using a dedicated lightweight LLM call, and produces the research plan that every downstream agent follows — scope, depth, focus areas, quality criteria, and workflow categories to investigate.

**Outputs**
- Scope and company type
- Inferred industry
- Agent configurations and quality criteria per downstream agent
- workflow_investigation_targets — the bounded list of service categories the Workflow Mapping Agent must investigate

---

### Agent 2 — Company Research Agent
Runs web searches and Jina fetches (homepage, careers, jobs) to build a structured company profile. Identifies competitors and their automation initiatives with sourced URLs. Assesses implementation readiness from engineering capacity, tool stack, ops headcount, and expansion signals.

**Outputs**
- Industry, size, stage, business model
- Hiring signals, news signals, operational pain points
- competitive_gap — up to 3 competitors with sourced automation initiative URLs
- implementation_readiness — complexity, engineering dependency risk, expansion potential, evidence, confidence
- displacement_risk — whether an incumbent solution already covers what was sold

---

### Agent 3 — Workflow Mapping Agent
Generates 6–8 candidate workflows from company signals and investigates each against the planner's target list. Assigns Confirmed / Likely / Excluded status verified deterministically in code — not by LLM judgment. Anchors canonical labels to investigation targets via keyword matching.

**Outputs**
- Workflow list with status (Confirmed / Likely / Excluded), team, frequency, manual steps, data inputs, and supporting signals
- confirmed_workflows — the subset that proceeds to wedge scoring
- Canonical labels anchored to planner investigation targets

---

### Agent 4 — Wedge Identification Agent
Scores every Confirmed workflow across six dimensions. Five of six dimensions are computed deterministically in code from structured workflow and context fields. Applies fixed weights and deterministic tie-breaking. Selects the top wedge in code as the highest composite-scoring Confirmed workflow.

**Scoring dimensions**
- Automation Feasibility (20%) — from workflow frequency and manual step count
- Business Impact (20%) — rubric-based, reasoned from workflow characteristics
- Data Readiness (15%) — from named digital systems in data inputs
- Manual Effort Concentration (15%) — from hiring signals referencing manual process roles
- Competitive Pressure (15%) — from sourced competitive_gap entry count
- Implementation Fit (15%) — from implementation_readiness complexity and engineering dependency risk

**Outputs**
- scored_workflows with all six dimension scores and composite
- top_wedge — deterministically selected highest-scoring Confirmed workflow

---

### Agent 5 — Success Metrics Agent
Generates 5 KPIs tied deterministically to the selected wedge's canonical label. Anchors metrics to real business outcomes with industry benchmarks. Flags all baselines as estimated or confirmed.

**Outputs**
- Primary metric with baseline, target, and mechanism
- Supporting indicators with business outcome linkage
- Guardrails — what must not degrade during rollout
- kpi_template_used field referencing the wedge canonical label

---

### Agent 6 — Brief Builder
Compiles all upstream agent outputs into a structured company brief. Synthesises the hypothesis, supporting evidence, and risk flags the PM must probe before committing to the wedge.

**Outputs**
- Hypothesis statement
- Supporting evidence with inline citations
- risk_flags — scope concerns, budget unknowns, change management risks

---

### Agent 7 — Deck Generator
Transforms all structured agent outputs into a 6-slide hypothesis deck formatted for discovery workshop delivery. Each slide includes speaker notes for the PM.

**Deck structure**
- Slide 1: Working Hypothesis
- Slide 2: Ranked Workflow Hypotheses
- Slide 3: Top Automation Wedge
- Slide 4: Strategic Fit
- Slide 5: Success Metrics
- Slide 6: Validation Questions and Next Steps

---

### Agent 8 — Evaluator
Scores all outputs across six weighted dimensions and applies deterministic Research Grounding caps in code after LLM scoring. Routes failures to one of three actions — route back, PM question, or known-limitation flag — and runs alongside a separate trace evaluator that checks process integrity with 14 deterministic assertions.

**Outputs**
- composite score (≥3.5 to pass) and dimension scores
- action and targeted feedback for the responsible agent
- process_verified and trace_score from the deterministic trace evaluator

---

## Why this is interesting

This is **not** a generic chatbot and **not** just a slide generator.

It is a productized multi-agent workflow that demonstrates:

- **Deterministic workflow gating** — Confirmed status is verified in code via keyword matching against context signals, not assigned by LLM judgment
- **Canonical label anchoring** — workflow labels are snapped to a bounded investigation target list so the same company always produces the same labels regardless of model variance
- **Confirmed-only wedge scoring** — Likely and Excluded workflows never enter scoring or appear in the ranked deck output
- **Five-of-six dimensions computed in code** — automation feasibility, data readiness, manual effort concentration, competitive pressure, and implementation fit are all deterministic; only business impact is LLM-scored
- **Dual evaluation layer** — an output evaluator (LLM, six weighted dimensions) runs alongside a trace evaluator (deterministic code, 14 assertions) so a plausible-looking output with skipped research steps is explicitly surfaced
- **Scope ambiguity gate** — a deterministic check blocks deck generation and routes to a PM question when displacement risk signals are present, regardless of output quality scores

---

## Example use case

**Input**
- URL: `https://www.higginsbotham.com`
- What was sold: `AI automation for insurance broker operations`

**System behavior**
1. Planner fetches the homepage and extracts investigation targets — commercial lines, employee benefits, surety bonds, risk consulting
2. Company Research Agent searches for hiring signals, news, competitive gap, and implementation readiness
3. Workflow Mapping Agent investigates each target category, confirms workflows with matching context signals, and anchors canonical labels
4. Wedge Identification Agent scores all Confirmed workflows across six dimensions and deterministically selects the top wedge
5. Success Metrics Agent generates 5 KPIs tied to the selected wedge canonical label
6. Brief Builder compiles the hypothesis, evidence, and risk flags
7. Deck Generator produces a 6-slide workshop-ready deck
8. Evaluator scores all outputs, applies deterministic grounding caps, and surfaces any failures with targeted action

---

## Architecture

**Data flow**

`Input → Planner → Company Research Agent → Workflow Mapping Agent → Wedge Identification Agent → [Metrics + Brief in parallel] → Scope check → Deck Generator → Evaluator`

**Repo structure**

```
agent-discovery-copilot/
├── agents/               # all 8 agent modules + orchestrator
│   └── evals/            # trace evaluator
├── app/                  # frontend — UI, styles, client JS
├── prompts/              # system prompts for each agent
├── schemas/              # structured output schemas
├── docs/                 # PRD
├── examples/             # sample inputs and outputs
├── assets/               # screenshots and demo visuals
├── server.js             # Express server + SSE pipeline
└── README.md
```

---

## Tech stack

- **Runtime:** Node.js ≥18, ES modules
- **LLM:** Anthropic API — `claude-sonnet-4-5` for all primary agents, `claude-haiku-4-5-20251001` for planner target extraction and web search fallback
- **Server:** Express with Server-Sent Events for live pipeline streaming
- **Web content:** Jina Reader API for homepage and careers page fetches
- **Search:** Anthropic `web_search_20250305` server-side tool (no external search API required)

**Setup**

```bash
npm install
ANTHROPIC_API_KEY=your_key npm start
```

Open `http://localhost:3000`, enter a company URL and what was sold, and all 8 agents will stream results live.# agent-discovery-copilot-overview
