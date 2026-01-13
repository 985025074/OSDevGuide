# add busy box support

- Commit: `165283f773d4b8c2c33ec55ccfa52e64d58782c5`
- Date: `2025-12-20`
- Range: `ceacb089766c4055cbdbb7fd58b0e378bb8788a3..HEAD`

## 影响范围

- `syscall`, `fs`, `mm`, `task`, `trap`

## 主要改动

- busybox 支持：补齐 readv/writev/exit_group/mprotect 等常用 syscall，未知 syscall 返回 -ENOSYS。
- glibc 启动链路：构造 Linux 风格初始栈（auxv/envp）、扩大用户栈、修复 TLS(tp) 保存/恢复。
- 目录读取与 I/O：优化 getdents64 与 stdout 输出路径，避免因非 UTF-8 输出崩溃。

## 新增功能（做了什么 + 在哪）

### 1) 补齐一批 busybox/glibc 常用 syscall（含兼容 stub）

**涉及文件**

- `src/syscall/mod.rs`：新增 syscall 号、分发逻辑、unknown syscall 行为、可选 syscall trace。
- `src/syscall/flow.rs`：实现 `readv/writev`。
- `src/syscall/memory.rs`：实现 `mprotect`（当前为兼容 stub）。
- `src/syscall/misc.rs`：补齐 `getuid/getgid/.../gettid/set_robust_list/set_tid_address`。
- `src/syscall/signal.rs`：补齐 `tgkill/rt_sigaction/rt_sigprocmask/rt_sigreturn`（兼容 stub）。

**新增/修改的 syscall（摘要）**

- I/O：
  - `readv(65)` / `writev(66)`：按 `iovec[]` 逐段调用 `read/write`，累计返回值；中途错误按 Linux 习惯返回“已完成字节数”或错误码。
- 进程退出：
  - `exit_group(94)`：当前直接复用 `exit()`（足够覆盖 busybox 典型单线程场景）。
- 线程与 glibc runtime：
  - `set_tid_address(96)`：记录 tid 指针（后续 futex/clear_child_tid 语义在另一个提交中补齐）。
  - `set_robust_list(99)`：返回 0（glbic 会在启动早期调用，不支持也要“先过”).
  - `gettid(178)`：返回 tid（本提交里暂用 pid 近似）。
- 权限信息：
  - `getuid/geteuid/getgid/getegid`：全部返回 0。
- 内存权限：
  - `mprotect(226)`：返回 0（当前不真正更改页权限，先保证程序启动链路不断）。
- 信号：
  - `tgkill(131)`：当前当作 `kill(tgid, sig)` 处理。
  - `rt_sigaction(134)` / `rt_sigprocmask(135)` / `rt_sigreturn(139)`：返回 0，并按需把 oldact/oldset 区域清 0。

### 2) unknown syscall 不再 panic：改为返回 -ENOSYS

**涉及文件**

- `src/syscall/mod.rs`

**会触发什么问题（修复前）**

- busybox/glibc 运行过程中探测/调用到未实现 syscall 时：
  - 旧行为 `panic!("Unknown syscall")` 直接把内核打崩。
  - 表现为：用户态程序一运行就触发内核 panic，无法进一步定位是哪个 syscall 缺失。

**修复后的行为**

- unknown syscall 返回 `-38`（`-ENOSYS`），更符合 Linux 语义，也便于用户态 fallback 或打印错误。

### 3) 可控 syscall trace：用于追 glibc/busybox 启动

**涉及文件**

- `src/debug_config.rs`：新增 `DEBUG_SYSCALL`。
- `src/syscall/mod.rs`：当 `DEBUG_SYSCALL=true` 时，打印一小段 syscall trace（带 a0~a5）。

### 4) stdout 输出不再假设 UTF-8；stdin/stdout 不再 panic

**涉及文件**

- `src/syscall/flow.rs`：stdout 分支改为逐字节 `console_putchar`，不再 `from_utf8().unwrap()`。
- `src/fs/stdio.rs`：`Stdin::write/Stdout::read` 从 panic 改为返回 0。

**会触发什么问题（修复前）**

- busybox 常见命令会输出非 UTF-8 或二进制（例如某些工具/重定向场景）：
  - 旧实现 `from_utf8(...).unwrap()` 直接 panic。
  - 表现为：一旦输出包含非法 UTF-8，内核/运行时崩溃。

### 5) getdents64：目录遍历更稳、更快

**涉及文件**

- `src/fs/inode.rs`：`OSInodeInner` 增加 `dir_offset`（独立于文件 read offset）。
- `src/syscall/filesystem.rs::syscall_getdents64`：
  - 用 `dir_offset` 维护目录枚举进度，避免和普通文件 read offset 冲突。
  - 先在内核缓冲区拼装记录，再批量拷贝到用户态（避免逐字节地址翻译开销）。

**会触发什么问题（修复前）**

