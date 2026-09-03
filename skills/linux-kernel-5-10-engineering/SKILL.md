---
name: linux-kernel-5-10-engineering
description: "Linux kernel engineering knowledge base grounded in the Linux 5.10.240 source tree. Use when implementing, reviewing, debugging, or testing kernel C code involving Kbuild/Kconfig, execution contexts, locking and RCU, memory/VFS/block/network/driver subsystems, syscalls/UAPI, security, tracing, or kernel test tooling."
---

<!-- argument-hint: [task, subsystem, API, bug symptom, or chapter number] -->

# Linux Kernel 5.10 Engineering

**Source**: Linux 5.10.240 (`Dare mighty things`) | **Maintainers**: Linux kernel community | **Files**: 70,654 | **Topics**: 18 | **Generated**: 2026-09-03

## How to Use This Skill

- **Without arguments** — apply the engineering workflow and safety gates below.
- **With a task** — describe the change, target architecture, configuration, entry path, and runtime context; load the relevant chapters before proposing code.
- **With an API or symbol** — find its declaration, implementation, representative callers, configuration guard, and tests in the live target tree.
- **With a chapter** — ask for `ch05` or another chapter to load the detailed rules, worked example, and source anchors.
- **For review/debugging** — state the observed evidence separately from source-derived possibilities.

Treat the current project tree as authoritative. Resolve the source root from
the repository being changed or an explicitly supplied `KERNEL_SRC`; never
guess a host-specific absolute path. The source anchors in this skill were
validated against Linux 5.10.240. Never assume a 5.10 API, `CONFIG_*` symbol,
callback signature, or lifetime rule is unchanged in another kernel release.

## Mandatory Kernel-Change Protocol

Before writing or approving kernel code:

1. **Bind the target** — identify kernel version, architecture, relevant
   `CONFIG_*` states (`y/m/n` where applicable), build output directory, and
   whether the tree has usable Git history.
2. **Name the entry and context** — syscall, IRQ, softirq/NAPI, workqueue,
   kthread, timer, probe/remove, file operation, or another callback; determine
   whether it may sleep and what locks/RCU read-side state are already held.
3. **Trace four layers** — documentation → public/internal header contract →
   implementation → representative call sites and tests. A declaration alone
   does not prove ownership, ordering, or callback context.
4. **Write a lifecycle ledger** — acquisition, publication, concurrent access,
   quiescing, unregister/cancel/flush, grace period, final put/free. Pair every
   successful step with its reverse-order rollback.
5. **Audit boundaries** — user pointers, lengths/overflow, credentials and LSM
   hooks, UAPI/compat ABI, architecture-specific code, DMA/device ownership, and
   observable ordering.
6. **Prefer an in-tree pattern** — start from the closest same-subsystem,
   same-context 5.10.240 caller. Do not transplant an API merely because its
   name looks suitable.
7. **Validate proportionally** — targeted build/test first, then affected
   config/architecture variants, static analysis, debug configurations, and
   runtime or fault-injection evidence where the claim requires it.

## Core Frameworks & Mental Models

### 1. Live-tree-first evidence

Use this skill as a navigation and reasoning framework, not as a substitute for
the target checkout. A correct answer names which facts came from documentation,
headers, implementation, callers, configuration, tests, build output, or
runtime evidence. When those layers disagree, investigate the disagreement
instead of averaging them.

### 2. The five-contract gate

Every kernel object or callback has five interacting contracts:

| Contract | Required question |
|---|---|
| Context | Can this path sleep, fault, recurse, or run in IRQ/NMI context? |
| Ownership | Who owns each reference, buffer, descriptor, page, skb, bio, or device? |
| Synchronization | Which lock, RCU domain, atomic state, barrier, or single-threading rule orders access? |
| Configuration | Under which `CONFIG_*`, architecture, built-in/module, SMP, and PREEMPT combinations does it exist? |
| Interface | Is it internal, exported to modules, UAPI, sysfs/procfs/debugfs, netlink, ioctl, or another persistent ABI? |

Do not start implementation while any applicable contract is unnamed.

### 3. Configuration is part of program semantics

Kconfig selects which declarations, objects, stubs, and call paths exist;
Kbuild decides what is compiled, linked built-in, or emitted as a module. Review
`y/m/n` behavior and dependency expressions together. A patch that works only
with the developer's `.config` is not yet a kernel patch.

