# LoongBoard V1 技术开发与多 Agent 执行方案

> 文档状态：可执行基线
>
> 面向对象：Codex 多 Agent 协同系统、技术负责人 Agent、实现 Agent、测试 Agent
>
> 产品形态：本地优先、单用户、浏览器界面、本地仓库与知识库、外部调用 DeepSeek Harness
>
> DSH 基线：`deepseek-ai/deepseek-harness@dsh-v0.1.2-alpha.5`
>
> 已检查的 DSH master：`49a606bc5b5934603f22a26957a07dc799ab0291`（2026-09-02）

---

## 0. 给多 Agent 团队的总指令

本项目必须从零新建，不复制旧 `loong-dashboard` 的架构、Local Runner、Vue 页面、Cloudflare Worker、D1、OpenCode/Codex Adapter 或事件归一化实现。旧仓库仅作为需求和视觉参考。

技术负责人必须遵守以下顺序：

1. 先固化仓库结构、架构边界、契约、数据库迁移和检查命令。
2. 按本文的纵向切片逐阶段交付，不同时铺开所有页面。
3. 每个子 Agent 只承担一个边界明确的任务，必须同时提交对应测试。
4. 共享契约、数据库 schema、DSH 适配层由负责人或指定高级 Agent 独占修改，避免并行冲突。
5. 不实现本文明确列入“V1 不做”的功能。
6. 不进行推测性抽象，不为未来可能出现的第二个实现提前建立复杂框架。
7. 只在外部边界做一次运行时校验；内部依赖 TypeScript 类型、数据库约束和明确的不变量。
8. 每个阶段完成后执行统一检查，负责人验收后再进入下一阶段。

本项目的目标不是“功能最多”，而是：

- 页面刷新快；
- 数据含义清楚；
- Agent 与 DSH 解耦；
- Markdown 和 Git 是长期数据；
- 代码对维护 Agent 友好；
- 发生问题时错误明确，而不是静默降级或堆叠兜底。

---

# 1. 已确认的产品需求

## 1.1 Repository PR 页面

一个仓库只有一个统一 PR 列表，不拆“今日合入”“今日活跃”等独立页面。

默认排序：

```text
last_activity_at DESC
```

`last_activity_at` 第一版直接使用 GitHub `updatedAt`。不查询完整 timeline，也不展示“最后一次活动具体是评论、改标题还是 push”。

页面支持：

- 最近活跃模式；
- 按日历日期筛选：查看“最后活跃时间落在某一天”的 PR；
- 状态筛选：`Draft / Open / Closed / Merged`；
- 多领域标签筛选；
- 展示 PR 标题、编号、作者、状态、最后活跃时间；
- 展示 `changed files / additions / deletions`；
- 展示多个确定性领域标签；
- 手动刷新及自动刷新状态。

Merge 只是状态筛选，不建立独立页面。数据库仍分别保存 `updatedAt` 和 `mergedAt`，避免概念混淆。

## 1.2 PR 领域分类

PR 领域分类不使用 AI。

每个仓库可由用户维护多条领域规则：

```yaml
name: CI
include:
  - .github/**
  - ci/**
  - scripts/ci/**
exclude: []
```

分类唯一依据是 PR 的 changed file paths。一个 PR 可以命中多个领域，不设置主领域，也不计算置信度。

规则更新后只在本地重新分类，不重新请求 GitHub。

## 1.3 Issue 页面

Issue 第一版不做领域分类。

默认按 GitHub `updatedAt` 倒序，支持按日历日期筛选最后活跃日期，并展示：

- 标题、编号；
- Open / Closed；
- 作者；
- 评论数；
- 最后活跃时间；
- GitHub 链接。

Issue 详情按需读取正文与评论。列表刷新不读取评论正文和 timeline。

## 1.4 PR 详情和代码阅读

PR 详情需要形成 GitHub Changes 风格的工作区：

```text
┌──────────────────┬──────────────────────────────┬──────────────────┐
│ Changed Files    │ Code Diff / Full File        │ Agent Chat       │
└──────────────────┴──────────────────────────────┴──────────────────┘
```

核心要求：

- 左侧 changed files 树；
- 中间显示 Base 与 Head 的代码差异；
- “Changes”模式折叠大部分未修改区域；
- “Full File”模式展示完整文件，但仍保留增删改标记；
- 不跳转到另一套仓库代码浏览器；
- Agent Chat 在右侧持续存在；
- 不自动把当前文件、选择范围、页面状态注入 Agent；
- 用户直接提问，Agent 根据当前本地仓库自行执行 `git diff`、搜索和阅读；
- 提供“复制本地 PR 命令”按钮，例如：

```bash
git fetch upstream pull/123/head:pr-123 && git switch pr-123
```

远端名由仓库设置决定，不写死为 `upstream`。

## 1.5 Agent Chat

Agent 是普通 coding agent，不建立产品级只读/写入权限档案。

- PR Chat：工作目录为对应 PR Head 的本地 worktree；
- Issue Chat：默认工作目录为该仓库主目录或默认分支目录；
- Knowledge Chat：工作目录为整个知识库 Git 仓库；
- Agent 可访问整个本地 `system/`，限制主要通过 `AGENTS.md` 软规则；
- 不设计 Analyze PR 专用工作流；
- 不自动生成 PR 摘要；
- 不写 DSH Plugin，优先直接调用 DSH SDK；
- 一个 Chat Session 可以读取、修改、创建、移动整个工作区内文件。

## 1.6 Knowledge Repository

知识库是普通 Git 仓库，内容以标准 Markdown 文件为源数据。

V1 功能：

- 文件夹树；
- `inbox/` 暂存区；
- 新建、移动、重命名、删除 Markdown；
- Markdown 源码编辑与渲染预览；
- GFM、表格、代码块、图片、链接、Mermaid；
- 每篇文档可映射一个默认 Chat Session；
- 文档移动后默认 Chat 映射不丢失；
- Agent 工作目录是整个知识库，而不是单篇文档；
- Agent 可根据用户指令续写或修改任何文档；
- 保存历史只保留最近 10 次有内容变化的版本；
- 用户手工保存算一次；一次 Agent 运行结束后产生的最终文件变化算一次；
- 长期版本历史由 Git 提供。

V1 不做“小贴士”、知识图谱、RAG 产品层、自动目录整理或自定义 Markdown 语法。

## 1.7 定时任务

定时任务的本质是：

```text
到达时间 → 将用户保存的 Prompt 发给 Agent → Agent 操作工作区
```

Prompt 自己说明要创建报告、更新报告、提交 Git 或执行其他工作。Scheduler 不理解报告结构，不实现 Workflow Builder。

任务字段只需：

- 名称；
- Cron/时间配置；
- 时区；
- Prompt；
- 工作目录；
- 使用的模型配置；
- 启用状态；
- 上次运行、下次运行、运行历史；
- 手动立即运行。

第一版每次定时执行创建新 Agent Session，不复用长期历史，避免定时会话无限增长。

## 1.8 本地目录

系统默认假设：

```text
system/
├── vllm/
├── vllm-ascend/
├── loong-dashboard/
├── knowledge/
├── dsh/                    # 可选，仅用于研究 upstream
├── .worktrees/
├── .loong/
├── system.yaml
└── AGENTS.md
```

`system/` 本身不是 Git 仓库；各子目录独立管理 Git。

仓库归属与远程维护约定（2026-09-03 明确，避免与个人开发仓混淆）：

- 目录内 `vllm/`、`vllm-ascend/` 是“被管理源码宿主”，即 LoongBoard 专用工作副本：平台只对它们执行 fetch，并把它们作为 `.worktrees/` 中 detached worktree 的宿主仓库；这些目录不做个人开发编辑。
- 个人开发使用独立 clone / fork（例如用户主目录下的个人工作仓），与被管理宿主彻底分离：worktree slot 与宿主仓库一律不得指向个人开发仓库，个人改动经 fork → PR 回流。
- `system.yaml` 只存在于本机容器根，本身不纳入任何 Git 仓库；需要分发/共享时提交脱敏模板 `system.example.yaml`。
- 远程维护采用多仓并行：`loong-dashboard/`、`knowledge/`、顶层配置与文档各自独立推送到 GitHub（默认私有）；`vllm/`、`vllm-ascend/` 等被管理宿主不纳入自有 GitHub 仓库（其上游本身就是公开源）。



