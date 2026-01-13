# busybox

## 什么是 busybox

busybox 是一个“多调用（multi-call）”的单一可执行文件：一个 `busybox` 二进制里包含了大量常用的 Unix/Linux 工具（如 `ls/cp/mv/grep/find`）以及一个轻量 shell（`sh/ash`）。

- 进程管理：`fork/clone/execve/wait/exit` 等
- 文件系统：`open/read/write/getdents/stat` 等
- 运行时启动：ELF 加载、初始栈布局（argc/argv/envp/auxv）、TLS
- glibc 兼容：信号、`mprotect`、`set_tid_address`、`set_robust_list`、`gettid` 等

因此 busybox 测试更像是“系统级集成测试”，覆盖面明显大于单个 syscall 的 basic 测试。

## 测试做什么

### 测试脚本入口

busybox 测试由脚本 [testsuits-for-oskernel/scripts/busybox/busybox_testcode.sh](../../testsuits-for-oskernel/scripts/busybox/busybox_testcode.sh) 驱动。它会读取同目录下的 `busybox_cmd.txt`，对每一行命令执行：

```sh
eval "./busybox $line"
```

并根据返回码判断 success/fail（`false` 是特例：允许非 0 仍判 success）。

在整套测试编排里，主机侧脚本 [scripts/run_os_tests.sh](../../scripts/run_os_tests.sh) 会在 QEMU 内分别进入 `/extra/riscv/musl` 与 `/extra/riscv/glibc`（两套 userland），并通常以 `./busybox sh ./busybox_testcode.sh` 的方式运行该脚本。

### 命令覆盖范围（busybox_cmd.txt）

命令清单在 [testsuits-for-oskernel/scripts/busybox/busybox_cmd.txt](../../testsuits-for-oskernel/scripts/busybox/busybox_cmd.txt)。大致分两部分：

1. independent command test

- shell：`ash -c exit`、`sh -c exit`
- 常用命令：`basename/dirname/date/df/du/expr/which/uname/uptime/printf/ps/pwd/free/hwclock/ls/sleep`
- 进程信号：`sh -c 'sleep 5' & ./busybox kill $!`

2. file operation test

- 文件创建/读写/重定向：`touch`、`echo > / >>`、`cat`、`head/tail/more`
- 文本/二进制处理：`cut/od/hexdump/md5sum/strings/wc/sort | uniq`
- 元信息：`stat`、`[ -f test.txt ]`
- 文件/目录操作：`rm/mkdir/mv/rmdir/cp/find/grep`

注意：脚本用的是 `eval`，因此重定向（`>`、`>>`）、管道（`|`）和后台（`&`）等语法由运行脚本的 shell 解析；每条用例的“第一个命令”会被自动加上 `./busybox` 前缀。

### 与内核能力的关系

这些命令会间接触发大量内核路径（不仅是文件读写）：

- 动态程序启动（glibc 目录）：需要更接近 Linux 的 ELF 初始化栈（`auxv`）、TLS、以及一组“启动早期 syscall”。
- `printf`/管道等场景常用 `writev/readv`。
- `ps/df/free` 等命令会读 `/proc` 或依赖若干兼容性 syscall（不同 busybox 配置会有所差异）。

## os 做的相应改变

以下内容对应 `os/` 目录中的提交 `165283f773d4b8c2c33ec55ccfa52e64d58782c5 (add busy box support)`：该提交的目标是让 busybox（尤其是 glibc 版本）可以启动并跑完 `busybox_cmd.txt`。

### 1) syscall 分发策略：未知 syscall 不再 panic

- [os/src/syscall/mod.rs](../../os/src/syscall/mod.rs)：补齐/登记一批 busybox/glibc 启动常见 syscall 号；并将“未知 syscall”从 panic 改为返回 `-ENOSYS`（Linux 语义是 `-38`）。
- [os/src/debug_config.rs](../../os/src/debug_config.rs)：新增 `DEBUG_SYSCALL`，用于在调试 busybox 启动阶段输出 syscall trace（默认关闭）。

### 2) 向量化 I/O：readv/writev

