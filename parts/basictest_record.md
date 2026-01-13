# basictest 记录

本文档记录 OSComp `basic` 组（syscalls 基础测例）的内容、期望输出与 CongCore 内核侧实现位置。

## 脚本入口与运行方式

- QEMU 内跑 `basic` 组的入口脚本是：
  - `testsuits-for-oskernel/scripts/basic/basic_testcode.sh`
- 它会 `cd ./basic` 后执行 `./run-all.sh`：
  - `testsuits-for-oskernel/basic/user/src/oscomp/run-all.sh`
- `run-all.sh` 逐个运行以下可执行文件：

```sh
tests="
brk
chdir
clone
close
dup2
dup
execve
exit
fork
fstat
getcwd
getdents
getpid
getppid
gettimeofday
mkdir_
mmap
mount
munmap
openat
open
pipe
read
sleep
times
umount
uname
unlink
wait
waitpid
write
yield
"
for i in $tests
do
    echo "Testing $i :"
    ./$i
done
```

> 说明：每个测例会先输出 `========== START <name> ==========`，结束时输出 `========== END <name> ==========`。
> 该输出由 `testsuits-for-oskernel/basic/user/include/stdio.h` 的 `TEST_START/TEST_END` 宏产生。

## 内核实现定位

CongCore 的 Linux syscall 分发入口：`os/src/syscall/mod.rs`。

常用实现文件：

- 进程/exec/wait/clone：`os/src/syscall/process.rs`
- 文件系统相关：`os/src/syscall/filesystem.rs`
- 内存相关：`os/src/syscall/memory.rs`
- 时间相关：`os/src/syscall/time_sys.rs`
- 杂项（uname/mount/umount/getppid 等）：`os/src/syscall/misc.rs`

下文按测例名逐个记录。

## brk

brk 语义
含义: brk/sbrk 用于控制进程的 program break（程序断点），即数据段(data segment)的尾端地址，间接决定进程堆(heap)的大小。
作用: 通过移动 program break 来扩展或收缩堆，从而为运行时分配器（如 malloc）提供或回收连续的地址空间。
接口差异: brk(addr) 将 program break 设为 addr；sbrk(delta) 将 program break 增减 delta，并返回

```rs

pub fn syscall_brk(addr: usize) -> isize {
    let process = current_process();
    let mut inner = process.borrow_mut();
    if addr == 0 {
        return inner.brk as isize;
    }
    if addr < inner.heap_start {
        return inner.brk as isize;
    }

    let old_brk = inner.brk;
    let new_brk = addr;
    let heap_start = inner.heap_start;
    let old_end = align_up(old_brk, PAGE_SIZE);
    let new_end = align_up(new_brk, PAGE_SIZE);
    if new_end > old_end {
        let _ = inner
            .memory_set
            .append_to(heap_start.into(), new_end.into());
    } else if new_end < old_end {
        let _ = inner
            .memory_set
            .shrink_to(heap_start.into(), new_end.into());
    }
    inner.brk = new_brk;
    inner.brk as isize
}


```

较为简单.需要注意的是对其地址.

## chdir

```rs
pub fn syscall_chdir(pathname: usize) -> isize {
    let token = get_current_token();
    let path = translated_str(token, pathname as *const u8);
    if path.is_empty() {
        return -1;
    }

    let process = current_process();
    let cwd = { process.borrow_mut().cwd.clone() };
    let new_cwd = normalize_path(&cwd, &path);
    let Some(inode) = ROOT_INODE.find_path(&new_cwd) else {
        return -1;
    };
    if !inode.is_dir() {
        return -1;
    }
    process.borrow_mut().cwd = new_cwd;
    0
}


```

实现简单,在 process 快中添加一个 cwd 来储存当前路径即可.
normalizePath 用于 handle .. 和 .之类的情况.规范化路径名称.

