# xagent — AI-Native Software Engineering Collaboration Platform

> **Design Document** | 2026-06-14 | Version 1.0

---

## Overview

xagent is an AI-native software engineering platform where an AI agent serves as the primary developer, while human team members interact through IM channels (企业微信/钉钉/飞书). The platform supports a small team (2-5 people + AI) collaborating on a single project.

### Core Principles

- **AI is the main worker** — the agent owns execution end-to-end
- **IM is the primary interface** — all human interaction flows through familiar chat tools
- **Full visibility** — every team member can inspect any part of the project at any time
- **Safety by default** — all agent actions are permission-gated, audited, and reversible
- **Context-bound decisions** — every confirmation is scoped to a specific context snapshot

---

## Section 1: Architecture Overview

### Three-Layer Architecture

```
Channel Layer  →  Agent Core  →  Project Brain
   (I/O)           (Think+Act)      (Memory)
```

| Layer | Responsibility | Principle |
|-------|---------------|-----------|
| **Channel Layer** | Translate between IM protocols and internal events | Pure I/O, no business logic |
| **Agent Core** | Message routing, task decomposition, LLM orchestration, tool execution | Central brain, owns all decisions |
| **Project Brain** | Persistent storage of all state, decisions, changes, contacts | Single source of truth |

### Component Diagram

```
┌─────────────────────────────────────────────────┐
│                CHANNEL LAYER                      │
│  企业微信 Adapter  │  钉钉 Adapter  │  飞书 Adapter  │  Web Dashboard │
└───────────────────────┬─────────────────────────┘
                        │  Channel Events
                        ▼
┌─────────────────────────────────────────────────┐
│                AGENT CORE                         │
│  Message Router  │  Task Orchestrator             │
│  Decision Escalator  │  LLM Engine  │  Tool Harness │
└───────────────────────┬─────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────┐
│              PROJECT BRAIN                        │
│  Task Graph  │  Decision Log  │  Code History     │
│  Contact Store  │  Discussion Threads             │
│  ─────────────────────────────────────            │
│  Event Store (append-only, immutable)            │
└─────────────────────────────────────────────────┘
```

---

## Section 2: Agent Core

### Agent Lifecycle

- **Singleton per project** — one agent instance per project
- **Full state persistence** — all agent state (task tree, conversation context, pending tool calls, active discussions) persisted to Project Brain
- **Periodic checkpointing** — agent snapshots state at regular intervals
- **Failover** — watchdog detects crash, restores from latest checkpoint, replays Event Store events since checkpoint, notifies team via IM

### Message Processing Pipeline

```
IM Message → Identity Resolve → Intent Parse → (Query/Execute)
                                                   │
                                          Task Decompose
                                                   │
                                          Confidence Check
                                        ┌────┼────┐
                                    SAFE │ UNCERTAIN
                                        │    │
                                   Execute  Escalate to Human
                                        │    (Discussion Thread)
                                   Broadcast
```

### Decision Escalation — Discussion Threads

When the agent is uncertain, it creates a **Discussion Thread** in the IM channel:

1. **Create thread** with question description, context, and optional deadline
2. **Select participants** — AI decides who to involve based on: code ownership, role, focus areas in contact profile
3. **Track discussion** — monitor who said what, detect conflicts
4. **Conflict types detected**:
   - Opinion conflict: A says X, B says Y
   - Temporal conflict: B's response based on stale context
   - Silence conflict: key person hasn't responded
5. **Resolution**:
   - Consensus → record decision, execute
   - Cannot reach consensus → escalate to higher authority
   - Timeout → AI judges safety, may proceed with default strategy

### Thread State Model

```
OPEN → CONFLICT → STALLED → RESOLVED
                   │          │
                   ▼          ▼
               ESCALATED    CLOSED
```

---

## Section 3: Fine-Grained Task Decomposition

### Task Hierarchy

```
Epic (user request)
 └─ Feature (logical unit)
     └─ Atomic Task (single action)
```

### Atomic Task Criteria

| Criterion | Description |
|-----------|-------------|
| **Single file** | One task touches one file (or one clear small scope) |
| **Single action** | "Create file" OR "Modify function" OR "Add test" |
| **Verifiable** | Clear success criteria (test passes, lint passes) |
| **Reversible** | Failure doesn't affect other tasks |
| **≤ 5 minutes** | Ideal execution time per atomic task |

