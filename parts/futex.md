# futex

## 什么是 futex

futex 是 Linux 提供的一种低层同步机制，用用户态原子操作完成快路径，在竞争时通过内核实现高效阻塞与唤醒
futex 不是一个内核对象

它是：

一个 用户态内存地址

一个 内核中的等待队列（按地址哈希）

关键点：

内核不维护 mutex 对象，只在必要时帮你“睡/醒”

```c
int futex(int *uaddr, int op, int val,
const struct timespec *timeout,
int \*uaddr2, int val3);
```

你平时几乎不会直接用，而是：

pthread_mutex

pthread_cond

sem_wait

std::mutex

Java / Go runtime

👇 全都基于 futex

## 1. 改动概览（文件级）

提交统计（`12 files changed, 531 insertions(+), 49 deletions(-)`），核心新增/修改点如下：

- 新增 futex syscall 实现：[os/src/syscall/futex.rs](../../os/src/syscall/futex.rs)
- 新增 sched 系列 syscall（大量是兼容/存取 PCB 字段的“最小实现”）：[os/src/syscall/sched.rs](../../os/src/syscall/sched.rs)
- syscall 分发表接线 futex/sched： [os/src/syscall/mod.rs](../../os/src/syscall/mod.rs)
- 线程式 `clone(CLONE_VM|...)` 支持： [os/src/syscall/process/fork.rs](../../os/src/syscall/process/fork.rs)
- tid/资源模型支持“仅 trap_cx，无内核分配用户栈”的线程： [os/src/task/id.rs](../../os/src/task/id.rs)
- 线程退出时的 `clear_child_tid + futex_wake`（pthread join 关键路径）： [os/src/task/processor.rs](../../os/src/task/processor.rs)
- TCB 增加 `clear_child_tid` 字段 + 新建 linux thread 的构造： [os/src/task/task_block.rs](../../os/src/task/task_block.rs)
- `set_tid_address/gettid` 改为返回 tid 并记录 clear 地址： [os/src/syscall/misc/other.rs](../../os/src/syscall/misc/other.rs)
- 信号投递时唤醒所有线程： [os/src/task/signal.rs](../../os/src/task/signal.rs)
- trap 返回前检查致命信号并退出： [os/src/trap/mod.rs](../../os/src/trap/mod.rs)

## 2. syscall 接线：新增 FUTEX=98 + sched(118~127)

在 [os/src/syscall/mod.rs](../../os/src/syscall/mod.rs) 中：

- 新增模块 `sched`、`futex`，并在分发函数 `syscall()` 里接入：
  - `SYSCALL_FUTEX: 98` → `futex::syscall_futex(a0..a5)`
  - `sched_*` 一组：`118~123`、`125~127`

这些 syscall 号与 riscv64 Linux 约定保持一致（尤其 futex=98）。

## 3. futex 实现：最小 WAIT/WAKE（含 bitset 变体）

实现位于 [os/src/syscall/futex.rs](../../os/src/syscall/futex.rs)。该版本的设计目标是“够用就行”，主要用于让 pthread 同步原语能跑（mutex/cond/join 等）。

### 3.1 数据结构

- 全局队列表：`FUTEX_QUEUES: BTreeMap<FutexKey, VecDeque<Arc<TaskControlBlock>>>`
- `FutexKey = (pid, uaddr)`
  - 注意：这里按“进程 pid + 用户地址 uaddr”做 key，因此天然是**进程内 futex**。这足够覆盖 pthread（线程共享地址空间）场景，但不支持跨进程共享 futex。

### 3.2 支持的操作码

- `FUTEX_WAIT`(0)、`FUTEX_WAIT_BITSET`(9)
- `FUTEX_WAKE`(1)、`FUTEX_WAKE_BITSET`(10)
- 其他 op：返回 `-ENOSYS`（`-38`）

额外说明：

