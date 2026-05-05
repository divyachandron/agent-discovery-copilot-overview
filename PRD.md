# Agent Discovery Copilot — PRD

### TL;DR

Agent Discovery Copilot is a multi-agent pre-kickoff research and hypothesis tool designed for AI implementation teams—namely Agent PMs, Solutions PMs, and implementation leads. It addresses the pervasive issue of shallow discovery prep by automatically researching a target company, mapping workflows, identifying the highest-leverage automation wedge, defining tailored success metrics, and generating a workshop-ready hypothesis deck. An evaluator agent scores each output before it reaches the PM, ensuring rigor and quality.

---

## Goals

### Business Goals

* **Reduce average pre-kickoff prep time by 60%.**
* **Increase hypothesis validation rate post-kickoff:** Target is >70% of recommended wedges confirmed as correct priority.
* **Accelerate time-to-value for AI implementation engagements.**
* **Establish a proprietary, defensible research and scoring workflow** that differentiates the platform from competitors.

### User Goals

* **Walk into every discovery workshop with a structured hypothesis** instead of blank slides.
* **Identify the highest-leverage automation wedge** before the first customer conversation.
* **Receive pre-defined success metrics** aligned to real business outcomes.
* **Reduce cognitive load** of cross-referencing disparate research sources.

### Non-Goals

* **Does not replace the discovery workshop itself:** The tool is for preparation, not replacement.
* **Does not integrate with CRM or opportunity management** in this phase.
* **Does not generate implementation plans or technical architecture.**

---

## User Stories

**Agent PM**

* As an Agent PM, I want to generate a company research brief so that I understand my target before the kickoff call.
* As an Agent PM, I want to see ranked workflow hypotheses so that I can pressure-test and challenge them before the workshop.
* As an Agent PM, I want a ready-to-use hypothesis deck so that I have specific, high-quality slides for the workshop.

**Solutions PM**

* As a Solutions PM, I want to view measurable success metrics for the automation wedge so that I can anchor the business case.
* As a Solutions PM, I want to flag which hypothesis scored lowest in evaluator review so that I can probe potential weak spots in the hypothesis.

**Implementation Lead**

* As an Implementation Lead, I want to know which workflow was identified as the top automation wedge so that I can focus my technical prep.
* As an Implementation Lead, I want to see the evidence trail behind each hypothesis so that I can defend or challenge assumptions in technical discussions.

---

## Functional Requirements

* **Planner Agent** (Priority: P0)

  * Is Agent 0 — runs first, before any research begins. Receives the company name and optional context from the PM, then dynamically constructs the research plan: which agents to invoke, in what order, with what parameters, and what tool calls each agent should make. Determines scope based on available context and company type. Does not do research itself — produces a structured execution plan that downstream agents follow. Also defines what a "good enough" output looks like for each agent before handing off to the Evaluator.

  * **Homepage fetch and fallback:** Before constructing the plan, the planner fetches the company homepage via Jina. If Jina returns nothing useful (<100 chars), a web search fallback runs using a lightweight model to retrieve company service and product information before planning begins.

  * **workflow_investigation_targets — LLM extraction, not planner judgment:** The planner does not generate `workflow_investigation_targets` through its main planning LLM call. Instead, a dedicated extraction call (`extractTargetsLLM`) using a lightweight model at temperature 0 reads the homepage content and returns a JSON array of 6–8 operational workflow category names, using `what_was_sold` as a lens (e.g. `["claims intake processing", "policy underwriting and approval", "employee benefits administration"]`). This extracted list overrides whatever the planning LLM may have produced for that field. If fewer than 3 targets are extracted, `workflow_investigation_targets` is omitted and the Workflow Mapping Agent relies on the quality criteria instead. This mechanism ensures investigation targets are grounded in the operational workflows that deliver what the company offers, not in customer-facing product names or deal context alone.

* **Company Research Agent** (Priority: P0)

  * Web search and public signal research as core. Job posting aggregators, news parsing, and social hiring signals as optional enrichment where publicly accessible.
  * Outputs a structured company brief summarizing signals and supporting evidence.
  * Competitor Research Step: Identifies the top 3 competitors in the same space. For each, searches for announced or observable automation initiatives, AI tool adoption, and operational capability signals. Identifies where the target company is behind. Output includes a required competitive_gap field: { competitor: string, automation_initiative: string, target_company_gap: string, source: string }. The source field must be a real HTTP URL — entries without a verifiable source URL must be omitted rather than included without citation. Competitor automation advantage is flagged as an external urgency signal for the wedge recommendation.
  * Implementation Readiness Assessment: The agent evaluates four public signals 
to determine how complex and expansive this implementation is likely to be. 
This is not a sales qualification signal — the deal is already signed. 
This is an implementation intelligence signal.

