# Chapter 32: System Management Mode

## Core Idea

SMM is a processor-managed execution environment entered by SMI, with state saved in SMRAM and restored by `RSM`. It is a platform security boundary with special memory protection, multiprocessor, VMX, and trace interactions—not merely a more privileged interrupt handler.

## Frameworks Introduced

- **SMM transition lifecycle**:
  1. SMI is recognized subject to blocking rules.
  2. Processor enters SMM and saves architectural state in the SMRAM state-save map.
  3. Handler runs from the SMBASE-derived entry environment.
  4. Handler preserves platform invariants and services the event.
  5. `RSM` validates/restores state and exits SMM.
- **SMRAM protection model**: Establish SMRAM placement, cacheability, SMRR coverage, lock state, and access policy before treating secrets or control flow as isolated.
- **Multiprocessor coordination**: Define which logical processors receive SMI, how rendezvous occurs, how nested/back-to-back SMIs are handled, and how shared SMM state is synchronized.
- **VMX/SMM treatment choice**:
  - Default treatment follows architectural SMI/RSM behavior across VMX contexts.
  - Dual-monitor treatment uses an SMM-transfer monitor and additional VMCS transitions, checks, and SMI-blocking state.

## Key Concepts

- **SMI**: System-management interrupt that triggers SMM entry when not blocked.
- **SMRAM**: Memory containing SMM handler code/data and processor state-save area.
- **SMBASE**: Per-processor base used to locate SMM entry and state-save structures.
- **SMRR**: System-management range registers used to protect/cache-control SMRAM on supporting systems.
- **RSM**: Instruction restoring saved state and leaving SMM.
- **Auto halt restart**: State-save behavior controlling return around a prior `HLT`.
- **I/O instruction restart**: Mechanism allowing selected interrupted I/O operations to restart after `RSM`.
- **STM**: SMM-transfer monitor used by dual-monitor treatment.

## Mental Models

- Treat SMM as a small firmware security domain with its own entry ABI and protected memory.
- Treat every saved-state field as untrusted input until platform provenance and ranges are validated; mutable state-save data can redirect post-`RSM` execution.
- Use a rendezvous protocol when system-wide state is mutated; one CPU in SMM does not automatically quiesce all others.

## Anti-patterns

- **Assuming ring-0 isolation protects SMRAM**: SMRAM requires platform/hardware protection and locking.
- **Leaving SMBASE/SMRAM relocatable or writable after initialization**: expands firmware attack surface.
- **Using ordinary locks that can be held by non-SMM code**: can deadlock the handler.
- **Ignoring cacheability aliases for SMRAM**: inconsistent mappings can undermine correctness and isolation.
- **Enabling dual-monitor treatment without validating executive/SMM-transfer VMCS state**: creates complex transition failures.

## Reference Table

| Concern | Required design evidence |
|---|---|
| Entry | SMI source, blocking, per-CPU SMBASE, handler address |
| Memory | SMRAM range, SMRR/support, cache type, lock timing |
| Saved state | exact state-save map/revision and validated resume fields |
| Concurrency | CPU rendezvous, reentrancy/nested-SMI policy, bounded handler time |
| VMX | default vs dual-monitor treatment and SMI/RSM transition checks |
| Tracing | Intel PT is cleared/handled across SMI; output must not target SMRR memory |

## Worked Example

Secure SMM initialization sequence:

```text
for each logical processor:
    relocate SMBASE to a non-overlapping protected region
install minimal handler and validated state-save accessors
configure SMRR/cache attributes for full SMRAM range
verify coverage and no unsafe aliases
lock chipset/CPU SMRAM controls at the platform-defined point
exercise SMI rendezvous and RSM on all CPUs
negative-test access from non-SMM software
```

Hardware/firmware validation is required; a software unit test cannot prove chipset SMRAM locks or routing.

## SMI Handler Security Review

Review the handler like a privileged parser with attacker-influenced inputs:

- Enumerate every SMI source and command/data channel: I/O ports, ACPI, chipset events, communication buffers, and software SMI interfaces.
- Validate command IDs, lengths, physical ranges, alignment, integer arithmetic, and ownership before copying or dereferencing non-SMRAM memory.
- Prevent time-of-check/time-of-use races when non-SMM agents can alter communication buffers. Copy validated request metadata/data into SMRAM or use a platform-defined protected protocol.
- Reject pointers overlapping SMRAM, MMIO with side effects, page tables, or other protected ranges unless the specific interface authorizes them.
- Bound loops, polling, and device waits. Long SMM residence increases latency and can disrupt real-time behavior.
- Scrub secrets and transient buffers where the threat model requires it; avoid leaking SMRAM content through response lengths, errors, or timing.
- Use control-flow integrity and write-protection features supported by the platform, including SMM code-access controls where available.

## Saved-State Validation

The SMM state-save map is both output from entry and input to `RSM`; handler code can modify it. Any path that accepts guest/OS-provided values and writes saved RIP, RSP, CR3, control registers, EFER, segment state, or SMBASE can become a privilege-transfer primitive.

Before intentional modification:

1. Identify the exact SMM revision/state-save-map layout for the CPU.
2. Validate canonicality, alignment, privilege/mode relationships, reserved bits, and memory ownership.
3. Restrict targets to an explicit allowlist or a verified caller context.
4. Log the reason and old/new values in a protected audit channel when practical.
5. Test malformed state so `RSM` failure cannot strand the system in SMM or create an uncontrolled reset loop.

## Multiprocessor Rendezvous Pattern

Use a bounded generation-based rendezvous rather than a lock that non-SMM code may hold:

```text
leader increments SMM generation and records required CPU set
each CPU entering SMM records arrival for generation
leader waits with timeout and platform recovery policy
leader performs global mutation after required arrivals
leader publishes completion
followers restore local state and leave
```

Account for offline CPUs, CPUs already in deeper states, nested/back-to-back SMIs, and a CPU that never arrives. Define whether failure aborts the operation, resets the platform, or proceeds with reduced guarantees.

## Default vs. Dual-Monitor Decision

Choose default treatment unless a trusted STM and full transition validation are required by the platform design. Dual-monitor treatment introduces executive and SMM-transfer VMCS pointers, SMM VM exits, special VMCALL activation, VM-entry checks returning from SMM, and additional failure/abort cases. It can separate SMM services from a VMM, but it also creates more privileged state and code to verify.

## Hardware Validation Plan

- Read back SMRR and chipset SMRAM configuration after lock.
- Attempt non-SMM reads/writes from authorized test code and confirm denial without exposing data.
- Trigger every documented SMI source on each CPU and verify state-save boundaries.
- Stress simultaneous and back-to-back SMIs with device/DMA activity.
- Verify suspend/resume and warm-reset paths preserve or safely rebuild locks.
- Test VMX guest/root/SMM transitions and Intel PT state if those features coexist.
- Review current processor/chipset errata; the SDM alone is not a platform security proof.

## Key Takeaways

1. SMM has a distinct memory and transition model.
2. SMRAM isolation depends on correct platform configuration and locking.
3. Saved-state integrity controls where and how execution resumes.
4. Multiprocessor and reentrancy behavior must be designed explicitly.
5. VMX dual-monitor treatment adds a second transition system and should be used only with full validation.

## Connects To

- **Ch 24/27/28**: VMX transitions interacting with SMI/RSM.
- **Ch 33**: trace state across SMM and STM.
