# Databricks Enterprise-Grade Agent Implementation Mapping

| | |
|----|------|
| **Version** | v1.4 |
| **Date** | 17 August 2026 |
| **Author** | Raphael |
| **Companion** | [Databricks Enterprise-Grade Agent Architecture](Databricks_Enterprise_Agent_Architecture_en.md) (conceptual v1.4) |
| **This doc answers** | For each conceptual module: **which Databricks components**, **how to assemble them**, **in what order** |
| **Since v1.2** | Runtime and tenancy: workspace patterns A/B/C, three Serving knobs, domain budget gates, Phase exits; version aligned with the architecture doc as v1.4 |
| **Since v1.1** | Domain Federation: Context namespaces, contract lifecycle, discovery as a pattern (not a GA Gateway API), domain tags from Phase 1, registry / cross-domain auth / quotas |
| **Since v1.0** | Product status as of Jul–Aug 2026: Unity AI Gateway GA, Managed MCP under Gateway, Lakeflow Connect / UC Secrets, Genie family names, pipeline unit tests, Genie Code Web Search |
| **Chinese original** | `Databricks 企业级 Agent 落地映射与实施路径.md` |

> **Read this first:** the Databricks data/AI stack moves fast. Every component is tagged **GA / Beta / Public Preview / Private Preview / Labs** (Labs = community, no official SLA). Preview / Beta / Labs need a **fallback**. Status is as of ~**August 2026**; re-check official docs before you commit a program.

---

## 1. Overview: module ↔ Databricks building blocks

| Module | Core components | Availability |
|--------|-----------------|--------------|
| **① Dynamic inputs** | Lakeflow Connect (100+ sources incl. SharePoint / Google Drive), Auto Loader, Lakeflow Spark Declarative Pipelines (stream+batch ETL), Lakeflow Jobs; optional **Genie Code Web Search** | Connect / Auto Loader / SDP / Jobs: **GA**; some Connect sources (OpenAI, PagerDuty): **Beta**; Genie Code Web Search: **Beta** (2026-08) |
| **② Knowledge & memory** | Unity Catalog (lineage / governance / **domain schemas**), Metric Views (corporate + domain scope), Vector Search (**metadata filters**), Genie Ontology + OntoRank, Ontobricks; **Genie Agents** on the business side | UC / Vector Search / Metric Views: **GA**; Genie Ontology: 2026 release; Ontobricks: **Labs** |
| **③ AI processing** | Lakeflow Jobs / Workflows, Mosaic AI Agent Framework, Agent Bricks, **Managed MCP (under Unity AI Gateway)**, Model Serving, Unity AI Gateway; department capabilities as **UC Functions / MCP contracts** | Agent Framework: **GA** (early 2026); Unity AI Gateway **core GA** (2026-08-04); Managed MCP ↔ Gateway: **Beta** (2026-08-06); some Service Policies / Agent Services: **Beta** |
| **④ Business execution** | Internal: Databricks Apps, Alerts, **Genie One Native Apps**, Lark–Meegle; external: CustomerLake; **people routing (org graph) + capability routing (contract discovery, as a pattern)** | Apps / Alerts / Genie One plugins: **GA**; CustomerLake: **Private Preview**; discovery: §2.4 (not a GA Gateway API) |
| **⑤ Eval / feedback / learning** | MLflow 3 (CLEARS + **domain Custom Judges**), Inference Tables, Lakehouse Monitoring, DABs + Git, **Lakeflow Pipeline unit tests** | MLflow / Inference Tables / Monitoring / DABs: **GA**; Pipeline unit testing: **Beta** (2026-07) |
| **Orchestration / governance** | Unity Catalog + **UC Secrets**, Unity AI Gateway, Lakeflow Jobs, MLflow 3, DABs + Git, system tables / budgets; **contract registry, cross-domain auth, domain quotas, workspace strategy, Serving caps** | UC / UC Secrets / Gateway core / Jobs / MLflow / DABs / Serving: **GA**; advanced Gateway policies, Agent Services, deep MCP bind: **Beta** |

---

## 2. Per-module mapping

### 2.1 Module ① Dynamic inputs

