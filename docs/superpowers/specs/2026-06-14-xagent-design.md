# xagent — AI 原生的软件工程协作平台

> **设计文档** | 2026-06-15 | Version 1.1

---

## 概述

xagent 是一个 AI 原生的软件工程平台。AI Agent 是主力开发者，人类团队成员通过 IM 渠道（企业微信/钉钉/飞书）与 Agent 交互。平台支持小团队（2-5 人 + AI）围绕单个项目协作。

### 核心原则

- **AI 是主力干活的** — Agent 端到端负责执行
- **IM 是主要交互界面** — 所有人通过熟悉的聊天工具与 Agent 沟通
- **全员可见** — 任何团队成员可以随时查看项目的任何部分
- **默认安全** — Agent 的所有操作都经过权限门禁、审计追踪、可回滚
- **上下文绑定决策** — 每次确认都限定在特定的上下文快照范围内

---

## Section 1: 整体架构

### 三层架构

```
Channel Layer  →  Agent Core  →  Project Brain
   (I/O 层)       (思考+执行)     (记忆存储)
```

| 层 | 职责 | 原则 |
|-------|---------------|-----------|
| **Channel Layer（渠道层）** | IM 协议与内部事件之间的翻译转换 | 纯 I/O，不含业务逻辑 |
| **Agent Core（智能体核心）** | 消息路由、任务拆解、LLM 编排、工具执行 | 中央大脑，拥有所有决策权 |
| **Project Brain（项目大脑）** | 所有状态、决策、变更、联系人的持久化存储 | 唯一真相源 |

### 组件架构图

```
┌─────────────────────────────────────────────────┐
│                  CHANNEL LAYER                     │
│  企业微信 Adapter  │  钉钉 Adapter  │  飞书 Adapter  │  Web Dashboard │
└───────────────────────┬─────────────────────────┘
                        │  Channel Events
                        ▼
┌─────────────────────────────────────────────────┐
│                  AGENT CORE                       │
│  Message Router（消息路由）│  Task Orchestrator（任务编排）│
│  Decision Escalator（决策升级）│  LLM Engine │  Tool Harness │
└───────────────────────┬─────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────┐
│                PROJECT BRAIN                      │
│  Task Graph（任务图）│  Decision Log（决策日志）    │
│  Code History（变更历史）│  Contact Store（联系人）  │
│  Discussion Threads（讨论线程）                    │
│  ─────────────────────────────────────            │
│  Event Store（append-only，不可变）                │
└─────────────────────────────────────────────────┘
```

---

## Section 2: Agent Core（智能体核心）

### Agent 生命周期

- **单例模式** — 每个项目一个 Agent 实例
- **全量状态持久化** — Agent 的所有状态（任务树、对话上下文、未完成的工具调用、进行中的讨论）都持久化到 Project Brain
- **定期 checkpoint** — Agent 定期快照状态
- **故障恢复（Failover）** — Watchdog 检测到崩溃后，从最新 checkpoint 恢复，重放 checkpoint 之后的 Event Store 事件，恢复未完成任务和进行中的讨论，并通过 IM 通知团队

### 消息处理流水线

```
IM 消息进入 → 身份识别 → 意图解析 → （查询类/执行类）
                                        │
                                   任务拆解
                                        │
                                   信心检查
                                 ┌────┼────┐
                            确定  │        │ 不确定
                                 │        │
                            直接执行   升级为人工确认
                                 │     （Discussion Thread）
                            广播通知
```

### Sub-Agent 模型

Sub-agent 是主 Agent 为并行执行原子任务而派生的**轻量级执行实例**。

| 属性 | 主 Agent | Sub-Agent |
|------|---------|-----------|
| **职责** | 拆解任务、路由决策、维护全局上下文 | 执行单个原子任务 |
| **生命周期** | 项目级别，持久运行 | 任务级别，执行完即销毁 |
| **上下文** | 完整项目上下文 + 对话历史 | 只含该原子任务相关的精简上下文 |
| **决策权** | 有（拆解、升级、确认） | 无（只执行，不确定时回传主 Agent） |
| **并发数** | 1 个 | 最多 5 个同时运行 |
| **工具权限** | 受完整 Harness 约束 | 继承主 Agent 的权限配置 |
| **故障处理** | Watchdog 恢复 | 挂了就挂了，主 Agent 重试或重分配 |

Sub-agent 的关键约束：
- **只执行，不决策** — 遇到任何不确定，回传主 Agent，不自行判断
- **上下文隔离** — sub-agent 不知道其他 sub-agent 在做什么，避免相互干扰
- **操作幂等** — 主 Agent 重试失败的 sub-agent 时，操作必须是幂等的
- **结果回传** — 执行完成后，结果（代码变更、工具输出）回传给主 Agent，由主 Agent 合并

### 决策升级 — Discussion Thread（讨论线程）

当 Agent 不确定时，在 IM 渠道中创建 **Discussion Thread**：

1. **创建线程** — 附带问题描述、上下文和可选的截止时间
2. **选择参与者** — AI 根据代码归属、角色、联系人 profile 中的关注领域来决定拉谁进来
3. **追踪讨论** — 监控每个人说了什么，检测冲突
4. **三种冲突检测**：
   - 观点冲突：A 说 X，B 说 Y
   - 时空冲突：B 的回复基于过时的上下文
   - 沉默冲突：关键人物未回复
