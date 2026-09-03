# 第 17 章 可观测性与调试：printk、dynamic debug、ftrace、tracepoint 和 eBPF

## Core Idea

内核调试的目标不是“多打日志”，而是以最低扰动回答一个可证伪问题。`printk` 适合稀疏、持久、面向运维的事件；dynamic debug 让现有 `pr_debug()`/`dev_dbg()` 调用点运行时开关；ftrace 观察函数与延迟；tracepoint 提供字段化的结构化事件；eBPF 在校验器和权限约束下组合事件、过滤与聚合。事件字段可从对应 `format` 文件发现，但 tracepoint 及字段布局不是跨版本 UABI。

选择工具前先写假设，例如“work 从 IRQ 投递后超过 20 ms 才开始”。然后只采集 IRQ 事件、workqueue tracepoint 和时间戳。证据链应区分：日志说明分支发生，函数跟踪说明调用路径，tracepoint 说明结构化状态变化，性能采样说明统计相关；任何一种都不自动证明因果。

## Frameworks Introduced

### 1. printk 与 dynamic debug

- **When**：需要启动期、错误或低频状态的持久记录；已有 debug 调用点需要按模块/文件/函数临时开启。
- **How**：使用 `pr_err/pr_warn/pr_info/pr_debug` 与 `dev_*`；通过 debugfs 的 `dynamic_debug/control` 写 query 修改 `+p/-p` 等 flags。
- **Why**：等级表达运维重要性，dynamic debug 避免为临时诊断重编译。
- **Failure**：热路径无 rate limit 打印可改变时序、填满 ring buffer；`pr_debug()` 是否可动态控制取决于配置和调用点编译方式。

### 2. ftrace 与 tracepoint

- **When**：未知调用路径用 function/function_graph；已知子系统状态转换用事件 tracepoint。
- **How**：在 tracefs 选择 `current_tracer`、写 `set_ftrace_filter` 或启用 `events/SUBSYSTEM/EVENT/enable`，从 `trace_pipe` 消费，并从相邻 `format` 文件发现字段名、偏移和打印格式。
- **Why**：ftrace 广而动态；tracepoint 字段结构化且可从 `format` 文件发现，适合针对当前目标内核编写工具，但不是跨版本 UABI。
- **Failure**：全函数跟踪产生巨大数据与扰动；`trace_pipe` 是消费式读取，多读者会分走记录；禁用 tracing 前忘记保存配置会污染后续实验。

### 3. eBPF

- **When**：需要在内核侧过滤、关联、直方图或按 key 聚合，减少向用户态搬运事件。
- **How**：用户态经 `bpf()` 的 `BPF_PROG_LOAD` 加载程序，附着到受支持的 tracepoint/kprobe/perf 事件，借助 map 传递聚合结果。
- **Why**：可编程且经 verifier 限制，适合生产诊断的窄探针。
- **Failure**：程序加载还受 capability、sysctl、类型和 BTF/工具链限制；kprobe 依赖内部符号，跨版本脆弱；聚合结果仍可能受丢事件和采样偏差影响。

## Key Concepts

- **日志等级不是调试开关**：重要错误用稳定等级，详细诊断用 dynamic debug/trace。
- **结构化字段优于解析文本**：tracepoint 字段可通过事件 `format` 文件发现并用于过滤、聚合；它比解析 printk 文本可靠，但仍不是跨内核版本 UABI。
- **观测扰动**：打印、全量函数图和复杂 BPF 都会改变量测对象，先从最窄范围开始。
- **时钟与 CPU**：跨 CPU 时间线要保留 CPU、PID、timestamp 和迁移信息，不能只按输出顺序推断执行顺序。
- **生命周期**：自定义 tracepoint probe 必须 unregister，并等待框架要求的同步期，避免探针卸载后仍被调用。

## Mental Models

把调试当成逐层缩小的漏斗：先用计数/错误日志确认症状，再用 tracepoint 定位阶段，用 ftrace 确认意外调用边，最后才上 kprobe/eBPF 做缺失字段或相关性分析。每一层都应减少下一层的观测范围。

