# add socket. add Copy on write to avoid too much space usage to pass the Cyclictest.

- Commit: `4cecd88c52302e3ae79b15723a0ac6c441d25751`
- Date: `2025-12-25`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`, `fs`, `mm`, `task`, `trap`

## 文件变更
```text
 src/fs/mod.rs             |   5 +-
 src/fs/pipe.rs            |  16 +++
 src/fs/pseudo.rs          | 122 ++++++++++++++++++++
 src/fs/socketpair.rs      |  62 +++++++++++
 src/mm/frame_allocator.rs |  32 +++++-
 src/mm/memory_set.rs      | 276 +++++++++++++++++++++++++++++++++++++++++++---
 src/mm/page_table.rs      |  36 +++++-
 src/syscall/filesystem.rs | 161 +++++++++++++++++++++++++--
 src/syscall/memory.rs     |  69 +++++++++++-
 src/syscall/misc.rs       |  95 ++++++++++------
 src/syscall/mod.rs        |  15 +++
 src/syscall/process.rs    |  57 +++++++++-
 src/syscall/sched.rs      |   4 +-
 src/syscall/socket.rs     |  56 ++++++++++
 src/syscall/time_sys.rs   |  39 ++++++-
 src/task/process_block.rs | 104 +++++++++++++++--
 src/task/processor.rs     |  27 +++++
 src/trap/mod.rs           |  19 ++++
 18 files changed, 1117 insertions(+), 78 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`src/fs/mod.rs`, `src/fs/pipe.rs`, `src/fs/pseudo.rs`, `src/fs/socketpair.rs`, `src/mm/frame_allocator.rs`, `src/mm/memory_set.rs`, `src/mm/page_table.rs`, `src/syscall/filesystem.rs`, `src/syscall/memory.rs`, `src/syscall/misc.rs` ...
- 若涉及 syscall 新增/改动：优先查看 `src/syscall/mod.rs` 的 syscall 号与分发分支。
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
