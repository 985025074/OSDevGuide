# 内存空间设计

基于当前主线实现整理（mm 相关代码主要在 `os/src/mm/*`，syscall 入口在 `os/src/syscall/memory.rs`）。

> 目标：在 SV39 虚拟内存下，完成「内核地址空间 + 用户进程地址空间」的构建/切换/扩展/回收，并在跑 glibc / busybox / ltp 等场景时尽可能稳健。

## COW 与 Lazy（按需分配）简介

这套内存空间设计的核心优化点主要是两件事：

- **Lazy allocation（懒分配）**：先“占虚拟地址”，不立刻分配物理页；等真正访问时再分配。
- **Copy-on-Write（写时复制，COW）**：`fork()` 时先共享物理页，只有发生写入时才复制。

两者共同目标：在 busybox / glibc / libc-bench / ltp 这类工作负载里，减少峰值内存占用，避免大量 `fork()`/`mmap()` 带来的 OOM 或性能抖动。

### Lazy allocation：先映射“区间”，缺页再分配

实现层面：用 `MapType::Lazy` 表示一段“允许存在但尚未分配物理页”的用户 VA 区间。

- 建立阶段：
  - 例如在 `MemorySet::from_elf()` 里，会为 heap 预留一个 `MapType::Lazy` 的 area（配合 `brk/sbrk`）。
  - `mmap` 的匿名映射也倾向使用 lazy（尤其是大块匿名映射）。
- 触发阶段：当用户态第一次访问该区间（读/写/取指）会触发 page fault。
- 修复阶段：trap 里调用 `MemorySet::resolve_lazy_fault(fault_va, access)`：
  - 在匹配的 `MapType::Lazy` area 内，检查权限 `map_perm` 是否包含本次访问类型（R/W/X）。
  - 分配一个物理页帧，建立 PTE 映射，并 `sfence.vma` 刷新 TLB。

注意：Lazy 不是“无限制分配”。真正分配时仍可能因为物理内存不足失败（返回 `false`），上层通常需要给出 `ENOMEM` 或直接终止非法/不可恢复访问。

### COW：fork 先共享，写入再复制

实现层面：用页表项上的软件标记 `PTEFlags::COW`（Sv39 PTE 的 RSW 位）标记“这是共享页，当前不可写”。

- 建立阶段：`MemorySet::from_existed_user_cow(user_space: &mut MemorySet)`
  - 对“用户态 + 可写 + 非共享（非 `SHARED`）”的页：
    - 去掉 `W`
    - 加上 `COW`
    - 父子地址空间映射同一物理页
  - 对 TrapContext 等内核私有页（非 `PTE.U`）：直接分配新页并拷贝。
- 触发阶段：当某一方尝试写入 COW 页，会触发 store page fault。
- 修复阶段：trap 里调用 `MemorySet::resolve_cow_fault(fault_va)`：
  - 分配新页，拷贝旧页内容
  - 将该 VA remap 到新页，并恢复 `W`（清除 `COW`）
  - 更新 area 的 `data_frames`，让旧页的引用计数能正确递减

为了避免“共享页被过早释放/双重释放”，`FrameTracker` 引入了 `FRAME_REFCOUNTS`：同一物理页被多个地址空间引用时，必须引用计数归零才真正 `frame_dealloc`。

### COW 与 Lazy 的配合点（一个容易忽略的细节）

在实际 workload 中，缺页不止发生在用户态“正常执行”时：

- 内核在做用户态拷贝（如 `copy_to_user`）时也可能触发缺页。
- 因此 trap 里除了用户态异常路径，也提供了 `try_handle_kernel_page_fault()`：当内核态因访问用户页产生 page fault 时，仍会尝试用 `resolve_cow_fault/resolve_lazy_fault` 修复，而不是直接 panic。

## 整体流程（从启动到运行时）

### 1) 启动阶段：建立内核地址空间

入口：`os/src/mm/mod.rs::init()`

流程大致如下：

