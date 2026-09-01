# Chapter 25: Virtual-Machine Control Structures

## Core Idea

The VMCS is an implementation-managed state machine whose fields define guest state, host restoration, non-root behavior, and transition policy. Software must program it through architectural field encodings and capability-derived control values, not by mapping an assumed C structure onto memory.

## Frameworks Introduced

- **VMCS lifecycle state machine**:
  - `VMPTRLD X`: X becomes active and current.
  - `VMLAUNCH`: requires current VMCS with clear launch state; success changes it to launched.
  - `VMRESUME`: requires launched state.
  - `VMCLEAR X`: writes back state, makes X inactive/not current, and sets launch state clear.
  - `VMPTRST`: reports the current pointer or all ones when none is current.
- **Six-area decomposition**: Use when reviewing or initializing a VMCS.
  1. Guest-state area.
  2. Host-state area.
  3. VM-execution controls.
  4. VM-exit controls.
  5. VM-exit information fields.
  6. VM-entry controls.
- **Control dependency graph**: Activate secondary/tertiary controls before using their dependent bits; check pin-based, primary, secondary, tertiary, exit, and entry capability MSRs.
- **Selective interception**: Use exception bitmap, I/O bitmaps, MSR bitmaps, CR masks/read shadows, CR3 targets, and instruction-exiting controls to retain only the control the VMM needs.

## Key Concepts

- **VMCS pointer**: 64-bit physical address, normally 4-KiB aligned and within physical-address width.
- **Revision identifier**: `IA32_VMX_BASIC[30:0]`, stored at VMCS-region offset 0.
- **Shadow-VMCS indicator**: VMCS-region header bit 31.
- **VMX-abort indicator**: nonzero processor-written diagnostic at offset 4.
- **Current VMCS**: target of `VMLAUNCH`, `VMRESUME`, `VMREAD`, and `VMWRITE` in root operation.
- **Guest interruptibility state**: STI, MOV SS, SMI, NMI, and enclave-interruption blocking state.
- **Activity state**: active, HLT, shutdown, or wait-for-SIPI, subject to capability support.
- **EPTP**: EPT root plus page-walk length, memory type, and optional A/D enable.
- **VPID**: nonzero translation-context identifier when VPID is enabled.

## Mental Models

- Think of VMCS fields as a serialized transition contract, not merely saved registers.
- Use “intercept only what must be mediated” to balance isolation and exit overhead.
- Use guest/host masks plus read shadows when the guest should observe one CR value while hardware enforces another.

## Anti-patterns

- **Writing the VMCS data area directly**: its layout is implementation-specific.
- **Reusing a launched VMCS with `VMLAUNCH`**: use `VMRESUME`, or `VMCLEAR` to reset launch state deliberately.
- **Using UC memory by default**: the manual strongly discourages it due to transition cost; follow `IA32_VMX_BASIC` memory-type reporting.
- **Leaving unsupported reserved control bits arbitrary**: VM entry will fail.
- **Enabling dependent APIC/EPT/VMFUNC controls in isolation**: control-consistency checks reject invalid combinations.

## Reference Tables

### VMCS region header

| Offset | Content | Rule |
|---:|---|---|
| 0 | Revision ID bits 30:0; shadow bit 31 | initialize before `VMPTRLD` |
| 4 | VMX-abort indicator | inspect after suspected abort |
| 8+ | Implementation-specific VMCS data | never access as a software-defined layout |

### Control categories

| Category | Governs | Typical decisions |
|---|---|---|
| Pin-based | asynchronous events | external interrupts, NMI, preemption timer, posted interrupts |
| Primary processor-based | common synchronous behavior | HLT/RDTSC/CR3/I/O/MSR interception, secondary-control activation |
| Secondary/tertiary | extended features | EPT, VPID, unrestricted guest, APIC, VMFUNC, #VE, HLAT |
| Exit controls | state saved/loaded on exit | host width, MSRs, PAT/EFER/PT/CET |
| Entry controls | state loaded/injected on entry | guest width, MSRs, event injection |

## Worked Example

Build a control word from capability masks rather than copying a constant:

```text
allowed0 = capability_msr[31:0]     // bits allowed to be 0
allowed1 = capability_msr[63:32]    // bits allowed to be 1
controls = requested
controls |= ~allowed0               // force must-be-1 bits
controls &= allowed1                // clear must-be-0 bits
assert((requested & ~allowed1) == 0) // reject unsupported requested features
```

