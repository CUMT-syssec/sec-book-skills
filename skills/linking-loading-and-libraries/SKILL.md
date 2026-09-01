---
name: linking-loading-and-libraries
description: "Knowledge base from \"程序员的自我修养：链接、装载与库\" by 俞甲子、石凡、潘爱民. Use when diagnosing compilation, object-file, symbol, relocation, executable loading, dynamic linking, ABI, runtime, memory, system-call, ELF, PE/COFF, DLL, or Mini CRT problems."
---

<!-- argument-hint: [topic, framework name, or chapter number] -->

# 程序员的自我修养：链接、装载与库

**作者**：俞甲子、石凡、潘爱民 | **页数**：约 485 | **章节**：13 | **生成日期**：2026-09-01

## How to Use This Skill

- 无参数：加载下面的核心分析框架。
- 给出故障或现象：先定位它属于编译、链接、装载、运行库还是内核边界，再读取对应章节。
- 给出术语：从 Topic Index 找到章节后读取该文件，不凭术语印象作答。
- 给出 `chNN`：深入该章，结合目标平台和工具输出解释。

本技能来自扫描版 OCR。概念和决策框架经过综合重述；精确命令、字段值、结构偏移和代码语法必须结合当前工具链文档或实际二进制复核。

## Core Frameworks & Mental Models

### 1. 沿程序生命周期定位问题

把“源代码到运行”的黑盒拆成连续阶段：

`预处理 → 编译 → 汇编 → 目标文件 → 静态链接 → 可执行文件 → 装载/映射 → 动态链接 → CRT 初始化 → main → 系统调用/API`

遇到故障时先问“第一个与预期不一致的阶段在哪里”，再选择证据：预处理结果、汇编、节表、符号表、重定位表、链接映射、装载映射、动态链接器日志、调用栈。不要只盯最终错误信息。

### 2. “增加中间层”的万能法则

用中间层解决耦合、复用和兼容问题，同时记住每层都会引入自己的契约和失败模式。虚拟地址隔离进程与物理内存；目标文件隔离编译与最终布局；符号隔离定义与地址；PLT/GOT 隔离调用点与运行时地址；运行库隔离语言语义与系统调用；Windows API 再隔离应用与内核接口。

### 3. 区分名字、地址与内容

符号是名字，重定位记录描述“哪里需要在以后修正”，段/节保存内容，链接器给它们分配最终地址。分析 undefined reference、重复定义或错误跳转时，不要把“符号存在”“符号可见”“符号解析到正确实体”“重定位已正确应用”混成一件事。

### 4. 用“视图”解释文件与进程

同一 ELF 可以有链接视图和执行视图：链接器关心 section，装载器关心 segment。装载的核心通常是把文件区间映射到虚拟地址空间并建立权限，而不是简单复制整个文件。解释磁盘大小、内存占用、BSS、共享页或权限时，先明确所用视图。

### 5. 接口稳定性由 ABI 决定

源代码兼容不等于二进制兼容。调用约定、名称修饰、对象布局、异常机制、符号版本、导入导出和运行库选择共同构成 ABI。跨模块尤其是 C++ 边界出现问题时，优先收窄到稳定的 C ABI 或显式版本化接口。

### 6. 静态与动态链接是约束交换

静态链接用更大的文件和更新成本换取部署独立性；动态链接用运行时搜索、版本管理和重定位复杂度换取共享、升级和模块化。选择时同时检查空间、启动成本、热更新、隔离、兼容性和可复现部署，不用“动态一定更先进”替代权衡。

### 7. 地址无关代码把变化推迟到运行时

共享对象不能假设固定装载地址。代码通过 PC-relative、GOT、PLT 和重定位记录间接访问最终地址；lazy binding 又把部分解析推迟到第一次调用。遇到首次调用异常、符号劫持、非预期绑定或文本重定位时，沿 `调用点 → PLT → GOT → 动态链接器 → 定义` 检查。

### 8. 运行环境是共同产物

