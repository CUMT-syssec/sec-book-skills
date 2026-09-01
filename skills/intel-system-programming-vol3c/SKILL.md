---
name: intel-system-programming-vol3c
description: "Knowledge base from Intel 64 and IA-32 Architectures Software Developer's Manual, Volume 3C (September 2023). Use when designing or debugging VMX, VMCS, VM entry/exit, EPT/VPID, APIC virtualization, SMM, or Intel Processor Trace."
---

<!-- argument-hint: [topic, control/MSR/instruction name, or chapter number] -->

# Intel 64 and IA-32 SDM, Volume 3C

**Author**: Intel Corporation | **Pages**: 316 | **Chapters**: 10 (manual chapters 24–33) | **Generated**: 2026-09-01

## How to Use This Skill

- Without arguments: load the design and debugging rules below.
- With a topic such as `EPT violation`, `VM-entry failure`, `posted interrupt`, or `ToPA`: read the indexed chapter before answering.
- With `ch24`–`ch33`: load that chapter file for implementation details and failure modes.
- Treat CPUID and VMX capability MSRs as runtime truth. Never assume a control, invalidation type, activity state, or packet feature exists merely because the manual defines it.

## Core Frameworks and Decision Rules

### Capability-first configuration

Use this order for every VMX or Intel PT feature:

1. Enumerate the architectural feature with CPUID.
2. Read the relevant capability MSR.
3. Derive legal control values, including reserved-must-be-0 and reserved-must-be-1 bits.
4. Initialize aligned, correctly typed memory structures.
5. Enable the feature only after all dependent controls are valid.
6. On failure, inspect architectural status before changing configuration.

Do not copy control words from another processor model. VM entry validates dependencies between primary, secondary, tertiary, exit, and entry controls.

### VMX lifecycle

Use `CPUID.1:ECX.VMX` → `IA32_FEATURE_CONTROL` → fixed CR0/CR4 masks → `CR4.VMXE` → initialized VMXON region → `VMXON`. Build one VMCS per virtual CPU, use `VMCLEAR` before first launch, `VMPTRLD` to make it current, `VMLAUNCH` only from clear launch state, and `VMRESUME` only after a successful launch. Leave with `VMXOFF` before clearing `CR4.VMXE`.

### VMCS as a state machine, not a struct

The VMCS memory format after its header is implementation-specific. Access fields with `VMREAD`/`VMWRITE`; track three independent properties: active, current, and launch state. Validate the revision identifier, 4-KiB alignment, physical-address width, and memory type. Treat guest state, host state, execution controls, entry controls, exit controls, and exit information as separate logical groups.

### VM entry debugging ladder

Classify failure before editing fields:

- `CF=1`: no valid current VMCS or equivalent VMfailInvalid condition.
- `ZF=1`: VMfailValid; read the VM-instruction error field.
- Exit reason bit 31 set with reason 33/34/41: entry began but failed on guest state, MSR loading, or machine check.
- Successful entry followed by immediate exit: decode exit reason, qualification, interruption/vectoring fields, and instruction information.

Checks can occur in implementation-dependent order. Fixing the reported field does not prove the VMCS has no other defects.

### Exit handling as evidence reconstruction

For every VM exit, capture before emulation: exit reason, qualification, guest linear/physical address where valid, interruption information, IDT-vectoring information, instruction length/information, and guest RIP/state. Distinguish direct event exits from exits during event delivery: direct exits usually avoid the event's ordinary architectural updates; indirect exits may already have updated CR2, debug state, interrupt acknowledgement, or stack memory.

### Translation-coherency rule

Changing paging structures is not equivalent to invalidating cached mappings. Associate translations with VPID, PCID, EPTP, and address. Use `INVVPID` for guest-linear context changes and `INVEPT` for EPT-derived mappings. Choose the narrowest supported invalidation type that covers the mutation; use broader invalidation when correctness cannot be proven. Preserve the rule that VPID 0 has special semantics and is invalid for several INVVPID types.

### Interrupt-virtualization dependency chain