- `FUTEX_PRIVATE_FLAG` 会被解析但实际未影响行为（实现里 `_private` 未使用）。
- `_timeout/_uaddr2/_val3` 在当前实现中被忽略。

### 3.3 WAIT 语义

`syscall_futex(uaddr, op, val, ...)` 在 WAIT 类操作中：

1. `uaddr == 0` → `-EINVAL`（`-22`）
2. 读取用户态 `*uaddr`（通过 `translated_mutref(token, uaddr)`）
3. 若 `*uaddr != val` → `-EAGAIN`（`-11`）
4. 否则：将当前任务 TCB 入队到 `(pid,uaddr)` 的队列，并 `block_current_and_run_next()` 阻塞当前任务
5. 被唤醒后返回 0

这对应 Linux futex 的经典用法：用户态先做原子比较交换，若冲突则 futex WAIT；内核再次验证值不匹配就立即返回 `EAGAIN`，避免丢信号。

### 3.4 WAKE 语义

`futex_wake(pid,uaddr,nr)`：

- 依次从该 key 的队列 `pop_front`，调用 `wakeup_task()`，最多唤醒 `nr` 个
- 队列空后从 map 移除 key
- 返回实际唤醒数量

此外该函数被导出为 `pub(crate)`，用于线程退出路径的“清 tid 并唤醒 join 等待者”。

## 4. pthread/glibc 线程链路：clone(CLONE_VM) + tid + clear_child_tid

仅有 `sys_futex` 还不够：glibc pthread 还要求内核提供“线程式 clone”与一套 tid 语义。

### 4.1 `clone(CLONE_VM|...)`：创建 linux thread

在 [os/src/syscall/process/fork.rs](../../os/src/syscall/process/fork.rs) 中：

- 识别 `CLONE_VM`：认为这是线程（共享地址空间）而不是进程 fork
- 通过 `TaskControlBlock::new_linux_thread(process)` 创建新线程 TCB（见 4.2）
- 从父线程复制 TrapContext，并设置：
  - 子线程返回值 `a0=0`
  - 若传入 `stack != 0`，则设置 `sp=stack`
  - 若 `CLONE_SETTLS`，设置 `tp=_tls`（用户态 TLS）
  - 更新内核态字段：`kernel_satp/kernel_sp/trap_handler`
- 处理 tid 指针：
  - `CLONE_PARENT_SETTID`：向 `_ptid` 写入 tid
  - `CLONE_CHILD_SETTID`：向 `_ctid` 写入 tid
  - `CLONE_CHILD_CLEARTID`：将 `_ctid` 记录到 `new_inner.clear_child_tid`，供线程退出时清零并 futex 唤醒
- 将线程挂到 `process_inner.tasks[tid]=Some(tcb)`，并 `add_task(new_task)` 进入调度

如果 `CLONE_VM` 不存在，则走原有“fork-like clone（进程）”路径。

### 4.2 “线程栈由用户态 mmap”：只分配 trap_cx

在 [os/src/task/id.rs](../../os/src/task/id.rs) 中新增：

- `TaskUserRes` 增加 `owns_ustack: bool`
- 新增 `TaskUserRes::new_trap_cx_only(process)`：
  - 为线程分配 tid
  - **只映射 TrapContext 页**（不分配内核管理的用户栈）
  - 适配 glibc：pthread 的线程栈通常由用户态通过 `mmap` 分配
- `alloc_user_res()`/`Drop` 时根据 `owns_ustack` 决定是否管理用户栈区域

`TaskControlBlock::new_linux_thread()`（见 [os/src/task/task_block.rs](../../os/src/task/task_block.rs)）会使用 `new_trap_cx_only()`，因此这类线程默认不带内核分配的 ustack。

### 4.3 `set_tid_address/gettid`：以 tid 为核心（非 pid）

在 [os/src/syscall/misc/other.rs](../../os/src/syscall/misc/other.rs) 中：