1. 初始化内核堆（用于后续分配页表、inode buffer、加载大 ELF 等）。
2. 初始化物理页帧分配器（frame allocator）。
3. 构建并激活 `KERNEL_SPACE`（内核页表），写入 satp。

关键点：

- `os/src/mm/memory_set.rs::MemorySet::new_kernel()` 会把内核各段（`.text/.rodata/.data/.bss`）、物理内存区、MMIO 都做恒等映射（Identical mapping）。
- `TRAMPOLINE` 页只给内核使用，另外单独映射 `SIGRETURN_TRAMPOLINE` 给用户态用于 `sigreturn`。
- 多核场景下，为避免二级核去借用/锁 `KERNEL_SPACE`（以及早期初始化时的借用冲突），会缓存 `KERNEL_SATP`：`os/src/mm/mod.rs::activate_kernel_space()`。

### 2) 进程创建：从 ELF 构建用户地址空间

入口：`os/src/mm/memory_set.rs::MemorySet::from_elf()`

典型布局：

- 映射 trap trampoline：
  - `TRAMPOLINE`：内核可执行（R|X），用户不可达。
  - `SIGRETURN_TRAMPOLINE`：用户可执行（R|X|U）。
- 映射 ELF program headers（PTE.U），按 `R/W/X` 权限映射。
- 用户栈（默认较大，适配 busybox/glibc）：`USER_STACK_SIZE`。
  - 带一个 guard page（空洞）防止越界。
- “用户堆/heap”：使用 `MapType::Lazy` 做一个 **懒分配区域**，配合 `brk/sbrk` 扩展。
- `TRAP_CONTEXT`：TrapContext 存放区（内核态可读写），每个线程/任务私有。

额外兼容点：

- ET_DYN（PIE/共享对象）会加一个 `load_bias`，避免把低地址（尤其是 NULL page）意外映射掉，也兼容动态链接器的映射习惯。

### 3) 运行时：brk / mmap / fork 等改变内存布局

- `brk/sbrk`：`os/src/syscall/memory.rs::syscall_brk()`

  - 通过 `MemorySet::append_to/shrink_to` 改变 heap 的末端。
  - 这里的 heap area 被设计成 `MapType::Lazy`，因此只调整 vpn_range，不会一次性分配大量物理页。

- `mmap/munmap`：`os/src/syscall/memory.rs::syscall_mmap()`

  - `MAP_FIXED`：实现 “覆盖式映射”，会先 unmap 目标范围内旧映射（并维护 `mmap_areas` 记账），同时拒绝覆盖内核页（例如 TrapContext/trampoline）。
  - 匿名大映射：倾向使用 `MapType::Lazy`，避免 glibc / libc-bench 出现大块匿名映射导致 OOM。
  - `MAP_SHARED`：会走共享页帧路径（见下面“共享内存”）。

- `fork`：使用 Copy-on-Write（COW）共享用户页：
  - 入口：`os/src/mm/memory_set.rs::MemorySet::from_existed_user_cow()`
  - 对于 **用户态且可写** 的页：父子双方都改成只读，并打 `PTEFlags::COW`。
  - TrapContext 等 **内核私有页** 不共享，直接拷贝新页。

### 4) 缺页异常：lazy/COW 的按需修复

入口：`os/src/trap/mod.rs::handle_user_exception()`

- store page fault：优先尝试 `resolve_cow_fault()`，将共享只读页复制为私有可写页。
- load/store/inst page fault：尝试 `resolve_lazy_fault()`，为 Lazy 区域按需分配物理页并建立映射。
- 都无法修复：认为是非法访问，按错误路径退出（或 panic / kill）。

## 核心数据结构与设计取舍

### 地址与页号类型

在 `os/src/mm/address.rs` 中定义：

- `VirtAddr/VirtPageNum`、`PhysAddr/PhysPageNum`
- `VirtPageNum::indexes()` 用于 SV39 三级页表 walk。
- `PhysPageNum::get_pte_array()` / `get_bytes_array()` 用于把物理页当作页表页或数据页访问。

