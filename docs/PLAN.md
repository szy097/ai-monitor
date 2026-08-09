# AI Monitor：成熟 Agent 驱动的实施与运行计划

> 状态：Draft v0.3<br>
> 日期：2026-08-09<br>
> 当前仓库：已初始化 Git，`main` 跟踪 `origin/main`，尚无产品代码<br>
> 实施主体：AI Agent；不安排人类工程师执行开发任务

## 1. 核心决策

本项目不按人类团队的“人数 × 周期”排期，而按依赖图、机器可验证任务和发布门禁推进。

执行原则只有四条：

1. **Git 中的规格与测试是真相**：架构约束、任务契约、Golden 样本和验收命令必须版本化。
2. **单栈、双实例**：Hermes 同时负责 AI 实施编排与生产 Gateway/Cron/Skills/消息渠道；实施实例和生产实例在进程、状态、权限、密钥和 profile 上完全隔离。
3. **确定性 Core 掌握业务真相**：采集 checkpoint、文章、去重、事件、digest 窗口、不可变 payload 和 delivery 状态必须由受测试的代码与数据库维护，不能只存在于 Agent 上下文；Hermes 只保存运行 attempt、消息投递证据和对话上下文。
4. **确定性验收决定完成**：worker 的总结和 Agent judge 都不能单独证明任务完成。

