# fix TID nug and shell bug

- Commit: `0821cbade5318be41f4286676e379b1b83a769bb`
- Date: `2025-12-21`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`, `fs`, `task`

## 文件变更
```text
 src/fs/pipe.rs            |  62 +++++++++------
 src/fs/stdio.rs           |  44 +++++++----
 src/syscall/filesystem.rs | 196 ++++++++++++++++++++++++++++++++++++++++++----
 src/syscall/flow.rs       |  35 +--------
 src/syscall/misc.rs       | 196 ++++++++++++++++++++++++++++++++++++++++++++--
 src/syscall/mod.rs        |  18 +++++
 src/syscall/process.rs    |  29 +++++--
 src/syscall/sched.rs      |  25 ++++--
 src/syscall/time_sys.rs   |  23 +++++-
 src/task/processor.rs     |  18 +++--
 10 files changed, 532 insertions(+), 114 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`src/fs/pipe.rs`, `src/fs/stdio.rs`, `src/syscall/filesystem.rs`, `src/syscall/flow.rs`, `src/syscall/misc.rs`, `src/syscall/mod.rs`, `src/syscall/process.rs`, `src/syscall/sched.rs`, `src/syscall/time_sys.rs`, `src/task/processor.rs`
- 若涉及 syscall 新增/改动：优先查看 `src/syscall/mod.rs` 的 syscall 号与分发分支。
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