| Need | Component | Notes |
|------|-----------|--------|
| 100+ enterprise sources (OLTP, SaaS, external) | **Lakeflow Connect** | Managed ingest for most structured upstream/downstream sources. **2026-08:** SharePoint and Google Drive connectors **GA**; OpenAI and PagerDuty connectors **Beta** |
| Incremental, idempotent object-store loads | **Auto Loader** | Fits high-frequency telemetry |
| Unified stream+batch ETL to bronze | **Lakeflow Spark Declarative Pipelines (SDP)** | SQL/Python; streaming tables, materialized views, flows, sinks |
| Schedule, deps, orchestration | **Lakeflow Jobs** | Cron / DAG |
| Non-engineers building simple pipelines | **Lakeflow Designer** (GA, 2026-06) | Drag-and-drop + NL |
| Public web as live unstructured supplement | **Genie Code Web Search** (Beta, 2026-08) | Full-page code Agent for data/AI builders; **not** a substitute for governed ingest; for time-sensitive public facts that should not sit in the lake first |

**Implementation notes:**
- Express stream vs batch in SDP: telemetry/trades → streaming tables; periodic rollups → materialized views.
- Structured data → Delta + UC tags. Unstructured Context (OKRs, project status, competitive intel) → files + metadata (SharePoint / Drive GA connectors). Graph + embed in module ②.
- **Live web search** vs **governed ingest:** the former fills “visible on the public web *now*”; the latter is traceable enterprise assets. Tag confidence differently when Agents consume both.

### 2.2 Module ② Knowledge and memory

| Need | Component | Notes |
|------|-----------|--------|
| Identity, lineage, grants, tags per table/column | **Unity Catalog** | Trust base and later egress control |
| Bronze → silver modeling | **SDP (medallion)** | Business-defined cleanses |
| **Single source of metric truth** | **Metric Views** | Stops colliding definitions |
| Semantic retrieve of unstructured knowledge | **Vector Search** | Experiential memory, Context docs |
| Context as graph + authority ranking | **Genie Ontology + OntoRank** | Authority scores (incl. org graph) drive review priority and ④ routing |
| Tables as graph nodes | **Ontobricks** (Labs) | Joint structured/unstructured retrieve; no official SLA |

**Two memory classes:**
- **Domain knowledge / rules** (policy, SOP, metric specs) → Metric Views + governed UC objects; changes go through approval.
- **Experiential memory** (cases, samples) → Vector Search indexes, written back from ⑤.

**Review:** OntoRank-ranked; high authority/risk gets humans; rest AI pre-screen + sample.

**Context Namespace (turn on in Phase 1—do not retrofit a dirty index in Phase 4):**
- **Tables:** `catalog.schema` per domain; row/column sensitivity via **ABAC / row filters / column masks**.
- **Files / Volumes:** schema grants—do not pretend row filters apply to files.
- **Vectors:** `domain` (etc.) metadata; default **filter expressions** such as `domain = 'supply_chain' OR domain = 'corporate'`.
- **Metrics:** corporate Metric Views in a shared schema; department specs in domain schemas; cross-domain only via contract-shaped read-only views.
- **Cross-domain retrieve:** control plane authenticates first (Gateway / UC). **After auth, the Agent retrieve call injects the filter mix**—filters live on the retrieve path, not as a Gateway “pass-through.” Definition conflicts: corporate-shared first, then OntoRank + requesting-domain priority.

### 2.3 Module ③ AI processing

| Role | Component | Notes |
|------|-----------|--------|
| Deterministic workflows (cron / DAGs) | **Lakeflow Jobs / Workflows + SDP** | Standardize into Jobs; **Pipeline unit tests** (Beta) in Lakeflow Editor before prod |
| Custom / complex Agents | **Mosaic AI Agent Framework** (GA, early 2026) | Any harness (LangGraph / CrewAI / Claude Agent SDK); MLflow, UC register, Model Serving |
| Task-first, low-ops Agents | **Agent Bricks** | Serverless via Databricks Apps; native MCP; platform/builder surface—complement to Genie One / Genie Code |
| Governed tools: UC Functions / Genie Agents / Vector Search / DBSQL | **Managed MCP servers** via **Unity AI Gateway** | **2026-08-06:** managed MCP connectors (incl. Genie One / Genie Code) moved under Gateway (**Beta** bind)—central auth, access logs, cost. No scatter-shot tool wiring |
| Deploy Agents / models | **Model Serving** | Shared serving surface |
| Unified AI control plane (route, throttle, cost, guardrails, traces) | **Unity AI Gateway** | **Core GA 2026-08-04**; some **Service Policies** and **Agent Services** still **Beta** |
| Staff AI Q&A traces (automation path ③) | **Unity AI Gateway → Inference Tables** | Request/response as UC Delta |
| Lifecycle, tracing, eval | **MLflow 3** | GenAI-native spine |

