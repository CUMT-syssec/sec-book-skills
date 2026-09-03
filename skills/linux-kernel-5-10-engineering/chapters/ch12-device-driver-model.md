# 第 12 章：设备模型、总线、驱动、probe/remove 与电源管理

## Core Idea

设备模型把“硬件/逻辑设备存在”与“某段驱动代码能服务它”拆成两个独立注册过程。`struct device` 表示实例并嵌入 kobject；`struct device_driver` 表示驱动；`bus_type` 提供 match、probe/remove 等规则。设备和驱动都注册到同一 bus 后，核心匹配并进入 `really_probe()`，成功才建立绑定关系。

`probe()` 是资源事务，不是初始化脚本：每成功取得一种资源，就必须定义后续失败或 remove 时的撤销方式。5.10 的 driver core 会替通用部分撤销 sysfs、DMA、devres、PM domain 等状态，但驱动仍需对自己未纳入 devres 的中断、work、timer、队列和硬件状态负责。remove 还必须先阻止新工作，再排空在途执行，最后释放内存。

## Frameworks Introduced

### 1. bus-match-bind 框架

- **When**：理解为什么设备未 probe、错误驱动绑定或 deferred probe。
- **How**：确认 device 和 driver 的 `bus` 相同，再读 bus `match()`，最后追 `driver_probe_device()`/`really_probe()`。
- **Why**：发现设备和加载驱动的顺序可以独立，核心负责双向触发匹配。
- **Failure**：match 成功但依赖 supplier 未就绪时应返回 `-EPROBE_DEFER`；硬编码重试循环会破坏依赖排序。

### 2. probe 事务框架

- **When**：任何驱动初始化。
- **How**：按总线和设备契约安排阶段，而不是套固定顺序。通常先建立供电/复位/时钟与映射并初始化锁、work 和所有 IRQ handler 可见状态；保持设备中断源 quiesced，确认或清除 pending 状态后再申请 IRQ，因为 `request_*irq()` 返回前 handler 就可能开始执行；随后按硬件契约启动设备，并只在对象已可安全服务回调时建立子系统/用户可见注册。优先使用 `devm_*` 管理合适的从属资源。
- **Why**：失败发生在任意阶段，逆序撤销才能避免回调访问半初始化对象。
- **Failure**：对外注册过早、错误标签顺序错误、devm 与手工 free 混用都会产生竞态或双重释放。

### 3. remove 排空框架

- **When**：模块卸载、热拔插、unbind。
- **How**：先设置 stopping/从上层注销，关闭硬件事件源，同步 IRQ，取消 timer/work，等待引用和 I/O，再释放资源。
- **Why**：`remove()` 返回意味着驱动资源可以消失，所有异步入口必须已经不可达。
- **Failure**：只释放 MMIO/内存而不排空 workqueue 或 IRQ，是经典 UAF。

### 4. PM 回调框架

- **When**：runtime suspend、系统 suspend/resume。
- **How**：使用 `dev_pm_ops`；suspend 先静止上层和设备，再保存状态/关资源，resume 反向恢复。系统 PM 核心按设备依赖顺序执行并在失败时恢复已 suspend 的设备。
- **Why**：设备间存在 parent/child 和 supplier/consumer 依赖。
- **Failure**：回调中持有会与 I/O completion 互等的锁，或 suspend 后仍允许新请求，会死锁或访问断电硬件。

## Key Concepts

- `device`：名称、parent、bus、driver、class、devt、release、PM 和 DMA 信息。
- `device_driver`：name、bus、owner、probe/remove、PM 匹配一类设备。
- `bus_type`：match、uevent、probe/remove 及 bus 级属性。
- kobject/sysfs：设备可见性与引用计数基础；`release()` 是 device 生命周期最终回收点。
- devres：绑定到 device 的 LIFO 资源动作，probe 失败和 detach 时统一释放。
- deferred probe/device links：表达消费者依赖尚未就绪的供应者。

## Mental Models

