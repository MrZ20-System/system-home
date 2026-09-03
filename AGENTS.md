# LoongBoard System Workspace Instructions

## Directory roles

- `loong-dashboard/` is the LoongBoard application repository.
- `knowledge/` is the durable Markdown knowledge repository.
- `.worktrees/` contains disposable Git worktree caches.
- `.loong/` contains local runtime state and may be rebuilt when explicitly safe.
- Other repository directories are independent Git repositories when configured in `system.yaml`.

## Working rules

- When analyzing a PR, prefer the current assigned worktree and its checked-out revision.
- Automated reports go to `knowledge/inbox/` unless the user specifies another durable path.
- Do not delete long-lived knowledge without an explicit request.
- Treat Markdown files and Git history as knowledge source data; SQLite is only an index and runtime store.
- `.worktrees/` and `.loong/` are runtime/cache locations, not knowledge sources.
- Code and knowledge changes may be committed to their owning Git repository when the task requires it.
- Do not copy architecture or implementation from the rejected legacy dashboard.
- DeepSeek Harness remains an external runtime; do not fork it or add a LoongBoard DSH plugin.