**Genie product matrix (do not mix names):**

| Product | Audience | One line |
|---------|----------|----------|
| **Genie Agents** (formerly Genie Spaces) | Business / analytics | Conversational analytics on governed data |
| **Genie One** | Business teams | Copilot-style colleague; office embed (module ④) |
| **Genie Code** | Data & AI builders | Full-page code Agent (Web Search Beta) |
| **Agent Bricks** | Platform / app builders | Task-first managed Agents |

**Two AI roles:**
- **Supervisor:** SDP/Jobs outputs and sensors → MLflow Tracing + scorers.
- **Advisor:** **Genie Agents** over Metric Views + ontology Context → OKR loop review.

**Promotion criteria:** Gateway inference tables + MLflow on **frequency × stability × risk**; promote exploratory Q&A to Lakeflow Jobs. That gate is the core craft of module ③.

**Capability Contract lifecycle:**

| Step | How |
|------|-----|
| **Define** | Public capabilities as **UC Functions (UC Tools) or Managed MCP tool defs**; strongly typed **JSON Schema** I/O (align with Structured Outputs) |
| **Publish** | Governed catalog (e.g. `corp.agent_registry` or `published_tools` under a domain schema); tags `agent-capability`, `domain`, `version`, `risk_class` |
| **Authorize** | Who may discover / invoke: UC grants + Gateway identity; default same-domain; cross-domain needs extra auth |
| **Discover** | Module ④ searches the registry by intent (§2.4)—no hardcoded Job/endpoint names |
| **Change** | Compatible (optional fields) can ship in-domain; breaking (required fields, semantics) hits the central gate and dual-runs at least one version window |
| **Deprecate** | `deprecated` + sunset date; still findable; callers get a hard error, not silent failure |

### 2.4 Module ④ Business execution

**Internal:**

| Channel | Component | Fit |
|---------|-----------|-----|
| Email / formal record | **Databricks Alerts / notifications** | Low-frequency operating reports |
| Interactive dept app | **Databricks Apps** | Feedback / approval / drill-down |
| Office embed (sheets + IM) | **Genie One Native Apps** (Sheets / Excel / Slack / Teams, **GA 2026-08**) | Lowest friction; Apps stay for custom UX |
| PM tasks | **Lark / Meegle** (build: open APIs / webhooks; maybe Lark CLI) | Recommendations become workflow items |

**External:**
- **CustomerLake (Agentic CDP, Private Preview):** infinity campaigns on customer context.
- **Fallback:** activation Job (Model Serving → channel) until CustomerLake is usable.

**Implementation notes:**
- Reuse the Genie Ontology org graph for people routing; always read latest.
- Execution grades (full auto / human review / suggest-only) via Agent Framework HITL; money and external commitments default to human confirm.
- Attach UC lineage to every recommendation.
- Channel order: Genie One plugins → Apps (approval) → Meegle (tracking) → email (archive).
- **People vs capability routing:** humans via org graph; department Agents via contract discovery—not one hardcoded table.
- **Agent Discovery (pattern; verify product names before you commit):** do not hardcode cross-dept routes. Preferred order: (1) query UC Functions / MCP tools tagged `agent-capability` (**most stable today**); (2) if **Agent Services (Beta)** is on, use its tool-search for intent match; (3) Genie Agent APIs as assist only. **Do not document Unity AI Gateway GA “Tool Search.”** If discovery fails: notify the human only.

### 2.5 Module ⑤ Eval / feedback / learning

| Need | Component | Notes |
|------|-----------|--------|
| Agent/process quality (dev = prod) | **MLflow 3 eval + monitoring** | **CLEARS:** Correctness, Latency, Execution, Adherence, Relevance, Safety |
| End-to-end traces, long retention | **MLflow Tracing → OTEL → UC tables** | Serverless, governed observability |
| Business feedback from automated actions | **Inference Tables** + CDP/CustomerLake | Fast policy-layer signal |
| Data/model drift | **Lakehouse Monitoring** | Infra-layer continuous signal |
| Versioned, reversible learning | **DABs + Git folders** | Seatbelt on “learning” |
| Pipeline tests before prod / hard-learning | **Lakeflow Pipeline unit tests** (Beta, 2026-07) | Python/SQL tests with mocks for SDP / Auto CDC / Expectations; dual CI/CD with MLflow |

