# Decision Cheatsheet

## VMX Bring-up

| If you see | Do this | Because |
|---|---|---|
| CPUID lacks VMX | stop | no architectural VMX support |
| `IA32_FEATURE_CONTROL` locked without selected VMX enable | report firmware policy blocker | cannot change until power-up reset |
| `VMXON` `#UD` | check CR4.VMXE, mode, VMX support | instruction recognition failed |
| `VMXON` `#GP`/VMfail | check feature-control policy, address, revision, fixed bits | enablement contract is inconsistent |

## VM-Entry Triage

```text
CF=1 → invalid/current-VMCS class
ZF=1 → read VM-instruction error
exit bit31 + 33 → invalid guest state
exit bit31 + 34 → MSR-load entry index
exit bit31 + 41 → machine check
ordinary exit → entry succeeded; decode reason normally
```

Never stop validation after the first reported error; check order can vary.

## Exit Handler

Capture before modification:

`reason, qualification, interruption info, IDT-vectoring info, GLA/GPA validity and values, instruction length/info, guest RIP/RSP/RFLAGS, activity/interruptibility`.

| Exit semantics | RIP rule |
|---|---|
| Fault-like / retry | do not advance |
| Emulated instruction completed | advance by valid exit length |
| Trap-like | saved RIP commonly already denotes next instruction |
| Async event | do not apply instruction-length logic |

## Translation Decision

| Mutation | Invalidation family |
|---|---|
| Guest page tables/context | INVVPID |
| EPT entries/EPTP semantics | INVEPT |
| Unsure of affected scope | broader supported type |

Smells: EPT write with no shootdown; VPID 0 passed to types 0/1/3; toggled EPT A/D semantics without INVEPT; treating misconfiguration as a guest permission fault.

## APIC Virtualization

Posted interrupts require a shared-state protocol: locked PIR update → outstanding-notification decision → physical notification → hardware merge/evaluate. A racy ordinary store is not acceptable.

APIC-write exits are trap-like; use qualification offset and do not replay the original instruction blindly.

## SMM

Before claiming isolation, verify all five:

1. Per-CPU SMBASE placement.
2. Complete SMRAM/SMRR coverage and cache policy.
3. Platform lock state.
4. Saved-state validation and bounded handler behavior.
5. Multiprocessor rendezvous and negative access test.

## Intel PT

```text
enumerate → allocate output → configure with TraceEn=0
→ enable last → capture → disable first → flush/snapshot
→ decode from PSB+
```

| Need | Enable/use |
|---|---|
| Conditional branches | TNT/BranchEn |
| Indirect targets | TIP/FUP |
| Context | PIP/VMCS/MODE |
| Resynchronization | PSB+ |
| Coarse exact anchors | TSC |
| Better interval timing | MTC/TMA and/or CYC+CBR |

If an OVF or corrupt byte sequence occurs, discard semantic state until a valid PSB+ rebuilds it. In VMX non-root execution, account for TSC offset/scaling separately from MTC/CYC/CBR.
