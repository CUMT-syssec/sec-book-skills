# Chapter 31: VMX Instruction Reference

## Core Idea

VMX instructions have three distinct outcome families—architectural exception, VM exit, and VM-instruction success/failure flags. Robust code must interpret them separately and must use the exact instruction preconditions, operand format, current-VMCS state, and capability enumeration.

## Frameworks Introduced

- **Outcome decoder**:
  - Exception (`#UD`, `#GP`, `#PF`, `#SS`): instruction did not report VMfail.
  - VM exit: instruction executed in non-root operation and interception/semantics transferred control.
  - VMsucceed: `CF=0`, `ZF=0`.
  - VMfailInvalid: `CF=1`, usually no valid current VMCS where required.
  - VMfailValid: `CF=0`, `ZF=1`, with error number written to current VMCS.
- **Instruction families**:
  - VMCS maintenance: `VMPTRLD`, `VMPTRST`, `VMCLEAR`, `VMREAD`, `VMWRITE`.
  - VMX lifecycle/transition: `VMXON`, `VMXOFF`, `VMLAUNCH`, `VMRESUME`.
  - Translation invalidation: `INVEPT`, `INVVPID`.
  - Guest interface: `VMCALL`, `VMFUNC`.
- **Check-before-side-effect rule**: Operand faults, CPL, mode, VMX operation, root/non-root status, pointer validity, and field support are ordered by each instruction definition; do not infer behavior from mnemonic alone.

## Key Concepts

- **VMsucceed**: Clears relevant arithmetic flags including CF and ZF.
- **VMfailInvalid**: Sets CF and clears ZF; no valid current VMCS is available for a detailed error.
- **VMfailValid**: Sets ZF, clears CF, and records a VM-instruction error number.
- **VMCS-field encoding**: Operand selecting an architectural VMCS component for `VMREAD/VMWRITE`.
- **INVEPT descriptor**: 128-bit operand whose low 64 bits contain EPTP for single-context invalidation.
- **INVVPID descriptor**: 128-bit operand containing VPID and linear address.
- **Compatibility mode restriction**: VMX instructions covered here are not recognized in compatibility mode.

## Mental Models

- Treat CF/ZF as a small typed result enum, not generic arithmetic status.
- Keep instruction wrappers small and side-effect explicit: invoke, capture flags immediately, then decode.
- Validate capabilities before choosing an invalidation type; unsupported type is a VMfail, not a portable fallback.

## Anti-patterns

- **Checking only ZF**: misses VMfailInvalid via CF.
- **Executing another flag-changing instruction before saving RFLAGS**: destroys VMX result evidence.
- **Using `VMLAUNCH` after a successful launch**: launch state is already launched; use `VMRESUME`.
- **Passing VPID 0 to INVVPID types requiring nonzero VPID**: produces invalid-operand VMfail.
- **Assuming unsupported VMCS fields read as zero**: `VMREAD/VMWRITE` may VMfailValid.

## Reference Tables

### Result decode

| CF | ZF | Meaning | Next step |
|---:|---:|---|---|
| 0 | 0 | VMsucceed | continue |
| 1 | 0 | VMfailInvalid | inspect current-VMCS/pointer/preconditions |
| 0 | 1 | VMfailValid | read VM-instruction error |
| 1 | 1 | not a defined VMX success/failure result | treat wrapper/evidence as corrupt |

### Invalidation scope

| Instruction | Types | Selector |
|---|---|---|
| `INVEPT` | 1 single EPT context; 2 all EPT contexts | EPTP |
| `INVVPID` | 0 address+VPID; 1 VPID; 2 all nonzero VPIDs; 3 VPID retaining globals | VPID and optional linear address |

## Code Example

Capture flags immediately after a VMX instruction (illustrative pseudocode, adapt syntax and clobbers to your toolchain):

```c
vmx_result r;
asm volatile("vmptrld %[pa]; setc %[cf]; setz %[zf]"
             : [cf] "=qm"(r.cf), [zf] "=qm"(r.zf)
             : [pa] "m"(vmcs_pa)
             : "cc", "memory");
if (r.cf) return VMFAIL_INVALID;
if (r.zf) return read_vm_instruction_error();
return VMX_SUCCESS;
```

The exact operand direction and assembler syntax must be verified for the selected compiler; the invariant is immediate flag capture with `cc` and memory effects declared.

## Worked Example

Choose an EPT invalidation after changing one EPT hierarchy:

