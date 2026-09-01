# Chapter 24: Introduction to Virtual Machine Extensions

## Core Idea

VMX gives a VMM selective control over privileged resources while allowing a guest OS to execute at its intended privilege level. Correct use begins with capability discovery and a strict lifecycle; VMX is not enabled by executing `VMXON` alone.

## Frameworks Introduced

- **VMX root vs. VMX non-root operation**: Use this distinction instead of ring level to reason about virtualization authority.
  - Root operation normally hosts the VMM.
  - Non-root operation normally hosts guest software.
  - VM entry moves root → non-root; VM exit moves non-root → root.
- **Capability-first enablement**: Use before touching VMX controls.
  1. Confirm `CPUID.1:ECX.VMX[5] = 1`.
  2. Verify `IA32_FEATURE_CONTROL` policy and lock state.
  3. Apply `IA32_VMX_CR0_FIXED0/1` and `IA32_VMX_CR4_FIXED0/1` constraints.
  4. Set `CR4.VMXE`.
  5. Allocate and initialize a naturally aligned VMXON region using the revision identifier from `IA32_VMX_BASIC`.
  6. Execute `VMXON` and check architectural success/failure.
- **VMM lifecycle**: Use for bring-up and teardown.
  - `VMXON` → configure/load VMCS → `VMLAUNCH` → handle exits → `VMRESUME` → `VMXOFF`.

## Key Concepts

- **VMX operation**: Processor mode containing root and non-root operation.
- **VMCS**: Per-virtual-CPU control/state object governing non-root execution and transitions.
- **VM entry**: Transition into guest non-root execution.
- **VM exit**: Transition to the VMM entry point in root operation.
- **IA32_FEATURE_CONTROL**: Reset-initialized platform policy MSR; bit 0 locks it, bits 1/2 authorize VMXON in/outside SMX.
- **CR4.VMXE**: Control bit enabling recognition of VMX operation.
- **VMXON region**: Aligned physical-memory region used by the processor for VMX operation.
- **Unrestricted guest**: Secondary control permitting guest `CR0.PE`/`CR0.PG` to be clear on supported CPUs.

## Mental Models

- Think of VMX as a second privilege axis orthogonal to CPL.
- Think of capability MSRs as a contract: every control vector is constrained by allowed-0 and allowed-1 masks.
- Use one VMCS per virtual CPU; do not share a current VMCS concurrently across logical processors.

## Anti-patterns

- **Assuming BIOS enabled VMX because CPUID advertises it**: `IA32_FEATURE_CONTROL` may forbid `VMXON`.
- **Setting the lock bit before finalizing policy**: the MSR cannot be changed until power-up reset.
- **Clearing `CR4.VMXE` while still in VMX operation**: leave with `VMXOFF` first.
- **Hardcoding CR0/CR4 values**: fixed-bit requirements differ across implementations.
- **Treating a 4-KiB allocation as sufficient**: alignment, physical-address width, revision identifier, and memory type also matter.

## Reference Table

| Check | Required evidence | Typical failure |
|---|---|---|
| VMX present | `CPUID.1:ECX.VMX=1` | `VMXON` is unsupported |
| Firmware policy | `IA32_FEATURE_CONTROL` permits selected SMX context | `#GP` on `VMXON` |
| Control registers | fixed-0/fixed-1 masks satisfied | `VMXON` fails |
| Execution mode | protected mode, not A20M; implementation constraints met | `#UD`/failure |
| VMXON memory | aligned, valid PA, correct revision ID | VMfail/undefined bring-up |

## Worked Example

Bring up VMX outside SMX:

```text
if !CPUID.VMX: stop unsupported
fc = RDMSR(IA32_FEATURE_CONTROL)
if fc.lock && !fc.vmx_outside_smx: stop firmware-disabled
if !fc.lock: set vmx_outside_smx and lock according to platform policy
CR0 = legalize(CR0, IA32_VMX_CR0_FIXED0, IA32_VMX_CR0_FIXED1)
CR4 = legalize(CR4, IA32_VMX_CR4_FIXED0, IA32_VMX_CR4_FIXED1) | VMXE
region = aligned_4K_writeback_memory()
region.revision = IA32_VMX_BASIC.revision_id
execute VMXON(region.physical_address); check CF/ZF or exception
```

