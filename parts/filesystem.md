# 文件系统

目前仅支持 ext4

## 自顶向下 ,文件系统大致结构

文件系统的大部分实现在 ext4-fs 包里.参照 rCore easy-fs 设计.

底层的系统被暴露为一个全局实例:

```rs

lazy_static! {
    /// ext4 filesystem handle (shared by all inodes).
    pub static ref EXT4_FS: Arc<spin::Mutex<Ext4FileSystem>> = {
        Ext4FileSystem::open(BLOCK_DEVICE.clone())
    };

    /// Root inode of the filesystem
    pub static ref ROOT_INODE: Arc<Inode> = {
        Arc::new(Ext4FileSystem::root_inode(&EXT4_FS))
    };

    /// User directory inode (for ext4, apps are in /user)
    pub static ref USER_INODE: Arc<Inode> = {
        ROOT_INODE.find("user").expect("[ext4] /user directory not found!")
    };
}


```

目前暂时 设计了两个点 `/user` 和 `/extra` 分别存放我自己写的测试和额外的编译完成测试文件

### OSInode

所有打开文件都保留为 inode 结构..是底层系统 inode 的包装.
readable writeable 暂时没有正对用户进行完备实现.

### ext4-fs 相关 从 vfs 说起

### Inode

真正的 存储在 block 内部的一个结构

```rs
pub struct Inode {
    /// Inode number
    inode_num: u32,
    /// Block containing the inode
    block_id: usize,
    /// Offset within the block
    block_offset: usize,
    /// Filesystem reference
    fs: Arc<Mutex<Ext4FileSystem>>,
    /// Block device reference
    block_device: Arc<dyn BlockDevice>,
    /// Cached block size (avoid locking fs repeatedly)
    block_size: usize,
}


```

todo..