5. **决议闭环**：
   - 达成共识 → 记录决策，执行
   - 无法共识 → 提级到更高决策权的人
   - 超时未决 → AI 判断安全性，可能按默认策略推进

### 线程状态模型

```
OPEN（开放）→ CONFLICT（冲突中）→ STALLED（停滞）→ RESOLVED（已解决）
                    │                  │
                    ▼                  ▼
              ESCALATED（已提级）   CLOSED（已关闭）
```

---

## Section 3: 细粒度任务拆解

### 任务层级

```
Epic（用户需求）
 └─ Feature（逻辑单元）
     └─ Atomic Task（原子任务，单次操作）
```

### 原子任务的 5 条标准

| 标准 | 说明 |
|-----------|-------------|
| **单文件** | 一个任务只动一个文件（或一个明确的小范围） |
| **单动作** | "创建文件" 或 "修改函数" 或 "添加测试"，不混 |
| **可验证** | 有明确的成功标准（测试通过、lint 过） |
| **可回滚** | 失败不影响其他任务，可独立回滚 |
| **≤ 5 分钟** | 理想情况下单个原子任务执行时间不超过 5 分钟 |

### 任务数据模型

```yaml
Task:
  id, parent_id, type: [epic|feature|atomic]
  title, description
  status: [pending|running|done|blocked|failed]
  depends_on: [task_ids]
  blocked_by: []
  assigned_to: sub_agent_id
  tool_calls: [{tool, input, output, duration}]
  decisions: [decision_ids]
  visible_to: all
```

### 并行执行策略

- Feature 之间可以并行
- Feature 内部按依赖关系串行
- 最多 5 个 sub-agent 并发（避免 LLM rate limit）
- Agent 自动分析依赖图以最大化并行度

---

## Section 4: Harness 安全 — 五层模型 + Safety Model

核心理念：**"干活的 Agent"和"审查安全的 Agent"必须是两个不同的脑子**。

### 安全架构全景

```
Agent（干活）要执行操作
       │
       ▼
┌──────────────────────────────────────────┐
│ L1: 能力边界                              │
│     没注册的能力根本不存在                  │
└──────────────┬───────────────────────────┘
               │ 操作已注册
               ▼
┌──────────────────────────────────────────┐
│ L2: 权限分级                              │
│     AUTO ──→ Safety Model 自动审查        │
│     CONFIRM → 人工确认                     │
│     REJECT ──→ 硬拒绝                      │
└──────────────┬───────────────────────────┘
               │
          ┌────┴────┐
          │         │
      AUTO/SM   CONFIRM
          │         │
          ▼         ▼
┌─────────────┐ ┌─────────────┐
│ L2.5        │ │ 人工确认     │
│ Safety      │ │ 多渠道推送   │
│ Model       │ │ 决策记录     │
│ 安全审查     │ └─────────────┘
│             │
│ 批准/拒绝    │
│ /升级人工    │
└──────┬──────┘
       │ 批准
       ▼
┌──────────────────────────────────────────┐
│ L3: 沙箱隔离                              │
│     文件系统、网络、进程全限制              │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│ L4: 审计追踪                              │
│     Event Store 全记录                    │
└──────────────────────────────────────────┘
```

### 第一层：能力边界（Capability Boundary）

Agent 只能调用已注册的工具。未注册的能力根本不存在。

**默认工具**：`file_read`、`file_write`、`shell_exec`、`git_*`、`web_fetch`、`lsp_query`
**可插件扩展**：MCP Server 提供的工具、Skill 引入的工具
**硬编码禁止**：`file_delete`（默认）、`network_bind`、`sudo`

### 第二层：权限分级 — 三级简化模型

| 级别 | 行为 | 谁审查 | 示例 |
|-------|------|--------|---------|
| 🟢 **AUTO** | Safety Model 自动审查，通过即执行 | Safety Model | file_read、git_log、test_run、web_fetch |
| 🟡 **AUTO+** | Safety Model 审查，高风险时升级人工 | Safety Model → 人工 | file_write、shell_exec、git_commit |
| 🔴 **CONFIRM** | 跳过 Safety Model，必须人工确认 | 直接人工 | pip_install、config_change、git_push |
| ⛔ **REJECT** | 硬编码拒绝，不可绕过 | 无（绝对禁止） | file_delete、rm -rf、force push |

> 不再有 AI_REVIEW 的概念——所有 AI 级别的审查都由 Safety Model 统一负责。
> AUTO 和 AUTO+ 的区别仅在于：AUTO+ 的操作如果 Safety Model 不确定，会升级到人工；AUTO 的操作 Safety Model 不确定时直接拒绝（因为这类操作默认就应该安全）。

### 第二点五层：Safety Model（安全审查模型）

```
Agent 要执行操作
       │
       ▼
┌──────────────────────────────────────────┐
│         SAFETY MODEL                      │
│                                           │
│  独立的、专注于安全的 LLM 实例             │
│  可能是一个专门的模型（如 fine-tuned）      │
│  初期用 BRAIN Tier + 安全专用 system prompt│
│                                           │
│  输入:                                     │
│  • 操作类型 + 参数                         │
│  • 上下文快照（文件、任务、影响范围）        │
│  • 历史同类操作的审查结果                   │
│                                           │
│  输出:                                     │
│  • APPROVE  → 执行                         │
│  • DENY     → 拒绝（记录原因）              │
│  • ESCALATE → 升级到人工确认               │
└──────────────────────────────────────────┘
```

