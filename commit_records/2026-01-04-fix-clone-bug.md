# fix clone bug

- Commit: `f1f1dfb0af2ae85f81c8ca9506956f946d1fccda`
- Date: `2026-01-04`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`

## 文件变更
```text
 src/syscall/process.rs | 17 ++++++++++++++---
 1 file changed, 14 insertions(+), 3 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`src/syscall/process.rs`
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