Then validate dependencies, e.g. posted interrupts require the corresponding pin control and external-interrupt exiting; secondary features require activation of secondary controls.

## VMCS Construction Checklist

Build the VMCS in passes so a review can isolate mistakes:

1. **Region pass**: verify 4-KiB alignment, physical-address width, revision ID, ordinary-versus-shadow bit, write-back memory type, and exclusive ownership.
2. **Lifecycle pass**: `VMCLEAR` the new region, then `VMPTRLD`; confirm the current pointer if diagnostics require it.
3. **Host-state pass**: populate the host CRs, selectors, bases, descriptor tables, RIP/RSP, SYSENTER state, EFER/PAT/CET and other enabled state. Host RIP and RSP must lead to a valid exit handler environment.
4. **Guest-state pass**: populate visible and hidden segment state, control/debug registers, RIP/RSP/RFLAGS, descriptor tables, activity/interruptibility state, link pointer, and enabled MSRs.
5. **Execution-control pass**: derive pin, primary, secondary, and tertiary controls from capabilities; initialize every address/list/bitmap used by a set control.
6. **Transition-control pass**: derive VM-exit and VM-entry controls together so width, PAT/EFER, CET, PT, and MSR-list behavior is coherent.
7. **Evidence pass**: initialize exit-information handling, clear stale entry injection state, and log the final requested/effective control values.

## Control-Dependency Examples

- “Activate secondary controls” must be enabled before any secondary control is effective; otherwise design logic that assumes EPT, VPID, unrestricted guest, VMFUNC, or APIC features is wrong even if a bit was written elsewhere.
- Posted-interrupt processing requires compatible pin-based interrupt controls plus valid notification vector and PID address. Hardware entry checks enforce some relationships; software must also enforce lifetime and concurrency.
- EPT requires a valid EPTP. The selected memory type, walk length, address bits, and A/D setting must be supported.
- VPID enablement requires a nonzero VPID for normal tagged use. Reuse needs an invalidation discipline.
- Event injection requires consistent interruption type, vector, optional error code, and instruction length. A stale valid bit can inject an unintended event on the next entry.
- VMCS shadowing requires a valid link pointer, VMREAD bitmap, and VMWRITE bitmap, plus a shadow VMCS header compatible with the capability.

## Why It Works and How It Fails

Capability-derived controls separate *requested policy* from *legal encoding*. This is crucial because reserved bits are not uniformly zero: some historical controls must be one unless “true controls” report that zero is permitted. A wrapper that only masks unsupported one-bits may still leave mandatory-one bits clear and cause entry failure.

The VMCS lifecycle state exists partly outside the directly readable fields. Software cannot discover launch state with VMREAD. The VMM therefore needs its own authoritative lifecycle record and must update it only after confirmed instruction outcomes. If a launch fails, do not mark the VMCS launched; if launch succeeds but the guest immediately exits, it is launched and future entry uses VMRESUME.

## Review Questions

- Can any two logical processors make the same VMCS current concurrently?
- Does every set control have a fully initialized backing pointer/list/bitmap?
- Are all physical addresses checked against width, alignment, and reserved bits?
- Are host-state selectors and bases valid for the exit handler's execution mode?
- Are guest segment “unusable” bits consistent with null selectors and mode?
- Are VM-entry/exit MSR counts bounded, and are list entries aligned and legal?
- Is the VMCS link pointer all ones when shadowing/nested behavior is unused?
- Are current, active, and launch-state transitions logged after confirmed success?

## Key Takeaways

1. The VMCS has active/current/launch dimensions; track each explicitly.
2. Its header is architectural, but its data layout is not.
3. Capability masks define legal controls on the current processor.
4. Guest and host state include more than general registers: hidden segment state, MSRs, interruptibility, and activity matter.
5. Every pointer-backed structure needs alignment, address-width, memory-type, and lifetime review.

## Connects To

- **Ch 27**: consumes controls and state during VM entry.
- **Ch 28**: populates exit information and restores host state.
- **Ch 30**: uses virtual-APIC and posted-interrupt structures.
- **Ch 31**: defines VMCS maintenance instruction semantics.