**Safety Model 的设计原则**：

| 原则 | 说明 |
|------|------|
| **独立实例** | 与干活 Agent 使用不同的 LLM 上下文，不共享对话历史 |
| **只读视角** | Safety Model 只做审查，永远不调用工具、不修改代码 |
| **专注安全** | system prompt 只包含安全规则，不包含执行逻辑 |
| **快速判定** | 输入精简（操作+上下文快照），期望 <2s 响应 |
| **可审计** | 每次审查结果写入 Decision Log，与人工决策同等待遇 |

**Safety Model 的审查 Prompt 结构**：

```
你是 xagent 安全审查引擎。你的唯一职责是判断操作是否安全。

安全规则：
1. 操作范围是否在项目目录内？
2. 是否涉及敏感信息（密码、token、密钥）？
3. 是否可能造成不可逆破坏（删除、覆写关键配置）？
4. 操作参数是否与任务描述一致？
5. 是否违反项目的安全策略（xagent.yaml）？

请审查以下操作：
- 操作: git_commit
- 参数: message="add auth module", files=["auth.py","models.py"]
- 上下文: task_042, Feature "数据库模型", 改动 +142 -0
- 风险等级: low

回答格式: { decision: APPROVE | DENY | ESCALATE, reason: "..." }
```

**Safety Model 与权限等级的映射**：

| 操作权限等级 | Safety Model 行为 |
|-------------|------------------|
| AUTO | Safety Model 审查 → APPROVE 执行 / DENY 拒绝 / ESCALATE 升级人工 |
| AUTO+ | 同上，但 ESCALATE 阈值更低（更容易升级） |
| CONFIRM | **绕过 Safety Model**，直接走人工确认 |
| REJECT | **不经过任何审查**，直接拒绝 |

**为什么要分离 Safety Model？**

| 分离的好处 | 如果不分离 |
|-----------|----------|
| 干活 Agent 和审查 Agent 利益冲突隔离 | Agent 可能给自己放水 |
| 安全 prompt 不与任务 prompt 混合 | 不会因为任务太长、安全规则被"挤掉" |
| 可以用更便宜的模型做审查（输入少） | 审查跟着任务上下文，成本高 |
| 审查结果独立审计 | 自己审自己，审计意义打折 |

### 第三层：沙箱隔离（Sandbox Isolation）

- **文件系统**：Agent 只能读写项目目录 + 显式白名单
- **网络**：域名白名单（pypi.org、github.com、…）
- **进程**：ulimit 限制，禁止 fork bomb
- **时间**：单次 shell 命令最长 5 分钟
- **实现方案**：生产环境用 Docker 容器，初期用 subprocess + chroot + seccomp

### 第四层：审计追踪（Audit Trail）

每次工具调用记录到 Event Store：
```yaml
{tool, input, output, duration, permission_level,
 safety_model_decision, safety_model_reason,
 sandbox_id, files_touched, snapshot_diff}
```

### 第五层：回滚机制

- **Git 级（粗粒度）**：Feature 开始前自动 git stash，需要时 reset
- **Overlay 级（细粒度）**：每个原子任务的 overlay delta；失败时丢弃该 delta，不影响并行的其他任务
- **触发方式**：Agent 自检发现失败、Dashboard 点击、IM 命令

---

## Section 5: 多渠道确认 & 决策记录

### 确认流程

1. Agent 需要确认 → 生成附带上下文快照的 Confirmation Request
2. **同时推送到所有渠道**（IM 私聊 + 群 @提醒 + Dashboard 通知）
3. 任一渠道先回复即生效
4. 超时（30 分钟）→ AI 判断是否继续推进或升级

### Decision Record（决策记录 — 不可变）

```yaml
DecisionRecord:
  id, request_id
  type: [git_commit|file_write|shell_exec|architecture|…]
  summary, context: {task_id, files, diff_hash, risk_level}
  decided_by: contact_id | "safety_model"
  channel: [wecom|dashboard|…]
  decision: [approved|rejected|expired]
  decided_at, comment
  hash: sha256(entire_record)
  prev_hash: sha256(previous_record)  # hash 链式关联，防篡改
```

### Decision Log 查询能力

- "这个文件谁同意改的？" → 按文件过滤
- "张三最近做了哪些决策？" → 按 decided_by 聚合
- "哪些确认被超时忽略了？" → 按 expired 过滤
- "Feature X 讨论了什么？" → 与 Discussion Thread 关联查询

---

## Section 6: 上下文绑定 & 智能超时

### 上下文绑定原则

> 确认的意思是"在这个具体的上下文下，你可以执行这个具体的操作"。上下文一变，授权自动失效。

执行任何已确认的操作前，Agent 强制检查：

| 检查项 | 不匹配时的行为 |
|-------|-------------------|
| 文件内容 hash 是否一致？ | 重新确认（文件被别人改了） |
| 依赖任务是否仍满足？ | 重新确认（前置条件变了） |
| 分支是否一致？ | 重新确认（可能在错误分支） |
| 权限配置是否变更？ | 重新确认（规则变了） |
| 决策者是否仍有效？ | 重新路由（人可能离职/换岗） |

### 智能超时

超时策略按权限等级分别处理：