## clone

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/clone.c`

- 用户态 `clone()` 最终会通过汇编封装调用 Linux `SYS_clone(220)`：
  - `testsuits-for-oskernel/basic/user/lib/arch/riscv/clone.s`
  - 约定：`syscall(SYS_clone, flags, stack, ptid, tls, ctid)`
- 本测例传入 `flags = SIGCHLD`，属于“fork-like clone”，创建子进程并等待回收。

### 期望输出（关键行）

- `Child says successfully!`
- `clone process successfully.`

### CongCore 对应实现

- `os/src/syscall/process.rs::syscall_clone`
  - 对 `CLONE_VM`（线程模型）与非 `CLONE_VM`（进程模型）做了分支处理。
  - 本测例属于非 `CLONE_VM` 分支：本质是 fork-like 语义。

---

## close

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/close.c`

- `open("test_close.txt", O_CREATE|O_RDWR)` 创建/打开文件
- `write()` 写入一段字符串
- `close(fd)` 返回 0

### 期望输出（关键行）

- `close <fd> success.`

### CongCore 对应实现

- `os/src/syscall/filesystem.rs::syscall_openat`
- `os/src/syscall/filesystem.rs::syscall_write`
- `os/src/syscall/filesystem.rs::syscall_close`

---

## dup

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/dup.c`

- `dup(STDOUT)` 复制标准输出 fd（通常 1）
- 期望返回一个新的 fd（测例注释期望为 3）

### 期望输出（关键行）

- `new fd is 3.`（允许因 fd 分配策略不同而不是固定 3，但必须是 >=0 的有效 fd）

### CongCore 对应实现

- `os/src/syscall/filesystem.rs::syscall_dup`

---

## dup2

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/dup2.c`

- 用户库里 `dup2(old,new)` 实际走 `SYS_dup3(old,new,0)`
- 将 `STDOUT` 复制到 fd=100，然后向 fd=100 `write()`

### 期望输出（关键行）

- `from fd 100`

### CongCore 对应实现

- `os/src/syscall/filesystem.rs::syscall_dup3`
- `os/src/syscall/filesystem.rs::syscall_write`

---

## execve

### 测试点

测例文件：

- `testsuits-for-oskernel/basic/user/src/oscomp/execve.c`
- 被 exec 的程序：`testsuits-for-oskernel/basic/user/src/oscomp/test_echo.c`

`execve("test_echo", argv, envp)` 应切换为运行 `test_echo`。

### 期望输出（关键行）

- `I am test_echo.`
- `execve success.`

如果失败则会打印：`execve error.`

### CongCore 对应实现

- `os/src/syscall/process.rs::syscall_execve`
  - 会在当前进程 `cwd` 下查找可执行文件；找不到时会尝试自动追加 `.bin`。
  - 支持 ELF 与脚本 shebang（用于后续 shell/脚本兼容）。

---

## exit

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/exit.c`

- 子进程 `exit(0)`
- 父进程 `wait()` 回收并判断返回 pid

### 期望输出（关键行）

- `exit OK.`

### CongCore 对应实现

- `os/src/syscall/flow.rs::syscall_exit`
- `os/src/syscall/process.rs::syscall_wait4`

---

## fork

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/fork.c`

- 用户库 `fork()` 实际是 `syscall(SYS_clone, SIGCHLD, 0)`
- 父进程 `wait()`，子进程打印后退出

### 期望输出（关键行）

- 子进程：`child process.`
- 父进程：`parent process. wstatus:<num>`

### CongCore 对应实现

- `os/src/syscall/process.rs::syscall_clone`（fork-like 分支）
- `os/src/syscall/process.rs::syscall_wait4`

---

## fstat

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/fstat.c`

- `open("./text.txt", O_RDONLY)`
- `fstat(fd, &kst)` 返回 >= 0，打印 inode/size/time 等字段

### 期望输出（关键行）

- `fstat ret: 0`（或其他非负值）
- `fstat: dev: ..., inode: ..., mode: ..., size: ...`

### CongCore 对应实现

- `os/src/syscall/filesystem.rs::syscall_fstat`
  - 需要确保填充的结构布局与 basic 测例使用的 `struct kstat` 兼容。

---

## getcwd

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/getcwd.c`

- `getcwd(buf, 128)` 返回非 NULL 指针并打印路径