Treat TPR shadowing, APIC-access virtualization, x2APIC virtualization, APIC-register virtualization, virtual-interrupt delivery, IPI virtualization, and posted interrupts as dependent controls—not independent switches. Posted-interrupt descriptors may be updated concurrently only with atomic locked operations. If hardware cannot safely complete an APIC write or IPI virtualization, expect a trap-like APIC-write exit and emulate from the recorded page offset.

### SMM is a separate security boundary

On SMI, the processor saves state to SMRAM and enters the SMI handler environment; `RSM` restores state. Protect SMRAM with platform mechanisms such as SMRR and lock it at the appropriate lifecycle point. Do not treat SMM as ordinary ring 0. Under VMX, explicitly choose default or dual-monitor treatment and validate the extra VMCS transitions and SMI-blocking rules.

### Intel PT is a packet protocol

Configure Intel PT only from enumerated CPUID capabilities. Select output mode (single range or ToPA), filters, timing, and packet sources; initialize output structures; clear error/stop state; then set `TraceEn` last. Disable by clearing `TraceEn` first and flush before consuming data. Decoders must resynchronize at PSB+, distinguish LIP/RIP context, follow packet ordering, and account for VMX TSC scaling and discontinuities.

## Chapter Index

| File | Manual chapter | Topic | Primary use |
|---|---:|---|---|
| [ch24](chapters/ch24-vmx-overview.md) | 24 | VMX architecture and lifecycle | Establish prerequisites and invariants |
| [ch25](chapters/ch25-vmcs.md) | 25 | VMCS fields and controls | Build a legal VMCS |
| [ch26](chapters/ch26-vmx-non-root.md) | 26 | Non-root behavior and exits | Choose interception and guest-visible behavior |
| [ch27](chapters/ch27-vm-entry.md) | 27 | VM-entry checks and loading | Diagnose failed or immediate entry |
| [ch28](chapters/ch28-vm-exit.md) | 28 | VM-exit state and evidence | Decode and emulate exits safely |
| [ch29](chapters/ch29-ept-vpid.md) | 29 | EPT, VPID, HLAT, invalidation | Maintain translation correctness |
| [ch30](chapters/ch30-apic-virtualization.md) | 30 | APIC and virtual interrupts | Reduce interrupt-related exits safely |
| [ch31](chapters/ch31-vmx-instructions.md) | 31 | VMX instruction reference | Interpret success, failure, and exceptions |
| [ch32](chapters/ch32-smm.md) | 32 | System Management Mode | Secure SMRAM and SMI transitions |
| [ch33](chapters/ch33-intel-pt.md) | 33 | Intel Processor Trace | Configure and decode hardware traces |

## Topic Index

- **APIC access / APIC write** → ch30, ch28
- **Capability MSRs / control masks** → ch24, ch25, ch27
- **EPT / EPTP / #VE** → ch25, ch26, ch29
- **Event injection / IDT vectoring** → ch27, ch28
- **INVEPT / INVVPID** → ch29, ch31
- **Intel PT / ToPA / PSB+ / timing** → ch33
- **Posted interrupts / virtual interrupt delivery** → ch25, ch30
- **SMM / SMRAM / SMRR / STM** → ch32
- **VM entry failure / VM-instruction error** → ch27, ch31
- **VM exit reason / qualification** → ch28
- **VMCS lifecycle / shadow VMCS** → ch25, ch31
- **VMFUNC / EPTP switching / #VE** → ch26, ch29, ch31
- **VMXON / VMLAUNCH / VMRESUME** → ch24, ch31
- **VPID / translation caches** → ch29

## Supporting Files

- [glossary.md](glossary.md) — architectural terms and fields
- [patterns.md](patterns.md) — reusable implementation and diagnosis workflows
- [cheatsheet.md](cheatsheet.md) — compact decision tables and fast checks

## Scope and Limits

This skill is a synthesized study aid for the September 2023 Volume 3C source. It does not replace the complete Intel SDM, current processor errata, later manual revisions, Volume 2 instruction definitions, Volume 3A/3B dependencies, or Volume 4 MSR definitions. Hardware behavior must be validated on the target CPU; emulator, unit-test, and static-analysis success do not prove firmware or bare-metal correctness.