**Implementation notes:**
- Three feedbacks → three layers: business data → policy; staff task feedback → knowledge/process; traces → infra.
- **Dual CI/CD:** model/Agent changes through MLflow eval; SDP/Job changes through pipeline unit tests, then DABs. Hard learning must not skip green tests.
- **Rollback:** MLflow drift/degrade → DABs + Git to last good. An action, not an incident.
- **Decoupled domain eval:** CLEARS ⊥ domain KPIs (quality/safety/cost vs business outcome).
  - **Local:** MLflow 3 Custom Judges in the domain DAB (e.g. false-decline rate; ROI lift).
  - **Central:** Gateway on Inference Tables runs global CLEARS (esp. Safety, Latency, Cost) and drift—global compliance only.
- **Attribution:** holdout / A/B on external reach.
- **Write-back:** soft → Vector Search; hard → promote + green tests → Job / Agent version.

### 2.6 Orchestration / governance

| Duty | Component | Notes |
|------|-----------|--------|
| Schedule, trigger, deps, retry, rollback | **Lakeflow Jobs / Workflows** | Deterministic backbone |
| Grants, lineage, egress (data stays in cloud) | **Unity Catalog** | Object governance |
| API keys / credentials for Agents | **UC Secrets (GA)** | Same control plane as data |
| AI traffic: route, throttle, guardrails, logs, cost | **Unity AI Gateway** | Core **GA** 2026-08-04; advanced policies / Agent Services **Beta** |
| Managed tool auth, logs, cost | **Managed MCP ⊂ Gateway** | Deep bind **Beta** 2026-08-06 |
| Observability | **MLflow 3** | Trace, eval, monitor |
| CI/CD (models + pipelines) | **DABs + Git** + **pipeline unit tests (Beta)** | Dual gates |
| Cost and quotas | **System tables + budget alerts** | Tag `domain` / `agent_id` / `env`; over threshold: Gateway **throttle first**, then alert—do not take down the control plane |
| Workspace / tenancy | **Default: one workspace + one catalog per domain** (§4 decision 7) | Catalog = what you see; workspace = failure domain and bill. Pattern B: registry stays in the central catalog |
| Serving and job concurrency | **Model Serving caps + cluster policy / serverless budgets + Gateway identity throttles** | Three knobs below; no unbounded cross-domain fan-out |
| Contract registry and discovery | **UC Functions / MCP + tags** | Lifecycle §2.3; discovery §2.4 (pattern) |
| Cross-domain auth | **UC grants + Gateway identity / Service Policies (advanced = Beta)** | Auth first; retrieve layer injects filters |
| Domain isolation | **catalog.schema + Vector Search filters + domain Metric Views** | From Phase 1; Phase 4 only adds cross-domain calls |

> Security side-effect lives here: UC for grants, egress, secrets; Gateway for AI traffic and managed tools. Federation does not weaken this: cross-domain emits **contract-shaped conclusions**, not raw peer Context.

**Three runtime knobs (no capacity formulas—lock the control points):**

1. **Model Serving / Agent endpoints:** one endpoint per domain (or per contract); **provisioned concurrency cap**; no unlimited shared endpoints. Suggest-only internal can scale-to-zero; approval flows and external reach keep a small provisioned floor (cold start). Cross-domain calls: Gateway throttle **by caller identity**.
2. **Lakeflow Jobs / SDP:** domain cluster policies or serverless budgets; hard-learning releases must not steal the pilot domain’s prod queue. Pipeline unit tests stay in CI.
3. **Retrieve and SQL:** Vector Search and DBSQL warehouses queued/warehoused **per domain**. Gateway throttle *is* the domain quota enforcer—do not build a second limiter.

---

## 3. Phased rollout (crawl → walk → run)

Foundation → knowledge → one Agent loop → multi-Agent expansion. Each phase has outputs and exit criteria.

### Phase 0 — Governance foundation
- **Goal:** data trusted, governed, traceable; credentials and egress under control.
- **Components:** Unity Catalog + **UC Secrets (GA)**, Lakeflow Connect + Auto Loader + SDP to bronze/silver, Unity AI Gateway core in place.
- **Output:** governed lake + lineage + secrets in UC; **workspace pattern A written down** (one workspace, one catalog per domain; §4 decision 7).
- **Exit:** core business data in-lake with lineage and tightened grants; Agent keys not in local/ungoverned config; tenancy policy recorded (default A, B upgrade triggers explicit).

