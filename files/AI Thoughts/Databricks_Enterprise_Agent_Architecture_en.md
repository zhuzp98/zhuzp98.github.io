# Databricks Enterprise-Grade Agent Architecture

| | |
|----|------|
| **Version** | v1.4 (conceptual architecture; v1.3 added domain federation; v1.4 adds deploy boundaries) |
| **Date** | 17 August 2026 |
| **Author** | Raphael |
| **Scope** | Turns the idea of “the enterprise as a learning Agent” into a coherent **conceptual architecture**. Component mapping and rollout live in *Databricks Enterprise-Grade Agent Implementation Mapping*. |
| **Chinese original** | `Databricks 企业级 Agent 架构设计文档.md` |

---

## 1. Core idea: the enterprise as a learning Agent

Treat the whole enterprise as one large Agent and it must run as a **closed loop**, not a one-way pipeline from input to action. Five functional modules plus a cross-cutting **orchestration / governance layer** map to the classic cycle: perceive → remember → reason → act → learn.

```
        ┌──────────── Orchestration / governance (control plane) ────────────┐
        │                                                                    │
   ① Dynamic inputs ──▶ ③ AI processing ──▶ ④ Business execution ──▶ (people / customers)
        ▲                     ▲                                      │
        │                     │                                      ▼
        │              ② Knowledge & memory ◀──── ⑤ Eval / feedback / learning ◀──┘
        └──────────────────────── (results flow back, memory updates) ────────
```

**The split that matters:** a one-way chain is automation. **Only the ⑤ feedback arrow makes the enterprise a learning, evolving Agent.** That is what separates this design from a generic data-automation platform.

### Six parts

| Part | Agent analog | One-line job |
|------|----------------|--------------|
| ① Dynamic inputs | Perception | Continuously collect internal and external batch/stream signals |
| ② Knowledge & memory | Memory | Distill raw input into trusted, retrievable knowledge |
| ③ AI processing | Reasoning | Abstract and automate processes; supervise and advise |
| ④ Business execution | Action | Route conclusions to the right people / customers; bridge to humans |
| ⑤ Eval / feedback / learning | Learning | Capture outcomes; write back knowledge; upgrade processes |
| Orchestration / governance | Control plane | Scheduling, permissions, observability, cost—end to end |

### Two layers: Agent unit vs Agent network

“Enterprise = one Agent” is a useful **starting metaphor**. A real company is closer to a **multi-agent system**—finance, supply chain, support, risk, marketing—each with its own ①–⑤. Keep the layers distinct so implementation granularity does not collapse:

- **Micro (Agent unit):** sections 2–7—the five modules plus governance **inside one domain Agent**.
- **Macro (Agent network):** collaboration, tasking, shared knowledge, and unified governance **across** domain Agents.

The same five-module frame holds at both layers: the macro layer is an Agent made of Agents. That is why the abstraction scales.

### Macro evolution: domain isolation and capability contracts (Domain Federation)

When the company has dozens of departments with independent goals, the macro layer cannot scale by “adding more hardcoded routes.” Use **domain-driven federation**:

1. **Capability Contract:** each department Agent publishes a standard I/O protocol and capability statement (think OpenAPI / MCP tools). Cross-department work is scheduled against contracts, not “call that team’s Job by name.”
2. **Federated Governance:** domains keep local goals and execution autonomy, while sharing the central control plane for safety gates, cost quotas, egress control, and baseline lineage—**decentralized execution, centralized risk and security**.

People routing (org chart → the right human) and capability routing (contract → the right Agent) stay separate; see §5.2. In-domain learning can be fast; cross-domain contracts and global safety still hit the central gate; see §6.5.

---

## 2. Module ① Dynamic inputs (perception)

### 2.1 Two data shapes

Everything that feeds the loop splits, roughly, into two processing classes. Boundaries are fuzzy (on-chain data and social graphs are semi-structured), but the cut is good enough for workstreams.