### FrameAllocator + FrameTracker（带引用计数）

在 `os/src/mm/frame_allocator.rs`：

- `StackFrameAllocator` 做最基本的物理页分配/回收。
- `FrameTracker` 采用 RAII：Drop 时回收物理页。
- 为支持 COW / SHARED，一帧物理页可能被多个 `FrameTracker` 引用，因此加了 `FRAME_REFCOUNTS`：
  - clone 增加引用计数
  - drop 引用计数归零才真正 `frame_dealloc`

这个设计的动机非常直接：**避免 COW / shared 场景下的 double-free 或过早回收**。

### PageTable / PTEFlags：使用 RSW 位标记 COW/SHARED

在 `os/src/mm/page_table.rs`：

- `PTEFlags::COW`：软件维护的 COW 标记（使用 Sv39 PTE RSW bit 0）。
- `PTEFlags::SHARED`：软件维护的共享映射标记（RSW bit 1）。

`SHARED` 的存在是为了让 “共享内存映射” 在 `fork()` 时能保持共享语义：共享页不应该被当成普通可写页去打 COW。

### MemorySet / MapArea：以 area 组织一段连续 VA

在 `os/src/mm/memory_set.rs`：

- `MemorySet`：一个完整的地址空间（包含一个 `PageTable` + 多个 `MapArea`）。
- `MapArea`：一段连续虚拟地址区间（vpn_range），并记录：
  - `map_type`：
    - `Identical`：恒等映射（不持有 frame）
    - `Framed`：一页一帧（area 持有 `data_frames`）
    - `Lazy`：不预先分配页，缺页时再分配
  - `map_perm`：R/W/X/U

并且做了一个非常关键的健壮性处理：

- `MapArea::map()` 在分配/映射过程中如果 OOM，会把已经映射的部分 rollback 掉，避免留下一个“半成品地址空间”。

## 关键实现细节（代码路径）

### 内核空间的激活与多核

- 单核：`KERNEL_SPACE.lock().activate()` 即可。
- 多核：`os/src/mm/mod.rs` 增加 `KERNEL_SATP` 缓存，二级核直接 `activate_token(cached)`，避免在不合适时机去锁全局 `KERNEL_SPACE`。

### COW fork 的核心逻辑

入口：`MemorySet::from_existed_user_cow(user_space: &mut MemorySet)`

策略：

1. 遍历父进程所有 `MapArea`。
2. 对 “用户态 + 可写 + 非 SHARED” 的页：
   - 去掉 W
   - 加上 COW
   - 子地址空间映射同一 ppn，`FrameTracker` clone 增引用计数。
3. 对 TrapContext 等非 U 页：直接分配新页并拷贝内容。

### Lazy fault / COW fault 的处理

- `MemorySet::resolve_lazy_fault(fault_va, access)`：

  - 只对 `MapType::Lazy` 的 area 生效
  - 检查权限（access 必须被 area.map_perm 包含）
  - 分配 frame，map，并 `sfence.vma` 刷新 TLB

- `MemorySet::resolve_cow_fault(fault_va)`：
  - 仅当页表项含 COW 才处理
  - 分配新页并拷贝旧页内容
  - remap 成可写（去 COW 加 W）
  - 更新 area 的 `data_frames`，让旧页的引用计数正确递减

### 共享内存：SHARED 映射 + fork 语义保持

入口：`MemorySet::insert_shared_frames_area(...)`

- 把一组指定的物理页帧映射到用户 VA 区间，并在 PTE 上打 `PTEFlags::SHARED`。
- `fork()` 时：`from_existed_user_cow()` 会跳过 `SHARED` 页的 COW 标记步骤，从而保证共享页仍然共享。

在 syscall 层面的触发点：`os/src/syscall/memory.rs::syscall_mmap()`