### 4. Context chooses primitives

Choose synchronization and allocation only after identifying the execution
context. Mutexes and blocking allocation belong only where sleeping is legal.
IRQ-shared state may require `spin_lock_irqsave()`; softirq-shared state may
require BH exclusion; RCU read-side sections have their own blocking rules.
The most familiar primitive is not automatically the correct one.

### 5. Lifetime is a partial order

Think in “publish before discover; acquire a stable reference before leaving
the lookup protection; remove before waiting; wait before free.” Reference
counts prevent premature destruction but do not by themselves make lockless
lookup safe. RCU delays reclamation but does not replace object ownership.

### 6. Ordering protects observations, not source order

Compiler order, CPU order, lock order, device/DMA order, and userspace-visible
order are different. Use the subsystem's existing lock, atomic, barrier, and
accessor pattern. Never add a barrier without naming the paired operation and
the state transition it publishes.

### 7. Registration is a transaction

Treat probe, module init, filesystem/network-device registration, and resource
setup as staged transactions. Each stage becomes visible to a wider audience;
on failure, unwind only completed stages in strict reverse order. On teardown,
stop new entrants before draining asynchronous users and releasing storage.

### 8. Data paths are ownership machines

For `sk_buff`, `bio`, pages, requests, and work items, draw the handoff path and
mark the exact transfer point. After a successful transfer, the previous owner
must not access or free the object unless the API explicitly returns ownership.
Most “random” UAF, double-free, and lost-completion bugs are broken handoffs.

### 9. UAPI outlives the implementation

Syscalls, ioctl layouts, netlink attributes, sysfs/procfs text, trace formats,
and exported headers are compatibility commitments. Validate sizes, padding,
alignment, compat handling, copy semantics, and documentation before exposing
new state. Internal elegance does not justify silent ABI breakage.

### 10. Instrument before perturbing

Prefer existing tracepoints, ftrace, dynamic debug, lockdep, sanitizers, BPF
observability, and subsystem counters before scattering `printk()` calls.
Capture a baseline and choose evidence at the layer of the claim: source proves
possible paths; traces prove executed paths; tests prove only the exercised
inputs/configuration.

### 11. Validation is a matrix

Select dimensions that can change semantics: `CONFIG` state, built-in/module,
SMP/UP, PREEMPT, architecture/word size, debug instrumentation, success/error
path, unload/remove, and injected allocation/I/O failure. Start with the
smallest test that can falsify the change, then widen based on risk.

### 12. Small diffs preserve reviewability

Keep behavior changes, cleanup, generated updates, and interface changes
separable. Include the header that owns each used facility, follow local style,
reuse existing helpers, and avoid speculative abstractions. Reviewers must be
able to trace every new state transition and rollback edge.

## Task Routing

| Task | Load first |
|---|---|
| Add or repair a driver | ch04, ch05, ch06, ch12, ch13, ch18 |
| Diagnose lockup, race, UAF, or lost wakeup | ch04, ch05, ch06, ch13, ch17 |
| Change allocation, mmap, VMA, or fault handling | ch05, ch06, ch08, ch18 |
| Change filesystems or block I/O | ch06, ch09, ch10, ch13 |
| Change network receive/transmit or netdevice lifecycle | ch05, ch06, ch11, ch13 |
| Add a syscall, ioctl, netlink, sysfs, or UAPI field | ch03, ch14, ch16, ch18 |
| Change module init/unload or registration | ch06, ch12, ch15, ch18 |
| Investigate performance without a proven cause | ch01, subsystem chapter, ch17 |
| Prepare a patch for review | ch02, ch03, ch18 |

## Chapter Index