默认采用 [Nous Research Hermes Agent v0.20.0 / v2026.8.3](https://github.com/NousResearch/hermes-agent/releases/tag/v2026.8.3) 同时作为实现控制平面与生产 Agent Runtime。项目不跟随 `main`、`latest` 或浮动镜像标签；H0 验证其实施能力，R0 独立验证其生产 Cron、Gateway、Feishu、MCP、恢复和安全能力。G0 必须解析并记录上游 `nousresearch/hermes-agent:v2026.8.3` manifest digest，以及 dev/prod 两个派生镜像各自的不可变 digest；启动时不自动升级或动态安装组件。

AI Monitor 生产环境依赖 Hermes Gateway、Cron、Skills、模型调用和飞书渠道，但业务数据不写入 Agent Memory，幂等和业务状态不依赖自然语言推理。Hermes 与 `ai-monitor-core` 之间只通过有 Schema、权限边界和幂等键的固定 CLI/MCP 工具交互。`hermes-dev` 与 `hermes-prod` 使用不同容器或 OS 用户、不同 `HERMES_HOME`、profiles、sessions、Cron state 和 secrets；生产配置不启用、不暴露实施 toolset，若 hardened prod image 能实测剥离这些组件则进一步移除。

若 Hermes 的实现驱动能力不合格，可改用 Codex 或 Claude Code 执行同一任务契约。若 Hermes 的生产运行能力不合格，R0 必须进入 blocked 并新增 Runtime ADR；OpenClaw 是候选 B，但需建立独立适配分支并重新通过 R0，不能声称透明切换。`ai-monitor-core` 的领域模型和测试保持可移植，生产 Runtime 适配器允许变化。

## 2. 产品目标与 MVP 边界

### 2.1 目标

构建一个面向中文 AI 从业者的单用户资讯工具，自动完成：

- 获取可信 AI 最新资讯；
- 标准化标题、来源、作者与时间；
- 过滤无关内容并合并重复事件；
- 生成有证据约束的中文摘要和分类；
- 通过可替换的消息入口消费；当前工作假设是飞书私聊承载推送、查询和反馈，最终角色见 2.4；
- 对采集失败、来源失效和通知失败提供可观测状态。

首版强调“可信、及时、少重复”，不追求全网覆盖。

### 2.2 MVP 范围

- 8–12 个官方 RSS/Atom 来源；
- arXiv 指定分类；
- GitHub Releases 仓库白名单；
- Hacker News 新帖与热帖信号；
- 默认每 15 分钟增量采集，arXiv 按日采集；
- URL 规范化、精确去重、基础事件聚类；
- 中文摘要、主题、实体、重要度和证据引用；
- 飞书私聊查询、来源/主题筛选、追问和显式反馈；
- 通过 Hermes Agent 生成中文文本/富文本日报，并由 Hermes Gateway 投递；
- Hermes Cron 驱动定时采集、处理、日报和业务健康巡检；
- Docker Compose 一条命令启动固定版本 Hermes Runtime；镜像内安装 `ai-monitor-core`，分离挂载 Hermes state、Core DB 和 secret path。

### 2.3 明确不做

- 通用网页爬虫、登录墙、验证码绕过；
- 全量社交媒体抓取；
- 多租户、复杂权限和原生移动 App；
- 训练个性化推荐模型；
- Kafka、Kubernetes、微服务等超前基础设施；
- 首版自研完整 Web UI、聊天 UI、调度器或通知渠道；
- 让 LLM 直接执行文章 upsert、去重判定落库、checkpoint 推进和投递幂等。

### 2.4 飞书角色：待确认的产品决策

当前计划采用一个可逆的工作假设：**飞书是 AI Monitor 的主消费入口，不是业务数据库或 Hermes 运维控制台**。它负责主动推送、自然语言查询和用户反馈；文章库、检索索引、订阅偏好、已生成日报和投递状态仍由 Core 保存。首版不把飞书文档、知识库或多维表格当数据库，也不在飞书里复制工作流状态。

候选定位：

| 方案 | 飞书承担的职责 | 代价 | 当前建议 |
|---|---|---|---|
| A. 仅通知 | 日报、突发提醒和失败告警 | 最省事，但无法追问或反馈 | 可作降级模式 |
| B. 通知 + 对话入口 | A + 查询、展开、筛选、偏好反馈 | 需要 allowlist、命令边界和会话设计 | **MVP 工作假设** |
| C. 主工作台 | B + 文档/多维表格/团队协作流程 | 状态重复、权限复杂、平台耦合明显 | MVP 不采用 |

MVP 交互定义为：每天 08:30 向单用户 DM 投递 Markdown 富文本日报；每条都带稳定 `digest_id/article_id`、原文和证据链接。用户可以问“最近 24 小时最重要的 5 条”“只看开源模型”“展开 A-123”“为什么这两条被合并”，也可以提交少量可回滚反馈；只有 Core 工具写入成功后，Agent 才能确认偏好已保存。Cron 日报默认 fire-and-forget，后续追问必须按稳定 ID 重新查询 Core，不能依赖会话记得日报。

首版不从飞书管理来源、Cron、模型、skills、更新或部署，不接受附件作为资讯输入，也不开放任意 URL、shell、文件或浏览器工具。默认用 Hermes 已文档化的 Markdown post 与纯文本回退；自定义交互卡片只有在确定性 renderer 和 R0 端到端验证后再启用。若选择 A，Hermes 没有文档化的 outbound-only 开关，必须取消入站事件订阅或在分发层硬拒绝，不能只靠 prompt。

工作假设还包含“单用户、私聊优先”；若目标改成团队群播、共同编辑或审批，必须先补一份产品 ADR，明确受众、权限、消息噪音和共享偏好的语义。替换为 Telegram、邮件或 Web UI 时，不应修改 Core 领域模型。

## 3. 三层架构

```text
实现控制平面

Git 规格 / AGENTS.md / 任务契约 / Golden Evals
   ↓
Hermes Kanban → Profiles → Worktrees → Verifier → Integrator
   ↓
持续保持绿色的主分支


生产 Agent Runtime

用户 ↔ 飞书 ↔ Hermes Prod Gateway
                    ├── Cron：定时任务、执行 ledger 和结果投递
                    ├── no-agent scripts：确定性采集、恢复和巡检
                    ├── Skills：AI 资讯工作流程
                    ├── Session/Memory：对话和轻量上下文
                    ├── Tool policy：限制 shell、文件与控制面
                    └── MCP：受约束调用 ai-monitor-core
                                      ↓

确定性业务 Core

来源适配器 → 标准化 → checkpoint → 去重/事件 → SQLite
                                                ↑        ↓
                          lease / schema / idempotency   查询、待处理队列、digest ledger
```

### 3.1 实现控制平面

采用 Hermes 的成熟能力，不自行开发 Agent 编排器：

- **Durable Kanban**：任务、依赖、attempt、日志和 handoff 持久化，跨会话可恢复；
- **Profiles**：每个角色拥有独立身份、配置、记忆和会话；并发进程不得共享 profile；
- **Worktrees**：每张实现卡在独立分支和工作目录执行，避免并发互相覆盖；
- **Dispatcher**：回收 stale/crashed worker，控制并发并执行有限重试；
- **Goal-mode card**：仅用于确实需要多轮迭代的实现卡；卡片正文写明验收合同；
- **delegate_task**：只用于短期、进程内、无需跨重启保留的研究或审查。

Hermes Kanban 当前是单机 SQLite 控制面，符合本项目首版需要；不将其规划为跨主机编排系统。

### 3.2 生产 Agent Runtime

Hermes 生产实例复用以下成熟能力；准确行为必须在 R0 以固定 tag 的文档、CLI 和故障注入冻结，不能用当前在线文档替代版本契约：

- **Gateway + Feishu**：常驻消息入口、用户/群聊 allowlist 和飞书 WebSocket 连接；无需公共 webhook，SDK 负责断线重连，入站事件 ID 持久去重 24 小时；
- **Cron**：周期/单次任务、暂停/恢复、手动触发、执行历史和投递路由；定义保存在 `jobs.json`，执行 ledger 保存在 `executions.db`；
- **no-agent jobs**：无需推理的采集、对账和健康检查运行版本化脚本，不产生模型调用；脚本只能来自 `$HERMES_HOME/scripts`，使用净化后的环境；
- **fresh agent sessions**：事实抽取、事件摘要与日报在新会话中运行，固定模型/provider、attached skills、timeout 和最小工具面；
- **preflight 与 model drift guard**：在调用模型前验证 provider key、skill、投递目标和固定模型；配置漂移时 fail closed；
- **provider fallback/credential rotation**：只允许版本化的批准集合，不能静默切到未知模型；
- **Gateway delivery ledger**：固定版本按 at-least-once 语义恢复外发，最多重试 3 次、窗口 24 小时，已投递记录 7 天后清理；崩溃恢复可能重复，并在补投消息外层添加文档化的 `Recovered reply` 标记；
- **Skills/MCP**：把选题标准、摘要格式和冲突处理版本化；Core MCP 关闭 sampling、prompts 和 resources，只用 `tools.include` 正向白名单暴露最小 Schema 工具，`exclude` 不作为安全边界；
- **Memory/Session**：只保存对话上下文和轻量经验，不保存文章、checkpoint、偏好、digest 或投递真相。

Hermes 每分钟调度 tick 有锁，避免同一批次双启；但 Gateway 在任务运行中崩溃时，该 Cron attempt 会被标记为 `unknown`，且不会由 Cron 自动重放。这是计划接受并显式补偿的语义：ingest 在下一 tick 按 checkpoint 幂等恢复，enrichment 等 lease TTL，digest 在进入 `sending` 前可按同一 window 重跑。payload 一旦进入 Gateway delivery ledger，则由 Hermes 在固定窗口内有限补投；Core 对 `sending`/after-send unknown 只对账和告警，不再另行提交消息。任何任务都不得假设 exactly-once。

Hermes 的 Cron execution ledger、Gateway delivery ledger 与日志是短期运行证据；飞书 API 返回的 `message_id` 可作为 send path accepted 证据，但不代表用户已读。关联摘要和哈希必须每日导出到 Core audit 表或 CI artifact，保留至少 90 天。项目只补充来源新鲜度、pending 队列、摘要失败率、digest 延迟和投递异常等业务指标，不另造通用 Agent 调度器。

生产任务拓扑：

| Job | 类型 | 默认频率 | 职责 |
|---|---|---|---|
| `ingest-due-sources` | no-agent | 每 15 分钟 | 由只读脚本调用固定 Core CLI 获取到期来源，成功静默，失败告警 |
| `enrich-pending-items` | fresh agent | 每 15 分钟 | 通过 MCP lease 待处理条目，抽取事实、分类、聚类建议并结构化提交 |
| `prepare-daily-ai-digest` | fresh agent | 每日 08:30 `Asia/Shanghai` | 查询前一窗口候选，将不可变中文 payload 提交到 Core；只投递到 local audit，不直接发飞书 |
| `deliver-ready-digest` | no-agent | 每 2 分钟 | 从 Core 读取唯一 ready payload，开始 delivery attempt，并以 stdout 原样交给 `deliver: feishu` |
| `reconcile-delivery-evidence` | no-agent | 每 2 分钟 | 读取 Hermes execution/output/delivery 证据，把 attempt 关联回 Core delivery |
| `recover-unknown-work` | no-agent | 每 5 分钟 | 释放过期 lease，恢复可证明未发送的任务；不盲目重发 `unknown` 消息 |
| `runtime-health-watch` | no-agent | 每 5 分钟 | 只检查 Core、来源停滞、积压和 Cron 异常；不能证明 Gateway 自身存活 |
| `export-runtime-audit` | no-agent | 每日 02:00 | 导出 run/delivery 关联摘要与哈希，执行 90 天保留策略 |

所有无人值守 agent job 必须固定 provider/model、批准的 fallback、timeout、skills 和 toolset。模型或 provider 配置漂移时 fail closed。Hermes 不被假定提供可靠的 per-job 金额上限；成本由 Core lease 数量、模型输入/输出上限、provider 账户额度和独立 usage meter 共同约束。

生产固定 `cron.wrap_response: false`，避免 Hermes 的默认 Cron 头尾改变不可变日报；`deliver-ready-digest` stdout 必须与 Core canonical Markdown 逐字节一致。Feishu adapter 允许把它编码为 Markdown post，API 拒绝时按固定版本规则回退纯文本；Gateway 恢复补投还允许添加版本化 `Recovered reply` 传输信封。delivery evidence 记录 envelope/renderer 变体、`message_id` 和 canonical hash；剥离批准信封并解码后，稳定 ID、事实、URL 与正文语义必须等价。生产 job 不使用 `context_from` 表达依赖：它只读取上游最近一次成功输出，不等待同一 tick 的上游完成；所有依赖都通过 Core queue/lease/window 状态表达。

首版 skills 全部存放在仓库并只读同步到 Hermes 生产 skill root：

- `ai-monitor-editorial`：相关性、重要度、事实、证据和冲突处理；
- `ai-monitor-digest`：日报选择、排序、格式和长度；
- `ai-monitor-qa`：飞书问答、历史搜索和用户反馈解释。

任何 Agent 提议的新 skill 或修改只能进入候选区，经 verifier 审查和 Git 提交后才能用于生产。生产 principals 禁用 `skill_manage`、文件写入和配置写入；不需要 skill 的角色以 `--no-skills` 启动，需要 skill 的角色只读挂载自己的批准集合，不能加载用户目录或同名高优先级副本。

Hermes、飞书 SDK/集成、Python 依赖和所有插件都在 lock/镜像中精确固定；R0 不允许启动时自动升级、动态安装 skill 或从生产写回 Git。

Core MCP 按 principal 注册三个独立 alias：`mcp-core-chat`、`mcp-core-enricher`、`mcp-core-digest`。每个 alias 均固定 `sampling.enabled: false`、`tools.prompts: false`、`tools.resources: false`，并只配置自己的 `tools.include`；生产不注册通配 MCP server，也不依赖黑名单兜底。

### 3.3 确定性业务 Core

`ai-monitor-core` 是安装在派生 Hermes Runtime 镜像内的小型 Python 包，不重复实现 Hermes 已有的调度、聊天、Memory、Skills 和消息渠道。Hermes 以 stdio MCP 启动受约束工具进程；仓库中的版本化 no-agent wrappers 以只读 bind mount 映射到生产 profile 的 `$HERMES_HOME/scripts`，由 wrapper 调用镜像内固定 Core CLI。R0 校验每个脚本的 realpath、权限和 Git 记录哈希，防止持久卷遮住镜像内容或路径逃逸。依赖只在镜像构建时安装，容器启动时不联网拉包。首版不增加独立业务服务。

默认技术栈：

- Python 3.13（与固定 Hermes 镜像一致）、`uv`、SQLAlchemy、SQLite/Alembic；`requires-python` 与 CI 至少覆盖 3.13，若要兼容 3.12 必须单独验证；
- HTTPX、feedparser；
- Typer/Click CLI；
- stdio MCP server；
- pytest、Ruff、mypy；
- SQLite WAL、有限写事务、进程锁、独立持久卷和结构化日志。

Core 至少提供下列幂等工具：

- `ingest_due_sources`：按 checkpoint 拉取、标准化和 upsert；
- `lease_pending_items`：有 TTL 地租约待处理条目，避免并发重复；
- `commit_enrichment`：提交符合 Schema 的事实、证据、分类和不确定性；
- `get_source_evidence`：按 article ID 返回已采集的受限正文/元数据，不接受任意 URL；
- `list_event_candidates` / `search_items`：只读查询；
- `reserve_digest_window`：以唯一时间窗口、target 和 lease 创建或恢复 draft，并返回稳定 `digest_id/delivery_id`；
- `commit_digest` / `get_digest_by_window`：保存并读取不可变 rendered payload、内容哈希和来源集合；
- `begin_delivery_attempt`：CLI-only，由 `deliver-ready-digest` 以确定性的 `delivery_id = H(digest_id, target)` 进入 `sending`；
- `attach_delivery_runtime_execution`：CLI-only，把 reconcile 读取的精确 Hermes execution ID 追加到 attempt；
- `resolve_delivery_attempt`：CLI-only，记录 `accepted / failed / unknown` 与 delivery evidence；`unknown` 只允许被后续更强证据单向提升为 `accepted/failed`；
- `source_health`：返回可机器判断的健康状态。

日报使用两个显式状态机：`Digest(draft → ready)` 与 `Delivery(pending → sending → accepted | failed | unknown; unknown → accepted | failed)`。默认窗口是相邻两个 `Asia/Shanghai` 08:30 的左闭右开区间，落库前转换为 UTC。`reserve_digest_window` 先返回稳定 `digest_id/delivery_id`，模型把二者写入只生成一次的 payload 并调用 `commit_digest`；随后 no-agent `deliver-ready-digest` 读取该 payload、调用 `begin_delivery_attempt` 并把原文输出给 Gateway。任何重试都读取同一 canonical body，不能重新改写；Hermes 恢复补投只允许增加批准的传输信封。Agent 不依赖运行中可见 execution ID：`reconcile-delivery-evidence` 从固定版本实际暴露的 execution、canonical output 和 delivery evidence 中，通过 payload marker 回写关联与结果。R0 必须先证明这些证据字段可以稳定读取；若缺少能够区分“明确失败”和“可能已发送”的证据，状态只能保守记为 `unknown`，不得伪造 `accepted`；后续更强证据可将其单向提升。通道 send path 接受不表示用户已读；Core 对 after-send 模糊结果不另行提交消息，但允许 Gateway delivery ledger 完成有限补投。重复必须携带同一 `digest_id` 和相同 canonical hash。

LLM 不直接连接数据库，不自行拼 SQL，不推进 source checkpoint。工具调用失败或摘要模型不可用时，原始资讯仍入库，处理状态保留为 pending，稍后可安全重试。

CLI 与 MCP 的并发写入必须经过同一 repository 层、SQLite busy timeout 和跨进程 ingest lock。若 R0 证明这种单机共享数据库拓扑无法满足恢复或锁竞争目标，再将 Core 提升为独立服务或迁移 PostgreSQL。

Hermes state 与 Core DB 使用两个独立持久卷，分别执行一致性备份；provider/飞书密钥使用第三个受限 secret mount，不与普通状态混存。生产优先通过运行时环境/secret file 注入；若固定版本必须从 `$HERMES_HOME/.env`、`auth.json` 或可刷新 token 文件读取，就把精确文件单独挂载到期望路径，并从普通 state 快照中排除，凭据材料另行加密备份和轮换。Core DB 使用 SQLite backup API/WAL checkpoint 每日备份并保留至少 7 个日快照；Hermes 备份至少覆盖 `jobs.json`、`executions.db`、config、profiles/sessions 和 skills 版本引用，并在配置变更与升级前额外生成加密快照。除非固定版本验证了安全的在线备份方法，否则先 quiesce Gateway 再快照；禁止直接复制正在写入的 WAL 文件。R0 和发布 Gate 都必须校验 checksum，并从备份恢复到全新卷验证旧镜像/schema 可回滚，不能只验证“备份命令成功”。

### 3.4 Runtime 选择

| 候选 | 结论 | 理由 |
|---|---|---|
| Hermes Agent | **实现与生产默认** | 单一成熟 Agent 栈覆盖 Kanban、Cron、Gateway、Feishu、Skills 与 MCP；Core 补偿其 `unknown` 不自动重放语义，显著减少双栈集成与运维成本 |
| OpenClaw | 生产候选 B | 若 Hermes R0 的 Cron 恢复、通道证据或权限隔离不达标，再用新 ADR 和独立适配分支评估；不同时维护两套生产 Runtime |
| n8n | 不选作主 Runtime | 工作流调度成熟，但它是 workflow-first，不是本项目需要的常驻对话式 Agent Runtime |
| 自研 LangGraph/CrewAI/工作流 | MVP 不选 | 灵活，但需要自行实现调度、渠道、状态、权限、恢复和运维，违背“优先复用成熟 Agent”目标 |

## 4. 状态与真相来源

| 内容 | 真相来源 | 说明 |
|---|---|---|
| 产品范围、架构、契约 | Git | 任何 Agent 不得仅在记忆中修改约束 |
| Golden 样本、验收阈值 | Git | 实现 worker 无权自行放宽 |
| 当前任务、依赖、attempt | Hermes Kanban DB | 是调度状态，不提交进仓库 |
| 代码和迁移 | Git commit | 每张卡必须从指定 base SHA 开始 |
| 验收证据 | Git/CI artifact | 保存命令、退出码、日志摘要和制品哈希 |
| 生产 Cron 定义模板 | Git | deployment reconcile 只通过固定 Hermes Cron CLI create/edit/pause/remove 同步；禁止直接改 `jobs.json` |
| Cron execution 与通道投递证据 | Hermes Runtime | 短期运行证据；每日导出关联摘要与哈希 |
| 文章、checkpoint、事件、digest 与 delivery 状态机 | AI Monitor Core DB | 唯一内容业务真相，和 Hermes Memory 隔离 |
| 用户主题偏好 | Core DB / 版本化配置 | 可由 Agent 修改，但必须通过受约束工具 |
| Agent memory | 非真相来源 | 只保存对话习惯和经验，清空后不能影响业务正确性 |

Hermes 开发看板与生产 Cron 都应能由 Git 中的模板重建，避免任何单机 Agent 状态成为唯一不可恢复资产。清空生产 Memory/Sessions 或重建 Gateway 后，Core 数据和业务状态必须保持完整。

## 5. H0/P0/R0：实现与生产 Agent 资格测试

Hermes 迭代频繁。同一固定版本必须分别通过 H0（实施实例）与 R0（生产实例）；H0 绿色不能替代生产调度、通道和安全验收。

### 5.1 H0：实现驱动资格

在临时 Git 仓库完成：

1. 固定并记录 Hermes release/tag、安装来源和校验信息；
2. 运行诊断，确认模型、terminal backend、Kanban 和 Gateway 可用；
3. 创建独立项目 board 和至少 `orchestrator`、`implementer`、`verifier` 三个 profiles；
4. 让 implementer 在 worktree 中完成一张最小改动卡；
5. 故意提交一个测试失败的实现，确认 verifier 拒收而不是被 worker 总结说服；
6. 在 worker 运行中强制终止进程，确认 dispatcher 能回收并留下 attempt/log；
7. 制造重复失败，确认重试预算耗尽后进入 blocked/triage，而非无限循环；
8. 确认 handoff 能记录 commit、changed files、验证命令、剩余风险和重试建议；
9. 确认 worker 只能写允许的 workspace，不能读取项目密钥或写宿主敏感目录。

H0 通过条件：上述场景全部留下可审计证据，且 verifier 能阻止坏实现进入集成阶段。

### 5.2 P0：外部前置条件

R0 之前冻结 `runtime/hermes/prerequisites.yaml`，至少包含：

- 已创建并发布、可用范围仅含测试用户的飞书应用；scopes 以固定 tag 所需的 `im:message`、`im:message:send_as_bot`、`im:resource`、`im:chat`、`im:chat:readonly` 为基线，只按 allowlist 解析需要增补身份读取权限，不授予 docs/drive/meeting；记录 App ID/Secret 注入与撤销方式；
- 明确的 `FEISHU_DOMAIN=feishu`、`FEISHU_CONNECTION_MODE=websocket`、`FEISHU_ALLOWED_USERS` 和 `FEISHU_HOME_CHANNEL`；按当前私聊假设设置 `FEISHU_GROUP_POLICY=disabled`、`FEISHU_REQUIRE_MENTION=true`、`FEISHU_ALLOW_BOTS=none`，日报固定 `deliver: feishu`，主动投递目标不能靠会话记忆猜测；
- 已授权的模型 provider、精确模型、账户额度与 secret 注入方式；
- 默认 Linux/systemd 的 Docker Compose 目标主机、至少 2 GB 构建内存、持久卷和独立于飞书 Gateway 的 dead-man 告警目标；其他宿主必须先提供等价 supervisor adapter；
- 每项的只读验证命令、授权人/系统和撤销方法。

这些是外部授权输入，不是人类工程开发任务。缺少任一必需项时标记 `external_prereq_blocked`；Agent 继续完成 G0–G4 的离线工作，但不得伪造凭据或宣称 R0/G5 已通过。

### 5.3 R0：Hermes 生产 Runtime 资格

使用 G4 已验证的 Core、固定 Runtime 和飞书测试应用完成：

1. 固定 tag 的 `hermes cron --help`、配置 schema 和 tag 内官方文档已保存哈希；Gateway 重启后，`jobs.json`、下一次运行时间、execution ledger、会话路由和失败记录仍可恢复；
2. no-agent cron 只能运行只读挂载到 `$HERMES_HOME/scripts` 的批准 wrapper；realpath/哈希匹配 Git，成功可静默、失败告警且模型调用数为零；
3. no-agent 子进程的净化环境不包含模型/飞书凭据；Core 真正需要的非模型 secret 只能通过显式白名单环境或只读 secret file 注入；
4. agent cron 在 fresh session 中只加载批准 attached skills 和 `enabled_toolsets`；三个 Core MCP alias 都关闭 sampling/prompts/resources，并只暴露各自 `tools.include`；
5. MCP 的读取、lease、结构化写回和幂等键均能端到端工作；任何未列入白名单的 tool/resource/prompt/sampling 请求失败；
6. 飞书 WebSocket 通道能接收查询、以 `deliver: feishu` 投递到固定 home channel，并拒绝 allowlist 外用户；Feishu platform toolset 不包含 terminal、file、web、browser、cronjob、update 或 skill 管理；
7. 重复回放同一 Cron execution evidence 或同一 digest window 时不产生第二份业务 digest；重试读取相同 rendered payload；
8. 模型超时、`429`、凭据失效和 Gateway 崩溃不会丢失 Core 队列；
9. reconcile guard 检测到固定模型、工具策略、MCP alias 或 job 模板漂移时任务 fail closed 并告警；
10. Prompt Injection fixture 不能获取 shell、文件、Cron 控制面、密钥或额外外发权限；
11. 清空 Hermes Memory/Session 后，文章、偏好、checkpoint、digest 和 delivery 状态保持不变；
12. 只读挂载和 tool deny 能阻止生产 agent 修改 skills、scripts、配置或运行时控制面；Hermes profile 隔离不被当作 sandbox；
13. Hermes 升级先在 canary 环境恢复状态快照并完整跑 R0；失败可回退到固定上游 manifest digest、dev/prod 派生镜像 digest 与备份状态；
14. 强制终止运行中的 no-agent 与 agent job，确认遗留 Cron attempt 记为 `unknown`，并按类型补偿：ingest 等下一 tick 幂等恢复，enrichment 等 lease TTL，尚未进入 `sending` 的 digest 可按同一 window 重跑；已进入 Gateway delivery ledger 的 `sending`/after-send unknown 只由其有限补投，Core 不另行提交消息；
15. 连续制造失败，确认项目告警、有限重试/补偿和 reconcile 熔断符合版本化策略，不把 Hermes 未承诺的自动重放当成 Gate；
16. 从 Core DB 与 Hermes `jobs.json`、`executions.db`、output、profiles/sessions 和独立 secret mount 恢复到全新卷，确认内容、调度和未完成任务可继续收敛；
17. 终止 Gateway 进程，确认固定镜像内 s6 先自动拉起；再制造 s6/真实 health 卡死，确认宿主 systemd 在 2 分钟内重启容器；最后禁用宿主恢复，确认独立于 Gateway/飞书的 dead-man 告警触发；
18. 注入 before-send、after-send 和 evidence-lost 故障，确认 Gateway delivery ledger 的有限补投仍关联同一 `delivery_id` 和 canonical hash，恢复消息只增加文档化 `Recovered reply` 信封；reconcile 优先用 `hermes cron runs`、output 与通道 `message_id` 关联 execution ID，只有纳入固定 schema contract 后才允许直读 `executions.db`。有充分证据时收敛为 `failed / accepted`，不足时保守为 `unknown`；
19. 固定 `cron.wrap_response: false`；`deliver-ready-digest` stdout 与 Core canonical Markdown 逐字节一致；剥离批准的 recovery envelope 并解码飞书 post/plain-text fallback 后，保留相同稳定 ID、事实、URL 和 canonical hash；
20. Runtime 内 `python --version`、no-agent wrapper 的 `sys.executable` 和 Core lock 均为冻结的 Python 3.13 组合。

R0 通过条件：真实飞书测试通道完成至少一个采集、处理、日报周期，重启和失败注入均留下可审计记录。

若 H0 失败，改用 Codex 或 Claude Code 按同一任务契约执行。若 Hermes R0 失败，项目在生产 Runtime ADR 处 blocked；OpenClaw 候选必须新增适配分支并重新执行 P0/R0，不能沿用 Hermes Gate 的结论。首版不把 Claude Code/Codex CLI 强行接成 Hermes 外部 worker lane，因为该路径尚不是 Hermes 官方标准化能力。

## 6. Agent 角色设计

| Profile | 职责 | 允许修改 | 禁止事项 |
|---|---|---|---|
| `orchestrator` | 分解目标、建立依赖、分配任务、处理阻塞 | Kanban 任务与评论 | 不写产品代码，不自行验收 |
| `contract-verifier` | 冻结契约、Golden fixtures 和 Gate 测试 | `docs/`、`evals/`、验收测试 | 不实现被测功能，不降低阈值 |
| `connector-worker` | RSS、arXiv、GitHub、HN 适配器 | connectors 与 worker-owned unit tests | 不改公共领域模型和 Gate 测试 |
| `pipeline-worker` | 标准化、去重、聚类、摘要和排序 | pipeline 与 worker-owned unit tests | 不改来源契约和 Gate 测试 |
| `runtime-worker` | Core MCP、Hermes Prod/Cron/skills 和飞书集成 | `runtime/hermes/` 与 worker-owned unit tests | 不改 runtime contract、security、Golden 和领域模型 |
| `integrator` | 合并已验收提交、解决接口冲突 | 集成文件、迁移、构建配置 | 不替实现卡补功能，不绕过 Gate |
| `security-release` | 安全审查、离线验收、Live Canary | 只读运行 protected Gate；写报告与发布证据 | 不修改测试/阈值，不开发新功能 |

默认并发上限为 3。只有修改路径无交集、公共契约已冻结且依赖已满足的卡才能并行。

生产环境在 Hermes 中划分以下最小权限逻辑 principals。固定版本若不能可靠实施 per-job tool filter，就必须以独立 production profiles/processes 强制隔离，不能只靠 prompt：

| Principal | 职责 | 工具面 | 明确禁止 |
|---|---|---|---|
| `news-chat` | 响应飞书查询、解释事件和管理主题偏好 | `mcp-core-chat` 的只读查询/受约束偏好写入；仅 `ai-monitor-qa` skill | shell、文件写入、任意 Web、部署、Cron 管理 |
| `news-enricher` | 事实抽取、分类和聚类建议 | `mcp-core-enricher` 的 lease/evidence/commit；仅 `ai-monitor-editorial` skill | 任意 SQL、任意 Web、任意外发、修改 skills/memory 权限 |
| `news-digest` | 生成并提交不可变日报 payload | `mcp-core-digest` 的 candidate/reserve/commit 工具；仅 `ai-monitor-digest` skill | attach/resolve delivery、改历史 digest、任意 Web、通用消息发送、Cron 管理 |
| no-agent runner（非 Agent） | 采集、evidence reconcile、导出审计和业务健康检查 | `$HERMES_HOME/scripts` 下只读 wrapper，包括 attach/resolve | 动态 shell、模型、浏览器；权限由 OS/container 而非 prompt 控制 |

Hermes 实施 profiles 与生产 principals 完全分离，不共享 `HERMES_HOME`、会话、Memory、Cron DB、skills 写权限、密钥或高权限工具集。Profiles 只隔离 Hermes 状态，本身不是 sandbox；真正权限边界仍由独立容器/OS 用户、只读挂载、网络策略和 Core capability 强制。三个批准 skill 位于各 principal 的生产只读 skill root；attached skill、`enabled_toolsets` 与 MCP `tools.include` 的最终集合由 Git desired state 生成并在运行前 preflight，防止同名覆盖和权限漂移。

no-agent runner 不受 Agent prompt 保护。其边界必须由容器/OS 强制：非 root UID、只读 rootfs、只挂载 Core/Hermes 必需卷、drop capabilities、`no-new-privileges`、无 Docker socket、净化环境变量和最小网络出口；R0 从容器外验证越权失败。

## 7. 统一任务卡契约

任务卡正文使用中立格式，Hermes、Codex 或 Claude Code 都能执行：

```yaml
id: G2-RSS-01
title: 实现 RSS/Atom 来源适配器
base_sha: <git-sha>
assignee: connector-worker
depends_on: [G1-VERTICAL-01]

outcome:
  - 将冻结的 RSS/Atom 响应转换为统一 NormalizedArticle

inputs:
  - docs/contracts/source-adapter.md
  - tests/fixtures/rss/
  - tests/contracts/test_source_adapter.py

allowed_paths:
  - src/ai_monitor_core/connectors/
  - tests/unit/connectors/

read_only_paths:
  - tests/contracts/
  - tests/runtime_contract/
  - tests/security/
  - evals/thresholds.yaml
  - evals/golden/

boundaries:
  - 不修改公共数据契约
  - 不访问任务未列出的域名
  - 不用 live response 替代 fixture 测试

verification:
  commands:
    - uv run pytest tests/unit/connectors/test_rss.py tests/contracts/test_source_adapter.py -q
    - make check
  expected:
    - exit_code: 0
    - skipped_tests: 0

permissions:
  workspace_write: true
  network_domains: []
  secrets: []
  external_side_effects: false

budget:
  max_attempts: 3
  max_goal_turns: 15

stop_when:
  - 公共契约与实现目标冲突
  - 相同失败连续出现 3 次且没有新增诊断证据
  - 需要未授权的密钥、网络或破坏性操作

completion_evidence:
  - commit_sha
  - changed_files
  - commands_and_exit_codes
  - test_log_path
  - residual_risk
```

卡片正文必须明确 `Outcome / Verification / Constraints / Boundaries / Stop-when`。Goal-mode 的 judge 只帮助 worker 续跑，最终仍需由独立 verifier 运行确定性命令。

每张实现卡必须有一张依赖它的验证卡。实现卡“完成”只表示交付候选 commit，不代表 Gate 已通过。

`tests/contracts/`、`tests/runtime_contract/`、`tests/security/`、`evals/thresholds.yaml` 与 `evals/golden/` 由 `contract-verifier` 独占修改。实现 worker 可以新增自己的 unit tests，`security-release` 可以只读运行 Gate 并写报告，但二者都不能修改、删除、skip 或放宽 Gate；确需变更时，先建立独立契约卡，由 verifier 提交并合并后，重新派发实现卡。

## 8. 实施 DAG

```text
H0-DRIVER        Hermes 实现驱动资格
   ↓
G0-CONTRACT      上下文、Core/MCP 契约、fixtures、runtime 模板与版本快照
   ↓
G1-VERTICAL      固定 Hermes Prod → Core → Agent 摘要 → fake receiver
   ↓
   ├── G2-RSS       官方 RSS/Atom 来源
   ├── G2-ARXIV     arXiv 连接器与限流
   ├── G2-GITHUB    GitHub Releases 连接器
   ├── G2-HN        HN 信号连接器
   ├── G2-FETCH     HTTP 缓存、退避、大小限制、错误分类
   ├── G2-DEDUP     URL、external ID、内容哈希与近重复
   ├── G2-MCP       lease、结构化提交、查询和幂等工具
   ├── G2-SKILLS    分类、摘要、冲突处理和日报 skills
   ├── G2-CRON      声明式 jobs、reconcile 与熔断
   └── G2-FEISHU    对话、日报和异常通知
            ↓
G3-INTEGRATION     Gateway/Core 集成、失败隔离、崩溃恢复
   ↓
G4-OFFLINE         Core + fake Runtime 的无公网全链路验收 ──┐
                                                           ├─→ R0-RUNTIME
P0-EXTERNAL        外部授权、真实模型/飞书/主机 ──────────────┘       ↓
G5-LIVE            固定 Hermes Prod + 真实来源、模型、飞书 Canary
   ↓
MVP-RELEASE
```

### G0：契约与仓库基线

产物：

- 当前 Git 基线上的首个实现 commit；
- `AGENTS.md`，记录架构、统一命令、不可绕过规则；
- `docs/adr/0001-hermes-runtime.md`，记录单栈双实例决策、Hermes/OpenClaw 证据、版本和替换边界；
- `pyproject.toml`、锁文件、Makefile、CI；
- `config/sources.yaml`：精确 URL/API、adapter、频率、限流、允许重定向 host、响应上限、条款 URL、owner 与停用策略；
- `config/schedules.yaml`：固定 tag 实测支持的 cron 表达式、容器 `Asia/Shanghai` 时区、UTC window 规则和 `FEISHU_HOME_CHANNEL` 的非秘密引用；日报默认每日 08:30，任务错峰由显式表达式实现；
- `SourceConfig`、`FetchedDocument`、`NormalizedArticle`、`EnrichmentResult`、`Digest`、`Delivery` 契约；
- Core CLI/MCP Schema、lease、checkpoint 和幂等契约；
- Hermes production profile/principals、skills、Cron desired-state 与安全配置模板；
- `v0.20.0 / v2026.8.3` tag 内官方文档 URL/commit、CLI help/config schema 输出及哈希，作为 runtime contract 版本快照；
- 固定版本的 Hermes/Feishu 集成清单、上游 manifest digest、dev/prod 派生镜像定义与 digest、Python 3.13 runtime contract，以及状态备份/恢复 runbook；
- dev/prod 独立 Compose/config/volume/network/secret 模板；生产 scripts/skills 只读挂载清单与哈希；
- 三个 Core MCP alias 的 sampling/prompts/resources 关闭配置、`tools.include` 白名单和越权 fixture；
- RSS/Atom/GitHub/arXiv/HN 冻结 fixtures；
- Golden 去重、摘要、安全样本；
- 任务模板和验收证据格式。

来源配置不得靠模型记忆猜 URL。每个 endpoint 必须附官方页面或官方仓库证据，由 verifier live probe Content-Type、缓存/限流行为和条款；找不到稳定官方 Feed/API 的来源移到 HTML 适配器候选，不进入 MVP 默认启用集。

Gate：`uv sync --all-extras && make check` 通过，且 verifier 确认实现 worker 无权修改任何 contract/runtime/security Gate 或 Golden 预期。

### G1：最小真实纵向切片

只接一个官方 RSS 冻结 fixture 和本地 fake receiver，打通不需要外部凭据的纵向切片：

```text
Hermes no-agent cron → Core fetch/normalize/dedupe → SQLite
Hermes fresh agent session → MCP lease → 结构化事实/摘要 → Core commit
Hermes no-agent canonical payload → Gateway delivery → fake receiver
```

Gate：

- 同一数据连续导入 3 次，文章数量不增加；
- Core 在无网络、无 LLM Key 时 fixture E2E 通过；
- Gateway 或模型中断后待处理条目仍可恢复；
- 相同 digest window 重放不产生第二份业务 digest；
- fake receiver 收到的每条日报内容都能回到原始 URL，并包含同一 `digest_id`；
- fresh install 只需 README 中一条 Compose 启动命令。

### G2：可并行组件卡

G1 通过且公共契约冻结后，来源、Core/MCP、Hermes Prod skills/Cron 和 Feishu 集成分 worktree 并行实现。每个组件使用自己的 fixtures、定向测试和 verifier 卡，不允许并行修改同一公共接口。

### G3：集成与可靠性

集成内容：

- Hermes Cron desired-state reconcile、版本化熔断、运行审计和防重入；
- `ETag`、`Last-Modified` 和增量 checkpoint；
- `429 Retry-After`、有限重试和死信；
- 单来源失败隔离；
- Gateway、抓取、解析、入库、lease、摘要各阶段的崩溃恢复，以及 ingest/enrichment/digest/delivery 各自的 `unknown` 补偿策略；
- 来源、Core、Cron 和积压健康状态；Gateway 自身由外部 supervisor/dead-man 监控；
- 日报/delivery 状态机、不可变 payload、`digest_id/delivery_id` marker，以及 delivery evidence reconcile 对 Cron execution 的精确关联；
- Hermes Memory/Session 清空后的业务状态恢复；
- skills/config 版本漂移检测；
- Hermes execution ledger/日志与 Core 业务指标的统一健康视图；
- 运行证据摘要/哈希的每日导出；
- 宿主级 restart/healthcheck、独立 dead-man 告警；
- 数据卷和 secret path 的一致性备份、保留、恢复演练和升级回滚。

Gate：相同批次运行 3 次和并发运行后 Core 业务不变量一致；每个阶段注入进程终止后，文章/checkpoint/digest 不丢失，delivery audit 在 SLO 内收敛为 `accepted / failed / unknown`。允许通道重复，但只能使用同一 `digest_id` 与不可变 payload。

恢复分两级：固定 Hermes 镜像内 s6 先拉起崩溃的 Gateway；宿主 systemd service/timer 再探测真实 Gateway health，而不是只看容器/s6 是否存活，s6 卡死或持续 `unhealthy` 时重启整个容器。dead-man heartbeat 从宿主直接发送到独立告警服务，不能经同一个 Hermes Gateway/飞书链路。Runtime 容器本身不挂载 Docker socket。

### G4：Offline Release Gate

在全新、禁止访问公网、没有任何 API Key 的环境中，以 fake Runtime/receiver 执行：

```bash
make check       # format、lint、type、unit
make verify      # 契约、幂等、恢复、Golden eval、安全
make smoke       # 构建 Core 与 fake Runtime，跑 fixture 全链路
make runtime-contract  # 验证固定 tag 的 Hermes MCP/Cron/skill/config/CLI 契约
```

任一测试被删除、注释、`skip` 或 `xfail` 均视为失败。

### G5：Live Canary Gate

使用固定版本 Hermes Prod、真实来源、真实模型和真实飞书测试应用运行至少 24 小时，覆盖不少于 3 个增量采集周期和 1 次日报：

```bash
make live-smoke
```

Live Gate 禁止 Mock。缺少模型凭据、飞书测试应用、部署授权或付费授权时，状态只能是 `offline_verified`，不能宣称 MVP 已上线。

## 9. 强制验收场景

### 9.1 来源与解析

- 正常 RSS/Atom、缺字段、非法 XML、错误编码、gzip、超大响应；
- `304`、重定向、超时、连接断开、`429`、`500`；
- 跨时区、未来时间、更新时间与首次发现时间；
- 一个来源损坏时其他来源仍能完成。

### 9.2 幂等与恢复

- 相同 fixture 重复导入；
- 多 worker 并发处理同一 external ID；
- 抓取、解析、入库、摘要和通知阶段强制中断；
- 重启后不丢数据；相同日报窗口只生成一个 `digest_id`，任何通道补投都复用同一 canonical body/hash（仅允许批准的 recovery envelope）并保留 `accepted / failed / unknown` 审计记录。

### 9.3 去重与摘要

- 同文不同 URL；
- 同事件不同报道；
- 标题相同但事件不同；
- 每条摘要事实必须关联 source ID 和短证据；
- Golden 样本中不得新增原文不存在的实体、数字或日期；
- 来源互相矛盾时必须表达不确定性，不能静默选边。

### 9.4 安全

- 网页 Prompt Injection 不得改变系统指令、触发越权工具调用、读取密钥或扩大外发范围；
- 重定向到 localhost、私网和云元数据地址必须被拒绝；
- 限制响应体、重定向次数和允许的 Content-Type；
- 通知正文不得包含密钥、原始系统提示词或内部路径。

### 9.5 生产 Agent Runtime

- Gateway 和宿主重启后 Cron 定义与 execution ledger 不丢失、不重复生成业务 digest；
- no-agent job 的测试证明没有模型调用；
- fresh agent job 只能看到 `enabled_toolsets` 和 MCP `tools.include` 正向批准的 Core 工具，不能直接访问任意 Web；
- 删除 Memory/Session 不影响文章、偏好、checkpoint、digest 和 delivery 状态；
- 模型不可用时采集继续，待处理队列不丢失并产生异常告警；
- Cron prompt、skill、model、provider、CLI/config schema 和 MCP Schema 发生漂移时能够被检测；
- 飞书 allowlist、群聊 @mention、重复投递和连接恢复经过真实通道验证；
- production agent 无法调用 shell、修改文件、创建 Cron、部署或读取秘密；
- no-agent runner 在容器/OS 边界无法提权、访问 Docker socket、宿主文件或未授权网络；
- Gateway 停止时，宿主 supervisor 负责重启，独立 dead-man 负责告警；Gateway 内部 Cron 不被当作自监控。

## 10. 权限与密钥策略

权限按任务卡最小化授予：

| 等级 | 权限 |
|---|---|
| P1 | 只读仓库和运行测试 |
| P2 | 指定 worktree 路径写入 |
| P3 | 安装锁定依赖、访问白名单域名 |
| P4 | 使用运行时注入的模型密钥 |
| P5 | 发送真实通知、部署、写外部系统 |
| P6 | 破坏性迁移或删除，仅显式人工授权 |

Hermes 实现 worker 默认使用隔离 backend；宿主机 local terminal 不能视为 sandbox。配置工作区 safe root、危险命令审批和 headless fail-closed，禁止在宿主机启用无审批的高权限模式。

生产 `news-chat`、`news-enricher` 和 `news-digest` principals 不启用 terminal、文件写入、Git、部署、Cron 管理、任意 Web 或通用消息发送 toolset，只暴露各自白名单 Core MCP 和 Gateway 当前回复通道。no-agent jobs 只能执行从 Git 制品只读 bind-mount 到 `$HERMES_HOME/scripts` 的批准 wrapper，不接受由文章正文或模型拼出的脚本、路径或参数。

其他要求：

- 密钥只能由环境或秘密代理注入，不写入任务卡、prompt、Git 或日志；
- Agent 不创建或修改真实 `.env`；
- Hermes Gateway 只绑定实现飞书 WebSocket 所需的最小接口；任何本地管理端口只发布到 loopback 或完全不发布，不向 LAN/公网开放；
- Gateway 使用用户 allowlist；
- 飞书配置显式设置 user/group allowlist、群聊 mention 规则、home channel 和投递 target；MVP 使用单用户 DM，设置 `FEISHU_GROUP_POLICY=disabled`、`FEISHU_REQUIRE_MENTION=true`、`FEISHU_ALLOW_BOTS=none`，不订阅文档评论/会议事件，不授予对应 scope；
- 固定版本 Feishu platform 的默认 toolset 可能包含 terminal，必须用显式最终集合覆盖；Hermes admin/regular 分层只限制 slash commands，不能代替普通聊天的工具裁剪；
- 入站附件会被缓存，文本附件还可能进入模型上下文；MVP 不启用附件工作流，并在 Runtime contract 中验证附件不会扩大工具或外发权限；固定版本不存在的配置键不得由当前在线文档臆造；
- 网页正文始终是不可信数据，不具有指令权限；
- 摘要 Agent 不拥有 shell、部署、任意文件或独立通知凭据；
- 生产 skills 从 Git 同步并设为只读，Agent 自动生成的 skill 必须经过 verifier 才能进入生产；
- 生产 agent jobs 的 `enabled_toolsets`、MCP `tools.include` 和 attached skills 使用显式最终集合；MCP sampling/prompts/resources 关闭，禁用 terminal、配置写入、`skill_manage` 和控制面工具，空或错误的集合必须在模型调用前失败；
- 只有确定性的 deployment reconcile 身份能通过固定 Hermes Cron CLI 管理任务；任何身份都不直接编辑 `jobs.json`，生产 agent sessions 不能创建、修改、删除或手动运行 Cron；
- Runtime 使用专用非 root UID、只读 rootfs、drop capabilities 和 `no-new-privileges`，不挂载 Docker socket，不使用 privileged 容器；
- fresh agent job 只使用固定 tag schema 实测存在的 model、provider fallback、`enabled_toolsets` 和 timeout 字段；token 上限由已验证的 provider model params、Core lease 和 usage meter 执行；
- Core MCP 对每个写操作实施输入 Schema、lease token、幂等键和审计日志；
- CAPTCHA、登录墙或条款不允许采集时立即停用来源；
- 安装脚本、依赖和 Agent 驱动器使用精确版本并先审查再运行。

## 11. 重试、成本与停止条件

### 11.1 Agent 实施预算

每张卡声明最大 attempt、goal turns、允许的模型/供应商和费用上限。默认同类失败最多 3 次；只有产生新的诊断证据时才允许下一次尝试。

停止并持久化现场的条件：

- 超过 attempt、turn、时间或费用上限；
- 缺少契约、密钥、权限或外部授权；
- 公共契约与任务目标冲突；
- 测试结果回退，或实现要求修改 Golden 预期；
- 怀疑密钥泄露、提示注入或供应链异常；
- 来源条款不允许采集；
- Agent 循环修改同一文件但没有新增验证证据。

阻塞 handoff 必须记录：复现命令、已尝试方案、最新日志、当前 commit、明确 blocker 和唯一建议的下一步。

### 11.2 产品模型成本

- 采集和业务健康巡检使用 Hermes no-agent cron，不调用模型；
- 规则过滤和精确去重先于模型调用；
- 先按事件聚类，再生成一次摘要；
- agent job 固定模型；Core 限制每轮 lease 数量与输入规模，输出上限只使用固定 tag/schema 验证过的 provider model params；
- 模型结果按内容哈希、skill/prompt 版本和模型版本缓存；
- Core 在发放 lease 前检查每轮文章数/token 预算，provider 账户设置独立额度，usage meter 对估算金额实施告警和 fail closed；
- 超过 Core/provider/usage 任一预算时保留原文元数据并延迟摘要，不阻断资讯入库；
- 不把 Hermes 当作原生 per-job 金额预算器。

## 12. 发布状态与完成定义

### 12.1 机器验收阈值

阈值以 `evals/thresholds.yaml` 版本化，Gate 输出 JSON，不允许 Agent 用主观总结替代：

| 指标 | MVP Gate |
|---|---|
| 精确重复与并发重复 | 重复导入/并发 fixture 新增业务文章数为 `0` |
| 崩溃数据安全 | 已提交 Core row 的 RPO 为 `0`；中断后 10 分钟内队列收敛 |
| 调度及时性 | due-source job 在计划时间后 2 分钟内启动、10 分钟内完成 |
| 处理积压 | 正常 provider 下 pending age p95 ≤ 30 分钟，最大值 ≤ 2 小时 |
| 日报 | 08:30 后 10 分钟内形成 immutable payload，15 分钟内进入可观测的 `accepted/failed/unknown`；非 accepted 必须告警，`unknown` 可被后续证据单向提升，超过 Gateway 24 小时恢复窗口仍不明则保持 unknown 并升级事故 |
| 摘要事实 | Golden 中新增的原文外实体、数字或日期为 `0`；Schema/证据 ID 合法率 100% |
| 分类质量 | 冻结 Golden 上 macro-F1 ≥ 0.85；样本或阈值变更只能由 verifier 卡完成 |
| 安全 | Prompt Injection/SSRF fixtures 的越权调用、私网访问和秘密泄露均为 `0` |
| Gateway 恢复 | healthcheck 失败后 2 分钟内由宿主 supervisor 恢复；dead-man 测试 5 分钟内告警 |
| Live Canary | 24 小时内至少 3 次采集和 1 次日报全部有 run/task/digest 关联，无永久 in-flight |

外部模型或来源明确不可用时不伪造绿色 SLO：任务进入已分类的 degraded/blocked 状态，原文入库继续，并在 5 分钟内产生告警证据。

### 12.2 发布状态

| 状态 | 含义 |
|---|---|
| `code_verified` | 组件定向测试通过 |
| `runtime_contract_verified` | 固定 Hermes tag 的 Cron/MCP/Skills/Agent/CLI 配置与 Core 契约通过 |
| `offline_verified` | 无网络、无密钥、fake Runtime 的完整 E2E 通过 |
| `external_prereq_blocked` | P0 外部授权缺失；不否定离线证据，但禁止进入 R0/G5 |
| `live_verified` | 固定 Hermes Prod、真实来源、模型和飞书 Canary 通过 |
| `released` | 目标环境部署、备份、监控和回滚验证完成 |

MVP 只有在 `live_verified` 后才能称为完成。worker、orchestrator 或 goal judge 的自然语言总结不改变发布状态。

必须达到：

- 精确重复采集不产生新增文章；
- 单个坏来源不影响其他来源；
- 所有资讯保留原文链接和四类时间：发布、更新、首次发现、抓取；
- 摘要关键事实可追溯，注入测试通过；
- 相同日报窗口只生成一个业务 digest；通道补投复用同一 `digest_id` 和 canonical body/hash，仅允许批准的 recovery envelope；模糊发送结果保持 `unknown`，不承诺 exactly-once 或用户已读；
- 外部 supervisor/dead-man 能判断 Gateway 存活；内部健康检查区分 Core 存活、采集停滞、处理积压和投递异常；
- 清空 Hermes Memory/Session 不影响业务正确性；
- Docker Compose 可一条命令启动包含 Core 的固定 Hermes Runtime；
- Offline 与 Live 证据均可重放和审计。

## 13. 目标仓库结构

```text
AGENTS.md
README.md
Makefile
.gitignore
.env.example
pyproject.toml
uv.lock
alembic.ini

config/
  sources.yaml
  schedules.yaml

deploy/
  compose.dev.yaml
  compose.prod.yaml
  hermes-dev.Dockerfile
  hermes-prod.Dockerfile
  healthcheck/

alembic/
  versions/

docs/
  PLAN.md
  contracts/
  adr/
  runbooks/

orchestration/hermes/
  config/
  profiles/
  board-templates/

runtime/hermes/
  prerequisites.yaml
  config/dev/
  config/prod/
  contracts/v2026.8.3/
  principals/news-chat/
  principals/news-enricher/
  principals/news-digest/
  cron/
  scripts/
  reconcile/
  skills/ai-monitor-editorial/
  skills/ai-monitor-digest/
  skills/ai-monitor-qa/

ops/
  backup/
  restore/
  supervisor/
  monitoring/

tasks/
  schema.json
  templates/

src/ai_monitor_core/
  connectors/
  pipeline/
  models/
  repository/
  cli/
  mcp_server/
  audit/

tests/
  fixtures/
  unit/
    connectors/
    pipeline/
    mcp/
  contracts/
  runtime_contract/
  integration/
  e2e/
  security/

evals/
  thresholds.yaml
  golden/
  reports/
```

当前步骤只创建本计划文档。其余文件在 G0 由 Agent 按 Gate 顺序生成。

dev/prod Compose 必须声明不同容器名、network、`HERMES_HOME` volume、Core DB volume 和 secret mount；生产 `scripts/` 与各 principal 的批准 skill 集合按只读 bind mount 注入，不与开发实例共享任何可写路径。

`tests/contracts/`、`tests/runtime_contract/`、`tests/security/`、`evals/thresholds.yaml` 与 `evals/golden/` 的唯一修改 owner 是 `contract-verifier`；`security-release` 只运行它们并写报告。`deploy/`、`runtime/hermes/` 和 `ops/` 的实现 owner 是 `runtime-worker`，其安全与恢复 Gate 仍只能由 `contract-verifier` 修改。

## 14. 延后能力的触发条件

- SQLite 出现稳定锁争用、需要多进程写入或文章规模明显增长时迁移 PostgreSQL；
- 确定性聚类后重复曝光率仍高于 5% 时引入 embedding/pgvector；
- Hermes Cron 出现持续重叠、`unknown` 无法在 SLO 内收敛或单机 Gateway 无法满足恢复目标时，再评估 OpenClaw、Redis + 独立 worker 或托管调度；
- 出现第二个真实用户后再设计账号、权限和个性化；
- 只有高价值来源确实没有稳定 Feed/API 时才写来源专用 HTML 适配器；
- 飞书对话和日报不能满足浏览需求时，再增加只读 Web UI；UI 复杂度确实需要时才引入 Next.js；
- R0 或持续生产证据表明 Hermes 不合格时，创建新 Runtime ADR 并 blocked；可评估 OpenClaw，但必须实现独立适配分支并通过完整 P0/R0，不同时维护两套生产 Runtime。

## 15. 实施启动顺序

收到“开始实施”后，Agent 按以下顺序执行，不再向用户反抛普通技术选择：

1. 以当前 `main` 和本计划作为基线；
2. 执行 Hermes H0；并行建立 P0 清单，缺少外部授权不阻塞离线实施；
3. 创建根 `AGENTS.md`、Core/MCP 契约、protected Gate 和任务卡 schema；
4. 建立 Hermes 项目 board、实现 profiles、并发和安全策略；
5. 将 G0–G5、P0/R0 DAG 写入 Kanban；
6. 完成 G0、固定版本 runtime contract 和离线 G1；
7. G1 通过后才并行执行 G2；
8. 建立隔离的 Hermes Prod profile/principals、版本化 skills、Cron desired-state、no-agent scripts、reconcile、supervisor 和最小权限策略；
9. 完成 G3/G4；P0 就绪后执行真实 R0，再进入 G5；
10. 逐 Gate 集成、验证、回滚或阻塞；只为 P0/R0/G5 请求真实模型凭据、飞书测试应用和部署授权。

## 16. 参考资料

Hermes 实现与生产 Runtime 官方资料：

- [Hermes Agent v0.20.0 / v2026.8.3 Release](https://github.com/NousResearch/hermes-agent/releases/tag/v2026.8.3)
- [固定 tag：Kanban](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/features/kanban.md)
- [固定 tag：Profiles](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/profiles.md)
- [固定 tag：Git Worktrees](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/git-worktrees.md)
- [固定 tag：Goals](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/features/goals.md)
- [固定 tag：Context Files](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/features/context-files.md)
- [固定 tag：Cron](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/features/cron.md)
- [固定 tag：Feishu / Lark](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/messaging/feishu.md)
- [固定 tag：Messaging Gateway](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/messaging/index.md)
- [固定 tag：MCP](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/features/mcp.md)
- [固定 tag：Skills](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/features/skills.md)
- [固定 tag：Toolsets Reference](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/reference/toolsets-reference.md)
- [固定 tag：Docker](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/docker.md)
- [固定 tag：Security](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/security.md)

在线文档只用于发现能力；H0/R0 和实现必须以以上固定 tag 快照、保存的 CLI/schema 输出与镜像 digest 为准。

OpenClaw 候选 B 官方资料（仅在 Hermes R0 失败并建立新 ADR 后使用；不预先固定旧版本）：

- [OpenClaw Releases](https://github.com/openclaw/openclaw/releases)

数据源官方资料：

- [arXiv API 条款](https://info.arxiv.org/help/api/tou.html)
- [Hacker News API](https://github.com/HackerNews/API)
- [GitHub REST API 限流](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)
