# Chapter 30: APIC Virtualization and Virtual Interrupts

## Core Idea

APIC virtualization moves common interrupt-controller operations onto hardware fast paths, but only when a chain of VM-execution controls and memory structures is consistent. The VMM must still handle unsupported or policy-sensitive APIC accesses and preserve interrupt-priority semantics.

## Frameworks Introduced

- **Virtual-APIC state model**: Use the 4-KiB virtual-APIC page to hold virtual TPR, PPR, EOI, self-IPI, ICR, IRR/ISR, RVI, and SVI state as supported.
- **APIC access paths**:
  - CR8-based TPR access.
  - Memory-mapped xAPIC access through the APIC-access page.
  - MSR-based x2APIC access (`ECX` 800H–8FFH).
- **Hardware-or-exit decision**: APIC-register, TPR, EOI, self-IPI, IPI, and virtual-interrupt features emulate supported cases; remaining writes produce trap-like APIC-write exits.
- **Posted-interrupt protocol**: Deliver a physical notification, atomically merge posted-interrupt requests into virtual IRR, update RVI, and evaluate virtual delivery without a normal external-interrupt exit.

## Key Concepts

- **VTPR/VPPR**: Virtual task and processor priority registers.
- **RVI/SVI**: Requested and servicing virtual-interrupt vectors.
- **APIC-access page**: Guest-physical page whose accesses can be virtualized.
- **Virtual-APIC page**: VMCS-referenced backing state used by hardware.
- **APIC-write exit**: Trap-like exit after an operation that requires VMM completion; qualification is the page offset.
- **Posted-interrupt descriptor (PID)**: 64-byte structure with 256 request bits and an outstanding-notification bit.
- **PIR**: Posted-interrupt request bitmap in the PID.
- **IPI virtualization**: Hardware path for eligible fixed, physical-destination IPIs.

## Mental Models

- Treat interrupt virtualization as a priority queue plus delivery gate, not a simple pending bit.
- Use the control dependency chain as a feature graph; enabling a downstream control without prerequisites is invalid.
- Treat the PID as a shared concurrent data structure between CPU and other agents.

## Anti-patterns

- **Updating a posted-interrupt descriptor with ordinary stores**: software/agents must use atomic locked read-modify-write operations where required.
- **Assuming every APIC write is replayable**: APIC-write exits are trap-like; the operation has completed up to the defined emulation boundary.
- **Ignoring xAPIC/x2APIC differences**: address layout and destination-ID placement differ.
- **Using virtual interrupt delivery while mismanaging RVI/SVI/TPR**: breaks priority and in-service semantics.
- **Treating PREFETCH like an ordinary APIC access**: instruction-specific rules differ; consult the exit behavior.

## Reference Table

| Feature | Primary role | Typical prerequisite/interaction |
|---|---|---|
| Use TPR shadow | virtual TPR/CR8 | basis for several APIC features |
| Virtualize APIC accesses | xAPIC MMIO path | APIC-access and virtual-APIC pages |
| Virtualize x2APIC mode | x2APIC MSR path | MSR-bitmap interactions |
| APIC-register virtualization | hardware register reads/writes | virtual-APIC state |
| Virtual-interrupt delivery | evaluate/deliver virtual IRQs | RVI/SVI and EOI behavior |
| IPI virtualization | eligible ICR/SENDUIPI path | APIC ID/vector policy |
| Process posted interrupts | merge PID.PIR without ordinary exit | external-interrupt exiting, PID, notification vector |

## Worked Example

Posted-interrupt notification handling in hardware is conceptually:

```text
acknowledge local APIC → physical_vector
if vector != configured_notification_vector: ordinary external-interrupt exit
atomically clear PID.outstanding_notification
EOI the notification
atomically merge PID.PIR into virtual IRR and clear PIR
RVI = max(old RVI, highest newly posted vector)
evaluate whether a virtual interrupt can now be delivered
```

The VMM/device side posts a vector with a locked atomic update and sends a notification only when the outstanding-notification protocol requires it.