**AUTO / AUTO+ 级别超时**（Safety Model 审查的操作）：
1. Safety Model 重新评估：上下文还干净吗？风险低吗？
2. 上下文干净 + 低风险 → Safety Model 确认继续（记录在案）
3. 上下文变了 → 重新生成确认请求，走完整审查链
4. 高风险或不确定 → @提醒 + 重置 TTL

**CONFIRM 级别超时**（必须人工确认的操作）：
1. AI **不能**自行推进——CONFIRM 的存在意义就是人的判断不可替代
2. 超时后 @提醒相关人 + 重置 TTL（再等 30 分钟）
3. 连续超时 3 次 → 升级到更高决策权的人
4. 记录超时事件到 Decision Log（审计用）

| 超时后的行为 | AUTO/AUTO+ | CONFIRM |
|------------|-----------|---------|
| Safety Model 重新审查 | ✅ | ❌（不经过 Safety Model） |
| AI 可自行推进 | ✅（低风险且上下文干净） | ❌（必须等人） |
| @提醒 + 重置 TTL | ✅（高风险/不确定） | ✅（每次超时） |
| 连续超时升级 | ✅ | ✅（升级决策链） |

---

## Section 7: Project Brain（项目大脑）— 数据模型

### 架构

```
Task Graph（任务图）│  Decision Log（决策日志）│  Code History（变更历史）│  Contact Store（联系人）
              └────────────┴───────────────┴───────────────┘
                                  │
                                  ▼
                      Event Store（append-only, 不可变）
```

### 四大核心实体

| 实体 | 关键字段 | 典型查询 |
|--------|-----------|-----------------|
| **Task** | id, parent_id, type, status, depends_on, tool_calls[], decisions[] | "现在有哪些任务在进行？"、"task_042 被什么阻塞了？" |
| **Decision** | id, type, context_snapshot, decided_by, channel, hash | "谁批准了这个？"、"最近的拒绝记录？" |
| **Change** | id, type, file_path, diff_hash, tool, task_id | "auth.py 的改动历史"、"最近 24h 改了哪些文件？" |
| **Contact** | id, name, im_accounts[], role, permissions, focus_areas[], profile_text | "这个模块该找谁？"、"张三关注哪些领域？" |

### Discussion Thread 实体

```yaml
DiscussionThread:
  id, title, question, status
  participants[], channel
  messages: [{from, content, at}]
  conflict_events: [{type, between, topic, at}]
  resolution: {type, final_choice, reason}
  → DecisionRecord（决议落地为 Decision）
```

### 存储方案

| 层 | 技术 | 说明 |
|-------|-----------|-------|
| Event Store | SQLite（初期）/ PostgreSQL（生产） | append-only 表，时间戳索引 |
| 物化视图 | 同库 view / 内存缓存 | Event Store 的实时投影 |
| 文件存储 | 本地 FS + Git | Git 天然提供版本历史 |
| 向量存储（未来）| Pinecone / pgvector | 历史语义搜索 |

---

## Section 8: Dashboard（项目活地图）

### 布局

```
┌────────────────────────────────────────────┐
│  xagent / my-project        🔔 3 条待确认  │
├────────────────────────────────────────────┤
│  📊 任务进度            │  🤖 Agent 状态    │
│  ████████░░ 18/22      │  正在执行 task_044│
│  用户登录功能 82%       │  3 个子 Agent 运行 │
├────────────────────────────────────────────┤
│  ⚡ 实时动态                               │
│  14:32 ✅ task_042 已完成 创建 User 表迁移  │
│  14:31 🤖 AI 自检通过   git commit auth 模块│
│  14:28 📝 task_045 开始 实现登录逻辑        │
├─────────────────────┬──────────────────────┤
│  🔑 待确认 (3)       │  💬 讨论中 (1)      │
│  pip install bcrypt │  "Redis 还是 MySQL？"│
│  [确认] [拒绝]       │  [查看讨论]         │
└─────────────────────┴──────────────────────┘
```

### 页面结构

| 页面 | 内容 |
|------|---------|
| **总览** | 任务进度 + Agent 状态 + 实时动态流 + 待确认列表 |
| **任务地图** | 任务树可视化、依赖关系图，点击查看详情和代码变更 |
| **决策日志** | 所有 Decision Record，可按时间/人/类型筛选，hash 链可验证 |
| **代码变更** | 文件变更历史，每次改动的 diff，关联的任务和决策 |
| **讨论区** | 所有 Discussion Thread，按状态筛选（进行中/已解决/已提级） |
| **联系人** | 团队人员列表，profile、活跃度、历史决策统计 |

### 技术选型

- **前端**：React + Tailwind（轻量 SPA）
- **实时更新**：WebSocket（Event Store 变更推送到前端）
- **数据**：REST API 从 Project Brain 视图读取

---

## Section 9: Contact System（联系人系统）

### Profile 模型

```yaml
Contact:
  id, name
  im_accounts: [{platform, id}]
  role, permissions, focus_areas[]
  profile_text: "自然语言描述，类 claude.md 格式"
```

### Profile 演进路径

| 阶段 | 内容 |
|-------|---------|
| **Phase A（初期）** | 角色、权限、关注领域、profile_text（静态） |
| **Phase C（远期）** | AI 自动学习：近期关注点、决策偏好、协作模式、活跃时段、交互统计 |

### 用途