## 1.9 V1 明确不做

- 多用户、组织、RBAC；
- 云部署；
- PR/Issue AI 自动摘要；
- Issue 领域分类；
- Review Risk Score；
- 自动 Surface Context；
- 独立源码浏览器、调用图、LSP UI；
- GitHub 写操作；
- DSH Web UI 集成；
- DSH Plugin；
- DSH 原始 SessionEvent 对外暴露；
- 复杂 Agent 权限系统；
- 复杂工作流编排器；
- 自动整理知识库；
- 多人实时协作编辑；
- 复杂重试、补偿事务和兼容层。

---

# 2. 工程原则

## 2.1 用户价值优先于字段完整度

任何展示字段必须评估：

```text
用户价值 / 请求成本 / 刷新频率 / 失败影响
```

如果一个字段需要显著增加 GitHub 请求、导致刷新超时，却不能改变用户决策，则舍弃、延迟到详情页，或使用已有低成本字段近似。

本项目的典型决定：

- 使用 `updatedAt` 作为最后活跃时间；
- 不获取最后活动具体事件类型；
- 不在列表读取评论正文、Review 和 timeline；
- changed files 只在新 PR 或 Head SHA 变化时获取；
- 完整 Diff 从本地 Git 按需生成。

## 2.2 Local-first

所有核心数据和操作发生在本机：

- Git 仓库；
- Worktree；
- Knowledge Markdown；
- SQLite；
- DSH 子进程；
- 浏览器访问本地 Node 服务。

V1 不引入 PostgreSQL、Redis、消息队列、Docker 编排或云端控制面。

## 2.3 DSH 是外部 Agent Runtime

业务代码只认识本项目定义的 `AgentRuntime` 接口。

只有 `packages/agent-runtime-dsh/**` 可以导入 `@deepseek-ai/*`。Web、数据库、GitHub、Knowledge、Scheduler 不得引用 DSH 类型。

## 2.4 外部边界校验一次，内部相信类型

运行时校验只放在：

- `system.yaml` 读取；
- HTTP API 输入；
- `gh` JSON 输出；
- DSH SDK 通知；
- 数据库迁移和约束。

进入内部领域对象后，不在每层重复做 `null`、类型、路径和状态检查。

## 2.5 Fail fast，不静默兜底

- 外部命令失败，返回带命令和 stderr 摘要的明确错误；
- 不 `catch` 后返回空数组；
- 不把失败伪装成“没有数据”；
- 不在多个层重复重试；
- 计划任务失败记录一次，等待下一次计划或手动重跑；
- 元数据同步成功、文件路径补充失败时允许部分成功，但必须显式标记 enrichment pending。

## 2.6 不做推测性抽象

只有两个真正的替换边界需要接口：

1. `GitHubMetadataProvider`；
2. `AgentRuntime`。

其余模块优先使用直接函数、明确服务和普通 TypeScript 类型。禁止引入通用 DI 容器、事件总线框架、仓储基类、Result Monad 泛滥或多层 Facade。

## 2.7 Git 和 Markdown 是源数据

- Knowledge 文档以文件系统为真；
- SQLite 只保存索引、映射、短期版本和运行状态；
- Worktree 是缓存；
- Agent Session 是用户资产；
- Git 是长期恢复机制。

---

# 3. 当前 DSH 基线与接入决定

## 3.1 已检查版本

截至 2026-09-02：

```text
Release: dsh-v0.1.2-alpha.5
Version: 0.1.2-alpha.5
Master: 49a606bc5b5934603f22a26957a07dc799ab0291
Node: ^22.19.0 || >=24.0.0
```

实现固定使用 Release，不直接依赖 master。

根目录建立：

```json
{
  "version": "0.1.2-alpha.5",
  "tag": "dsh-v0.1.2-alpha.5",
  "masterInspected": "49a606bc5b5934603f22a26957a07dc799ab0291"
}
```

文件名：`dsh.lock.json`。

## 3.2 使用 TypeScript SDK 子进程模式

依赖：

```json
{
  "@deepseek-ai/dsh": "0.1.2-alpha.5",
  "@deepseek-ai/dsh-sdk-client": "0.1.2-alpha.5"
}
```

使用 `@deepseek-ai/dsh-sdk-client` 启动 `dsh --profile sdk`，通过 stdio JSON-RPC 驱动。

选择原因：

- 官方明确用于进程外 TypeScript 调用；
- cwd、provider、model、reasoning effort 在初始化时传入；
- 可发送 Prompt；
- 可收到 Session Event 和 Agent Status 通知；
- 可使用持久 Session ID 恢复上下文；
- DSH 进程可独立更新和替换；
- 不依赖 DSH Web、Cordis UI、内部 Agent Loop 或内部数据库。

## 3.3 当前 SDK 限制及我们的处理

当前 SDK 协议：

- 无协议版本协商；
- 无 prompt cancel；
- 无 session close；
- cancel 等价于关闭运行时进程；
- 通知中仍包含 DSH SessionEvent 词汇，预发布版本可能变动。

因此采用：

- 一个 LoongBoard Chat Session 对应一个惰性 DSH 子进程；
- Cancel 只终止该 Session 的进程，不影响其他 Chat；
- 进程空闲一段时间后关闭，后续从该 Session 独立的 DSH_HOME 恢复；
- DSH 通知只在适配包中解析并转换为 LoongBoard ChatEvent；
- 数据库和前端永远不保存或依赖 DSH 原始类型。

## 3.4 V1 不写 DSH Plugin

使用 DSH 已有文件、Shell、Git、Web 等 coding agent 能力。除非以后出现“Agent 必须直接调用 LoongBoard 内部领域 API”的明确需求，否则不写 LoongBoard DSH Plugin。

## 3.5 权限配置

本项目是用户本机的可信开发环境。DSH 子进程默认：

```text
DSH_PERMISSION_MODE=danger-full-access
```

并继承本机环境中的模型凭据。约束依赖 `system/AGENTS.md` 和各仓库 `AGENTS.md`，不在 LoongBoard 代码中再实现一套权限系统。

---

# 4. 总体技术架构

```text
Browser
  │ REST + SSE
  ▼
LoongBoard Local Server
  ├── Repository API
  ├── GitHub Sync
  ├── Git Workspace Manager
  ├── DSH Session Supervisor
  ├── Knowledge Repository Service
  ├── Scheduler
  └── SQLite
       │
       ├──────── gh api graphql / gh api ───────► GitHub
       ├──────── git commands ──────────────────► Local Repositories
       ├──────── files ─────────────────────────► Knowledge Repository
       └──────── stdio JSON-RPC ────────────────► DSH SDK Runtime Processes
```

V1 只运行两个长期进程：

1. 浏览器；
2. LoongBoard Node Server。

DSH 按 Chat Session 惰性创建子进程，不单独部署服务。

---

# 5. 技术栈

## 5.1 运行环境

- Node.js 24；
- TypeScript；
- pnpm workspace；
- Git；
- GitHub CLI `gh`，用户已完成 `gh auth login`；
- SQLite。

## 5.2 Web

- React；
- Vite；
- TanStack Query；
- React Router 或 TanStack Router，二选一后固定；
- Monaco Editor：代码 Diff、完整文件 Diff、Markdown 源码编辑；
- `react-markdown` + `remark-gfm`；
- Mermaid；
- 原生 CSS Modules 或普通模块化 CSS，不引入大型 UI 框架作为第一版前提。

## 5.3 Server

