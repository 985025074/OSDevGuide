# add shebang support

- Commit: `d18a8273a190fd6a90cd47bd656c24ef46d91d7b`
- Date: `2025-12-21`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`

## 文件变更
```text
 src/syscall/filesystem.rs |  4 +-
 src/syscall/process.rs    | 95 ++++++++++++++++++++++++++++++++++-------------
 2 files changed, 72 insertions(+), 27 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`src/syscall/filesystem.rs`, `src/syscall/process.rs`
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