**Structured data**—tabular, tagged, queryable:

| Subclass | Content | Character |
|----------|---------|-----------|
| Product telemetry | User behavior, page/feature interactions | High frequency, streaming, large volume |
| Downstream business events | Transactions, top-ups, and similar commercial acts | High-value signals, tied to revenue |
| Upstream / external | On-chain addresses, ad channels, traffic features, social graphs | Semi-structured, external, needs cleaning and join keys |

**Unstructured data** is mostly **enterprise Context**—hard to land as tables: annual/quarterly OKRs and plans, how departments actually run, new products/features, internal project status, competitor intel, macro news. Time-sensitive public information that should **not** be pre-ingested can still enter as a perception-side supplement (implementation: Genie Code Web Search and similar). Label it separately from lake-governed Context for confidence and provenance.

> Value density differs: structured data answers *what happened*; unstructured Context answers *why, and in what setting*. The first drives real-time reaction; the second decides whether the judgment is even in the right frame.

### 2.2 Landing in the lake and how it is organized

**Structured data** is the easier path: tables, tags, and two ingest modes—**streaming** (telemetry, trades) vs **batch** (periodic aggregates).

**Unstructured Context** is the hard part and the design core of this module. Organize it as a **topology / graph**, not a flat table: each piece of Context is a node, edges are semantic or citation links, weights come from authority scores. Databricks **OntoRank** (the ranking behind Genie Ontology) is the natural fit.

Two clarifications on OntoRank:

1. **It scores authority / trust, not narrow data quality.** Like PageRank, but over heterogeneous enterprise assets (code, docs, tables, files). Signals include owner credibility, usage breadth, ties to certified assets, freshness. It answers “which definition should the Agent adopt?”—complementary to completeness/accuracy “quality.”
2. **Org structure is already a signal.** OntoRank already uses the **org graph** (who accesses what, how often, role). The real **incremental dimension is workflow structure**—where in a process a fact was born and which decision it feeds. Inject that and the node space goes from “org” to “org × process,” closer to how the company actually runs.

### 2.3 A convergence line

Structured tables and unstructured Context eventually **meet in one ontology / knowledge graph** (Ontobricks can materialize Unity Catalog tables as graph nodes). The two-shape split is a **collect-and-process** distinction; in module ② they become one governed, searchable graph. That line is how ① hands off to ②.

> **Without this layer:** the Agent reasons from generic common sense and stale snapshots; business signals lag; departments duplicate collection and fight over definitions; nobody can say where a number came from.

---

## 3. Module ② Knowledge and memory

### 3.1 Job: from “input” to “trusted knowledge”

This module **does not create new business/production data**. It **refines** module ① inputs, with human review on the precise extracts, turning volume into knowledge an Agent can trust and retrieve. In Agent terms: long-term memory. ① is ingest; ② is remember *correctly*.

> **Clarification:** runtime **Memory, Trace, Evaluation** metadata should also land back in the lake as fuel for ⑤. So: no new **business** data, but a continuous stream of **data about the Agent’s own operation**.

Two memory subclasses, implemented differently:

| Subclass | Nature | Examples | How it updates |
|----------|--------|----------|----------------|
| **Domain knowledge / rules** | Governed; change needs approval | Policy, SOP, business definitions, metric specs | Human + approval |
| **Experiential memory** | Evolutionary; accumulates automatically | Cases, feedback samples, wins/losses | Written back from ⑤ |

The first is the Agent’s “statute”; the second is “intuition.” Together they are judgment Context.

### 3.2 Structured data: a traceable map

- **Unity Catalog lineage** is the identity card for every table and column: any number can be traced to source and transforms—the trust base.
- Humans govern raw/bronze against business definitions into **silver**, then **Metric Views** as the semantic metric layer.
- That **medallion + Metric Views** path is a traceable **data → information → knowledge** map.
- **Payoff:** Metric Views are a **single source of metric truth**, so the Agent does not get conflicting definitions—the structured counterpart of OntoRank’s “adopt the authoritative definition.”