## Control-Dependency Review

Before entry, verify the intended path end to end:

- **TPR virtualization**: valid virtual-APIC page and “use TPR shadow”; decide threshold-exit behavior.
- **xAPIC MMIO virtualization**: valid APIC-access page and virtual-APIC page, “virtualize APIC accesses,” plus EPT mappings that do not create unintended aliases.
- **x2APIC MSR virtualization**: MSR bitmap policy must be consistent with “virtualize x2APIC mode” and APIC-register virtualization. A bitmap exit takes precedence over a hoped-for hardware fast path.
- **Virtual-interrupt delivery**: initialize RVI, SVI, virtual IRR/ISR, TPR/PPR state, EOI-exit bitmap, and entry/exit interrupt controls coherently.
- **IPI virtualization**: accept only the architecturally supported delivery/destination forms; unsupported ICR encodings must exit for VMM policy.
- **Posted interrupts**: configure external-interrupt exiting, process-posted-interrupts, notification vector, 64-byte-aligned PID address, and concurrency protocol.

## Posted-Interrupt Race Analysis

The outstanding-notification bit prevents redundant notifications without losing interrupts. A producer sets a PIR bit and uses an atomic operation to determine whether a notification is already outstanding. Hardware receiving the notification atomically clears outstanding state, merges and clears PIR, updates RVI, and reevaluates delivery. If producer and CPU use ordinary loads/stores, the following races appear:

- a vector is posted after hardware reads PIR but before it clears the same word, losing the request;
- two producers both decide no notification is outstanding and flood notifications;
- hardware clears outstanding while a producer's non-atomic set is lost, leaving PIR pending without a future wakeup;
- partially updated descriptors expose inconsistent vectors/reserved bits.

Use the exact atomic/locked protocol and memory ordering required by the platform; do not improvise a lock-free variant from the field diagram alone.

## APIC-Write Exit Handling

1. Confirm basic reason is APIC write and read qualification page offset.
2. Remember the exit is trap-like: the triggering operation has reached the architectural emulation point, and saved RIP normally references the next instruction.
3. Decode which virtual register/operation the offset represents; validate reserved bits and guest mode.
4. Complete only the remaining VMM-side action—do not re-execute the guest store.
5. Update virtual priority/in-service state and decide whether a virtual interrupt becomes deliverable.
6. Log unsupported offsets/encodings and inject the architecturally appropriate fault or enforce VMM policy.

## Priority and Blocking Checklist

- Compare virtual interrupt priority against VPPR/TPR and current in-service state.
- Respect guest `RFLAGS.IF`, STI/MOV SS blocking, activity state, and interrupt-window behavior where applicable.
- Distinguish virtual NMIs from maskable virtual interrupts.
- Ensure EOI virtualization clears/updates the correct virtual in-service vector and honors the EOI-exit bitmap.
- When entering HLT, decide whether pending virtual interrupts wake immediately or remain blocked.
- For user interrupts/SENDUIPI, validate UINV-related controls and destination translation separately from ordinary APIC IPI behavior.

## Test Scenarios

Test xAPIC and x2APIC guests, TPR changes across threshold, EOI with and without exit bitmap, self-IPI valid/invalid vectors, physical fixed IPI eligible/ineligible encodings, simultaneous posted vectors, notification coalescing, HLT wakeup, vCPU migration with pending PIR, and teardown while device producers can still post. Assertions must cover both guest-visible delivery order and absence/presence of VM exits.

## Key Takeaways

1. APIC virtualization is a family of dependent features.
2. Memory-mapped, MSR-based, and CR8 paths have different rules.
3. APIC-write exits are normally trap-like and carry a page offset.
4. Posted-interrupt state is concurrently shared and requires atomic operations.
5. Correctness includes priority, masking, in-service, EOI, and wake-state semantics.

## Connects To

- **Ch 25**: VMCS controls and pointer-backed structures.
- **Ch 27/28**: immediate delivery and exit ordering.
- **Ch 26**: changed non-root instruction/event behavior.
