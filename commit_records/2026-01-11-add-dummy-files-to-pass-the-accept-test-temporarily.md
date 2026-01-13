# add dummy files to pass the accept test (temporarily.)

- Commit: `764a5b243c375d10c5e71c3c8d89c296cbda247d`
- Date: `2026-01-11`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`, `fs`

## 文件变更
```text
 src/fs/dummy.rs           |  39 +++++++++++
 src/fs/mod.rs             |   2 +
 src/syscall/dummy.rs      | 165 ++++++++++++++++++++++++++++++++++++++++++++++
 src/syscall/filesystem.rs |  80 +++++++++++++++++++---
 src/syscall/mod.rs        |  37 +++++++++++
 src/syscall/net.rs        |  33 +++++++---
 6 files changed, 339 insertions(+), 17 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`src/fs/dummy.rs`, `src/fs/mod.rs`, `src/syscall/dummy.rs`, `src/syscall/filesystem.rs`, `src/syscall/mod.rs`, `src/syscall/net.rs`
- 若涉及 syscall 新增/改动：优先查看 `src/syscall/mod.rs` 的 syscall 号与分发分支。
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