### Task Data Model

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

### Parallel Execution

- Features can run in parallel
- Within a feature, tasks follow dependency order
- Max 5 concurrent sub-agents (to avoid LLM rate limits)
- Agent analyzes dependency graph to maximize parallelism

---

## Section 4: Harness Security — 4-Layer Model

### Layer 1: Capability Boundary
Agent can only call registered tools. Unregistered capabilities don't exist.

**Default tools**: `file_read`, `file_write`, `shell_exec`, `git_*`, `web_fetch`, `lsp_query`
**Plugins**: `browser`, `db_query`, `docker`, …
**Hard-forbidden**: `file_delete` (default), `network_bind`, `sudo`

### Layer 2: Permission Gate

| Level | Behavior | Example |
|-------|----------|---------|
| 🟢 **AUTO** | Execute, log after | file_read, git log, test_run |
| 🟡 **AI_REVIEW** | AI self-checks first; if safe → auto; if uncertain → escalate | file_write, shell_exec, git_commit |
| 🔴 **CONFIRM** | Always ask human | pip_install, config_change |
| ⛔ **REJECT** | Hard-coded deny, cannot override | file_delete, git_push --force |

### Layer 3: Sandbox Isolation

- **FS**: Agent can only read/write project directory + explicit allowlist
- **Network**: Domain allowlist (pypi.org, github.com, …)
- **Process**: ulimit, no fork bombs
- **Time**: Single shell command max 5 minutes
- **Implementation**: Docker container (production), subprocess + chroot + seccomp (initial)

### Layer 4: Audit Trail

Every tool call logged to Event Store:
```yaml
{tool, input, output, duration, permission_level,
 auto_approved, sandbox_id, files_touched, snapshot_diff}
```

### Rollback Mechanism

- **Git-level** (coarse): auto-stash before feature, reset if needed
- **Overlay-level** (fine): per-task overlay delta; discard on failure without affecting parallel tasks
- **Triggers**: agent self-detected failure, Dashboard click, IM command

---

## Section 5: Multi-Channel Confirmation & Decision Records

### Confirmation Flow

1. Agent needs confirmation → generates Confirmation Request with context snapshot
2. Dispatches to **all channels simultaneously** (IM DM + group @mention + Dashboard)
3. First response from any channel takes effect
4. Timeout (30 min) → AI judges whether to proceed or escalate

### Decision Record (Immutable)

```yaml
DecisionRecord:
  id, request_id
  type: [git_commit|file_write|shell_exec|architecture|…]
  summary, context: {task_id, files, diff_hash, risk_level}
  decided_by: contact_id | "ai_review"
  channel: [wecom|dashboard|…]
  decision: [approved|rejected|expired]
  decided_at, comment
  hash: sha256(entire_record)
  prev_hash: sha256(previous_record)  # chain for tamper-proofing
```

### Decision Log Queries

- "Who approved changes to this file?" → filter by file
- "What decisions did 张三 make recently?" → aggregate by decided_by
- "Which confirmations timed out?" → filter by expired
- "What was discussed for Feature X?" → join with discussion threads

---

## Section 6: Context Binding & Smart Timeout

### Context Binding Principle

> A confirmation means "you may execute THIS specific operation in THIS specific context." If the context changes, authorization is void.

Before executing any confirmed action, the agent checks:

| Check | Mismatch Behavior |
|-------|-------------------|
| File content hash unchanged? | Re-confirm (someone else modified the file) |
| Dependent tasks still satisfied? | Re-confirm (preconditions changed) |
| Branch unchanged? | Re-confirm (wrong branch) |
| Permission config unchanged? | Re-confirm (rules changed) |
| Decision-maker still valid? | Re-route (person left/changed role) |

### Smart Timeout

Instead of auto-reject on timeout:
1. AI judges: is the context still clean? Is the risk low?
2. Clean context + low risk → AI confirms and proceeds (logged)
3. Context changed → regenerate confirmation request
4. High risk or uncertain → @remind + reset TTL

Applies equally to AI_REVIEW and CONFIRM levels.

---

## Section 7: Project Brain — Data Model

### Architecture

```
Task Graph  │  Decision Log  │  Code History  │  Contact Store
     └────────────┴───────────────┴───────────────┘
                        │
                        ▼
              Event Store (append-only, immutable)
```

### Core Entities