- Fastify；
- Zod：HTTP 和配置边界；
- Drizzle ORM；
- `better-sqlite3`；
- `execa`：执行 `git`、`gh`；
- `picomatch`：领域文件规则；
- `chokidar`：Knowledge 文件监听；
- `cron-parser`：计划时间计算；
- SSE：Agent 流式事件。

## 5.4 测试

- Vitest；
- fast-check；
- Playwright；
- 临时 Git fixture repositories；
- Fake `gh` executable / recorded JSON fixtures。

所有第三方依赖在项目初始化时选择确切版本并写入 lockfile，不使用宽泛的 major floating。

---

# 6. 新仓库结构

```text
loong-dashboard/
├── apps/
│   ├── web/
│   │   ├── src/
│   │   ├── AGENTS.md
│   │   └── package.json
│   └── server/
│       ├── src/
│       ├── AGENTS.md
│       └── package.json
├── packages/
│   ├── contracts/
│   ├── database/
│   ├── github/
│   ├── git-workspace/
│   ├── agent-runtime/
│   ├── agent-runtime-dsh/
│   ├── knowledge/
│   └── scheduler/
├── docs/
│   ├── requirements.md
│   ├── architecture.md
│   ├── data-model.md
│   ├── github-sync.md
│   ├── dsh-integration.md
│   ├── testing.md
│   ├── operations.md
│   ├── implementation-status.md
│   └── adr/
├── tests/
│   ├── fixtures/
│   ├── integration/
│   ├── e2e/
│   └── dsh-compat/
├── scripts/
├── AGENTS.md
├── README.md
├── ARCHITECTURE.md
├── package.json
├── pnpm-workspace.yaml
├── tsconfig.json
├── eslint.config.js
├── vitest.config.ts
├── playwright.config.ts
├── dsh.lock.json
└── system.example.yaml
```

包数量保持有限。不要把每个业务名词拆成独立 package。

---

# 7. 本地 System Workspace

## 7.1 示例配置

```yaml
version: 1
timezone: Asia/Shanghai

repositories:
  - key: vllm
    name: vLLM
    github: vllm-project/vllm
    path: ./vllm
    remote: upstream
    defaultBranch: main
    worktreeSlots: 3

  - key: vllm-ascend
    name: vLLM Ascend
    github: vllm-project/vllm-ascend
    path: ./vllm-ascend
    remote: upstream
    defaultBranch: main
    worktreeSlots: 3

  - key: loong-dashboard
    name: LoongBoard
    github: MrZ20/loong-dashboard
    path: ./loong-dashboard
    remote: origin
    defaultBranch: main
    worktreeSlots: 2

knowledge:
  path: ./knowledge
  inbox: inbox
  historyLimit: 10

runtime:
  statePath: ./.loong
  worktreesPath: ./.worktrees
  serverHost: 127.0.0.1
  serverPort: 4174

agent:
  defaultProvider: deepseek-official
  defaultModel: deepseek-v4-flash
  defaultReasoningEffort: high
  idleProcessMinutes: 20
```

所有相对路径相对于 `system.yaml` 所在目录解析一次，内部统一使用绝对路径。

## 7.2 全局 AGENTS.md

`system/AGENTS.md` 至少说明：

- 各目录用途；
- `.worktrees/` 和 `.loong/` 是缓存/运行状态；
- `knowledge/` 是长期 Markdown；
- 分析 PR 时优先使用当前工作目录；
- 自动报告默认写入 `knowledge/inbox/`；
- 不在没有明确要求时删除长期知识；
- 修改代码或知识后可以提交 Git。

---

# 8. 数据模型

## 8.1 核心表

### repositories

```text
id
key UNIQUE
display_name
github_owner
github_name
local_path
remote_name
default_branch
worktree_slots
enabled
created_at
updated_at
```

### repository_sync_state

```text
repository_id
entity_kind            # pull_request | issue
watermark_updated_at
last_attempt_at
last_success_at
status                 # idle | running | failed
last_error
rate_limit_remaining
rate_limit_reset_at
```

### pull_requests

```text
repository_id
node_id
number
title
url
author_login
state_raw
status                 # draft | open | closed | merged
is_draft
created_at
updated_at
closed_at
merged_at
base_ref_name
head_ref_name
head_sha
additions
deletions
changed_files_count
detail_body             # 详情按需填充
PRIMARY KEY(repository_id, number)
```

### pull_request_files

```text
repository_id
pr_number
head_sha
path
previous_path
change_type
additions
deletions
PRIMARY KEY(repository_id, pr_number, head_sha, path)
```

### domain_rules

```text
id
repository_id
name
color
position
enabled
include_patterns_json
exclude_patterns_json
created_at
updated_at
```

### pull_request_domains

```text
repository_id
pr_number
domain_rule_id
classification_key
PRIMARY KEY(repository_id, pr_number, domain_rule_id)
```

### issues

```text
repository_id
node_id
number
title
url
author_login
state
comments_count
created_at
updated_at
closed_at
detail_body
PRIMARY KEY(repository_id, number)
```

### agent_sessions

```text
id
scope_type              # pr | issue | knowledge | general
repository_id
pr_number
target_sha
knowledge_document_id
dsh_session_id
dsh_home_path
workspace_path
provider
model
reasoning_effort
status
created_at
last_used_at
```

### agent_messages

```text
id
session_id
sequence
role                    # user | assistant | tool | system-status
content_markdown
metadata_json
created_at
```

### worktree_slots

```text
id
repository_id
slot_name
path
pr_number
target_sha
busy_session_id
last_used_at
```

### knowledge_documents

```text
id
path UNIQUE
title
content_hash
default_session_id
created_at
updated_at
```

### document_versions

```text
id
document_id
version_number
content
source                  # manual | agent | external | restore
agent_run_id
created_at
```

### scheduled_tasks

```text
id
name
cron_expression
timezone
prompt
workspace_path
provider
model
reasoning_effort
enabled
last_run_at
next_run_at
created_at
updated_at
```

### scheduled_task_runs

```text
id
task_id
scheduled_for
started_at
finished_at
status                  # running | completed | failed | skipped
agent_session_id
error
```

## 8.2 状态派生

```ts
function derivePullRequestStatus(input: {
  isDraft: boolean
  state: 'OPEN' | 'CLOSED' | 'MERGED'
  mergedAt: string | null
}): 'draft' | 'open' | 'closed' | 'merged' {
  if (input.mergedAt || input.state === 'MERGED') return 'merged'
  if (input.isDraft) return 'draft'
  if (input.state === 'CLOSED') return 'closed'
  return 'open'
}
```

只保留这一处派生逻辑，前端不得重复实现。

## 8.3 时间含义

- 数据库存 UTC；
- 日历分组在查询时按 `system.yaml.timezone` 转换；
- PR/Issue “最后活跃日”均由 `updated_at` 得出；
- Merge 日期由 `merged_at` 单独筛选，第一版 UI 可不暴露。

---

# 9. GitHub 数据同步

## 9.1 Provider 边界

```ts
interface GitHubMetadataProvider {
  fetchPullRequestUpdates(input: PullRequestSyncInput): AsyncIterable<PullRequestPage>
  fetchIssueUpdates(input: IssueSyncInput): AsyncIterable<IssuePage>
  fetchPullRequestFiles(input: PullRequestFilesInput[]): Promise<PullRequestFilesResult[]>
  fetchPullRequestDetail(repository: RepositoryRef, number: number): Promise<PullRequestDetail>
  fetchIssueDetail(repository: RepositoryRef, number: number): Promise<IssueDetail>
}
```

V1 唯一实现：`GhGitHubMetadataProvider`。

## 9.2 为什么使用 `gh api graphql`

- 复用用户已有 `gh auth`；
- 一次命令获取一页多个 PR/Issue；
- 可显式选择字段；
- GraphQL 支持 `UPDATED_AT` 排序；
- 避免为每个 PR 运行 `gh pr view`；
- 后续可替换为 Octokit，但上层契约不变。