程序启动不是“内核直接调用 main”。内核/装载器建立映射和初始栈，入口代码初始化 CRT、堆、I/O、参数、环境、TLS、全局构造，才调用 main；返回后还要执行析构和退出回调。启动前崩溃与退出阶段崩溃应检查入口函数、运行库和构造/析构链，而不是业务 main。

### 9. 栈、堆与虚拟内存分层分析

栈由调用约定和函数活动记录组织；堆由分配器在操作系统提供的地址区间上管理；虚拟内存由内核和 MMU 提供映射与保护。内存故障先判断是非法映射/权限、栈帧破坏、堆元数据破坏、生命周期错误还是分配策略问题。

### 10. 自底向上重建以验证理解

Mini CRT 的价值是把入口、堆、I/O、格式化、退出、C++ new/delete、string 与全局构造串起来。学习或排障时可以构造“最小可运行链”，逐层添加能力；每加入一层都用可执行证据验证，而不是只背结构名。

## Chapter Index

| # | Title | Key Frameworks |
|---|---|---|
| [ch01](chapters/ch01-foundations.md) | 温故而知新 | 中间层、虚拟内存、线程 |
| [ch02](chapters/ch02-compilation-and-linking.md) | 编译和链接 | 构建流水线、编译器前后端 |
| [ch03](chapters/ch03-object-files.md) | 目标文件里有什么 | ELF/COFF、节、符号表 |
| [ch04](chapters/ch04-static-linking.md) | 静态链接 | 空间分配、符号解析、重定位 |
| [ch05](chapters/ch05-pe-coff.md) | Windows PE/COFF | PE、数据目录、调试信息 |
| [ch06](chapters/ch06-loading-and-process.md) | 可执行文件的装载与进程 | 映射、页错误、进程地址空间 |
| [ch07](chapters/ch07-dynamic-linking.md) | 动态链接 | PIC、GOT/PLT、lazy binding |
| [ch08](chapters/ch08-linux-shared-libraries.md) | Linux 共享库的组织 | SO-NAME、版本、搜索路径 |
| [ch09](chapters/ch09-windows-dynamic-linking.md) | Windows 下的动态链接 | DLL、导入导出、rebasing |
| [ch10](chapters/ch10-memory.md) | 内存 | 栈、调用约定、堆分配器 |
| [ch11](chapters/ch11-runtime-library.md) | 运行库 | CRT 入口、I/O、TLS、构造析构 |
| [ch12](chapters/ch12-system-calls-and-api.md) | 系统调用与 API | 特权边界、Linux syscall、Windows API |
| [ch13](chapters/ch13-mini-crt.md) | 运行库实现 | Mini CRT、最小运行链 |

## Topic Index

- **ABI / 调用约定** → ch03, ch04, ch09, ch10
- **BSS / section / segment** → ch03, ch06
- **CRT / main 之前与之后** → ch11, ch13
- **DLL / PE / COFF** → ch05, ch09
- **ELF / 符号表 / 重定位** → ch03, ch04, ch06, ch07
- **GOT / PLT / PIC / lazy binding** → ch07
- **SO-NAME / 共享库版本** → ch08
- **堆 / 栈 / TLS** → ch10, ch11, ch13
- **系统调用 / Windows API** → ch12
- **编译流水线 / 链接错误** → ch02, ch04
- **虚拟地址 / 页映射 / 装载** → ch01, ch06

## Supporting Files

- [glossary.md](glossary.md) — 核心术语
- [patterns.md](patterns.md) — 可复用分析方法
- [cheatsheet.md](cheatsheet.md) — 排障决策表与命令提示

## Scope & Limits

本技能覆盖书中以 32 位 x86、Linux、Windows、ELF、PE/COFF、GCC 4.1.2、binutils 2.18、glibc 2.6.1、Visual C++ 2005/2008 为主的机制。核心原理仍有价值，但现代 x86-64、ASLR、PIE、RELRO、TLS、异常展开、现代链接器和平台安全机制必须结合当前资料验证。
