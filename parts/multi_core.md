# 多核设计

基于 multi_core_v3branch 编写.

# Future target(TODO):

更好的调度策略

## 基本修改

对于 lazy_static 本身采用 refcell 的 ,全面改成 mutex 来保障安全性.
**refcell 是 runtime check 的,大概率会出现 alreay borrowed panic**

## 基本任务结构

首先还是遵循了 mutable 和 nt mutable 分离的原则.
为了提供多核支持提供了 cpu_id,on_cpu,等结构

```rs
pub struct TaskControlBlock {
    // immutable
    // 对于所有的线程,共享一个父进程
    pub process: Weak<ProcessControlBlock>,
    pub kstack: KernelStack,
    /// Preferred CPU (hart) to run this task on.
    ///
    /// This is used by the scheduler to decide which per-hart run queue the task should be
    /// enqueued into when it becomes runnable.
    pub cpu_id: AtomicUsize,
    /// The hart id currently running this task, or OFF_CPU if none.
    pub on_cpu: AtomicUsize,
    /// Set by a waker if it tried to wake while the task was still on_cpu.
    pub wakeup_pending: AtomicBool,
    /// Whether this task is currently enqueued in the global ready queue.
    pub in_ready_queue: AtomicBool,
    // mutable
    inner: Mutex<TaskControlBlockInner>,
}

pub struct TaskControlBlockInner {
    // 对于所有的线程,共享一个父进程
    pub res: Option<TaskUserRes>,
    pub trap_cx_ppn: PhysPageNum,
    pub task_cx: TaskContext,
    pub task_status: TaskStatus,
    pub exit_code: Option<i32>,
    pub join_waiters: VecDeque<Arc<TaskControlBlock>>,
}

```

# Cpu hart 数据结构

```rs
pub struct Processor {
    now_task_block: Option<Arc<TaskControlBlock>>,
    idle_task_context: TaskContext,
    /// A task that should be enqueued after we have switched back to idle.
    ///
    /// This avoids a race where we would put the current task back into the global
    /// ready queue *before* context switching away, letting another hart run the
    /// same task concurrently on the same kernel stack.
    pending_ready: Option<Arc<TaskControlBlock>>,
    /// A task that is transitioning into Blocked state; finalized after switching to idle.
    pending_blocked: Option<Arc<TaskControlBlock>>,
}
```

pending_ready? pending_block?

# hart 调度过程

相关代码:

```rs
#![no_std]
#![no_main]
#![feature(alloc_error_handler)]
#![feature(str_from_raw_parts)]
#![allow(unreachable_code)]
use core::{arch::global_asm, panic};
extern crate alloc;
use crate::fs::list_apps;
use core::sync::atomic::{AtomicBool, Ordering};
mod config;
mod console;
mod debug_config;
mod drivers;
mod fs;
mod lang_items;
mod log;
mod mm;
mod sbi;
mod syscall;
mod task;
mod time;
mod trap;
mod utils;

global_asm!(include_str!("entry.asm"));
global_asm!(include_str!("link_app.asm"));

// Keep this flag in .data so clearing .bss doesn't reset it after the
// bootstrap hart marks initialization as done.
#[unsafe(link_section = ".data")]
static BOOT_HART_INITED: AtomicBool = AtomicBool::new(false);
// Secondary harts must not touch .bss-backed globals before the boot hart clears .bss.
#[unsafe(link_section = ".data")]
static BOOT_BSS_CLEARED: AtomicBool = AtomicBool::new(false);
// Secondary harts must not enter the scheduler before the boot hart finishes global init.
#[unsafe(link_section = ".data")]
static BOOT_GLOBAL_INIT_DONE: AtomicBool = AtomicBool::new(false);

fn clear_bss() {
    unsafe extern "C" {
        safe fn sbss();
        safe fn ebss();
    }
    unsafe {
        let bss_start = sbss as usize;
        let bss_end = ebss as usize;
        let bss_size = bss_end - bss_start;
        core::ptr::write_bytes(bss_start as *mut u8, 0, bss_size);
    }
}

fn start_other_harts(boot_hart_id: usize, dtb_pa: usize) {
    for hart_id in 0..config::MAX_HARTS {
        if hart_id == boot_hart_id {
            continue;
        }
        // Ignore failures for now; OpenSBI returns non-zero on error.
        let _ = sbi::hart_start(hart_id, config::KERNEL_ENTRY_PA, dtb_pa);
    }
}

fn secondary_main(hart_id: usize, dtb_pa: usize) -> ! {
    // Wait until the boot hart clears .bss and completes global initialization.
    while !BOOT_BSS_CLEARED.load(Ordering::SeqCst) {
        core::hint::spin_loop();
    }
    while !BOOT_GLOBAL_INIT_DONE.load(Ordering::SeqCst) {
        core::hint::spin_loop();
    }
    // Activate the page table built by the boot hart so we can safely run in S-mode.
    mm::activate_kernel_space();
    trap::init_trap();
    trap::trap::enable_timer_interrupt();
    time::set_next_trigger();
    println!(
        "[kernel] secondary hart {} online (dtb_pa={:#x}), entering scheduler...",
        hart_id, dtb_pa
    );
    task::task_start_secondary();
}

#[unsafe(no_mangle)]
fn rust_main(hart_id: usize, dtb_pa: usize) -> ! {
    // Avoid timer interrupts preempting early-boot code that may hold spin::Mutex locks
    // (e.g., heap allocator, ext4, ready queue). We'll re-enable interrupts in the
    // scheduler/idle loop and on sret back to user.
    unsafe { riscv::register::sstatus::clear_sie() };

    unsafe extern "C" {
        fn num_user_apps();
    }
    if BOOT_HART_INITED
        .compare_exchange(false, true, Ordering::SeqCst, Ordering::SeqCst)
        .is_ok()
    {
        clear_bss();
        BOOT_BSS_CLEARED.store(true, Ordering::SeqCst);
        let num_of_apps = unsafe { *(num_user_apps as *const i64) };
        println!(
            "Number of user apps: {}, from adress {}",
            num_of_apps, num_user_apps as usize
        );
        println!(
            "[kernel] bootstrap hart {} starting with dtb @ {:#x}",
            hart_id, dtb_pa
        );
        mm::init();
        mm::remap_test();
        log::init();
        if debug_config::DEBUG_LOG_TEST {
            log::test();
        }
        println!("[kernel] memory management initialized.");
        BOOT_GLOBAL_INIT_DONE.store(true, Ordering::SeqCst);
        start_other_harts(hart_id, dtb_pa);
        trap::init_trap();
        trap::trap::enable_timer_interrupt();
        time::set_next_trigger();
        list_apps();
        task::task_start();
    } else {
        secondary_main(hart_id, dtb_pa);
    }
    panic!("shouldn't be here");
}


```