### Phase 1 — Knowledge layer
- **Goal:** trusted knowledge + retrievable Context; **domain tags and schema isolation from day one** so Phase 4 is not isolating a dirty index.
- **Components:** Metric Views (corporate schema + pilot domain schema), Vector Search (`domain` metadata, default filters), Genie Ontology; accept with **Genie Agents**.
- **Output:** one metric definition per key KPI; Context retrievable; at least `corporate` + one business `domain` namespace.
- **Exit:** Genie Agents answer correctly on governed data; in-domain retrieve does not hit other departments’ docs.

### Phase 2 — First Agent unit (narrow pilot)
- **Goal:** one high-value domain Agent, **suggest-only**.
- **Components:** Agent Framework / Agent Bricks, Managed MCP via Gateway, Model Serving (**concurrency cap**), Gateway (route/throttle/cost/trace); register the pilot’s public capability in UC per §2.3 even if only same-domain calls for now.
- **Output:** e.g. “conversion drop attribution and recommendations.”
- **Exit:** stable useful suggestions on real data; full traces; no tools bypassing Gateway/MCP; at least one schema’d public contract; Serving capped and billed with `domain` tags.

### Phase 3 — Close the learning loop
- **Goal:** ③→④→⑤; the Agent can improve itself.
- **Components:** MLflow 3 (global CLEARS + **that domain’s Custom Judge**), Inference Tables, Lakehouse Monitoring, DABs + Git, **pipeline unit tests (Beta)**, holdout/A-B.
- **Output:** write-back to ②/③; promote/rollback live; hard-learning pipelines must test green.
- **Exit:** one full “feedback → improve → holdout-validated effect”; both model and pipeline gates have actually passed *and* blocked.

### Phase 4 — Execution + multi-Agent (federation)
- **Goal:** internal/external delivery; **Domain Federation connects already-isolated domains**—this phase is **discovery and cross-domain calls**, not retrofitting isolation.
- **Components:** Apps + Alerts + Genie One Native Apps; CustomerLake (PP, with fallback); **UC contract registry + discovery pattern (§2.4)**; per-dept MLflow evaluators; domain quotas and Gateway throttles.
- **Output:** each dept owns Context namespace and eval rules; cross-dept work via MCP/UC contracts; macro network exists.
- **Exit:** ≥3 domain Agents stable under isolation; ≥1 successful **cross-domain capability call and write-back** (fallback: notify human only); **load the pilot domain to its QPS quota and other domains’ Serving stays up** (noisy-neighbor test). Stay on pattern A unless §4 decision 7 upgrade triggers fire.

---

## 4. Decisions and trade-offs

1. **Agent Bricks vs Agent Framework vs Genie family:** task-first, low-ops → **Agent Bricks**; custom orchestration → **Agent Framework** (any harness); NL analytics → **Genie Agents**; in-office copilot → **Genie One**; builder assist → **Genie Code**. Mixable. Genie One ≠ Agent Bricks.
2. **Tools only via Managed MCP (Gateway):** UC Functions / Genie Agents / Vector Search / DBSQL as governed tools—no scatter-shot access. Auth, logs, cost on the Gateway plane.
3. **Preview/Beta needs a fallback:** CustomerLake still PP; MCP↔Gateway bind, pipeline tests, Genie Code Web Search, some Connect sources, advanced Gateway policies are Beta. Plan them in; keep activation Jobs, local/CI tests, kill-switch on web search.
4. **Promotion is a mechanism:** Gateway tables + MLflow quantify frequency/stability/risk; Job/SDP changes also need pipeline tests.
5. **Governance first:** UC + **UC Secrets** + AI Gateway before “data stays in cloud, conclusions leave.”
6. **Central vs federated:** keep central: secrets, Gateway identity, egress, global Safety/CLEARS, shared Metric Views. Push out: Context indexes, department specs, Custom Judges, in-domain Jobs. Discover via UC tagged registry—not a non-GA Gateway Tool Search.
7. **Workspace tenancy:** logical isolation is always Catalog. Split workspaces only for compute, billing, or compliance.

