# Chapter 33: Intel Processor Trace

## Core Idea

Intel PT emits a compressed packet stream describing control flow, context, timing, events, and power state. A correct implementation has two halves: capability-driven producer configuration and a stateful decoder that respects synchronization, packet ordering, address compression, filters, overflow, VMX, and SMM transitions.

## Frameworks Introduced

- **Producer configuration pipeline**:
  1. Enumerate Intel PT and subfeatures with CPUID.
  2. Choose single-range or ToPA output supported by the target.
  3. Allocate and initialize output buffers/tables.
  4. Configure privilege, CR3, IP, branch, timing, event, and power filters.
  5. Program output and filter MSRs while tracing is disabled.
  6. Clear relevant status and set `IA32_RTIT_CTL.TraceEn` last.
- **Safe disable/consume pipeline**: Clear `TraceEn` first → serialize/flush according to the programming model → read final output position/status → decode only committed data.
- **Packet-enable equation**: Reason about output through `PacketEn = TriggerEn && ContextEn && FilterEn && BranchEn` for branch tracing; other packet sources add their own enables.
- **Decoder state machine**: Synchronize at PSB+; maintain current IP compression base, execution mode, paging/VMCS context, timing bases, enable state, and pending deferred packets.
- **Time reconstruction**: Combine TSC (anchor), MTC/TMA (crystal-clock progress), CYC (core cycles), CBR (ratio), and software offsets; apply VMX TSC offset/scaling only where architecturally applicable.

## Key Concepts

- **TNT**: Taken/not-taken outcomes for conditional branches.
- **TIP/TIP.PGE/TIP.PGD**: Target-IP packets and packet-generation enable transitions.
- **FUP**: Flow update packet associating asynchronous/state events with an IP.
- **PIP/VMCS/MODE**: Context packets for paging, VMX, and execution mode.
- **PSB/PSBEND (PSB+)**: Synchronization boundary and associated context packet block.
- **ToPA**: Table of Physical Addresses defining one or more output regions and stop/interrupt behavior.
- **OVF**: Overflow marker; indicates lost trace and need for state recovery.
- **TSC/MTC/TMA/CYC/CBR**: Timing packets for different clock domains.
- **PTWRITE**: Software-generated trace payload when enabled.

## Mental Models

- Treat PT as a protocol, not a log of instructions.
- Treat PSB+ as a decoder checkpoint; data before a valid synchronization point may be undecodable.
- Use filters to reduce data at the source, but record configuration alongside the trace so absence of packets is interpretable.
- Treat timestamp reconstruction as estimation between exact anchors, not a guarantee of per-instruction wall time.

## Anti-patterns

- **Setting `TraceEn` before output/filter MSRs are complete**: may produce invalid or misplaced output.
- **Decoding from an arbitrary byte after loss**: search for a valid PSB sequence and rebuild state.
- **Assuming TSC packets alone give dense timing**: they are infrequent; enable MTC/CYC as accuracy and bandwidth require.
- **Ignoring LIP versus RIP and address compression**: produces wrong control-flow targets.
- **Sharing RTIT MSRs without ownership protocol**: trace collectors can corrupt each other's configuration.
- **Tracing through VMX without accounting for per-VM TSC scaling**: packet timestamps and other timing packets live in different adjustment domains.

## Reference Tables

### Output choice

| Need | Prefer | Trade-off |
|---|---|---|
| Simple bounded capture | Single range | straightforward, limited capacity/control |
| Multiple buffers/ring/interrupt/stop behavior | ToPA | more setup and table-validation complexity |

### Packet roles

| Packet family | Decoder purpose |
|---|---|
| TNT | conditional branch decisions |
| TIP/FUP | control-flow target and asynchronous-flow association |
| PIP/VMCS/MODE | execution context |
| PSB+ | resynchronization checkpoint |
| TSC/MTC/TMA/CYC/CBR | time reconstruction |
| OVF | loss boundary |
| CFE/EVD | architectural event and associated data |

## Worked Example

Configure a kernel trace using ToPA:

```text
caps = CPUID Intel-PT leaves
require ToPA and desired address/timing/filter capabilities
allocate aligned WB output regions and ToPA table
while TraceEn=0:
    set OUTPUT_BASE and OUTPUT_MASK_PTRS
    set CPL/CR3/IP filters
    set BranchEn and selected timing/event enables
    clear status/errors as specified
set TraceEn=1
run workload
clear TraceEn=0; flush/serialize; snapshot final pointers/status
decoder: find PSB+, restore context, process packets until committed end
```

Store CPUID capabilities, RTIT MSR configuration, CPU identity, VMX context policy, and buffer boundaries beside the capture so later decoding is reproducible.

## Timing Rule