不要在循环中执行：

```text
100 PR × gh pr view
```

## 9.3 列表字段

PR 元数据查询只请求：

```text
id
number
title
url
state
isDraft
author.login
createdAt
updatedAt
closedAt
mergedAt
baseRefName
headRefName
headRefOid
additions
deletions
changedFiles
```

Issue 元数据查询只请求：

```text
id
number
title
url
state
author.login
comments.totalCount
createdAt
updatedAt
closedAt
```

列表查询不请求：

- body；
- comments nodes；
- reviews；
- timelineItems；
- diff patch；
- CI；
- labels。

## 9.4 Bootstrap

首次同步按以下四个流执行：

1. 全部 Open PR；
2. 最近 90 天更新的 Closed/Merged PR；
3. 全部 Open Issue；
4. 最近 90 天更新的 Closed Issue。

90 天作为默认可配置 lookback。分页按 `updatedAt DESC`，最近关闭流读到阈值后停止。

Open 历史需要全部获取，确保长期未活跃但仍打开的事项可见。

## 9.5 增量同步

每个实体类型保存成功 watermark。

增量查询所有状态，按 `updatedAt DESC` 获取，直到：

```text
node.updatedAt < watermark - 2 minutes
```

两分钟 overlap 用于边界和时钟误差；所有写入幂等 upsert。只有整个元数据流成功后推进 watermark。

不因为 changed files 补充失败而回滚元数据或阻止 watermark。

## 9.6 Changed file path enrichment

需要重新获取 changed file paths 的条件：

```text
新 PR
OR head_sha 发生变化
OR 当前 head_sha 没有文件记录
```

不因为 base branch 前进而重新获取。这里有意选择速度优先；领域分类只需要识别 PR 自身改动的大致目录。

执行策略：

1. 收集本轮需要补充文件的 PR Node IDs；
2. 每批最多 20 个，通过 GraphQL `nodes(ids: ...)` 获取每个 PR 的 `files(first: 100)`；
3. 若某个 PR `hasNextPage=true`，只对该 PR 调用 REST files endpoint，并使用 `per_page=100` 分页；
4. GitHub REST 对单个 PR 最多返回 3000 个文件；超过该限制标记 `files_truncated=true`，V1 不继续解决。

文件 enrichment 是中等成本任务，最多 2 个批次并发。

## 9.7 建议的 GraphQL 查询

```graphql
query PullRequests(
  $owner: String!
  $name: String!
  $cursor: String
  $states: [PullRequestState!]
) {
  repository(owner: $owner, name: $name) {
    pullRequests(
      first: 100
      after: $cursor
      states: $states
      orderBy: { field: UPDATED_AT, direction: DESC }
    ) {
      nodes {
        id
        number
        title
        url
        state
        isDraft
        createdAt
        updatedAt
        closedAt
        mergedAt
        baseRefName
        headRefName
        headRefOid
        additions
        deletions
        changedFiles
        author { login }
      }
      pageInfo { hasNextPage endCursor }
    }
  }
  rateLimit { cost remaining resetAt }
}
```

通过 `execa('gh', ['api', 'graphql', '--input', '-'])` 将 JSON request body 写入 stdin，避免 Shell quoting 和超长参数。

## 9.8 并发预算

```text
同时同步的仓库：2
每个仓库 PR metadata：1 个分页流
每个仓库 Issue metadata：1 个分页流
文件 enrichment：最多 2 个批次
同一仓库 git fetch：1
```

先筛选需要更新的对象，再并发，不全量并发后过滤。

## 9.9 失败策略

- metadata 失败：保留旧数据，记录同步错误；
- enrichment 失败：PR 可显示但领域暂缺，下一次同步重试；
- rate limit 较低：停止 enrichment，优先完成 metadata；
- 不立即做多次自动重试；
- 页面显示最后成功同步时间和错误；
- 用户可手动重试。

---

# 10. 领域分类

## 10.1 规则格式

```ts
interface DomainRule {
  id: string
  repositoryId: string
  name: string
  include: string[]
  exclude: string[]
  enabled: boolean
}
```

所有路径先转为 `/` 分隔、去除前导 `./`。

匹配语义：

```text
命中任一 include
AND
未命中任一 exclude
```

一个 changed path 命中规则即可给 PR 添加领域。

## 10.2 计算

规则在保存时编译为 picomatch matcher。一次遍历 PR 文件集合并累积 `Set<domainId>`。

```text
classification_key = SHA256(rule_set_hash + file_set_hash)
```

如果 key 不变，不重复写入。

## 10.3 规则更新

规则 CRUD 成功后：

1. 更新规则集 hash；
2. 在本地后台逐批重算已有 `pull_request_files`；
3. 不调用 GitHub；
4. 页面允许短暂显示旧标签并显示“重新分类中”。

不建立复杂队列；使用 Server 进程内的串行后台任务即可。

---

# 11. PR Diff 和完整文件

## 11.1 数据来源

GitHub 只提供 PR 元数据、Base 分支名和 Head SHA。完整代码来自本地 Git。

首次打开 PR 或 Head 变化时，确保对象存在：

```bash
git -C <repo> cat-file -e <headSha>^{commit}
```

缺失则执行一次 fetch：

```bash
git -C <repo> fetch --no-tags <remote> \
  +refs/pull/<pr>/head:refs/loong/pull/<pr>/head \
  +refs/heads/<base>:refs/remotes/<remote>/<base>
```

同一仓库 fetch 串行。

## 11.2 Diff 基准

```bash
git merge-base refs/remotes/<remote>/<base> <headSha>
```

使用 merge base 与 head 读取完整文件并交给 Monaco Diff Editor。

Changed files：

```bash
git diff --name-status -z --find-renames <mergeBase> <headSha>
git diff --numstat -z <mergeBase> <headSha>
```

文件内容：

```bash
git show <mergeBase>:<oldPath>
git show <headSha>:<newPath>
```

Added 文件 original 为空；Deleted 文件 modified 为空；Rename 使用 old/new path。

## 11.3 Monaco 视图

统一使用同一个 Diff Editor，不维护两套 Diff 实现。

Changes 模式：

```ts
hideUnchangedRegions: {
  enabled: true,
  contextLineCount: 5,
  minimumLineCount: 8,
  revealLineCount: 20,
}
```

Full File 模式：

```ts
hideUnchangedRegions: { enabled: false }
```

默认统一 diff，后续若确有需求再加 side-by-side 开关。第一版不实现自定义 Patch 行渲染器。

## 11.4 大文件和二进制

只保留两个必要分支：

- Git 判断为 binary：显示文件状态和下载/本地打开提示，不送 Monaco；
- Monaco 拒绝或明显超过其 `maxFileSize`：显示“文件过大，请在本地查看”。

不要自行建立几十种文件限制和降级链。

## 11.5 本地命令按钮

按钮根据设置生成：

```bash
git fetch <remote> pull/<number>/head:pr-<number> && git switch pr-<number>
```

只复制，不由 LoongBoard 执行，不检查用户当前 VS Code 工作区。

---

# 12. Worktree Pool

## 12.1 目录

```text
system/.worktrees/vllm/slot-01
system/.worktrees/vllm/slot-02
system/.worktrees/vllm/slot-03
```

每个 slot 是该仓库的 detached worktree。

## 12.2 分配算法

为 `(repository, pr_number, head_sha)` 准备工作区：

1. 找到 target SHA 完全匹配且不 busy 的 slot，直接复用；
2. 找到尚未绑定的 slot；
3. 找到最久未使用、非 busy、Git clean 的 slot；
4. 如果没有可用 slot，返回明确错误，提示增加 slot 或清理已有 worktree。

只有两项保护：

- `busy_session_id != null` 的 slot 不回收；
- `git status --porcelain` 非空的 slot 不自动回收。

不实现租约续期、分布式锁、抢占队列或自动提交脏 worktree。

## 12.3 切换

可回收 slot：

