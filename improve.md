# 1. 添加了 COW 页

## COW:

查看 forK:

```rs
  }
    /// Fork a user address space using copy-on-write for user pages.
    ///
    /// - User pages (PTE.U) that were writable are remapped read-only and tagged with `PTEFlags::COW`
    ///   in both parent and child.
    /// - Kernel-only pages (e.g., TrapContext, no PTE.U) are copied eagerly.
    pub fn from_existed_user_cow(user_space: &mut MemorySet) -> MemorySet {
```

### 基本逻辑:

对于恒等映射. 不需要 做特别处理. 直接 映射过去就是了 .

对于 非 User 页面 映射

其余的 COW

缺页逻辑的时候才会进行 页面分配

# 2.多核
