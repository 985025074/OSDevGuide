# netperf_test pass

- Commit: `177c2f708ac24618c0e81d077d01c83d96a274cb`
- Date: `2026-01-10`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`, `fs`, `task`

## 文件变更
```text
 src/debug_config.rs     |  8 ++---
 src/fs/net_socket.rs    | 35 ++++++++++++++++--
 src/syscall/mod.rs      | 21 ++++++++++-
 src/syscall/net.rs      | 36 +++++++++++++++++--
 src/syscall/time_sys.rs | 78 ++++++++++++++++++++++++++++++++++++++++
 src/task/block_sleep.rs | 96 +++++++++++++++++++++++++++++++++++++++++++++++++
 6 files changed, 263 insertions(+), 11 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`src/debug_config.rs`, `src/fs/net_socket.rs`, `src/syscall/mod.rs`, `src/syscall/net.rs`, `src/syscall/time_sys.rs`, `src/task/block_sleep.rs`
- 若涉及 syscall 新增/改动：优先查看 `src/syscall/mod.rs` 的 syscall 号与分发分支。
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