```bash
git -C <slot> reset --hard <targetSha>
git -C <slot> clean -fd
```

`clean -fd` 不删除 ignored 文件；V1 不使用 `-x`。

若现有 worktree 初始化异常，允许删除并重新 `git worktree add --detach`。

## 12.4 Chat 与 Revision

默认 PR Chat key：

```text
repository_id + pr_number + head_sha
```

同一个 Head SHA 复用默认 Chat。PR 新 push 后 Head SHA 改变，创建新的默认 Chat；旧 Chat 仍可查看。

Worktree 是缓存。Chat Session 保存目标 SHA；重新打开旧 Chat 时重新寻找匹配 slot。

如果旧 SHA 已不存在于本地对象库且无法再从 GitHub ref 获取，历史 Chat 仍可读，但不能继续进行该旧版本代码分析。V1 不构建远端旧 commit 归档系统。

## 12.5 Revision 检查按钮

PR Chat 显示：

```text
Target: abc1234
Workspace: abc1234 ✓
```

或：

```text
Target: abc1234
Workspace: def5678 ⚠
```

“同步工作区”仅在 slot 非 busy 且 clean 时执行。否则直接提示当前状态，不增加复杂修复逻辑。

---

# 13. Agent Runtime 与 DSH Adapter

## 13.1 项目内契约

```ts
export interface AgentSessionSpec {
  sessionId: string
  workspacePath: string
  provider: string
  model: string
  reasoningEffort?: string
  maxTokens?: number
  dshHomePath: string
  runtimeSessionId?: string
}

export type AgentRuntimeEvent =
  | { type: 'status'; status: 'starting' | 'running' | 'idle' | 'stopped' }
  | { type: 'assistant.delta'; text: string }
  | { type: 'assistant.completed'; markdown: string }
  | { type: 'tool.started'; callId: string; name: string; summary?: string }
  | { type: 'tool.completed'; callId: string; name: string; summary?: string; isError: boolean }
  | { type: 'error'; message: string }

export interface AgentRuntime {
  run(spec: AgentSessionSpec, prompt: string): AsyncIterable<AgentRuntimeEvent>
  stop(sessionId: string): Promise<void>
  health(): Promise<AgentRuntimeHealth>
}
```

该契约只覆盖 V1 UI 需要的信息。不复制完整 DSH SessionEvent。

## 13.2 DSH Session 目录

每个 LoongBoard Agent Session：

```text
system/.loong/agent-sessions/<session-id>/dsh-home/
```

这样：

- Session 独立；
- 关闭子进程后可恢复；
- Cancel 不影响其他 Session；
- DSH 升级问题容易隔离；
- 不要求多个 DSH 进程共享同一 persistence 目录。

## 13.3 进程生命周期

1. 用户发送消息；
2. Server 获取该 Session 的 workspace；
3. 若 DSH 进程不存在，创建 `DeepSeekHarness`；
4. 使用 `profile: 'sdk'` 初始化；
5. 将 Prompt 发送给既有 `runtimeSessionId`，没有则新建；
6. `onNotification` 转换成 `AgentRuntimeEvent`；
7. 事件写数据库并通过 SSE 推送；
8. Agent idle 后结束本轮；
9. 进程空闲 20 分钟后关闭；
10. 下次重建进程并传入同一 DSH_HOME 和 runtimeSessionId。

## 13.4 Cancel

由于当前 SDK 没有轮次取消方法：

```text
Cancel → close/terminate 该 Chat Session 的 DSH 子进程
```

本轮消息标记为 `interrupted`。后续消息会启动新进程并恢复 Session。

## 13.5 事件存储

数据库保存 LoongBoard 归一化消息和工具摘要。DSH 自己保存完整 Session；LoongBoard 不解析其 JSONL 文件。

发生 Server 重启时，页面历史从 SQLite 读取；下一次对话依靠 DSH Session ID 恢复模型上下文。

## 13.6 无隐藏 Surface Context

创建 Session 时只设置：

- cwd；
- 模型；
- DSH profile；
- DSH_HOME。

不向模型自动发送 PR 编号、当前文件、选区、页面 URL或文档正文。用户 Prompt 和 Agent 自己的 Git/文件操作决定分析内容。

## 13.7 模型设置

产品默认建议：

```text
provider: deepseek-official
model: deepseek-v4-flash
```

允许在 Settings 和单个 Chat 创建时覆盖 provider、model、reasoning effort。第一版不实现任务级复杂路由策略。

---

# 14. Issue Chat

Issue 详情中心区域展示 Markdown 正文和评论，右侧是普通 Agent Chat。

Agent cwd 默认使用对应 repository 的主目录。系统确保默认分支可用，但不为 Issue 创建 worktree。

不自动注入 Issue 内容。页面提供 GitHub 链接和 Issue 编号，用户可以在 Prompt 中说明；后续若使用频率证明有价值，再增加显式“插入 Issue 链接”按钮。

---

# 15. Knowledge Repository 实现

## 15.1 文件系统为真

知识树通过扫描 `knowledge.path` 得到，排除：

```text
.git/
node_modules/
.loong/
```

Markdown 内容不复制到 SQLite 作为主数据。

## 15.2 稳定 Document ID

为支持移动文件后保留默认 Chat 映射，每篇文档使用标准 YAML Front Matter：

```markdown
---
loongboard_id: doc_01J...
---

# Document Title
```

- LoongBoard 创建文档时自动写入；
- 扫描到没有 ID 的既有 Markdown 时，在用户第一次通过 LoongBoard 保存该文档时补写；
- Agent 移动文件时 Front Matter 不变；
- 数据库用 `loongboard_id` 识别文档并更新 path。

不要建立私有 Markdown 标签或嵌入式对象语法。

## 15.3 编辑与预览

- 编辑：Monaco Editor；
- 预览：`react-markdown` + GFM；
- Mermaid：识别 `language-mermaid` code block，在独立组件中渲染；
- 图片：标准相对路径；
- HTML：默认不允许原始 HTML，避免额外复杂性；
- 保存：写临时文件后 rename 到目标路径，随后更新索引。

## 15.4 文档历史

每个文档保留最近 10 个完整内容快照。由于数量很小，存完整文本，不实现增量 diff 存储。

来源：

- `manual`：Web Save；
- `agent`：某次 Agent run 结束后的最终内容；
- `external`：VS Code 等外部编辑；
- `restore`：恢复旧版本。

### Agent 修改聚合

Agent 运行期间 chokidar 只收集 changed paths，不立即为每次写入创建版本。Agent 进入 idle 后：

1. 读取本轮改变的 Markdown；
2. 与该文档最后快照 hash 比较；
3. 有内容变化则创建一个 `agent` 版本；
4. 删除第 11 条及更老的版本。

这保证“一次 Agent 修改算一次”，不会因为 Agent 多次写文件产生大量历史。

外部编辑使用 1 秒 debounce 后创建 `external` 版本。

## 15.5 默认 Chat 映射

```text
knowledge_documents.default_session_id
```

打开文档时默认打开该 Session。用户可以新建或切换 Session。Agent 的 cwd 始终是知识库根目录，不限制到当前文档。

## 15.6 Git checkpoint

V1 可提供两个可选的确定性计划，不调用模型：

### autoCommit

```bash
git status --porcelain
# 非空时
git add -A
git commit -m "chore(knowledge): checkpoint <timestamp>"
```

### autoPush

```bash
git push <remote> <branch>
```

默认只为 Knowledge Repository 开启配置入口，源代码仓库默认关闭。失败只记录，不自动 merge、pull、rebase 或解决冲突。

---

# 16. Scheduler

## 16.1 调度模型

Server 启动后从 SQLite 读取 enabled tasks，计算 `next_run_at`，使用一个最小优先队列和单个 timer 调度。

任务到期：

1. 原子地插入 `scheduled_task_runs(status=running)`；
2. 若该 task 已有 running run，则本次标记 skipped；
3. 新建一次 Agent Session；
4. cwd 使用任务配置；
5. 原样发送 Prompt；
6. 等待 Agent idle；
7. 标记 completed/failed；
8. 计算下一次时间。

