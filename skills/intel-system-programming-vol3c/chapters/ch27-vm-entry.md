# Chapter 27: VM Entries

## Core Idea

VM entry is a staged validation-and-load transaction. Diagnose failures by the stage reached: early instruction/current-VMCS checks, controls and host-state checks, guest-state checks, MSR loading, or events that occur immediately after a successful state load.

## Frameworks Introduced

- **Entry validation pipeline**:
  1. Basic instruction/current-VMCS/launch-state checks.
  2. VMX control-field checks.
  3. Host-state checks.
  4. Guest-state consistency checks.
  5. Guest-state and MSR loading.
  6. Event injection and post-entry pending-event evaluation.
- **Failure-channel classification**:
  - `CF=1`: VMfailInvalid, commonly no valid current VMCS.
  - `ZF=1`: VMfailValid; inspect VM-instruction error.
  - Exit-reason bit 31 plus basic reason 33, 34, or 41: invalid guest state, MSR-load failure, or machine check during entry.
- **State-consistency groups**: Validate controls, host state, guest control/MSR state, segment/descriptor state, RIP/RFLAGS/SSP, non-register state, and PDPTEs as separate checklists.
- **Injection-before-execution rule**: A successful VM entry may inject an event and may then immediately take a timer, window, TPR, MTF, debug, or other exit before the first guest instruction.

## Key Concepts

- **VMLAUNCH vs. VMRESUME**: First requires clear launch state; second requires launched state.
- **VM-entry interruption-information field**: Vector/type/valid/error-code control for event injection.
- **IDT-vectoring information**: Records an event whose delivery was in progress when a later exit occurred.
- **Guest activity state**: Active, HLT, shutdown, or wait-for-SIPI.
- **Guest interruptibility state**: Blocking by STI, MOV SS, SMI, NMI, and enclave interruption.
- **Host-state area**: State loaded when entry fails late or when a normal exit occurs.
- **Entry-failure exit qualification**: More detail for selected invalid-guest-state and MSR-load failures.

## Mental Models

- Treat VM entry as a transaction with several commit points; the observable recovery path tells you how far it progressed.
- Use a generated VMCS validator mirroring the SDM groups, but do not rely on software validation as proof: hardware may check fields in a different order.
- Think of injected events as part of entry, not as the first guest instruction.

## Anti-patterns

- **Reading only the VM-instruction error field**: late failures are reported through exit information instead.
- **Assuming the first reported invalid field is the only invalid field**: check order is not guaranteed.
- **Ignoring segment hidden state**: selectors, bases, limits, access rights, and unusable bits must be mutually consistent.
- **Resuming at guest RIP after an immediate exit without checking injection/vectoring state**: can duplicate or lose events.
- **Treating a successful `VMLAUNCH` as proof a guest instruction executed**: immediate post-entry exits are valid.

## Reference Table

| Observation | Stage | Read next |
|---|---|---|
| `CF=1` | Basic/current VMCS | current pointer, launch state, operand validity |
| `ZF=1` | Control/host or instruction-specific VMfailValid | VM-instruction error field |
| Exit bit 31 + reason 33 | Guest-state check | exit qualification; all guest-state groups |
| Exit bit 31 + reason 34 | Entry MSR loading | failing MSR-list index from qualification |
| Exit bit 31 + reason 41 | Machine check | machine-check status and recovery policy |
| Normal exit immediately after entry | Entry succeeded | reason, injected event, windows/timer/MTF/TPR |

## Worked Example

Diagnose a failed first launch:

```text
execute VMLAUNCH
if CF: verify current VMCS and instruction preconditions
else if ZF: err = VMREAD(VM_INSTRUCTION_ERROR); fix that class, then rerun full validator
else if control returned to host RIP:
    reason = VMREAD(EXIT_REASON)
    if reason.entry_failure:
        inspect reason 33/34/41 and qualification
    else:
        entry succeeded; decode as an immediate ordinary exit
```