### 3.3 Unstructured: prepare, sync, periodically review

- **Prepare:** preprocess, chunk, embed, land as retrievable nodes.
- **Sync:** keep pulling new ① information; incremental updates.
- **Periodic human review:** keep the “new input → knowledge” pipeline honest so bad Context does not poison memory.
- **Not only add—change and retire:** replaced knowledge needs **versioning and expiry**, or Context rots. Review also deprecates stale nodes.

### 3.4 Review: stratify by authority and risk

Human review does not scale to 100%. **High-authority, high-risk knowledge gets close reading**; the rest is AI pre-screen plus sampled audit. Reuse OntoRank **authority scores** as the queue so scarce humans sit on what must be trusted.

### 3.5 Downstream interface

Module ③ consumes **structured knowledge** via Metric Views / SQL and **unstructured knowledge** via vector retrieval. Both can live on **one governed ontology / graph**—that is *physical* convergence after ingest.

**Default consume semantics are not “search the whole graph.”** The graph may be physically unified; retrieval must be **logically isolated** (§3.6). A department Agent sees **its domain + corporate-shared knowledge** by default. Cross-domain joint retrieval is an orchestration-layer exception after auth—not the default API of module ②.

> **Without this layer:** raw ungoverned data drives decisions; hallucinations and wrong metric specs become “facts”; nothing compounds; audit is impossible. Highest **risk and trust** cost in the stack.

### 3.6 Context namespaces for many departments (Context Domains)

In a large company with many independent departments, flattening all unstructured Context (OKRs, SOPs, project status) into one index **destroys signal-to-noise**. Same for metrics: if every team’s “conversion” and “active” share one semantic layer, the Advisor gets colliding definitions.

- **Domain partitioning:** give departments/projects a **Context Namespace** (unstructured) and a **Metric Scope** (structured). Default retrieve: **this domain + corporate-shared** (HR policy, compliance, group-level Metric Views).
- **Authorized cross-domain retrieve:** only when module ④ starts cross-function work, and only after the control plane checks permission. If two domains disagree on an authoritative definition: **corporate-shared definition first, then OntoRank + requesting-domain priority**—do not just open the filter.
- **Cross-domain read-only views:** department metrics are not globally visible by default. Expose contract-shaped read-only metrics/conclusions, not “company-wide facts.”

---

## 4. Module ③ AI processing (reasoning)

### 4.1 Stance: automate the work, do not unleash the model

This module is **not** “let AI execute freely.” Start from **how the business actually runs**, abstract and modularize, then automate heavily. One line: **deterministic work stays deterministic (workflows); judgment goes to AI (supervise and advise).**

### 4.2 Three paths to find automation

A complementary triangle: declared need + emergent build + observed use.

| Path | How | What you get |
|------|-----|----------------|
| ① Top-down | Talk to the business at **dept / project / person** | Processes people **say** they run |
| ② Middle-out | Staff build **skills** via Databricks + Claude Q&A, then distill to workflows | Processes people **built** |
| ③ Bottom-up | Track Agent **traces** (Unity AI Gateway → UC inference tables) | Processes people **actually use** |

> The three rarely agree. **Cross-checking** said / built / used is the best filter for real vs fake demand.

### 4.3 From exploratory AI use to production automation: promotion criteria

Identified automations become **cron jobs** or **dependent DAGs**—this is where employee workflow actually links to ① (data) and ② (knowledge).

> **Put a gate on it.** When does a repeating AI Q&A become a deterministic job? Define **promotion criteria**, e.g. **frequency × stability × risk**. That gate is the craft of turning skills/traces into production automation.

### 4.4 Two AI roles here

- **Supervisor:** watch each workflow pipeline—summarize / monitor / learn from **outputs and sensors** (business transforms of ①). How anomalies are *handled* belongs to orchestration; this module only defines **how anomaly signals are produced and reported**.
- **Advisor:** loop-review tasks / objectives / OKRs against module ② Context; emit execution and commercial recommendations.