```text
caps = RDMSR(IA32_VMX_EPT_VPID_CAP)
if caps supports single-context INVEPT:
    descriptor.eptp = active_eptp
    execute INVEPT type 1; decode CF/ZF
else if caps supports global INVEPT:
    execute INVEPT type 2; decode CF/ZF
else:
    feature design is invalid; do not pretend a software fence replaces architectural invalidation
```

## Safe Wrapper Design

A VMX instruction wrapper should expose a typed result rather than a Boolean:

```text
Success
VMfailInvalid
VMfailValid(error_number)
ArchitecturalException(vector, error_code, fault_address)
VMExit(exit_record)
```

Keep the assembly stub responsible only for correct operand encoding, immediate flag capture, compiler barriers/clobbers, and saving registers required by the calling convention. Keep policy and error reporting in higher-level code. This separation makes it possible to unit-test result decoding without executing VMX and to audit the small privileged assembly surface independently.

Important compiler concerns include operand width in 32/64-bit modes, memory versus register constraints, `cc` clobber, memory ordering/compiler reordering, and preserving the flags before a function epilogue changes them. Disassemble the final binary; inline-assembly source that looks correct can still produce the wrong operand order or size for a particular assembler dialect.

## Instruction-Family Checklists

### VMCS maintenance

- `VMPTRLD`: operand is the physical address of a compatible VMCS region; header revision/shadow bit and address constraints must be legal.
- `VMCLEAR`: writes processor-maintained state back, makes the VMCS inactive, and clears launch state. Coordinate so it is not being used elsewhere.
- `VMREAD/VMWRITE`: validate field encoding and access direction; exit-information fields may be read-only depending on capability.
- In non-root operation with VMCS shadowing, bitmap policy determines whether VMREAD/VMWRITE access the linked shadow VMCS or exit.

### Entry and lifecycle

- `VMXON`: requires processor support, legal mode/control state, policy authorization, and a VMXON region.
- `VMXOFF`: only after all guest execution and dependent state are quiesced; current VMCS concepts cease with VMX operation.
- `VMLAUNCH`: current VMCS, clear launch state, complete checks; success does not imply any guest instruction ran.
- `VMRESUME`: current VMCS, launched state; use for all later entries unless VMCLEAR deliberately resets lifecycle.

### Invalidation

- Enumerate supported INVEPT/INVVPID instruction and type bits.
- Construct the full 128-bit descriptor with reserved bits zero.
- For single EPT context, EPTP itself must be a value that could pass entry validation.
- For individual VPID address, require nonzero VPID and canonical linear address.
- Decode VMfail; an invalid descriptor must not be treated as a successful conservative flush.

## Why Exception vs. VMfail Matters

An exception usually indicates the instruction could not be used in the current architectural context—wrong CPL, unsupported/mode-disabled instruction, or memory-operand fault. VMfail indicates the instruction was recognized in a usable VMX context but its VMX-specific state or operand was invalid. A VM exit from non-root operation is yet another policy path. Logging all three as “VMX instruction failed” loses the evidence needed to fix the right layer.

## Negative-Test Matrix

| Test | Expected channel |
|---|---|
| VMX instruction outside VMX operation | generally `#UD` per instruction |
| CPL > 0 in root operation | `#GP(0)` where defined |
| faulting memory operand | `#PF`/`#SS` before later VMX checks as specified |
| VMREAD with no current VMCS | VMfailInvalid |
| unsupported VMCS field | VMfailValid with error |
| VMLAUNCH on launched VMCS | VMfailValid |
| VMCALL in non-root operation | VM exit |
| disabled VMFUNC | VM exit or `#UD` according to enable/function state |
| unsupported INVEPT type | VMfailValid invalid operand |

Run these only in an authorized test hypervisor with exception/exit recovery. A mistaken bare-metal VMX instruction can crash the host kernel.

## Key Takeaways

1. Exceptions, exits, VMfailInvalid, and VMfailValid are different channels.
2. Capture CF/ZF before any flag-clobbering instruction.
3. VMCS launch/current state determines legal instruction use.
4. Capability enumeration selects legal invalidation types.
5. Operand and mode rules are security-relevant preconditions, not incidental details.

## Connects To

- **Ch 24**: VMX lifecycle.
- **Ch 25**: VMCS state and fields.
- **Ch 27**: transition failure diagnosis.
- **Ch 29**: invalidation obligations.
