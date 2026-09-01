# sec-book-skills

Private collection of Agent Skills distilled from systems-security books and
manuals. The repository follows the common multi-skill layout:

```text
skills/
├── computer-systems-a-programmers-perspective/
│   └── SKILL.md
├── intel-system-programming-vol3c/
│   └── SKILL.md
├── linking-loading-and-libraries/
│   └── SKILL.md
├── linux-virtualization-principles-implementation/
│   └── SKILL.md
└── processor-virtualization-technology/
    └── SKILL.md
```

## Available skills

| Skill | Scope |
|---|---|
| `computer-systems-a-programmers-perspective` | Data representation, x86-64 machine code, processor architecture, optimization, memory hierarchy, linking, processes, virtual memory, system I/O, networking, and concurrency |
| `intel-system-programming-vol3c` | Intel VMX/VMCS, VM entry and exit, EPT/VPID, APIC virtualization, SMM, and Intel Processor Trace |
| `linking-loading-and-libraries` | Compilation, ELF/PE/COFF, symbols, relocation, loading, dynamic linking, ABI, CRT, memory, and system calls |
| `linux-virtualization-principles-implementation` | KVM/x86 virtualization, VMX and VM exits, Guest memory translation, interrupt and PCI virtualization, Virtio/Virtqueue, and Overlay/OVS packet paths |
| `processor-virtualization-technology` | Intel VMX operation, VMCS configuration, VM entry and exit, EPT/VPID cache domains, exception reflection, and Local APIC virtualization |

Each skill contains a concise `SKILL.md`, chapter references loaded on demand,
a glossary, reusable patterns, and a decision-oriented cheatsheet.

## Install with `npx skills`

This is a private GitHub repository. Authenticate your SSH key for
`github.com` first, then list the skills without installing:

```bash
npx skills add git@github.com:CUMT-syssec/sec-book-skills.git --list
```

Install one skill globally for Codex:

```bash
npx skills add git@github.com:CUMT-syssec/sec-book-skills.git \
  --skill computer-systems-a-programmers-perspective \
  --global --agent codex --yes
```

```bash
npx skills add git@github.com:CUMT-syssec/sec-book-skills.git \
  --skill linking-loading-and-libraries \
  --global --agent codex --yes
```

```bash
npx skills add git@github.com:CUMT-syssec/sec-book-skills.git \
  --skill intel-system-programming-vol3c \
  --global --agent codex --yes
```

```bash
npx skills add git@github.com:CUMT-syssec/sec-book-skills.git \
  --skill linux-virtualization-principles-implementation \
  --global --agent codex --yes
```

```bash
npx skills add git@github.com:CUMT-syssec/sec-book-skills.git \
  --skill processor-virtualization-technology \
  --global --agent codex --yes
```

Install every skill in the repository:

```bash
npx skills add git@github.com:CUMT-syssec/sec-book-skills.git \
  --skill '*' --global --agent codex --yes
```

For other supported agents, omit `--agent codex` for interactive selection or
replace `codex` with the target agent identifier supported by the current CLI.

## Validate a local checkout

From the repository root:

```bash
npx skills add . --list
```

The expected result is exactly these five skill names:

```text
computer-systems-a-programmers-perspective
intel-system-programming-vol3c
linking-loading-and-libraries
linux-virtualization-principles-implementation
processor-virtualization-technology
```

## Content and distribution boundary

These skills are synthesized learning notes and operational decision aids. The
source PDFs/manuals are not included. The underlying publications are
third-party copyrighted works, so this repository must remain private unless
the relevant public redistribution rights are established separately.

Generated guidance can contain OCR or interpretation errors. Verify exact
commands, bit fields, structure offsets, ABI details, and hardware behavior
against current authoritative manuals and the target environment.
