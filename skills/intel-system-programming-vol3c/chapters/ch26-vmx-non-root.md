# Chapter 26: VMX Non-Root Operation

## Core Idea

VMX non-root operation modifies instruction and event behavior according to VMCS controls. Design the interception policy around what must be mediated, then use hardware features such as EPT, APIC virtualization, VMFUNC, and #VE where they preserve correctness with fewer exits.

## Frameworks Introduced

- **Exit classification**:
  - Unconditional instruction exits: VMX-sensitive operations that always transfer control under applicable conditions.
  - Conditional instruction exits: governed by execution controls, bitmaps, masks, and target lists.
  - Event exits: interrupts, NMIs, SMIs, INIT, exceptions, preemption timer, EPT/APIC events, and windows.
- **Interception hierarchy**: Prefer narrow filters over broad exits.
  - Exception bitmap for selected exception vectors.
  - I/O bitmaps for selected ports.
  - MSR bitmaps for selected MSRs.
  - CR masks/read shadows and CR3 targets for selected control-register changes.
- **Hardware fast path**: Use EPT, VPID, virtual interrupt delivery, VM functions, or #VE only when their control dependencies and invalidation rules are satisfied.
- **Event-priority reasoning**: Treat each operation as an instruction, REP iteration, or IDT delivery; determine which fault/exit wins before assuming handler state.

## Key Concepts

- **VMX-preemption timer**: Counts down during non-root operation and can trigger an exit at zero.
- **Monitor Trap Flag (MTF)**: Schedules a trap-like exit after the next operation boundary, with special handling around blocked events.
- **VMFUNC**: Guest-callable hardware function that may complete without an exit when enabled.
- **EPTP switching**: VM function 0; selects one of up to 512 preconfigured EPTPs using `ECX`.
- **#VE**: Virtualization exception, vector 20, used to deliver selected convertible EPT violations to the guest.
- **Suppress #VE**: EPT entry bit 63 controlling convertibility for the relevant non-present or leaf entry.
- **Unrestricted guest**: Allows real-address/unpaged guests subject to secondary-control support.

## Mental Models

- Use an exit as a policy boundary, not as the default implementation of every privileged operation.
- Think of a bitmap as a per-resource allow/exit matrix.
- Treat #VE as delegated fault handling: the VMM still controls which EPT violations are convertible.

## Anti-patterns

- **Exiting on every I/O/MSR/instruction without measuring need**: creates avoidable transition overhead.
- **Assuming a fault and VM exit both occur**: priority rules normally select one architectural outcome.
- **Enabling VMFUNC without constraining its function mask and data structures**: disabled selections exit; invalid inputs may fault or exit.
- **Switching EPTP without considering cached mappings/A-D state**: later accesses may use stale or semantically mismatched translations.
- **Using #VE without clearing the information-area guard**: offset 4 is set to all ones on delivery; another #VE requires software to clear it.

## Reference Tables

### VMFUNC decision

| Condition | Result |
|---|---|
| Enable-VM-functions control is 0, or `EAX > 63` | `#UD` |
| Selected VM-function bit is 0 | VM exit, reason 59 |
| EPTP switching and `ECX >= 512` | VM exit |
| Selected EPTP changes page-walk length or is otherwise invalid | VM exit |
| Valid enabled EPTP list entry | EPTP changes without modifying registers/flags |

### Convertible EPT violation

Deliver `#VE` only when the #VE control is enabled, the relevant EPT entry does not suppress it, `CR0.PE=1`, no event is being delivered through the IDT, no prohibited PT/shadow-stack case applies, and the information-area guard is clear. EPT misconfiguration always exits.

## Worked Example

Allow a guest runtime to switch between two prevalidated memory views:

```text
root: validate EPTP_A and EPTP_B
root: place them in aligned EPTP list entries 0 and 1
root: enable secondary controls, VM functions, EPT, and EPTP-switching function 0
guest: EAX = 0; ECX = desired_index; execute VMFUNC
root on exit reason 59: reject invalid index/EPTP or emulate only after policy checks
```