- Agent 读取 profile 决定升级目标
- profile_text 提供沟通风格参考
- 交互历史持续积累，支撑上下文感知路由

---

## Section 10: IM Channel（IM 渠道）

### Adapter 接口

```python
class IMAdapter:
    platform: "wecom" | "dingtalk" | "feishu"

    # 入站：IM → Agent
    def on_message(msg) → ChannelEvent:
        # 标准化为：from_contact, channel_type, content, mentions, reply_to

    # 出站：Agent → IM
    def send_message(target, formatted_msg)
    def send_confirmation(target, confirm_request)
    def create_discussion_thread(group, question)
```

### 消息路由规则

| 消息类型 | 送达方式 |
|-------------|----------|
| 进度通知 | 项目群（所有人可见） |
| 确认请求 | 私聊 + 群 @提醒 + Dashboard |
| 讨论发起 | 群内 thread，@必选参与者 |
| 查询响应 | 回复到消息来源渠道 |
| 错误/异常 | 群内广播 + 私聊技术负责人 |

### 初始平台

从企业微信开始。架构预留了钉钉、飞书等后续平台的扩展空间。

---

## Section 11: LLM Engine — 多模型能力抽象

### 架构

```
Agent Request → Model Router（模型路由）→ Provider Adapter（适配器）→ Model API
                         │
                 Capability Registry（能力注册中心）
```

### 三级 Tier 模型

| Tier | 角色 | 所需能力 | 典型任务 |
|------|------|----------------------|---------------|
| **BRAIN（大脑）** | 推理与决策 | reasoning ✓, tool_use ✓, 大上下文 | 任务拆解、架构决策、冲突调解 |
| **WORKER（主力）** | 代码生成与执行 | code ✓, tool_use ✓, 中等上下文 | 代码生成、代码审查、测试生成、文档 |
| **FAST（轻量）** | 简单查询 | text ✓, 小上下文 | 意图识别、消息摘要、状态检查 |

### Capability Profile（能力档案）

```yaml
ModelCapability:
  model_id, provider, tier
  capabilities:
    text, code, reasoning: [excellent|good|basic|none]
    tool_use, vision, structured, streaming: [true|false]
  constraints: {max_context, rpm, tpm}
  cost: {input_per_1k, output_per_1k}
  context:
    max_tokens, effective_tokens
    cache_supported, cache_mechanism, cache_ttl_seconds
    compression: [summarize|truncate|hybrid]
```

### Vision 能力降级策略

当模型缺少 vision（图像处理）能力时：

1. **Re-route（切换模型，默认策略）**：临时切换到有 vision 能力的模型处理该次调用
2. **Pre-process（预处理）**：先让 vision 模型描述图片，将文字描述喂给非 vision 模型
3. **Degrade（退化）**：跳过图片，只用文字上下文

### 能力不足时的 fallback 策略

| 缺少的能力 | 应对策略 |
|-------------------|----------|
| vision（图像处理）| re_route（切换模型） |
| tool_use（工具调用）| error（无法工作，报错） |
| reasoning（推理能力）| degrade（降级到上层 Tier） |

---

## Section 12: Context Manager（上下文管理器）

### Router 与 Context Manager 协同

```
Router 选模型 → Context Manager 为该模型构建请求
```

Router 管"用谁"，Context Manager 管"喂什么"。

### Context Profile（每个模型的上下文配置）

每个模型在其 Capability Profile 中声明：
- `max_tokens`、`effective_tokens`（给输出预留空间）
- 缓存机制、TTL、breakpoint 数量
- 首选压缩策略
- 消息格式开销估算

### 分层压缩策略

当上下文超过模型限制时：

| 层级 | 策略 | 节省空间 |
|-------|----------|---------|
| L1 | 裁剪无关消息（已完成的工具调用结果、中间问询） | ~30% |
| L2 | 通过 FAST Tier 模型生成对话摘要 | ~80% |
| L3 | 截断长输出（shell 结果、大型 diff） | ~20% |
| L4 | 拆分成更小的子任务（紧急降级） | 不定 |

### Context Bridge（跨模型上下文桥接）

当 Router 在会话中途切换模型时（如 DeepSeek → Claude 处理图片）：

1. 将完整对话历史通过 FAST Tier 模型压缩成摘要
2. 将摘要 + 当前任务上下文 + 新输入传给目标模型
3. 目标模型收到的是：system prompt + 压缩历史 + 任务状态 + 当前请求
4. 不是 47 轮原始对话

### 核心接口

```python
class ContextManager:
    def build_request(model, system_prompt, messages, tools, task_context) → PreparedRequest
    def bridge_context(from_model, to_model, history, task) → list[Message]
    def estimate_tokens(messages, model) → int
    def update_after_response(response, model)
```

---

## Section 13: MCP（Model Context Protocol）— 标准化工具接入

### 为什么需要 MCP

MCP 是一个开放协议，标准化了 AI Agent 与外部工具、数据源之间的通信方式。通过 MCP，xagent 可以接入任何兼容 MCP 的服务——数据库查询、浏览器自动化、知识库搜索、第三方 API 等——而不需要为每个服务写定制代码。

### 架构

