# 第 15 章 模块、initcall、注册、错误回退与卸载

## Core Idea

模块初始化不是“执行一次 setup”，而是一笔多阶段资源事务：每成功注册一个外部可见对象，就形成一个必须按逆序撤销的责任。`module_init()` 对可加载模块生成加载入口；同一代码内建时则进入 initcall 段。初始化返回 0 后模块进入 LIVE，退出函数只有在引用与使用者允许时才会执行。

因此可靠驱动应把 init/probe 看作提交过程，把错误标签和 remove/exit 看作回滚过程。`module_exit()` 对应函数的类型是 `void (void)`，不能报告卸载失败：依赖关系、LIVE 状态、是否提供 exit 以及模块引用等卸载资格，必须由模块核心在进入 exit 前阻止或放行。exit 自身负责同步排空模块内部异步执行后再释放资源。

## Frameworks Introduced

### 1. module_init/module_exit 与 initcall

- **When**：模块或内建设备功能需要启动/停止入口。
- **How**：可加载模块各有一个 `module_init()`；内建代码可按依赖选择 `core_initcall`、`subsys_initcall`、`device_initcall`、`late_initcall` 等层级。
- **Why**：链接段决定内建初始化顺序，模块加载器则执行 `mod->init()` 并根据返回值提交或释放。
- **Failure**：依赖偶然链接顺序、滥用更早 initcall、或给内建代码写只有 `module_exit()` 才会清理的运行期资源。

### 2. 注册/注销对

- **When**：向字符设备、总线、网络、文件系统或其他核心注册对象。
- **How**：为每个 `register/add/create` 写出对应的 `unregister/del/destroy`，成功后才推进阶段。
- **Why**：注册会发布入口并建立引用关系，简单 `kfree()` 不能撤销全局可见性。
- **Failure**：注销顺序与注册顺序相同；错误路径遗漏早期资源；注销后误以为既有打开文件或回调立即消失。

### 3. 模块引用与卸载

- **When**：模块代码可能被异步任务、文件实例、回调或其他模块继续调用。
- **How**：框架通常通过 `.owner = THIS_MODULE` 自动持有引用；手工跨边界时使用 `try_module_get()`/`module_put()`。模块核心先冻结引用并判定可卸载，之后调用无返回值的 exit；exit 必须同步注销和排空内部资源。
- **Why**：引用计数阻止代码仍在使用时卸载。
- **Failure**：引用只能保住模块映像，不会自动保护私有对象；永久引用或漏 `module_put()` 会使卸载总是 `-EBUSY`。

## Key Concepts

- **发布点**：`cdev_add()`、`misc_register()`、driver registration 成功后，外部线程可能立即进入回调。
- **逆序回退**：若顺序为 A→B→C，C 失败时按 B→A 撤销；每个标签对应一个已完成阶段。
- **错误指针**：注册/创建 API 可能返回 `ERR_PTR()`，必须按 API 契约用 `IS_ERR()`/`PTR_ERR()`，不能只判 NULL。
- **init 内存**：`__init` 标记的代码在初始化后可被释放，运行期函数指针绝不能指向它；`__exit` 对内建代码可能被丢弃。
- **卸载栅栏**：进入 exit 前，模块核心已完成依赖、状态和引用资格判断；exit 中仍要先从子系统移除入口，再关闭 IRQ/timer/work/thread，等待模块自己建立的异步调用离开，最后释放数据。

## Mental Models

维护一张“资源栈”：每次成功 acquisition 就压栈一个 release 动作；错误和退出都从栈顶弹出。不同之处是错误路径只回滚已完成阶段，正常退出回滚全部运行期阶段。

把注册理解为 publication barrier：它不是单纯拿到编号，而是把对象交给并发世界。所有不变量要在发布前建立；注销只是关闭新入口，是否还需等待旧使用者取决于具体子系统契约。

## Anti-patterns