| # | Title | Key frameworks |
|---|---|---|
| [ch01](chapters/ch01-source-orientation.md) | Source orientation and evidence | version binding, four-layer trace, scope |
| [ch02](chapters/ch02-kbuild-kconfig-build.md) | Kbuild, Kconfig, and configuration | y/m/n, generated state, O= builds |
| [ch03](chapters/ch03-engineering-workflow-patches.md) | Engineering workflow and patch shape | local patterns, style, review surface |
| [ch04](chapters/ch04-execution-context-preempt-irq-sleep.md) | Execution contexts | sleepability, preemption, IRQ/BH |
| [ch05](chapters/ch05-locking-atomics-memory-order-lockdep.md) | Locking and memory ordering | lockdep, atomics, barriers, seqcount |
| [ch06](chapters/ch06-lifetime-refcount-rcu.md) | Lifetime, refcounts, and RCU | stable lookup, grace periods, teardown |
| [ch07](chapters/ch07-scheduler-tasks.md) | Scheduler and tasks | runqueues, states, wakeups |
| [ch08](chapters/ch08-memory-management.md) | Memory management | GFP, pages, VMAs, faults |
| [ch09](chapters/ch09-vfs-filesystems.md) | VFS and filesystems | inode/dentry/file, operations, mounts |
| [ch10](chapters/ch10-block-io.md) | Block I/O | bio, requests, blk-mq, completion |
| [ch11](chapters/ch11-networking-stack.md) | Networking | skb ownership, NAPI, netdevice, namespaces |
| [ch12](chapters/ch12-device-driver-model.md) | Device and driver model | bus match, probe/remove, PM, devres |
| [ch13](chapters/ch13-async-interrupts-work.md) | IRQ and deferred work | softirq, timers, workqueues, waits |
| [ch14](chapters/ch14-syscalls-uaccess-abi.md) | Syscalls, uaccess, and ABI | copy semantics, compat, UAPI |
| [ch15](chapters/ch15-modules-init-registration.md) | Modules, init, and unwind | initcalls, registration, unload |
| [ch16](chapters/ch16-credentials-capabilities-lsm.md) | Security, credentials, and LSM | subjective creds, capabilities, hooks |
| [ch17](chapters/ch17-observability-debugging.md) | Tracing, debugging, and BPF | tracepoints, ftrace, dynamic debug |
| [ch18](chapters/ch18-verification-tooling.md) | Verification tooling | KUnit, kselftest, sparse, Coccinelle |

## Topic Index

- **ABI / compat / ioctl / syscall / UAPI / uaccess** → ch14, ch16, ch18
- **atomic / barrier / lockdep / mutex / rwsem / seqcount / spinlock** → ch04, ch05
- **bio / blk-mq / block device / request** → ch10, ch13
- **BPF / dynamic debug / ftrace / printk / tracepoint** → ch17
- **build / Kbuild / Kconfig / module / toolchain** → ch02, ch15, ch18
- **completion / IRQ / softirq / tasklet / timer / waitqueue / workqueue** → ch04, ch13
- **credentials / capabilities / LSM / security hook** → ch14, ch16
- **device / devm / DMA / driver / PM / probe / remove** → ch06, ch12, ch13
- **dentry / file / filesystem / inode / mount / VFS** → ch06, ch09
- **GFP / mmap / page / page fault / slab / VMA** → ch04, ch06, ch08
- **kref / module reference / RCU / refcount / UAF** → ch05, ch06
- **kselftest / KUnit / checkpatch / Coccinelle / fault injection / sparse** → ch03, ch18
- **NAPI / net_device / namespace / skb / socket** → ch05, ch06, ch11, ch13
- **scheduler / task state / wakeup / runqueue / preemption** → ch04, ch07
- **source navigation / call chain / version / evidence** → ch01, ch03

## Supporting Files

- [glossary.md](glossary.md) — key terms and chapter routes
- [patterns.md](patterns.md) — reusable implementation and review patterns
- [cheatsheet.md](cheatsheet.md) — decision tables and fast safety checks

## Scope & Limits

This skill is synthesized from the local Linux 5.10.240 source snapshot, not
from commit history. The snapshot is not a Git checkout, so provenance,
backports, and commit intent cannot be inferred. Version identity comes from
the root Makefile (`5.10.240`); reference fingerprints are:

- `Makefile`: `0203d7a1a1e93768f27dff2cca2ee1903770d2a1f1ca264757f5252c582ea029`
- `Kconfig`: `a592dae7d067cd8e5dc43e3f9dc363eba9eb1f7cf80c6178b5cd291c0b76d3ec`
- `Kbuild`: `75df66064f75e91e6458862cd9413b19e65b77eefcc8a95dcbd6bf36fd2e4b59`
- `README`: `bad58d396f62102befaf23a8a2ab6b1693fdc8f318de3059b489781f28865612`

Line numbers in chapter anchors are navigation hints for this snapshot and may
shift in patched trees. Commands are recipes, not claims that this machine has
successfully built or booted the kernel. For upstream policy, active CVEs,
maintainer assignments, or APIs in newer releases, verify current primary
sources before acting.