### 期望输出（关键行）

- `getcwd: <path> successfully!`

### CongCore 对应实现

- `os/src/syscall/filesystem.rs::syscall_getcwd`
  - 依赖进程控制块中维护的 `cwd` 字符串。

---

## getdents

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/getdents.c`

- `open(".", O_RDONLY)` 打开当前目录
- `getdents(fd, buf, 512)` 返回正数并打印 `dirp64->d_name`

### 期望输出（关键行）

- `getdents success.`
- 后续打印某个目录项名（不要求固定）

### CongCore 对应实现

- `os/src/syscall/filesystem.rs::syscall_getdents64`

---

## getpid

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/getpid.c`

- `getpid()` 返回 >= 0

### 期望输出（关键行）

- `getpid success.`
- `pid = <num>`

### CongCore 对应实现

- `os/src/syscall/process.rs::syscall_getpid`

---

## getppid

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/getppid.c`

- `getppid()` 返回 > 0

### 期望输出（关键行）

- `getppid success. ppid : <num>`

### CongCore 对应实现

- `os/src/syscall/misc.rs::syscall_getppid`

---

## gettimeofday

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/gettimeofday.c`

- 用户库 `get_time()` 实际调用 `SYS_gettimeofday(169)`
- 通过忙等制造时间差，要求 interval > 0

### 期望输出（关键行）

- `gettimeofday success.`
- `interval: <num>`（应大于 0）

### CongCore 对应实现

- `os/src/syscall/time_sys.rs::syscall_gettimeofday`

---

## mkdir\_

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/mkdir_.c`

- `mkdir("test_mkdir", 0666)` 成功
- `open("test_mkdir", O_RDONLY|O_DIRECTORY)` 成功

### 期望输出（关键行）

- `mkdir ret: 0`（或非 -1）
- `mkdir success.`

### CongCore 对应实现

- `os/src/syscall/filesystem.rs::syscall_mkdirat`
- `os/src/syscall/filesystem.rs::syscall_openat`

---

## mmap

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/mmap.c`

- 创建并写入 `test_mmap.txt`
- `fstat` 获取文件长度
- `mmap(NULL, len, PROT_READ|PROT_WRITE, MAP_FILE|MAP_SHARED, fd, 0)`
- 读取映射内容并打印，随后 `munmap`

### 期望输出（关键行）

- `file len: <num>`
- `mmap content:   Hello, mmap successfully!`

### CongCore 对应实现

- `os/src/syscall/memory.rs::syscall_mmap`
- `os/src/syscall/memory.rs::syscall_munmap`

---

## mount

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/mount.c`

- `mount("/dev/vda2", "./mnt", "vfat", 0, NULL)` 返回 0
- 随后 `umount("./mnt")`

### 期望输出（关键行）

- `mount return: 0`
- `mount successfully`

### CongCore 对应实现

- `os/src/syscall/misc.rs::syscall_mount`
  - 当前实现是兼容性 stub：直接返回 0。

---

## munmap

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/munmap.c`

- 先 mmap，再 `munmap(array, len)`，要求返回 0

### 期望输出（关键行）

- `munmap return: 0`
- `munmap successfully!`

### CongCore 对应实现

- `os/src/syscall/memory.rs::syscall_munmap`

---

## open

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/open.c`

- `open("./text.txt", O_RDONLY)`
- `read()` 读取内容并打印

### 期望输出（关键行）

- 打印 `text.txt` 内容（该文件位于 basic 测例目录）

### CongCore 对应实现

- `os/src/syscall/filesystem.rs::syscall_openat`
- `os/src/syscall/filesystem.rs::syscall_read`

---

## openat

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/openat.c`

- `open("./mnt", O_DIRECTORY)` 打开目录
- `openat(fd_dir, "test_openat.txt", O_CREATE|O_RDWR)` 创建文件

### 期望输出（关键行）

- `openat success.`

### CongCore 对应实现

- `os/src/syscall/filesystem.rs::syscall_openat`
  - 需要支持 `dirfd` 为目录 fd 的相对路径解析。

