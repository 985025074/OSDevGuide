# implement the pseudo system

- Commit: `09c23d8552d7ed6ae25b77202eac093e75517257`
- Date: `2025-12-21`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`, `fs`

## 文件变更
```text
 src/fs/mod.rs             |   2 +-
 src/fs/pseudo.rs          |  95 ++++++++
 src/syscall/filesystem.rs | 561 ++++++++++++++++++++++++++++++++++++++++++++--
 src/syscall/misc.rs       |  66 +++++-
 src/syscall/mod.rs        |  20 +-
 5 files changed, 720 insertions(+), 24 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`src/fs/mod.rs`, `src/fs/pseudo.rs`, `src/syscall/filesystem.rs`, `src/syscall/misc.rs`, `src/syscall/mod.rs`
- 若涉及 syscall 新增/改动：优先查看 `src/syscall/mod.rs` 的 syscall 号与分发分支。
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