### 4.5 Boundaries with neighbors

- **③:** processed intelligence and review (what happened, what drifted, where to go).
- **④:** wrap recommendations into executable, graded delivery.
- **⑤:** after execution, write back outcomes.

Analysis → action → retrospective. Do not swap jobs.

> **Without this layer:** either endless manual toil, or unconstrained AI (unauditable). Automation never scales; AI stays a clever toy.

---

## 5. Module ④ Business execution (action)

### 5.1 Job: get intelligence to the right person, actionable and traceable

Heavy dependence on **enterprise Context**: org chart, hierarchy, current Lark / Meegle (or equivalent PM) status. Route results to the **department lead** or **owner** for that problem.

### 5.2 Two routes: the right person ≠ the right Agent

At scale, “find a human” and “find a capability” are different maps. Do not share one hardcoded table.

**People routing (org graph → human):** consume the **OntoRank org graph** already maintained in ①/②. Do not rebuild a people map. Keep it **fresh**—reorgs turn “right person” into “wrong inbox.” Always read the latest graph from ②.

**Capability routing (contract → Agent):** a static org chart will not stay true with many departments. Cross-department tasks should **discover** the matching department Agent/API via the macro **Capability Contract** (**Agent Discovery**), then invoke published capabilities. The org graph answers *who to notify*; the contract answers *which Agent to call for what*.

> Fallback if discovery fails: notify the human only; do **not** auto-invoke the other Agent; do **not** fall back to a hardcoded Job name.

### 5.3 Two delivery directions: internal coordination + external reach

**Internal**—to staff and leads:

| Channel | Shape | Fit |
|---------|--------|-----|
| **Email** (Databricks alerts / notifications) | One-way, formal, auditable | Low-frequency archive (e.g. operating reports) |
| **Databricks App** (dept mini-app) | Interactive | Feedback / approval / drill-down |
| **Genie One Native Apps** (Google Sheets / Excel / Slack / Teams) | Inside tools people already use | Analysis and structured recommendations land in the spreadsheet or IM (Sheets/Excel native plugins GA as of Aug 2026)—lowest-friction internal path |
| **Databricks × Lark/Meegle** (open APIs / webhooks; maybe Lark CLI) | Inside the PM system | Recommendations become tasks |

> **Channel rule:** urgency × interactivity × auditability—push (mail/IM) vs pull (App / tasks / in-sheet collab). Prefer surfaces already open (sheets / Slack·Teams), then approval or project systems.

**External**—automated customer reach via **Databricks CustomerLake (Agentic CDP)**:

- Native agentic CDP on Databricks: identity, audiences, activation.
- **Infinity campaigns:** a continuous agentic loop on customer context, instead of one-shot campaigns—much of what used to be a **manual trigger** can automate.
- Intelligence from ①–③ can drive CustomerLake directly.

> **Structural note:** an infinity campaign *is* a miniature enterprise-Agent loop (perceive customer context → decide → activate → learn)—a prebuilt “customer-ops Agent” and a node in the macro network. Module ④’s external arm should **orchestrate it, not rebuild it**.
> **Status:** CustomerLake is **Private Preview**—plan it in, keep a fallback (activation Job / manual reach) until GA.
>
> **Product names (do not mix):** business copilot = **Genie One** (incl. native apps); governed conversational analytics = **Genie Agents** (formerly Genie Spaces); builder/code = **Genie Code**; task-first platform Agents = **Agent Bricks**.

### 5.4 What gets delivered

Including: operating reports, meeting briefs, project tracking, improvement points—**cross-function** to product / eng / ops. That coordination *is* macro multi-agent collaboration.

### 5.5 Three design rules