(1) Internal engineering capacity: total engineering headcount relative to 
company size. High ratio signals the customer will want to own parts of the 
implementation, reduce vendor scope, and require deeper technical integration. 
Low ratio signals higher dependency on the vendor and faster time-to-value.

(2) Tool consolidation appetite: job postings and company profiles scanned for 
named existing tools. Tools in the same category as what was sold are 
integration or replacement requirements. Tools in adjacent categories are 
expansion opportunities.

(3) Change management capacity: ops and program management headcount signals. 
Low ops capacity = higher risk of implementation stalling post-go-live due to 
lack of internal adoption driver. High ops capacity = faster adoption and 
expansion potential.

(4) Expansion signals: hiring velocity, market expansion announcements, 
new product launches. High growth signals more automation appetite, 
faster ROI realization, and higher expansion probability.

Output includes a required implementation_readiness field:
{ 
  complexity: low | medium | high, 
  engineering_dependency_risk: low | medium | high,
  expansion_potential: low | medium | high,
  evidence: [string], 
  confidence: low | medium | high 
}

* **Workflow Mapping Agent** (Priority: P0)

  * Infers likely operational workflows based on company profile, industry, and team structure.
  * Outputs a ranked workflow map with supporting rationale.
  * The agent must consider 6-8 candidate workflows before ranking. Only Confirmed workflows appear in the visible ranked output — the number of Confirmed workflows will vary based on available evidence. The full candidate list including Likely workflows is available in the brief for discovery probing.
  * Workflow evidence gating and statusing: The Workflow Mapping Agent may 
generate candidate workflows using company evidence and generalized workflow 
reasoning, but each candidate must be assigned one of three statuses:

- Confirmed — the workflow's canonical_label and name together contain ≥2 
  keywords that appear within a single context signal text (hiring signal, 
  operational pain point, or news signal). This is verified deterministically 
  in code — it is not an LLM judgment call.
- Likely — the workflow is structurally plausible but no single context signal 
  contains ≥2 matching keywords. Includes workflows where the LLM assessed 
  confidence but the code check did not find a keyword match.
- Excluded — no meaningful company-specific support exists; the workflow was 
  investigated but ruled out.

Only Confirmed workflows may appear in the visible ranked workflow output 
or proceed into wedge scoring. Likely workflows may be retained for notes, 
appendix, or discovery probing, but must not influence the primary 
recommendation.

The keyword match check uses these rules: keywords are extracted from 
canonical_label, workflow name, and team name by splitting on whitespace and 
punctuation, lowercasing, and removing stop words and tokens shorter than 3 
characters. A signal text must contain at least 2 distinct keywords from this 
combined set for the workflow to qualify as Confirmed.

Workflow normalization — anchored to investigation targets: Canonical labels 
are not freely generated by the model. They are anchored to the planner's 
workflow_investigation_targets list. Each workflow is matched to its closest 
investigation target by keyword overlap (≥2 keyword matches required, using 
keywords from canonical_label, name, and team). The matched target is then 
normalised into snake_case to produce the canonical_label. If no target 
matches with ≥2 keywords, the workflow receives no anchored label.

This ensures canonical labels are stable across runs — same target always 
produces the same label — and prevents free-form model-generated labels from 
drifting between runs. The agent may generate company-specific display names 
for presentation, but scoring and ranking operate on the anchored canonical label.

* **Wedge Identification Agent** (Priority: P0)

  * Scores each workflow across six dimensions. The first four are internal workflow quality signals: automation feasibility, business impact, data readiness, and manual effort concentration. The final two are external strategic signals: (1) Competitive Pressure — if competitors are ahead on automating this workflow, urgency to buy increases. Scored 1–5, sourced from competitive_gap. (2) Implementation Fit — based on implementation_readiness from the Company Research Agent. Scores how well this specific workflow matches the customer's implementation capacity and expansion appetite. A workflow requiring deep engineering integration scores lower if the customer has low engineering capacity. A workflow with high expansion potential scores higher if the customer is in high growth mode. Scored 1–5.
  * Dimension weights: Automation Feasibility 20%, Business Impact 20%, Data Readiness 15%, Manual Effort Concentration 15%, Competitive Pressure 15%, Implementation Fit 15%.
  * Minimum evidence threshold for Competitive Pressure and Implementation Fit: if fewer than two independent sources support a dimension score, the Evaluator treats that dimension as low-confidence and surfaces a known-limitation flag rather than accepting the score at face value. Zero sources triggers an automatic known-limitation flag regardless of the model's stated confidence.
  * Selects the top automation wedge based on composite score and provides rationale. A workflow that is internally painful but has low implementation fit and no competitive pressure is ranked lower than one with moderate pain but high external urgency and strong implementation alignment. The wedge score reflects not just whether the workflow is broken, but whether this company can successfully implement a solution for it.
  * Confirmed-only scoring: Only workflows marked Confirmed by the Workflow 
