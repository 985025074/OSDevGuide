# fix system semnatics

- Commit: `704bd3185028a85cee8ea81eb735c3a57bcef678`
- Date: `2025-12-21`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`, `fs`

## 文件变更
```text
 src/fs/pseudo.rs          |  19 +-
 src/syscall/filesystem.rs | 782 +++++++++++++++++++++++++++++++++-------------
 2 files changed, 583 insertions(+), 218 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`src/fs/pseudo.rs`, `src/syscall/filesystem.rs`
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
