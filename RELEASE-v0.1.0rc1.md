# LoongBoard system-home v0.1.0rc1

这是 system-home 的首个候选发布说明。本次发布只保留最小部署骨架、工作区约束、目录职责和安全边界；它不承载 LoongBoard 应用源码或运行数据库。

发布边界：

- 跟踪内容仅包括工作区说明、忽略规则和非秘密部署文档。
- `system.yaml`、`system.yaml*.bak`、`settings.json`、`domains/`、`prompts/`、`.loong/`、`.worktrees/` 及嵌套源码仓库均属于机器本地数据，不进入 Git 版本或镜像发布。
- GitHub token、Agent API key、镜像仓库凭据和其他 secrets 只通过目标部署环境的安全配置注入，不写入本仓库、Release Note 或镜像层。

应用功能、版本变更和应用部署步骤以 [loong-dashboard 文档](https://github.com/MrZ20-System/loong-dashboard/blob/main/docs/README.md) 及其应用级 Release Note 为准。本文件只记录 system-home 的发布边界，避免与应用仓库重复。