Use the relationship:

```text
Timestamp ≈ CoreCrystalClock * P + AdjustedProcessorCycles + SoftwareOffset
P = CPUID.15H.EBX / CPUID.15H.EAX
```

TSC anchors correct drift; MTC/TMA refine crystal-clock progress; CYC plus CBR estimates core-cycle contribution. In non-root operation, TSC packets reflect VMX offset/scaling, while MTC/CYC/CBR are not adjusted the same way.

## Decoder State Model

Maintain explicit decoder state rather than deriving each packet independently:

- synchronization status and current PSB+ boundary;
- last IP and compression context for TIP/FUP reconstruction;
- execution mode (`CS.L`, `CS.D`) and privilege/context filters;
- paging/CR3 context and VMX root/non-root/VMCS context;
- packet-generation enable state and deferred TIP/FUP relationships;
- transactional, power, and event state when those packet families are enabled;
- last TSC anchor, crystal-clock counter, fast-counter fraction, accumulated CYC, and CBR;
- loss/overflow status and confidence interval for reconstructed time.

On OVF or corrupt ordering, invalidate all state that could depend on missing packets. Resume semantic decoding only after a valid PSB+ supplies enough context. Do not “fill in” missing branch outcomes from disassembly when asynchronous events or code modification make the path ambiguous.

## ToPA Validation Checklist

- Verify ToPA support and output-region size encodings through CPUID.
- Align the ToPA table and every output region as required; clear reserved entry bits.
- Bound the number of entries and prove the table cannot form an unintended cycle unless a deliberate circular capture is configured.
- Set END/STOP/INT and size fields according to the capture policy.
- Ensure output memory is writable by the tracing hardware and protected from untrusted modification while active.
- Initialize `IA32_RTIT_OUTPUT_BASE` and mask/pointer state with tracing disabled.
- On stop, snapshot the final table index and offset before reusing or freeing buffers.

## Filter and Bandwidth Decisions

| Goal | Controls | Cost/risk |
|---|---|---|
| One process/address space | CR3 filtering | context-switch configuration and CR3 semantics |
| Specific code ranges | IP filters | gaps outside selected ranges; preserve config metadata |
| Kernel or user only | CPL filter | loses cross-boundary context unless event packets cover it |
| Control flow | BranchEn with TNT/TIP/FUP | potentially high data rate |
| Software markers | PTWRITE | requires explicit enable and instrumentation |
| Fine timing | CYC + MTC/TMA/TSC | substantial bandwidth and decode complexity |
| Power/event analysis | power/event trace packets | capability/model-specific interpretation |

Start with the minimum packet set that answers the question. Increase timing or event detail only after measuring buffer pressure and overflow.

## VMX and SMM Integration

For system-wide tracing, the decoder crosses VM entry/exit and needs context packets plus knowledge of per-VM TSC offset/scaling. For guest-only tracing, entry/exit controls may load and clear guest PT state; failed entry and VMX abort paths need explicit treatment so the trace is not attributed to the wrong context. “Conceal VMX from PT” controls intentionally suppress some VMX indications and change what a decoder can infer.

On SMI, TraceEn is cleared by the architectural interaction described in the manual. An SMM handler that traces itself must save relevant RTIT state, establish its own legal output (not in SMRR memory), enable only after setting a suitable execution environment, disable before `RSM`, and restore the interrupted owner's state. Multiple collectors should honor an ownership protocol; silently overwriting enabled RTIT MSRs corrupts both captures.

## Trace Acceptance Tests

1. Decode a known direct/conditional/indirect branch microbenchmark and compare control-flow edges.
2. Force PSB insertion and verify restart from each boundary.
3. Trigger output wrap/STOP/interrupt and confirm pointer math.
4. Induce overflow and prove the decoder rejects the uncertain region until resynchronization.
5. Cross user/kernel, CR3, VM entry/exit, interrupts, and SMI where authorized; verify context packets and filter behavior.
6. Compare timestamp reconstruction against a controlled workload and report error bounds, not just point estimates.
7. Persist CPU signature, microcode, CPUID leaves, RTIT MSRs, ToPA metadata, and decoder version with the trace.

## Key Takeaways

1. Enumerate every PT capability before programming its MSR bit.
2. Configure with tracing disabled and enable last.
3. Decode as a synchronized state machine.
4. Record configuration/provenance with the packet bytes.
5. Timing between anchors is reconstructed and may contain uncertainty.
6. VMX and SMM change trace enable/context behavior and require explicit handling.

## Connects To

- **Ch 24/25**: VMX controls that load, clear, or conceal PT state.
- **Ch 28**: trace packets around exits.
- **Ch 32**: SMI clears `TraceEn`; SMM tracing has special rules.