---

## pipe

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/pipe.c`

- `pipe(fd)` 创建管道
- fork 后子写父读，父进程读出后打印

### 期望输出（关键行）

- `Write to pipe successfully.`

### CongCore 对应实现

- `os/src/syscall/filesystem.rs::syscall_pipe2`
- `os/src/fs/pipe.rs`（管道实现）

---

## read

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/read.c`

- `open("./text.txt")` 后 `read()` 并将内容写到 stdout

### 期望输出（关键行）

- 打印 `text.txt` 内容

### CongCore 对应实现

- `os/src/syscall/filesystem.rs::syscall_read`

---

## sleep

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/sleep.c`

- 通过 `get_time()` 取时间戳（底层是 `SYS_gettimeofday`）
- `sleep(1)` 底层使用 `SYS_nanosleep(101)`

### 期望输出（关键行）

- `sleep success.`

### CongCore 对应实现

- `os/src/syscall/time_sys.rs::syscall_gettimeofday`
- `os/src/syscall/time_sys.rs::syscall_nanosleep`

---

## times

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/times.c`

- `times(&tms)` 返回 >= 0，并打印 4 个字段

### 期望输出（关键行）

- `mytimes success`

### CongCore 对应实现

- `os/src/syscall/time_sys.rs::syscall_times`

---

## umount

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/umount.c`

- 先 mount 再 umount，umount 返回 0

### 期望输出（关键行）

- `umount success.`

### CongCore 对应实现

- `os/src/syscall/misc.rs::syscall_umount2`
  - 当前实现是兼容性 stub：直接返回 0。

---

## uname

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/uname.c`

- `uname(&un)` 返回 >= 0，打印 6 个字段

### 期望输出（关键行）

- `Uname: ...`（字段非空，且 release/version 等不应导致用户态程序误判“内核太旧”）

### CongCore 对应实现

- `os/src/syscall/misc.rs::syscall_uname`
  - 当前实现将 `release` 伪装为 `5.15.0` 以提升 glibc/busybox 兼容性。

---

## unlink

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/unlink.c`

- 创建文件 `./test_unlink`
- `unlink()` 后再次 `open()`，应失败

### 期望输出（关键行）

- `unlink success!`

测例注释说明：不强制必须回收 inode/data block，只要路径查找不到即可。

### CongCore 对应实现

- `os/src/syscall/filesystem.rs::syscall_unlinkat`

---

## wait

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/wait.c`

- fork 后子进程退出
- 父进程 `wait(&wstatus)` 回收

### 期望输出（关键行）

- `wait child success.`

### CongCore 对应实现

- `os/src/syscall/process.rs::syscall_wait4`（pid = -1 表示任意子进程）

---

## waitpid

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/waitpid.c`

- 子进程 `exit(3)`
- 父进程 `waitpid(cpid, &wstatus, 0)`
- 使用 `WEXITSTATUS(wstatus)` 期望得到 3

### 期望输出（关键行）

- `waitpid successfully.`
- `wstatus: 3`

### CongCore 对应实现

- `os/src/syscall/process.rs::syscall_wait4`
  - 需要按 Linux wait status 编码：正常退出为 `(code & 0xff) << 8`。

---

## write

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/write.c`

- `write(STDOUT, "Hello operating system contest.\n", len)` 返回写入长度

### 期望输出（关键行）

- `Hello operating system contest.`

### CongCore 对应实现

- `os/src/syscall/filesystem.rs::syscall_write`

---

## yield

### 测试点

测例文件：`testsuits-for-oskernel/basic/user/src/oscomp/yield.c`

- fork 3 个子进程，每个子进程循环 5 次：
  - `sched_yield()`
  - `printf("I am child process ...")`

理想现象：三组输出交替出现（体现调度让出）。

### 期望输出（关键行）

- 会出现多行：`I am child process: <pid>. iteration <i>.`

### CongCore 对应实现

- `os/src/syscall/flow.rs::syscall_yield`
  - 调用 `suspend_current_and_run_next()` 让出 CPU。
