# 第 1 章：源码定位、版本与证据边界

## Core Idea

内核工程的第一步不是“读懂整个内核”，而是建立可复核的定位链：**正在讨论哪个版本、哪个配置、哪个架构、哪个符号、哪条调用路径**。本书固定源码基线为 Linux 5.10.240；顶层 `Makefile` 的 `VERSION=5`、`PATCHLEVEL=10`、`SUBLEVEL=240` 是版本事实，目录名和记忆都不能替代它。

源码回答“实现可能怎样工作”，构建产物回答“这个配置实际包含什么”，运行态回答“这台机器此刻发生了什么”。三类证据不能互相冒充：看到 `#ifdef CONFIG_X` 下的函数，不等于目标配置启用了它；看到 `vmlinux` 中有符号，不等于当前路径执行过；看到日志，不一定能证明日志来自你刚构建的映像。

## Frameworks Introduced

### 四坐标定位法

- **When**：接到 bug、移植、审计或性能问题时立即使用。
- **How**：记录 `release + commit/源码快照 + ARCH/config + symbol/path`；先找定义，再找声明、调用者、配置门控和构建归属。
- **Why**：同名 API 在版本间会改变，架构实现也可能覆盖通用实现。
- **Failure**：只贴一段代码或一个函数名，导致结论跨版本、跨架构漂移。

### 证据阶梯

1. 文档和注释用于形成假设；2. 定义、调用点、Kconfig/Makefile 用于证明静态关系；3. 编译、反汇编、符号表用于证明产物；4. trace、日志、崩溃栈用于证明运行行为。越靠后越接近现实，但也越依赖环境。

## Key Concepts

- **源码树入口**：顶层 `Makefile` 先组织 `init/ usr/`，随后加入 `kernel/ mm/ fs/ ...`；这比按目录名猜职责更可靠。
- **符号是导航主键**：以 `start_kernel()` 为例，从 `init/main.c` 的定义出发，可继续追踪初始化调用；模块入口则由 `module_init()` 宏接入 initcall 机制。
- **通用层与架构层**：`include/linux/` 常给出跨架构接口，`arch/<arch>/` 可能提供真正指令级实现。搜索命中多个定义时必须结合预处理条件。
- **生成文件边界**：`include/generated/`、`include/config/` 和 `System.map` 属于构建输出，不应当被当作干净源码快照的固有内容。
- **负证据很弱**：一次 `rg` 未命中，只能说明当前搜索范围和模式未命中，不能直接断言功能不存在。

## Mental Models

把内核看成一张受配置裁剪的图：Kconfig 决定节点是否可选，Makefile 决定对象是否进入链接，C 预处理决定某段实现是否存在，链接器决定最终符号，运行上下文决定路径能否安全执行。阅读任务是沿这张图收缩范围，而不是线性翻目录。

另一个实用模型是“声明—实现—归属—调用—观测”五联表。任一结论至少落在其中两格；涉及“实际运行”时必须补观测格。

## Anti-patterns

- 以 `uname -r` 证明某个源码目录的版本，或反过来以目录名证明正在运行该内核。
- 搜到第一个同名函数就停止，忽略静态内联、宏展开和 `arch/` 覆盖。
- 从最新在线文档倒推 5.10.240 行为；本章所有结论应回到该树核验。
- 把 `grep` 命中当作功能启用证据，忽略 `.config`、依赖和链接结果。
- 修改生成文件来“修复”源码问题。

## Commands & APIs

以下命令是可复制的调查模板，并非本书声称已执行过构建：

```bash
export KERNEL_SRC=/path/to/linux-5.10.240
make -s -C "$KERNEL_SRC" kernelversion
rg -n 'start_kernel\(' "$KERNEL_SRC"
rg -n 'CONFIG_PREEMPT' "$KERNEL_SRC"/{Kconfig,init,kernel,arch}
git -C "$KERNEL_SRC" describe --always --dirty 2>/dev/null
```

常用工具链：`rg` 找候选，`git grep` 在受版本控制文件中定位，`git log -L` 追踪函数历史，`scripts/get_maintainer.pl -f` 确认维护范围，`nm/readelf/objdump` 检查实际产物。

## Worked Example：定位启动入口而不越过证据边界

问题：“5.10.240 是否从 `start_kernel()` 启动所有子系统？”

1. 顶层 `Makefile` 证明版本和 `init/` 属于核心构建输入。
2. `init/main.c` 定义 `start_kernel()`，可以静态确认它是 C 侧核心初始化入口。
3. 继续检查具体子系统的 initcall，而不是说所有初始化都直接由它调用。
4. 若要证明某 initcall 在目标机执行，需匹配 `.config`、构建映像和启动日志/trace。

因此严谨结论是：“该源码树中 `start_kernel()` 是核心初始化入口；某子系统是否编入并执行需要额外的配置、链接与运行态证据。”

## Key Takeaways

- 先固定版本、架构、配置和符号，再解释代码。
- 静态源码、构建产物、运行态是三层不同强度的证据。
- 从符号和构建关系导航，比按目录漫游更高效。
- 结论应包含适用边界；无法证明的部分明确标为待验证。

## Source Anchors

- `$KERNEL_SRC/Makefile:2` — `VERSION = 5`。
- `$KERNEL_SRC/Makefile:3` — `PATCHLEVEL = 10`。
- `$KERNEL_SRC/Makefile:4` — `SUBLEVEL = 240`。
- `$KERNEL_SRC/Makefile:661` — `core-y := init/ usr/` 核心目录起点。
- `$KERNEL_SRC/Makefile:1163` — `core-y` 加入 `kernel/ mm/ fs/ ...`。
- `$KERNEL_SRC/init/main.c:848` — `start_kernel()` 定义。
- `$KERNEL_SRC/init/version.c:46` — `linux_banner` 版本横幅。
- `$KERNEL_SRC/include/linux/module.h:131` — 可加载模块场景的 `module_init()`。

## Connects To

下一章把“配置和构建归属”展开为可操作的 Kbuild/Kconfig 路径；第 3 章把证据边界落实到可审查补丁；第 4–6 章均沿用本章的定义—调用—配置—运行四层检查法。
