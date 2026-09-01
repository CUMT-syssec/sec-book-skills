# Patterns

## Capability-Derived Control Word

**When to use**: Before writing any VMX control vector.

**How**:

1. Read the applicable basic/true control capability MSR.
2. Reject requested 1-bits not allowed by its high half.
3. Force must-be-1 bits derived from its low half.
4. Clear must-be-0 bits.
5. Validate dependencies across primary, secondary, tertiary, entry, and exit controls.

**Trade-offs**: Slightly more initialization logic; prevents processor-version-dependent entry failures.

## Per-vCPU VMCS Lifecycle

**When to use**: Creating or scheduling a virtual CPU.

**How**: Allocate aligned WB memory → write revision header → `VMCLEAR` → `VMPTRLD` → populate all groups → `VMLAUNCH` once → `VMRESUME` thereafter. Serialize migration so a VMCS is not current on two logical processors; clear/load under the required coordination.

**Trade-offs**: More per-vCPU memory/state tracking; greatly simplifies launch-state and concurrency reasoning.

## VM-Entry Failure Ladder

**When to use**: `VMLAUNCH` or `VMRESUME` does not run the guest as expected.

**How**:

1. Capture CF/ZF immediately.
2. On CF, inspect current VMCS and instruction preconditions.
3. On ZF, read VM-instruction error.
4. On host-state return, check exit-reason bit 31 and reason 33/34/41.
5. If entry succeeded, decode the immediate ordinary exit.
6. Rerun the full grouped VMCS validator after every fix.

**Trade-offs**: More diagnostics; avoids random field changes and false fixes.

## Typed Exit Decoder

**When to use**: Every VM-exit handler.

**How**: Snapshot the complete exit bundle; dispatch on basic reason; parse only fields defined for that reason; classify completion semantics; emulate/deny/retry; advance RIP only when the instruction completed or emulation replaced it; preserve/reinject in-flight events.

**Trade-offs**: Larger handler schema; makes exit behavior auditable and testable.

## EPT Permission Update with Shootdown

**When to use**: Changing an EPT mapping or access permission.

**How**: Lock mapping domain → atomically update EPT entry → publish memory write → coordinate all logical processors using the EPTP → perform sufficient INVEPT scope → release → retry blocked access only after policy validation.

**Trade-offs**: Cross-CPU cost; required for reliable revocation and remapping.

## Narrow Translation Invalidation

**When to use**: Guest paging or EPT state changes.

**How**: Identify whether stale state is guest-linear (`INVVPID`) or EPT-derived (`INVEPT`); list affected VPID/EPTP/PCID/addresses; choose the narrowest enumerated type covering all; broaden when proof is uncertain.

**Trade-offs**: Narrow invalidations preserve performance but demand stronger mutation tracking.

## Posted-Interrupt Producer Protocol

**When to use**: Device/VMM posts a virtual interrupt without an ordinary exit.

**How**: Atomically set the PIR vector bit; atomically set/test outstanding notification; send the configured physical notification when required; let hardware clear notification and merge PIR; never modify shared descriptor fields with racy stores.

**Trade-offs**: Lower exit rate with more concurrent-state complexity.

## SMM Minimal Trusted Handler

**When to use**: Firmware SMI service design.

**How**: Relocate per-CPU SMBASE → minimize handler and writable data → validate saved-state pointers/values → configure SMRR/cache policy → lock SMRAM → use bounded multiprocessor rendezvous → avoid dependencies on locks held by non-SMM code → negative-test non-SMM access.

**Trade-offs**: Less handler functionality; smaller firmware attack surface and more predictable latency.

## Intel PT Reproducible Capture

**When to use**: Collecting traces for debugging, security, or performance work.

**How**: Record CPU/CPUID capabilities → configure buffers/ToPA and filters with `TraceEn=0` → enable last → run → disable first → flush/snapshot pointers/status → save MSR configuration beside bytes → decode from PSB+.

**Trade-offs**: Extra metadata and lifecycle code; makes traces portable and interpretable.

## Trace Resynchronization After Loss

**When to use**: Decoder starts mid-stream or encounters OVF/corruption.

**How**: Mark state unknown → scan for a valid PSB → parse the complete PSB+ context block → rebuild IP/mode/paging/VMCS/timing state → resume semantic decoding only after PSBEND.

**Trade-offs**: Discards an undecodable interval; prevents invented control flow.