- init 中连续调用多个注册函数，只在最后统一 `return ret`，没有阶段化回退。
- 先 `cdev_add()`，后初始化锁、wait queue 或私有指针。
- remove/exit 先 `kfree(dev)`，再 `cancel_work_sync()` 或 `free_irq()`。
- 用 `try_module_get(THIS_MODULE)` 代替对象引用计数或 RCU 生命周期设计。
- 把 initcall 等级当“数值越小越快”，而不描述实际依赖。
- 把 exit 写成返回 `int` 并试图用错误码拒绝卸载；`module_exit()` 要求 `void`，此时已没有失败回滚协议。

## Commands & APIs

```text
module_init / module_exit / MODULE_LICENSE
core_initcall / subsys_initcall / device_initcall / late_initcall
try_module_get / module_put
alloc_chrdev_region / cdev_add / cdev_del / unregister_chrdev_region
misc_register / misc_deregister
IS_ERR / PTR_ERR / devm_add_action_or_reset
```

静态检查模块元数据和引用关系：

```sh
modinfo ./demo.ko
readelf -S ./demo.ko | rg 'init|exit|modinfo'
lsmod
cat /sys/module/demo/refcnt
```

`refcnt` 是否存在和可见受配置影响；`lsmod` 只显示当前模块状态，不能证明退出路径无竞态。

## Worked Example

下面的错误标签只撤销已成功阶段，并按逆序执行：

```c
static int __init demo_init(void)
{
	int ret;
	ret = alloc_chrdev_region(&devt, 0, 1, "demo");
	if (ret)
		return ret;
	cdev_init(&demo_cdev, &demo_fops);
	ret = cdev_add(&demo_cdev, devt, 1);
	if (ret)
		goto err_region;
	ret = demo_async_start();
	if (ret)
		goto err_cdev;
	return 0;
err_cdev:
	cdev_del(&demo_cdev);
err_region:
	unregister_chrdev_region(devt, 1);
	return ret;
}
```

正常退出还需先关闭 `cdev` 新入口，再同步停止 `demo_async_start()` 建立的全部异步源；具体顺序取决于打开文件和异步路径的引用契约。

## Key Takeaways

1. 初始化是事务，成功阶段必须有精确的逆序回退。
2. 注册即发布；发布前对象必须完整，注销后仍要处理既有使用者。
3. 模块引用保护代码映像，不自动解决私有对象生命周期。
4. 卸载资格在调用 exit 前决定；exit 不可失败，必须完成“撤入口、排异步、释资源”。

## Source Anchors

- `$KERNEL_SRC/include/linux/module.h:82` — `module_init()` 语义；`$KERNEL_SRC/include/linux/module.h:92` — `module_exit()`。
- `$KERNEL_SRC/include/linux/init.h:191` — initcall section 生成；`$KERNEL_SRC/init/main.c:1192` — `do_one_initcall()`。
- `$KERNEL_SRC/kernel/module.c:3713` — `do_init_module()`；`$KERNEL_SRC/kernel/module.c:3741` — 转为 `MODULE_STATE_LIVE`。
- `$KERNEL_SRC/kernel/module.c:940` — `try_stop_module()` 冻结引用；`$KERNEL_SRC/kernel/module.c:973` — `delete_module` 在调用 `mod->exit()` 前完成卸载资格检查。
- `$KERNEL_SRC/include/linux/init.h:117` — `exitcall_t` 是 `void (*)(void)`；`$KERNEL_SRC/include/linux/module.h:522` — 模块 exit 函数指针类型。
- `$KERNEL_SRC/fs/char_dev.c:226` — `alloc_chrdev_region()`；`$KERNEL_SRC/fs/char_dev.c:470` — `cdev_add()`；`$KERNEL_SRC/fs/char_dev.c:584` — `cdev_del()` 生命周期说明。
- `$KERNEL_SRC/drivers/char/misc.c:173` — `misc_register()`；`$KERNEL_SRC/drivers/char/misc.c:239` — `misc_deregister()`。

## Connects To

- 第 13 章：模块退出必须同步取消 IRQ、timer 和 work。
- 第 12 章：设备模型的 probe/remove 本质上也是资源事务。
- 第 18 章：用 fault injection 逐点强迫 init 失败，检查每条回退路径。