Mapping Agent may be passed into wedge scoring. Likely workflows may be 
retained for notes or discovery probing, but must not be eligible for top 
wedge selection.
  * Normalized ranking: Final wedge scoring must operate on normalized workflow 
labels and structured workflow signals rather than unconstrained free-form 
workflow names. This ensures semantically equivalent workflows are scored 
consistently across repeated runs.
  * Deterministic recommendation requirement: Wherever possible, final workflow 
ranking and wedge selection must be produced through deterministic or tightly 
constrained scoring logic applied to structured signals. Repeated runs with 
the same workflow set and same structured inputs should produce the same 
ranked order and top wedge.

* **Success Metrics Agent** (Priority: P0)

  * Generates 3–5 KPIs that would measurably improve if the automation wedge is solved.
  * Anchors KPIs to specific business outcomes and industry norms.
  * Deterministic KPI mapping: KPI generation must be tied deterministically 
  to the selected wedge and the supporting company context. Repeated runs 
  that produce the same selected wedge should produce the same primary metric, 
  supporting indicators, and guardrails unless the PM explicitly refreshes 
  evidence or changes deal context.
  * Metric template behavior: Each wedge category should map to a stable KPI 
  template that is then filled using company-specific context. The LLM may 
  adapt labels and wording, but it must not invent materially different KPI 
  sets for the same wedge across repeated runs.

* **Hypothesis Deck Generator** (Priority: P0)

  * Compiles all agent outputs into a presentation-ready, PM-editable hypothesis deck.
  * The generated deck follows a defensible reasoning flow that helps the PM explain not only the recommendation but also the logic behind it. The deck is a workshop tool for hypothesis validation, not a final answer or sales qualification artifact.

**Slide order:**

1. **Working Hypothesis** — one-sentence wedge hypothesis, company name, 
proposed automation scope, and a note that this is based on public signals 
and intended for validation in the workshop

2. **Ranked Workflow Hypotheses** — up to 5 Confirmed workflows ranked by 
composite score, with one-line rationale, confidence label, and compact 
visibility into the strongest scoring dimensions. The full candidate list 
including Likely workflows remains available in the brief.

3. **Top Automation Wedge** — recommended wedge, all six dimension scores — automation feasibility, business impact, data readiness, manual effort concentration, competitive pressure, implementation fit — each with evidence source labeled, why this wedge is the recommended starting point, why alternatives ranked lower, explicit evidence gaps

4. **Strategic Fit** — why this wedge is relevant, timely, and feasible for 
this specific company at this time. Derived from company signals, workflow 
characteristics, and implementation readiness. Positioned after the wedge 
recommendation so it supports and contextualizes a recommendation already 
presented.

5. **Success Metrics** — 1 primary business outcome, 2–4 supporting 
indicators, guardrails where applicable, confidence label on every number, 
and an explicit note that baselines require customer validation

6. **Validation Questions and Next Steps** — 3–5 discovery questions derived 
from low-confidence signals, explicit assumptions to validate, what would 
confirm or invalidate the wedge, the immediate next step if confirmed, and 
the fallback path if invalidated

**Strategic Fit is not:**
- a sales qualification section
- a generic company context dump
- a standalone judgment about whether there is a deal

**Strategic Fit is:**
- the reasoning layer that explains why the recommended wedge makes sense 
for this company now
- derived from the company brief, workflow ranking, and implementation 
readiness signals
- positioned after the wedge so it defends a recommendation the audience 
has already seen

The deck should help the PM walk into kickoff with a defensible hypothesis 
about where automation may create the most value, why that wedge is a 
sensible place to start, and what must be validated with the customer before 
implementation planning begins.
  * Supports export to PDF and slide-editable formats.

* **Evaluator Agent** (Priority: P0)

  * Scores every section produced by other agents before surfacing to the PM.
  * Uses defined evaluation dimensions; applies explicit decision logic when sections fail.
  * Displays reasoning for any failed output and allows per-section regeneration.
  * **Research Grounding caps are deterministic code overrides, not LLM rubric guidance.** After the evaluator LLM produces its dimension scores, deterministic checks run in code and may override Research Grounding downward. The composite score and pass/fail are recomputed after each cap:
    1. Zero real HTTP URLs across all context source fields → Research Grounding hard-capped at 2.0.
    2. competitive_gap has zero entries with real HTTP source URLs → known_limitation_flag set, Research Grounding hard-capped at 2.0.
    3. competitive_gap has fewer than 2 sourced entries (but at least 1) → low_confidence_flag set, Research Grounding capped at 3.0.
    The stricter cap always applies when multiple conditions fire. These overrides cannot be bypassed by model output.

