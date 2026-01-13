# fix shared memory and abort bugs. this is used in ltp_testcode.sh

- Commit: `a5e07946d33ceea730c652a0e0db44ccd2314005`
- Date: `2026-01-10`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围
- `syscall`, `fs`, `task`, `trap`

## 文件变更
```text
 src/fs/mod.rs             |   2 +-
 src/fs/net_socket.rs      |  12 +++++
 src/fs/pseudo.rs          | 127 ++++++++++++++++++++++++++++++++++++++--------
 src/link_app.asm          |  15 +++++-
 src/syscall/filesystem.rs | 107 ++++++++++++++++++++++++++++++++++++++
 src/syscall/memory.rs     |  84 +++++++++++++++++++++++++-----
 src/syscall/misc.rs       |  19 ++++++-
 src/syscall/mod.rs        |   8 +++
 src/syscall/net.rs        |  87 ++++++++++++++++++++++++-------
 src/syscall/process.rs    |  14 ++++-
 src/syscall/signal.rs     |  29 +++++++++--
 src/task/process_block.rs |   7 +++
 src/trap/mod.rs           |  10 ++++
 13 files changed, 460 insertions(+), 61 deletions(-)
```

## 摘要
- 本提交的具体语义以 diff 为准；下面给出基于文件路径的快速定位建议。
- 重点文件：`src/fs/mod.rs`, `src/fs/net_socket.rs`, `src/fs/pseudo.rs`, `src/link_app.asm`, `src/syscall/filesystem.rs`, `src/syscall/memory.rs`, `src/syscall/misc.rs`, `src/syscall/mod.rs`, `src/syscall/net.rs`, `src/syscall/process.rs` ...
- 若涉及 syscall 新增/改动：优先查看 `src/syscall/mod.rs` 的 syscall 号与分发分支。
- 若涉及用户态兼容：结合 `src/syscall/*` 与对应子系统（fs/mm/task/net）一起看。