首先,所有 CPU 的都进入 rust main.根据谁先抢夺到原子变量,谁来进入 hart 来进行 boot 或者 second_hart

boot 负责进行内存初始化 trap 初始化 ,以及 task_start,task_start 包含了我们的 0 号进程的初始化.

```rs
pub fn task_init() {
    // Force INITPROC initialization in release builds.
    lazy_static::initialize(&INITPROC);
    crate::println!("[kernel] INITPROC initialized and enqueued");
}
```

## idle_task

此部分是多核的核心,首先是确保中断工作正常.防止工作中途被修改等意外情况(这部分代码是否必要? 进行测试研究 todo)
然后便开始

```rs

pub fn idle_task() {
    #[allow(dead_code)]
    static EMPTY_SPINS: core::sync::atomic::AtomicUsize =
        core::sync::atomic::AtomicUsize::new(0);
    loop {
        // Ensure kernel-mode traps use the kernel handler (stvec points to alltraps_k)
        init_trap();
        // Disable interrupts while accessing TASK_MANAGER to prevent
        // timer interrupt from calling check_timer -> wakeup_task -> add_task
        // while we hold the TASK_MANAGER lock in fetch_task
        unsafe {
            riscv::register::sstatus::clear_sie();
        }

        // Finalize a task that just switched away and wanted to become Blocked.
        if let Some(task) = local_processor().lock().take_pending_blocked() {
            // The task is now off CPU on this hart.
            task.clear_on_cpu();
            if task.wakeup_pending.swap(false, core::sync::atomic::Ordering::AcqRel) {
                let mut inner = task.borrow_mut();
                inner.task_status = TaskStatus::Ready;
                drop(inner);
                add_task(task);
            }
        }

        // Enqueue a task that was marked runnable by this hart *before* it switched
        // to idle. This makes the task visible to other harts only after we are
        // no longer running on its kernel stack.
        if let Some(task) = local_processor().lock().take_pending_ready() {
            task.clear_on_cpu();
            task.wakeup_pending
                .store(false, core::sync::atomic::Ordering::Release);
            add_task(task);
        }

        if let Some(task) = fetch_task() {
            if crate::debug_config::DEBUG_WATCHDOG {
                EMPTY_SPINS.store(0, core::sync::atomic::Ordering::Relaxed);
            }
            let mut processor = local_processor().lock();
            let idle_task_cx_ptr = processor.get_idle_task_ptr();
            // access coming task TCB exclusively
            let mut task_inner = task.borrow_mut();
            let next_task_cx_ptr = &task_inner.task_cx as *const TaskContext;
            if DEBUG_SCHED {
                let tid = task_inner.res.as_ref().map(|r| r.tid).unwrap_or(usize::MAX);
                log::debug!(
                    "[idle] hart={} switch to tid={} ra={:#x} sp={:#x}",
                    hart_id(),
                    tid,
                    task_inner.task_cx.ra,
                    task_inner.task_cx.sp
                );
            }
            task.mark_on_cpu(hart_id());
            task_inner.task_status = TaskStatus::Running;

            drop(task_inner);
            // release coming task TCB manually
            processor.now_task_block = Some(task);
            // release processor manually
            drop(processor);

            // Keep interrupts disabled while resuming kernel context; sret will enable them for user.
            unsafe {
                switch::switch(
                    idle_task_cx_ptr as *const usize,
                    next_task_cx_ptr as *const usize,
                );
            }
            if DEBUG_SCHED {
                log::debug!("[idle] hart={} switch returned to idle", hart_id());
            }
        } else {
            if crate::debug_config::DEBUG_WATCHDOG {
                let c = EMPTY_SPINS.fetch_add(1, core::sync::atomic::Ordering::Relaxed);
                if c == 1_000 {
                    crate::task::manager::dump_system_state();
                }
            }
            // crate::println!("[idle] No tasks, entering wfi...");
            // No ready tasks - enable interrupts and wait
            // Use wfi to save power while waiting for timer interrupt
            // Timer interrupt will call check_timer() to wake up sleeping tasks
            //
            // IMPORTANT: We must loop back to check fetch_task() after wfi returns
            // because the interrupt handler may have woken up a task
            unsafe {
                riscv::register::sstatus::set_sie();
                core::arch::asm!("wfi");
            }
            // crate::println!("[idle] Woke up from wfi");
            // Loop back immediately to check for newly ready tasks
        }
    }
}

```