另一个模型是“事件账本”：每条记录包含谁、何时、在哪个 CPU、什么状态、关联键。没有关联键的日志很难把 IRQ、work 和请求串起来；没有丢失/覆盖说明的 trace 也不能当完整账本。

## Anti-patterns

- 在高频路径循环 `pr_info()`，然后依据被改变的时序诊断竞态。
- 把 `dmesg | tail` 当成完整事件历史，忽略 ring buffer 覆盖和 console level。
- 不设 `set_ftrace_filter` 就开启 function_graph 全系统追踪。
- 多个消费者同时读 `trace_pipe`，再误判事件“随机丢失”。
- 将 tracepoint 当稳定用户 ABI，或将 kprobe 参数布局当跨版本承诺。
- eBPF 程序能加载就认为语义正确，未验证 attach 点、过滤条件和 map key。

## Commands & APIs

```text
pr_debug / dev_dbg / pr_warn_ratelimited / printk_deferred
register_ftrace_function / unregister_ftrace_function
TRACE_EVENT / register_trace_* / unregister_trace_*
bpf(BPF_PROG_LOAD, ...) / BPF maps
```

一次窄范围 tracefs 会话：

```sh
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo nop > current_tracer
echo 1 > events/workqueue/workqueue_execute_start/enable
echo 1 > events/workqueue/workqueue_execute_end/enable
echo 1 > tracing_on
cat trace_pipe
```

结束后写 `0` 禁用已启用事件和 `tracing_on`。实际 tracefs 也可能挂载在 `/sys/kernel/tracing`。

## Worked Example

验证某个模块中的 debug 调用点，而不重编译：

```sh
DBG=/sys/kernel/debug/dynamic_debug/control
echo 'module demo function demo_rx +p' > "$DBG"
dmesg -w
# 复现一次问题后立即关闭
echo 'module demo function demo_rx -p' > "$DBG"
```

若 query 返回错误，先从 control 文件读取真实 module/file/function 字段。启用成功只证明调用点会打印；要衡量延迟仍应改用 tracepoint/ftrace 时间戳。

## Key Takeaways

1. 先写可证伪假设，再选择最窄、最低扰动的观测工具。
2. printk 面向稀疏运维事件，动态调试和 tracing 面向临时诊断。
3. tracepoint 提供可发现的结构化字段但不是跨版本 UABI；ftrace/eBPF 扩展路径与聚合能力。
4. 输出顺序、无丢失和因果关系都不能未经证明地假设。

## Source Anchors

- `$KERNEL_SRC/kernel/printk/printk.c:2021` — `vprintk_emit()`；`$KERNEL_SRC/kernel/printk/printk.c:3119` — deferred printk。
- `$KERNEL_SRC/include/linux/printk.h:412` — `pr_debug()` 配置展开。
- `$KERNEL_SRC/lib/dynamic_debug.c:739` — dynamic debug control 写路径；`$KERNEL_SRC/Documentation/admin-guide/dynamic-debug-howto.rst:125` — file query 示例。
- `$KERNEL_SRC/kernel/trace/ftrace.c:7612` — `register_ftrace_function()`；`$KERNEL_SRC/Documentation/trace/ftrace.rst:136` — `trace_pipe`。
- `$KERNEL_SRC/include/linux/tracepoint.h:444` — `TRACE_EVENT` 使用说明；`$KERNEL_SRC/kernel/tracepoint.c:539` — probe 注册。
- `$KERNEL_SRC/Documentation/trace/events.rst:105` — 每个事件的 `format` 文件描述字段。
- `$KERNEL_SRC/include/trace/events/sched.h:222` — `sched_switch` trace event。
- `$KERNEL_SRC/kernel/bpf/syscall.c:4397` — `bpf()` syscall；`$KERNEL_SRC/kernel/bpf/syscall.c:4438` — `BPF_PROG_LOAD` 分派。

## Connects To

- 第 13 章：串联 IRQ、timer、work 与唤醒的延迟时间线。
- 第 16 章：BPF 与 tracing 的 capability/安全策略边界。
- 第 18 章：将 trace 作为测试诊断证据，而不是测试断言的替代品。
