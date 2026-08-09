# AI Monitor：成熟 Agent 驱动的实施与运行计划

> 状态：Draft v0.4<br>
> 日期：2026-08-09<br>
> 当前仓库：已初始化 Git，`main` 跟踪 `origin/main`，尚无产品代码<br>
> 实施主体：AI Agent；不安排人类工程师执行开发任务

## 1. 核心决策

本项目不按人类团队的“人数 × 周期”排期，而按依赖图、机器可验证任务和发布门禁推进。

执行原则只有四条：

1. **Git 中的规格与测试是真相**：架构约束、任务契约、Golden 样本和验收命令必须版本化。
2. **单 Agent 栈、隔离运行面**：Hermes 同时负责 AI 实施编排与生产 API/Cron/Skills/通知接入；实施、生产调度和生产查询 Agent profiles 完全隔离，确定性 mailer 作为独立 one-shot principal 只复用 Hermes sender CLI，Open WebUI 只是产品入口，不是第二套 Agent Runtime。
3. **确定性 Core 掌握业务真相**：采集 checkpoint、文章、去重、事件、digest 窗口、不可变 payload 和 delivery 状态必须由受测试的代码与数据库维护，不能只存在于 Agent 上下文；Hermes/mailer 只产生短期运行证据和对话上下文，业务终态仍写回 Core。
4. **确定性验收决定完成**：worker 的总结和 Agent judge 都不能单独证明任务完成。

