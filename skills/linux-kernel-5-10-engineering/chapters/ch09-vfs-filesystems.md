# 第 9 章：VFS、文件系统对象与读写挂载

## Core Idea

VFS 用对象和操作表把系统调用与具体文件系统解耦。一次路径访问不是“找到 inode 就结束”：mount 决定从哪个挂载树进入，dentry 缓存名字到 inode 的解析，inode 表示持久对象元数据，`struct file` 表示一次打开实例并保存 flags、当前位置和操作表。`super_block` 则代表一个已挂载文件系统实例。

工程阅读应沿对象生命周期走：注册文件系统类型，创建 superblock/root dentry，接入 mount namespace；路径遍历取得 `path`，open 生成 `file`；read/write 经 VFS 做权限、范围、冻结和通知，再分派到 `file_operations`；umount 和最后引用释放最终拆除对象。任何一步失败，都只能撤销自己已经取得的引用和可见性。

## Frameworks Introduced

### 1. 四对象框架

- **When**：分析路径缓存、硬链接、同一文件多次打开或卸载忙。
- **How**：用 `super_block → inode ← dentry → file` 追踪关系；额外把 `vfsmount` 视为 namespace 中的挂载实例。
- **Why**：名字、持久对象、打开状态和文件系统实例的生命周期不同，拆开才能缓存和共享。
- **Failure**：把 dentry 当 inode 会误判负 dentry；把 file 当 inode 会忽略每次打开的 `f_pos/f_flags/private_data`。

### 2. 路径遍历框架

- **When**：排查 symlink、rename、mount crossing 或权限问题。
- **How**：从 `path_openat()` 的 nameidata 状态机看 RCU walk 到 ref walk 的转换，以及最终 open 回调。
- **Why**：路径解析同时面对并发 rename、符号链接限制和挂载点切换。
- **Failure**：在不稳定 dentry 上保存裸指针，或遗漏 `path_get/path_put`，会产生 UAF/泄漏。

### 3. I/O 分派框架

- **When**：实现 `file_operations` 或追查 read/write 返回值。
- **How**：VFS 先验证 mode、用户地址和区域，再选择旧式 `read/write` 或迭代器接口；写路径用 `file_start_write/file_end_write` 与文件系统冻结协议配对。
- **Why**：通用检查只做一次，具体文件系统专注数据实现。
- **Failure**：正数可能是短读写，不是“全部完成”；负数必须保持 errno，清理和冻结计数仍需配对。

### 4. 挂载事务框架

- **When**：处理 bind/remount/move 或 mount namespace。
- **How**：区分 superblock flags 与每挂载点 flags，先做 LSM/权限检查，再分派不同挂载操作。
- **Why**：同一 superblock 可被多处挂载且每个挂载点策略不同。
- **Failure**：新 mount 构造成功但接入 namespace 失败时，必须释放 tree、路径和模块引用。

## Key Concepts

- `super_block`：已挂载文件系统实例及其 `s_op`、根 dentry、状态与锁。
- `inode`：文件身份和元数据；多个 dentry 可指向同一 inode。
- `dentry`：路径分量缓存，可为 negative；生命周期由 dcache 和引用/RCU 管理。
- `file`：一次 open 的内核对象，最终由 `fput()` 驱动释放。
- `address_space`：页缓存与文件后端的桥梁。
- mount namespace：每个 namespace 有自己的挂载拓扑，共享并不等于全局唯一。

## Mental Models

把 VFS 想成“对象图 + 分派表”。路径字符串只是一段输入；解析后真正持有的是 `(vfsmount, dentry)` 的 `path`。读写时不要从系统调用直接跳到磁盘：先到 `file_operations`，可能命中 page cache，只有缓存未命中或回写时才进入块层。

## Anti-patterns

- 长期保存 `dentry`、`path` 或 `file` 裸指针却不持引用。
- 在 `->write_iter` 内假设 VFS 已保证整次写入不可失败或不可短写。
- 忘记 `file_start_write()` 与 `file_end_write()` 必须在所有出口配对。
- 将 per-mount 的 `MNT_NOEXEC` 与 superblock 的只读等属性混为一谈。
- 新内核代码随意从内核地址走用户态式文件 I/O；应优先使用子系统接口。

## Commands & APIs

- `findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS`：检查挂载拓扑与选项。
- `cat /proc/<pid>/mountinfo`：查看进程 mount namespace 的精确关系。
- `stat <path>`、`namei -l <path>`：观察 inode 信息和逐段权限。
- `lsof <path>`：辅助定位仍持有打开引用的进程。
- `filp_open()` / `fput()`、`path_get()` / `path_put()`：必须成对。
- `vfs_read()` / `vfs_write()`：5.10 的通用同步入口；具体实现优先提供迭代器操作。

## Worked Example

读取固定内核缓冲区时，应复用内核已有 helper，让它统一处理负偏移、EOF、短读和部分用户拷贝：

```c
static ssize_t demo_read(struct file *file, char __user *buf,
			 size_t count, loff_t *ppos)
{
	return simple_read_from_buffer(buf, count, ppos,
				       data, data_len);
}
```

`simple_read_from_buffer()` 在偏移为负时返回 `-EINVAL`，到达末尾时返回 0，并按实际复制字节推进 `*ppos`；若 `copy_to_user()` 只复制一部分，它返回已完成字节数而不是错误地丢掉进度。若对象允许并发修改，调用前仍需用 mutex 或快照稳定 `data_len/data`，且不能在持 spinlock 时进入可能因用户页缺页而睡眠的复制路径。

## Key Takeaways

1. superblock、inode、dentry、file 和 mount 各自承载不同身份与生命周期。
2. 路径解析是并发状态机，引用和 RCU 边界不可省略。
3. VFS read/write 负责通用契约，文件系统回调仍要处理短 I/O 和错误。
4. 挂载是 namespace 事务；flags 分层、权限检查和失败回滚同等重要。

## Source Anchors

- `$KERNEL_SRC/include/linux/fs.h:612` — `struct inode`
- `$KERNEL_SRC/include/linux/fs.h:918` — `struct file`
- `$KERNEL_SRC/include/linux/fs.h:1451` — `struct super_block`
- `$KERNEL_SRC/include/linux/dcache.h:89` — `struct dentry`
- `$KERNEL_SRC/fs/namei.c:3405` — `path_openat()`
- `$KERNEL_SRC/fs/read_write.c:476` — `vfs_read()`
- `$KERNEL_SRC/fs/read_write.c:585` — `vfs_write()` 与冻结写配对
- `$KERNEL_SRC/fs/libfs.c:717` — `simple_read_from_buffer()`
- `$KERNEL_SRC/fs/namespace.c:3174` — `path_mount()`

## Connects To

- 第 8 章：page cache、文件映射与缺页在 `address_space` 汇合。
- 第 10 章：缓存未命中和回写把文件 I/O 变成 bio/request。
- 第 11 章：socket 也以 fd 暴露，但其数据路径不经过普通文件系统页缓存。
