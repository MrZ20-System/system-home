# LoongBoard System Home

此目录是独立的 system-home Git 仓库，保存工作区约束、个人部署布局说明和安全边界。它不是 LoongBoard 的架构真相；应用实现、配置 schema 和运行行为统一以 [LoongBoard 文档](https://github.com/MrZ20-System/loong-dashboard/blob/main/docs/README.md) 及 [canonical system.example.yaml](https://github.com/MrZ20-System/loong-dashboard/blob/main/system.example.yaml) 为准。这里不再维护 V1、MultiAgent 方案正文或重复配置模板。

| 目录 | Git 仓库角色 | 职责 |
| --- | --- | --- |
| loong-dashboard/ | 独立应用仓库 | LoongBoard 源码、构建、架构和正式文档 |
| 本目录（system-home） | 独立部署/布局仓库 | 工作区规则、个人配置布局和脱敏模板 |
| knowledge/ | 独立 Markdown 知识仓库 | Markdown source of truth、inbox 和 Knowledge Git 历史 |
| agent-history/（可选） | 独立私有归档仓库 | Agent Archive 导出的会话归档；按需单独保护 |
| vllm/、vllm-ascend/ | 独立第三方源码仓库 | 应用配置引用的本地源码 |
| system.yaml、settings.json | 本机部署数据 | 实例配置、账户验证状态和调度设置；不提交到 system-home |
| domains/、prompts/ | 本机部署数据 | Domain 定义与更新提示；随 data root 单独备份，不提交到 system-home |
| .worktrees/ | runtime/cache | 可复用 Git worktree 缓存，不是知识源 |
| .loong/ | runtime/cache | 本地数据库、会话和运行状态，不是知识源 |

本机 `system.yaml` 不提交；使用应用仓库中的 [canonical system.example.yaml](https://github.com/MrZ20-System/loong-dashboard/blob/main/system.example.yaml) 配置新机器。相对路径以 YAML 所在目录解析，详细字段和运行方式见 [应用运行说明](https://github.com/MrZ20-System/loong-dashboard/blob/main/docs/operations.md) 与 [部署说明](https://github.com/MrZ20-System/loong-dashboard/blob/main/docs/deployment.md)。

知识文件的正文事实来源是独立的 [Knowledge 仓库](https://github.com/MrZ20-System/knowledge)，不是 SQLite、`.loong/` 或 `.worktrees/`。Knowledge 的人工编辑、watcher/index projection、checkpoint、push 和备份边界见 [Knowledge 文档](https://github.com/MrZ20-System/loong-dashboard/blob/main/docs/knowledge.md)。

工作区规则见 [AGENTS.md](AGENTS.md)，应用文档入口见 [LoongBoard 文档](https://github.com/MrZ20-System/loong-dashboard/blob/main/docs/README.md)。
