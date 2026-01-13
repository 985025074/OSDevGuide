# 这里记载整个的构建过程

# 首先要编译出 risc-V?

add .cargo :

```
[build]
target = "riscv64gc-unknown-none-elf"

[target.riscv64gc-unknown-none-elf]
rustflags = [
    "-Clink-arg=-Tsrc/linker.ld", "-Cforce-frame-pointers=yes"
]


```

# OK,so 我想看看这个东西长什么养

可以查看 temp 下的 first_produce .
Program Header 和 section 的区别
Program Header 程序 Loader 的时候用.
section 则是连接的时候合并用.

# ~~只留下 text 段的命令~~ 应该是 能够被直接顺序执行的文件的产生命令:

rust-objcopy -O binary（等同 objcopy -O binary）会：

把 ELF 中所有 Program Header 类型为 LOAD 的段，按虚拟地址顺序打包成裸二进制文件

顺序是按照段的虚拟地址填充空洞（padding 0）

.text、.rodata、.data、.bss 都会包含在 binary 中吗？
`rust-objcopy --strip-all  target/riscv64gc-unknown-none-elf/debug/os -O binary temp.bin`

# sbi call 不会进入内核的 trap_heandler

fn sbi_call(...) {
unsafe {
asm!(
"ecall",
in("a7") which,
in("a0") arg0,
in("a1") arg1,
...
);
}
}
内核本身在 S-mode 执行 sbicall。

你可能认为它会进入 S-mode trap handler，但 实际触发的 trap 是 “Supervisor Environment Call”。

S-mode 的 ecall 并不会进入内核 trap handler，而是 会被 OpenSBI 固件捕获。

# 解决 rust-analyzer 报错: no test 的问题:

https://github.com/rust-lang/rust-analyzer/issues/3801

# 添加 gdb 支持:

使用方法 make debug 然后另外开一个终端,再来 make client...

# 经过调试 发现 我们第一阶段程序不能运行的原因是没有展区:

0x80201016 <rust_main+22>: ld ra,8(sp)
(gdb) si
0x0000000080201002 in rust_main ()
1: x/10i $pc
=> 0x80201002 <rust_main+2>: sd ra,8(sp)
0x80201004 <rust_main+4>: sd s0,0(sp)
0x80201006 <rust_main+6>: addi s0,sp,16
0x80201008 <rust_main+8>: li a0,72
0x8020100c <rust_main+12>: li a7,1
0x8020100e <rust_main+14>: li a1,0
0x80201010 <rust_main+16>: li a2,0
0x80201012 <rust_main+18>: ecall
0x80201016 <rust_main+22>: ld ra,8(sp)
0x80201018 <rust_main+24>: ld s0,0(sp)
(gdb) si
0x0000000000000000 in ?? ()
1: x/10i $pc
=> 0x0: <error: Cannot access memory at address 0x0>

# Ok 我们成功实现了.

# 接下来的任务 实现 println. panic.

由于这部分不是重点,我直接复制了.

# 接下来的任务 实现 任务处理. 也就是 user 程序.

注意 weak.

```
pub mod console;
mod lang_items;
pub mod syscall;
#[unsafe(no_mangle)]
fn _start() {
    syscall::exit(main());
}
#[linkage = "weak"]
#[unsafe(no_mangle)]
fn main() -> usize {
    println!("Hello, world!");
    return 0;
}


```

# swicth 的实现:

首先要知道硬件帮我们做了什么:
sstatus 的 SPP 字段会被修改为 CPU 当前的特权级（U/S）。

sepc 会被修改为被中断或触发异常的指令的地址。如 CPU 执行 ecall 指令会触发异常，则 sepc 会被设置为 ecall 指令的地址。

scause/stval 分别会被修改成这次 Trap 的原因以及相关的附加信息。

CPU 会跳转到 stvec 所设置的 Trap 处理入口地址，并将当前特权级设置为 S ，然后从 Trap 处理入口地址处开始执行。
而当 CPU 完成 Trap 处理准备返回的时候，需要通过一条 S 特权级的特权指令 sret 来完成，这一条指令具体完成以下功能：

CPU 会将当前的特权级按照 sstatus 的 SPP 字段设置为 U 或者 S ；

CPU 会跳转到 sepc 寄存器指向的那条指令，然后继续执行。

这些基本上都是硬件不得不完成的事情，还有一些剩下的收尾工作可以都交给软件，让操作系统能有更大的灵活性

注意 swicth 开头有一行启用了红语法

# 64 位系统注意事!!!

