# improve makefile

- Commit: `c0cbe3ff9ac9a86fe7614db128fe7e1d932e0b07`
- Date: `2025-12-21`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`, `build`

## 文件变更
```text
 Makefile                  | 37 ++++++++++++++-----
 src/debug_config.rs       |  3 ++
 src/syscall/filesystem.rs | 92 +++++++++++++++++++++++++++++++++++++++++++----
 3 files changed, 117 insertions(+), 15 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`Makefile`, `src/debug_config.rs`, `src/syscall/filesystem.rs`
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