## 16.2 重启语义

V1 不补跑关机期间错过的任务。Server 启动后直接计算下一个未来时间。

理由：避免用户开机后突然并发执行大量旧报告。页面提供 Run Now。

## 16.3 工作区串行

同一个绝对 workspace path 同时只允许一个 Agent run。不同 workspace 可以并行。

这是唯一的文件写入并发控制；不建立分布式锁。

---

# 17. Backend API

所有 request/response schema 定义在 `packages/contracts`，Server 和 Web 共用。

## 17.1 Repositories

```text
GET    /api/repositories
POST   /api/repositories/:id/sync
GET    /api/repositories/:id/sync-status
```

## 17.2 PR

```text
GET /api/repositories/:id/pulls
    ?date=YYYY-MM-DD
    &status=draft|open|closed|merged
    &domain=<id>
    &cursor=<opaque>

GET /api/repositories/:id/pulls/activity-days?from=&to=
GET /api/repositories/:id/pulls/:number
GET /api/repositories/:id/pulls/:number/files
POST /api/repositories/:id/pulls/:number/prepare
GET /api/repositories/:id/pulls/:number/file?path=
GET /api/repositories/:id/pulls/:number/local-command
```

分页 cursor 使用 `(updated_at, number)`，页大小默认 50。

## 17.3 Domain rules

```text
GET    /api/repositories/:id/domains
POST   /api/repositories/:id/domains
PUT    /api/repositories/:id/domains/:domainId
DELETE /api/repositories/:id/domains/:domainId
```

## 17.4 Issue

```text
GET /api/repositories/:id/issues
GET /api/repositories/:id/issues/activity-days
GET /api/repositories/:id/issues/:number
```

## 17.5 Agent

```text
POST /api/agent-sessions
GET  /api/agent-sessions/:id
GET  /api/agent-sessions/:id/messages
POST /api/agent-sessions/:id/messages
GET  /api/agent-sessions/:id/events       # SSE
POST /api/agent-sessions/:id/cancel
```

POST message 返回 message id 后，流式内容通过 SSE。

## 17.6 Knowledge

```text
GET    /api/knowledge/tree
POST   /api/knowledge/documents
GET    /api/knowledge/documents/:id
PUT    /api/knowledge/documents/:id
POST   /api/knowledge/documents/:id/move
DELETE /api/knowledge/documents/:id
GET    /api/knowledge/documents/:id/versions
POST   /api/knowledge/documents/:id/versions/:versionId/restore
```

## 17.7 Scheduled tasks

```text
GET    /api/scheduled-tasks
POST   /api/scheduled-tasks
PUT    /api/scheduled-tasks/:id
DELETE /api/scheduled-tasks/:id
POST   /api/scheduled-tasks/:id/run
GET    /api/scheduled-tasks/:id/runs
```

---

# 18. Frontend 页面

```text
/repositories/:repo/pulls
/repositories/:repo/pulls/:number
/repositories/:repo/issues
/repositories/:repo/issues/:number
/knowledge/:documentId?
/scheduled-tasks
/settings/repositories
/settings/domains
/settings/agent
```

## 18.1 PR List

Toolbar：

- Repository；
- 最近活跃 / 日期；
- 状态；
- 领域；
- Sync；
- 最后成功同步时间。

行内容：

```text
Status | #number Title | Domain chips | files +add -del | updated time
```

不显示 AI 文本。

## 18.2 PR Detail

- 左栏 changed files；
- 中间 Monaco Diff；
- 顶部 Changes / Full File；
- 右栏 Chat，可折叠；
- Workspace revision 状态；
- 复制本地命令。

## 18.3 Knowledge

- 左栏文件树；
- 中间 Preview/Edit；
- 右栏默认 Chat；
- 保存、移动、历史、恢复；
- 不做富文本编辑器。

---

# 19. 并发、缓存和性能

## 19.1 明确预算

```text
Repository metadata sync: max 2 repositories
GitHub file enrichment: max 2 batches
Git fetch: 1 per repository
Agent run: 1 per workspace path
Active DSH processes: no fixed hard cap; idle timeout handles回收
SQLite writer: single Node process
```

## 19.2 列表性能

- 列表只读 SQLite；
- 不在 HTTP 请求中调用 GitHub；
- DB 索引：

```text
pull_requests(repository_id, updated_at DESC, number DESC)
pull_requests(repository_id, status, updated_at DESC)
pull_request_domains(repository_id, domain_rule_id, pr_number)
issues(repository_id, updated_at DESC, number DESC)
```

- 日期筛选使用预计算/SQL 时区表达式；如果 SQLite 时区支持不足，在写入时额外存 `activity_date_local`，由 system timezone 变化时重建。负责人基于实际 SQLite 方案选择其一，不同时维护两套。

## 19.3 内容缓存

第一版不建立服务端 Git Blob LRU。`git show` 按需读取，浏览器保留当前 Monaco models。只有实际 profiling 表明重复读取明显时再加缓存。

## 19.4 DSH 进程缓存

- 用户正在使用的 Chat 复用其进程；
- idle 20 分钟关闭；
- Session 持久数据保留；
- 不预启动 Agent；
- 不为列表页面启动 Agent。

---

# 20. 错误处理与“不过度防御”规则

## 20.1 必须做的防护

只保留会导致数据破坏或边界不清的必要检查：

- 配置路径存在且类型正确；
- Git repo 验证；
- 外部 JSON schema 验证；
- Worktree busy/dirty 不回收；
- 文件路径必须在知识库根目录内；
- 文档保存使用原子替换；
- 同一 workspace 不并发运行 Agent；
- 数据库唯一键和外键。

## 20.2 禁止的模式

- 每个内部函数重复检查同一字段；
- `try/catch` 后忽略错误；
- “如果 A 失败再试 B，再试 C，再返回空数据”的长降级链；
- 同时保留新旧两套实现；
- 为不存在的未来 provider 写兼容层；
- 任何地方都返回 `{ success, data, error }`；
- 为普通本地函数建立十层接口；
- 无需求的 feature flags；
- 过多可选参数和默认值；
- 日志代替错误传播；
- 测试只为了覆盖率编写无行为价值的断言。

## 20.3 推荐错误风格

```text
GitHubCommandError
  command: gh api graphql
  repository: vllm-project/vllm
  exitCode: 1
  stderr: ...
```

只有调用方需要做不同处理时才创建错误子类。其他错误使用带上下文的普通 `Error`。

---

# 21. 日志与可观测性

本地结构化日志至少包含：

```text
requestId
repositoryId
syncRunId
agentSessionId
scheduledTaskRunId
command
elapsedMs
```

记录：

- GitHub 每页耗时、节点数、rate limit；
- enrichment 数量；
- Git fetch 耗时；
- DSH 启动、首 token、完成时间；
- Scheduler 状态；
- Knowledge 文件变化与版本创建。

日志写入：

```text
system/.loong/logs/server.ndjson
```

第一版不引入 OpenTelemetry Collector。

---

# 22. 测试策略

## 22.1 Unit Tests

必须覆盖纯逻辑：

- PR status 派生；
- UTC 到日历日期；
- watermark stop 条件；
- Domain include/exclude；
- classification key；
- Worktree slot 选择；
- 文档 Front Matter ID；
- 版本保留 10 条；
- Scheduler next run；
- DSH notification → AgentRuntimeEvent 映射。

纯逻辑包目标：分支覆盖率 85% 以上。不要给 UI 和适配器设置机械的全局覆盖率门槛。

## 22.2 Property Tests

使用 fast-check 验证：

- Domain 匹配顺序不影响结果；
- cursor encode/decode 可逆；
- 版本淘汰永远不超过 limit；
- path normalize 后不产生反斜杠；
- status 派生只返回四种合法值。

## 22.3 Integration Tests

