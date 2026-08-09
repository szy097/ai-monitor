# AI Monitor：AI Agent 驱动实施计划

> 状态：Draft v0.1  
> 日期：2026-08-09  
> 当前仓库：除本计划外尚无代码，且未初始化 Git  
> 实施主体：AI Agent；不安排人类工程师执行开发任务

## 1. 核心决策

本项目不按人类团队的“人数 × 周期”排期，而按依赖图、机器可验证任务和发布门禁推进。

执行原则只有三条：

1. **Git 中的规格与测试是真相**：架构约束、任务契约、Golden 样本和验收命令必须版本化。
2. **Hermes Kanban 是执行状态**：负责分解、调度、重试、阻塞、审计和恢复，不替代项目规格。
3. **确定性验收决定完成**：worker 的总结和 Agent judge 都不能单独证明任务完成。

默认采用 [Nous Research Hermes Agent](https://github.com/NousResearch/hermes-agent) 作为实现控制平面的候选驱动器，精确固定到 `v0.20.0 / v2026.8.3`，不跟随 `main` 或 `latest`。Hermes 在正式驱动项目前必须先通过 H0 资格测试。

Hermes 只用于**实现项目**。AI Monitor 的生产运行不依赖 Hermes；即使以后改用 Codex、Claude Code 或其他驱动器，产品架构和任务验收契约也不改变。

## 2. 产品目标与 MVP 边界

### 2.1 目标

构建一个面向中文 AI 从业者的单用户资讯工具，自动完成：

- 获取可信 AI 最新资讯；
- 标准化标题、来源、作者与时间；
- 过滤无关内容并合并重复事件；
- 生成有证据约束的中文摘要和分类；
- 通过 Web 信息流及日报消费；
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
- Web 列表、详情、来源/主题筛选；
- HTML/text 日报，邮件或飞书二选一接通真实渠道；
- Docker 一条命令启动，单机持久化运行。

### 2.3 明确不做

- 通用网页爬虫、登录墙、验证码绕过；
- 全量社交媒体抓取；
- 多租户、复杂权限和原生移动 App；
- 训练个性化推荐模型；
- Kafka、Kubernetes、微服务等超前基础设施；
- 把 Hermes 或其他 Agent 放进每次资讯采集的关键路径。

## 3. 两层架构

```text
实现控制平面

用户目标
   ↓
Git 规格 / AGENTS.md / 任务契约 / Golden Evals
   ↓
Hermes Kanban（任务、依赖、重试、审计、恢复）
   ↓
具名 Profiles → 隔离 Worktrees → 独立 Verifier → Integrator
   ↓
持续保持绿色的主分支


产品数据平面

确定性调度器
   ↓
来源适配器 → 获取 → 解析/标准化 → 过滤 → 去重/聚类
                                                ↓
                                      分类/摘要/证据抽取
                                                ↓
                                   SQLite → Web/API → 日报
```

### 3.1 实现控制平面

采用 Hermes 的成熟能力，不自行开发 Agent 编排器：

- **Durable Kanban**：任务、依赖、attempt、日志和 handoff 持久化，跨会话可恢复；
- **Profiles**：每个角色拥有独立身份、配置、记忆和会话；并发进程不得共享 profile；
- **Worktrees**：每张实现卡在独立分支和工作目录执行，避免并发互相覆盖；
- **Dispatcher**：回收 stale/crashed worker，控制并发并执行有限重试；
- **Goal-mode card**：仅用于确实需要多轮迭代的实现卡；卡片正文写明验收合同；
- **delegate_task**：只用于短期、进程内、无需跨重启保留的研究或审查；
- **Cron/Gateway**：仅做看板调度、健康巡检和状态通知，不承担产品采集任务。

Hermes Kanban 当前是单机 SQLite 控制面，符合本项目首版需要；不将其规划为跨主机编排系统。

### 3.2 产品数据平面

默认技术栈：

- Python 3.12；
- `uv` 管理依赖和锁文件；
- FastAPI + SQLAlchemy；
- SQLite + Alembic；
- APScheduler；
- HTTPX + feedparser；
- Jinja/HTMX 服务端页面；
- pytest、Ruff、mypy；
- Docker 单容器与持久卷。

AI 摘要通过可替换的 `Enricher` 接口接入。无密钥时应用使用明确标识的规则型/抽取型降级结果；测试使用确定性 fake provider。真实 AI 摘要只有通过 Live Gate 后才能标记为可用。

## 4. 状态与真相来源

| 内容 | 真相来源 | 说明 |
|---|---|---|
| 产品范围、架构、契约 | Git | 任何 Agent 不得仅在记忆中修改约束 |
| Golden 样本、验收阈值 | Git | 实现 worker 无权自行放宽 |
| 当前任务、依赖、attempt | Hermes Kanban DB | 是调度状态，不提交进仓库 |
| 代码和迁移 | Git commit | 每张卡必须从指定 base SHA 开始 |
| 验收证据 | Git/CI artifact | 保存命令、退出码、日志摘要和制品哈希 |
| 产品运行数据 | AI Monitor 数据库 | 与 Hermes 状态完全隔离 |
| Agent memory | 非真相来源 | 只保存偏好和经验，不能替代任务状态 |

Hermes 看板应能由 Git 中的任务模板和依赖定义重新建立，避免单机 Kanban DB 成为唯一不可恢复资产。

## 5. H0：Hermes 驱动器资格测试

Hermes 仍处于 `<1.0` 且迭代频繁。正式实施前，在临时 Git 仓库完成以下测试：

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

若 H0 失败，改用 Codex 或 Claude Code 按同一任务契约串行/并行执行；不为迁就某个驱动器修改产品代码。首版不把 Claude Code/Codex CLI 强行接成 Hermes 外部 worker lane，因为该路径尚不是 Hermes 官方标准化能力。

## 6. Hermes 角色设计

| Profile | 职责 | 允许修改 | 禁止事项 |
|---|---|---|---|
| `orchestrator` | 分解目标、建立依赖、分配任务、处理阻塞 | Kanban 任务与评论 | 不写产品代码，不自行验收 |
| `contract-verifier` | 冻结契约、Golden fixtures 和 Gate 测试 | `docs/`、`evals/`、验收测试 | 不实现被测功能，不降低阈值 |
| `connector-worker` | RSS、arXiv、GitHub、HN 适配器 | connectors 与对应测试 | 不改公共领域模型 |
| `pipeline-worker` | 标准化、去重、聚类、摘要和排序 | pipeline 与对应测试 | 不改来源契约和 UI |
| `app-worker` | API、页面、日报和通知适配器 | api/web/delivery 与对应测试 | 不改 Golden 数据 |
| `integrator` | 合并已验收提交、解决接口冲突 | 集成文件、迁移、构建配置 | 不替实现卡补功能，不绕过 Gate |
| `security-release` | 安全审查、离线验收、Live Canary | 安全测试与发布证据 | 不开发新功能 |

默认并发上限为 3。只有修改路径无交集、公共契约已冻结且依赖已满足的卡才能并行。

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

allowed_paths:
  - src/ai_monitor/connectors/
  - tests/connectors/

boundaries:
  - 不修改公共数据契约
  - 不访问任务未列出的域名
  - 不用 live response 替代 fixture 测试

verification:
  commands:
    - uv run pytest tests/connectors/test_rss.py -q
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

## 8. 实施 DAG

```text
H0-DRIVER        Hermes 资格测试
   ↓
G0-CONTRACT      Git 初始化、上下文、契约、fixtures、统一命令
   ↓
G1-VERTICAL      单个 RSS → 入库 → 摘要降级 → API → 页面
   ↓
   ├── G2-RSS       官方 RSS/Atom 来源
   ├── G2-ARXIV     arXiv 连接器与限流
   ├── G2-GITHUB    GitHub Releases 连接器
   ├── G2-HN        HN 信号连接器
   ├── G2-FETCH     HTTP 缓存、退避、大小限制、错误分类
   ├── G2-DEDUP     URL、external ID、内容哈希与近重复
   ├── G2-ENRICH    分类、中文摘要、证据和提示注入防护
   ├── G2-WEB       列表、详情、筛选和健康状态
   └── G2-DELIVERY  日报与幂等通知
            ↓
G3-INTEGRATION     调度、检查点、失败隔离、崩溃恢复
   ↓
G4-OFFLINE         无公网、无密钥的全链路 Release 验收
   ↓
G5-LIVE            真实来源、真实模型、真实通知 Canary
   ↓
MVP-RELEASE
```

### G0：契约与仓库基线

产物：

- Git 仓库和首个绿色 commit；
- `AGENTS.md`，记录架构、统一命令、不可绕过规则；
- `pyproject.toml`、锁文件、Makefile、CI；
- `SourceConfig`、`FetchedDocument`、`NormalizedArticle`、`EnrichmentResult`、`Delivery` 契约；
- RSS/Atom/GitHub/arXiv/HN 冻结 fixtures；
- Golden 去重、摘要、安全样本；
- 任务模板和验收证据格式。

Gate：`uv sync --all-extras && make check` 通过，且 verifier 确认实现 worker 无权修改 Golden 预期。

### G1：最小真实纵向切片

只接一个官方 RSS 来源，打通：

```text
fixture/live fetch → parse → normalize → exact dedupe → SQLite
                  → extractive fallback → API → server-rendered page
```

Gate：

- 同一数据连续导入 3 次，文章数量不增加；
- 无网络、无 LLM Key 时 E2E 通过；
- fresh install 只需 README 中一条启动命令；
- 页面中的每条内容都能回到原始 URL。

### G2：可并行组件卡

G1 通过且公共契约冻结后，来源、pipeline、Web 和 delivery 分 worktree 并行实现。每个组件使用自己的 fixtures、定向测试和 verifier 卡，不允许并行修改同一公共接口。

### G3：集成与可靠性

集成内容：

- 定时采集和防重入锁；
- `ETag`、`Last-Modified` 和增量 checkpoint；
- `429 Retry-After`、有限重试和死信；
- 单来源失败隔离；
- 抓取、解析、入库、摘要各阶段的崩溃恢复；
- 来源健康状态和手动重跑；
- 日报窗口与通知幂等键。

Gate：相同批次运行 3 次和并发运行后数据库快照一致；每个阶段注入进程终止后，恢复结果与无中断运行一致。

### G4：Offline Release Gate

在全新、禁止访问公网、没有任何 API Key 的环境中执行：

```bash
make check       # format、lint、type、unit
make verify      # 契约、幂等、恢复、Golden eval、安全
make smoke       # 构建容器并跑 fixture 全链路
```

任一测试被删除、注释、`skip` 或 `xfail` 均视为失败。

### G5：Live Canary Gate

使用真实来源、真实模型和真实通知渠道运行至少一个完整采集周期：

```bash
make live-smoke
```

Live Gate 禁止 Mock。缺少密钥、Webhook、部署授权或付费授权时，状态只能是 `offline_verified`，不能宣称 MVP 已上线。

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
- 重启后不丢数据、不重复通知。

### 9.3 去重与摘要

- 同文不同 URL；
- 同事件不同报道；
- 标题相同但事件不同；
- 每条摘要事实必须关联 source ID 和短证据；
- Golden 样本中不得新增原文不存在的实体、数字或日期；
- 来源互相矛盾时必须表达不确定性，不能静默选边。

### 9.4 安全

- 网页 Prompt Injection 不得改变系统指令、调用工具或读取密钥；
- 重定向到 localhost、私网和云元数据地址必须被拒绝；
- 限制响应体、重定向次数和允许的 Content-Type；
- 通知正文不得包含密钥、原始系统提示词或内部路径。

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

Hermes 无人值守 worker 默认使用隔离 backend；宿主机 local terminal 不能视为 sandbox。配置工作区 safe root、危险命令审批和 headless fail-closed，禁止在宿主机启用无审批的高权限模式。

其他要求：

- 密钥只能由环境或秘密代理注入，不写入任务卡、prompt、Git 或日志；
- Agent 不创建或修改真实 `.env`；
- Hermes Dashboard 只绑定 localhost，不暴露到共享网络；
- Gateway 使用用户 allowlist；
- 网页正文始终是不可信数据，不具有指令权限；
- 摘要进程不拥有 shell、部署或通知凭据；
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

- 规则过滤和精确去重先于模型调用；
- 先按事件聚类，再生成一次摘要；
- 模型结果按内容哈希、prompt 版本和模型版本缓存；
- 每轮采集设置文章数、token 和费用上限；
- 超过预算时保留原文元数据并延迟摘要，不阻断资讯入库。

## 12. 发布状态与完成定义

| 状态 | 含义 |
|---|---|
| `code_verified` | 组件定向测试通过 |
| `offline_verified` | 无网络、无密钥的完整 E2E 通过 |
| `live_verified` | 真实来源、模型和通知 Canary 通过 |
| `released` | 目标环境部署、备份、监控和回滚验证完成 |

MVP 只有在 `live_verified` 后才能称为完成。worker、orchestrator 或 goal judge 的自然语言总结不改变发布状态。

必须达到：

- 精确重复采集不产生新增文章；
- 单个坏来源不影响其他来源；
- 所有资讯保留原文链接和四类时间：发布、更新、首次发现、抓取；
- 摘要关键事实可追溯，注入测试通过；
- 相同日报窗口只投递一次；
- `/healthz` 能区分“进程存活”和“采集长期失败”；
- Docker 可一条命令启动；
- Offline 与 Live 证据均可重放和审计。

## 13. 目标仓库结构

```text
AGENTS.md
README.md
Makefile
pyproject.toml
uv.lock

docs/
  PLAN.md
  contracts/
  adr/
  runbooks/

tasks/
  schema.json
  templates/

src/ai_monitor/
  connectors/
  pipeline/
  models/
  api/
  web/
  delivery/

tests/
  fixtures/
  connectors/
  pipeline/
  integration/
  e2e/
  security/

evals/
  golden/
  reports/
```

当前步骤只创建本计划文档。其余文件在 G0 由 Agent 按 Gate 顺序生成。

## 14. 延后能力的触发条件

- SQLite 出现稳定锁争用、需要多进程写入或文章规模明显增长时迁移 PostgreSQL；
- 确定性聚类后重复曝光率仍高于 5% 时引入 embedding/pgvector；
- 定时任务出现持续重叠或需要水平扩展时引入 Redis + 独立 worker；
- 出现第二个真实用户后再设计账号、权限和个性化；
- 只有高价值来源确实没有稳定 Feed/API 时才写来源专用 HTML 适配器；
- UI 复杂度确实超过服务端渲染能力后才引入 Next.js。

## 15. 实施启动顺序

收到“开始实施”后，Agent 按以下顺序执行，不再向用户反抛普通技术选择：

1. 初始化 Git，并提交本计划作为基线；
2. 执行 H0 Hermes 资格测试；
3. 创建根 `AGENTS.md` 和任务卡 schema；
4. 建立 Hermes 项目 board、profiles、并发和安全策略；
5. 将 G0–G5 DAG 写入 Kanban；
6. 完成 G0 和 G1；
7. G1 通过后才并行执行 G2；
8. 逐 Gate 集成、验证、回滚或阻塞；
9. 到 G5 时仅请求真实模型密钥、通知凭据和部署授权。

## 16. 参考资料

Hermes 官方资料：

- [Hermes Agent Releases](https://github.com/NousResearch/hermes-agent/releases)
- [Kanban：持久化多 Agent 看板](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)
- [Profiles](https://hermes-agent.nousresearch.com/docs/user-guide/profiles/)
- [Git Worktrees](https://hermes-agent.nousresearch.com/docs/user-guide/git-worktrees)
- [Context Files / AGENTS.md](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files)
- [Subagent Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security/)
- [Scheduled Tasks](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron/)

数据源官方资料：

- [arXiv API 条款](https://info.arxiv.org/help/api/tou.html)
- [Hacker News API](https://github.com/HackerNews/API)
- [GitHub REST API 限流](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)
