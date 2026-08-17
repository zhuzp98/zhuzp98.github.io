# Databricks 企业级 Agent 落地映射与实施路径

| 项 | 内容 |
|----|------|
| 版本 | v1.4 |
| 日期 | 2026-08-17 |
| 作者 | Raphael |
| 关联文档 | 《Databricks 企业级 Agent 架构设计文档》（概念架构 v1.4） |
| 本文回答 | 概念架构里的每个模块，**用哪些 Databricks 组件实现**、**怎么搭**、**按什么顺序落地** |
| 相对 v1.2 | 补运行时与租户：工作区切法 A/B/C、Serving 三旋钮、域预算验收、Phase 门禁；版本号与概念架构对齐为 v1.4 |
| 相对 v1.1 | 补齐 Domain Federation 落地：Context Namespace、能力契约生命周期、发现模式降调、Phase 1 即打域标签、编排层注册表/跨域授权/域配额 |
| 相对 v1.0 | 同步 2026-07～08 产品状态：Unity AI Gateway GA、Managed MCP 归入 Gateway 控制面、Lakeflow Connect / UC Secrets 进度、Genie 家族命名、Pipeline 单元测试与 Genie Code Web Search |

> **阅读提示**：Databricks 的数据/AI 栈更新很快，本文所有组件都标注了**可用性状态**（GA = 已正式发布；Beta / Public Preview = 预览；Private Preview = 私有预览；Labs = Databricks Labs 社区项目，非官方 SLA）。凡标注 Preview / Beta / Labs 的组件，落地时都应准备**兜底方案**。状态以文末来源核实时点（约 **2026-08**）为准，正式立项前建议再核对一次官方文档。

---

## 1. 总览：模块 ↔ Databricks 落地件对照表