| Pattern | When | Upside | Cost |
|---------|------|--------|------|
| **A. One workspace + one catalog per domain** (default) | Phases 0–3; few domains; small platform team; no mandated account split | Simplest lineage, contracts, discovery; one Gateway | Compute contention—quotas and the three knobs must hold |
| **B. Hub workspace (gov) + domain workspaces (run)** | One domain’s Jobs/Serving crowd others, or audit wants split bills | Failure-domain split; budgets per workspace | Cross-workspace identity, DAB `target`s, extra hop on discovery; registry stays `corp.agent_registry` in the hub catalog |
| **C. One workspace per domain** | Hard regulation, post-merger IT, mandated zero-trust between domains | Smallest blast radius | Registry / Ontology / Gateway need hub copies; ops cliff—**do not pick by default** |

On B: same UC metastore + SCIM; domains host implementations, public contracts stay central; DABs use `gov` / `domain_x` targets—no incompatible bundle shapes per domain. Splitting workspaces **does not** redo Context namespaces.

---

## 5. Risks and dependencies

- **Private Preview:** CustomerLake timing is not yours → fallback required.
- **Beta still needs fallback:** Gateway *core* is GA; advanced Service Policies / Agent Services, MCP deep bind, pipeline tests, Genie Code Web Search are not zero-risk prod deps.
- **Discovery productization:** cross-dept tool search may be Agent Services (Beta) or a homegrown registry query; verify APIs; failure path is “notify human only.”
- **Contract sprawl / breaking changes:** silent caller failures → §2.3 versioning and dual-run.
- **Over-wide cross-domain filters:** “open all” kills namespaces → whitelist domains + audit.
- **Noisy neighbor:** uncapped shared Serving/warehouses → Phase 4 quota-saturation drill; no broadcast fan-out.
- **Splitting workspaces too early:** B/C before a central registry shatters discovery → A + quotas first; upgrade only on decision 7 triggers.
- **Custom integration:** Lark/Meegle is build work; Genie One plugins reduce some channels, not PM-system integration.
- **Attribution must be designed in Phase 3**, not bolted on later.
- **Labs:** Ontobricks has no official SLA; have an alternative on the critical path.
- **Status freshness:** ~August 2026; re-check docs before kickoff.

---

## 6. Theory map: essay claims ↔ Databricks (including gaps)

Maps *[Topological Wiring of High-Dimensional Logic and Reality Anchors](Enterprise_AI_thoughts_en.html)* onto platform capabilities, and **marks what you must build yourself**. This section is *why we govern this way*, not *which widget to click*.

### 6.1 Scorecard

| Essay idea | Databricks | Fit |
|------------|------------|-----|
| KG = explicit atomic facts (symbolic) vs embeddings = continuous similarity (connectionist), unified in Context | **Genie Ontology / OntoRank** (symbolic) + **AI Search** (vectors) | ⭐ Strong |
| Context purification: hybrid retrieve + cross-entropy rerank | **AI Search** hybrid (lexical + ANN, RRF) + **cross-encoder rerank** | ⭐ Strong |
| Semantic quality red line / Gatekeeper | **OntoRank authority scores** | ⭐ Strong |
| Strong types: JSON Schema / Protobuf constrained decoding | **Structured Outputs** (constrained decoding) | ✅ Hit |
| Deterministic skeleton (FSM / LangGraph) + LLM as local parser/router | **Lakeflow Jobs** + **Agent Framework** (LangGraph) | ✅ Hit |
| Open-ended planning as real-world experiment loop | **MLflow 3 + CLEARS** + holdout / A/B | ✅ Hit |
| Context roles (system prompt / RAG / observation / memory) | **Agent Framework** + **Managed MCP** + memory | ✅ Hit |
| Pre-softmax / logit-level UQ + hard stop before first token | **Not on hosted endpoints:** FM API / Gateway-fronted external models expose **sampled-token `logprobs`**, not full pre-softmax logits; generation is server-side | 🚫 Platform limit |
| Dirichlet UQ, aleatoric/epistemic split, ECE, credal sets | **Not on hosted endpoints:** needs model-head access or full logits; only custom serving of open weights | 🚫 Platform limit |
| Explicit renormalization-bias handling (UNK/OOD dump) | **Partial:** constrained decoding is platform; schema “spillway” is your design | ⚠️ Gap / build |

### 6.2 Strongest fit: symbolic × connectionist knowledge

The essay’s Tractatus split—triples as pictures of facts vs embeddings as smoothed similarity—maps onto **first-class complementary products**: OntoRank as native Gatekeeper signal; AI Search as hybrid retrieve + rerank. You do not have to stitch that layer yourself. That is the theoretical landing of module ②.