For reason 33, validate at least CR0/CR4/EFER mode relationships, canonical addresses, segment access rights, TR/LDTR, RIP/RFLAGS, activity and interruptibility state, link pointer, and PDPTE conditions. Do not patch only the qualification-indicated item.

## Event Injection Checklist

1. Set a legal interruption type and vector.
2. Set “deliver error code” only for a vector/type that defines one.
3. Ensure instruction length is present where software-event injection requires it.
4. Check guest mode, IDTR, stack, and blocking state.
5. After an exit during delivery, use IDT-vectoring information to decide whether and how to reinject.

## Grouped Guest-State Validator

Use a deterministic software validator before entry, with errors grouped rather than stopped at the first field:

- **Mode group**: CR0.PE/PG, CR4.PAE/PCIDE/CET/UINTR, `IA32_EFER.LME/LMA`, and the IA-32e guest control must describe one coherent execution mode.
- **Address group**: RIP/RSP, descriptor-table bases, FS/GS bases, SYSENTER targets, SSPs, and enabled MSR addresses must be canonical where required and within physical-address constraints where applicable.
- **Segment group**: CS/SS type, S bit, DPL, present bit, L and D/B relationships, unusable flags, selector RPL, bases, and limits must match the selected guest mode. TR must describe a usable TSS; LDTR rules differ when marked unusable.
- **RFLAGS group**: reserved/fixed bits and VM/IF relationships must be legal for the selected mode and injected event.
- **Non-register group**: activity state, interruptibility bits, pending debug state, VMCS link pointer, and preemption timer must satisfy capability and cross-field constraints.
- **Paging group**: PDPTEs are checked when entering the relevant PAE paging mode; EPTP and related translation controls must already be valid.

The validator is a diagnostic aid. Hardware remains authoritative, and a processor may choose a different check order or expose a different qualifying cause when several fields are invalid.

## Immediate-Exit Decision Tree

When host control returns with no observed guest progress:

1. If CF/ZF indicate VMfail, entry did not proceed; use the failure ladder.
2. If exit-reason bit 31 is set, entry failed after beginning the transition; inspect reasons 33/34/41.
3. If a normal reason is reported, entry succeeded. Check whether:
   - an injected event triggered an intercepted exception or failed during delivery;
   - the preemption timer started at zero;
   - interrupt-window or NMI-window exiting was already eligible;
   - TPR-threshold logic requested an exit;
   - a pending MTF/debug condition took priority;
   - guest RIP immediately executed an intercepted instruction.
4. Compare guest RIP and instruction count evidence. “No user-visible progress” is not equivalent to “no entry.”

## Why Failure Stage Matters

Early VMfailValid returns to the instruction following VMLAUNCH/VMRESUME in the current host context. A late guest-state or MSR-load failure loads host state much like an exit and transfers to host RIP. Conflating the two paths can corrupt the host stack or make a diagnostic wrapper read the wrong VMCS/context.

MSR-load failure qualification identifies a 1-based list entry, but the list and host recovery configuration still need full validation. Invalid guest state usually cannot be localized reliably to one field; qualification is nonzero only for selected cases, and different CPUs may report different causes for the same multiply-invalid VMCS.

## Test Matrix

Create negative tests for each failure channel:

- no current VMCS;
- wrong launch state;
- unsupported control bit or missing dependency;
- invalid host selector/address;
- inconsistent guest mode/segment state;
- invalid MSR-load entry;
- injected event with illegal type/error-code combination;
- preemption timer value zero;
- exit during event delivery with valid IDT-vectoring information.

Assert not just that control returns, but that CF/ZF, VM-instruction error, exit reason, qualification, guest-state preservation, and host RIP/RSP match the expected channel.

## Key Takeaways

1. Entry failures have different evidence channels depending on stage.
2. Control dependencies and host state fail earlier than guest-state loading.
3. Guest-state checks are cross-field consistency checks, not isolated range checks.
4. Event injection can itself fault or cause an exit.
5. Immediate exit does not mean entry failed.

## Connects To

- **Ch 25**: fields being checked and loaded.
- **Ch 28**: host recovery and exit evidence.
- **Ch 31**: VMfail conventions and error numbers.