| 模块 | 核心 Databricks 落地件 | 可用性 |
|------|------------------------|--------|
| **① 动态信息输入** | Lakeflow Connect（100+ 源接入，含 SharePoint / Google Drive 等）、Auto Loader（对象存储增量）、Lakeflow Spark Declarative Pipelines（流+批 ETL）、Lakeflow Jobs（编排）；可选：**Genie Code Web Search**（公共 Web 实时检索） | Connect / Auto Loader / SDP / Jobs：**GA**；部分 Connect 源（如 OpenAI、PagerDuty）：**Beta**；Genie Code Web Search：**Beta**（2026-08） |
| **② 知识与记忆** | Unity Catalog（血缘/治理/**域 schema**）、Metric Views（集团级 + 域作用域）、Vector Search（**metadata Filter**）、Genie Ontology + OntoRank、Ontobricks；业务问答侧对接 **Genie Agents** | UC / Vector Search / Metric Views：**GA**；Genie Ontology 为 2026 新发布；Ontobricks：**Labs** |
| **③ AI 处理逻辑** | Lakeflow Jobs / Workflows、Mosaic AI Agent Framework、Agent Bricks、**Managed MCP（归 Unity AI Gateway）**、Model Serving、Unity AI Gateway；部门能力以 **UC Functions / MCP 契约**发布 | Agent Framework：**GA**（2026 初）；Unity AI Gateway **核心能力 GA**（2026-08-04）；Managed MCP ↔ Gateway 深度集成：**Beta**（2026-08-06）；部分高级 Service Policies / Agent Services：**Beta** |
| **④ 面向业务的执行策划** | 对内：Databricks Apps、Alerts、**Genie One Native Apps**、Lark-Meegle；对外：CustomerLake；**人员路由（组织图）+ 能力路由（契约发现，模式）** | Apps / Alerts / Genie One 办公插件：**GA**；CustomerLake：**Private Preview**；发现机制见 §2.4（非 Gateway 既成 API） |
| **⑤ 评估/反馈/学习** | MLflow 3（CLEARS + **域 Custom Judges**）、Inference Tables、Lakehouse Monitoring、DABs + Git、**Lakeflow Pipeline 单元测试** | MLflow / Inference Tables / Monitoring / DABs：**GA**；Pipeline Unit Testing：**Beta**（2026-07） |
| **编排 / 治理层** | Unity Catalog + **UC Secrets**、Unity AI Gateway、Lakeflow Jobs、MLflow 3、DABs + Git、系统表/预算；**契约注册表、跨域授权、域配额、工作区策略、Serving 上限** | UC / UC Secrets / Gateway 核心 / Jobs / MLflow / DABs / Serving：**GA**；Gateway 高级策略与 Agent Services、Managed MCP 深度绑定部分：**Beta** |

---

## 2. 逐模块落地件详解

### 2.1 模块① 动态信息输入

| 需求 | 落地件 | 说明 |
|------|--------|------|
| 接入 100+ 企业数据源（业务库、SaaS、外部源） | **Lakeflow Connect** | 高性能托管接入，覆盖业务前/后链路的多数结构化源。**2026-08**：SharePoint Connector、Google Drive Connector 已转 **GA**（非结构化文件 / SaaS 入湖可靠度显著提升）；OpenAI Connector、PagerDuty Connector 为 **Beta** |
| 对象存储文件的增量、幂等加载（埋点、链上导出等） | **Auto Loader** | incremental + idempotent，天然适合高频埋点 |
| 流式 + 批式统一 ETL，落 bronze 层 | **Lakeflow Spark Declarative Pipelines (SDP)** | 声明式（SQL/Python），核心概念：streaming tables、materialized views、flows、sinks |
| 管道编排、调度、依赖 | **Lakeflow Jobs** | 原生编排引擎，承载 cron / DAG |
| 非技术人员搭简单管道 | **Lakeflow Designer**（GA，2026-06） | 拖拽 + 自然语言的 no-code 管道，可让业务分析师自助接数据 |
| 公共 Web 实时知识补充（外部非结构化） | **Genie Code Web Search**（Beta，2026-08） | 面向数据 / AI 开发的全页代码 Agent（Genie Code）可检索公共 Web，作为企业内源之外的**实时外部 Context**补充通道；勿与托管数据源入湖混淆，适合时效性强、不宜预先入湖的公开信息 |

**落地要点**：
- **流式 vs 批式两条路径**在 SDP 里统一表达：埋点/交易走 streaming table，周期汇总走 materialized view。
- **结构化数据**直接落 Delta 表并在 Unity Catalog 打 tag；**非结构化 Context**（OKR、项目进度、竞品情报等）以文件 + 元数据入湖（SharePoint / Google Drive 等 GA Connector 可直连协作文档源），等待模块②做图化与嵌入。
- **外部实时检索**（Genie Code Web Search）与**托管入湖**（Connect / Auto Loader）分工：前者补"此刻公开可见"的缺口，后者补"受治理、可溯源"的企业资产；Agent 消费时二者需打不同置信度标签。

### 2.2 模块② 知识与记忆

| 需求 | 落地件 | 说明 |
|------|--------|------|
| 每张表/字段的"身份证"、全链路血缘、权限、tag | **Unity Catalog** | 知识可信度底座；也是后续"数据出口管控"的抓手 |
| bronze → silver 的治理与建模 | **SDP（medallion）** | 参考业务口径清洗、建中间层 |
| 指标的**单一事实来源** | **Metric Views** | 指标语义层，避免 Agent 拿到打架的口径 |
| 非结构化知识的语义检索 | **Vector Search** | 经验记忆、Context 文档的召回 |
| 企业 Context 图化 + 权威性排序 | **Genie Ontology + OntoRank** | 用权威分（含组织图信号）为知识排序；驱动复核优先级与模块④路由 |
| 把结构化表也物化成知识图谱节点 | **Ontobricks**（Labs） | 让结构化/非结构化在同一张图上被联合检索；Labs 项目，注意支持级别 |

**两类记忆的落地区分**：
- **领域知识 / 规则**（合规、SOP、指标口径）→ Metric Views + Unity Catalog 受治理对象，改动走审批。
- **经验记忆**（案例、反馈样本）→ Vector Search 索引，由模块⑤自动回写。

**复核策略落地**：
- 用 OntoRank 权威分排序，高权威/高风险知识人工精审，其余 AI 预筛 + 抽样。

**多部门 Context Namespace（域隔离）落地机制**（Phase 1 即启用，不要等到多 Agent 再补）：
- **表**：各部门结构化对象按 Unity Catalog `catalog.schema` 划界；行级敏感用 **ABAC / Row Filter / Column Mask**。
- **文件 / Volume**：用 schema 权限隔离，而不是把行级过滤套到文件上。
- **向量**：索引带 `domain` 等 metadata；检索默认加 **Filter Expression**（如 `domain = 'supply_chain' OR domain = 'corporate'`）。
- **指标**：集团级 Metric Views 放公共 schema；部门口径放域 schema，跨域只暴露契约化只读视图。
- **跨域检索**：编排层先做身份与授权（Gateway / UC 权限）；**授权通过后，由 Agent 检索调用注入跨域 Filter 组合**——Filter 在检索层，不在 Gateway 控制面「传入」。两域定义冲突时：集团公共定义优先，其次 OntoRank + 请求域优先级。


### 2.3 模块③ AI 处理逻辑

| 角色 | 落地件 | 说明 |
|------|--------|------|
| 确定性工作流（cron / 带依赖 DAG） | **Lakeflow Jobs / Workflows + SDP** | 把可标准化流程固化为 Job；进生产前可用 Lakeflow Editor 的 **Pipeline 单元测试**（Beta）校验逻辑 |
| 构建 Agent（复杂/自定义） | **Mosaic AI Agent Framework**（GA，2026 初） | 可用 LangGraph / CrewAI / Claude Agent SDK 等任意 harness；MLflow 记录、UC 注册、Model Serving 部署 |
| 构建 Agent（task-first、少运维） | **Agent Bricks** | serverless（经 Databricks Apps）部署，原生支持 MCP；偏**开发者 / 平台团队**的 task-first 构建面，与业务侧 Genie One、开发侧 Genie Code 互补 |
| 把 UC Functions / Genie Agents / Vector Search / DBSQL 作为**受治理工具**暴露给 Agent | **Managed MCP servers**（经 **Unity AI Gateway**） | **2026-08-06**：托管 MCP connectors（含供 Genie One / Genie Code 使用的连接器）已迁移并集成入 Unity AI Gateway（绑定部分为 **Beta**），集中化权限、访问日志与成本治理——工具接入不再散接，而归 Gateway **原生托管工具治理** |
| 部署 Agent / 模型 | **Model Serving** | 统一服务端点 |
| 统一 AI 控制面（路由、限流、成本、guardrails、trace） | **Unity AI Gateway** | **核心功能于 2026-08-04 正式 GA**（路由、限流、成本治理、统一控制面、inference tables）；部分高级 **Service Policies** 与 **Agent Services** 仍为 **Beta** |
| 记录员工 AI 问答 trace（识别自动化机会·路径三） | **Unity AI Gateway → Inference Tables** | 请求/响应自动落 Unity Catalog Delta 表 |
| Agent 生命周期、追踪、评估 | **MLflow 3** | GenAI-native 追踪与评估骨干 |

**Genie 产品矩阵（命名对齐，避免混称）**：

| 产品 | 受众 | 一句话 |
|------|------|--------|
| **Genie Agents**（原 Genie Spaces） | 业务 / 分析场景 | 受治理数据上的对话式分析 Agent（空间/工作区语义统一改称 Agents） |
| **Genie One** | 业务团队 | 全能协同同事型 Agent，可嵌入办公生态（见模块④） |
| **Genie Code** | 数据与 AI 开发 | 全页代码 Agent（含 Web Search Beta） |
| **Agent Bricks** | 平台 / 应用构建 | task-first 受管 Agent 构建与部署 |

**两个 AI 角色的落地**：
- **监督者**：SDP/Jobs 产出 output 与 sensor 指标 → MLflow Tracing + scorers 汇总、监控、发现异常。
- **决策顾问**：基于 **Genie Agents**（over Metric Views）+ Ontology 上下文做 OKR loop review，产出建议。

**"固化判据"怎么落地**：
- 用 Unity AI Gateway 的 inference table + MLflow 分析某类 AI 问答的**频率 × 稳定性 × 风险**，达标者从"探索性用法"晋升（promote）为 Lakeflow Job。这道闸门是模块③的核心工艺。

**能力契约（Capability Contract）生命周期**：

| 步骤 | 做法 |
|------|------|
| **定义** | 每个部门 Agent 的公开能力写成 **Unity Catalog Functions（UC Tools）或 Managed MCP Tool Definitions**，I/O 必须是强类型 **JSON Schema**（可与 Structured Outputs 对齐）。 |
| **发布** | 写入受治理 catalog（建议 `corp.agent_registry` 或域 schema 下的 `published_tools`），打标签如 `agent-capability`、`domain`、`version`、`risk_class`。 |
| **鉴权** | 谁可发现、谁可调用，走 UC 权限 + Gateway 身份；默认本域可调，跨域需授权。 |
| **发现** | 模块④按意图检索注册表（见 §2.4），不写死 Job / 端点名。 |
| **变更** | 兼容变更（加可选字段）域内发布；破坏性变更（改必填、改语义）走中央闸门，双跑至少一个版本窗口。 |
| **废弃** | 标记 `deprecated` + 截止日期；注册表仍可查到，调用方收到明确错误而非静默失败。 |

### 2.4 模块④ 面向业务的执行策划

**对内（送给员工/leader）**：

| 通道 | 落地件 | 适用 |
|------|--------|------|
| 邮件 / 正式留痕 | **Databricks Alerts / 通知** | 经营报告等低频正式传达 |
| 交互式部门小应用 | **Databricks Apps** | 需反馈 / 审批 / 下钻 |
| 办公软件内嵌（表格 + IM） | **Genie One Native Apps**（Google Sheets / Microsoft Excel / Slack / Teams，**2026-08 GA**） | 分析结论与结构化建议直接落入业务日常表格与 IM，无需离开现有办公软件；与 Apps 互补（后者适合定制交互，前者适合零迁移触达） |
| 落成项目管理任务 | **Lark / Meegle 集成**（自研：开放平台 API / webhook，可能需 Lark CLI） | 建议直接进 Meegle 工作流 |

**对外（触达客户）**：
- **CustomerLake（Agentic CDP，Private Preview）**：以 "infinity campaigns" 持续响应客户 context，把手动 trigger 的触达自动化。
- **兜底**：在 CustomerLake 可用前，用自建激活 Job（Model Serving 输出 → 触达渠道）替代。

**落地要点**：
- **路由复用**模块② Genie Ontology 的组织图，别重建；始终读最新组织结构防"送错人"。
- **执行分级**（全自动 / 人工审核 / 仅建议）用 Agent Framework 的 human-in-the-loop / approval 机制实现；高风险（资金、对外承诺）默认人工确认。
- **每条建议附血缘**：复用 Unity Catalog lineage，保证可解释、可核对。
- **通道优先序建议**：能落表格 / Slack·Teams 的先走 Genie One 插件（摩擦力最低）→ 需审批下钻走 Apps → 需项目跟踪走 Meegle → 正式存档走邮件。
- **人员路由 vs 能力路由**：找人复用 Ontology 组织图；找部门 Agent 走契约发现，二者不要写进同一张硬编码表。
- **动态发现（Agent Discovery，落地模式，产品名需立项前核对）**：跨部门调度时不写死路由。推荐实现顺序：① 查 UC 中带 `agent-capability` 标签的 Functions / MCP 工具（**当前最稳**）；② 若已开通 **Agent Services（Beta）**，再用其工具检索能力做意图匹配；③ Genie Agent / 相关 API 仅作辅助，**不要写成 Unity AI Gateway 已 GA 的「Tool Search」API**。发现失败则只送人、不自动调用对方 Agent。

### 2.5 模块⑤ 评估 / 反馈 / 学习

| 需求 | 落地件 | 说明 |
|------|--------|------|
| Agent/流程质量评估（开发 + 生产一致） | **MLflow 3 评估 + 监控**（built-in / custom LLM judges & scorers） | **CLEARS 六维**：Correctness、Latency、Execution、Adherence、Relevance、Safety |
| 全链路追踪、长期留存 | **MLflow Tracing → OTEL → Unity Catalog 表** | serverless 落表，治理化的可观测数据 |
| 自动化执行的业务数据反馈 | **Inference Tables** + CDP/CustomerLake 追踪 | 策略层快信号 |
| 数据/模型漂移监控 | **Lakehouse Monitoring** | 基础设施层持续信号 |
| 版本化、可回退的持续学习 | **Databricks Asset Bundles（DABs）+ Git folders** | CI/CD + VCS，是"学习"的安全带 |
| DAG / ETL 管道进生产前与硬学习改动时的自动化测试 | **Lakeflow Pipeline 单元测试**（Beta，2026-07） | Lakeflow Editor 可对 Spark Declarative Pipelines（SDP）编写 Python/SQL 单元测试，用 mock 数据验证 Pipeline / Auto CDC / Expectations；为模块③确定性工作流与模块⑤硬学习改动提供**数据管道侧**安全带，与模型侧 MLflow 评估形成双重 CI/CD |

**落地要点**：
- **三类反馈 → 三层**：业务数据（MLflow/Inference Tables）→ 策略层；员工任务反馈（Meegle 回执 + 文档）→ 知识/流程层；trace 监控（Lakehouse Monitoring + DABs）→ 基础设施层。
- **双重 CI/CD**：模型/Agent 改动过 MLflow 评估门禁；SDP / Job 管道改动过 Pipeline 单元测试门禁，再经 DABs 发布。硬学习（改 Job / 管道）不得跳过测试绿通。
- **回滚判据**：MLflow 监控发现漂移/劣化 → 经 DABs + Git 回退到上一个良好版本，回滚是一个动作而非事故。
- **解耦的异构评估（Domain Evaluation）**：CLEARS 与域内 KPI **正交**——前者是质量/安全/成本，后者是业务结果，不能互相替代。
  - **部门局部评估**：各部门基于 **MLflow 3 Custom Judges** 编写域内指标（如风控「拒贷误伤率」，营销「ROI 提升度」），在部门专属 DABs 管道中跑评估，驱动域内软/硬学习。
  - **中央治理评估**：Unity AI Gateway 统一在 **Inference Tables** 上挂载全局 CLEARS（尤其 Safety、Latency、Cost）与漂移告警，只拦截全局合规与模型风险。
- **归因纪律**：对外触达用 **holdout / A-B**（CDP 通常原生支持受众 holdout），确保学到的是**因果**增量。
- **回写路径**：软学习 → 更新 Vector Search 索引（经验记忆）；硬学习 → 经晋升判据 + 单元测试绿通改 Lakeflow Job / Agent 版本。

### 2.6 编排 / 治理层（神经中枢）

| 职责 | 落地件 | 说明 |
|------|--------|------|
| 编排、触发、依赖、重试、回退 | **Lakeflow Jobs / Workflows** | 确定性调度骨干 |
| 权限、血缘、数据出口管控（数据留云上、只让结论流出） | **Unity Catalog** | 数据与对象治理底座 |
| Agent / 工具调用的 API Key 与凭据治理 | **Unity Catalog Secrets（UC Secrets，GA）** | 在 UC 中直接管理与治理 Agent 调用凭据，权限/审计与数据资产同一控制面——安全与权限治理的基石 |
| 模型/Agent 流量的统一控制面：路由、限流、guardrails、日志、成本 | **Unity AI Gateway** | 核心功能 **GA**（2026-08-04）；高级 Service Policies / Agent Services 部分仍为 **Beta** |
| 托管工具的权限、访问日志与成本 | **Managed MCP servers ⊂ Unity AI Gateway 控制面** | 深度绑定 **Beta**（2026-08-06）；含 Genie One / Genie Code 所用托管 MCP connectors |
| 全链路可观测性 | **MLflow 3** | 追踪、评估、监控 |
| CI/CD + 版本控制（模型 + 管道） | **Databricks Asset Bundles + Git** + **Pipeline 单元测试（Beta）** | 模型侧与数据管道侧双重门禁 |
| 成本与配额 | **系统表（system tables）+ 预算/告警** | 按 `domain` / `agent_id` / `env` 打标签；超阈先经 Gateway **限流**再告警，不先停治理面 |
| 工作区 / 租户 | **默认：单 Workspace + 一域一 Catalog**（切法见 §4 决策 7） | Catalog 管看见什么；Workspace 管故障域与账单。升 B 时契约注册表仍留中心 Catalog |
| Serving 与作业并发 | **Model Serving 配额 + Cluster Policy / Serverless 预算 + Gateway 按身份限流** | 三旋钮见本节下文；跨域禁止无上限扇出 |
| 契约注册与发现 | **UC Functions / MCP 工具 + 标签（`agent-capability` / `domain` / `version`）** | 发布、鉴权、废弃见 §2.3；发现实现见 §2.4（模式，非 Gateway GA API） |
| 跨域授权 | **UC 权限 + Gateway 身份 / Service Policies（高级策略部分 Beta）** | 先授权，再由检索层注入 Filter；授权失败不得联合检索 |
| 域隔离策略 | **catalog.schema + Vector Search Filter + 域 Metric Views** | Phase 1 即落地，Phase 4 只增加跨域调用 |

> 数据安全副产品正落在这一层：Unity Catalog 管权限、出口与 **Secrets** + Unity AI Gateway 管 AI 流量与托管工具，共同保证员工在"信息最小必要"下工作。联邦不改变这条原则：跨域只流出契约化结论，不默认打开对方原始 Context。

**运行时三旋钮（不写容量公式，只锁定执行点）**：

1. **Model Serving / Agent 端点**：一域一 endpoint（或一契约一 endpoint），设 **provisioned concurrency 上限**；禁止无上限共享端点。对内「仅建议」可 scale-to-zero；审批流与对外触达保留最小 provisioned，避免冷启动误事。跨域调用经 Gateway **按调用方身份限流**。
2. **Lakeflow Jobs / SDP**：域级 Cluster Policy 或 Serverless 预算；硬学习发布不得挤占试点域生产队列。Pipeline 单元测试留在 CI，不占生产并发。
3. **检索与 SQL**：Vector Search 与 DBSQL 仓库按域分队列/仓，避免全公司共一个 warehouse。Gateway 限流就是域配额的执行点，不另建限流服务。

---

## 3. 分阶段实施路径（爬 → 走 → 跑）

按"先地基、再知识、后单 Agent 闭环、最后多 Agent 扩展"推进，每阶段有明确产出与退出标准。

### Phase 0 — 治理底座（地基）
- **目标**：数据可信、可治理、可追溯；凭据与出口受控。
- **关键组件**：Unity Catalog（权限/血缘/tag）+ **UC Secrets（GA）**、Lakeflow Connect + Auto Loader + SDP（入湖到 bronze/silver）、Unity AI Gateway 核心面就位（为后续 Agent 流量预留）。
- **产出**：受治理的数据底座 + 全链路血缘 + 凭据入 UC 治理；**书面锁定工作区切法 A**（单 Workspace + 一域一 Catalog，见 §4 决策 7）。
- **退出标准**：核心业务数据已入湖、有血缘、权限收敛（数据出口开始受控）；Agent 相关密钥不落本地/非治理配置；租户策略已记录（默认 A，升级 B 的触发条件写清）。

### Phase 1 — 知识层（把数据变知识）
- **目标**：可信知识 + 可检索 Context；**域标签与 schema 隔离从第一天就有**，避免 Phase 4 在脏索引上补隔离。
- **关键组件**：Metric Views（集团公共 schema + 试点域 schema）、Vector Search（索引必带 `domain` metadata，默认 Filter）、Genie Ontology（权威分/组织图）；业务问答侧用 **Genie Agents** 验收。
- **产出**：指标口径统一 + 企业 Context 可被 Agent 检索；至少一套 `corporate` + 一个业务 `domain` 的 Namespace 约定。
- **退出标准**：关键指标有唯一定义；Genie Agents 能基于治理数据正确回答业务问题；本域检索不会命中无关部门文档。

### Phase 2 — 首个 Agent 单元（窄场景 pilot）
- **目标**：跑通一个高价值窄场景的领域 Agent（先只做"仅建议"档）。
- **关键组件**：Mosaic AI Agent Framework / Agent Bricks（构建）、Managed MCP via Unity AI Gateway（受治理工具）、Model Serving（部署，**带 concurrency 上限**）、Unity AI Gateway（路由/限流/成本/trace）；试点部门将其公开能力按 §2.3 **先注册进 UC**（即使暂时只有本域调用）。
- **产出**：一个能产出建议的领域 Agent（如"转化率异常归因与改进建议"）。
- **退出标准**：Agent 在真实数据上稳定产出可用建议，全程 trace 可查；工具调用经 Gateway/MCP，无散接绕过；该 Agent 至少有一条带 Schema 的公开契约（可供后续被发现）；Serving 端点有上限且账单带 `domain` 标签。

### Phase 3 — 闭合闭环（让它会学习）
- **目标**：③→④→⑤ 闭环，Agent 能自我改进。
- **关键组件**：MLflow 3（评估/监控：全局 CLEARS + **该域 Custom Judge**）、Inference Tables、Lakehouse Monitoring、DABs + Git（CI/CD + 回滚）、**Lakeflow Pipeline 单元测试（Beta）**、holdout/A-B。
- **产出**：反馈回写模块②③；晋升/回滚判据生效；硬学习改管道前强制单测绿通。
- **退出标准**：至少一次"反馈 → 改进 → 效果验证（带 holdout）"完整闭环跑通；模型门禁与管道门禁均曾真实触发并放行/拦截过。

### Phase 4 — 执行落地与多 Agent 扩展（联邦化演进）
- **目标**：接通对内/对外执行，并以 **Domain Federation** 把已隔离的域连成网络——本阶段做**发现与跨域调用**，而不是从零补隔离。
- **关键组件**：Databricks Apps + Alerts + Genie One Native Apps（对内）、CustomerLake（对外，Private Preview，带兜底）、**UC 契约注册表 + 发现模式（§2.4）**、分部门 MLflow Evaluators、域配额与 Gateway 限流。
- **产出**：各部门拥有自治的 Context Namespace 与评估规则；跨部门协同通过 MCP/UC 契约动态完成；宏观层「多 Agent 网络」成形。
- **退出标准**：至少 3 个独立部门的 Agent 在各自 Context 隔离下稳定运行，且能通过注册表发现机制成功完成至少一次**跨部门能力调用与结果回流**（发现失败有降级：只送人）；**人为把试点域 QPS 打到配额后，其他域 Serving 仍可用**（noisy-neighbor 验收）。仍停留在切法 A，除非已触发 §4 决策 7 的升级条件。

---

## 4. 关键落地决策与取舍

1. **Agent Bricks vs Agent Framework vs Genie 家族**：task-first、少运维的场景用 **Agent Bricks**；需要复杂自定义编排的用 **Agent Framework**（可挂 Claude Agent SDK 等 harness）；业务自然语言分析用 **Genie Agents**，办公内嵌协同用 **Genie One**，开发侧辅助用 **Genie Code**。四者可混用，勿把 Genie One 与 Agent Bricks 当成同一产品。
2. **工具接入统一走 Managed MCP（经 Unity AI Gateway）**：把 UC Functions / Genie Agents / Vector Search / DBSQL 作为受治理工具暴露，避免 Agent 散接数据、绕过治理；权限、访问日志与成本计入 Gateway 控制面。
3. **Preview / Beta 组件必设兜底**：CustomerLake 仍为 Private Preview；Managed MCP ↔ Gateway 深度绑定、Pipeline 单元测试、Genie Code Web Search、部分 Connect 源与 Gateway 高级策略为 Beta——规划纳入，但关键路径准备降级方案（自建激活 Job、本地/CI 单测脚本、关闭 Web 检索通道等）。
4. **固化判据要有机制而非拍脑袋**：用 Gateway inference table + MLflow 量化频率/稳定性/风险，作为 promote 到 Job 的客观闸门；Job/SDP 改动再过 Pipeline 单元测试门禁。
5. **治理先行**：数据安全副产品依赖 Unity Catalog + **UC Secrets** + AI Gateway 先就位；治理不到位，"数据留云上、只让结论流出"就不成立。
6. **中央 vs 联邦**：Secrets、Gateway 身份、出口、全局 Safety/CLEARS、共享 Metric Views 保持中央；Context 索引、部门口径、Custom Judges、域内 Job 下放到域。发现用 UC 标签注册表，不把未 GA 的 Gateway Tool Search 当关键路径。
7. **工作区切法（租户）**：逻辑隔离永远用 Catalog；Workspace 只在算力/账单/合规需要时再拆。默认与升级条件如下。

| 模式 | 何时用 | 优点 | 代价 |
|------|--------|------|------|
| **A. 单 Workspace + 一域一 Catalog**（默认） | Phase 0–3；域数量不大、平台团队小、合规未强制分账号 | 血缘、契约、发现最简单，一套 Gateway | 计算可能争抢，需靠配额与三旋钮兜住 |
| **B. 中心 Workspace（治理）+ 域 Workspace（运行）** | 单域 Jobs/Serving 已挤占别人，或审计要求分账 | 故障域分离，预算可按 workspace 打 | 跨空间身份、DABs `target`、发现多一跳；契约注册表仍放中心 Catalog `corp.agent_registry` |
| **C. 一域一 Workspace** | 强监管、并购后独立 IT、或域间零信任被点名 | 爆炸半径最小 | 注册表/Ontology/Gateway 都要中心副本，运维陡升——**默认不选** |

升 B 时：同一 UC metastore + SCIM；域侧只放实现，公开契约仍在中心；DABs 用 `gov` / `domain_x` 两套 target，禁止每域一份互不兼容的 Bundle 结构。切 Workspace **不**重做 Context Namespace。

---

## 5. 风险与依赖

- **Private Preview 依赖**：CustomerLake 的可用时点不完全可控 → 兜底方案必备。
- **Beta 面扩大但仍需兜底**：Unity AI Gateway 核心已 GA，但高级 Service Policies / Agent Services、Managed MCP 深度绑定、Pipeline 单元测试、Genie Code Web Search 等仍为 Beta，不可当作零风险生产依赖。
- **发现能力产品化程度**：跨部门 Tool Search 若依赖 Agent Services（Beta）或自建注册表查询；立项前核对官方 API，失败路径是「只送人」。
- **契约蔓延与破坏性变更**：部门私自改 Schema 会导致调用方静默出错 → 必须走 §2.3 版本与双跑。
- **跨域 Filter 过宽**：授权一旦变成「全打开」，Namespace 形同虚设 → 跨域只加白名单 domain，并记录审计。
- **noisy neighbor**：共享 Serving / warehouse 无上限时，一域峰值拖死全家 → Phase 4 必须做配额打满演练；跨域调用禁止广播扇出。
- **过早切多 Workspace**：在契约注册表未中心化前升 B/C，联邦会碎成 N 套发现 → 先 A + 配额，触发 §4 决策 7 再升。
- **自研集成成本**：Lark / Meegle 集成需自建（开放平台 API / webhook / Lark CLI），排期要留出开发量；Genie One 原生办公插件可覆盖部分通道，减少自研面，但不能替代项目管理系统集成。
- **归因纪律要前置**：holdout / A-B 必须在 Phase 3 设计阶段就规划，事后补做难度大。
- **Labs 组件支持级别**：Ontobricks 为 Databricks Labs 项目，无官方 SLA，关键路径上需评估替代或自建。
- **组件状态时效**：本文状态以约 **2026-08** 核实时点为准，正式立项前请再核对官方文档最新可用性。

---

## 6. 理论映射：文章理念 ↔ Databricks 能力（含留白）

> 本节把《高维逻辑的拓扑连线与现实锚点》（企业级 Agent 的贝叶斯本质与工程治理）中的核心主张，对到具体 Databricks 能力，并**诚实标出平台留白**——即文章最锋利、但需要在推理层自建的部分。本节回答的不是"用什么搭"，而是"**为什么这么治理**"，为整套架构补上理论依据。

### 6.1 对照总表

| 文章理念 | Databricks 落地件 | 契合度 |
|----------|-------------------|--------|
| 知识图谱=显式原子事实（符号/离散）vs 向量嵌入=连续语义相似度（联结/概率），统一于 Context | **Genie Ontology / OntoRank**（符号侧、权威知识图谱）+ **AI Search**（联结侧、向量相似度） | ⭐ 突出 |
| Context 提纯：混合检索（Keyword + Vector）+ 交叉熵重排（Reranking） | **AI Search** hybrid（全文 + 向量 ANN、RRF 融合）+ **cross-encoder 重排** | ⭐ 突出 |
| 语义质量红线 / Gatekeeper：给知识按权威性打分、可信才入库 | **OntoRank 权威分**（原生量化"哪条定义可信"） | ⭐ 突出 |
| 强类型逻辑围栏：JSON Schema / Protobuf 约束解码 | **Structured Outputs**（由 constrained decoding 驱动，保证符合 schema） | ✅ 命中 |
| 确定性骨架（FSM / LangGraph 状态图）+ LLM 降格为局部意图解析器与条件路由 | **Lakeflow Jobs / Workflows**（确定性 DAG）+ **Agent Framework**（支持 LangGraph） | ✅ 命中 |
| 开放式规划的"现实实验闭环"：假说→执行→客观新证据→贝叶斯更新→收敛 | **MLflow 3 评估/追踪/监控 + CLEARS 六维** + holdout / A-B | ✅ 命中 |
| Context 角色分化（System Prompt / RAG / Observation / Memory） | **Agent Framework** 上下文管理 + **Managed MCP** 工具（检索/工具返回）+ Memory | ✅ 命中 |
| Logit 级 / Softmax 前不确定性监控 + 首 token 前"架构级硬拦截" | **端点不可实施**：托管 FM API / 外部模型（经 Gateway）只暴露**输出 token 的 logprobs**，不给完整 pre-softmax logit 向量，且生成在服务端——无法在首 token 前插拦截 | 🚫 平台受限 |
| Dirichlet UQ、aleatoric/epistemic 拆分、ECE 校准、Credal Set | **端点不可实施**：需改模型头 / 拿全量 logits，托管与外部端点均不开放；仅自托管开源模型 + 自定义 serving 才可能 | 🚫 平台受限 |
| Renormalization Bias 的显式处理（UNK/OOD 泄洪、概率流失补偿） | **部分**：constrained decoding 是平台能力，但 schema 泄洪设计是架构智慧 | ⚠️ 留白·需自建 |

### 6.2 最突出的一处：符号 × 联结的知识统一

文章第一节的核心区分——KG 三元组是维特根斯坦意义上的"原子事实图像"（离散/符号/确定），向量嵌入是"分布式语义相似度"（连续/联结/概率）——Databricks 恰好把两者做成**互补的一等组件**：

- **Genie Ontology / OntoRank** 提供权威的显式知识图谱（符号侧）。其"权威分"直接把文章第五节的"语义质量红线 / Gatekeeper Agent"变成**平台原生的量化信号**——它本就是给知识节点按权威性打分、决定"哪条定义可信"。
- **AI Search** 提供连续语义相似度（联结侧），原生支持 hybrid 检索（全文 + 向量 ANN 并行、RRF 融合）与 cross-encoder 重排，正对应文章的"混合检索 + 交叉熵重排提纯"。

也就是说，文章里"符号 vs 联结统一于 Context"的抽象，在 Databricks 有对应的一等组件，而非要自己拼装。这也正是本落地文档模块② 的理论落点。

### 6.3 平台硬约束：logit 级机制在托管 / 外部端点上不可实施

文章最锋利的两个主张——**logit 级不确定性拦截**与 **Dirichlet / UQ 校准**——在 Databricks 的实际端点上**无法落地**，原因是接口层面的硬约束，而非"没做"：

- **托管 Foundation Model API 与经 AI Gateway 调用的外部模型（Claude、GPT 等）只暴露输出 token 的 `logprobs`**——即已采样 token 的对数概率，**不是**全词表的 pre-softmax logit 向量。文章要的"原始 logit 绝对强度、方差、epistemic 拆分"拿不到。
- **生成过程在服务端**，调用方无法在"自回归首个 token 产生前"插入架构级硬拦截。
- Dirichlet 建模、ECE 校准、Credal Set 都需要**改模型头或拿全量 logits**，托管与外部端点都不开放这个层级。

**唯一可能的路径**是自托管开源权重模型（GPU + 自定义 pyfunc serving），此时你拥有 forward pass，可加 evidential/Dirichlet 头、算 epistemic uncertainty、在 emit 前硬拒绝。但代价是：放弃托管 FM API 的便利、自担模型运维与 UQ 研究工作，且**对专有模型（Claude/GPT）完全不适用**。对于本架构主要依赖托管 + 外部模型的现实，这两项应视为**超出实施范围**。

### 6.4 可落地的替代：用行为级手段逼近文章的"OOD 即沉默"意图

虽然 logit 级机制做不了，但文章的**治理意图**（分布外 / 不确定时硬拒绝、保持沉默）可以用端点确实提供的能力逼近——只是作用在**输出级 / 行为级**，而非模型内部表征：

- **检索置信度门控（最实用）**：若 AI Search / Genie 召回的最高相似度低于阈值，直接判定"围栏外 = OOD"并硬拒绝。这不需要任何 logits，且直接对应文章"边界之外即不可言说"。
- **输出 `logprobs` 启发式**：用已采样 token 的 logprobs 做序列级置信度估计，低置信触发复核 / 拒绝（粗粒度，但可用）。
- **语义熵 / 自一致性**：对同一输入采样 N 次，测输出分歧度，近似 epistemic uncertainty——无需模型内部信息。
- **独立 OOD / 不确定性分类器**：作为 Agent 前置路由步，判定是否放行。
- **AI Gateway guardrails + CLEARS Safety**：输出层最后一道拦截。

> **一句话**：文章的 logit 级内核在 Databricks 端点上**不可实施**（除非自托管开源模型，成本高且不覆盖专有模型）；但其"OOD 即沉默"的治理意图，可用**检索置信度门控 + 输出 logprobs + 语义熵**在行为级实现。Databricks 提供的是产品级的"围栏与骨架"，模型内部的"概率驯服"要么自托管自建、要么退而用行为级代理。

---

## 附录：来源

以下关于 Databricks 产品与算法的说明基于公开文档与报道核实（组件状态复核时点约 **2026-08**）：

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
- Databricks Docs — *AI Search retrieval quality guide（hybrid search & reranking）* — https://docs.databricks.com/aws/en/vector-search/vector-search-retrieval-quality
- Databricks Docs — *Structured outputs on Databricks（constrained decoding）* — https://docs.databricks.com/aws/en/machine-learning/model-serving/structured-outputs
- Raphael Zhu — *高维逻辑的拓扑连线与现实锚点* — https://zhuzp98.github.io/files/AI%20Thoughts/Enterprise_AI_thoughts.html

**v1.4 增量**：版本号与概念架构对齐为 v1.4。内容：工作区切法 A/B/C（默认 A）；Serving / Jobs / 检索三旋钮；域预算标签与 noisy-neighbor 验收；Phase 0/2/4 门禁。

**v1.2 增量**：Domain Federation 落地闭环（契约生命周期、发现降调为 UC 注册表模式、Phase 1 即域隔离、编排层补注册表/跨域授权/域配额）；风险补契约蔓延与 Filter 过宽。

**v1.1 增量同步说明（产品公告时点）**：Unity AI Gateway 核心 GA（2026-08-04）；Managed MCP connectors 归入 Unity AI Gateway 控制面（Beta，2026-08-06）；Lakeflow Connect 的 SharePoint / Google Drive Connector GA，OpenAI / PagerDuty Connector Beta（2026-08）；UC Secrets GA；Genie Spaces 统一更名为 Genie Agents，并与 Genie One / Genie Code 形成矩阵；Lakeflow Pipeline 单元测试 Beta（2026-07）；Genie Code Web Search Beta（2026-08）；Genie One Excel / Google Sheets 原生插件发布（2026-08）。正式立项前请以官方文档为准复核。
