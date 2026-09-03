# 第 16 章 凭据、capability、LSM 与安全钩子

## Core Idea

Linux 权限决策不是简单比较 `euid == 0`。任务的 `struct cred` 包含 UID/GID、capability 集、user namespace、keyring 和 LSM blob；核心路径按各自语义组合 DAC、capability 与 LSM 检查。LSM 并不总是“最后追加”：`capable()` 本身就经 `security_capable()` 进入 LSM hook。正确实现要选对**主体凭据、目标 namespace、检查时点和对象语义**。

凭据对象发布后按不可变对象使用。永久修改采用 `prepare_creds()` 复制、修改新对象、`commit_creds()` 原子替换；失败用 `abort_creds()`。内核代用户访问资源时可以 `override_creds()`，但必须在所有返回路径 `revert_creds()`，并且临时覆盖不能被误当成授权来源。

## Frameworks Introduced

### 1. Credential 生命周期

- **When**：setuid、exec、keyring 或服务线程需要更新/临时采用身份。
- **How**：读取当前主观凭据用 `current_cred()`；长期持有用 `get_cred()`/`put_cred()`；修改走 prepare/commit 或 abort；临时覆盖成对调用 override/revert。
- **Why**：RCU 与引用计数允许并发读者看到稳定快照。
- **Failure**：直接修改 `current->cred` 指向对象、保存无引用 cred 指针、或 override 后错误分支遗漏 revert。

### 2. Capability 与 user namespace

- **When**：传统 root 特权需要拆分为具体能力。
- **How**：全局系统资源常用 `capable(cap)`；命名空间对象用 `ns_capable(owner_ns, cap)`；已有 file 的 opener 权限语义可用 `file_ns_capable()`。
- **Why**：容器中的“root”只在其 user namespace 及后代范围拥有能力。
- **Failure**：一律对 `init_user_ns` 检查会错误拒绝命名空间管理员；一律对 `current_user_ns()` 检查又可能把宿主资源授权给错误主体。

### 3. LSM hook

- **When**：核心安全语义点需要让 SELinux、AppArmor 等安全模块参与决策；具体 hook 可能位于 DAC 前后，也可能像 capability 检查一样构成 helper 本身。
- **How**：核心通过 `security_*()` wrapper 调用 hook 链；LSM 使用 `LSM_HOOK_INIT()` 建表、`security_add_hooks()` 注册，并用 `DEFINE_LSM()` 声明初始化。
- **Why**：核心只提供稳定的安全语义点，不硬编码具体策略。
- **Failure**：绕过已有 `security_*()` helper、自行调用具体 LSM、在错误层重复 hook，或误认为返回 0 等于绕过普通 DAC。

## Key Concepts

- **主观与客观凭据**：`current_cred()` 是当前任务发起操作时采用的主观身份；某些对象保存打开者/创建者凭据作为客观历史。
- **授权不是身份相等**：先定义操作与受保护对象，再选择 capability；不要把 `uid_eq(current_euid(), GLOBAL_ROOT_UID)` 当通用权限模型。
- **namespace 所有权**：检查必须针对对象所属 user namespace，而不是便利地取当前 namespace。
- **hook 组合**：`call_int_hook(..., 0, ...)` 遍历 hook，返回值偏离默认值即可终止。`capable()` 经 `ns_capable_common()` 调用 `security_capable()`，不能把 capability 与 LSM 画成固定的先后两阶段。
- **检查与使用**：授权后对象仍可能变化；权限检查要尽量靠近实际操作，并依赖已锁定/引用的内核对象。

## Mental Models

把一次授权写成四元组：`subject × operation × object × namespace`。如果代码评审中说不清这四项，`capable()` 放在哪里通常也说不清。LSM hook 则是这个语义点上的策略扩展槽，而不是散落在实现细节里的审计回调。

把 cred 看成版本化不可变快照：读者拿到某一版本稳定读取；修改者复制出新版本并一次提交。临时 override 只是线程当前主观视图的栈式替换，必须严格恢复。

