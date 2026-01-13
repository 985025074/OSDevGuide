# make the memory mange more robust;accoring to the libbenchtest

- Commit: `92c63f6e163bc741c51a5696ea8647a2eca609c8`
- Date: `2026-01-07`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`, `fs`, `mm`, `task`, `trap`, `build`

## 文件变更
```text
 Cargo.lock                |   7 ++
 Cargo.toml                |   1 +
 src/config.rs             |  24 +++-
 src/debug_config.rs       |   6 +
 src/fs/inode.rs           |   4 +
 src/main.rs               |   1 +
 src/mm/dtb.rs             |  38 +++++++
 src/mm/frame_allocator.rs |   6 +-
 src/mm/memory_set.rs      | 282 ++++++++++++++++++++++++++++++++++++++++++----
 src/mm/mod.rs             |   2 +
 src/mm/page_table.rs      |  75 ++++++++++--
 src/syscall/filesystem.rs |  63 ++++++++++-
 src/syscall/futex.rs      |  45 +++++++-
 src/syscall/memory.rs     | 111 ++++++++++--------
 src/syscall/misc.rs       |  45 +++++++-
 src/syscall/mod.rs        |   4 +
 src/syscall/process.rs    |  47 ++++++++
 src/task/processor.rs     |  58 +++++++++-
 src/trap/mod.rs           |  56 +++++++--
 19 files changed, 768 insertions(+), 107 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`Cargo.lock`, `Cargo.toml`, `src/config.rs`, `src/debug_config.rs`, `src/fs/inode.rs`, `src/main.rs`, `src/mm/dtb.rs`, `src/mm/frame_allocator.rs`, `src/mm/memory_set.rs`, `src/mm/mod.rs` ...
- 若涉及 syscall 新增/改动：优先查看 `src/syscall/mod.rs` 的 syscall 号与分发分支。
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