- **Execution grades live here:** full auto / human review / suggest-only. Money and external commitments default to human confirm.
- **Channels are sensors:** not fire-and-forget. Capture receipts (Meegle status, App approve/reject, mail response, sheet/IM traces, customer reaction) for ⑤.
- **Every recommendation carries lineage:** sources and reasoning (Unity Catalog lineage from ②). Humans adopt what they can check.

> **Without this layer:** insight dies in a dashboard (“analysis paralysis”); wrong inbox or no evidence → no adoption; missed real-time customer windows. Compute never becomes business outcome.

---

## 6. Module ⑤ Evaluation / feedback / learning (the loop)

This closes ③→④→⑤ and upgrades automation into a **learning Agent**.

### 6.1 Three feedback sources → three learning layers

Feedback is not one blob:

| Source | Content | Writes back to | Cadence |
|--------|---------|----------------|---------|
| **① Automated execution data** | Reach tracking (CDP / CustomerLake) | **Policy:** ③ models, ④ reach strategy | Fast (min–hours) |
| **② Staff task feedback** | Done on time? Written notes / docs | **Knowledge + process:** ② experiential memory; ③ workflows | Medium (days–weeks) |
| **③ Platform trace / CI** | CI/CD + VCS; traces and logs | **Infra:** local module fixes | Continuous |

> Match learning speed to signal speed: fast signals auto-tune; slow signals go through periodic human review (reuse ②’s review cadence).

#### Heterogeneous evaluation (domain-specific)

Risk cares about precision and compliance; growth cares about iteration speed and conversion. Do **not** score every Agent with one global KPI:

- **Local eval autonomy:** each domain Agent defines **Domain Evaluation Metrics** (e.g. false-decline rate; ROI/conversion) that drive in-domain soft/hard learning.
- **Global eval roll-up:** the center watches **safety/compliance, cost quotas, drift alerts, and cross-domain success rate**.

The two sets are **orthogonal**: global (e.g. CLEARS) is quality, safety, cost; domain metrics are business outcomes. Neither substitutes for the other.

### 6.2 Three write-back shapes

- **Update unstructured docs → soft learning:** ② experiential memory (slow, accumulative, low risk).
- **Change a process step → hard learning:** ③ workflows / models (fast, structural, risky).
- **Fix structured knowledge / metric specs:** when feedback shows the *definition* is wrong, change ② domain knowledge / Metric Views—not only the prose.

### 6.3 CI/CD + VCS: the seatbelt on “learning”

**Learning = changing production, and change is dangerous.** CI/CD + VCS make continuous learning **versioned and reversible**.

Two change surfaces, dual protection:

| Surface | Gate before prod | Typical tools |
|---------|------------------|---------------|
| **Model / Agent** | Eval bars, regressions, drift alerts | MLflow Evaluation / Tracing / Monitoring + Asset Bundles |
| **DAG / ETL** | Unit tests (mock data for pipeline / CDC / expectations), then release | Lakeflow Pipeline unit tests (SDP) + Asset Bundles + Git |

If hard learning changes the pipeline under a ③ workflow, **model eval alone is not enough**—pipeline tests must go green too.

> **Symmetric criteria:** ③ has **promote**; ⑤ needs **demote / rollback**—alert and revert to last known good when quality drops. With VCS, rollback is an action, not an incident.

### 6.4 Attribution discipline (the usual hole)

Many things move at once. **You cannot cleanly attribute “business got better” to “that decision.”** Without attribution, “learning” chases correlation and drifts.

> On external reach, use **experimental discipline**—holdouts, A/B (CDPs usually support audience holdout). Only **causal** lift should write back.

### 6.5 Close the loop: ⑤ writes ② and ③; gates by blast radius

The two back-arrows—**into ② (memory)** and **into ③ (workflow/model)**—are the loop. **Learning and governance must both hold**, but gates **scale with blast radius**, or federation dies under “everything needs central approval”:

| Change | Who gates | Examples |
|--------|-----------|----------|
| **In-domain, low risk** | Domain autonomy (still local CI/CD + rollback) | Domain memory, Job params, Custom Judges |
| **Cross-domain or global** | Central orchestration gate + ③ promotion criteria | Breaking contract changes, shared Metric Views, egress, global Safety, cross-domain retrieve auth |

> **Without this layer:** the system automates but does not evolve; ROI is unprovable; drift accumulates in the dark.

---

## 7. Orchestration / governance (control plane)

Without an explicit control layer—**who schedules, on what trigger, how failure rolls back**—capability scatters. Six jobs:

1. **Orchestration:** triggers, deps, retries, rollback.
2. **AuthZ / security:** which Agent may use which data and tools (least privilege); cross-domain retrieve and invoke must pass here.
3. **Observability and evaluation:** logs, traces, metrics, plus **evaluation as a first-class peer of tracing**—MLflow Evaluation / Trace so “was the decision *right*?” is measurable, not only “it happened.” Global CLEARS / Safety roll up here; domain business KPIs are not centrally scored.
4. **Cost and quotas:** monitor and throttle; in federation, **per-domain** quotas so one domain cannot blow the global bill.
5. **Contract registry and discovery:** publish, version, deprecate department capabilities; feed ④ Agent Discovery. Breaking changes hit the central gate.
6. **Domain isolation policy:** default Context Namespace / Metric Scope, and the exception path for cross-domain auth.

Enterprise vs toy Agents is almost entirely this layer.

**Side effect—data security:** staff running a workflow **do not need the raw underlying numbers** (e.g. group revenue). Data **stays in cloud**, **egress is tightly controlled** (Unity Catalog + Unity AI Gateway): conclusions/actions leave; raw data does not. Less manual data wrangling, much higher safety—need-to-know.

> **Without this layer:** leaks, runaway token bills, no replay, no accountability, no rollback—why toy Agents die in production.

### 7.1 Deploy boundary: logical federation ≠ shared physical warehouse

Domain Federation answers *what you may see / whom you may call*. **Workspaces and compute** answer *who saturates the cluster / whose bill it is*. Do not mix them: **catalog/schema = logical boundary; workspace/budget = failure domain and quota**.

Three principles; cut-over patterns live in the mapping doc:

1. **Shared control plane, isolated data plane:** one UC metastore, Gateway, secrets, contract registry if possible; heavy Serving and job compute throttled or split by domain.
2. **Verticalize one domain loop before horizontal fan-out:** prove throughput and quotas on the pilot domain, then copy. Do not stand up N platforms on day one.
3. **Cross-domain is sparse authorized calls, not broadcast:** contracts already imply this; unbounded fan-out at peak will take down the callee domain.

Splitting workspaces does **not** reinvent Context isolation—namespaces stay in Unity Catalog.

---

## 8. One end-to-end cycle

Example: conversion drop on a product line.

1. **① Perceive:** streaming telemetry and transactions show a sustained drop; OKR Context says this line is a quarterly priority.
2. **② Remember:** Metric Views give one definition of “conversion”; lineage to the field; related cases retrieved from experiential memory.
3. **③ Reason:** supervisor flags the anomaly; advisor loop-reviews against OKRs → attribution + recommendations.
4. **④ Act:** people-route to the product owner (Meegle task, with lineage). If the fix depends on supply-chain inventory policy, **discover** that domain Agent via contract and invoke it—do not hardcode their Job. In parallel, CustomerLake win-back on the affected cohort. High-risk steps need human confirm.
5. **⑤ Learn:** task completion + write-ups + CustomerLake results (with holdout) flow back—domain experience into ②; proven playbooks promote into ③ under domain criteria; shared metrics or contract changes go through the central gate. Auditable, rollback-able.

After one cycle the enterprise Agent “knows” this class of problem better.

---

## 9. Design principles