After changing an EPT hierarchy or A/D-mode semantics, use the required INVEPT discipline before relying on new permissions or status bits.

## Interception Design Procedure

1. List the resources that the guest must not access directly: selected MSRs, I/O ports, control-register bits, instructions, exceptions, timing state, APIC functions, and address ranges.
2. For each resource, decide whether hardware can safely execute it, virtualize it, convert it to a guest exception, or must exit.
3. Select the narrowest control mechanism. An MSR bitmap is preferable to unconditional MSR exits when only a few MSRs need mediation; CR masks/read shadows are preferable when only selected bits are virtualized.
4. Define the exit handler's completion semantics: emulate and advance, deny with an injected exception, modify policy and retry, or terminate the guest.
5. Measure exit frequency and handler cost, but never remove an intercept until the security invariant has another enforcement point.

## Event and Instruction Reasoning

Use three questions for any non-root operation:

- **What would happen outside VMX?** Identify the ordinary instruction result, exception, or event delivery.
- **Which VMCS control changes it?** Include bitmap bits, exception bitmap, masks/read shadows, target lists, and secondary controls.
- **Which outcome has priority?** Operand faults, debug events, APIC/EPT conditions, and VM exits may compete. The winning outcome determines whether state changed and whether RIP should advance.

This prevents a common emulator error: treating a VM exit as if the instruction executed normally and then also emulating it. For fault-like exits, the operation has not completed. For explicitly trap-like paths, the saved RIP/state may already reflect completion.

## #VE Handler Contract

A guest #VE handler needs more than an IDT entry:

- The VMM supplies a valid, appropriately protected virtualization-exception information area.
- The handler verifies the recorded exit-reason equivalent and qualification before acting.
- It treats guest linear/physical addresses as cause-specific evidence, not universally valid fields.
- It clears the 32-bit guard at offset 4 only after consuming the record; otherwise subsequent convertible violations cannot generate another #VE.
- It has a policy for nested faults. #VE has page-fault-like severity for double-fault combination rules.
- It cannot repair EPT misconfiguration; that remains a VMM exit and should be treated as a VMM-side defect.

## Performance/Security Trade-off Matrix

| Choice | Benefit | Risk/control obligation |
|---|---|---|
| Broad instruction exiting | simple initial VMM | high exit rate; larger handler surface |
| Bitmap-selective exiting | lower overhead | bitmap address/lifetime correctness |
| VMFUNC fast path | no exit for enabled function | prevalidated list, strict function mask, invalidation |
| EPT-violation #VE | guest-local fault handling | protected info area and guest handler correctness |
| Unrestricted guest | direct real/unpaged guest support | more guest-mode combinations to validate |
| APIC virtualization | fewer interrupt exits | complex dependent controls and shared state |

## Failure Smells

- An exit handler returns to the same instruction forever because it neither emulates nor injects nor changes policy.
- A control bit is enabled globally even though only one guest needs the feature.
- A bitmap is freed or modified while a running VMCS still references it.
- EPTP switching is used as a permission revocation mechanism without invalidating cached mappings on all affected CPUs.
- The VMM treats `#UD` from disabled VMFUNC as an ordinary VMFUNC exit.

## Key Takeaways

1. VMCS controls transform ordinary instruction/event behavior into direct execution, fault, or VM exit.
2. Narrow interception minimizes exits and simplifies evidence.
3. VMFUNC is a controlled no-exit path, not unrestricted guest authority.
4. #VE delegates selected EPT violations; misconfigurations remain VMM-visible exits.
5. Priority and blocking rules determine the state observed by an exit handler.

## Connects To

- **Ch 25**: defines the controls and supporting fields.
- **Ch 27/28**: specifies transition ordering and evidence.
- **Ch 29**: defines EPT translation and invalidation.
- **Ch 30**: details APIC fast paths.
