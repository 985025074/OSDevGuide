# update ps semantics

- Commit: `e909497efff3bd342d6b8e86c41d53cbce8d99c8`
- Date: `2025-12-22`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`, `task`

## 文件变更
```text
 src/syscall/filesystem.rs | 159 ++++++++++++++++++++++++++++++++++++++++------
 src/task/process_block.rs |  10 +++
 2 files changed, 150 insertions(+), 19 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`src/syscall/filesystem.rs`, `src/task/process_block.rs`
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
