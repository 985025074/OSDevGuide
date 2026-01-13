# fix kill bug

- Commit: `ca09c25623137a9530d25f6de29a7b99985ecef6`
- Date: `2025-12-22`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`, `task`

## 文件变更
```text
 src/syscall/process.rs | 3 +++
 src/task/processor.rs  | 3 +--
 2 files changed, 4 insertions(+), 2 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`src/syscall/process.rs`, `src/task/processor.rs`
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
