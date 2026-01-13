# add some functions and add more syscalls

- Commit: `3e1b15218d744ff2721c719fe43c7210fd2538e4`
- Date: `2025-12-20`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围

- `syscall`, `fs`, `mm`

## 主要改动

- 引入 PseudoFile 伪文件框架，为 /sys、/proc、/dev 提供最小可用的兼容文件。
- 新增 setpgid/getsid/setsid 等 syscall（兼容 stub）。
- wait4 状态码编码更贴近 Linux；内存映射路径在 OOM 时更稳健。

## 新增功能（做了什么 + 在哪）

### 1) 伪文件框架：让最小的 /sys、/proc、/dev 可用

**涉及文件**

- `src/fs/pseudo.rs`：新增 `PseudoFile`，实现 `File` trait。
- `src/fs/mod.rs`：导出 `PseudoFile`。
- `src/syscall/filesystem.rs`：`openat` 在路径命中 `/sys/` `/proc/` `/dev/` 时尝试走 `open_pseudo()`。

**支持的伪文件类型**（`src/fs/pseudo.rs`）

- `Static(Vec<u8>)`：只读静态内容（用于 `/proc/loadavg`、`/sys/...`）。
- `Urandom(u64)`：只读伪随机（xorshift64\*），用于 `/dev/random` `/dev/urandom`。
- `Null`：可读写；读返回 0，写丢弃并返回写入长度（`/dev/null`）。
- `Zero`：只读；读返回全 0（`/dev/zero`）。

**openat 的路径分流逻辑**（`src/syscall/filesystem.rs::syscall_openat`）

- `path.starts_with("/sys/") || path.starts_with("/proc/") || path.starts_with("/dev/")` 时先尝试打开伪文件。
- 命中则直接分配一个 fd，并把 `Arc<dyn File>` 放进 `fd_table`，绕过 ext4 实际路径解析。

**已实现的具体路径**（`src/syscall/filesystem.rs::open_pseudo`）

- CPU 相关：
  - `/sys/devices/system/cpu/possible|present|online`：返回 `0-(MAX_HARTS-1)` 或 `0`。
  - `/sys/devices/system/cpu/kernel_max`：返回 `MAX_HARTS-1`。
- NUMA node：`/sys/devices/system/node/online|possible`：返回 `0`。
- 负载：`/proc/loadavg`：返回固定占位字符串。
- 设备：`/dev/null` `/dev/zero` `/dev/random` `/dev/urandom`。

### 2) 补齐 setpgid/getsid/setsid（兼容 stub）

**涉及文件**

- `src/syscall/misc.rs`：新增 `syscall_setpgid/syscall_getsid/syscall_setsid`。
- `src/syscall/mod.rs`：添加 riscv64 对应 syscall 号并分发。

**语义（当前实现）**

- `setpgid(pid, pgid)`：不建模进程组，直接返回 0（成功）。
- `getsid(pid)`：`pid==0` 时返回当前 pid；否则原样返回 `pid`。
- `setsid()`：不建模 session，返回当前 pid 作为 sid。

### 3) wait4 状态码编码贴近 Linux

**涉及文件**

- `src/syscall/process.rs::syscall_wait4`

**变化点**

- 以前：把内部 `exit_code` 原样写回 `*wstatus`，用户态用 Linux 规则解析会读错。
- 现在：
  - 正常退出：`status = (code & 0xff) << 8`
  - 信号导致退出（内部用负数编码）：`status = (-code) & 0x7f`

**会触发什么问题（修复前）**

- 父进程/测试框架按 Linux 规则解析 `WEXITSTATUS/WIFSIGNALED` 时结果不一致：
  - 表现为：程序明明 `exit(1)`，但父进程读到的可能不是 `1`；或者误判为被信号杀死。

### 4) OOM 时不再直接 panic（避免内核被打挂）

**涉及文件**

- `src/mm/memory_set.rs`：`MapArea::map_one` 从 `unwrap()` 改为返回 `bool`，失败打印 OOM 并让上层中止继续映射。

**会触发什么问题（修复前）**

- 当 `frame_alloc()` 返回 `None`（物理页耗尽）时：
  - 旧代码 `unwrap()` 直接 panic。
  - 表现为：一次大 mmap / 加载大 ELF / 连续分配用户栈等场景下，内核直接崩溃。

**修复后的行为**

- 不再 panic；会停止继续映射剩余页（可能得到“部分成功”的映射结果）。
- 这属于“容错降级”，后续可以再演进到：返回 -ENOMEM 给 syscall 层、并回滚已映射区域。

## 文件变更

```text
 src/fs/mod.rs             |   2 +
 src/fs/pseudo.rs          | 146 ++++++++++++++++++++++++++++++++++++++++++++++
 src/mm/memory_set.rs      |  16 +++--
 src/syscall/filesystem.rs |  61 ++++++++++++++++++-
 src/syscall/misc.rs       |  23 ++++++++
 src/syscall/mod.rs        |   6 ++
 src/syscall/process.rs    |  10 +++-
 7 files changed, 258 insertions(+), 6 deletions(-)
```

## 简单验证（建议）

- `/dev/null`：打开后 write 返回写入长度；read 返回 0。
- `/dev/zero`：read 返回全 0。
- `/sys/devices/system/cpu/online`：返回与 `MAX_HARTS` 对应的范围。
- `wait4`：父子进程验证 `WEXITSTATUS` 是否为预期（例如子进程 `exit(42)`）。
