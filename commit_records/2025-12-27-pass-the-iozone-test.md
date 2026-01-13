# pass the iozone_test

- Commit: `0220c1e110eacc74236ce4bb9cdf8974b34e4574`
- Date: `2025-12-27`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`, `fs`, `mm`, `task`, `trap`, `drivers`, `build`

## 文件变更
```text
 Cargo.lock                      |   1 -
 Cargo.toml                      |   2 +-
 Makefile                        |  13 +-
 src/drivers/block/virtio_blk.rs |  63 +++++++--
 src/fs/inode.rs                 | 157 ++++++++++++++++++---
 src/mm/memory_set.rs            |  96 ++++++++++++-
 src/mm/mod.rs                   |   4 +-
 src/mm/page_table.rs            |  97 ++++++++++++-
 src/syscall/filesystem.rs       | 201 ++++++++++++++++++++++++---
 src/syscall/flow.rs             |   4 +-
 src/syscall/futex.rs            |   4 +-
 src/syscall/memory.rs           | 119 +++++++++++++---
 src/syscall/misc.rs             |  14 +-
 src/syscall/mod.rs              |  23 +++-
 src/syscall/process.rs          |   8 +-
 src/syscall/sched.rs            |  12 +-
 src/syscall/socket.rs           |  13 +-
 src/syscall/sysv_shm.rs         | 279 +++++++++++++++++++++++++++++++++++++
 src/syscall/time_sys.rs         |  37 ++++-
 src/task/process_block.rs       | 299 +++++++++++++++++++++++++++++++++++++---
 src/task/processor.rs           |   6 +-
 src/task/signal.rs              |   6 +-
 src/trap/mod.rs                 |  90 ++++++++++++
 23 files changed, 1415 insertions(+), 133 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`Cargo.lock`, `Cargo.toml`, `Makefile`, `src/drivers/block/virtio_blk.rs`, `src/fs/inode.rs`, `src/mm/memory_set.rs`, `src/mm/mod.rs`, `src/mm/page_table.rs`, `src/syscall/filesystem.rs`, `src/syscall/flow.rs` ...
- 若涉及 syscall 新增/改动：优先查看 `src/syscall/mod.rs` 的 syscall 号与分发分支。
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
