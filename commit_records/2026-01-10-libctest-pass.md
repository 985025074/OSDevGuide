# libctest pass

- Commit: `b5cee3fde17a232229f15bf8260c623794db2cc9`
- Date: `2026-01-10`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`, `fs`, `mm`, `task`, `trap`

## 文件变更
```text
 src/config.rs              |   6 +-
 src/debug_config.rs        |   2 +-
 src/fs/net_socket.rs       |  67 ++++-
 src/mm/memory_set.rs       |  30 ++-
 src/mm/mod.rs              |   1 +
 src/mm/page_table.rs       | 119 +++++++-
 src/syscall/filesystem.rs  | 391 ++++++++++++++++++++++++---
 src/syscall/futex.rs       | 151 ++++++++++-
 src/syscall/misc.rs        |  83 +++++-
 src/syscall/mod.rs         |  13 +-
 src/syscall/net.rs         |  34 ++-
 src/syscall/process.rs     |  16 +-
 src/syscall/robust_list.rs |  63 +++++
 src/syscall/signal.rs      | 658 +++++++++++++++++++++++++++++++++++++++++++--
 src/syscall/socket.rs      |  25 +-
 src/syscall/thread.rs      |  17 ++
 src/task/block_sleep.rs    |  55 +++-
 src/task/id.rs             |  14 +-
 src/task/manager.rs        |   4 +
 src/task/process_block.rs  |  41 ++-
 src/task/processor.rs      |  62 +++--
 src/task/signal.rs         |  12 +
 src/task/task_block.rs     |  28 ++
 src/trap/mod.rs            |  68 ++++-
 src/trap/trap.asm          |   6 +
 25 files changed, 1830 insertions(+), 136 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`src/config.rs`, `src/debug_config.rs`, `src/fs/net_socket.rs`, `src/mm/memory_set.rs`, `src/mm/mod.rs`, `src/mm/page_table.rs`, `src/syscall/filesystem.rs`, `src/syscall/futex.rs`, `src/syscall/misc.rs`, `src/syscall/mod.rs` ...
- 若涉及 syscall 新增/改动：优先查看 `src/syscall/mod.rs` 的 syscall 号与分发分支。
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