### GitHub Provider

- 使用 fake `gh` executable 输出 fixture；
- 验证分页、watermark、字段映射、命令数量；
- 一个测试明确断言 100 个 PR 不产生 100 次 `gh pr view`。

### Git Workspace

使用临时 Git repos 覆盖：

- added/modified/deleted/renamed；
- merge base；
- worktree reuse；
- busy/dirty slot 不回收；
- fetch ref command 构造。

### Knowledge

- 扫描 Markdown；
- move 保持 document ID；
- manual/agent/external 历史；
- Agent 多次写聚合为一个版本；
- Mermaid block 保留。

### Scheduler

- 到期运行；
- 同任务重复触发 skipped；
- workspace mutex；
- restart 不补跑。

## 22.4 DSH Compatibility Tests

分三层：

1. `dsh --profile sdk --help` 可以启动并退出；
2. 使用 fake stdio JSON-RPC peer 测试我们的 adapter 状态机和事件归一化；
3. 可选 nightly/live smoke：使用低成本模型发送固定 Prompt，检查能收到 user/assistant/status 事件并恢复同一 Session。

不要读取或断言 DSH 内部 JSONL 格式。

## 22.5 E2E

Playwright 至少覆盖：

### E2E-PR-LIST

- 加载 fixture GitHub 数据；
- 最近活跃排序；
- 日期、状态、领域筛选；
- 代码量显示。

### E2E-PR-DIFF

- 打开 PR；
- 选择文件；
- Changes 折叠未修改区；
- Full File 展示完整内容和 Diff；
- 复制本地命令。

### E2E-AGENT

- 创建 Chat；
- fake DSH 流式返回；
- 页面显示 delta、tool、完成；
- 刷新后历史存在；
- Cancel 终止对应 runtime。

### E2E-KNOWLEDGE

- 新建文档；
- Edit/Preview；
- 保存；
- 移动；
- 默认 Chat 保持；
- 查看和恢复版本。

### E2E-SCHEDULER

- 新建任务；
- Run Now；
- fake Agent 修改 inbox 文档；
- run completed；
- 文档历史产生 agent 版本。

---

# 23. CI 和统一命令

根 scripts：

```json
{
  "scripts": {
    "dev": "...",
    "build": "...",
    "lint": "...",
    "typecheck": "...",
    "test": "vitest run",
    "test:integration": "...",
    "test:e2e": "playwright test",
    "check:architecture": "...",
    "check:dsh-boundary": "...",
    "check": "pnpm lint && pnpm typecheck && pnpm check:architecture && pnpm test && pnpm test:integration && pnpm build",
    "check:full": "pnpm check && pnpm test:e2e"
  }
}
```

架构检查必须验证：

```text
@deepseek-ai/* imports outside packages/agent-runtime-dsh = 0
circular package dependencies = 0
raw SQL outside packages/database = 0
gh execution outside packages/github = 0
git execution outside packages/git-workspace and knowledge git service = 0
```

错误信息必须说明违规文件、规则和修复方向。

---

# 24. 面向维护 Agent 的文档体系

初始代码合入前必须有：

- 根 `AGENTS.md`；
- 每个 app/package 的简短 `AGENTS.md`；
- `docs/requirements.md`：本文产品部分；
- `docs/architecture.md`：组件和依赖方向；
- `docs/data-model.md`；
- `docs/github-sync.md`；
- `docs/dsh-integration.md`；
- `docs/testing.md`；
- `docs/operations.md`；
- `docs/implementation-status.md`；
- ADR。

首批 ADR：

```text
0001-local-first-single-user.md
0002-use-gh-graphql-for-metadata.md
0003-use-local-git-for-pr-content.md
0004-dsh-is-an-external-runtime.md
0005-markdown-and-git-are-knowledge-source-of-truth.md
0006-no-product-level-agent-permission-system.md
0007-avoid-excessive-defensive-programming.md
```

每个 package README 使用固定结构：

```text
Purpose
Owns
Does not own
Public API
Dependencies
Invariants
Tests
```

---

# 25. 多 Agent 团队分工

以下分工基于用户提供的可用模型，不代表对模型的通用排名。

## 25.1 Codex 协同控制器

负责：

- 启动和管理子 Agent；
- 为每个任务创建隔离 worktree；
- 收集状态；
- 将结果交给 Team Lead；
- 不自行改变已冻结需求。

## 25.2 Team Lead：Luna Max

职责：

- 维护总体计划和 `docs/implementation-status.md`；
- 将阶段拆成可验收任务；
- 冻结共享 contract；
- 决定并行边界；
- 审查子 Agent 提交；
- 运行集成检查；
- 做最终功能验收；
- 只有在阻塞或跨模块问题时亲自编码。

## 25.3 Deputy / Integration Lead：Luna High

职责：

- 负责 Web 与 Server 接口联调；
- 处理共享类型、路由、数据库迁移冲突；
- 在 Lead 忙于规划和验收时合并低风险任务；
- 维护 E2E fixture 和集成环境。

## 25.4 常规实现 Agent：DeepSeek V4 Flash

作为默认劳动力，负责边界清楚的任务：

- CRUD；
- UI 页面和组件；
- 数据映射；
- Unit/Integration Tests；
- 文档更新；
- 简单 bug 修复；
- fixture 建设。

每个任务必须给出明确 files scope、输入/输出 contract 和验收命令。

## 25.5 Senior Engineer / Reviewer：Sol High

负责：

- GitHub incremental sync；
- Git worktree manager；
- Monaco Diff 数据流；
- Scheduler 并发；
- DSH process lifecycle；
- 重要代码 review；
- 性能和命令次数审查。

## 25.6 Technical Expert：Sol Very High

仅在以下情况调用：

- DSH SDK 协议或升级发生 breaking change；
- Session 恢复/Cancel 语义无法可靠实现；
- Git merge-base/worktree 数据正确性问题；
- GitHub GraphQL 查询成本或分页出现结构性问题；
- 复杂并发死锁或文件一致性问题；
- Lead 与 Senior 对架构结论不一致。

不要让最高成本专家承担普通 CRUD 和样式工作。

## 25.7 QA Agent：DeepSeek V4 Flash + Sol High 复核

- Flash 编写 fixture、测试和复现；
- Sol High 对关键路径做最终测试审查；
- 最终发布前由一个未参与实现的 Agent 执行黑盒验收。

---

# 26. 多 Agent 协作规则

## 26.1 工作隔离

- 一个任务一个 Git worktree；
- 一个 package 同一时刻最多一个实现 Agent；
- 共享 schema/migration 只能由指定 owner 修改；
- UI 和 Server 可以在 API contract 冻结后并行；
- 最大同时进行 3 个实现任务，避免负责人 review 积压。

## 26.2 每个任务 Brief

负责人必须提供：

```markdown
# Task

## Goal

## Non-goals

## Files allowed

## Existing contracts

## Required behavior

## Required tests

## Acceptance commands

## Dependencies / blocked by
```

禁止只发“把 GitHub 同步做完”之类的模糊任务。

## 26.3 子 Agent 完成标准

子 Agent 交付时必须报告：

- 修改了什么；
- 为什么放在这些模块；
- 运行了哪些测试；
- 有哪些明确未完成项；
- commit SHA。

不得报告隐藏推理过程，只报告可验证事实。

## 26.4 Review 流程

```text
Implementer self-check
    ↓
Peer/Senior review
    ↓
Lead integration
    ↓
Stage acceptance
```

负责人不得在大量测试失败时继续合并后续功能。

---

# 27. 实施阶段

## Stage 0：Foundation

负责人：Luna Max
协作：Luna High、Sol High

交付：

- 新仓库；
- pnpm workspace；
- apps/packages 目录；
- root/scoped AGENTS；
- system config schema；
- SQLite migration runner；
- contracts package；
- Fastify health API；
- React shell；
- `pnpm check`；
- DSH exact pin 和 lock 文件；
- 架构边界检查。

