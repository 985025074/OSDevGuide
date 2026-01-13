# fix single hart's dead problem

- Commit: `6d5fde0e004aac9fdb739feefde254db04b6cb3f`
- Date: `2025-12-22`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`, `fs`, `task`

## 文件变更
```text
 src/fs/inode.rs           |  4 ++++
 src/fs/mod.rs             |  1 +
 src/main.rs               |  9 +++++++--
 src/syscall/filesystem.rs | 25 ++++++++++++++++++++++---
 src/task/manager.rs       | 44 +++++++++++++++++++++++++++++++++++++++++---
 5 files changed, 75 insertions(+), 8 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`src/fs/inode.rs`, `src/fs/mod.rs`, `src/main.rs`, `src/syscall/filesystem.rs`, `src/task/manager.rs`
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
