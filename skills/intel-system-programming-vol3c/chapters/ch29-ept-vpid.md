# Chapter 29: VMX Support for Address Translation

## Core Idea

EPT and VPID reduce virtualization overhead by separating guest-linear translation from guest-physical translation and by tagging cached mappings. Correctness depends on permission semantics, EPT-misconfiguration versus EPT-violation handling, and explicit cache invalidation after translation changes.

## Frameworks Introduced

- **Two-stage translation**: Guest paging maps guest linear → guest physical; EPT maps guest physical → host physical.
- **Failure distinction**:
  - EPT misconfiguration: an EPT entry contains an architecturally illegal combination; always VM exits.
  - EPT violation: translation is structurally valid but permissions/presence reject the access; may exit or become #VE when eligible.
- **Mapping identity**: Reason about cached translations using VPID, PCID, EPTP root, and address.
- **Invalidation selection**:
  - Use `INVVPID` after changes affecting guest-linear translations/context.
  - Use `INVEPT` after changes to EPT paging structures or EPT semantics.
  - Select individual/single/global scope only if that type is enumerated and covers all stale entries.
- **Permission accumulation**: Effective EPT access is constrained by permissions across every walked entry; mode-based execute control can distinguish supervisor and user execution.

## Key Concepts

- **EPTP**: EPT root physical address plus EPT memory type, page-walk length, and A/D enable.
- **VPID**: 16-bit tag associated with cached guest translations; VPID 0 retains special non-tagged semantics.
- **Combined mapping**: Cached result of guest paging plus EPT translation.
- **EPT violation qualification**: Reports attempted access type, permissions, translation context, and validity of guest-linear information.
- **Accessed/dirty flags**: Optional EPT hardware updates enabled by EPTP bit 6 on supporting processors.
- **Page-modification logging (PML)**: Logs guest-physical pages made dirty until the log becomes full.
- **Sub-page write permissions (SPP)**: Finer write control for eligible accesses.
- **HLAT**: Hypervisor-managed linear-address translation mechanism with its own control fields and cache implications.

## Mental Models

- Think of translation caches as derived state: page-table writes are incomplete until stale derived mappings can no longer be used.
- Use EPT violation for policy; treat misconfiguration as a VMM bug or corrupted structure.
- Associate every invalidation with a concrete mutation and a proof of scope.

## Anti-patterns

- **Editing an EPT PTE and immediately resuming**: stale EPT or combined mappings may remain.
- **Treating INVEPT and INVVPID as interchangeable**: they invalidate different dimensions.
- **Using VPID 0 with individual/single-context INVVPID types**: those operands are invalid.
- **Assuming A/D flags start working after toggling EPTP bit 6 without invalidation**: cached mappings may prevent expected updates.
- **Converting all EPT violations to #VE**: misconfigurations and non-convertible conditions still exit.

## Reference Tables

### INVVPID types

| Type | Scope | Key constraint |
|---:|---|---|
| 0 | one linear address + VPID | VPID nonzero; canonical address |
| 1 | one VPID | VPID nonzero |
| 2 | all nonzero VPIDs | broader context invalidation |
| 3 | one VPID, retain globals | VPID nonzero; only when supported |

### INVEPT types

| Type | Scope | Descriptor use |
|---:|---|---|
| 1 | one EPT context | valid EPTP required |
| 2 | all EPT contexts | global EPT-derived invalidation |

## Worked Example

Revoke guest write access to one page:

```text
lock EPT mutation domain
atomically clear write permission in leaf EPT entry
ensure page-table write is globally visible
execute supported INVEPT scope covering this EPTP
unlock
resume vCPU; a later write must produce violation/#VE per policy
```

If other logical processors may run vCPUs using the same EPTP, coordinate a cross-CPU invalidation protocol before treating the revocation as enforced.

## EPT Walk Review Checklist

- Verify EPTP memory type and page-walk length are enumerated and correctly encoded.
- Verify the EPT root address is aligned, within physical-address width, and has reserved bits clear.
- At each level, separate three states: non-present/permission denied, legal pointer or leaf, and misconfigured encoding.
- Accumulate read/write/execute permissions across the walk; a permissive leaf cannot override a restrictive parent.
- For leaf entries, validate large-page support/alignment, memory type, ignore-PAT behavior, suppress-#VE, SPP, and mode-based execute bits only when supported.
- If A/D is enabled, check how accessed and dirty bits are created and cleared and what invalidation is required before observing new transitions.
- On violation, decode whether GLA is valid and whether the access was a read, write, instruction fetch, page walk, or other defined class before applying policy.

## Invalidation Proof Template

For every mutation, write down:

1. **Changed source**: guest PTE, EPT entry, EPTP, PCID state, CR3, permission mode, or A/D semantics.
2. **Affected derived mapping**: guest-linear, guest-physical, or combined.
3. **Tags**: VPID, PCID, EPTP root, address range, global status.
4. **Running consumers**: logical processors/vCPUs that might have cached it.
5. **Architectural invalidation**: supported INVVPID/INVEPT type and descriptor.
6. **Completion barrier**: point after all consumers have acknowledged invalidation.

If any dimension is unknown, choose a broader invalidation scope. Performance optimization comes after a correctness proof.

## Why VPID Is Not an ASID-Free Pass

VPID lets translations from different virtual processors coexist in caches, reducing flushes at VM transitions. It also makes VPID reuse dangerous: stale translations tagged with a reused value can become visible to a different address space unless the required INVVPID scope runs before reuse. VPID 0 is special and does not provide the ordinary tagging behavior expected from a nonzero identifier.

EPTP switching similarly allows multiple EPT contexts to coexist. Switching the pointer does not prove old translations are gone; it changes which tagged mappings are used/created. Modifying a hierarchy already associated with an EPTP still requires INVEPT discipline.

## EPT Violation Policy Examples

| Cause | Safe policy question | Possible action |
|---|---|---|
| demand paging | is GPA valid and owned by guest? | map, INVEPT if needed, retry |
| copy-on-write | is write authorized and page shared? | clone, update leaf, invalidate, retry |
| executable-policy fault | should this page ever execute? | deny/inject/terminate; do not auto-add X |
| MMIO trap | is GPA in an emulated device range? | emulate access with instruction evidence |
| dirty tracking | is write-protect intentionally used for logging? | record then restore W under synchronized policy |
| EPT misconfiguration | none—encoding is illegal | stop/rescue VMM path; repair structure |

## Concurrency Failure Modes

- Updating a multiword or shared EPT entry without a safe atomic/publication protocol.
- Freeing a page-table page before every CPU has stopped walking/using it.
- Revoking access locally while another CPU retains a combined mapping.
- Reusing an EPTP root or VPID before invalidation completes.
- Reading A/D bits while hardware may still update them without synchronization.

## Key Takeaways

1. EPT is a second translation stage with its own permissions and caches.
2. Misconfiguration indicates illegal EPT encoding; violation indicates a denied access.
3. VPID tags translations but does not eliminate invalidation obligations.
4. Invalidation scope must cover EPTP/VPID/PCID/address dimensions affected by the change.
5. Cross-vCPU coordination is part of memory-permission correctness.

## Connects To

- **Ch 26**: EPTP switching and #VE.
- **Ch 28**: EPT exit evidence.
- **Ch 31**: INVEPT/INVVPID instruction rules.
