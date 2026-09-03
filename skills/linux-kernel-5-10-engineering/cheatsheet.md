# Kernel Engineering Cheatsheet

## First five questions

| Gate | Write down before coding |
|---|---|
| Target | exact version/tree, arch, config, built-in or module |
| Entry | callback/entry symbol and full caller path |
| Context | may sleep/fault? IRQ/BH/preemption state? locks held? |
| Lifetime | acquire → publish → quiesce → drain → final free |
| Interface | internal/exported/UAPI plus compatibility obligations |

## Context decision

| Context | Sleep? | Typical choices | Immediate smell |
|---|---:|---|---|
| NMI | No | NMI-safe, lockless/per-CPU primitives only | ordinary spin/mutex, allocation, printk assumptions |
| Hard IRQ | No | short `spin_lock_irqsave` region, defer work | mutex, blocking wait, `GFP_KERNEL` |
| Softirq/NAPI | No | BH-safe spin locking, per-CPU, schedule work | sleeping or holding lock across slow path |
| Process, atomic state | No | spin/atomic/per-CPU; leave atomic state first | faulting user access or blocking allocation |
| Process, sleepable | Usually | mutex/rwsem, completion/waitqueue, `GFP_KERNEL` | assuming callbacks cannot race while asleep |
| Workqueue/kthread | Usually | blocking operations if worker contract permits | freeing owner without cancel/flush/stop |

## Lock and read-mostly choice

- Need mutual exclusion and may sleep → mutex/rwsem.
- Atomic or IRQ path, tiny critical section → spinlock with the variant required by shared contexts.
- Read-mostly traversal with deferred reclamation → established RCU pattern.
- Consistent scalar snapshot, readers can retry, writers serialized → seqcount/seqlock.
- Counter only → atomic/refcount API; it does not protect surrounding fields.
- Two locks → record global order before implementation; use lockdep, not timing luck.

## Allocation flags

| Constraint | Start from | Check |
|---|---|---|
| Normal sleepable path | `GFP_KERNEL` | reclaim recursion and locks |
| Atomic/IRQ path | `GFP_ATOMIC` | preallocate if failure is unacceptable |
| Filesystem reclaim recursion risk | `GFP_NOFS` | can allocation move outside the lock/path? |
| Block-I/O reclaim recursion risk | `GFP_NOIO` | can resources be reserved earlier? |

Do not “fix” failures by changing GFP flags without proving the context.

## Teardown order

`hide/unregister → reject new work → disable IRQ/NAPI/timer → cancel/flush work → synchronize callbacks/RCU → drop references → unmap/free`

- `cancel_*_sync()`/`flush_*()` must run from a context where waiting is legal.
- A refcount reaching zero selects final release; it does not prove no lockless lookup can still find the object.
- Module/device code must not unload while callbacks can still execute its text.

## Boundary rules

- User pointers: copy once, validate lengths/overflow/flags, operate on kernel memory, initialize all copied-out bytes.
- `copy_from_user()` can return bytes not copied; do not treat nonzero as success.
- New UAPI: document ABI, padding, compat behavior, error codes, and extensibility.
- DMA/skb/bio: mark the exact ownership-transfer call and completion/free endpoint.
- Barriers: name producer, consumer, published fields, and pairing primitive.

## Evidence ladder

| Claim | Minimum useful evidence |
|---|---|
| API exists | target-tree declaration and config guard |
| Semantics/lifetime | docs + implementation + representative callers |
| Compiles | fresh target/config build output |
| Runs | boot/module/test execution on named kernel |
| Fixes bug | reproduced failure, same observation after patch |
| Works on hardware | named device/arch runtime evidence |

## Validation selector

- Local C change → affected object/subsystem build, `checkpatch.pl`, sparse where annotations matter.
- Kconfig/Kbuild → `y/m/n` and restrictive configs; out-of-tree `O=` build.
- Concurrency/lifetime → lockdep/RCU debug/KASAN as applicable plus teardown/error-path stress.
- UAPI → selftest, compat/word-size review, installed-header check, ABI docs.
- Semantic patch → `make coccicheck MODE=report` first; inspect every match.
- Large change → KUnit for isolated logic, kselftest for running-kernel behavior, cross-arch build.

## Fast smells

- Lock chosen before context is known.
- Lookup returns a pointer without a stable reference.
- Error label frees a resource acquired after that label.
- `unregister` is followed immediately by `kfree` despite async users.
- Barrier has no documented counterpart.
- New `CONFIG` lacks disabled stub or `y/m/n` reasoning.
- Successful submit/transmit is followed by touching the transferred object.
- Source inspection is reported as runtime proof.
