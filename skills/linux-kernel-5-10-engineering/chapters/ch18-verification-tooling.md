# 第 18 章 验证工具链：KUnit、kselftest、静态分析、配置矩阵与故障注入

## Core Idea

内核改动的“通过编译”只证明某一配置、某一编译器路径的语法和链接成立。可信验证需要分层：KUnit 锁定内核内纯逻辑与边界条件；kselftest 从用户态验证 ABI 和系统行为；checkpatch 检查提交/风格启发式问题；sparse 检查地址空间和类型语义；Coccinelle 查找跨树 API 模式；配置矩阵暴露条件编译；fault injection 强迫稀有错误路径执行。

验证计划应从风险反推，而不是机械跑全套。修改 UAPI 就覆盖错误码、旧结构和 compat；修改锁与引用就启用对应 debug 配置并加并发压力；修改分配/注册路径就注入失败，确认逆序回退。每个结论应标明是静态、构建、测试还是真机运行证据。

## Frameworks Introduced

### 1. KUnit 与 kselftest

- **When**：KUnit 用于可隔离的内核函数、解析器和状态机；kselftest 用于 syscall、procfs/sysfs、网络和真实用户态交互。
- **How**：KUnit 以 `KUNIT_CASE` 组成 suite，用 `kunit_test_suite()` 注册；kselftest 子目录按 `lib.mk` 约定声明 `TEST_PROGS/TEST_GEN_PROGS` 并输出 TAP/ksft 结果。
- **Why**：分别覆盖内核内白盒和用户态黑盒边界。
- **Failure**：KUnit 测试依赖真实全局状态而不清理；kselftest 把环境缺失记作 FAIL 而非 SKIP；断言只验证“未崩溃”。

### 2. checkpatch、sparse 与 Coccinelle

- **When**：提交前查常见样式/提交信息；编译期查 `__user`、bitwise 等语义；跨大量文件找错误 API 模式。
- **How**：`checkpatch.pl --strict` 针对 patch；`make C=1` 检查重编文件、`C=2` 检查目标全部文件；`make coccicheck MODE=report COCCI=...` 运行 SmPL。
- **Why**：三者覆盖启发式文本、编译语义和结构化程序模式。
- **Failure**：把 checkpatch clean 当正确性；忽略 sparse warning；直接用 Coccinelle patch 模式大范围改树且不逐项审查。

### 3. 配置矩阵与 fault injection

- **When**：代码受 `#ifdef`、模块/内建、架构或 debug 配置影响；错误路径难以自然触发。
- **How**：合并小 config fragment，至少覆盖 feature on/off、`=y/=m` 与关键架构；fault injection 通过 debugfs 的 probability/interval/times/space 或 subsystem 参数控制 `should_fail()`。
- **Why**：条件编译与失败恢复是内核回归高发区。
- **Failure**：只测发行版 config；随机概率导致不可复现；注入后忘记归零污染后续测试。

## Key Concepts

- **证据分层**：checkpatch/sparse 是静态证据，编译是配置证据，KUnit/kselftest 是行为证据，硬件复现是运行期证据，不能互相替代。
- **最小证明**：先跑能直接证明改动契约的 targeted test，再扩展到相关子系统与矩阵。
- **负路径优先**：错误回退、超时、取消、重复调用和资源耗尽通常比 happy path 更能揭示生命周期 bug。
- **可复现注入**：诊断时优先 `probability=100` 配合有限 `times` 和精确 task/filter，避免概率噪声。
- **SKIP 语义**：环境不支持时输出 SKIP；只有行为违约才 FAIL。

## Mental Models

把验证看成二维矩阵：横轴是配置/架构/编译器，纵轴是正常、边界、错误、并发和卸载路径。无需穷举所有格子，但每个改动的高风险交点必须有证据。

把工具当不同传感器：checkpatch 会误报也会漏报，sparse 只看它建模的类型语义，Coccinelle 匹配取决于 SmPL，测试只覆盖执行到的路径。多个独立传感器一致，置信度才提高。

## Anti-patterns

- 只跑 `allmodconfig` 编译，就声称运行行为正确。
- 为了 checkpatch 零告警修改无关历史代码，扩大补丁和回归面。
- KUnit 测试复制实现算法，二者同错仍通过。
- kselftest 依赖 root、工具或设备，却没有 feature probe 和 SKIP。
- `make coccicheck MODE=patch` 后不审查 diff、不重新编译。
- 故障注入使用随机概率、无清理 trap，导致后续结果不可解释。

