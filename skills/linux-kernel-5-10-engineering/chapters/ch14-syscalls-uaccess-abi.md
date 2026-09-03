# 第 14 章 系统调用、uaccess、UAPI、compat 与架构 ABI

## Core Idea

系统调用不是普通 C 函数入口，而是一个长期兼容的信任边界。用户寄存器经架构入口转换为内核参数，`SYSCALL_DEFINE<n>` 生成包装层，再进入可复用的内核 helper。任何 `__user` 指针都只能通过 uaccess API 访问；任何暴露给用户空间的结构、编号和错误码都应按永久 ABI 设计。

工程判断应从 ABI 开始：现有 syscall、ioctl、netlink、sysfs 是否已经能表达需求？若确需新 syscall，先定义可扩展参数、权限和取消语义，再实现入口。内核内部 helper 不应接受未经拷贝的用户指针，以便内核调用者、compat 层和测试复用。

## Frameworks Introduced

### 1. Syscall 定义与架构接线

- **When**：功能是系统级、跨设备/文件描述符且现有 ABI 无法自然承载。
- **How**：用 `SYSCALL_DEFINE<n>` 实现通用入口，在 generic unistd 和目标架构 syscall table 分配编号，并提供原型、fallback 与自测。
- **Why**：宏生成元数据、类型检查包装和架构期望的符号。
- **Failure**：只在一个表加编号会造成架构缺失；直接写 `sys_foo()` 绕过包装；发布后改变结构布局或标志含义会永久破坏用户程序。

### 2. uaccess

- **When**：系统调用或 ioctl 读写用户地址。
- **How**：标注 `__user`；标量用 `get_user()`/`put_user()`，块用 `copy_from_user()`/`copy_to_user()`；非零返回表示仍有字节未复制，通常映射为 `-EFAULT`。
- **Why**：用户页可能未映射、被并发修改或触发 fault，不能直接解引用。
- **Failure**：把返回值当 errno、在持有 spinlock/禁抢占上下文中执行可能 fault 的复制、或复制后再次读取用户内存导致 TOCTOU。

### 3. UAPI 与 compat

- **When**：头文件会被 libc/应用包含，或 64 位内核支持 32 位进程。
- **How**：稳定类型用 `__u32/__u64`；结构显式保留字段并校验 flags/size；指针或 `long` 布局变化时提供 `COMPAT_SYSCALL_DEFINE<n>`/`.compat_ioctl` 转换。
- **Why**：同一 ABI 要跨编译器、字长、端序和多年内核版本。
- **Failure**：UAPI 使用裸指针、`long`、隐式 padding 或内核私有类型；对 scalar ioctl 参数错误调用 `compat_ptr()`。

## Key Concepts

- **入口薄、helper 厚**：入口负责拷贝、验证、权限与 ABI 翻译；内部 helper 只接收内核对象。
- **一次快照**：先完整 `copy_from_user()` 到内核结构，再验证副本；不要验证用户字段后又直接使用原地址。
- **可扩展结构**：将用户提供的 `usize` 作为独立 syscall 参数；先检查最小已发布版本，再用 `copy_struct_from_user()` 处理新旧结构。未知非零尾部返回 `-E2BIG`，未知 flags 返回明确错误。
- **ABI 错误语义**：返回负 errno；部分成功、重启和阻塞语义必须在设计阶段确定。
- **compat 是数据模型转换**：不是简单“把指针截成 32 位”，而是检查每个字段在 ILP32 与 LP64 下的大小和对齐。

## Mental Models

把 syscall 看成协议解码器：寄存器/用户内存是“不可信线格式”，内核结构是“已验证内部表示”。解码只发生一次，验证在副本上完成，业务逻辑永不接触线格式。compat 层则是另一个线格式解码器，最终汇入同一个 helper。

ABI 设计像数据库 schema 迁移：新增字段可以有默认值，旧字段不能重解释，编号不能复用，删除实现也要保留可预测的 `-ENOSYS` 或兼容行为。

## Anti-patterns

