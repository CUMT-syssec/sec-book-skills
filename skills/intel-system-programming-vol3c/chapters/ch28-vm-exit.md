# Chapter 28: VM Exits

## Core Idea

A VM exit is both a control transfer and an evidence record. Correct handling depends on whether the triggering event caused the exit directly, or whether delivery/execution began and a secondary condition caused the exit; the architectural state can differ substantially.

## Frameworks Introduced

- **Direct vs. indirect event exit**:
  - Direct: the intercepted event itself exits before ordinary delivery updates most architectural state.
  - Indirect: event delivery begins, then a nested fault, APIC access, EPT condition, task switch, or other event exits; some state may already be updated.
- **Exit evidence bundle**: Capture atomically before emulation.
  1. Exit reason and flags.
  2. Exit qualification.
  3. VM-exit interruption information and error code.
  4. IDT-vectoring information and error code.
  5. Guest linear/physical address where defined.
  6. Instruction length and instruction information.
  7. Saved guest RIP/RSP/RFLAGS, interruptibility/activity, and relevant registers.
- **Exit transition pipeline**: Record exit information → save guest state → save configured MSRs → load host state → load configured MSRs → begin host handler.
- **Resume-safety rule**: Emulate, advance RIP, reinject, or retry only after classifying the exit as fault-like, trap-like, event-related, or instruction-related.

## Key Concepts

- **Basic exit reason**: Low 16 bits identify the cause; other bits report entry failure, enclave mode, bus lock, and related state.
- **Exit qualification**: Cause-specific payload; zero/undefined for causes that do not define it.
- **VM-exit interruption information**: Describes a vectored event that directly caused exit.
- **IDT-vectoring information**: Describes an event being delivered when another exit occurred.
- **Acknowledge interrupt on exit**: Determines whether an external interrupt is acknowledged and its vector recorded.
- **VMX abort**: Catastrophic condition recorded in the VMCS-region abort indicator; ordinary exit recovery is not assumed.

## Mental Models

- Treat exit fields like a typed union keyed by basic exit reason.
- Treat instruction completion as a question, not an assumption. Trap-like APIC-write and selected debug exits occur after the operation; fault-like exits do not.
- Preserve event-delivery state explicitly: an exit handler is part of an interrupt/exception state machine.

## Anti-patterns

- **Reading qualification without checking the reason**: its meaning and validity are reason-specific.
- **Always incrementing guest RIP by exit-instruction length**: wrong for faults, asynchronous exits, and many event-delivery cases.
- **Reinjecting the VM-exit event while ignoring IDT-vectoring information**: can double-deliver or lose a nested event.
- **Assuming direct page-fault exit updated CR2**: direct interception does not; the fault address is in exit qualification.
- **Resuming after machine check without consulting recovery-validity state**: guest state may be suspect.

## Reference Table

| Exit class | Did ordinary operation complete? | Typical resume action |
|---|---|---|
| Fault-like instruction exit | No | emulate/fix and retry or inject fault |
| Trap-like exit | Yes | usually continue at saved next RIP |
| Direct exception/NMI/interrupt exit | Delivery not performed | handle or reinject according to policy |
| Exit during event delivery | Partially progressed | inspect IDT-vectoring and nested cause |
| EPT violation | Memory access did not complete | update mapping/permissions, invalidate if needed, retry |
| Entry failure | Guest execution did not begin | repair VMCS; do not treat as ordinary guest exit |

## Worked Example

Handle an EPT violation that occurred while delivering a guest page fault:

```text
reason = EXIT_REASON
qual = EXIT_QUALIFICATION
gpa = GUEST_PHYSICAL_ADDRESS
gla = GUEST_LINEAR_ADDRESS if reported-valid
vectoring = IDT_VECTORING_INFO

repair_or_reject_ept_access(gpa, qual)
perform_required_invept()
if vectoring.valid:
    preserve/reinject the interrupted guest event according to its type
resume without blindly advancing RIP
```

The EPT access and the in-flight guest event are two separate obligations; fixing only the mapping may lose the original event.

## Exit-Handler Transaction

Organize every handler into explicit phases:

1. **Freeze evidence**: copy VMCS exit fields and relevant guest registers into a typed record before VMWRITE or emulation changes anything.
2. **Validate evidence**: confirm reason-specific validity bits before using guest linear address, physical address, instruction length, or interruption fields.
3. **Authorize action**: apply guest policy to the requested resource. An exit is not automatic permission to emulate successfully.
4. **Perform side effects**: emulate, update mappings, acknowledge virtual devices, inject an exception, or schedule the vCPU.
5. **Repair transition state**: update RIP only under proven completion rules; maintain interruptibility, pending event, and vectoring information.
6. **Resume or terminate**: use VMRESUME for a launched VMCS, with diagnostics ready for another entry failure or immediate exit.

This structure separates evidence parsing from policy and makes exit paths testable with recorded VMCS fixtures.

## Direct/Indirect Event Examples

- A directly intercepted page fault does not update CR2 as ordinary delivery would; exit qualification provides the faulting linear address. If delivery starts and a nested EPT violation exits, CR2 may already reflect the page fault.
- A directly intercepted external interrupt normally remains pending and unacknowledged unless “acknowledge interrupt on exit” is enabled. With that control, the exit information carries the acknowledged vector.
- A directly intercepted debug exception does not perform the ordinary DR6/DR7 updates. A secondary exit during delivery may observe debug-state changes.
- An exit during event delivery may have written part of a guest stack frame even though CS/RIP never reached the handler. The VMM must not infer an all-or-nothing guest memory update.

## RIP and Reinjection Rules

Before advancing RIP, prove all three:

1. The exit cause is tied to the current instruction.
2. The instruction did not already complete as a trap-like exit.
3. The VMM has fully emulated the architectural effect or intentionally skips it under guest-visible policy.

Before reinjecting, decide whether the original event was never delivered, partially delivered, or superseded. Use the VM-exit interruption field for the event that caused exit and IDT-vectoring information for the event already in flight. Preserve type, vector, deliver-error-code semantics, error code, and software-instruction length as applicable.

## Failure Containment

- Treat unknown exit reasons or reserved field combinations as unsupported CPU/VMM state; do not silently resume.
- Bound repeated identical exits. A same-RIP/same-reason loop is a diagnostic signal and a denial-of-service risk.
- For VMX abort, stop using the affected VMCS/processor path until the abort indicator and platform state are analyzed.
- For machine check, consult `IA32_MCG_STATUS` recovery validity before resuming; state saved in the VMCS may not be reliable.
- Sanitize guest-controlled values before using them as host pointers, array indices, MSR numbers, ports, or instruction lengths.

## Minimal Exit Log Schema

Record vCPU ID, host CPU, timestamp/sequence, reason raw+decoded, qualification, GLA/GPA validity+value, interruption and vectoring fields, instruction length/information, guest RIP/RSP/RFLAGS/CR3, EPTP/VPID, and chosen action. This is enough to reconstruct first deviation and distinguish repeated false recovery from genuine progress.

## Key Takeaways

1. Capture the complete evidence bundle before changing guest or VMCS state.
2. Direct and indirect event exits observe different architectural side effects.
3. Exit reason selects the schema for qualification and auxiliary fields.
4. Correct RIP advancement depends on completion semantics.
5. VMX abort and machine-check paths require stronger recovery boundaries than ordinary exits.

## Connects To

- **Ch 27**: entry failures and event injection.
- **Ch 29**: EPT qualification and retry.
- **Ch 30**: APIC-access and APIC-write trap-like exits.
- **Ch 31**: instruction-specific completion and VMfail semantics.