验收：

```text
pnpm install
pnpm check
pnpm dev
GET /api/health = 200
```

## Stage 1：GitHub Metadata Vertical Slice

负责人：Sol High
实现：DeepSeek V4 Flash

交付：

- Repository config；
- `GhGitHubMetadataProvider`；
- bootstrap/incremental sync；
- SQLite PR/Issue；
- PR/Issue list API；
- PR 页面基本列表；
- Issue 页面基本列表；
- 日期和状态筛选；
- 命令次数 integration tests。

验收：

- 两个真实仓库可同步；
- 页面打开不调用 GitHub；
- 第二次同步只读取 watermark 后的页面；
- GitHub 失败时旧列表可读。

## Stage 2：Domain Classification

负责人：Luna High
实现：DeepSeek V4 Flash

交付：

- changed file enrichment；
- Domain CRUD；
- picomatch 分类；
- 多标签显示/筛选；
- 规则变更本地重算；
- unit/property/integration tests。

验收：

- 修改 `.github/**` 的 PR 命中 CI；
- 一个 PR 可命中多个领域；
- 规则修改不调用 GitHub；
- head_sha 未变化不重新抓 files。

## Stage 3：PR Diff Workspace

负责人：Sol High
UI：Luna High
实现协作：DeepSeek V4 Flash

交付：

- Git object preparation；
- merge base；
- changed files tree；
- Base/Head 完整内容 API；
- Monaco Changes / Full File；
- rename/add/delete；
- copy command；
- Git fixture tests；
- E2E。

验收：

- Changes 模式折叠未修改区；
- Full File 显示完整源码；
- 两种模式显示相同差异；
- 不使用 GitHub patch；
- PR 详情首开只进行必要的一次 Git fetch。

## Stage 4：DSH Agent Chat

负责人：Sol High
技术专家按需：Sol Very High
UI：Luna High

交付：

- `AgentRuntime`；
- DSH adapter；
- per-session DSH_HOME；
- DSH process supervisor；
- SQLite normalized messages；
- SSE；
- Chat UI；
- Cancel；
- idle shutdown；
- Worktree pool；
- revision check；
- DSH compatibility tests。

验收：

- PR Chat cwd 是目标 worktree；
- 不存在隐藏页面 context；
- 刷新浏览器历史保留；
- 进程关闭后可以继续同一 Session；
- Cancel 不影响其他 Chat；
- DSH 类型没有泄漏出 adapter package。

## Stage 5：Knowledge Repository

负责人：Luna High
实现：DeepSeek V4 Flash
UI 难点 review：Sol High

交付：

- Knowledge tree；
- Front Matter document ID；
- Markdown Preview/Edit；
- GFM/Mermaid/images；
- move/rename/delete；
- default Chat mapping；
- history 10 条；
- Agent run version aggregation；
- Git checkpoint/push 可选设置；
- tests/E2E。

验收：

- 文档移动后默认 Chat 不变；
- Agent 一次运行多次写文件只生成一个版本；
- VS Code 外部编辑被索引；
- Git repo 内容在 LoongBoard 之外仍是普通 Markdown。

## Stage 6：Scheduler

负责人：Sol High
实现：DeepSeek V4 Flash

交付：

- Scheduled task CRUD；
- Cron/timezone；
- Run Now；
- 每次新 Agent Session；
- workspace mutex；
- run history；
- 失败状态；
- tests/E2E。

验收：

- Prompt 原样发送；
- Scheduler 不解析报告格式；
- 同一 workspace 不同时写；
- 重启不补跑；
- 生成/更新的 Markdown 能进入历史。

## Stage 7：Release Acceptance

负责人：Luna Max
复核：Sol High
独立黑盒 Agent：DeepSeek V4 Flash
专家按需：Sol Very High

交付：

- 全部 docs；
- `pnpm check:full`；
- 真实 vLLM/vLLM-Ascend smoke；
- DSH live smoke；
- 性能和命令次数报告；
- 已知限制；
- 本地安装运行说明。

---

# 28. 功能验收标准

## Repository

- PR/Issue 列表数据库读取时间稳定；
- 日期筛选语义明确为 `updatedAt` 所属日期；
- 四种 PR 状态互斥；
- 无 AI 列表分析；
- GitHub API 请求与页面浏览解耦。

## Domain

- 用户可以 CRUD 规则；
- 规则完全由 changed files 决定；
- 多标签；
- 无主标签；
- 无 AI。

## Diff

- 本地 Git 完整文件；
- Changes 和 Full File 无缝切换；
- changed files tree；
- 不依赖 GitHub patch 截断结果。

## Agent

- DSH 作为子进程；
- per-session isolation；
- Chat 历史本地可恢复；
- Agent 有完整本地能力；
- 无 Surface Context 自动注入；
- DSH 可升级边界明确。

## Knowledge

- 文件即数据；
- Git 可独立使用；
- 标准 Markdown；
- 文档移动不丢 Chat；
- 版本最多 10；
- Agent 可操作整个仓库。

## Scheduler

- 保存 Prompt；
- 到时发送；
- Agent 决定文件操作；
- 有运行历史；
- 无 Workflow Builder。

---

# 29. Definition of Done

一个任务只有满足以下条件才完成：

- 功能符合本文；
- 没有增加 V1 非目标；
- 代码放在正确边界；
- 外部输入只校验一次；
- 没有静默 catch；
- 有必要的 unit/integration tests；
- API contract 更新；
- package README/相关文档更新；
- `pnpm check` 通过；
- 对用户可见功能有 E2E 或明确的阶段性 E2E；
- commit 独立、描述清楚；
- Lead 已验收。

---

# 30. DSH 升级流程

1. 检查最新 Release 和 master changelog；
2. 创建独立 upgrade worktree；
3. 更新 `dsh.lock.json` 和两个 exact dependency；
4. 安装；
5. 运行 SDK help smoke；
6. 编译 `agent-runtime-dsh`；
7. 运行 recorded notification normalization tests；
8. 运行 live smoke；
9. 检查只有以下目录需要改动：

```text
packages/agent-runtime-dsh
tests/dsh-compat
dsh.lock.json
```

如果升级要求修改 Web、Knowledge、GitHub 或数据库领域代码，Lead 必须先判定是不是 DSH 类型泄漏造成的架构回归。

10. 由 Sol High review；重大协议变化升级给 Sol Very High；
11. `pnpm check:full`；
12. 合并。

当前 DSH 处于预发布，升级不做自动合并。

---

# 31. 已知限制

V1 接受以下限制：

- GitHub `updatedAt` 不能说明最后活动具体类型；
- PR files REST 最多 3000 个；
- Head 未变化时不因 Base 更新重算文件分类；
- DSH SDK Cancel 需要杀进程；
- DSH SDK 无协议版本协商；
- 历史 PR Head 对象若本地丢失，旧 Chat 不能继续代码分析；
- Issue Chat 不自动知道当前 Issue；
- 同一 workspace 的 Agent 写操作串行；
- Scheduler 关机期间不补跑；
- Git autoPush 不处理远端冲突；
- Markdown 不支持任意原始 HTML；
- 单用户本地运行，不做认证。

这些是有意控制范围，不应由子 Agent自行“补全”。

---

# 32. 负责人最终执行检查

在开始编码前，Luna Max 必须输出并提交：

```text
1. docs/implementation-status.md
2. Stage 0 任务列表
3. package ownership 表
4. API contract owner
5. database migration owner
6. 并行 Agent 数量
7. 第一轮 task briefs
```

每个 Stage 完成后更新：

```text
Done
Validated
Known limitations
Next stage blockers
```

负责人不得为了并行率让多个 Agent 同时修改相同 shared package，也不得把尚未验收的中间实现当作下一阶段稳定依赖。

最终交付应是一个可以在 `system/loong-dashboard` 中直接运行、可以读取兄弟仓库、可以调用固定版本 DSH、可以由后续 Agent 继续维护的清晰本地项目。