```
Agent Core
    │
    ▼
Tool Harness
    │
    ├── Built-in Tools（内置工具）: file_read, shell_exec, git_*, …
    ├── Plugin Tools（插件工具）: browser, docker, …
    └── MCP Client ──── MCP Servers（外部工具）
         │                  ├── MCP Server A: 数据库查询
         │                  ├── MCP Server B: 浏览器自动化
         │                  ├── MCP Server C: 知识库搜索
         │                  └── MCP Server D: …
         │
    ── 所有工具统一走权限门禁（AUTO/AUTO+/CONFIRM/REJECT）
```

### MCP Client 职责

```python
class MCPClient:
    """管理与 MCP Server 的连接和工具发现"""

    def connect(server_config) → MCPConnection
        """建立与 MCP Server 的连接（stdio / SSE / streamable HTTP）"""

    def list_tools(connection) → list[ToolDef]
        """发现该 Server 提供的所有工具"""

    def list_resources(connection) → list[ResourceDef]
        """发现该 Server 提供的资源（文件、数据等）"""

    def call_tool(connection, tool_name, args) → ToolResult
        """调用工具，经过权限门禁"""

    def health_check(connection) → bool
        """检查连接是否存活"""
```

### 配置方式

```yaml
# xagent.yaml — MCP 配置
mcp:
  servers:
    - name: "codegraph"
      command: "codegraph"
      args: ["--stdio"]
      env: {}
      auto_connect: true           # Agent 启动时自动连接

    - name: "playwright"
      command: "npx"
      args: ["-y", "@anthropic/mcp-playwright"]
      auto_connect: false          # 需要时再连接

    - name: "knowledge-base"
      url: "https://kb.internal.com/mcp"
      transport: "sse"
      headers:
        Authorization: "Bearer ${KB_TOKEN}"

  # MCP 工具的权限映射
  tool_permissions:
    "codegraph.*": auto            # codegraph 的工具全自动
    "playwright.browser_navigate": auto_plus
    "playwright.browser_click": auto_plus
    "knowledge-base.*": auto
```

### MCP 工具的安全边界

- MCP 工具与内置工具走**同一套权限门禁**（4 级模型）
- 每个 MCP Server 的工具可以单独配置权限级别
- MCP Server 的进程在**沙箱内运行**（受 L3 沙箱隔离约束）
- 工具调用记录到 Event Store，与内置工具无差别审计

---

## Section 14: Skill System（技能系统）— Agent 行为扩展

### 为什么需要 Skill

Skill 是封装了特定领域知识、工作流程和最佳实践的可复用模块。它不是"工具"（工具是 Agent 调用的外部能力），而是"方法论"——告诉 Agent **怎么做某类事情**。例如：

- **test-driven-development**：教 Agent 先写测试再写代码的完整流程
- **systematic-debugging**：教 Agent 如何系统性地排查 bug
- **frontend-design**：教 Agent 如何创建高质量的 UI
- **code-review**：教 Agent 如何审查代码

### Skill 在 xagent 中的位置

```
Agent Core
    │
    ├── System Prompt（系统提示词）
    ├── Skill Registry（技能注册中心）
    │     ├── Project Skills（项目级）
    │     ├── User Skills（用户级）
    │     └── System Skills（系统级）
    │
    └── Skill × Tier 映射
           ├── 复杂 Skill（TDD、架构设计）→ BRAIN Tier
           ├── 执行 Skill（代码生成、测试）→ WORKER Tier
           └── 轻量 Skill（格式检查、摘要）→ FAST Tier
```

### Skill 数据模型

```yaml
Skill:
  name: "test-driven-development"
  description: "编写代码前先写测试，红-绿-重构循环"
  trigger_keywords: ["测试", "tdd", "单元测试", "test"]
  tier: "worker"               # 默认用哪个 Tier 的模型执行

  # 技能内容（类 markdown，含指令和工作流）
  body: |
    ## TDD 开发流程
    1. 先写一个失败的测试
    2. 写最少代码让测试通过
    3. 重构代码
    ...

  # 可选：技能需要的额外工具
  required_tools: ["file_read", "file_write", "shell_exec", "test_run"]

  # 可选：技能的安全约束
  safety: {
    max_files_per_cycle: 3,
    require_test_before_code: true
  }

  # 来源
  source: "project"             # project | user | system
  enabled: true
```

### Skill 生命周期

```
1. 加载 ── Agent 启动时从 Registry 加载所有已启用的 Skill
2. 匹配 ── 收到任务后，根据关键词/任务类型匹配相关 Skill
3. 激活 ── 匹配到的 Skill 内容注入 Agent 的 system prompt
4. 执行 ── Agent 按照 Skill 定义的工作流执行
5. 反馈 ── Skill 执行效果记录到 Event Store（可选）
```

### Skill 与 Tier 的协同

不同的 Skill 对模型能力要求不同，Skill 注册时声明自己需要哪一层 Tier：

| Skill 类型 | 推荐 Tier | 示例 |
|-----------|----------|------|
| 架构/设计类 | BRAIN | 系统架构设计、技术选型 |
| 开发/测试类 | WORKER | TDD、代码重构、API 开发 |
| 检查/摘要类 | FAST | 代码格式检查、变更摘要 |

如果当前可用的 WORKER Tier 模型不支持 tool_use，Agent 会将该 Skill 升级到 BRAIN Tier 的模型执行。

### Skill 注册中心