* **Outcome Evals Module** (Priority: P2)

  * Post-kickoff feedback: PM marks each recommendation as Validated / Partially Validated / Invalidated.
  * Data is aggregated for scoring calibration and ongoing product improvement.

---

## Reliability and Recommendation Stability

Because Agent Discovery Copilot is used to prepare for live customer 
workshops, recommendation stability is a core trust requirement. A PM 
running the same company and deal context multiple times using the same 
evidence snapshot should receive the same workflow ranking and top wedge 
recommendation unless underlying evidence has materially changed.

#### Sources of instability to control

1. Web research variance — search and retrieval results may differ across runs
2. Workflow ranking variance — model-generated ranking logic may drift across runs
3. Wedge scoring variance — small differences in dimension scoring can 
change the top recommendation

#### Design requirements

- Final workflow ranking and wedge recommendation must be based on 
deterministic or tightly constrained scoring logic applied to structured 
signals, not free-form model judgment alone
- The same company URL and deal context should reuse a cached evidence 
snapshot by default rather than re-running public web retrieval on every 
execution. Evidence snapshot reuse is the target steady-state behavior. 
In Phase 1, recommendation stability is improved through workflow gating, 
normalization, deterministic scoring, and KPI mapping. Explicit evidence 
snapshot caching is introduced in Phase 2.
- The system must support explicit refresh behavior when the PM wants 
updated public signals
- Recommendation outputs must expose the evidence snapshot timestamp used 
for scoring
- If refreshed evidence materially changes the recommendation, the system 
must surface a recommendation-change warning and identify which signals 
changed

#### Refresh semantics

The UI must make clear on every run:
- Evidence snapshot timestamp — when the underlying research was collected
- Whether the run used cached or refreshed evidence
- If the recommendation changed from the prior run, which signals changed 
and why the recommendation shifted

This matters for demos and for trust. A PM must always know whether they 
are looking at current signals or a cached snapshot.

#### Deterministic scoring requirement

The Wedge Identification stage must calculate dimension scores from 
explicit structured signals extracted upstream. Scoring logic should be 
implemented in code with fixed weights wherever possible. LLMs may assist 
in signal extraction and workflow inference, but final wedge selection 
must not depend on unconstrained model judgment.

Where a dimension cannot yet be fully rule-based, the system must use a 
constrained rubric prompt and cache the resulting score with the evidence 
snapshot so repeat runs remain stable.

#### Initial dimension scoring rules

- Manual effort concentration: derived from the count of distinct 
workflow-relevant hiring or operating signals referencing manual process 
roles. 0 signals = 1, 1 signal = 3, 2 or more distinct signals = 5

- Competitive pressure: derived from the count of distinct 
workflow-relevant competitive_gap entries with sources. 0 entries = 1, 
1 entry = 3, 2 or more entries = 5

- Implementation fit: derived from implementation_readiness. Complexity 
high = 2, medium = 3, low = 5. Engineering dependency risk high = -1, 
medium = 0, low = +1. Final score is clamped to the 1–5 range

- Evidence richness: derived from the count of workflow-relevant sourced 
signals with real HTTP URLs that directly support the specific workflow 
being scored. Company-wide sources that do not materially inform that 
workflow do not count. A source is considered workflow-relevant only if 
it supports the workflow's existence, manual burden, operational 
importance, or implementation conditions. 0–2 = 2, 3–5 = 3, 6 or more = 5

- Automation feasibility: deterministic. Derived from two workflow fields:
  - frequency: daily = 3, weekly = 2, monthly or ad-hoc = 1
  - manual_steps count: ≥6 steps = 3, 3–5 steps = 2, <3 steps = 1
  - Combined score (2–6) mapped to 1–5: ≤2 → 1, 3 → 2, 4 → 3, 5 → 4, 6 → 5

- Business impact: rubric-based. Explicitly reasoned from workflow 
characteristics stated in the context output. Reasoning must be visible in 
the output and cached with the evidence snapshot.

- Data readiness: deterministic. Derived from the count of named digital 
systems in data_inputs. A data input is "named" if it references a specific 
platform, acronym (e.g. CRM, AMS, HRIS), system keyword (e.g. "platform", 
"database", "portal"), or proper noun mid-phrase. 0 named systems → 2, 
1–2 named systems → 3, 3 or more → 5.

#### Deterministic tie-breaking