- `struct foo *p = (void *)arg; p->field` 直接解引用用户地址。
- `if (copy_from_user(...)) return copied;` 把“未复制字节数”直接返回用户。
- UAPI 结构中放 `size_t`、`time_t`、裸 enum 或自然对齐指针。
- ioctl 未知命令返回 `-EINVAL`；5.10 文档要求使用 `-ENOTTY`。
- compat handler 无条件 `compat_ptr(arg)`，即使 `arg` 是整数而非指针。
- 先发布固定结构，后来通过改变同一命令号的结构大小扩展 ABI。

## Commands & APIs

```text
SYSCALL_DEFINE0..6 / COMPAT_SYSCALL_DEFINE0..6
copy_from_user / copy_to_user / get_user / put_user
access_ok / compat_ptr / compat_ptr_ioctl
_IO / _IOR / _IOW / _IOWR
```

审查接线可搜索目标表与实现：

```sh
rg -n 'foo|__NR_foo' include/uapi arch/*/entry/syscalls
rg -n 'SYSCALL_DEFINE.*foo|COMPAT_SYSCALL_DEFINE.*foo' .
scripts/checksyscalls.sh gcc -E -D__KERNEL__ -x c /dev/null
```

最后一条依赖构建环境，输出需结合具体架构判断。

## Worked Example

入口只解码一次，并拒绝未知标志：

```c
struct foo_args {
	__u32 flags;
	__u32 value;
};
#define FOO_SIZE_VER0 offsetofend(struct foo_args, value)
SYSCALL_DEFINE2(foo, struct foo_args __user *, uarg, size_t, usize)
{
	struct foo_args a = {};
	int ret;
	if (usize > PAGE_SIZE)
		return -E2BIG;
	if (usize < FOO_SIZE_VER0)
		return -EINVAL;
	ret = copy_struct_from_user(&a, sizeof(a), uarg, usize);
	if (ret)
		return ret;
	if (a.flags)
		return -EINVAL;
	return do_foo(a.value);
}
```

`copy_struct_from_user()` 会为较旧短结构补零，并要求较新结构超出当前内核已知大小的尾部全零。真实新增 syscall 还需编号、原型、Kconfig/Makefile（如适用）、fallback 和自测；示例只展示信任边界。

## Key Takeaways

1. 用户地址永远通过 uaccess，非零 copy 返回值不是 errno。
2. syscall 入口完成 ABI 解码，内部 helper 接收已验证内核数据。
3. UAPI 一旦发布就按永久兼容处理，优先固定宽度类型与可扩展布局。
4. compat 必须基于字段布局审计，不能凭“都是整数”推断兼容。

## Source Anchors

- `$KERNEL_SRC/include/linux/syscalls.h:206` — `SYSCALL_DEFINE0`；`$KERNEL_SRC/include/linux/syscalls.h:214` — `SYSCALL_DEFINE1..6`。
- `$KERNEL_SRC/include/linux/uaccess.h:59` — raw copy 契约；`$KERNEL_SRC/include/linux/uaccess.h:89` — `copy_from_user()` 短复制补零差异。
- `$KERNEL_SRC/include/linux/uaccess.h:298` — `copy_struct_from_user()` 的版本化结构契约；`$KERNEL_SRC/include/linux/uaccess.h:343` — 实现与尾部检查。
- `$KERNEL_SRC/fs/open.c:1211` — `do_sys_openat2()` 内部 helper；`$KERNEL_SRC/fs/open.c:1262` — `SYSCALL_DEFINE4(openat2)`。
- `$KERNEL_SRC/arch/x86/entry/syscalls/syscall_64.tbl:7` — x86-64 stub/table 说明。
- `$KERNEL_SRC/include/linux/compat.h:48` — compat syscall 宏；`$KERNEL_SRC/include/linux/compat.h:922` — `compat_ptr()`。
- `$KERNEL_SRC/Documentation/process/adding-syscalls.rst:245` — 新 syscall 接线清单；`$KERNEL_SRC/Documentation/driver-api/ioctl.rst:116` — compat ioctl 指南。

## Connects To

- 第 9 章的 VFS/file_operations：很多设备 ABI 应落在 fd/ioctl/read/write。
- 第 16 章的 credentials/LSM：权限检查必须使用正确 user namespace 与 security hook。
- 第 18 章的 kselftest：ABI 需要从用户态覆盖正常、错误和 compat 路径。