- 目录读取和普通 read 共用 offset 时：
  - 可能导致 readdir 进度错乱或影响后续文件 read。
- 逐字节写用户缓冲区：
  - 性能差；且跨页时更容易因为边界处理粗糙导致返回异常。

## glibc 启动链路兼容（关键 bug 修复）

### 1) Linux 风格初始栈：argc/argv/envp/auxv

**涉及文件**

- `src/task/process_block.rs`：新增 `build_linux_stack()` 并用于 `ProcessControlBlock::new/exec`。
- `src/mm/memory_set.rs`：`MemorySet::from_elf` 额外返回 `ElfAux { phdr, phent, phnum }`，用于构造 auxv。

**做了什么**

- 把用户栈布局改成 Linux ABI 常见格式：
  - `argc`、`argv[]`、`envp[]`、`auxv[]`（包括 `AT_PHDR/AT_PHENT/AT_PHNUM/AT_PAGESZ/AT_ENTRY/AT_UID/.../AT_RANDOM`）。
- 保证最终 `sp` 16 字节对齐。
- 启动寄存器约定：`a0=argc`、`a1=argv`、`a2=envp`。

**会触发什么问题（修复前）**

- glibc 动态链接器/启动代码会读取 `auxv`（例如 PHDR 信息、随机数种子、页大小等）。
- 旧栈布局缺少 envp/auxv 或对齐不符合预期时：
  - 表现为：glibc 在启动早期崩溃、非法访问，或直接 `abort()`。

### 2) 扩大用户栈：从 8KiB 提升到 1MiB

**涉及文件**

- `src/config.rs`：`USER_STACK_SIZE = 4096 * 256`。

**会触发什么问题（修复前）**

- busybox/glibc 初始栈上会放较多字符串、auxv、动态链接器数据结构：
  - 8KiB 很容易爆栈。
  - 表现为：启动阶段就出现用户态 page fault / 随机崩溃。

### 3) 修复 TLS(tp) 保存/恢复：避免 glibc 线程局部变量被破坏

**涉及文件**

- `src/trap/trap.asm`

**修复点**

- trap 入口：保存用户态 `tp(x4)`，并把 `tp` 切换为内核使用的 hart id。
- trap 返回：保存内核 `tp` 到 TrapContext，再恢复用户态 `tp`。

**会触发什么问题（修复前）**

- glibc 使用 `tp` 作为 TLS 基址；若内核把 `tp` 当 hart id 用且不恢复：
  - 表现为：用户态访问 TLS（errno、pthread 结构等）直接读写到错误地址，导致随机崩溃。

### 4) uname 版本号“过旧”导致的早退

**涉及文件**

- `src/syscall/misc.rs::syscall_uname`：`release` 从 `0.1` 改为 `5.15.0`。

**会触发什么问题（修复前）**

- 部分 glibc/busybox/测试程序会依据 `uname.release` 做能力判断：
  - 过旧版本可能触发“直接放弃/走不兼容路径”。
  - 表现为：启动早期异常退出（通常无太多上下文）。

## 细节改动（容易忽略但很关键）

- `src/syscall/filesystem.rs`：read/write 先检查 `file.readable()/writable()`，行为更像 Linux（避免写只读 fd）。
- `src/mm/memory_set.rs`：
  - `from_elf` 计算 `max_end_vpn` 改为取 max（避免被后续更小 segment 覆盖）。
  - `MapArea::copy_data` 支持段起始地址非页对齐（用 `start_offset` 修正首页拷贝位置），否则程序段内容会被“错位拷贝”。

## 验证建议

- busybox：运行 `ls/cat/echo` 等基础命令，重点覆盖 `getdents64` 与 stdout。
- glibc：运行一个最简单的动态链接程序，观察是否能稳定启动（TLS/auxv 是关键）。
- 若需要追启动问题：把 `src/debug_config.rs::DEBUG_SYSCALL` 打开，观察前几百个 syscall。

## 文件变更

```text
 src/config.rs             |   3 +-
 src/debug_config.rs       |   3 +
 src/fs/inode.rs           |  15 +++-
 src/fs/stdio.rs           |   4 +-
 src/mm/memory_set.rs      |  64 ++++++++++++----
 src/mm/mod.rs             |   2 +-
 src/syscall/filesystem.rs |  56 ++++++++------
 src/syscall/flow.rs       |  55 +++++++++++++-
 src/syscall/memory.rs     |   9 +++
 src/syscall/misc.rs       |  39 +++++++++-
 src/syscall/mod.rs        |  75 ++++++++++++++-----
 src/syscall/process.rs    |   4 +-
 src/syscall/signal.rs     |  49 +++++++++++-
 src/task/process_block.rs | 185 ++++++++++++++++++++++++++++++++++++----------
 src/trap/trap.asm         |   6 +-
 15 files changed, 464 insertions(+), 105 deletions(-)
```