- `MAP_SHARED` 会走共享 frames 的分配/复用逻辑。
- 对 pseudo shm file（`PseudoShmFile`）可以拿到 “同一份” frames，从而实现跨进程共享。

### MAP_FIXED：覆盖式映射与内核区域保护

`syscall_mmap()` 对 `MAP_FIXED` 做了两件事情：

1. 拒绝覆盖内核私有页（PTE.U 不存在的页）。
2. 先 `unmap_user_range()` 清理目标范围的旧映射，同时更新 `mmap_areas` 记账（split/trim overlaps）。

这个点是为了适配 glibc/ld.so 经常用 `MAP_FIXED` 重建布局的行为。

## 为修复某些 bug 做的工作（偏内存空间）

下面只记录和“地址空间/页表/映射语义”强相关的修复点：

### 1) COW / SHARED 的语义冲突

现象：

- 共享内存（System V / pseudo shm）映射出来的页，在 fork 后被当作普通可写页打上 COW，导致子进程写入后不再共享，甚至出现测试脚本中 “共享内存不一致/abort” 一类问题。

修复：

- 在 PTE 的 RSW 位加入 `PTEFlags::SHARED` 标记。
- fork 过程中遇到 `SHARED` 页时不对它做 COW 降权（保持共享语义）。

涉及文件：

- `os/src/mm/page_table.rs`（SHARED flag）
- `os/src/mm/memory_set.rs`（from_existed_user_cow + insert_shared_frames_area）
- `os/src/syscall/memory.rs`（MAP_SHARED 路径）

### 2) COW/shared 下的物理页过早回收

现象：

- 多个地址空间映射同一物理页（COW 前、或 shared mmap），如果没有引用计数，某一方退出/munmap 时释放物理页会导致另一方出现野指针/数据破坏。

修复：

- `FrameTracker` 引入 `FRAME_REFCOUNTS`，通过 clone/drop 管理真实物理页释放时机。

涉及文件：

- `os/src/mm/frame_allocator.rs`

### 3) OOM 时留下“半映射”的地址空间

现象：

- 大 mmap / 加载大 ELF / 扩展用户栈等触发 OOM 时，如果已经映射了一部分页而中途失败，会导致地址空间处于不一致状态，后续访问随机崩溃。

修复：

- `MapArea::map()`/`append_to()` 在失败时 rollback 已完成的映射。
- 相关 API 增加 `try_*` 版本（例如 `try_insert_framed_area/try_insert_lazy_area`）并向 syscall 返回 `ENOMEM`。

涉及文件：

- `os/src/mm/memory_set.rs`
- `os/src/syscall/memory.rs`

### 4) MAP_FIXED 覆盖语义与内核区域保护

现象：

- glibc/ld.so 使用 `MAP_FIXED` 重建映射，如果内核没有 “替换旧映射” 的语义，可能导致重叠映射、旧映射泄漏，或错误覆盖 TrapContext/trampoline。

修复：

- 增加 `MemorySet::unmap_user_range()` + `syscall_mmap()` 的 MAP_FIXED 分支：先清理，再建立新映射，并拒绝覆盖 kernel-only 页。

涉及文件：

- `os/src/mm/memory_set.rs`
- `os/src/syscall/memory.rs`

### 5) 物理内存大小探测（DTB）

现象：

- 若默认物理内存上限与实际 QEMU/平台不一致，会导致 frame allocator 可用范围错误（过小导致莫名 OOM，过大可能踩到不存在的 RAM）。

修复：

- 从 dtb 解析 memory region，动态设置 `phys_mem_start/end()`。

涉及文件：

- `os/src/mm/dtb.rs`
- `os/src/config.rs`

## TODO

- 更精细的 mmap 区域管理（空洞复用、合并相邻 area 等），减少 VA 碎片。
- file-backed mmap 的页缓存/回写策略目前很简化（能跑通测试优先）。
- 若要做更强健的 uaccess，需要把 `copy_from_user/copy_to_user` 的 fault/retry 路径进一步统一。