| Entity | Key Fields | Typical Queries |
|--------|-----------|-----------------|
| **Task** | id, parent_id, type, status, depends_on, tool_calls[], decisions[] | "What's in progress?", "What blocks task_042?" |
| **Decision** | id, type, context_snapshot, decided_by, channel, hash | "Who approved this?", "Recent rejections?" |
| **Change** | id, type, file_path, diff_hash, tool, task_id | "History of auth.py", "What changed in last 24h?" |
| **Contact** | id, name, im_accounts[], role, permissions, focus_areas[], profile_text | "Who owns this module?", "张三's focus areas?" |

### Discussion Thread Entity

```yaml
DiscussionThread:
  id, title, question, status
  participants[], channel
  messages: [{from, content, at}]
  conflict_events: [{type, between, topic, at}]
  resolution: {type, final_choice, reason}
  → DecisionRecord (resolution recorded as Decision)
```

### Storage

| Layer | Technology | Notes |
|-------|-----------|-------|
| Event Store | SQLite (init) / PostgreSQL (prod) | Append-only table, timestamp-indexed |
| Materialized Views | Same DB views / in-memory cache | Real-time projections of Event Store |
| File Storage | Local FS + Git | Git provides version history natively |
| Vector Store (future) | Pinecone / pgvector | Semantic search over history |

---

## Section 8: Dashboard — Project Living Map

### Layout

```
┌────────────────────────────────────────────┐
│  xagent / my-project        🔔 3 pending   │
├────────────────────────────────────────────┤
│  📊 Task Progress    │  🤖 Agent Status    │
│  ████████░░ 18/22   │  Executing task_044 │
│  User Login 82%     │  3 sub-agents active │
├────────────────────────────────────────────┤
│  ⚡ Live Feed                              │
│  14:32 ✅ task_042 completed               │
│  14:31 🤖 AI self-review passed            │
│  14:28 📝 task_045 started                 │
├─────────────────────┬──────────────────────┤
│  🔑 Pending (3)     │  💬 Discussions (1)  │
│  pip install bcrypt │  "Redis or MySQL?"   │
│  [Approve] [Reject] │  [View Thread]       │
└─────────────────────┴──────────────────────┘
```

### Pages

| Page | Content |
|------|---------|
| **Overview** | Task progress + Agent status + Live feed + Pending list |
| **Task Map** | Task tree visualization, dependency graph |
| **Decision Log** | All Decision Records, filterable, hash chain verifiable |
| **Code Changes** | File change history, diffs, linked tasks/decisions |
| **Discussions** | All threads by status (active/resolved/escalated) |
| **Contacts** | Team list, profiles, activity stats |

### Tech Stack

- **Frontend**: React + Tailwind (lightweight SPA)
- **Real-time**: WebSocket (Event Store changes pushed to frontend)
- **Data**: REST API reading from Project Brain views

---

## Section 9: Contact System

### Profile Model

```yaml
Contact:
  id, name
  im_accounts: [{platform, id}]
  role, permissions, focus_areas[]
  profile_text: "自然语言描述，类 claude.md 格式"
```

### Profile Evolution

| Phase | Content |
|-------|---------|
| **Phase A (initial)** | Role, permissions, focus areas, profile text (static) |
| **Phase C (future)** | AI-learned: recent focus, decision preferences, collaboration patterns, active hours, interaction stats |

### Usage

- Agent reads profile to decide escalation targets
- profile_text provides communication style guidance
- Interaction history builds up over time for context-aware routing

---

## Section 10: IM Channel

### Adapter Interface

```python
class IMAdapter:
    platform: "wecom" | "dingtalk" | "feishu"

    # Inbound: IM → Agent
    def on_message(msg) → ChannelEvent:
        # Normalizes to: from_contact, channel_type, content, mentions, reply_to

    # Outbound: Agent → IM
    def send_message(target, formatted_msg)
    def send_confirmation(target, confirm_request)
    def create_discussion_thread(group, question)
```

### Message Routing

| Message Type | Delivery |
|-------------|----------|
| Progress notification | Project group (visible to all) |
| Confirmation request | DM + group @mention + Dashboard |
| Discussion initiation | Group thread, @required participants |
| Query response | Reply to source channel |
| Error/exception | Group broadcast + DM tech lead |

### Initial Platform

Start with 企业微信 (WeCom). Architecture supports adding 钉钉, 飞书 later.

---

