# 第 3 章：工程工作流、风格与补丁范围

## Core Idea

内核补丁的基本交付单位是一个**可独立解释、构建、审查和回退的逻辑变更**。好补丁不只是代码正确：提交信息要解释问题和因果，diff 要小而完整，风格应服务于长期维护，验证结果要与实际执行严格对应。

5.10.240 自带的提交流程文档要求将每个 logical change 分成独立 patch，并把 `checkpatch.pl` 定位为辅助工具而非正确性证明。工程目标不是“让脚本全绿”，而是让审查者能从问题、约束、实现到验证建立闭环。

## Frameworks Introduced

### 问题—机制—范围—证据四段式

- **When**：写代码前、写提交信息时、回复 review 时。
- **How**：先陈述可观察问题，再解释根因机制，列出变更边界，最后给出与声明匹配的验证。
- **Why**：审查者首先判断问题是否真实、方案是否必要，而非语法是否漂亮。
- **Failure**：提交信息只写 “fix bug” 或复述 diff，无法判断回退和稳定版适用性。

### 单逻辑变更切片

- **When**：同时出现重命名、重构、行为修复、测试或配置修改时。
- **How**：按可独立回退的因果单元拆分；纯机械改动与行为改变分开。
- **Why**：便于 bisect、review 和 backport。
- **Failure**：为了追求“一文件一补丁”而切断必要原子性，导致中间提交不可构建。

## Key Concepts

- **作用域由维护边界决定**：先查 `MAINTAINERS` 和 `get_maintainer.pl`，再判断需要哪些子系统审查者。
- **风格是可维护性协议**：8 字符 tab、短函数、清晰命名和一致错误出口降低认知负担；内核允许 `goto` 集中释放资源。
- **标签是机器可消费元数据**：`Fixes:` 指向引入问题的提交；`Signed-off-by:` 表示 DCO 责任链；它们不能凭空添加。
- **验证陈述分级**：“编译通过”“测试通过”“启动验证”“硬件验证”是不同结论。没执行的测试不能写成已通过。
- **稳定版补丁约束**：面向 stable 的修复应更小、更保守，并准确判断受影响版本；不是所有改进都适合 `Cc: stable`。

## Mental Models

把每个 patch 看成一个小定理：前提是旧行为和环境，命题是问题，证明是最小代码变更，检验是测试和静态工具。diff 中每一行都应服务于证明；无关格式化会扩大需要重新证明的面积。

审查顺序可采用“外到内”：提交范围与维护者 → 提交信息 → Kconfig/Kbuild → API 与并发约束 → 错误路径 → 测试。先确认边界，避免在错误方案上雕琢局部代码。

## Anti-patterns

- 修 bug 顺手全文件格式化，掩盖真正语义变化。
- 用新增 wrapper 或抽象层替代一个局部、已有模式能解决的问题。
- 为消除 `checkpatch` 警告改变外部 ABI 或稳定接口。
- 错误路径逐层复制释放逻辑，造成后续新增资源时遗漏；应考虑逆序 `goto` 清理。
- 提交信息宣称“tested”却只做静态搜索，或把未运行命令列在 Test 行。
- 未核对引入提交便填写 `Fixes:`。

## Commands & APIs

```bash
git status --short
git diff --stat
git diff --check
./scripts/checkpatch.pl --strict /tmp/change.patch
./scripts/get_maintainer.pl /tmp/change.patch
git log --oneline -- path/to/file.c
```

局部构建可用 `make O=... path/to/object.o`，但对象级成功不等价于最终链接成功。提交前还应按变更性质选择配置矩阵、目标架构、静态分析或运行测试，并保存真实输出。

## Worked Example：资源申请失败路径的最小补丁

假设函数依次申请缓冲区、注册 IRQ、创建设备节点。新增第三步失败处理时，不复制两段释放代码，而使用逆序出口：

```c
buf = kmalloc(size, GFP_KERNEL);
if (!buf)
	return -ENOMEM;
ret = request_irq(irq, handler, 0, name, dev);
if (ret)
	goto out_free;
ret = device_create_file(dev, &dev_attr_state);
if (ret)
	goto out_irq;
return 0;
out_irq:
free_irq(irq, dev);
out_free:
kfree(buf);
return ret;
```

补丁范围只包含错误路径修复和能触发该失败的测试/注入说明；变量重命名另做 patch。验证声明应具体，例如“目标对象编译通过、故障注入返回原错误码且资源计数恢复”，前提是这些步骤确实执行。

## Key Takeaways

- 一个 patch 对应一个完整逻辑变化，而不是一个文件或一个函数。
- 提交信息解释因果和边界，diff 提供最小实现，测试提供证据。
- 工具提示不能替代人工语义审查。
- 明确区分计划执行的命令与已经取得的验证结果。

## Source Anchors

- `$KERNEL_SRC/Documentation/process/submitting-patches.rst:40` — “Describe your changes”。
- `$KERNEL_SRC/Documentation/process/submitting-patches.rst:122` — `Fixes:` 标签要求。
- `$KERNEL_SRC/Documentation/process/submitting-patches.rst:144` — “Separate your changes”。
- `$KERNEL_SRC/Documentation/process/submitting-patches.rst:147` — 每个 logical change 独立 patch。
- `$KERNEL_SRC/Documentation/process/submitting-patches.rst:177` — 风格检查章节。
- `$KERNEL_SRC/Documentation/process/submitting-patches.rst:356` — DCO 与 sign-off。
- `$KERNEL_SRC/Documentation/process/coding-style.rst:21` — 8 字符 tab。
- `$KERNEL_SRC/Documentation/process/coding-style.rst:430` — 函数应短且单一职责。
- `$KERNEL_SRC/Documentation/process/coding-style.rst:478` — `goto` 错误退出讨论。

## Connects To

第 2 章决定配置/构建验证矩阵；第 4–6 章提供并发补丁必须写清的上下文、锁、内存序与生命周期约束。任何并发修复都应把这些约束写进提交因果链。
