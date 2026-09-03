# 第 2 章：Kbuild、Kconfig 与可复现构建

## Core Idea

Kconfig 决定“允许选择什么以及最终选择了什么”，Kbuild 决定“选中的内容如何编译、链接成什么”。`.config` 是用户选择的主记录，但编译实际消费的是同步生成的 `include/config/auto.conf` 与 `include/generated/autoconf.h` 等产物。工程上必须把源码目录、输出目录、架构、工具链和配置文件作为一个整体记录。

5.10.240 的顶层 `Makefile` 明确：`O=` 优先于 `KBUILD_OUTPUT`；`KCONFIG_CONFIG` 默认是 `.config`；`include/generated/autoconf.h` 等目标依赖配置并通过 `syncconfig` 更新。由此可知，复制一个 `.config` 后直接解释产物仍不够，还要确认同步和完整构建成功。

## Frameworks Introduced

### 配置闭包

- **When**：新增选项、裁剪内核、复现构建差异时。
- **How**：从目标 `CONFIG_FOO` 追到 `depends on`、`select/imply`、类型与默认值，再追到 `obj-$(CONFIG_FOO)` 和 `#ifdef`。
- **Why**：菜单是否可见、配置值、对象是否链接是不同问题。
- **Failure**：手改 `.config` 后未运行 Kconfig 前端，依赖会把值改回或构建文件保持过期。

### 隔离输出构建

- **When**：并行维护多架构、多配置或需要保持源码树干净时。
- **How**：每组 `(ARCH, toolchain, config)` 使用独立 `O=` 目录，所有后续 make 命令都带同一 `O=`。
- **Why**：避免生成头文件和旧对象交叉污染。
- **Failure**：配置时使用 `O=out`，构建时漏掉 `O=`，实际操作了另一个 `.config`。

## Key Concepts

- **三态语义**：布尔值是 `y/n`，tristate 还可为 `m`；`m` 是否有效还受 `MODULES` 总开关影响。
- **依赖方向**：`depends on` 限制当前符号；`select` 强制另一个符号，容易绕过被选符号自身依赖，应谨慎使用。
- **对象归属**：`obj-y` 进入内建，`obj-m` 形成模块；目录级 Kbuild 逐层聚合到 `vmlinux` 或模块。
- **配置迁移**：`olddefconfig` 以旧配置为基准，对新符号取默认值；`oldconfig` 会交互询问；`savedefconfig` 生成最小差异配置，不能替代完整构建记录。
- **架构与交叉编译**：`ARCH` 选择目标架构目录，`CROSS_COMPILE` 是工具名前缀；两者不是同一个维度。

## Mental Models

把构建看成两阶段编译器：Kconfig 将规则和用户意图编译成配置状态；Kbuild 再把“源码 + 配置状态 + 工具链”编译成产物。任何缓存命中都只在输入身份一致时可信。

调试配置问题时使用链条：**符号定义 → 可见性/依赖 → `.config` 值 → `auto.conf/autoconf.h` → Makefile 对象 → 链接产物**。链条在哪一段断，修复就落在哪一层。

## Anti-patterns

- 用 `sed` 强改 `.config`，忽略依赖求值；应使用 Kconfig 前端或 `scripts/config` 后再 `olddefconfig`。
- 把 `CONFIG_FOO=y` 当作对象必然存在，未检查 Kbuild 条件和链接失败。
- 在源码树内混合 x86、arm64 输出，随后把陈旧头文件解释成源码缺陷。
- 只保存 `.config`，不记录编译器版本、`ARCH`、`CROSS_COMPILE` 和源码标识。
- 将 `menuconfig` 中不可见误判为符号不存在；可能是依赖未满足。

## Commands & APIs

```bash
export KERNEL_SRC=/path/to/linux-5.10.240
export KERNEL_OUT=/tmp/linux-5.10-out
make -C "$KERNEL_SRC" O="$KERNEL_OUT" ARCH=x86 defconfig
make -C "$KERNEL_SRC" O="$KERNEL_OUT" ARCH=x86 olddefconfig
make -C "$KERNEL_SRC" O="$KERNEL_OUT" ARCH=x86 -j"$(nproc)"
```

调查命令：

```bash
rg -n '^config FOO|^menuconfig FOO' "$KERNEL_SRC"
rg -n 'CONFIG_FOO' "$KERNEL_SRC" --glob 'Makefile' --glob 'Kbuild'
rg -n '^CONFIG_FOO=' "$KERNEL_OUT/.config"
```

注意：上面是建议工作流，不代表本章对当前环境执行过构建。生产复现还应保存 `make V=1` 的关键命令或构建日志。

## Worked Example：为什么 `CONFIG_MODULES=n` 时驱动不能是 `m`

先从 `init/Kconfig` 找 `menuconfig MODULES`。一个驱动即便声明为 tristate，其 `m` 语义仍依赖模块设施；Kconfig 会在求值时限制可选范围。接着检查对应目录中的 `obj-$(CONFIG_DRIVER) += driver.o`：值为 `y` 时内建，为 `m` 时模块，为 `n` 时不参与。

验证应分层：配置层检查输出目录 `.config`；生成层检查 `auto.conf`；构建层检查 `.ko` 或 `modules.order`；运行层再用目标系统的模块信息。不能因为源码中存在 `module_init()` 就断言产生了 `.ko`。

## Key Takeaways

- Kconfig 管状态，Kbuild 管产物；二者必须串起来阅读。
- 始终固定并复用同一个 `O=` 输出目录。
- `ARCH`、工具链、完整配置和源码标识共同定义一次构建。
- 对配置问题按闭包逐层查证，不直接编辑生成文件。

## Source Anchors

- `$KERNEL_SRC/Makefile:120` — `O=` 输出目录说明。
- `$KERNEL_SRC/Makefile:127` — `O=` 优先于 `KBUILD_OUTPUT`。
- `$KERNEL_SRC/Makefile:404` — `KCONFIG_CONFIG ?= .config`。
- `$KERNEL_SRC/Makefile:730` — `auto.conf/autoconf.h` 配置依赖目标。
- `$KERNEL_SRC/Makefile:732` — 调用 `syncconfig`。
- `$KERNEL_SRC/scripts/kconfig/Makefile:62` — `syncconfig` 是内部实现细节。
- `$KERNEL_SRC/scripts/kconfig/Makefile:75` — `savedefconfig` 目标。
- `$KERNEL_SRC/scripts/kconfig/Makefile:87` — `%_defconfig` 规则。
- `$KERNEL_SRC/init/Kconfig:2090` — `menuconfig MODULES`。

## Connects To

第 1 章提供版本与证据坐标；第 3 章将配置变更拆成可审查补丁；第 4、5 章中的 PREEMPT、LOCKDEP 等行为都必须回到本章的配置闭包确认。