1. **Loop first:** feedback is what makes a learning Agent; a chain is only automation.
2. **Deterministic → workflow; judgment → AI.**
3. **Automate real processes; do not unleash the model.**
4. **Triangulate demand:** said / built / used.
5. **Data stays in cloud; only conclusions leave.**
6. **Authority scores drive priority** (review queues and routing).
7. **Every change is versioned and reversible** (dual gates: model eval + pipeline tests).
8. **Attribution discipline** (holdout / A/B).
9. **Learning and governance together:** in-domain low risk can be local; cross-domain contracts and global safety are central.
10. **Reuse, don’t rebuild:** org graph, CustomerLake, Genie One plugins.
11. **Domain autonomy, central gates:** default local Context / metrics / eval; cross-dept only via contracts—no hardcoded routes, no flattened search.
12. **Logical federation ≠ shared warehouse:** Catalog for what you see; workspace / domain quotas for compute and bill. Shared control plane, isolated data plane under load.

---

## 10. Implementation mapping

Component choices, availability, and phased rollout:

**[Databricks Enterprise-Grade Agent Implementation Mapping](Databricks_Enterprise_Agent_Implementation_Mapping_en.md)** (v1.4, August 2026)

Chinese: `Databricks 企业级 Agent 落地映射与实施路径.md`

That document also maps this architecture to *[Topological Wiring of High-Dimensional Logic and Reality Anchors](Enterprise_AI_thoughts_en.html)* and marks platform gaps. Update the mapping doc first when product status moves; update this doc when conceptual channels or loop principles change.

Quick index (details in the mapping doc):

- **① Input:** Delta Lake / Auto Loader / Lakeflow Connect / SDP (incl. file SaaS connectors); optional Genie Code Web Search
- **② Memory:** Unity Catalog (lineage, governance, domain schemas) / Metric Views (corporate + domain scope) / Vector Search (`domain` filters) / Genie Ontology (OntoRank) / Ontobricks; Genie Agents on the business side
- **③ Reason:** Jobs/DAG / Model Serving / Agent Framework / Agent Bricks / Managed MCP via Unity AI Gateway; department capabilities as UC Tools / MCP contracts
- **④ Act:** Databricks Apps / alerts / **Genie One Native Apps** / Lark–Meegle / CustomerLake; people route via org graph, capability route via discovery
- **⑤ Learn:** Lakehouse Monitoring / MLflow / Inference Tables / DABs + Git / **pipeline unit tests**; domain Custom Judges + central CLEARS
- **Control plane:** UC + **UC Secrets** / Unity AI Gateway / contract registry and cross-domain auth / domain quotas and Serving caps / workspace strategy (default: one workspace, one catalog per domain)

---

## Appendix: sources

Product claims checked against public docs and reporting:

- ITdaily — *Not pagerank, but ontorank: Databricks Genie Ontology brings context and authority to AI* — https://itdaily.com/blogs/cloud/databricks-genie-ontology/
- Atlan — *What is Genie Ontology? Databricks' Context Layer, Explained* — https://atlan.com/know/ai-agent/databricks/genie-ontology/
- GitHub — *databrickslabs/ontobricks* — https://github.com/databrickslabs/ontobricks
- Databricks Docs — *AI governance with Unity AI Gateway* / *AI Gateway-enabled inference tables* — https://docs.databricks.com/aws/en/ai-gateway/
- Databricks Blog — *Introducing CustomerLake: The Agentic CDP embedded in Databricks* — https://www.databricks.com/blog/introducing-customerlake-agentic-cdp
- Databricks Newsroom — *Databricks Enters the Marketing Industry with CustomerLake* — https://www.databricks.com/company/newsroom/press-releases/databricks-enters-marketing-industry-customerlake-agentic-customer

**v1.4:** §7.1 deploy boundary; principle 12. Cut-over and Serving knobs in the mapping doc.  
**v1.3:** Domain Federation after the two-layer split; default retrieval isolation; people vs capability routing; scoped gates; contract registry on the control plane; principle 11.  
**v1.2:** Genie One native office embed; dual CI/CD; pointer to the mapping doc.