# BUG: Num_of_apps 莫名其妙变成 0 :

    unsafe extern "C" {
        fn num_user_apps() -> usize;
        fn sbss();
        fn ebss();

    }
    unsafe {
        let num_of_apps = *(num_user_apps as *const i64);
        println!(
            "Number of user apps: {}, from adress {}",
            num_of_apps, num_user_apps as usize
        );
    }
    经过测试 发现lazy_static可能修改了这里的内存,或许是重复? 暂时不清楚原理如何...
    改成rodata段就没问题. 但是user数据如果放rodata 有有问题...

# BUG: MAX_TASKS 竟然影响 任务名是否能够加载???

发现.代码必须放到 .data 然后其余的放到.rodata 不然会出问题

.rodata 和 .data 的影响???

# 诡异 bug: trap context 忘记入栈了 出现了一个非常奇怪的错误...

~~指令访问异常?? r13 ==0 ?~~
最后解决.... 竟然是因为代码长度不够,真是曹了.

# 又出现问题. 又来一个访问异常...

这次是某个 return 返回 0 的时候出错了..

# Refcell 的解决.

具体我也不知道哪里出现了 borrow 的问题.这是个静态变量.实在太难发现了.我就一次性所有的都进行了 drop.结果竟然可以了.而且解决了指令访问异常的问题...

# tag 3 主要任务:

首先: 完成任务上下文切换的机制.

大概包括:上下文存储.以及任务切换的接口.

核心是 swicth 函数.

然后是分时.

以及程序摆放. 这里要调整 link...

根据设计,应该有一步: 在初始化中 loadapps 就推入 trapcontext

# trap context 是怎么切换的??

并没有单独切换 跟着 kernel sp 切换的
trap context 在初始化的时候就已经存在了
任务切换的时候由于 sp 的交换,自然的,trap 也被切换了.

# 用户程序出错,, 可以考虑是不是没更新 导致了没有 build...可以尝试 make clean 一下

# is_vallid

V(Valid)：仅当位 V 为 1 时，页表项才是合法的；

R(Read)/W(Write)/X(eXecute)：分别控制索引到这个页表项的对应虚拟页面是否允许读/写/执行；

U(User)：控制索引到这个页表项的对应虚拟页面是否在 CPU 处于 U 特权级的情况下是否被允许访问；

G：暂且不理会；

A(Accessed)：处理器记录自从页表项上的这一位被清零之后，页表项的对应虚拟页面是否被访问过；

D(Dirty)：处理器记录自从页表项上的这一位被清零之后，页表项的对应虚拟页面是否被修改过。

# 这里是真麻烦...

我发现第二级别页表 竟然是无效的..我操你的....
报道是 trap7
访问内存无效..

## 决定借用原先代码.

问题 : 空页似乎是被映射的..只是没有数据 TODO

## todo : sfence.vma 暂时不清楚做了什么,但是执行之后会显示 我的断电都无效了.

# 添加了多文件调试支持:

具体方法: 新增一个 tests cargo projet 然后创建 tests/tests/ 在下面添加测试,同时在 cargo.toml 中增加对于 os 的依赖项目.

对了 在此时需要在 os 的内部添加一个 lib.rs
“main.rs 和 lib.rs 是两个世界。”

“crate:: 总是指当前世界。”

“binary 想访问 library，要走包名。”
貌似是两个目标,crate 不同...

# 问题 idle task 无法被切换回到正确位置

1. 修改写法 从 raw 指针(size ) 变成\*mut

2. 不要尝试强转 数组& 到普通指针 你无法想象内部的情况...

3. extern ! ??一个感叹号的问题????????????????????????????????????

4. todo !!

# 一直不动

可能是进入了 a 原状态.就是刚刚加载 e 完成 rustsbi 的那个情况..

# todo pid 回收

#

# 在 os 的 pipe mutex 加上 spin 发现 无法执行二进制文件 o

warning: `os` (bin "os") generated 32 warnings (28 duplicates) (run `cargo fix --bin "os"` to apply 3 suggestions)
Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.29s
Running `target/riscv64gc-unknown-none-elf/debug/os`
target/riscv64gc-unknown-none-elf/debug/os: target/riscv64gc-unknown-none-elf/debug/os: 无法执行二进制文件

# condavar:

condvar 要注意唤醒丢失的问题,就是唤醒进程执行过快,会导致被唤醒进程一直卡住,因此需要其余资源来维护.


# todo:
# 清理 日志 使用log
# 添加brk.等系统调用..
# 通过basic测试...
# shell 完备
# busy box ? ...


