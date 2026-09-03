# LoongBoard System Home

Control-plane documents and templates of the LoongBoard V1 system.

Layout on the maintenance machine (`~/system`) — itself **not** a git repo;
every subdirectory manages its own repository:

| Directory | GitHub repository | Purpose |
|---|---|---|
| `loong-dashboard/` | MrZ20-System/loong-dashboard | LoongBoard application |
| `knowledge/` | MrZ20-System/knowledge | Knowledge repository (Markdown, managed by LoongBoard) |
| `vllm/`, `vllm-ascend/` | *(third-party)* | Upstream mirrors fetched at deploy time; not stored here |
| this folder | MrZ20-System/system-home | AGENTS, plans, sanitized config template |

Machine-local state (`.loong`, `.worktrees`, `.pnpm-store`, `.workbuddy`) and the
real `system.yaml` (absolute local paths) stay untracked; commit
`system.example.yaml` instead and copy it to `system.yaml` on a new machine.