## Section 11: LLM Engine — Multi-Model Capability Abstraction

### Architecture

```
Agent Request → Model Router → Provider Adapter → Model API
                      │
              Capability Registry
```

### Three-Tier Model

| Tier | Role | Required Capabilities | Example Tasks |
|------|------|----------------------|---------------|
| **BRAIN** | Reasoning & decisions | reasoning ✓, tool_use ✓, large context | Task decomposition, architecture, conflict resolution |
| **WORKER** | Code generation & execution | code ✓, tool_use ✓, medium context | Code gen, code review, test gen, docs |
| **FAST** | Simple queries | text ✓, small context | Intent parsing, message summary, status checks |

### Capability Profile

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

### Vision Fallback Strategies

When a model lacks vision capability:
1. **Re-route** (default): temporarily switch to a vision-capable model for that call
2. **Pre-process**: call vision model to describe image, feed text to non-vision model
3. **Degrade**: skip image, use text-only context

### Capability Fallback

| Missing Capability | Strategy |
|-------------------|----------|
| vision | re_route |
| tool_use | error (can't work) |
| reasoning | degrade to lower tier |

---

## Section 12: Context Manager

### Router + Context Manager Coordination

```
Router selects model → Context Manager builds request for that model
```

The Router owns "which model", the Context Manager owns "what to feed it."

### Context Profile (per model)

Each model declares in its Capability Profile:
- `max_tokens`, `effective_tokens` (reserving space for output)
- Cache mechanism, TTL, breakpoint count
- Preferred compression strategy
- Message format overhead estimate

### Layered Compression

When context exceeds model limit:

| Layer | Strategy | Savings |
|-------|----------|---------|
| L1 | Trim irrelevant messages (completed tool results, intermediate queries) | ~30% |
| L2 | Summarize history via FAST Tier model | ~80% |
| L3 | Truncate long outputs (shell results, diffs) | ~20% |
| L4 | Split task into smaller sub-tasks (emergency) | Variable |

### Context Bridge (Cross-Model)

When the Router switches models mid-session (e.g., DeepSeek → Claude for vision):

1. Compress full conversation history into a summary (using FAST Tier model)
2. Pass summary + current task context + new input to target model
3. Target model gets: system prompt + condensed history + task state + current request
4. NOT the full 47-message conversation

### Core Interface

```python
class ContextManager:
    def build_request(model, system_prompt, messages, tools, task_context) → PreparedRequest
    def bridge_context(from_model, to_model, history, task) → list[Message]
    def estimate_tokens(messages, model) → int
    def update_after_response(response, model)
```

---

## Security Summary

| Concern | Mechanism |
|---------|-----------|
| Agent can't do forbidden things | REJECT level hard-coded, cannot override |
| Agent can't exceed scope | Sandbox isolation (FS, network, process limits) |
| Every action is traceable | Event Store append-only log |
| Decisions can't be tampered with | Hash chain on Decision Records |
| Authorization is context-scoped | Context binding checks before every confirmed action |
| Timeouts don't silently fail | Smart timeout with AI judgment |
| All confirmations preserved | Multi-channel delivery, all records immutable |

---

## Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Backend** | Python (FastAPI) | AI/LLM ecosystem, rapid development |
| **Event Store** | SQLite → PostgreSQL | Simple start, production-ready upgrade |
| **Frontend** | React + Tailwind | Lightweight SPA |
| **Real-time** | WebSocket | Live Dashboard updates |
| **IM** | 企业微信 Bot API | Target initial platform |
| **LLM Providers** | Anthropic, OpenAI, DeepSeek | Multi-model support |
| **Sandbox** | Docker / subprocess+chroot | Isolation |
| **Vector DB (future)** | Pinecone / pgvector | Semantic search |

---

## Implementation Phases (High-Level)

| Phase | Focus |
|-------|-------|
| **Phase 1: Core Agent** | Agent singleton, basic harness (file+shell+git), single-model, CLI interface |
| **Phase 2: IM Integration** | WeCom adapter, message pipeline, simple confirmation flow |
| **Phase 3: Task System** | Task decomposition, sub-agent pool, Project Brain Event Store |
| **Phase 4: Dashboard** | Web UI, live feed, task map, decision log |
| **Phase 5: Multi-Model** | Provider adapters, capability registry, model router |
| **Phase 6: Advanced** | Discussion threads, context binding, smart timeout, AI-learned profiles |