- `syscall_set_tid_address(tidptr)`：
  - 若 `tidptr != 0`，将当前线程的 `clear_child_tid = Some(tidptr)`
  - 返回当前线程的 `tid`
- `syscall_gettid_linux()`：返回当前线程的 `tid`

这与 pthread 相关：glibc 会用 `set_tid_address` 注册一个用户地址，线程退出时内核需将其清零，并在该地址上 futex 唤醒等待者。

### 4.4 线程退出：清 \*ctid 并 futex_wake

在 [os/src/task/processor.rs](../../os/src/task/processor.rs) 的 `exit_current_and_run_next()` 中加入：

- 取出当前线程 `clear_child_tid`
- 在线程所属进程的地址空间里将 `*ctid = 0`
- 调用 `crate::syscall::futex::futex_wake(process_pid, ctid, 1)` 唤醒一个等待者

这是 pthread join/cleanup 的关键语义：典型实现会在 join 时对 `tidptr` 做 futex WAIT，线程退出后由内核 wake。

## 5. 信号与阻塞：投递信号时唤醒所有线程

futex wait 在 Linux 上通常是“可被信号打断”的（实际 errno 语义更细致）。该提交至少确保：当进程收到信号时，不会让所有线程永久睡死。

在 [os/src/task/signal.rs](../../os/src/task/signal.rs) 中：

- `kill(pid, signum)`：
  - pid 不存在返回 `-ESRCH`（`-3`）
  - `signum==0` 作为探测直接返回 0
  - 参数合法性检查（越界返回 `-EINVAL`）
  - 将 signal flag 写入 PCB，并唤醒该进程的所有线程 `wakeup_task(t)`
  - 对 `SIGINT(2)`/`SIGKILL(9)` 还会向子进程递归传播（类似“整棵进程树 kill”）
- `kill_current(signum)` 同理：设置自身进程信号并唤醒所有线程

同时在 [os/src/trap/mod.rs](../../os/src/trap/mod.rs) 中，把 `check_if_current_signals_error()` 从注释变为实际执行：

- 若检测到“当前信号需要退出”，打印信息并 `exit_current_and_run_next(errno)`

## 6. sched 系列 syscall：为 rt-tests 提供最小可用接口

虽然 commit 名叫 futex，但同一提交还新增了大量 `sched_*` syscall（[os/src/syscall/sched.rs](../../os/src/syscall/sched.rs)），并在 PCB 里加入 `sched_policy/sched_priority` 字段（[os/src/task/process_block.rs](../../os/src/task/process_block.rs)）。

实现特点：

- `sched_get/set*` 多数只是读写 PCB 字段，或“best-effort 接受但不真正生效”（如 setaffinity）
- `sched_getaffinity` 会返回一个覆盖 `MAX_HARTS` 的全 1 mask
- `sched_rr_get_interval` 返回 0

这些接口常被 cyclictest/hackbench 这类程序在启动时调用，用于探测/设置调度属性；最小实现能避免它们因为 `ENOSYS` 直接退出。

## 7. 当前实现的限制（与 Linux 的差距）

该提交的 futex/线程模型是“面向通过测试的最小子集”，与完整 Linux futex 相比差距明显：

- futex key 使用 `(pid,uaddr)`：不支持跨进程共享 futex（也未区分 shared/private 的正确语义）
- `timeout/bitset/uaddr2/val3` 被忽略：`WAIT_BITSET/WAKE_BITSET` 等同于普通 WAIT/WAKE
- 未实现：REQUEUE、CMP_REQUEUE、PI futex、robust list 的真实语义等
- 信号中断 futex wait 的 errno 语义未完整对齐（当前更偏向“唤醒 + trap 侧退出/继续”）

不过对 glibc pthread 的“基本可跑”来说，这个组合（`clone(CLONE_VM)` + `gettid/set_tid_address` + `exit 清 tid 并 futex_wake` + `FUTEX_WAIT/WAKE`）是关键里程碑。