- [os/src/syscall/flow.rs](../../os/src/syscall/flow.rs)：实现 `syscall_readv/syscall_writev`（解析 iovec 数组，逐段调用已有 `read/write`）。
- 同时将 stdout 的写入从“按 UTF-8 字符串打印”改为“逐字节 `console_putchar`”，避免 busybox 输出包含非 UTF-8 数据时触发 `unwrap()` 崩溃。

### 3) stdio 更接近 Linux：不再对错误方向读写 panic

- [os/src/fs/stdio.rs](../../os/src/fs/stdio.rs)：
  - stdin 的 `write()` 返回 0（以前会 panic）。
  - stdout 的 `read()` 返回 0（以前会 panic）。

这类行为在真实 Linux 上通常是返回错误码（如 EBADF），但“返回 0 不崩”能让更多用户态程序继续跑下去，有利于通过测试。

### 4) ELF 加载与初始栈：支持 glibc 需要的 auxv/envp/TLS

glibc 动态链接器启动时非常依赖初始栈布局（argc/argv/envp/auxv）以及 TLS。提交中做了几件关键事：

- [os/src/config.rs](../../os/src/config.rs)：将 `USER_STACK_SIZE` 扩大到 1MiB（busybox/glibc 的初始栈更“吃空间”）。
- [os/src/mm/memory_set.rs](../../os/src/mm/memory_set.rs)：
  - `MemorySet::from_elf` 额外返回 `ElfAux { phdr, phent, phnum }`，并尽力计算 `AT_PHDR` 所需的 program header 表虚拟地址。
  - 修复段加载时的“页内偏移”拷贝问题（`MapArea` 增加 `start_offset`，`copy_data` 支持非页对齐段）。
- [os/src/task/process_block.rs](../../os/src/task/process_block.rs)：
  - 引入 `build_linux_stack()`：在用户栈上构造 Linux 风格布局，包含 `envp` 与 `auxv`（如 `AT_PHDR/AT_PHENT/AT_PHNUM/AT_ENTRY/AT_PAGESZ/AT_RANDOM/...`），并保持 16 字节对齐。
  - 在 `ProcessControlBlock::new/exec` 中使用该栈布局，并设置 `a0/a1/a2` 为 `argc/argv/envp`。
- [os/src/trap/trap.asm](../../os/src/trap/trap.asm)：trap 入口/出口正确区分并恢复用户态 `tp`（TLS），同时在内核中使用 `tp` 作为 hart id。

### 5) 目录遍历与文件权限语义：getdents64 与 readable/writable

- [os/src/fs/inode.rs](../../os/src/fs/inode.rs) + [os/src/syscall/filesystem.rs](../../os/src/syscall/filesystem.rs)：
  - 为目录读取引入独立的 `dir_offset`，避免 `getdents64` 与普通文件 `read` 共享同一个 offset。
  - `getdents64` 使用内核缓冲区批量构造 `linux_dirent64` 并再拷回用户态，降低逐字节地址翻译开销。
  - `read/write` 增加 `readable()/writable()` 检查，避免 busybox 在错误 fd/权限场景下出现异常行为。

### 6) glibc 启动常用 syscall：mprotect / set_tid_address / robust_list / uid/gid/tid / 信号

- [os/src/syscall/memory.rs](../../os/src/syscall/memory.rs)：加入 `mprotect` stub（直接返回 0）。
- [os/src/syscall/misc.rs](../../os/src/syscall/misc.rs)：
  - `uname` 伪装 `release` 为 `5.15.0`（避免用户态因“内核版本过低”走到不兼容路径）。
  - `getuid/geteuid/getgid/getegid` 返回 0。
  - `gettid` 返回当前 pid（简化实现）。
  - `set_tid_address` 返回 pid；`set_robust_list` 返回 0。
- [os/src/syscall/signal.rs](../../os/src/syscall/signal.rs)：
  - `tgkill` 兼容实现（退化调用已有 `kill` 路径）。
  - `rt_sigaction/rt_sigprocmask/rt_sigreturn` 以“最小可用”的 stub 形式返回成功，并在需要时清零 old buffer，满足 glibc/busybox 启动阶段对信号接口的探测。

### 7) execve 细节：argv[0] 兜底

- [os/src/syscall/process.rs](../../os/src/syscall/process.rs)：当用户传入的 argv 为空时，补上 `argv[0] = path`（一些程序/脚本依赖 argv[0] 非空）。