If two workflows produce identical composite scores, ties are broken using 
fixed dimension precedence in this order:
1. Competitive pressure score — higher wins
2. Evidence richness — more workflow-relevant sourced signals wins
3. Implementation fit score — higher wins
4. Alphabetical workflow name — last-resort deterministic tie-breaker

This ensures the same input produces the same ranked output regardless 
of minor model variance.

#### Deterministic top wedge selection

The top wedge is not chosen by the LLM. After all dimension scores are 
computed and the deterministic sort is applied, the top wedge is set in code 
to the highest composite-scoring workflow whose status is Confirmed. The 
entire workflow object is copied — no individual fields are patched from LLM 
output. This ensures the top wedge recommendation is always the unambiguous 
leader of the deterministic ranking, and cannot be overridden by model 
judgment or presentation-layer framing.

#### Recommendation stability target

For repeated runs on the same company URL and deal context using the same 
evidence snapshot, the workflow ranking and top wedge recommendation must 
be identical.

Recommendation stability applies not only to the top wedge recommendation, 
but also to the visible Confirmed workflow set, ranked workflow order, and 
KPI set generated from the selected wedge.

---

## User Experience

**Entry Point & First-Time User Experience**

* User (PM) accesses Agent Discovery Copilot via web dashboard.
* They enter a target company name and optionally: industry, deal size, or known use case.
* Light onboarding walks first-time users through the process and explains agent roles.

**Core Experience**

1. **Step 1:** Company brief is generated and displayed.
  * Minimal friction: auto-populated fields where possible.
  * Validation: system checks for sufficient public data; surfaces a data confidence indicator.
2. **Step 2:** Workflow map appears, showing ranked hypotheses.
  * UI clarity: each workflow labeled, rationale and supporting sources linked inline.
3. **Step 3:** Automation wedge is highlighted, with rationale and confidence score.
  * Interactive: PM can reveal details or evidence behind top recommendation.
4. **Step 4:** Success metrics panel populates with 3–5 KPI suggestions.
  * Each metric clearly tied to business impact; generic metrics flagged if confidence is lower.
5. **Step 5:** Evaluator agent runs in background, surfaces a pass/fail badge and dimension scores for each section.
  * Scores and reasoning are immediately visible to the PM.
6. **Step 6:** PM reviews, edits, and exports the hypothesis deck (PDF or editable slide format).
  * Edits per section; regenerate option if section failed evaluation.
  * Deck export supports both sharing and additional customization.

**Advanced Features & Edge Cases**

* If any section fails evaluation: PM sees failure reason per dimension, can regenerate single section or add an override note.
* If public data is insufficient: system flags low data confidence and adjusts hypothesis specificity, warns PM of lower certainty.
* Power-user: Ability to regenerate only the weakest-scoring dimension for rapid iteration.

**UI/UX Highlights**

* Evaluation scores and pass/fail badge are always visible inline, not buried in menus.
* Regeneration of sections is granular—reduces costly re-runs.
* Export options (PDF, slides) match PM workflow needs.
* Accessibility: strong color contrast, readable fonts, responsive for both desktop and tablet.
* Evidence links provided for each research claim for transparency.

---

## Evals

**Evaluator Agent Architecture**

* Runs after every agent output; blocks results from PM if they do not meet quality threshold.
* Designed for rapid, consistent, and explainable assessment.

**Evaluation Dimensions**

1. **Research Grounding:** Are claims traceable to a real, citable source? Before scoring, the Evaluator verifies that real tool use occurred — checking the tool call log and substitution log for evidence that the agent actively retrieved data rather than generating claims from parametric memory alone. Outputs with no tool call record fail Research Grounding automatically.
2. **Workflow Plausibility:** Does the hypothesized workflow match the company profile and industry norms?
3. **Wedge Specificity:** Is the recommended automation wedge actionable and specifically relevant?
4. **Metric Relevance:** Are proposed success metrics tied to real business outcomes, not vanity metrics?
5. **Deck Coherence:** Does the consolidated hypothesis narrative hang together logically and without contradiction?
6. **Recommendation Stability:** Does the same company and deal context produce 
a consistent Confirmed workflow set, ranked workflow order, top wedge, and 
KPI set when evidence has not materially changed?

**Scoring Methodology**

* Each dimension is scored 1–5 by the evaluator LLM using a structured rubric prompt.
* Composite score is a weighted average:
  * Research Grounding: 25%
  * Workflow Plausibility: 25%
  * Wedge Specificity: 20%
  * Metric Relevance: 15%
  * Deck Coherence: 5%
  * Recommendation Stability: 10%

**Pass/Fail Threshold**

* Composite score must be ≥ 3.5 to pass.
* Any single dimension scoring below 2.0 triggers a hard fail, regardless of total score.

**Fail Behavior**