```python
class SkillRegistry:
    """管理所有 Skill 的加载和匹配"""

    def load_from_path(path) → None
        """从目录加载 Skill 文件（.md / .yaml）"""

    def match(task_description: str) → list[Skill]
        """根据任务描述匹配相关 Skill"""

    def get_skill_prompt(skill: Skill) → str
        """获取 Skill 的指令文本，用于注入 system prompt"""

    def list_all() → list[Skill]
        """列出所有已注册 Skill"""
```

### 与 Harness 安全模型的协同

- Skill 本身可以声明安全约束（如 TDD skill 要求"必须先写测试"）
- Skill 不能绕过权限门禁——它定义的流程仍然受 Harness 4 层模型约束
- Skill 中可以引用 MCP 工具——两者无缝组合
- 示例：一个"数据库迁移"Skill 通过 MCP 连接数据库，但 write 操作仍走 AUTO+（Safety Model 审查）

---

## Section 15: Token Cost Management（Token 成本管控）

> AI native 平台最大的隐性成本是 token 消耗。xagent 必须在设计层面就内置成本管控，而不是事后补救。

### 成本分层模型

```
                    ┌─────────────────────┐
  高成本           │  BRAIN Tier          │  ← 只在真正需要时用
  (谨慎使用)        │  Opus / GPT-5        │
                    ├─────────────────────┤
  中成本           │  WORKER Tier         │  ← 主力，但要优化
  (主力优化)        │  Sonnet / DeepSeek    │
                    ├─────────────────────┤
  低成本           │  FAST Tier           │  ← 能用的地方尽量用
  (优先使用)        │  Haiku / GPT-4o-mini  │
                    └─────────────────────┘
```

### 策略一：Tier 降级 — 能用便宜的绝不用贵的

Agent 每次调用 LLM 前，走降级判断：

```
需要 BRAIN？
  ├─ 真的是架构决策/冲突调解？ → 用 BRAIN
  ├─ 其实 WORKER 也能做？ → 降级到 WORKER
  └─ 只是简单判断？ → 降级到 FAST

需要 WORKER？
  ├─ 真的需要生成/修改代码？ → 用 WORKER
  └─ 只是格式检查/摘要？ → 降级到 FAST
```

**降级规则配置**：

```yaml
# xagent.yaml
cost:
  tier_downgrade:
    # 某些 task_type 可以降级
    task_type_overrides:
      "simple_query": fast       # 简单查询直接用 FAST
      "status_check": fast       # 状态检查直接用 FAST
      "message_summary": fast    # 消息摘要直接用 FAST
      "intent_parse": fast       # 意图解析直接用 FAST

    # BRAIN Tier 的降级条件
    brain_downgrade_if:
      - task_is_routine          # 常规任务不需要 BRAIN
      - confidence_high          # 高置信度不需要 BRAIN
      - single_file_scope        # 单文件范围不需要 BRAIN

  # BRAIN Tier 每日配额（超过后必须人工确认）
  brain_daily_limit: 20
```

### 策略二：Context 瘦身 — 不喂多余的东西

Context 是 token 消耗的大头。压缩策略详见 **Section 12（Context Manager — 分层压缩策略）**，此处只做成本视角补充：

| 压缩动作 | 省多少 | 成本收益分析 |
|---------|--------|------|
| 裁剪已完成的 tool 调用结果 | ~30% | 低成本高收益，默认开启 |
| 对话历史摘要（FAST Tier 做） | ~80% | 摘要本身消耗少量 token，净收益高 |
| 截断长输出（shell/diff） | ~20% | 零成本（纯文本截断） |
| 代码文件只传 diff 不传全量 | ~50% | 需要 Agent 理解 diff 上下文 |

**Context 瘦身优先级**：BRAIN > WORKER > FAST。贵模型的上下文瘦身要更激进。

### 策略三：Cache 利用 — 同样的话不说第二遍

不同模型的 cache 机制不同，Context Manager 针对性优化：

| 模型 | Cache 机制 | 优化策略 |
|------|-----------|---------|
| Claude | 手动标记 breakpoints，5min TTL | System prompt + 工具定义放前 4 个 breakpoint |
| OpenAI | 自动缓存 | 不变的前缀尽量大块（>1024 tokens） |
| DeepSeek | 磁盘 KV-cache | 相同 system prompt 复用 cache |

**Cache 命中率目标**：>80%（即 80% 的输入 token 不重复计费）。

### 策略四：Token 预算 — 任务级别的硬限制

```yaml
# xagent.yaml
cost:
  budgets:
    per_task: 50000            # 单个原子任务最多 5 万 token
    per_feature: 500000        # 单个 Feature 最多 50 万 token
    per_day: 5000000           # 每天最多 500 万 token
    per_brain_call: 20000      # BRAIN 单次调用最多 2 万 token

  # 超预算行为
  on_budget_exceeded:
    per_task: pause_and_confirm    # 暂停，人工确认是否继续
    per_feature: warn_and_downgrade # 告警，后续任务强制降级到 FAST
    per_day: hard_stop              # 硬停止，等第二天
```

### 策略五：智能重试 — 失败不重复烧钱

```
操作失败
  ├─ 网络/超时错误 → 重试（不消耗额外 token 思考）
  ├─ 模型输出格式错误 → 最多重试 2 次
  │   第 1 次：原模型修复
  │   第 2 次：降级到 FAST 修复（便宜）
  │   第 3 次：人工介入（不烧 token）
  └─ 逻辑错误 → 不重试，记录并继续
```

