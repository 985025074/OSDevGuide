# finish iperf_test

- Commit: `b674c52322b632cfd50fef8ca3bf5fd65260b0a2`
- Date: `2026-01-04`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`, `fs`, `net`, `build`

## 文件变更
```text
 Cargo.lock                | 140 ++++++++
 Cargo.toml                |   8 +
 src/debug_config.rs       |   3 +
 src/fs/mod.rs             |   4 +-
 src/fs/net_socket.rs      | 801 ++++++++++++++++++++++++++++++++++++++++++++++
 src/fs/pseudo.rs          |  17 +
 src/lib.rs                |   1 +
 src/main.rs               |   1 +
 src/net/mod.rs            |  80 +++++
 src/syscall/filesystem.rs |  15 +
 src/syscall/misc.rs       |   7 +
 src/syscall/mod.rs        |  25 ++
 src/syscall/net.rs        | 436 +++++++++++++++++++++++++
 src/syscall/time_sys.rs   | 138 +++++++-
 14 files changed, 1666 insertions(+), 10 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`Cargo.lock`, `Cargo.toml`, `src/debug_config.rs`, `src/fs/mod.rs`, `src/fs/net_socket.rs`, `src/fs/pseudo.rs`, `src/lib.rs`, `src/main.rs`, `src/net/mod.rs`, `src/syscall/filesystem.rs` ...
- 若涉及 syscall 新增/改动：优先查看 `src/syscall/mod.rs` 的 syscall 号与分发分支。
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