把 driver core 想成“注册表 + 匹配器 + 事务协调器”。bus 判断是否相配，probe 真正取得资源，绑定成功后对象才稳定服务请求。remove 是 probe 的时间反演，但要多一个关键步骤：先切断所有并发入口并排空在途工作。

## Anti-patterns

- 在 probe 前半段就创建设备节点/网络接口，使用户可访问半初始化状态。
- 所有 probe 错误都返回 `-EPROBE_DEFER`；永久配置错误因此无限延迟。
- 使用 devm 分配后又在 remove 中手工释放同一资源。
- `device_unregister()` 后直接 `kfree(dev)`，忽略引用者与 `release()` 回调。
- remove 先 free 私有数据，后 `cancel_work_sync()` 或 `free_irq()`。
- runtime PM 回调与系统 PM 回调各自修改硬件状态却没有共同状态机。

## Commands & APIs

- `ls -l /sys/bus/<bus>/devices /sys/bus/<bus>/drivers`：观察匹配关系。
- `readlink /sys/bus/<bus>/devices/<dev>/driver`：确认当前绑定。
- `cat /sys/bus/<bus>/devices/<dev>/power/runtime_status`：runtime PM 状态。
- `modinfo <module>`、`dmesg -w`：核对 modalias、probe 与 deferred 日志。
- `device_initialize()` + `device_add()` / `device_del()` + `put_device()`：分阶段生命周期。
- `device_register()` / `device_unregister()`：组合接口。
- `devm_kzalloc()`、`devm_request_irq()`：device-managed 资源。

## Worked Example

手工资源的 probe 回滚必须镜像成功顺序：

```c
static int demo_probe(struct platform_device *pdev)
{
    struct demo *d;
    int ret;

    d = devm_kzalloc(&pdev->dev, sizeof(*d), GFP_KERNEL);
    if (!d)
        return -ENOMEM;
    ret = demo_hw_enable(d);
    if (ret)
        return ret;
    ret = demo_register_frontend(d);
    if (ret)
        demo_hw_disable(d);
    return ret;
}
```

若 frontend 注册成功，remove 必须先注销 frontend，确保没有新入口和在途回调，再关闭硬件。`devm_kzalloc()` 无需手工释放，但 `demo_hw_enable()` 若不是 devm action 管理，就必须显式回滚。

## Key Takeaways

1. device、driver 与 bus 独立注册，match 后才进入绑定事务。
2. probe 应按依赖顺序获取资源、最后暴露接口，失败严格逆序撤销。
3. remove 的第一任务是切断和排空并发入口，释放资源在最后。
4. PM 是跨设备依赖的状态机，suspend/resume 顺序和失败恢复不可省略。

## Source Anchors

- `$KERNEL_SRC/include/linux/device.h:481` — `struct device`
- `$KERNEL_SRC/include/linux/device/driver.h:95` — `struct device_driver`
- `$KERNEL_SRC/include/linux/device/bus.h:82` — `struct bus_type`
- `$KERNEL_SRC/drivers/base/core.c:2890` — `device_add()` 及分段错误回滚
- `$KERNEL_SRC/drivers/base/core.c:3145` — `device_del()`
- `$KERNEL_SRC/drivers/base/bus.c:806` — `bus_register()`
- `$KERNEL_SRC/drivers/base/driver.c:216` — `driver_register()`
- `$KERNEL_SRC/drivers/base/dd.c:497` — `really_probe()`
- `$KERNEL_SRC/drivers/base/dd.c:617` — probe 失败反向清理
- `$KERNEL_SRC/drivers/base/power/main.c:1749` — `dpm_suspend()`

## Connects To

- 第 13 章：remove 前必须排干 IRQ、timer、workqueue 和 completion 等异步来源。
- 第 10 章：块驱动把 device model 生命周期接到 blk-mq request 生命周期。
- 第 11 章：网卡驱动把 NAPI/netdevice 注销纳入 remove 和 PM 顺序。