## Anti-patterns

- 直接写 `current->cred->euid` 或 capability bitmap。
- 缓存 `current_cred()` 返回指针到异步 work，却不 `get_cred()`。
- `override_creds()` 后通过多个 `return` 离开，导致服务线程持续以高权限运行。
- 用 `capable(CAP_SYS_ADMIN)` 作为“万能通过”，没有寻找更窄 capability。
- 对命名空间对象检查 `capable()` 而不审计对象 owner namespace。
- 新增安全敏感核心路径却绕过同类路径已有的 `security_*()` hook。

## Commands & APIs

```text
current_cred / get_current_cred / get_cred / put_cred
prepare_creds / commit_creds / abort_creds
override_creds / revert_creds
capable / ns_capable / file_ns_capable
security_inode_permission / security_file_open
LSM_HOOK_INIT / security_add_hooks / DEFINE_LSM
```

观察当前进程身份与 LSM 状态：

```sh
cat /proc/self/status | rg 'Uid|Gid|Cap(Inh|Prm|Eff|Bnd|Amb)'
cat /sys/kernel/security/lsm
capsh --decode=0000000000000000
```

文件是否存在取决于 securityfs 挂载与配置；能力位图只是主体状态，不能单独证明某次操作为何被允许或拒绝。

## Worked Example

临时凭据必须结构化恢复，避免错误分支泄漏：

```c
static ssize_t read_as(const struct cred *cred, struct file *file,
		       char *buf, size_t len, loff_t *pos)
{
	const struct cred *old;
	ssize_t ret;

	old = override_creds(cred);
	ret = kernel_read(file, buf, len, pos);
	revert_creds(old);
	return ret;
}
```

调用者必须持有 `cred` 引用，并明确“为何允许采用该凭据”。`override_creds()` 解决访问时使用哪个主观身份，不负责证明调用者有权切换身份。

## Key Takeaways

1. cred 发布后不可原地修改；更新走复制、提交或放弃。
2. capability 检查要绑定具体操作和对象所属 user namespace。
3. 临时 override 是严格成对的栈式操作，并不等于授权。
4. LSM hook 应位于稳定语义边界，核心 DAC/引用/锁规则仍然有效。

## Source Anchors

- `$KERNEL_SRC/include/linux/cred.h:292` — `current_cred()`；`$KERNEL_SRC/kernel/cred.c:237` — `prepare_creds()`。
- `$KERNEL_SRC/kernel/cred.c:424` — `commit_creds()`；`$KERNEL_SRC/kernel/cred.c:517` — `abort_creds()`。
- `$KERNEL_SRC/kernel/cred.c:538` — `override_creds()`；`$KERNEL_SRC/kernel/cred.c:579` — `revert_creds()`。
- `$KERNEL_SRC/kernel/capability.c:384` — `ns_capable()`；`$KERNEL_SRC/kernel/capability.c:447` — `capable()` 对 init user namespace 的定义。
- `$KERNEL_SRC/security/security.c:472` — `security_add_hooks()`；`$KERNEL_SRC/security/security.c:700` — `call_int_hook`。
- `$KERNEL_SRC/security/security.c:779` — `security_capable()`；`$KERNEL_SRC/security/security.c:1606` — `security_file_open()`。
- `$KERNEL_SRC/include/linux/lsm_hooks.h:1552` — `struct security_hook_list`；`$KERNEL_SRC/include/linux/lsm_hooks.h:1612` — `DEFINE_LSM()`。

## Connects To

- 第 14 章：系统调用入口的权限检查必须选择正确主体和 namespace。
- 第 9 章：VFS permission/open 路径是 LSM 的主要语义边界。
- 第 17 章：审计、tracepoint 和 BPF 可观察拒绝路径，但不能替代授权。
- 第 18 章：用 user namespace 与 capability 矩阵验证安全边界。