The policy write belongs in trusted firmware or equally privileged platform initialization; an OS should not casually lock a platform-wide MSR.

## Implementation Checklist

### Before `VMXON`

- Confirm the current logical processor supports VMX; repeat capability work per heterogeneous CPU class if the platform can expose different capabilities.
- Confirm the current execution environment is appropriate: protected mode requirements are satisfied, `RFLAGS.VM=0`, and the processor is not in a condition such as A20M that forbids entry.
- Read `IA32_FEATURE_CONTROL` before writing it. If locked, accept it as policy. If unlocked, use a platform-owned initialization path that sets only the intended SMX/outside-SMX enables and locks after policy is final.
- Compute legal CR0/CR4 values with both FIXED0 and FIXED1 MSRs. Preserve unrelated operating-system bits rather than replacing the registers with constants.
- Derive VMX region size, revision ID, address-width restrictions, and memory type from `IA32_VMX_BASIC` and related capability state.
- Zero or initialize the whole region according to the architecture, write the revision header, and pass the physical—not virtual—address to the instruction operand.

### After `VMXON`

- Save the VMX instruction result before flag-clobbering code runs.
- Do not assume VMXON created a current VMCS; it did not. Prepare, clear, and load a separate VMCS.
- Record which logical processor entered VMX operation. Entry/exit and per-CPU teardown must be coordinated, especially during hotplug, suspend, crash, or shutdown.
- Establish an emergency teardown path that can stop vCPUs, leave VMX operation, and restore host control state without executing guest code.

## Why the Order Matters

`CR4.VMXE` only enables the VMX instruction set; it does not override firmware policy, legalize CR0/CR4, initialize processor-owned memory, or select a VMCS. Moving `VMXON` earlier converts a diagnosable configuration problem into an instruction exception or VMfail. Moving `IA32_FEATURE_CONTROL` locking too early can turn a software bug into a power-cycle-only recovery condition.

VMX root operation also constrains the VMM itself. Fixed CR0/CR4 rules apply in root operation, and features such as Intel PT can have VMX-specific availability. Therefore, successful entry is not a permanent proof that every later host-state change is legal.

## Diagnostic Guide

| Symptom | Likely class | Evidence to collect |
|---|---|---|
| VMX absent in CPUID | unsupported processor/virtual CPU exposure | raw CPUID leaf 1 and hypervisor policy |
| `#UD` on `VMXON` | VMX disabled at instruction-recognition layer | CR4.VMXE, mode, VMX CPUID, nesting policy |
| `#GP(0)` on `VMXON` | platform policy or operand/state restriction | `IA32_FEATURE_CONTROL`, CPL, mode, address |
| CF/ZF reports failure | region/control-state validation | immediate RFLAGS, physical address, header, fixed masks |
| Works on one CPU only | per-CPU policy/capability/initialization bug | per-CPU MSRs, region ownership, CPU topology |
| Failure after resume/hotplug | lifecycle state not restored | per-CPU VMXON/VMCS state and teardown logs |

## Validation Boundary

A hypervisor running inside another hypervisor may see synthetic VMX behavior, different capability exposure, or nested-virtualization restrictions. Emulator success can validate software state-machine logic but not physical firmware policy, cache type, SMI interaction, or errata. For production evidence, capture the target CPU signature, microcode revision, capability MSRs, and any relevant specification update alongside the test result.

## Key Takeaways

1. Enumerate, legalize, initialize, then enable.
2. Root/non-root status is separate from CPL.
3. `VMXON` and VMCS regions are architectural objects with headers and address constraints.
4. Unrestricted guests are capability-dependent, not a baseline assumption.
5. Teardown order matters: `VMXOFF` precedes clearing `CR4.VMXE`.

## Connects To

- **Ch 25**: VMCS construction after VMX enablement.
- **Ch 27/28**: precise entry and exit transitions.
- **Ch 31**: instruction-level failure semantics.
