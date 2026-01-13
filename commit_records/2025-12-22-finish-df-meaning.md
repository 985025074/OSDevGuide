# finish df meaning

- Commit: `a346c454712191a4e05f403f5ef62f9afebd4683`
- Date: `2025-12-22`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`, `fs`

## 文件变更
```text
 src/fs/inode.rs           | 17 ++++++++-
 src/fs/mod.rs             |  3 +-
 src/fs/pseudo.rs          | 48 ++++++++++++++++++++++++
 src/syscall/filesystem.rs | 96 ++++++++++++++++++++++++++++++++++++-----------
 src/syscall/mod.rs        | 27 +++++++++++--
 5 files changed, 163 insertions(+), 28 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`src/fs/inode.rs`, `src/fs/mod.rs`, `src/fs/pseudo.rs`, `src/syscall/filesystem.rs`, `src/syscall/mod.rs`
- 若涉及 syscall 新增/改动：优先查看 `src/syscall/mod.rs` 的 syscall 号与分发分支。
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