**关键原则**：不要让昂贵的 BRAIN Tier 模型做"格式修正"这种低级工作。

### 策略六：成本可视化

Dashboard 实时展示：

```
┌─────────────────────────────────────────────┐
│  💰 Token 消耗                              │
│                                              │
│  今日: 234,567 tokens  ≈ ¥3.52              │
│  本周: 1,234,567 tokens  ≈ ¥18.50           │
│  本月: 5,234,567 tokens  ≈ ¥78.50           │
│                                              │
│  ██████░░░░ BRAIN (任务)  30% (¥23.55)       │
│  ███░░░░░░░ BRAIN (安全)  15% (¥11.78)  ← Safety Model│
│  ██████░░░░ WORKER        35% (¥27.48)       │
│  ████░░░░░░ FAST          20% (¥15.70)       │
│                                              │
│  Feature "用户登录": 456,789 tokens (¥6.85)  │
│  最贵操作: task_042 架构决策 (¥0.52)         │
└─────────────────────────────────────────────┘
```

### 成本优化决策树（Agent 每次调用前执行）

```
要调 LLM 了
  │
  ├─ 可以用 cache 吗？ → 用，省 ~90%
  ├─ 可以降级 Tier 吗？ → 降，省 ~80%
  ├─ context 可以再压吗？ → 压，省 ~30%
  ├─ 这次调用真有必要吗？
  │   ├─ 只是格式转换？ → 不调 LLM，用代码
  │   ├─ 只是模板填充？ → 不调 LLM，用模板
  │   └─ 确实需要 LLM → 调吧
  └─ 调完之后 → 记录消耗，更新预算
```

### 成本效率指标

| 指标 | 目标值 | 说明 |
|------|--------|------|
| FAST Tier 使用占比 | >60% | 大部分调用用便宜模型 |
| Cache 命中率 | >80% | 减少重复计费 |
| BRAIN 调用/天（含 Safety Model） | <50 | 其中 Safety Model <30, 任务决策 <20 |
| Safety Model 单次审查 | <3,000 tokens | 精简输入，不含任务上下文 |
| 每次原子任务平均 token（含审查） | <15,000 | 任务 10K + Safety 审查 3K + 开销 |
| 重试率 | <5% | 一次做对，不反复烧钱 |

---

## 安全总览

| 安全关切 | 保障机制 |
|---------|-----------|
| Agent 不能做禁止的操作 | REJECT 级别硬编码，不可绕过 |
| Agent 不能自我审查 | Safety Model 独立实例审查所有 AUTO 操作 |
| Safety Model 不被任务上下文干扰 | 审查用精简输入，独立 system prompt |
| Agent 不能越界 | 沙箱隔离（文件系统、网络、进程限制） |
| 所有操作可追踪 | Event Store append-only 日志 |
| 决策不可篡改 | Decision Record hash 链（含 Safety Model 审查记录） |
| 授权受上下文限定 | 每次已确认操作前做上下文绑定检查 |
| 超时不静默失败 | 智能超时 + AI 判断 |
| 所有确认记录保留 | 多渠道送达，全部不可变记录 |

---

## 技术栈

| 层 | 技术 | 选型理由 |
|-------|-----------|-----------|
| **后端** | Python（FastAPI） | AI/LLM 生态最强，快速开发 |
| **Event Store** | SQLite → PostgreSQL | 初期简单，无缝升级到生产 |
| **前端** | React + Tailwind | 轻量 SPA |
| **实时通信** | WebSocket | Dashboard 实时更新 |
| **IM** | 企业微信 Bot API | 初始目标平台 |
| **LLM Providers** | Anthropic、OpenAI、DeepSeek | 多模型支持 |
| **沙箱** | Docker / subprocess+chroot | 进程隔离 |
| **向量数据库（未来）** | Pinecone / pgvector | 语义搜索 |

---

## 实施阶段（概要）

实施顺序已按依赖关系排列。每阶段的"前置依赖"列说明了为什么必须先做前面的。

| 阶段 | 重点 | 前置依赖 |
|-------|-------|---------|
| **Phase 1: Core Agent** | Agent 单例、基础 harness（文件+shell+git）、单模型、CLI 界面、最小 Event Store（SQLite，只为 agent 状态持久化和 failover） | 无 |
| **Phase 2: IM 接入** | 企业微信 Adapter、消息流水线、简单确认流程 | Phase 1（需要 Agent 能执行才能回复 IM） |
| **Phase 3: 任务系统** | 任务拆解、sub-agent 池、完整 Project Brain（Event Store → Task Graph + Decision Log + Code History 视图） | Phase 1（Event Store 基础在 Phase 1 已建） |
| **Phase 4: Dashboard** | Web 界面、实时动态流、任务地图、决策日志 | Phase 3（Dashboard 展示的是 Project Brain 的数据） |
| **Phase 5: 多模型** | Provider Adapter、能力注册中心、Model Router、Context Manager | Phase 1（需要 Agent 框架来接入多模型） |
| **Phase 6: 扩展机制** | MCP Client、Skill Registry、Skill 匹配和执行、Safety Model 独立实例 | Phase 5（MCP/Skill 需要多模型路由能力） |
| **Phase 7: 高级特性** | Discussion Thread、上下文绑定、智能超时、AI 自学习 Profile | Phase 2+3（Discussion 需要 IM 和任务基础） |
