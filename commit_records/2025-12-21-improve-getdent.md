# improve getdent

- Commit: `016873869c7e590b364eec6e3b43896a55002fe2`
- Date: `2025-12-21`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`

## 文件变更
```text
 src/console/mod.rs        |   3 ++
 src/klog.rs               | 106 ++++++++++++++++++++++++++++++++++++++++++++
 src/lib.rs                |   1 +
 src/main.rs               |   1 +
 src/syscall/filesystem.rs | 109 ++++++++++++++++++++++++++++++++++------------
 src/syscall/misc.rs       |  53 +++++++++++++++++++---
 6 files changed, 239 insertions(+), 34 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`src/console/mod.rs`, `src/klog.rs`, `src/lib.rs`, `src/main.rs`, `src/syscall/filesystem.rs`, `src/syscall/misc.rs`
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