### 6.3 Hard limit: logit-level mechanisms

Hosted Foundation Model APIs and Gateway-fronted Claude/GPT expose **output-token `logprobs`**, not the full pre-softmax vector. You cannot intercept before the first token. Dirichlet / ECE / credal sets need a custom head.

The only path is **self-hosted open weights** (GPU + custom pyfunc). Cost: you give up managed FM convenience and you **cannot** do this on proprietary models. For an architecture that leans hosted + external models, treat logit-level UQ as **out of scope**.

### 6.4 Behavioral stand-ins for “OOD → silence”

- **Retrieve-confidence gate (most useful):** max similarity below threshold → refuse. No logits; matches “whereof one cannot speak.”
- **Sequence `logprobs` heuristic:** low confidence → review/refuse (coarse).
- **Semantic entropy / self-consistency:** N samples, measure disagreement ≈ epistemic uncertainty.
- **Separate OOD classifier** as a router step.
- **Gateway guardrails + CLEARS Safety** as last output gate.

> **One line:** logit internals are not implementable on Databricks hosted endpoints (unless you self-host open models, which miss Claude/GPT). The *intent*—refuse when out of distribution—can be approximated with **retrieve gates + logprobs + semantic entropy**. The platform gives fences and skeletons; taming probability inside the model is either self-host or a behavioral proxy.

---

## Appendix: sources

Checked against public docs and reporting (~**August 2026**):

- Databricks Blog — *Lakeflow: A new era of agentic data engineering* — https://www.databricks.com/blog/lakeflow-new-era-agentic-data-engineering
- Databricks Docs — *What is Lakeflow Spark Declarative Pipelines* — https://docs.databricks.com/aws/en/ldp/concepts
- Databricks Docs — *What is Auto Loader?* — https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/
- Databricks Blog — *Agent Bricks: Data + AI Summit 2026* — https://www.databricks.com/blog/agent-bricks-dais-2026
- Databricks Product — *Agent Bricks* — https://www.databricks.com/product/artificial-intelligence/agent-bricks
- Databricks Docs — *Evaluate and monitor AI agents (MLflow 3)* — https://docs.databricks.com/aws/en/mlflow3/genai/eval-monitor/
- Databricks Blog — *MLflow 3.0: Build, Evaluate, and Deploy Generative AI with Confidence* — https://www.databricks.com/blog/mlflow-30-unified-ai-experimentation-observability-and-governance
- Databricks Docs — *AI governance with Unity AI Gateway / AI Gateway-enabled inference tables* — https://docs.databricks.com/aws/en/ai-gateway/
- ITdaily — *Not pagerank, but ontorank: Databricks Genie Ontology* — https://itdaily.com/blogs/cloud/databricks-genie-ontology/
- GitHub — *databrickslabs/ontobricks* — https://github.com/databrickslabs/ontobricks
- Databricks Blog — *Introducing CustomerLake: The Agentic CDP embedded in Databricks* — https://www.databricks.com/blog/introducing-customerlake-agentic-cdp
- Databricks Docs — *AI Search retrieval quality guide (hybrid search & reranking)* — https://docs.databricks.com/aws/en/vector-search/vector-search-retrieval-quality
- Databricks Docs — *Structured outputs on Databricks (constrained decoding)* — https://docs.databricks.com/aws/en/machine-learning/model-serving/structured-outputs
- Raphael Zhu — *Topological Wiring of High-Dimensional Logic and Reality Anchors* — https://zhuzp98.github.io/files/AI%20Thoughts/Enterprise_AI_thoughts_en.html

**v1.4:** Version aligned with the architecture doc. Workspace patterns A/B/C (default A); Serving / Jobs / retrieve knobs; domain billing tags and noisy-neighbor exit; Phase 0/2/4 gates.  
**v1.2:** Domain Federation loop (contract lifecycle, UC registry discovery, Phase 1 isolation, control-plane registry/auth/quotas).  
**v1.1 product dates:** Unity AI Gateway core GA (2026-08-04); Managed MCP under Gateway (Beta, 2026-08-06); SharePoint/GDrive Connect GA; OpenAI/PagerDuty Connect Beta; UC Secrets GA; Genie Spaces → Genie Agents; pipeline unit tests Beta (2026-07); Genie Code Web Search Beta (2026-08); Genie One Excel/Sheets plugins (2026-08). Re-check official docs before kickoff.