首先先不管 pending 的设计,先来看 一个基本的抢夺任务 和 切换任务的过程
此过程与基本的单核 rCore 是大同小异的.获取当前 Cpu Idle task context 然后 switch.

# 任务生命周期 (Task Manager 的相关设计)

首先 为了减少多核竞争导致的 race case,这里的 task manager 升级成了每个 hart 一个.也就是一个二维数组.

接口除了多了一个 hart_id 基本上不变.

- add_task()
  添加任务 这里模仿了 Linux 再添加任务而时候使用一个 send_ipi 来对远端进行唤醒.经过设计测试,在多线程测试中,这样可以有效提升性能
- wakeup_task()
  wakeuptask 需要处理一个 race case,需要确保任务已经离开其他的 CPU

## Pending 的设计

```rs
pub struct Processor {
    now_task_block: Option<Arc<TaskControlBlock>>,
    idle_task_context: TaskContext,
    /// A task that should be enqueued after we have switched back to idle.
    ///
    /// This avoids a race where we would put the current task back into the global
    /// ready queue *before* context switching away, letting another hart run the
    /// same task concurrently on the same kernel stack.
    pending_ready: Option<Arc<TaskControlBlock>>,
    /// A task that is transitioning into Blocked state; finalized after switching to idle.
    pending_blocked: Option<Arc<TaskControlBlock>>,
}
```

正如上面所提到的,ready 与 block 的任务的移交并不是原子的.因此我们暂时将这些任务放在 cpu 上面来避免 race condition.与之对应的,拿到 idle_task 的处理器应当马上处理掉对应任务在 cpu 的状态.

## add_task 和 fetch_task 的设计

为了适应多核 做了一些改进

- 阻止中断.(linux 的 task_rq_lock 采用了 spinlock,会进行 local_irq_disabled)
- 加入到的对应 hart_id 是这样来的,默认是 cur 如果没有(少核) 那么就去第一个
- **ipi** ipi 是一种跨处理器中断 用来一对一的唤醒,避免不必要的唤醒来浪费资源.

```rs

pub fn add_task(task: Arc<TaskControlBlock>) {
    // Protect the ready queue from timer interrupt re-entrancy, but restore the previous SIE state.
    let prev_sie = riscv::register::sstatus::read().sie();
    unsafe { riscv::register::sstatus::clear_sie() };
    let desired = task.get_cpu_id() % MAX_HARTS;
    let mask = online_hart_mask();
    let cur = crate::task::processor::hart_id() % MAX_HARTS;
    let hart_id = if (mask & (1usize << desired)) != 0 {
        desired
    } else if (mask & (1usize << cur)) != 0 {
        // If the preferred hart is offline, run it where we are.
        task.set_cpu_id(cur);
        cur
    } else {
        // Last resort: pick any online hart.
        let picked = pick_online_hart(0);
        task.set_cpu_id(picked);
        picked
    };
    TASK_MANAGER.lock().add(task, hart_id);
    // Linux-style: if we queued to a remote hart, kick it out of `wfi` via IPI.
    if cur < MAX_HARTS && cur != hart_id {
        crate::sbi::send_ipi(hart_id);
    }
    if prev_sie {
        unsafe { riscv::register::sstatus::set_sie() };
    }
}


```

fetch_task 略.与之对应的.从自己的队列里取出就执行.

## global_queue

注意到对于每个任务还有一个 in_ready_queue 的设计.
这个是为了避免一个任务被多次添加.(比如 多个线程 同时尝试)
任务 T1 当前处于 Blocked 状态。
核心 A 和核心 B 几乎同时尝试唤醒任务 T1：
核心 A 调用 wakeup_task(T1)，检查到任务状态为 Blocked，准备调用 add_task(T1)。
核心 B 也调用 wakeup_task(T1)，同样检查到任务状态为 Blocked，准备调用 add_task(T1)。
**防患于未然.总是好的.**
