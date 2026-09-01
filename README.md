# sec-book-skills

Private collection of Agent Skills distilled from systems-security books and
manuals. The repository follows the common multi-skill layout:

```text
skills/
├── intel-system-programming-vol3c/
│   └── SKILL.md
├── linking-loading-and-libraries/
│   └── SKILL.md
└── linux-virtualization-principles-implementation/
    └── SKILL.md
```

## Available skills

| Skill | Scope |
|---|---|
| `linking-loading-and-libraries` | Compilation, ELF/PE/COFF, symbols, relocation, loading, dynamic linking, ABI, CRT, memory, and system calls |
| `intel-system-programming-vol3c` | Intel VMX/VMCS, VM entry and exit, EPT/VPID, APIC virtualization, SMM, and Intel Processor Trace |
| `linux-virtualization-principles-implementation` | KVM/x86 virtualization, VMX and VM exits, Guest memory translation, interrupt and PCI virtualization, Virtio/Virtqueue, and Overlay/OVS packet paths |

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

The expected result is exactly these three skill names:

```text
intel-system-programming-vol3c
linking-loading-and-libraries
linux-virtualization-principles-implementation
```

## Content and distribution boundary

These skills are synthesized learning notes and operational decision aids. The
source PDFs/manuals are not included. The underlying publications are
third-party copyrighted works, so this repository must remain private unless
the relevant public redistribution rights are established separately.

Generated guidance can contain OCR or interpretation errors. Verify exact
commands, bit fields, structure offsets, ABI details, and hardware behavior
against current authoritative manuals and the target environment.