## Commands & APIs

```sh
# patch/style（针对实际 patch 文件）
scripts/checkpatch.pl --strict 0001-change.patch

# 语义静态分析
make C=1 CHECK=sparse M=drivers/example
make coccicheck MODE=report COCCI=scripts/coccinelle/api/err_cast.cocci

# 用户态回归：每个 TARGET 分开构建、运行
make -C tools/testing/selftests TARGETS=size FORCE_TARGETS=1 all
make -C tools/testing/selftests TARGETS=size FORCE_TARGETS=1 run_tests
make -C tools/testing/selftests TARGETS=timers FORCE_TARGETS=1 all
make -C tools/testing/selftests TARGETS=timers FORCE_TARGETS=1 run_tests

# 配置片段
scripts/kconfig/merge_config.sh -m .config test.config
make olddefconfig
```

KUnit 的具体命令取决于树内工具和配置；5.10 文档的常见入口是 `./tools/testing/kunit/kunit.py run`。所有命令都应记录 config、架构、编译器和退出码。

## Worked Example

对 page allocation 做确定、有限且可清理的失败注入：

```sh
FAIL=/sys/kernel/debug/fail_page_alloc
cleanup() {
	echo 0 > "$FAIL/probability"
	echo N > "$FAIL/task-filter"
}
trap cleanup EXIT INT TERM
echo 0 > "$FAIL/probability"
echo 1 > "$FAIL/times"
echo 0 > "$FAIL/space"
echo Y > "$FAIL/task-filter"
echo N > "$FAIL/ignore-gfp-wait"
echo 0 > "$FAIL/min-order"
echo 100 > "$FAIL/probability"
bash -c 'echo 1 > /proc/self/make-it-fail; exec ./targeted-test'
```

这里先保持 `probability=0` 完成所有参数设置，再最后启用注入；`task-filter=Y` 与子进程的 `make-it-fail` 将范围收窄。`ignore-gfp-wait=N`、`min-order=0` 覆盖普通 order-0 可睡眠页分配。测试不仅要期待 `-ENOMEM`，还要检查已注册资源、引用计数和后续重试是否恢复。

## Key Takeaways

1. 从风险选择工具，先 targeted contract test，再扩展配置和范围。
2. 静态、构建、测试和运行证据必须明确区分。
3. 配置关闭、模块化和错误注入路径是一等验证对象。
4. 自动修复输出必须人工审查并重新编译/测试。

## Source Anchors

- `$KERNEL_SRC/include/kunit/test.h:156` — `KUNIT_CASE()`；`$KERNEL_SRC/include/kunit/test.h:294` — `kunit_test_suites()`。
- `$KERNEL_SRC/lib/kunit/test.c:353` — `kunit_run_tests()`；`$KERNEL_SRC/Documentation/dev-tools/kunit/start.rst:170` — expectation 示例。
- `$KERNEL_SRC/tools/testing/selftests/lib.mk:21` — common `run_tests` 约定；`$KERNEL_SRC/Documentation/dev-tools/kselftest.rst:73` — `TARGETS`。
- `$KERNEL_SRC/tools/testing/selftests/Makefile:85` — `FORCE_TARGETS` 强制每个目标构建成功。
- `$KERNEL_SRC/scripts/checkpatch.pl:55` — checkpatch 配置入口。
- `$KERNEL_SRC/Documentation/dev-tools/sparse.rst:96` — `make C=1/C=2` 区别。
- `$KERNEL_SRC/scripts/coccicheck:108` — `MODE`；`$KERNEL_SRC/Documentation/dev-tools/coccinelle.rst:154` — 单个 `COCCI`。
- `$KERNEL_SRC/scripts/kconfig/merge_config.sh:4` — config fragment 合并脚本。
- `$KERNEL_SRC/lib/fault-inject.c:103` — `should_fail()`；`$KERNEL_SRC/Documentation/fault-injection/fault-injection.rst:61` — debugfs 控制字段；`$KERNEL_SRC/Documentation/fault-injection/fault-injection.rst:99` — task filter。

## Connects To

- 第 13 章：超时、取消与异步析构需要并发和故障路径测试。
- 第 14 章：kselftest 是系统调用/UAPI/compat 的主要回归层。
- 第 15 章：逐个 acquisition 点注入失败，验证 init 回退与 unload。
- 第 16 章：user namespace、capability 和 LSM 组合需要权限矩阵。
- 第 17 章：trace 可解释失败，但不能代替明确断言和退出码。