默认采用 [Nous Research Hermes Agent v0.20.0 / v2026.8.3](https://github.com/NousResearch/hermes-agent/releases/tag/v2026.8.3) 同时作为实现控制平面与生产 Agent Runtime，并采用 [Open WebUI v0.11.0](https://github.com/open-webui/open-webui/releases/tag/v0.11.0) 提供可安装为 PWA 的主交互界面。项目不跟随 `main`、`latest` 或浮动镜像标签；H0 验证 Hermes 实施能力，R0 独立验证生产 Cron、OpenAI-compatible API、Open WebUI、standalone Email sender、MCP、恢复和安全能力。G0 必须解析并记录上游 `nousresearch/hermes-agent:v2026.8.3`、`ghcr.io/open-webui/open-webui:v0.11.0` manifest digest，以及 dev/prod 派生镜像各自的不可变 digest；启动时不自动升级或动态安装组件。G0 还必须扫描两个固定版本的已公开安全公告；存在适用于当前配置且未缓解的 Critical/High 时 fail closed，通过 verifier 更新版本与 ADR 后才能继续。

AI Monitor 生产环境依赖 Hermes API/Cron/Skills、模型调用、Open WebUI 和 Email standalone sender，但业务数据不写入 Agent Memory、Open WebUI 会话库或邮箱，幂等和业务状态不依赖自然语言推理。Hermes 与 `ai-monitor-core` 之间只通过有 Schema、权限边界和幂等键的固定 CLI/MCP 工具交互。`hermes-dev`、`hermes-prod-scheduler` 与 `hermes-prod-chat` 使用不同容器或 OS 用户、不同持久化 `HERMES_HOME`、profiles、sessions、Cron state 和 secrets；`hermes-prod-mailer` 是由宿主 systemd timer 触发的独立无模型 one-shot 容器，每次只使用临时 `HERMES_HOME`、只读配置和 SMTP secret，不启动 Agent、Cron 或 Gateway。`prod-chat` 不持有 Cron/Email 密钥，`prod-scheduler` 不暴露用户 API 或 SMTP 密钥，只有 `prod-mailer` 能外发。生产配置不启用、不暴露实施 toolset，若 hardened prod image 能实测剥离这些组件则进一步移除。

若 Hermes 的实现驱动能力不合格，可改用 Codex 或 Claude Code 执行同一任务契约。若 Hermes 的生产运行能力不合格，R0 必须进入 blocked 并新增 Runtime ADR；OpenClaw 是候选 B，但需建立独立适配分支并重新通过 R0，不能声称透明切换。`ai-monitor-core` 的领域模型和测试保持可移植，生产 Runtime 适配器允许变化。

## 2. 产品目标与 MVP 边界

### 2.1 目标

构建一个面向中文 AI 从业者的单用户资讯工具，自动完成：

- 获取可信 AI 最新资讯；
- 标准化标题、来源、作者与时间；
- 过滤无关内容并合并重复事件；
- 生成有证据约束的中文摘要和分类；
- 通过可替换的 Query/Notification Edge 消费；MVP 由 Open WebUI/PWA 承载查询、追问和反馈，由 Email 承载日报与异常提醒，边界见 2.4；
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
- Open WebUI/PWA 中的查询、来源/主题筛选、追问和显式反馈；
- 通过 Hermes Agent 生成中文纯文本日报，由 Core notification outbox 冻结，再由独立 Hermes mailer 投递到固定地址；
- Hermes Cron 驱动定时采集、处理、日报和业务健康巡检；
- 生产 `make deploy` 一条命令 reconcile 四个长驻 Compose 服务（Hermes scheduler、Hermes chat、edge proxy、Open WebUI）与宿主 `ai-monitor-mailer.timer`；timer 每 2 分钟启动 Compose 中声明的 one-shot mailer service。`ai-monitor-core` 是安装在派生 Runtime 镜像中的包，不是独立服务；chat/scheduler/mailer 通过同一 repository/锁协议访问唯一 Core DB。生产 HTTPS/私有入口是 P0 中已配置的宿主 ingress，不由应用栈自动申请域名或证书。

### 2.3 明确不做

- 通用网页爬虫、登录墙、验证码绕过；
- 全量社交媒体抓取；
- 多租户、复杂权限和原生移动 App；
- 训练个性化推荐模型；
- Kafka、Kubernetes、微服务等超前基础设施；
- 首版自研资讯前端、聊天 UI、调度器或通知渠道；Open WebUI 作为固定版本的现成界面使用，不在 MVP 内 fork 或深度定制；
- 让 LLM 直接执行文章 upsert、去重判定落库、checkpoint 推进和投递幂等。

### 2.4 Query Edge + Notification Edge

MVP 已确认采用一个可逆组合：**Open WebUI/PWA 是 Query Edge，Email 是 Notification Edge；二者都不是业务数据库或 Hermes 运维控制台**。文章库、检索索引、订阅偏好、已生成日报、notification outbox 和投递状态仍由 Core 保存。Open WebUI 只保存账号、界面设置与对话展示历史，邮箱只保存外部消息副本；删除任一侧数据都不能破坏 Core 的业务正确性。

| Edge | MVP 职责 | 明确边界 |
|---|---|---|
| Open WebUI/PWA Query Edge | 查询最近资讯、按来源/主题筛选、展开稳定 ID、追问证据、提交可回滚偏好 | 不消费 `ChannelEnvelope`，不承载采集状态、任务编排、来源配置、模型/skill 管理或发布运维 |
| Email Notification Adapter | 每天 08:30 日报、重要业务异常和来源/处理 dead-letter 提醒 | 使用 SMTP 专用账号、纯文本 UTF-8；不启动 Email Gateway/IMAP，mailer/delivery 自身故障走独立 incident 链路，不把已接受或已读当作业务真相 |
| Core notification contract | 把逻辑 target、canonical payload 与 delivery evidence 映射到具体通知渠道 | 不在领域模型中出现 SMTP 地址或未来 Weixin/Telegram/ntfy 的平台字段 |

每封日报都带稳定 `digest_id/article_id`、原文与证据 URL，以及 Open WebUI 入口地址。用户在 Open WebUI 中可以问“最近 24 小时最重要的 5 条”“只看开源模型”“展开 A-123”“为什么这两条被合并”；只有 Core 工具写入成功后，Agent 才能确认偏好已保存。后续追问必须按稳定 ID 重新查询 Core，不能依赖 Open WebUI 或 Email 会话记得日报。

MVP 不从 Open WebUI 或 Email 管理来源、Cron、模型、skills、更新或部署。Open WebUI 的文件/RAG、任意 Web、代码执行、工具导入和公开分享能力不进入产品路径；Hermes API server 背后的 `news-chat` principal 仍只暴露白名单 Core MCP，因此前端管理员权限也不能扩大 Agent 工具面。Email 仅由 `prod-mailer` 调用固定版本 `hermes send --to email:<fixed-address> --file - --json` 发出；不启动 Email Gateway、IMAP 或入站会话。需要邮件回复对话时，必须新增独立受限 chat profile 并重跑渠道 R0，不能直接给 scheduler 或 mailer 开放入站。

PWA 首版只利用 Open WebUI 已有的可安装 Web 体验；日报和故障提醒仍以 Email 为准，不把浏览器 Push 设为上线依赖。未来替换为 Weixin、Telegram、ntfy 或自研只读资讯页时，只新增/替换对应 Query Edge 或 Notification Adapter 并重新执行对应 P0/R0；不修改 Core 领域模型、digest 状态机或来源管线。

## 3. 三层架构

```text
实现控制平面

Git 规格 / AGENTS.md / 任务契约 / Golden Evals
   ↓
Hermes Kanban → Profiles → Worktrees → Verifier → Integrator
   ↓
持续保持绿色的主分支


生产 Agent Runtime

用户 ↔ Browser/PWA ↔ Open WebUI ↔ edge proxy ↔ Hermes prod-chat
                                               └── mcp-core-chat

Hermes prod-scheduler ── Cron / no-agent / fresh agents ──┐
                                                          ├── unique Core DB / notification outbox
systemd timer ── one-shot prod-mailer ── claim/attempt ── hermes send ───────┘                 ↓
                                      └── SMTP ──→ Email inbox ──→ 用户

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

Hermes 生产面固定拆成两个长驻 Agent profile/process/container，再加一个 externally scheduled one-shot sender principal：`hermes-prod-chat` 只提供查询 API，`hermes-prod-scheduler` 只提供采集/处理/摘要 Cron，宿主 `ai-monitor-mailer.timer` 每 2 分钟启动一次 `hermes-prod-mailer` 容器。mailer 不运行 Agent/Gateway/Cron，只运行版本化 wrapper；wrapper 通过 Core CLI claim/mark/resolve attempt，并调用固定 Hermes CLI 发送。三者使用相同固定 Runtime 制品和 Core 契约，但不共享 session、密钥或控制面；chat/scheduler 的 `HERMES_HOME` 分别持久化，mailer 每次获得新的临时 `HERMES_HOME`、只读 desired state 和 SMTP secret。准确行为必须在 R0 以固定 tag 的文档、CLI、源码快照、systemd unit/timer、进程树和故障注入冻结，不能用当前在线文档替代版本契约：

- **API server + Open WebUI**：`prod-chat` 以固定 Bearer key 在 backend-only network 暴露 OpenAI-compatible Chat Completions API；edge proxy 只放行 `/v1/models` 与 `/v1/chat/completions`。Open WebUI 提供单用户账号、会话管理和 PWA 界面；Hermes API 不发布到宿主或公网，Open WebUI 经 HTTPS/私有访问层对用户开放；
- **Standalone Email sender**：`prod-mailer` 不启动 Email Gateway/IMAP，只显式启用 Email platform，并用固定 `hermes send --to email:<address> --file - --json` 走 SMTP standalone sender。固定 tag 的 Cron tick 依附 Gateway，因此 mailer 禁止运行 Hermes Cron；stock Cron Email delivery 也不具备本项目需要的 durable attempt 语义，禁止使用 `deliver: email` 承担业务通知；
- **Cron**：仅在 `prod-scheduler` 中承载周期/单次任务、暂停/恢复、手动触发与执行历史；定义保存在 scheduler 的 `jobs.json`，执行 ledger 保存在 scheduler 的 `executions.db`；
- **no-agent jobs**：无需推理的采集、对账和业务健康检查在 scheduler 中运行版本化脚本，不产生模型调用；脚本只能来自 scheduler 的 `$HERMES_HOME/scripts`，使用净化后的环境；
- **fresh agent sessions**：事实抽取、事件摘要与日报在新会话中运行，固定模型/provider、attached skills、timeout 和最小工具面；
- **preflight 与 model drift guard**：在调用模型前验证 provider key、skill、Core 契约和固定模型；mailer 另行验证冻结 target/route，任一配置漂移都 fail closed；
- **provider fallback/credential rotation**：只允许版本化的批准集合，不能静默切到未知模型；
- **执行与发送证据**：scheduler Cron `executions.db` 只证明 job attempt；Core 创建的 `delivery_attempt_id`、mailer 进程证据、`hermes send --json` 输出与退出码共同描述一次发送，且同步 success 也只证明 sender path 接受。MVP 不使用入站消息回复，因此不把 Gateway reply delivery ledger、3 次/24 小时补投或 `Recovered reply` 当作通知能力；
- **Skills/MCP**：把选题标准、摘要格式和冲突处理版本化；Core MCP 关闭 sampling、prompts 和 resources，只用 `tools.include` 正向白名单暴露最小 Schema 工具，`exclude` 不作为安全边界；
- **Memory/Session**：只保存对话上下文和轻量经验，不保存文章、checkpoint、偏好、digest 或投递真相。

Open WebUI 默认会把 connection 与权限配置持久化到自身数据库，因此生产固定使用 `ENABLE_PERSISTENT_CONFIG=false` 的环境驱动模式，Git 保存所有非秘密 desired state，secret mount 注入 proxy credential。G0 必须在 v0.11.0 实测该模式能覆盖本计划要求的 auth、signup、secure cookie、API keys/direct connections、files/RAG、code execution、Web search/webhooks、sharing、sub-agents/automations 和 dynamic install 开关，并从用户可见 API/UI 读取实际状态做 drift 检查，禁止绕过应用直接改 SQLite。任何必需项不能由该模式可靠冻结时，G0 fail closed，先提交新的前端配置 ADR，不能临时退回“只在首次启动生效”的环境变量。

Hermes 每分钟调度 tick 有锁，避免同一批次双启；但 scheduler 在任务运行中崩溃时，该 Cron attempt 会被标记为 `unknown`，且不会由 Cron 自动重放。这是计划接受并显式补偿的语义：ingest 在下一 tick 按 checkpoint 幂等恢复，enrichment 等 lease TTL，digest 在 notification 尚未被 mailer claim 时可按同一 window 恢复同一 envelope。mailer 的 attempt 若停在 `claimed` 且能证明尚未启动 sender，可在 lease 到期后有限重试；进入 `sending` 后若进程中断、超时或证据丢失，Core 记 `unknown`，由独立事故链路告警且不自动重发。任何任务都不得假设 exactly-once。

Hermes scheduler 的 Cron execution ledger、mailer runner log、`hermes send --json` 输出与退出码是短期运行证据；sender 成功只代表 send path accepted，不代表到达收件箱或用户已读，external message ID 仅在固定 sender 实际返回时作为可选 evidence。Open WebUI 的会话记录同样不是业务或 delivery 真相。关联摘要和哈希必须每日导出到 Core audit 表或 CI artifact，保留至少 90 天。项目只补充来源新鲜度、pending 队列、摘要失败率、digest 延迟和投递异常等业务指标，不另造通用 Agent 调度器。

生产任务拓扑：

| Job | 类型 | 默认频率 | 职责 |
|---|---|---|---|
| `ingest-due-sources` | no-agent | 每 15 分钟 | 由只读脚本调用固定 Core CLI 获取到期来源，成功静默，失败告警 |
| `enrich-pending-items` | fresh agent | 每 15 分钟 | 通过 MCP lease 待处理条目，抽取事实、分类、聚类建议并结构化提交 |
| `prepare-daily-ai-digest` | fresh agent | 每日 08:30 `Asia/Shanghai` | 查询前一窗口候选，将不可变中文 payload 提交到 Core，并确定性 enqueue `daily_digest` notification；只写 local audit/outbox |
| `enqueue-business-alerts` | no-agent | 每 5 分钟 | 将来源停滞、处理积压等业务异常归并为幂等 notification outbox item；不处理 mailer/delivery 自身失败 |
| `mailer-tick` | systemd timer + one-shot wrapper | 每 2 分钟 | 先 reconcile abandoned attempts，再按冻结优先级逐个 CAS claim；每 tick 最多 10 条/90 秒，每次 sender 最多 20 秒，调用 `hermes send --to email:<address> --file - --json`、解析结果并退出 |
| `recover-unknown-work` | no-agent | 每 5 分钟 | 释放过期 lease，恢复可证明未发送的任务；不盲目重发 `unknown` 消息 |
| `runtime-health-watch` | no-agent | 每 5 分钟 | 只检查 Core、来源停滞、积压和 Cron 异常；不能证明承载它的 Runtime 自身存活 |
| `export-runtime-audit` | no-agent | 每日 02:00 | 导出 run/delivery 关联摘要与哈希，执行 90 天保留策略 |

业务来源/处理告警可以进入 Email notification outbox；mailer、SMTP、notification delivery 或 outbox 自身故障绝不能再递归创建同渠道 `ops_alert`。MVP 默认的独立 incident adapter 使用托管 Healthchecks.io Pinging API，由宿主对三个预创建 check 发送 HTTPS 信号：`runtime-deadman` 接收固定心跳并以 missed heartbeat 发现整机/监控失联，`mailer-tick` 接收每次 one-shot 的 `start/success/fail`，`delivery-integrity` 由只读 Core probe 在存在逾期/未处置 `unknown` 时保持 failure，只有充分证据或显式处置后才恢复。check UUID URL 是 secret，`rid` 使用每次运行 UUID，POST body 只含稳定 incident key、时间和脱敏错误分类。Healthchecks 的 check state 负责同类故障合并，重复提醒策略在 P0 冻结；默认通知使用 Healthchecks 托管基础设施直接发给 owner Email，不经过本项目 SMTP 账号或 mailer。协议通过版本化 `IncidentSignal`/adapter contract 隔离，未来可替换 PagerDuty、Better Stack 或自托管到独立故障域的兼容服务。incident 链路不可用时生产 Gate 失败。

Notification 调度策略也属于固定契约：`daily_digest` 优先级 100 并保留独立容量，`ops_alert` 优先级 50 且按 business key 合并；每次 claim 都重新按 `priority DESC, enqueued_at ASC, notification_id ASC` 选择，不能预先 claim 整批低优先级任务。非 digest backlog 硬上限 100，超限时只合并已有业务告警并触发 `delivery-integrity`，不能挤掉日报。优先级、batch、wall-time 和 backlog 上限均保存在 `config/channels.yaml`，修改后需要 verifier 与 R0。

所有无人值守 agent job 必须固定 provider/model、批准的 fallback、timeout、skills 和 toolset。模型或 provider 配置漂移时 fail closed。Hermes 不被假定提供可靠的 per-job 金额上限；成本由 Core lease 数量、模型输入/输出上限、provider 账户额度和独立 usage meter 共同约束。

生产 scheduler 固定 `cron.wrap_response: false`，但 Email 正文不经过 Cron delivery；`mailer-tick` 从 Core 读取 canonical UTF-8 plain text，并固定通过 stdin + `--file -` 原样交给 `hermes send`。mailer wrapper 对子进程施加固定总超时、捕获 JSON/退出码并禁止 shell interpolation。Email standalone sender 负责 RFC/MIME、CRLF 和传输编码；R0 解码收到的 `text/plain` 后按固定换行规范化规则与 canonical payload 比较。delivery evidence 记录 attempt ID、route/renderer version、sender JSON/退出码、可选 external IDs 和 canonical hash；稳定 ID、事实、URL 与正文语义必须等价。生产 scheduler job 不使用 `context_from` 表达依赖：它只读取上游最近一次成功输出，不等待同一 tick 的上游完成；所有依赖都通过 Core queue/lease/window/outbox 状态表达。

首版 skills 全部存放在仓库并只读同步到 Hermes 生产 skill root：

- `ai-monitor-editorial`：相关性、重要度、事实、证据和冲突处理；
- `ai-monitor-digest`：日报选择、排序、格式和长度；
- `ai-monitor-qa`：Open WebUI 问答、历史搜索和用户反馈解释。

任何 Agent 提议的新 skill 或修改只能进入候选区，经 verifier 审查和 Git 提交后才能用于生产。生产 principals 禁用 `skill_manage`、文件写入和配置写入；不需要 skill 的角色以 `--no-skills` 启动，需要 skill 的角色只读挂载自己的批准集合，不能加载用户目录或同名高优先级副本。

Hermes、Open WebUI、Email 集成、Python 依赖和所有插件都在 lock/镜像中精确固定；R0 不允许启动时自动升级、动态安装 skill 或从生产写回 Git。

Core MCP 按 principal 注册三个独立 alias：`prod-chat` 只注册 `mcp-core-chat`，`prod-scheduler` 只注册 `mcp-core-enricher` 与 `mcp-core-digest`。每个 alias 均固定 `sampling.enabled: false`、`tools.prompts: false`、`tools.resources: false`，并只配置自己的 `tools.include`；生产不注册通配 MCP server，也不依赖黑名单兜底。

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
- `reserve_digest_window`：以唯一时间窗口和 lease 创建或恢复 draft，并返回稳定 `digest_id`；
- `commit_digest` / `get_digest_by_window`：保存并读取不可变 rendered payload、内容哈希和来源集合；
- `enqueue_notification`：Core-only/CLI-only，以 `kind + logical_target + business_key` 幂等创建 outbox item；`commit_digest` 在同一事务中创建 `daily_digest` notification，业务来源/积压巡检可创建 `ops_alert`，但 delivery/mailer 自身故障禁止走同一 outbox；
- `claim_ready_notification`：mailer CLI-only，以 CAS/lease 取得冻结的 `ChannelEnvelope`，生成稳定 `delivery_id = H(notification_id, route_revision)` 与唯一 `delivery_attempt_id`，创建 `DeliveryAttempt(claimed)`；
- `mark_delivery_sending`：mailer CLI-only，在 spawn `hermes send` 前以 lease token 把 attempt 原子推进为 `sending`；
- `resolve_delivery_attempt`：mailer CLI-only，记录 `accepted / failed / unknown` 与 sender evidence；`unknown` 只允许被后续更强证据单向提升为 `accepted/failed`，不会自动重发；
- `reconcile_abandoned_delivery_attempts`：mailer CLI-only，过期 `claimed` 在能证明未 spawn 时按预算回队，过期 `sending` 只能收敛为 `unknown`；
- `source_health`：返回可机器判断的健康状态。

通知渠道替换以版本化 `ChannelEnvelope` 为边界：`schema_version`、`notification_id`、`kind`、可选 `digest_id/business_ref`、`delivery_id`、`delivery_attempt_id`、`logical_target`、`route_revision`、`adapter_kind`、`renderer_version`、`content_type`、可选 presentation hints、`canonical_body` 与 `canonical_hash`。route 在 enqueue 时冻结；pending/retry 期间修改 `config/channels.yaml` 不会让同一 delivery 静默换渠道。Runtime adapter 只能返回归一化的 `accepted / failed / unknown`、attempt ID、route/renderer version 和可选的 namespaced external evidence parts；SMTP 地址、平台 token、chat ID 和分片规则只存在于 Runtime 配置与 secret 中。新增 Weixin/Telegram/ntfy adapter 必须复用同一 envelope、outbox 和 attempt 状态机，因此不需要迁移文章、事件、digest 或偏好表。

业务使用三个显式状态机。`Digest` 为 `draft → ready`；`Notification` 是聚合状态：`queued → delivering → accepted | failed | unknown`，只有 attempt 明确 retryable 且预算未耗尽时才从 `delivering` 回到 `queued`，`unknown` 只允许被更强证据提升为 `accepted/failed`。`DeliveryAttempt.state` 只有 `claimed | sending | accepted | failed | unknown`；另设正交字段 `phase = before_spawn | spawned | sender_returned | evidence_persisted`、`retryable: bool` 与 `error_class`。`claimed` lease 过期且有证据证明未 spawn 时写 `state=failed, phase=before_spawn, retryable=true`；`sending` 过期或 spawn 后证据不全时只能写 `state=unknown, phase=spawned, retryable=false`。每个 attempt 使用唯一 ID、lease token、开始/结束时间与 append-only evidence；除文档化的 `unknown → accepted/failed` 单向提升外，已决结果不可覆盖。默认日报窗口是相邻两个 `Asia/Shanghai` 08:30 的左闭右开区间，落库前转换为 UTC。`reserve_digest_window` 先返回稳定 `digest_id`；模型调用 `commit_digest` 时，Core 在同一事务中保存只生成一次的 payload 并创建稳定 `notification_id`。mailer claim 后只读取冻结 envelope 并同步调用 `hermes send`，任何明确失败重试都复用同一 canonical body、route 和 renderer。若缺少能够区分“明确失败”和“可能已发送”的证据，状态只能保守记为 `unknown`，不得伪造 `accepted` 或自动重发；sender path 接受不表示收件箱送达或用户已读。

LLM 不直接连接数据库，不自行拼 SQL，不推进 source checkpoint。工具调用失败或摘要模型不可用时，原始资讯仍入库，处理状态保留为 pending，稍后可安全重试。

CLI 与 MCP 的并发写入必须经过同一 repository 层、SQLite busy timeout 和跨进程 ingest lock。若 R0 证明这种单机共享数据库拓扑无法满足恢复或锁竞争目标，再将 Core 提升为独立服务或迁移 PostgreSQL。

`prod-chat` Hermes state、`prod-scheduler` Hermes state、唯一 Core DB 与 Open WebUI data 使用四个独立持久卷，分别执行一致性备份；mailer 只使用从 Git 只读挂载的配置、受限 SMTP secret 与 tmpfs scratch，所有业务 attempt/evidence 写回 Core，不拥有需恢复的本地状态卷。provider key、真实 Hermes API Bearer key、Open WebUI-to-proxy credential、Open WebUI secret 和 SMTP app password 使用按容器拆分的受限 secret mount。Open WebUI data 会持久化 server-side proxy credential，必须按敏感状态加密备份、限权恢复并支持轮换，不能把它当普通 UI 缓存。Core DB 使用 SQLite backup API/WAL checkpoint 每日备份并保留至少 7 个日快照；Hermes 备份 chat/scheduler 各自的 config/profile/session，以及 scheduler 的 `jobs.json`、`executions.db` 和 skills 版本引用；Open WebUI 备份账号、连接配置与会话数据库，并在配置变更与升级前额外生成加密快照。除非固定版本验证了安全的在线备份方法，否则先 quiesce 对应写入进程再快照；禁止直接复制正在写入的 WAL 文件。R0 和发布 Gate 都必须校验 checksum，并从备份恢复到全新卷验证旧镜像/schema 可回滚，不能只验证“备份命令成功”。

### 3.4 Runtime 选择

| 候选 | 结论 | 理由 |
|---|---|---|
| Hermes Agent | **实现与生产默认** | 单一成熟 Agent 栈覆盖 Kanban、Cron、standalone Email sender、Skills、MCP 与 OpenAI-compatible API；Core 补偿其 Cron/send `unknown` 语义，显著减少双栈集成与运维成本 |
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
| Cron execution 与 sender 进程证据 | Hermes scheduler / mailer logs | 短期运行证据；每日导出关联摘要与哈希，业务终态仍写 Core |
| 文章、checkpoint、事件、digest、notification outbox 与 delivery 状态机 | AI Monitor Core DB | 唯一内容业务真相，和 Hermes Memory 隔离 |
| Incident check 状态 | Healthchecks.io + 脱敏本地审计 | 独立故障域的告警证据，不替代 Core 业务状态；Ping URL 不进入日志/Git |
| 用户主题偏好 | Core DB / 版本化配置 | 可由 Agent 修改，但必须通过受约束工具 |
| Agent memory | 非真相来源 | 只保存对话习惯和经验，清空后不能影响业务正确性 |

Hermes 开发看板与生产 Cron 都应能由 Git 中的模板重建，避免任何单机 Agent 状态成为唯一不可恢复资产。清空生产 Memory/Sessions 或重建 chat/scheduler/mailer 后，Core 数据和业务状态必须保持完整。

## 5. H0/P0/R0：实现与生产 Agent 资格测试

Hermes 迭代频繁。同一固定版本必须分别通过 H0（实施实例）与 R0（生产实例）；H0 绿色不能替代生产调度、通道和安全验收。

### 5.1 H0：实现驱动资格

在临时 Git 仓库完成：

1. 固定并记录 Hermes release/tag、安装来源和校验信息；
2. 运行诊断，确认模型、terminal backend、Kanban、profiles 和 CLI 可用；
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

- 已创建的专用 SMTP 发送账号与 owner 收件地址；按固定 tag schema 冻结 `EMAIL_ADDRESS`、`EMAIL_PASSWORD`、`EMAIL_SMTP_HOST`、兼容 STARTTLS 的 `EMAIL_SMTP_PORT`，并在 mailer 的只读配置中显式设置 `platforms.email.enabled: true`。不配置 `EMAIL_IMAP_HOST`、不启动 Email Gateway；mailer 只允许 `hermes send --to email:<固定地址> --file - --json`，目标不能靠 home channel 或会话记忆猜测；
- 固定 `ghcr.io/open-webui/open-webui:v0.11.0` manifest digest、一次性 break-glass admin bootstrap、关闭开放注册的方法、单独非 admin 产品账号、外部 HTTPS 或私有访问地址、备份/撤销方式；产品账号只见唯一 Hermes model，不启用 sub-agents、automations、files/RAG、Web search、代码执行、skills/tools 导入、公开分享、notification webhook 或其他 provider connection；启动阶段禁止从公网下载 embedding/model，若禁用相关能力仍会下载，则在构建阶段预取并固定制品哈希；
- Hermes `API_SERVER_ENABLED=true`、真实 `API_SERVER_KEY`、backend-only 监听地址，以及 Open WebUI 指向 edge proxy 的 `OPENAI_API_BASE_URL`/独立 proxy credential；Open WebUI 只加入 frontend network，Hermes chat 只加入 backend network，edge proxy 同时加入两者并注入真实 Bearer key，两个 key 均按 secret 轮换；
- 已授权的模型 provider、精确模型、账户额度与 secret 注入方式；
- 默认 Linux/systemd 的 Docker Compose 目标主机、允许安装受版本控制 `ai-monitor-mailer.service/.timer` 的部署权限、满足 Hermes 与固定 Open WebUI 镜像实测要求的内存、四个隔离持久卷、mailer tmpfs/config/secret mount，以及独立于 Hermes/Open WebUI/Email 的 dead-man/incident 告警目标；其他宿主必须先提供等价 supervisor/timer adapter；
- 托管 Healthchecks.io 项目中预创建的 `runtime-deadman`、`mailer-tick`、`delivery-integrity` 三个 checks、各自 secret Ping URL、period/grace/repeated-notification 设置，以及由 Healthchecks 托管基础设施直接发送给 owner Email 的默认通知 integration；该 integration 不复用本项目 SMTP credential。宿主网络只允许访问 `hc-ping.com:443`，生产不允许 auto-provision slug，撤销时轮换/删除 UUID；
- 每项的只读验证命令、授权人/系统和撤销方法。

这些是外部授权输入，不是人类工程开发任务。缺少任一必需项时标记 `external_prereq_blocked`；Agent 继续完成 G0–G4 的离线工作，但不得伪造凭据或宣称 R0/G5 已通过。

### 5.3 R0：Hermes 生产 Runtime 资格

使用 G4 已验证的 Core、固定 Hermes/Open WebUI Runtime 和专用测试邮箱完成：

1. 固定 tag 的 `hermes cron --help`、`hermes send --help`、API/config schema、相关 sender 源码和 tag 内官方文档已保存哈希；scheduler 重启后 `jobs.json`、下一次运行时间、execution ledger 和失败记录可恢复，chat 重启后 API profile/session 可恢复且仍不获得 scheduler/mailer 权限；宿主重启后 `ai-monitor-mailer.timer` 恢复调度，one-shot mailer 只从 Core outbox/attempt 恢复，不依赖本地 ledger；
2. scheduler no-agent cron 只能运行只读挂载到 scheduler `$HERMES_HOME/scripts` 的批准 wrapper；realpath/哈希匹配 Git，成功可静默、失败写入业务告警或独立事故链，且模型调用数为零；
3. scheduler no-agent 子进程不包含模型、SMTP、Hermes API 或 Open WebUI 凭据；mailer 不持有模型/API key，只能从隔离 config/secret 读取固定 SMTP 配置并调用 `hermes send`。Core 真正需要的非模型 secret 只能通过显式白名单环境或只读 secret file 注入；
4. agent cron 在 fresh session 中只加载批准 attached skills 和 `enabled_toolsets`；三个 Core MCP alias 都关闭 sampling/prompts/resources，并只暴露各自 `tools.include`；
5. MCP 的读取、lease、结构化写回和幂等键均能端到端工作；任何未列入白名单的 tool/resource/prompt/sampling 请求失败；
6. 非 admin 产品账号经 HTTPS/私有入口完成登录、PWA 安装和流式查询；Open WebUI 只能访问 edge proxy，不能路由到 backend network；proxy 只向 Hermes chat 转发 `/v1/models` 与 `/v1/chat/completions` 并注入真实 Bearer key。真实 key 不进入 Open WebUI/浏览器，未认证请求、其他 API path 和非允许源失败；
7. Open WebUI 产品账号只展示唯一 `news-chat` model；sub-agents、automations、memories、files/RAG、Web search、代码执行、skills/tools 导入、公开分享、notification webhooks 与额外 provider connection 在 UI/API 均不可用。break-glass admin 不作日常账号；即使前端配置漂移，Hermes `news-chat` principal 仍只能调用 `mcp-core-chat` 白名单工具；
8. mailer 的固定 entrypoint/process tree 只有只读版本化 wrapper、Core CLI 子进程与短生命周期 `hermes send --to email:<固定地址> --file - --json` 子进程；不存在 Hermes Gateway/Cron/Agent/IMAP poller。wrapper 的 realpath/hash 匹配 Git，并固定 stdin 调用、总超时、参数数组和动态 target 拒绝规则；
9. 重复回放同一 Cron execution evidence 或同一 digest window 时不产生第二份业务 digest；重试读取相同 rendered payload；
10. 模型超时、`429`、SMTP/API/proxy 凭据失效、chat/scheduler/mailer 或 Open WebUI 崩溃不会丢失 Core 队列/outbox；
11. reconcile guard 检测到固定模型、工具策略、MCP alias、Open WebUI connection、proxy route、mailer entrypoint/target/route revision 或 scheduler job 模板漂移时任务 fail closed 并告警；
12. Prompt Injection fixture 不能获取 shell、文件、Cron 控制面、密钥、Open WebUI 管理面或额外外发权限；
13. 清空 Hermes Memory/Session、Open WebUI 会话或收件箱副本后，文章、偏好、checkpoint、digest、notification outbox 和 delivery 状态保持不变；
14. 只读挂载和 tool deny 能阻止生产 agent 修改 skills、scripts、配置或运行时控制面；Hermes profile 与 Open WebUI 账号隔离都不被当作 sandbox；
15. Hermes 或 Open WebUI 升级先在 canary 环境恢复状态快照并完整跑 R0；失败可回退到各自固定 manifest digest、dev/prod 派生镜像 digest 与备份状态；Open WebUI 冷启动不得动态下载未固定的 embedding/model/插件；
16. 强制终止运行中的 no-agent 与 agent job，确认遗留 Cron attempt 记为 `unknown`，并按类型补偿：ingest 等下一 tick 幂等恢复，enrichment 等 lease TTL，尚未进入 `sending` 的 digest/notification 可按稳定 business key 恢复；mailer 在调用 send 之后的模糊结果记 `unknown`、告警且不自动重发；
17. 连续制造失败，确认项目告警、有限重试/补偿和 reconcile 熔断符合版本化策略，不把 Hermes 未承诺的自动重放当成 Gate；
18. 从 Core DB、Open WebUI data、chat/scheduler Hermes state、scheduler `jobs.json`/`executions.db`/output/profile/session 和独立 secret mount 恢复到全新卷；从 Git 只读配置 + 新 tmpfs 重建 mailer，确认内容、账号/连接、调度、outbox 和未完成任务可继续收敛；
19. 捕获 chat/scheduler/Open WebUI/edge proxy 的真实 entrypoint 与 process tree；终止进程、制造 hang 与 health failure，确认 Compose restart policy/宿主 systemd 在 2 分钟内重启对应长驻容器。另行停止/禁用 timer、模拟宿主重启、重复和并发触发，确认 `Persistent=true` 的 timer 恢复调度而 Core CAS 只允许一个 mailer attempt；固定镜像内 s6 仅在实际监督目标进程且通过故障注入时作为额外防线；最后禁用宿主恢复与 timer，确认独立 dead-man/incident 告警触发；
20. 注入 before-spawn、after-spawn/before-SMTP-result、after-send 和 evidence-lost 故障；只有 `claimed` 且能证明未 spawn 的 attempt 可按同一 envelope 有限重试，明确 sender failure 记 `failed`，同步 sender success 记 `accepted`，其余模糊结果记 `unknown` 且不自动重发。sender JSON/退出码、`delivery_attempt_id`、route/renderer version 与 canonical hash 必须关联；external IDs 只有实际返回时才记录；
21. `mailer-tick` 读取的 Core canonical text 与通过 `--file -` 交给 `hermes send` 的 stdin 逐字节一致；按固定 MIME/CRLF 规则解码 Email `text/plain` 后，保留相同 `notification_id`、可选 `digest_id`、事实、URL 和 canonical hash；
22. Runtime 内 `python --version`、no-agent wrapper 的 `sys.executable` 和 Core lock 均为冻结的 Python 3.13 组合；
23. `platform_toolsets.api_server` 的最终集合只有 `mcp-core-chat`；生产镜像没有未批准 plugin/global MCP。R0 捕获一次真实 chat request 中 Agent 实际收到的完整 tool schema 并与 Golden allowlist 比较，不能只检查配置文件或 `/v1/models`；
24. edge proxy 对 body size、并发、每用户速率、总运行时、SSE idle timeout 和上游响应大小执行固定上限；并行新建/切换/删除多个 Open WebUI 对话不会串 Hermes session，删除前端对话不改变 Core；
25. `ENABLE_PERSISTENT_CONFIG=false` 下重启 Open WebUI，所有安全配置仍与 Git desired state 一致；普通产品账号不能获得 admin/API key/direct connection 权限，break-glass admin 的登录与变更留下审计记录。
26. 注入 SMTP 失效、`mailer-tick` crash 和 delivery `unknown`，确认不会向同一 Email outbox 递归创建 `ops_alert`；宿主只向独立 incident target 发送一个稳定 incident key，重复探测被抑制，恢复事件可关联原 incident。
27. 在 100 条已合并业务告警积压下，于 08:30 enqueue 日报；每次 claim 均遵守冻结优先级/稳定排序，日报在首个可用 tick 被处理并满足 15 分钟 SLO，tick 不超过 10 条或 90 秒，低优先级任务未被提前批量 claim；
28. fake incident sink 验证 `IncidentSignal` Schema、脱敏、稳定 check key、run UUID 与重复抑制；Live R0 对三个 Healthchecks checks 各跑 start/success/fail/missed-heartbeat，确认只访问 `hc-ping.com:443`、secret URL 不入日志、HTTP body/状态严格校验且实际通知 integration 可达；
29. 普通 `docker compose up` 不会启动 mailer；one-shot service 位于非默认 profile、`restart: "no"`、无端口且并发调用仍受 Core CAS 约束。`make deploy` 先停止 timer、校验/备份/迁移 Core DB，再启动四个长驻服务并等待真实 readiness，最后才安装/daemon-reload/enable timer；任一步失败都保持 timer disabled 并执行版本化回滚。

R0 通过条件：真实 Open WebUI/PWA 与专用 Email 测试通道完成至少一个查询、采集、处理、日报周期，重启和失败注入均留下可审计记录。

若 H0 失败，改用 Codex 或 Claude Code 按同一任务契约执行。若 Hermes R0 失败，项目在生产 Runtime ADR 处 blocked；OpenClaw 候选必须新增适配分支并重新执行 P0/R0，不能沿用 Hermes Gate 的结论。首版不把 Claude Code/Codex CLI 强行接成 Hermes 外部 worker lane，因为该路径尚不是 Hermes 官方标准化能力。

## 6. Agent 角色设计

| Profile | 职责 | 允许修改 | 禁止事项 |
|---|---|---|---|
| `orchestrator` | 分解目标、建立依赖、分配任务、处理阻塞 | Kanban 任务与评论 | 不写产品代码，不自行验收 |
| `contract-verifier` | 冻结契约、Golden fixtures 和 Gate 测试 | `docs/`、`evals/`、验收测试 | 不实现被测功能，不降低阈值 |
| `core-worker` | 领域模型、repository、迁移、Core CLI/MCP、audit 与 notification outbox | `src/ai_monitor_core/models/`、`repository/`、`cli/`、`mcp_server/`、`audit/`、`alembic/` 与 worker-owned unit tests | 不改公共契约、Runtime 配置或 Gate 测试 |
| `connector-worker` | RSS、arXiv、GitHub、HN 适配器 | connectors、`config/sources.yaml` 与 worker-owned unit tests | 不改来源配置 Schema、公共领域模型和 Gate 测试 |
| `pipeline-worker` | 标准化、去重、聚类、摘要和排序 | pipeline 与 worker-owned unit tests | 不改来源契约和 Gate 测试 |
| `runtime-worker` | Hermes Prod/Cron/skills、mailer/incident transport、edge proxy、Open WebUI 与 Email 集成 | `runtime/`、`deploy/`、`ops/`、`config/schedules.yaml`、`config/channels.yaml`、`config/incidents.yaml` 与 worker-owned unit tests | 不改配置 Schema、Core 领域模型、runtime contract、security 或 Golden |
| `integrator` | 合并已验收提交、解决接口冲突 | 集成文件、迁移、构建配置 | 不替实现卡补功能，不绕过 Gate |
| `security-release` | 安全审查、离线验收、Live Canary | 只读运行 protected Gate；写报告与发布证据 | 不修改测试/阈值，不开发新功能 |

默认并发上限为 3。只有修改路径无交集、公共契约已冻结且依赖已满足的卡才能并行。

生产环境在 Hermes 中划分以下最小权限逻辑 principals。固定版本若不能可靠实施 per-job tool filter，就必须以独立 production profiles/processes 强制隔离，不能只靠 prompt：

| Principal | 职责 | 工具面 | 明确禁止 |
|---|---|---|---|
| `news-chat` | 响应 Open WebUI 查询、解释事件和管理主题偏好 | `mcp-core-chat` 的只读查询/受约束偏好写入；仅 `ai-monitor-qa` skill | shell、文件写入、任意 Web、部署、Cron/邮箱管理 |
| `news-enricher` | 事实抽取、分类和聚类建议 | `mcp-core-enricher` 的 lease/evidence/commit；仅 `ai-monitor-editorial` skill | 任意 SQL、任意 Web、任意外发、修改 skills/memory 权限 |
| `news-digest` | 生成并提交不可变日报 payload；Core 同事务 enqueue notification | `mcp-core-digest` 的 candidate/reserve/commit 工具；仅 `ai-monitor-digest` skill | claim/resolve notification、改历史 digest、任意 Web、通用消息发送、Cron 管理 |
| scheduler no-agent runner（非 Agent） | 采集、异常入 outbox、导出审计和业务健康检查 | scheduler `$HERMES_HOME/scripts` 下只读 wrapper | SMTP/API secret、动态 shell、模型、浏览器；权限由 OS/container 强制 |
| mailer no-agent runner（非 Agent） | claim/resolve notification，调用固定 `hermes send` | mailer 只读 wrapper、Core mailer CLI、固定 SMTP config/secret 和唯一目标 | 模型、MCP、IMAP、动态 target、读取其他 secret、重发 `unknown` |

Hermes 实施 profiles 与生产 chat/scheduler principals 完全分离，不共享持久化 `HERMES_HOME`、会话、Memory、Cron DB、skills 写权限、密钥或高权限工具集；mailer 不是 Agent profile，使用单独容器、临时 `HERMES_HOME` 与只读配置。Profiles 只隔离 Hermes 状态，本身不是 sandbox；真正权限边界仍由独立容器/OS 用户、只读挂载、网络策略和 Core capability 强制。三个批准 skill 位于各 principal 的生产只读 skill root；attached skill、`enabled_toolsets` 与 MCP `tools.include` 的最终集合由 Git desired state 生成并在运行前 preflight，防止同名覆盖和权限漂移。

no-agent runner 不受 Agent prompt 保护。其边界必须由容器/OS 强制：非 root UID、只读 rootfs、只挂载 Core/Hermes 必需卷、drop capabilities、`no-new-privileges`、无 Docker socket、净化环境变量和最小网络出口；mailer 额外只允许到固定 SMTP host/port，scheduler 不允许到 SMTP。R0 从容器外验证越权失败。

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
G1-VERTICAL      固定 Hermes Prod → Core outbox → fake mailer
   ↓
   ├── G2-RSS       官方 RSS/Atom 来源
   ├── G2-ARXIV     arXiv 连接器与限流
   ├── G2-GITHUB    GitHub Releases 连接器
   ├── G2-HN        HN 信号连接器
   ├── G2-FETCH     HTTP 缓存、退避、大小限制、错误分类
   ├── G2-DEDUP     URL、external ID、内容哈希与近重复
   ├── G2-MCP       lease、结构化提交、查询和幂等工具
   ├── G2-OUTBOX    Notification/DeliveryAttempt、Core mailer CLI 与状态机
   ├── G2-SKILLS    分类、摘要、冲突处理和日报 skills
   ├── G2-CRON      声明式 jobs、reconcile 与熔断
   ├── G2-WEBUI     Open WebUI/PWA、Hermes API 与查询入口
   ├── G2-INCIDENT  Healthchecks-compatible incident adapter 与宿主探针
   └── G2-NOTIFY    deterministic mailer 与 Email adapter
            ↓
G3-INTEGRATION     Runtime/Core 集成、失败隔离、崩溃恢复
   ↓
G4-OFFLINE         Core + fake Runtime 的无公网全链路验收 ──┐
                                                           ├─→ R0-RUNTIME
P0-EXTERNAL        外部授权、真实模型/邮箱/Web 入口/主机 ─────┘       ↓
G5-LIVE            固定 Hermes/Open WebUI + 真实来源、模型、Email Canary
   ↓
MVP-RELEASE
```

### G0：契约与仓库基线

产物：

- 当前 Git 基线上的首个实现 commit；
- `AGENTS.md`，记录架构、统一命令、不可绕过规则；
- `docs/adr/0001-hermes-runtime.md`，记录单 Agent 栈、dev/scheduler/chat/mailer 隔离决策、Hermes/OpenClaw 证据、版本和替换边界；
- `pyproject.toml`、锁文件、Makefile、CI；
- `config/sources.yaml`：精确 URL/API、adapter、频率、限流、允许重定向 host、响应上限、条款 URL、owner 与停用策略；
- `config/schedules.yaml`：固定 tag 实测支持的 cron 表达式、容器 `Asia/Shanghai` 时区和 UTC window 规则；日报默认每日 08:30，任务错峰由显式表达式实现；
- `config/channels.yaml`：`owner-digest`、`ops-alert` 等业务逻辑 target 到 adapter/route/renderer revision 的非秘密映射；不保存 SMTP address/password 或 Open WebUI 配置；
- `config/incidents.yaml`：三个固定 check key、period/grace、脱敏 body Schema、网络 host、重复抑制和 adapter revision；只保存 `${SECRET_REF}`，不保存 Ping URL/UUID；
- `SourceConfig`、`FetchedDocument`、`NormalizedArticle`、`EnrichmentResult`、`Digest`、`Notification`、`DeliveryAttempt` 契约及各自状态转移；
- `ChannelEnvelope`、notification outbox、逻辑 target registry 与 normalized multi-part delivery evidence 契约；Email 和 fake adapter 都必须实现同一 contract；
- `IncidentSignal`、Healthchecks-compatible adapter、fake sink、独立故障域和防递归契约；保存固定官方协议页面与响应 fixture 的哈希；
- Core CLI/MCP Schema、lease、checkpoint 和幂等契约；
- Hermes production chat/scheduler profiles/principals、mailer deterministic entrypoint、skills、Cron desired-state 与安全配置模板；
- `v0.20.0 / v2026.8.3` tag 内官方文档 URL/commit、CLI help/config schema 输出及哈希，作为 runtime contract 版本快照；
- Open WebUI `v0.11.0` release commit、镜像 manifest、脱敏后的实际 admin/API 能力快照、适用安全公告清单与哈希；secret 字段只验证存在性/轮换能力，不写入 artifact，在线文档仅作发现资料，不代替镜像实测；
- 固定版本的 Hermes/Open WebUI/standalone Email sender 集成清单、API path allowlist、上游 manifest digest、dev/prod 派生镜像定义与 digest、Python 3.13 runtime contract，以及四个生产状态卷与 mailer 无状态重建的备份/恢复 runbook；
- dev/prod 独立 Compose/config/volume/network/secret 模板；生产 chat/scheduler 各自的 profile、secret 和持久化 `HERMES_HOME`，mailer 使用临时 `HERMES_HOME`、只读 config 和独立 secret；生产 scripts/skills 只读挂载清单与哈希；
- 版本化 `ai-monitor-mailer.service/.timer`、`Persistent=true`、固定工作目录/镜像 digest/one-shot service 名、超时、并发抑制、安装/撤销/状态检查与宿主重启 runbook；Compose mailer service 必须位于非默认 profile、`restart: "no"`、无端口且不被 `compose up`/依赖自动启动；
- `make deploy` 顺序契约：停用 timer → 校验 digest/config/secret reference → 一致性备份 → Core migration → 启动并等待四个长驻服务 ready → 安装/校验/启用 timer → canary；任一步失败保持 timer disabled 并按 runbook 回滚；
- 固定 digest/lock 的依赖与镜像安全扫描，以及每周只读重扫工作流；适用的 Critical/High 立即阻断 release 并发送独立告警；
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
Core notification outbox → fake mailer → fake receiver
```

Gate：

- 同一数据连续导入 3 次，文章数量不增加；
- Core 在无网络、无 LLM Key 时 fixture E2E 通过；
- scheduler/mailer 或模型中断后待处理条目与 notification outbox 仍可恢复；
- 相同 digest window 重放不产生第二份业务 digest；
- fake receiver 收到的每条日报内容都能回到原始 URL，并包含同一 `notification_id/digest_id`；
- 在不迁移 Core DB/领域模型的前提下，把 fake Email adapter 替换为第二个 fake channel，完整 delivery contract 测试仍通过；
- fresh install 只需 README 中一条 `make smoke` 即可在 loopback 完成 fake mailer 的离线产品栈；生产部署使用 `make deploy` 同时 reconcile Compose、systemd timer 与 P0 已授权的 HTTPS/私有 ingress 配置。

### G2：可并行组件卡

G1 通过且公共契约冻结后，来源、Core/MCP、Core outbox、Hermes Prod skills/Cron、deterministic mailer、Open WebUI 和 standalone Email sender 集成分 worktree 并行实现。每个组件使用自己的 fixtures、定向测试和 verifier 卡，不允许并行修改同一公共接口。

### G3：集成与可靠性

集成内容：

- Hermes Cron desired-state reconcile、版本化熔断、运行审计和防重入；
- `ETag`、`Last-Modified` 和增量 checkpoint；
- `429 Retry-After`、有限重试和死信；
- 单来源失败隔离；
- chat/scheduler/mailer、抓取、解析、入库、lease、摘要各阶段的崩溃恢复，以及 ingest/enrichment/digest/notification 各自的 `unknown` 补偿策略；
- 来源、Core、Cron、outbox 和积压健康状态；各 Runtime 自身由外部 supervisor/dead-man 监控；
- digest/notification/delivery-attempt 状态机、不可变 payload、`notification_id/digest_id/delivery_id/delivery_attempt_id` marker，以及 sender evidence 对 mailer attempt 的精确关联；
- Hermes Memory/Session 清空后的业务状态恢复；
- skills/config 版本漂移检测；
- Hermes execution ledger/日志与 Core 业务指标的统一健康视图；
- 运行证据摘要/哈希的每日导出；
- 宿主级 restart/healthcheck、独立 dead-man 告警；
- Healthchecks incident adapter、三个 check 的 missed heartbeat/fail/recovery、secret redaction 与重复抑制；
- 数据卷和 secret path 的一致性备份、保留、恢复演练和升级回滚。

Gate：相同批次运行 3 次和并发运行后 Core 业务不变量一致；每个阶段注入进程终止后，文章/checkpoint/digest/notification 不丢失，delivery audit 在 SLO 内收敛为 `accepted / failed / unknown`。允许通道重复，但只能使用同一 `notification_id`、可选 `digest_id` 与不可变 payload；`unknown` 不自动重发。

恢复以真实 health 为准：Compose restart policy 处理长驻进程退出，宿主 systemd 探测 chat/scheduler/Open WebUI/edge proxy 的实际请求路径并在 hang 或持续 `unhealthy` 时重启对应容器；`ai-monitor-mailer.timer` 的 last-trigger/last-result/next-elapse 与 Core outbox age 单独受监控。固定镜像内 s6 只在 R0 证明它确实监督目标进程时作为额外防线。dead-man heartbeat 从宿主直接发送到独立告警服务，不能经同一 Hermes/Open WebUI/Email 链路；同一 incident key 做抑制，避免告警递归。Runtime 容器本身不挂载 Docker socket。

### G4：Offline Release Gate

在全新、禁止访问公网、没有任何 API Key 的环境中，以 fake Runtime/receiver 执行：

```bash
make check       # format、lint、type、unit
make verify      # 契约、幂等、恢复、Golden eval、安全、fake incident sink
make smoke       # 构建 Core 与 fake Runtime/mailer/incident sink，跑 fixture 全链路
make runtime-contract  # 验证 Hermes/Open WebUI、mailer/systemd 与 incident adapter 契约
```

任一测试被删除、注释、`skip` 或 `xfail` 均视为失败。

### G5：Live Canary Gate

使用固定版本 Hermes Prod/Open WebUI、真实来源、真实模型、专用 Email 测试账号和三个 Healthchecks checks 运行至少 24 小时，覆盖不少于 3 个增量采集周期、1 次 Open WebUI 查询、1 次日报和 1 轮 incident start/fail/recovery：

```bash
make live-smoke
```

Live Gate 禁止 Mock。缺少模型凭据、专用 Email 账号、Open WebUI HTTPS/私有入口、Healthchecks checks/通知 integration、部署授权或付费授权时，状态只能是 `offline_verified`，不能宣称 MVP 已上线。

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
- 重启后不丢数据；相同日报窗口只生成一个 `digest_id`。仅有充分 before-spawn/retryable 证据的 attempt 可以有限重试，并复用同一 `notification_id`、canonical body/hash、route 和 renderer；after-spawn 模糊结果保持 `unknown` 且不重发，所有 attempt 均保留审计记录。

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

- chat/scheduler/mailer 与宿主重启后，scheduler 的 Cron 定义/ledger、chat 的 API profile/session 和 Core 中的 mailer attempt 均按各自契约恢复，不重复生成业务 digest 或 notification；
- no-agent job 的测试证明没有模型调用；
- fresh agent job 只能看到 `enabled_toolsets` 和 MCP `tools.include` 正向批准的 Core 工具，不能直接访问任意 Web；
- 删除 Memory/Session 不影响文章、偏好、checkpoint、digest 和 delivery 状态；
- 模型不可用时采集继续，待处理队列不丢失并产生异常告警；
- Cron prompt、skill、model、provider、CLI/config schema 和 MCP Schema 发生漂移时能够被检测；
- Open WebUI break-glass admin 与非 admin 产品账号、关闭开放注册、双网络/API 隔离、并发多会话、重复请求和连接恢复经过真实入口验证；
- Email standalone sender 的固定 SMTP host/port/STARTTLS、固定收件地址、正文编码、明确失败与 after-send `unknown` 经过真实通道验证；未启动 Gateway/IMAP，也不存在入站或自动回复路径；
- production agent 无法调用 shell、修改文件、创建 Cron、部署或读取秘密；
- no-agent runner 在容器/OS 边界无法提权、访问 Docker socket、宿主文件或未授权网络；
- 任一生产 Runtime 或 Open WebUI 停止/卡死时，宿主 supervisor 负责重启，独立 dead-man 负责告警；同一 Runtime 内的 Cron 不被当作自监控。

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

生产 `news-chat`、`news-enricher` 和 `news-digest` principals 不启用 terminal、文件写入、Git、部署、Cron/邮箱管理、任意 Web 或通用消息发送 toolset，只暴露各自白名单 Core MCP；`news-chat` 只能写当前 API response stream，后台 agent 不能直接外发。Open WebUI 只连接独立的 `news-chat` API profile/process，不能连接调度或实施 profile；Email 只能由固定 systemd-triggered one-shot mailer 外发。scheduler no-agent jobs 只能执行从 Git 制品只读 bind-mount 到 `$HERMES_HOME/scripts` 的批准 wrapper，mailer 只能执行固定 wrapper/Core CLI 与参数数组，不接受由文章正文或模型拼出的脚本、路径或参数。

其他要求：

- 密钥只能由环境或秘密代理注入，不写入任务卡、prompt、Git 或日志；
- Agent 不创建或修改真实 `.env`；
- Open WebUI 只加入 frontend network，Hermes API server 只加入 backend-only network，且不发布宿主端口、不配置浏览器 CORS；edge proxy 同时加入两个网络，只接受 Open WebUI 的 `/v1/models` 与 `/v1/chat/completions`，注入真实 Hermes Bearer key，并限制请求体、并发/速率、总运行时、SSE idle 和上游响应大小。外部反向代理只把 Open WebUI 暴露给 HTTPS/私有访问层，不转发 Hermes Dashboard、Runtime 管理端点或任一内部 key；
- 浏览器只持有 Open WebUI session，Open WebUI server-side 只持有独立 proxy credential；真实 Hermes Bearer key 只存在于 edge proxy 与 `prod-chat`。固定镜像关闭开放注册，首次 bootstrap owner 转为离线 break-glass 管理身份，日常使用单独的非 admin 产品账号和唯一 `news-chat` connection/model。Open WebUI 的 sub-agents、automations、memories、Web search、代码执行、files/RAG、skills/tools 导入、公开分享、notification webhooks 与额外 provider 均关闭，并由 Runtime contract 通过 UI/API 双路径验证；
- Hermes `news-chat` API profile/process 不持有 Cron 管理权、Email app password、provider 之外的运行密钥或通用 platform toolset；`platform_toolsets.api_server` 的最终集合只能包含 `mcp-core-chat`，MCP `tools.include` 也必须是显式正向集合，不能沿用 API server 的默认完整工具面；
- Email 使用专用发送账号和 app password；只有 `prod-mailer` 能读取 `EMAIL_ADDRESS`、`EMAIL_PASSWORD`、固定 `EMAIL_SMTP_HOST`/`EMAIL_SMTP_PORT`，并使用固定 adapter 实测的 STARTTLS 路径。mailer 不配置 `EMAIL_IMAP_HOST`/home channel，不启动 Gateway，不加载 Himalaya skill，只能向配置中冻结的唯一地址发送；scheduler/chat/Open WebUI 均拿不到 SMTP secret；
- Open WebUI 输入、网页正文和通知展示内容都视为不可信；文件、附件、远程图片、HTML、分享链接和客户端生成元数据不能扩大 Agent 工具面或 mailer 外发范围。固定版本不存在的配置键不得由当前在线文档臆造；
- 网页正文始终是不可信数据，不具有指令权限；
- 摘要 Agent 不拥有 shell、部署、任意文件或独立通知凭据；
- 生产 skills 从 Git 同步并设为只读，Agent 自动生成的 skill 必须经过 verifier 才能进入生产；
- 生产 agent jobs 的 `enabled_toolsets`、MCP `tools.include` 和 attached skills 使用显式最终集合；MCP sampling/prompts/resources 关闭，禁用 terminal、配置写入、`skill_manage` 和控制面工具，空或错误的集合必须在模型调用前失败；
- 只有确定性的 deployment reconcile 身份能通过固定 Hermes Cron CLI 管理任务；任何身份都不直接编辑 `jobs.json`，生产 agent sessions 不能创建、修改、删除或手动运行 Cron；
- 只有确定性的宿主 deployment identity 能安装/启停 mailer systemd unit；unit 只能从固定 working directory 启动 manifest digest 已冻结的 one-shot service，命令、超时和参数不可由文章、模型或普通产品账号修改。该宿主身份可以操作容器，因此不暴露给任何 Agent，所有变更必须走 Git desired state、哈希校验和审计；
- incident adapter 只在宿主运行，只能向 `hc-ping.com:443` 发送固定 `IncidentSignal`；三个 Ping URL/UUID 通过 systemd `LoadCredential` 或等价 root-only secret file 注入，禁止写入参数列表、普通环境变量、日志、Git、Core 通知正文或容器环境。POST body 最大 4 KiB，只含允许的 check key、run UUID、时间和枚举错误分类，不含文章、prompt、token、内部 URL 或堆栈；
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
| 日报 | 08:30 后 10 分钟内形成 immutable payload，15 分钟内进入可观测的 `accepted/failed/unknown`；非 accepted 必须由独立 incident 链路告警，不能写回同一 Email outbox；`unknown` 可被后续充分证据单向提升但不得自动重发，持续 15 分钟仍不明则升级事故 |
| 摘要事实 | Golden 中新增的原文外实体、数字或日期为 `0`；Schema/证据 ID 合法率 100% |
| 分类质量 | 冻结 Golden 上 macro-F1 ≥ 0.85；样本或阈值变更只能由 verifier 卡完成 |
| 安全 | Prompt Injection/SSRF fixtures 的越权调用、私网访问和秘密泄露均为 `0` |
| Runtime 恢复 | chat/scheduler/Open WebUI/edge proxy healthcheck 失败后 2 分钟内由宿主恢复；mailer timer 迟到不超过 2 分钟，异常 last-result 5 分钟内触发独立 incident |
| Live Canary | 24 小时内至少 3 次采集、1 次查询和 1 次日报全部有 run/task/notification/digest 关联，无永久 in-flight |

外部模型或来源明确不可用时不伪造绿色 SLO：任务进入已分类的 degraded/blocked 状态，原文入库继续，并在 5 分钟内产生告警证据。

### 12.2 发布状态

| 状态 | 含义 |
|---|---|
| `code_verified` | 组件定向测试通过 |
| `runtime_contract_verified` | 固定 Hermes/Open WebUI tag 的 Cron/API/MCP/Skills/Agent/CLI 配置与 Core 契约通过 |
| `offline_verified` | 无网络、无密钥、fake Runtime 的完整 E2E 通过 |
| `external_prereq_blocked` | P0 外部授权缺失；不否定离线证据，但禁止进入 R0/G5 |
| `live_verified` | 固定 Hermes Prod/Open WebUI、真实来源、模型和 Email Canary 通过 |
| `released` | 目标环境部署、备份、监控和回滚验证完成 |

MVP 只有在 `live_verified` 后才能称为完成。worker、orchestrator 或 goal judge 的自然语言总结不改变发布状态。

必须达到：

- 精确重复采集不产生新增文章；
- 单个坏来源不影响其他来源；
- 所有资讯保留原文链接和四类时间：发布、更新、首次发现、抓取；
- 摘要关键事实可追溯，注入测试通过；
- 相同日报窗口只生成一个业务 digest；仅有充分 before-spawn/retryable 证据的 attempt 可有限重试并复用同一 `notification_id/digest_id`、canonical body/hash、route 和 renderer；after-spawn 模糊结果保持 `unknown` 且不重发，不承诺 exactly-once 或用户已读；
- 外部 supervisor/dead-man 能分别判断 chat/scheduler/Open WebUI/edge proxy 存活，并判断 mailer timer 是否按时触发；内部健康检查区分 Core 存活、采集停滞、处理积压和投递异常；
- 清空 Hermes Memory/Session 不影响业务正确性；
- `make deploy` 可一条命令 reconcile 包含 Core 的固定 Hermes scheduler/chat、edge proxy、Open WebUI、one-shot mailer service definition 与宿主 timer，且无需启动时下载未固定组件；
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
  channels.yaml
  incidents.yaml

deploy/
  compose.dev.yaml
  compose.prod.yaml
  hermes-dev.Dockerfile
  hermes-prod.Dockerfile
  healthcheck/
  edge-proxy/
  reverse-proxy/

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
  config/prod-chat/
  config/prod-scheduler/
  config/prod-mailer/
  contracts/v2026.8.3/
  principals/news-chat/
  principals/news-enricher/
  principals/news-digest/
  cron/
  scripts/
  mailer/
  reconcile/
  skills/ai-monitor-editorial/
  skills/ai-monitor-digest/
  skills/ai-monitor-qa/

runtime/open-webui/
  contracts/v0.11.0/
  config/

ops/
  backup/
  restore/
  incident/
  supervisor/
    systemd/
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

dev/prod Compose 必须声明不同容器名、network、chat/scheduler `HERMES_HOME` volume、唯一 Core DB volume、Open WebUI data volume 和 secret mount。`prod-chat` 与 `prod-scheduler` 不共享 `HERMES_HOME` 或 secret；`prod-mailer` 是只供 systemd unit 启动的 one-shot service definition，位于非默认 profile、`restart: "no"`、不发布端口，使用临时 `HERMES_HOME`、只读 config 与独立 SMTP secret，三者只通过同一 Core repository/锁协议访问唯一 Core DB。生产 `scripts/` 与各 principal 的批准 skill 集合按只读 bind mount 注入，不与开发实例共享任何可写路径。Open WebUI 只加入 frontend network 并持有 proxy credential；edge proxy 再通过 backend-only network 连接 Hermes API 并独占真实 Hermes Bearer，不共享文件卷、Docker socket 或生产控制面。

`tests/contracts/`、`tests/runtime_contract/`、`tests/security/`、`evals/thresholds.yaml`、`evals/golden/` 与所有配置 Schema 的唯一修改 owner 是 `contract-verifier`；`security-release` 只运行它们并写报告。`src/ai_monitor_core/models/`、`src/ai_monitor_core/repository/`、`src/ai_monitor_core/cli/`、`src/ai_monitor_core/mcp_server/`、`src/ai_monitor_core/audit/` 与 `alembic/` 的实现 owner 是 `core-worker`；`config/sources.yaml` 的值由 `connector-worker` 维护，`config/schedules.yaml`、`config/channels.yaml`、`config/incidents.yaml`、`deploy/`、`runtime/hermes/` 和 `ops/` 的实现 owner 是 `runtime-worker`。各实现 owner 的公共契约、安全与恢复 Gate 仍只能由 `contract-verifier` 修改。

## 14. 延后能力的触发条件

- SQLite 出现稳定锁争用、需要多进程写入或文章规模明显增长时迁移 PostgreSQL；
- 确定性聚类后重复曝光率仍高于 5% 时引入 embedding/pgvector；
- Hermes Cron 出现持续重叠、`unknown` 无法在 SLO 内收敛或单机 Runtime 无法满足恢复目标时，再评估 OpenClaw、Redis + 独立 worker 或托管调度；
- 出现第二个真实用户后再设计账号、权限和个性化；
- 只有高价值来源确实没有稳定 Feed/API 时才写来源专用 HTML 适配器；
- Open WebUI 的聊天/历史不能满足资讯流、全文搜索、收藏和批量筛选需求时，再增加只读领域 Web UI；UI 复杂度确实需要时才引入 Next.js，不能把 Open WebUI 数据库反向当作业务库；
- 确实需要即时移动 Push 时再增加 ntfy 或 Web Push adapter；确实更偏好微信/Telegram 时替换 Notification Adapter 或 Query Edge，并重跑渠道 P0/R0，不改 Core；
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
8. 建立隔离的 Hermes Prod chat/scheduler profiles、无状态 one-shot mailer 与 systemd unit/timer、版本化 skills、edge proxy/API allowlist、Cron desired-state、no-agent scripts、reconcile、supervisor 和最小权限策略；
9. 完成 G3/G4；P0 就绪后执行真实 R0，再进入 G5；
10. 逐 Gate 集成、验证、回滚或阻塞；只为 P0/R0/G5 请求真实模型凭据、专用 Email 账号、Open WebUI 外部入口、systemd unit/timer 安装和部署授权。

## 16. 参考资料

Hermes 实现与生产 Runtime 官方资料：

- [Hermes Agent v0.20.0 / v2026.8.3 Release](https://github.com/NousResearch/hermes-agent/releases/tag/v2026.8.3)
- [固定 tag：Kanban](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/features/kanban.md)
- [固定 tag：Profiles](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/profiles.md)
- [固定 tag：Git Worktrees](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/git-worktrees.md)
- [固定 tag：Goals](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/features/goals.md)
- [固定 tag：Context Files](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/features/context-files.md)
- [固定 tag：Cron](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/features/cron.md)
- [固定 tag：CLI Commands / `hermes send`](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/reference/cli-commands.md)
- [固定 tag：Open WebUI integration](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/messaging/open-webui.md)
- [固定 tag：Email](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/messaging/email.md)
- [固定 tag 源码：`hermes send`](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/hermes_cli/send_cmd.py)
- [固定 tag 源码：standalone send router](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/tools/send_message_tool.py)
- [固定 tag 源码：Email adapter](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/plugins/platforms/email/adapter.py)
- [固定 tag 源码：platform config](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/gateway/config.py)
- [固定 tag：MCP](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/features/mcp.md)
- [固定 tag：Skills](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/features/skills.md)
- [固定 tag：Toolsets Reference](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/reference/toolsets-reference.md)
- [固定 tag：Docker](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/docker.md)
- [固定 tag：Security](https://github.com/NousResearch/hermes-agent/blob/v2026.8.3/website/docs/user-guide/security.md)
- [Open WebUI v0.11.0 Release](https://github.com/open-webui/open-webui/releases/tag/v0.11.0)
- [Open WebUI GHSA-vjqm-6gcc-62cr（v0.9.6 修复）](https://github.com/open-webui/open-webui/security/advisories/GHSA-vjqm-6gcc-62cr)
- [Open WebUI PWA 官方说明](https://docs.openwebui.com/getting-started/open-webui-as-app/)
- [Healthchecks.io Pinging API](https://healthchecks.io/docs/http_api/)
- [Healthchecks.io Notification Configuration](https://healthchecks.io/docs/configuring_notifications/)

Hermes 的 H0/R0 与实现以固定 tag 快照、保存的 CLI/schema 输出和镜像 digest 为准；Open WebUI 以固定 release/digest 与实际配置快照为准；Healthchecks.io 作为托管外部协议，G0 保存文档/响应 fixture 哈希，R0 必须 live 验证。任何在线文档都不能单独替代运行证据。

OpenClaw 候选 B 官方资料（仅在 Hermes R0 失败并建立新 ADR 后使用；不预先固定旧版本）：

- [OpenClaw Releases](https://github.com/openclaw/openclaw/releases)

数据源官方资料：

- [arXiv API 条款](https://info.arxiv.org/help/api/tou.html)
- [Hacker News API](https://github.com/HackerNews/API)
- [GitHub REST API 限流](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)