When a section fails, the Evaluator Agent applies explicit decision logic to determine which of three actions to take — it does not choose arbitrarily.

* Route back to the originating agent if the issue is fixable with more or better research (e.g., an unsourced claim, a generic wedge, a plausibility gap that additional signal could resolve). The Evaluator sends structured feedback naming the specific dimension, the specific gap, and what a passing output would look like.
* Ask the PM one clarifying question if the issue requires information only the PM has — context not publicly available and cannot be inferred (e.g., the customer's known priority, a deal constraint, an industry nuance). The Evaluator surfaces exactly one question per fail cycle.
* Surface as a known-limitation flag if the issue is a structural data gap — insufficient public signal, no established industry workflow norms, or genuinely ambiguous wedge space. The Evaluator annotates the output with a confidence caveat and surfaces it for PM review with the limitation clearly labeled.

The PM sees the Evaluator's action type and reasoning inline for every failure. Manual override with a note remains available as a last resort.

**Per-Agent Retry Instructions**

Each agent prompt contains a dedicated retry section with specific, dimension-targeted guidance for handling Evaluator feedback. Each agent's retry logic is scoped to what that agent actually produces.

* **Company Research Agent:** If Research Grounding fails, re-run the primary search query with a narrower scope, verify that every claim maps to a retrievable URL, and remove any claim that cannot be sourced. Do not infer or generalize from adjacent information.
* **Workflow Mapping Agent:** If Workflow Plausibility fails, re-examine the company profile for signals that contradict the current workflow ranking. Re-weight or remove workflows with no supporting evidence in the company brief. Do not retain a workflow hypothesis based on industry assumption alone.
* **Wedge Identification Agent:** If Wedge Specificity fails, return to the workflow map and select a wedge that names a specific process, team, and failure mode — not a category. Generic wedges (e.g., "automate data entry") are invalid outputs. If Competitive Pressure or Implementation Fit scores are flagged as unsupported, return to the company brief and verify that competitive_gap and Implementation Fit fields contain specific, sourced evidence. Do not assign a score to either dimension without a traceable signal. If neither field contains sufficient data, surface a known-limitation flag on those dimensions rather than estimating.
* **Success Metrics Agent:** If Metric Relevance fails, discard any metric that cannot be tied to a named business outcome for this specific company. Replace with metrics directly anchored to the identified wedge and the company's observable operational context.
* **Hypothesis Deck Generator:** If Deck Coherence fails, re-read all upstream outputs in sequence and identify the first point of logical contradiction. Resolve that contradiction before regenerating the deck narrative.

**Future Phase — Outcome Evals**

* Post-kickoff, PMs mark each recommendation as: Validated / Partially Validated / Invalidated.
* System aggregates validation rates by dimension and by score band.
* Over time, real-world data is used to recalibrate dimension weights and rubric prompts, continuously improving agent reliability.

---

## Agentic Behaviors

**Dynamic Research Planning:** The Planner Agent does not follow a fixed script — it constructs a tailored research plan per company based on available signals, deal context, and company type. For a well-documented public company, it may deprioritize broad web scraping and go deep on earnings signals; for a private SMB with thin public presence, it shifts weight toward job posting analysis and LinkedIn org structure.

**Adaptive Branching:** Agents downstream of the Planner can trigger conditional paths based on what they find. If the Company Research Agent surfaces a strong signal in a specific operational area (e.g., repeated hiring for a manual process role), the Workflow Mapping Agent receives that signal and up-weights workflows in that category. The workflow is staged but adaptively branches based on evidence quality and evaluator feedback.

**Tool Choice:** The Planner defines the allowed tool set and preferred invocation order for each agent. Downstream agents choose autonomously among allowed tools based on what the task requires. If an agent determines it needs a tool outside its allowed set, it escalates to the Planner rather than substituting unilaterally. Agents may substitute tools within their permitted set if their primary tool returns insufficient signal, but must log every substitution with a reason. The Evaluator checks whether any tool deviation was justified when scoring Research Grounding.

**Planner / Executor Separation:** The Planner does not just classify company type — it produces a research directive that meaningfully changes what downstream agents do. For a well-known enterprise with abundant public data, it instructs the Company Research Agent to skip broad web scraping and focus on implementation-specific signals: engineering headcount ratio, tool stack, ops capacity, growth trajectory. For an obscure SMB with thin public presence, it instructs broader discovery first. The Planner's output must include specific search directives that the Company Research Agent follows — not just metadata labels that get ignored.

**Flexible discovery, stable recommendation:** Upstream agents may remain 
adaptive in how they search for signals, infer candidate workflows, and 
collect evidence. Downstream recommendation stages must be more constrained. 
Discovery should be flexible; recommendation should be stable.

This means:
- Workflow candidates may be generated through adaptive reasoning
- Candidate workflows must be normalized into stable canonical labels 
  before ranking
- Only Confirmed workflows may influence the primary recommendation
- Final ranking, wedge selection, and KPI generation should be deterministic 
  or tightly constrained from the same structured inputs

**Evaluator with Action Authority:** The Evaluator Agent is not a passive scorer. When output falls below threshold, it applies explicit decision logic to determine whether to route back to the originating agent, ask the PM one targeted question, or surface a known-limitation flag. It does not simply block and wait — it drives resolution with a specific, reasoned action every time.

---

## Trace-based Evaluation

Output evaluation scores whether each agent produced good results.
Trace-based evaluation scores whether each agent did its job correctly.
These are two different questions. A system can produce plausible output
while skipping the work entirely. Trace evaluation catches that.

The trace evaluator is deterministic code, not an LLM. This is intentional —
deterministic checks cannot be fooled by plausible-sounding output the way
an LLM judge can. The output evaluator handles judgment. The trace evaluator
handles facts.

### What gets traced

Every agent emits a structured trace log alongside its output containing:
- Which tools it called and with what parameters
- How many claims it made vs how many have real citable URLs
- Whether it used the Planner config it was given
- Any tool substitutions made and the reason

### Assertions per agent

| Agent | Assertion | Critical | Pass condition |
|---|---|---|---|
| Planner | Fetched homepage before planning | No | Jina call logged for company_url |
| Context | Used real tools | Yes | At least 1 Jina fetch and 1 web search logged |
| Context | Claims grounded | Yes | More than 80% of claims have real HTTP URLs |
| Context | Competitor research completed | Yes | competitive_gap contains at least 1 entry with source |
| Context | Implementation readiness assessed | Yes | implementation_readiness.confidence is not unknown |
| Workflow | Used context signals | No | At least 1 supporting_signal references context output |
| Workflow | Confirmed-only visible ranking | Yes | Every visibly ranked workflow has at least 1 company-specific supporting signal |
| Workflow | Canonical workflow labels used | Yes | Every ranked workflow maps to a stable normalized workflow label |
| Wedge | Used external signals | Yes | competitive_gap and implementation_readiness present in input |
| Wedge | Only Confirmed workflows scored | Yes | No workflow marked Likely or Excluded is passed into wedge scoring |
| Wedge | Evidence threshold handling correct | Yes | If Competitive Pressure or Implementation Fit has fewer than 2 independent sources, the dimension is flagged as low-confidence / known-limitation rather than accepted without caveat |
| Metrics | KPI template tied to selected wedge | Yes | KPI template is generated from the selected wedge label and logged with the selected wedge id |
| Evaluator | Action field populated | Yes | If eval failed, action is one of route_back, pm_question, or known_limitation_flag — not null or missing |

### Pass threshold

- process_verified is true only when all critical assertions pass
- trace_score below 70% triggers a process integrity warning surfaced alongside the brief
- trace_score below 50% triggers a flag the PM must acknowledge before proceeding

### How it surfaces to the PM

The PM sees two scores side by side in the UI:
- Output quality: did the agents produce good results
- Process integrity: did the agents actually do their jobs

A high output score with a low trace score means the brief looks good
but the work was not done. The PM knows to validate more aggressively
in the workshop and treat the output with lower confidence.

A brief can only be fully trusted when both scores are high. One without
the other is insufficient.

---

## Success Metrics

**User-Centric Metrics**

* **Pre-kickoff prep time reduction (%):** Tracked baseline vs. post-adoption.
* **% of PMs who rate hypothesis deck “ready to use without major edits”:** Target >75%.
* **Adoption and repeat usage rates** among PMs and implementation leads.

**Business Metrics**

* **Wedge validation rate post-kickoff:** Target >70% Validated or Partially Validated in follow-ups.
* **Retention/repeat usage among implementation teams:** Monthly active teams and session frequency.

**Technical Metrics**

* **Evaluator agent latency:** <30s per output section, measured via backend logs.
* **Pipeline completion time (full deck):** <20 min from start to export.
* **First-run eval pass rate:** Target >80% across all sessions.

**Tracking Plan**

* Company brief generated (event)
* Workflow map viewed (event)
* Wedge selected and rationale surfaced (event)
* Evaluator scores per session and per dimension (data point)
* Hypothesis deck exported (PDF/slide) (event)
* Outcome eval submitted (validated/invalidated) (event)
* Section regeneration (count, by reason/dimension)

---

## Technical Considerations

### Technical Needs

* **Multi-agent orchestration layer:** Agents run sequentially, passing structured context.
* **Web research/LinkedIn/job parsing modules:** Search, scrape, parse public company signals.
* **Structured schema per agent:** All agent outputs conform to defined data schemas.
* **Evaluator LLM:** Adheres to rubric-prompt architecture, logs all scores and rationales.
* **Workflow normalization layer:** semantically similar workflow candidates 
  must be mapped to stable canonical workflow labels before ranking
* **Workflow evidence gating layer:** candidate workflows must be assigned 
  Confirmed, Likely, or Excluded status based on company-specific supporting 
  signals
* **Deterministic recommendation layer:** final ranking and wedge selection 
  must run on structured signals using fixed scoring rules and deterministic 
  tie-breaking
* **KPI mapping layer:** selected wedges must map to stable KPI templates 
  and guardrail structures

### Integration Points

* Web search APIs (for public company data)
* LinkedIn public signal parsing tools
* Export to PDF and slide formats (e.g., PowerPoint, Google Slides)

### Data Storage & Privacy

* Session state: stored per company research run (ephemeral or per-account).
* Eval scores and PM override notes: logged for calibration & feedback loops.
* No PII stored beyond minimal session info; all target company data from public, non-sensitive sources to ensure compliance.

### Scalability & Performance

* Pipeline is designed for async step execution, parallelization where possible.
* Handles dozens of simultaneous sessions in parallel; performance bottlenecks flagged for regression.

### Potential Challenges

* **Low Public Signal for Some Companies:** Copilot must gracefully degrade, with confidence flags and appropriate user warnings.
* **Evaluator LLM Consistency:** Rubric prompts must be versioned, tested, and periodically recalibrated.
* **Mitigating Research Hallucinations:** All claims must cite real, verifiable sources—strict research grounding filter in evaluator.

---

## Milestones & Sequencing

### Project Estimate

* **Large:** 6–8 weeks from first commit to internal beta.

### Team Size & Composition

* **Small Team (3–4 total people):**
  * 1 Product Manager (full-time)
  * 2 Engineers (1 AI/backend, 1 frontend)
  * 1 Designer (part-time, focused on flows, slides, and UI polish)

### Suggested Phases

**Phase 1 — Core Pipeline + Recommendation Stability** ✓ Complete

* Built: Planner agent (LLM extraction of investigation targets at temperature 0, 
  Jina homepage fetch + web search fallback), company research agent (web search 
  + Jina fetch, competitive gap, implementation readiness assessment), workflow 
  mapping agent (Confirmed / Likely / Excluded statusing verified in code, 
  canonical label anchoring to investigation targets, deterministic keyword 
  matching for reclassification), wedge identification agent (all six dimensions 
  deterministic except business impact, confirmed-only scoring, deterministic 
  tie-breaking, deterministic top wedge selection in code), metrics agent 
  (deterministic KPI generation from selected wedge canonical label), brief 
  builder agent, SSE streaming pipeline, basic review UI.
* Dependencies met: Web search API, Jina fetch, deterministic scoring rules 
  agreed and implemented.

**Phase 2 — Evaluator + Deck + Evidence Snapshot Management** — Partially complete

Built:
* Evaluator agent with scored rubric (six dimensions, weighted composite, 
  pass/fail threshold, three-action decision logic)
* Deterministic Research Grounding caps applied in code after LLM scoring 
  (0 sourced entries → 2.0 cap; <2 sourced entries → 3.0 cap)
* Pass/fail UI with dimension scores and reasoning inline
* Hypothesis deck generator (6-slide structure)
* Trace evaluator — deterministic process integrity checks (14 assertions, 
  separate from output quality score, surfaces process_verified and trace_score 
  alongside output quality)
* Scope ambiguity check — deterministic, runs after brief and before slides; 
  blocks deck generation and routes to PM question when displacement risk is 
  high or brief risk_flags contain scope-related language

Not yet built:
* Export to PDF and editable slide formats
* Evidence snapshot caching layer (storage decision pending — see TODO in orchestrator.js)
* Snapshot timestamp visibility in UI (evidence_snapshot_timestamp field exists 
  in context output but is not surfaced in the UI)
* Explicit refresh behavior (re-run always does full re-research; no cached vs. 
  refresh distinction)
* Recommendation-change warning when refreshed evidence produces a different wedge

* Dependencies remaining: Storage decision for evidence snapshots (file-based, 
  in-memory, or database). Designer input for export format and UI refresh flow.

**Phase 3 — Outcome Evals** — Not started

* Planned: Post-kickoff feedback module (PM marks each recommendation as 
  Validated / Partially Validated / Invalidated), validation tracking dashboard, 
  calibration loop for scoring weights.
* Dependencies: Real-world usage data to seed feedback loop; close PM 
  involvement for marking validations.

---
