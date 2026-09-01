# Glossary

**A/D flags** — EPT accessed and dirty status bits enabled through EPTP on supporting CPUs (Ch 29).

**Active VMCS** — VMCS whose state may be maintained partly on a logical processor (Ch 25).

**APIC-access page** — Guest-physical page whose accesses can be redirected or virtualized by VMX (Ch 30).

**APIC-write exit** — Trap-like VM exit requesting VMM completion of an APIC write; qualification is a page offset (Ch 30).

**CF/ZF VMX result** — Flags encoding VMsucceed, VMfailInvalid, or VMfailValid (Ch 31).

**Combined mapping** — Cached result combining guest paging and EPT translation (Ch 29).

**Current VMCS** — Sole active VMCS targeted by VM entry and VMREAD/VMWRITE in root operation (Ch 25).

**EPT** — Extended Page Tables, translating guest-physical to host-physical addresses (Ch 29).

**EPT misconfiguration** — Architecturally illegal EPT entry or hierarchy condition; always causes a VM exit (Ch 29).

**EPT violation** — Permission/presence failure in an otherwise structurally usable EPT walk (Ch 29).

**EPTP** — EPT pointer containing root address and walk/memory-type/A-D controls (Ch 25, Ch 29).

**Exit qualification** — Exit-reason-specific diagnostic field; interpret only under its defining reason (Ch 28).

**FUP** — Intel PT flow-update packet associating flow/context change with an IP (Ch 33).

**Guest activity state** — Active, HLT, shutdown, or wait-for-SIPI state stored in VMCS (Ch 25, Ch 27).

**Guest interruptibility state** — VMCS state for STI, MOV SS, SMI, NMI, and enclave blocking (Ch 25, Ch 27).

**HLAT** — Hypervisor-managed linear-address translation facility (Ch 25, Ch 29).

**IDT-vectoring information** — VMCS information about an event whose delivery was in progress when exit occurred (Ch 28).

**INVEPT** — Instruction invalidating EPT-derived cached mappings (Ch 29, Ch 31).

**INVVPID** — Instruction invalidating cached mappings by VPID/address/context (Ch 29, Ch 31).

**IPI virtualization** — Hardware handling for eligible guest interrupt-command writes (Ch 30).

**Launch state** — Clear/launched VMCS property selecting VMLAUNCH versus VMRESUME (Ch 25).

**MTF** — Monitor Trap Flag, scheduling a VM exit after an operation boundary (Ch 26).

**PIR** — 256 posted-interrupt request bits in a posted-interrupt descriptor (Ch 30).

**PML** — Page-modification logging of guest-physical pages made dirty (Ch 29).

**Posted-interrupt descriptor** — Shared 64-byte structure holding PIR and outstanding-notification state (Ch 30).

**PSB+** — Intel PT synchronization packet block beginning with PSB and ending with PSBEND (Ch 33).

**RSM** — Instruction restoring state from SMRAM and leaving SMM (Ch 32).

**RVI/SVI** — Requested and servicing virtual-interrupt vector fields (Ch 30).

**Shadow VMCS** — VMCS accessible under VMCS-shadowing rules but not usable for VM entry (Ch 25).

**SMBASE** — Per-processor base locating SMM entry/state-save structures (Ch 32).

**SMI** — System-management interrupt that initiates SMM entry when recognized (Ch 32).

**SMM** — System Management Mode, a processor-managed firmware execution environment (Ch 32).

**SMRAM** — Protected memory for SMM handler code, data, and saved processor state (Ch 32).

**SMRR** — System-management range registers for SMRAM memory protection/cache policy (Ch 32).

**SPP** — Sub-page write permissions for selected EPT write accesses (Ch 29).

**STM** — SMM-transfer monitor used by VMX dual-monitor treatment (Ch 32).

**TIP/TNT** — Intel PT packets encoding indirect targets and conditional branch outcomes (Ch 33).

**ToPA** — Table of Physical Addresses defining Intel PT output regions and control (Ch 33).

**Virtual-APIC page** — VMCS-referenced hardware backing for virtual APIC state (Ch 30).

**VM entry** — Transition from VMX root operation into non-root guest execution (Ch 24, Ch 27).

**VM exit** — Transition from non-root guest execution to a host-state entry point (Ch 24, Ch 28).

**VMCS** — Virtual-machine control structure governing guest/host state, controls, and transition information (Ch 25).

**VMfailInvalid** — VMX instruction failure indicated by CF=1, without a valid current-VMCS error record (Ch 31).

**VMfailValid** — VMX instruction failure indicated by ZF=1 with VM-instruction error recorded (Ch 31).

**VMFUNC** — Guest instruction invoking an enabled VM function, possibly without an exit (Ch 26, Ch 31).

**VMLAUNCH/VMRESUME** — Entry instructions for clear/launched VMCS launch states (Ch 25, Ch 31).

**VMX non-root operation** — VMX execution class normally used for guest software (Ch 24, Ch 26).

**VMX root operation** — VMX execution class normally used for the VMM (Ch 24).

**VMX-preemption timer** — VMCS-controlled timer that can cause a non-root VM exit (Ch 26, Ch 27).

**VMXON region** — Aligned, revision-tagged region used to enter VMX operation (Ch 24).

**VPID** — Virtual-processor identifier tagging cached guest translations (Ch 29).

**VTPR/VPPR** — Virtual task/processor priority registers used by APIC virtualization (Ch 30).

**#VE** — Virtualization exception vector 20, optionally replacing selected convertible EPT-violation exits (Ch 26).
