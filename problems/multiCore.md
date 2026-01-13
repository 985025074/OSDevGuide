# 多核开发日志

## 参考了 rocketos.

## 看这个项目首先要 配置下 rust-analyzer,否则 跳转不好

```json
{
  "rust-analyzer.cargo.target": "riscv64gc-unknown-none-elf",
  "rust-analyzer.cargo.features": ["smp"]
}
```

## 启用多核

```rs
    time::set_next_trigger();
    list_apps();
    task::task_init();
    KERNEL_READY.store(true, Ordering::Release);
    start_other_harts(hart_id);
    task::task_start();

```

start_other harts
通过 sbi 启用?

