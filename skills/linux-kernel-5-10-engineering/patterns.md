# Engineering Patterns

## Four-layer source triangulation

**When to use**: Before changing or explaining an unfamiliar kernel API.

**How**:
1. Read the relevant `Documentation/` contract.
2. Find the owning declaration and configuration guards.
3. Trace the implementation and every state transition it performs.
4. Inspect several same-subsystem callers plus tests and error paths.
5. Record facts that remain version-, architecture-, or config-dependent.

**Trade-offs**: Slower than copying the first caller, but exposes implicit context, ownership, and teardown requirements.

## Context-first primitive selection

**When to use**: Choosing an allocator, lock, wait, or deferred-execution mechanism.

**How**: Classify NMI/hardirq/softirq/process context; note disabled IRQ/BH/preemption state and held locks; decide whether faulting or sleeping is legal; then select the in-tree primitive used by equivalent callers.

**Trade-offs**: A more restrictive primitive may work but harm latency; a blocking primitive in atomic context is incorrect.

## Lock-order ledger

**When to use**: A path acquires two or more lock classes or crosses callbacks.

**How**: Write `L1 → L2` edges for success and error paths, including locks acquired inside helpers; include IRQ-safe/unsafe classification; compare with lockdep splats and existing subsystem nesting annotations.

**Trade-offs**: Manual ledgers require upkeep, but make ABBA cycles and callback re-entry visible before runtime.

## Stable lookup plus reference acquisition

**When to use**: An object is found in a shared list, hash, xarray, IDR, or RCU-protected structure.

**How**: Hold the lookup protection; reject dying/zero-ref objects; acquire the supported reference while the object is stable; release lookup protection; operate; finally put. For RCU lookup, use the object's documented RCU/refcount pattern rather than composing APIs by intuition.

**Trade-offs**: Reference acquisition adds contention; omitting it creates UAF windows that may survive ordinary testing.

## Remove, quiesce, reclaim

**When to use**: Driver remove, module exit, object deletion, or failed registration.

**How**:
1. Mark the object unavailable and remove public lookup/registration points.
2. Stop new IRQ, timer, work, NAPI, callback, and userspace entrants.
3. Synchronize or flush each asynchronous domain.
4. Wait for RCU grace periods and outstanding references where required.
5. Release child resources, mappings, and storage in reverse order.

**Trade-offs**: Conservative synchronization increases teardown latency but prevents callbacks from entering freed code or memory.

## Reverse-order error unwind

**When to use**: Initialization has multiple fallible acquisition or registration stages.

**How**: Give every successful stage one cleanup edge; jump only to labels that undo completed stages; order labels as the exact reverse of acquisition; keep the primary error code; make cleanup safe for partial state.

**Trade-offs**: More labels than a generic cleanup block, but far less risk of double-free, leaked registration, or cleanup of uninitialized state.

## Publish state with an explicit pair

**When to use**: Lockless producer/consumer state, ring indices, flags, sequence counters, or waitqueue optimizations.

**How**: Name the data written before publication, the publishing store/unlock/barrier, the acquiring load/lock/barrier, and what observation it guarantees. Use `READ_ONCE()/WRITE_ONCE()` only for access properties they actually provide.

**Trade-offs**: Extra barriers can reduce performance; missing or unpaired barriers produce architecture-dependent failures.

## Condition-loop waiting

**When to use**: Waiting for mutable state that can change before or after wakeup.

**How**: Define a predicate over protected state; enqueue and set task state through a wait-event API or the documented prepare/finish loop; recheck after every wake; handle signal/timeout returns; update the predicate before waking with the required ordering.

**Trade-offs**: Predicate loops tolerate spurious wakeups; badly chosen predicates can still miss ownership transitions.

## Data-path ownership handoff

**When to use**: Passing `skb`, `bio`, request, page, DMA mapping, or work item to another layer.

**How**: For each call, mark ownership as borrowed, shared by reference, or transferred; mark success and failure separately; after transfer, clear or stop using the old pointer; define the completion/drop/free endpoint.

**Trade-offs**: Extra documentation during review, but prevents double-free, leaks, and use-after-submit bugs.

## UAPI copy-in/validate/execute/copy-out

**When to use**: Syscall, ioctl, netlink, proc/sysfs, or another userspace boundary.

**How**: Copy a fixed header or validated length into kernel memory; reject reserved flags, overflow, and inconsistent sizes; translate compat layouts explicitly; execute using kernel-owned state; zero padding and copy out only initialized fields; document the ABI.

**Trade-offs**: Versioned structures cost code, but preserve forward/backward compatibility and prevent TOCTOU on user memory.

## Configuration matrix

**When to use**: Code is guarded by Kconfig, can be built as a module, or depends on architecture/SMP/PREEMPT.

**How**: Test or reason through applicable `y/m/n` states, dependency combinations, built-in/module linkage, at least one restrictive config, and affected architectures. Inspect stub behavior when a symbol is disabled.

**Trade-offs**: Matrix size grows quickly; select dimensions tied to the changed semantics rather than running arbitrary configurations.

## Instrument-then-change

**When to use**: Root cause is not proven or the path is timing-sensitive.

**How**: State a falsifiable hypothesis; choose existing tracepoint/counter/lockdep/sanitizer evidence; capture baseline and failure; make the smallest semantic change; rerun the same observation; remove temporary instrumentation.

**Trade-offs**: Requires deliberate evidence collection, but avoids patches that hide symptoms or introduce printk-driven timing changes.

## Targeted-to-broad verification

**When to use**: Any kernel change before claiming completion.

**How**: Run the narrow object/subsystem build and nearest test; inspect warnings; expand to relevant configs and architecture builds; run sparse/checkpatch/Coccinelle as applicable; exercise error, unload, concurrency, and fault-injection paths; distinguish build, boot, runtime, and device evidence.

**Trade-offs**: Broad validation is expensive; the risk model must justify which dimensions are mandatory and which remain gaps.
