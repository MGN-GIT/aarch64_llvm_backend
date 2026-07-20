# 📘 AArch64 Memory Management Guide
### *A Beginner-Friendly Deep Dive into ARMv8-A Memory Management*

> **Source:** Learn the Architecture – AArch64 Memory Management (101811_0103_03_en, Version 1.3)  
> **Architecture:** ARMv8-A / ARMv9-A | **State:** AArch64

---

## 📚 Table of Contents

1. [What is Memory Management?](#1-what-is-memory-management)
2. [Virtual vs Physical Addresses](#2-virtual-vs-physical-addresses)
3. [Address Spaces in AArch64](#3-address-spaces-in-aarch64)
4. [Physical Address Spaces (PAS)](#4-physical-address-spaces-pas)
5. [Address Sizes](#5-address-sizes)
6. [Address Space Identifiers (ASID)](#6-address-space-identifiers-asid)
7. [Virtual Machine Identifiers (VMID)](#7-virtual-machine-identifiers-vmid)
8. [The Memory Management Unit (MMU)](#8-the-memory-management-unit-mmu)
9. [Translation Table Entries](#9-translation-table-entries)
10. [Multi-Level Translation Tables](#10-multi-level-translation-tables)
11. [Translation Table Formats](#11-translation-table-formats)
12. [Translation Granule](#12-translation-granule)
13. [Key Registers That Control Address Translation](#13-key-registers-that-control-address-translation)
14. [Translation Lookaside Buffer (TLB) Maintenance](#14-translation-lookaside-buffer-tlb-maintenance)
15. [Address Translation (AT) Instructions](#15-address-translation-at-instructions)
16. [Quick Reference: Glossary](#16-quick-reference-glossary)
17. [Knowledge Check Q&A](#17-knowledge-check-qa)

---

## 1. What is Memory Management?

Memory management is the mechanism by which the hardware **controls access to memory** in a system. Every time the OS or an application accesses memory, the hardware performs memory management behind the scenes.

### Why Do We Need It?

Modern processors (like those running Linux) are designed to support **virtual memory systems**. This means:

- Software **never sees real (physical) addresses** — it only sees **virtual addresses**.
- The processor **translates** virtual addresses → physical addresses automatically.
- Physical addresses point to the **actual locations** in RAM, Flash, or peripherals.

### The Big Picture

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHY VIRTUAL ADDRESSES?                       │
│                                                                 │
│  ┌──────────────┐    Translation    ┌──────────────────────┐   │
│  │  Application │  ─────────────►  │  Physical Memory     │   │
│  │  (sees VA)   │   (done by MMU)  │  (RAM, Flash, etc.)  │   │
│  └──────────────┘                  └──────────────────────┘   │
│                                                                 │
│  Benefits:                                                      │
│  ✅ OS controls what memory each app can see                   │
│  ✅ Apps are isolated from each other (sandboxing)             │
│  ✅ Fragmented physical memory looks contiguous to apps        │
│  ✅ Developers don't need to know physical addresses           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Virtual vs Physical Addresses

### Virtual Address (VA)
- The address that **software uses** (what your program sees).
- Managed by the OS — the OS decides what virtual address maps to what physical address.
- Stored in **translation tables** (also called page tables).

### Physical Address (PA)
- The **real address** in the hardware memory system.
- Points to actual bytes in RAM, ROM, Flash, or peripheral registers.

### How the Mapping Works

```
┌──────────────────────────────────────────────────────────────────────┐
│              VIRTUAL → PHYSICAL ADDRESS TRANSLATION                  │
│                                                                      │
│   Virtual Address Space          Physical Address Space              │
│   ┌──────────────────┐           ┌──────────────────────┐           │
│   │  Kernel Code     │──────────►│  DDR RAM             │           │
│   │  Kernel Data     │──────────►│  DDR RAM             │           │
│   │  App Code        │──────────►│  Flash               │           │
│   │  App Data        │──────────►│  SRAM                │           │
│   │  (unmapped)      │           │  ROM                 │           │
│   └──────────────────┘           │  Peripherals         │           │
│                                  └──────────────────────┘           │
│                                                                      │
│   Translation Tables (in memory, managed by OS/hypervisor)          │
│   ┌──────────────────────────────────────────────────────┐          │
│   │  VA 0x0000_8000  →  PA 0x4000_0000  (App Code)       │          │
│   │  VA 0xFFFF_0000  →  PA 0x8000_0000  (Kernel Data)    │          │
│   │  ...                                                  │          │
│   └──────────────────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────────────┘
```

> 💡 **Key Insight:** Each application can use the **same virtual addresses** (e.g., `0x8000`) but they map to **different physical locations**. When the OS switches between apps, it reprograms the translation tables.

---

## 3. Address Spaces in AArch64

AArch64 has **multiple independent virtual address spaces**. Each has its own translation tables and settings — called a **translation regime**.

### The Exception Levels (ELs)

```
┌──────────────────────────────────────────────────────────────────────┐
│                  EXCEPTION LEVELS IN ARMv8-A                         │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  EL3 – Secure Monitor (Firmware / TrustZone)                │   │
│   │         Most privileged. Controls security state.           │   │
│   ├─────────────────────────────────────────────────────────────┤   │
│   │  EL2 – Hypervisor (e.g., KVM, Xen)                         │   │
│   │         Manages virtual machines (VMs).                     │   │
│   ├─────────────────────────────────────────────────────────────┤   │
│   │  EL1 – OS Kernel (e.g., Linux kernel)                       │   │
│   │         Manages apps, drivers, memory.                      │   │
│   ├─────────────────────────────────────────────────────────────┤   │
│   │  EL0 – User Applications (e.g., your app)                   │   │
│   │         Least privileged. Runs user code.                   │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   Higher EL = More Privilege = More Control                          │
└──────────────────────────────────────────────────────────────────────┘
```

### Virtual Address Spaces

```
┌──────────────────────────────────────────────────────────────────────┐
│              VIRTUAL ADDRESS SPACES IN ARMv8-A                       │
│                                                                      │
│  EL0/EL1 Virtual Address Space (Split into two regions):            │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │  0xFFFF_FFFF_FFFF_FFFF ┐                                 │       │
│  │         ...            │  KERNEL SPACE (upper)           │       │
│  │  0xFFFF_0000_0000_0000 ┘  ← controlled by TTBR1_EL1     │       │
│  │  ~~~ INVALID GAP ~~~                                     │       │
│  │  0x0000_FFFF_FFFF_FFFF ┐                                 │       │
│  │         ...            │  USER SPACE (lower)             │       │
│  │  0x0000_0000_0000_0000 ┘  ← controlled by TTBR0_EL1     │       │
│  └──────────────────────────────────────────────────────────┘       │
│                                                                      │
│  EL2 / EL3 Virtual Address Space (Single region):                   │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │  0x0000_FFFF_FFFF_FFFF ┐                                 │       │
│  │         ...            │  SINGLE SPACE                   │       │
│  │  0x0000_0000_0000_0000 ┘  ← controlled by TTBR0_EL2/3   │       │
│  └──────────────────────────────────────────────────────────┘       │
└──────────────────────────────────────────────────────────────────────┘
```

> 💡 **Why two regions for EL0/EL1?** Kernel space and user space have **separate translation tables**, keeping their mappings isolated. The kernel always lives in the upper region; apps live in the lower region.

### Stage 1 and Stage 2 Translation (Virtualization)

When a **hypervisor** is present, EL0/EL1 translations go through **two stages**:

```
┌──────────────────────────────────────────────────────────────────────┐
│              TWO-STAGE ADDRESS TRANSLATION                           │
│                                                                      │
│  ┌──────────┐  Stage 1   ┌──────────┐  Stage 2   ┌──────────────┐  │
│  │ Virtual  │ ─────────► │  IPA     │ ─────────► │  Physical    │  │
│  │ Address  │  (OS       │(Intermed.│ (Hypervisor│  Address     │  │
│  │  (VA)    │  tables)   │ Phys.    │  tables)   │   (PA)       │  │
│  │          │            │ Address) │            │              │  │
│  └──────────┘            └──────────┘            └──────────────┘  │
│                                                                      │
│  • Stage 1: OS thinks IPA = real physical address                   │
│  • Stage 2: Hypervisor translates IPA → real PA                     │
│  • This lets the hypervisor control what physical memory a VM sees  │
└──────────────────────────────────────────────────────────────────────┘
```

| Term | Full Name | Who Controls It | What It Does |
|------|-----------|-----------------|--------------|
| **VA** | Virtual Address | Application/OS | Address software uses |
| **IPA** | Intermediate Physical Address | OS (thinks it's PA) | Output of Stage 1 |
| **PA** | Physical Address | Hypervisor/Hardware | Real hardware address |

---

## 4. Physical Address Spaces (PAS)

AArch64 has **multiple physical address spaces** for security isolation:

```
┌──────────────────────────────────────────────────────────────────────┐
│              PHYSICAL ADDRESS SPACES (PAS)                           │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Non-secure PAS  │  Secure PAS  │  Realm PAS*  │  Root PAS* │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                          (* ARMv9-A only)                            │
│                                                                      │
│  Which PAS you can access depends on your Security State:           │
│                                                                      │
│  Security State      │  Can Access                                  │
│  ──────────────────────────────────────────────────────────────     │
│  Non-secure state    │  Non-secure PAS only                         │
│  Secure state        │  Secure PAS + Non-secure PAS                 │
│  Realm state         │  Realm PAS + Non-secure PAS                  │
│  Root state          │  ALL physical address spaces                 │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 5. Address Sizes

### Virtual Address Size

Virtual addresses are stored in **64-bit registers** (X registers), but **not all 64 bits are valid**. The valid range is controlled by the `TCR_ELx.TnSZ` field:

```
┌──────────────────────────────────────────────────────────────────────┐
│              VIRTUAL ADDRESS SIZE CALCULATION                        │
│                                                                      │
│  Formula:  Number of address bits = 64 - TnSZ                       │
│                                                                      │
│  Example: If TCR_EL1.T1SZ = 32                                      │
│           → Address bits = 64 - 32 = 32 bits                        │
│           → Kernel space: 0xFFFF_FFFF_0000_0000 to                  │
│                           0xFFFF_FFFF_FFFF_FFFF                      │
│                                                                      │
│  Example: If TCR_EL1.T0SZ = 34                                      │
│           → Address bits = 64 - 34 = 30 bits (1 GB user space)      │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │  Bit 63                                          Bit 0   │       │
│  │  [63:48 unused] [47 ─────────────────────── 0 valid]    │       │
│  │                  ↑                                        │       │
│  │                  Maximum 48-bit VA (all Armv8-A)         │       │
│  │                  (52-bit optional, Armv8.2-A+)           │       │
│  └──────────────────────────────────────────────────────────┘       │
└──────────────────────────────────────────────────────────────────────┘
```

### Physical Address Size

| Feature | Detail |
|---------|--------|
| Maximum PA size | Up to **52 bits** (since Armv8.2-A) |
| Typical Cortex-A | **40 bits** or **44 bits** |
| How to check | Read `ID_AA64MMFR0_EL1` register |

> ⚠️ If you specify an output address larger than the implemented maximum, the MMU generates an **Address Size Fault**.

---

## 6. Address Space Identifiers (ASID)

### The Problem

Multiple applications all use the **same virtual address range** (user space). When the OS switches between apps, how does the TLB know which translation to use for VA `0x8000`?

### The Solution: ASIDs

```
┌──────────────────────────────────────────────────────────────────────┐
│              HOW ASIDs WORK IN THE TLB                               │
│                                                                      │
│  TLB Entry Format:                                                   │
│  ┌──────────┬──────────┬──────────────┬────────────┐                │
│  │  VA Tag  │   ASID   │  PA Base     │ Attributes │                │
│  └──────────┴──────────┴──────────────┴────────────┘                │
│                                                                      │
│  Example TLB contents:                                               │
│  ┌──────────┬──────────┬──────────────┬────────────┐                │
│  │ 0x8000   │  0x01    │ 0x4000_0000  │ RW, Normal │  ← App 1      │
│  │ 0x8000   │  0x02    │ 0x5000_0000  │ RW, Normal │  ← App 2      │
│  │ 0xFFFF.. │   --     │ 0x8000_0000  │ RO, Normal │  ← Kernel     │
│  └──────────┴──────────┴──────────────┴────────────┘                │
│                                                                      │
│  When App 2 is running (ASID=0x02):                                  │
│  → VA 0x8000 matches ASID 0x02 → uses PA 0x5000_0000               │
│  → Kernel entry has no ASID (Global) → always matches               │
│                                                                      │
│  ✅ Both apps can coexist in TLB — no need to flush on switch!      │
└──────────────────────────────────────────────────────────────────────┘
```

### Global vs Non-Global Translations

| Type | `nG` bit | ASID Tagged? | Used For |
|------|----------|--------------|----------|
| **Global (G)** | `nG = 0` | ❌ No | Kernel mappings (same for all apps) |
| **Non-Global (nG)** | `nG = 1` | ✅ Yes | Application mappings (per-app) |

### Where is the ASID Stored?

The ASID is stored in the **TTBR (Translation Table Base Register)**:

```
┌──────────────────────────────────────────────────────────────────────┐
│  TTBR0_EL1 Register Layout:                                          │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ [63:48] ASID  │  [47:1] BADDR (Table Base Address)  │ [0] CnP│  │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  • ASID: Identifies which application this table belongs to         │
│  • BADDR: Physical address of the start of the translation table    │
│  • CnP: Common not Private bit (Armv8.2-A+)                        │
└──────────────────────────────────────────────────────────────────────┘
```

> 💡 **From Armv9.5-A:** Software can use the ASID field in **both** `TTBR0_EL1` and `TTBR1_EL1` simultaneously — `TTBR0_EL1.ASID` for lower VA space, `TTBR1_EL1.ASID` for upper VA space.

---

## 7. Virtual Machine Identifiers (VMID)

Similar to ASIDs but for **virtual machines**:

```
┌──────────────────────────────────────────────────────────────────────┐
│              VMID vs ASID                                            │
│                                                                      │
│  ASID  → Distinguishes translations between different APPLICATIONS  │
│  VMID  → Distinguishes translations between different VMs           │
│                                                                      │
│  TLB Entry with both:                                               │
│  ┌──────────┬──────────┬──────────┬──────────────┬────────────┐    │
│  │  VA Tag  │   VMID   │   ASID   │  PA Base     │ Attributes │    │
│  └──────────┴──────────┴──────────┴──────────────┴────────────┘    │
│                                                                      │
│  Both VMID AND ASID must match for a TLB entry to be used!         │
│                                                                      │
│  ⚠️  Even if Stage 2 is not enabled, EL0/EL1 translations are      │
│      always tagged with a VMID when virtualization is supported.    │
│      → Always set a known VMID before enabling Stage 1 MMU!        │
└──────────────────────────────────────────────────────────────────────┘
```

### Common not Private (CnP) — Armv8.2-A+

```
┌──────────────────────────────────────────────────────────────────────┐
│              COMMON NOT PRIVATE (CnP) BIT                            │
│                                                                      │
│  Problem: In a multi-core system, can Core 0's TLB entries be       │
│           used by Core 1?                                            │
│                                                                      │
│  Armv8.0-A: No requirement — ASID 5 on Core 0 might mean           │
│             something different than ASID 5 on Core 1.              │
│                                                                      │
│  Armv8.2-A CnP bit: When set, software PROMISES to use ASIDs       │
│  and VMIDs the same way on ALL cores.                               │
│  → TLB entries from one core CAN be used by another core.          │
│  → Better performance in multi-core systems!                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 8. The Memory Management Unit (MMU)

The MMU is the hardware block responsible for **translating virtual addresses to physical addresses**.

### MMU Internal Structure

```
┌──────────────────────────────────────────────────────────────────────┐
│                    THE MMU INTERNALS                                  │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                        ARM CORE                             │    │
│  │                                                             │    │
│  │  Software issues VA                                         │    │
│  │         │                                                   │    │
│  │         ▼                                                   │    │
│  │  ┌─────────────────────────────────────────────────────┐   │    │
│  │  │                      MMU                            │   │    │
│  │  │                                                     │   │    │
│  │  │  ┌──────────────┐      ┌──────────────────────┐    │   │    │
│  │  │  │    TLBs      │      │   Table Walk Unit    │    │   │    │
│  │  │  │  (Cache of   │      │  (Reads translation  │    │   │    │
│  │  │  │ translations)│      │   tables from memory)│    │   │    │
│  │  │  └──────┬───────┘      └──────────┬───────────┘    │   │    │
│  │  │         │ Hit?                     │ Miss → walk    │   │    │
│  │  │         └──────────────────────────┘                │   │    │
│  │  │                        │                            │   │    │
│  │  │                        ▼ PA                         │   │    │
│  │  └─────────────────────────────────────────────────────┘   │    │
│  │                           │                                 │    │
│  │                           ▼                                 │    │
│  │              ┌────────────────────────┐                    │    │
│  │              │  Caches / Memory       │                    │    │
│  │              └────────────────────────┘                    │    │
│  └─────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

### How a Translation Works (Step by Step)

```
Step 1: Software issues a Virtual Address (VA)
         │
         ▼
Step 2: MMU checks TLBs for a cached translation
         │
         ├── HIT  → Use cached PA directly ✅ (fast!)
         │
         └── MISS → Table Walk Unit reads translation tables from memory
                     │
                     ▼
Step 3: Table Walk Unit finds the correct entry
         │
         ▼
Step 4: PA is returned, TLB is updated with new translation
         │
         ▼
Step 5: Memory access proceeds using PA
```

> 💡 **Why must VA be translated before cache lookup?** Because Armv6+ processors use **physically-tagged caches** — the cache is indexed by PA, not VA. So translation must happen first.

---

## 9. Translation Table Entries

### How Tables Work

The virtual address space is divided into **equal-sized blocks**. Each block has one entry in the table:

```
┌──────────────────────────────────────────────────────────────────────┐
│              SINGLE-LEVEL TABLE LOOKUP                               │
│                                                                      │
│  Virtual Address issued by software:                                 │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  [Upper bits: "Which entry"]  │  [Lower bits: "Offset"]      │   │
│  └──────────────────────────────────────────────────────────────┘   │
│           │                                    │                     │
│           │ Index into table                   │ Unchanged           │
│           ▼                                    │                     │
│  ┌──────────────────────┐                      │                     │
│  │  Translation Table   │                      │                     │
│  │  ┌────────────────┐  │                      │                     │
│  │  │ Entry 0: PA+Attr│  │                      │                     │
│  │  │ Entry 1: PA+Attr│  │                      │                     │
│  │  │ Entry N: PA+Attr│◄─┘ (selected entry)    │                     │
│  │  └────────────────┘  │                      │                     │
│  └──────────┬───────────┘                      │                     │
│             │ PA Base                           │                     │
│             ▼                                   ▼                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Physical Address = PA Base + Offset                         │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 10. Multi-Level Translation Tables

In practice, a **hierarchy of tables** is used (not just one level). This allows both large and small memory blocks to be described efficiently.

### Why Multi-Level?

```
┌──────────────────────────────────────────────────────────────────────┐
│              MULTI-LEVEL TABLE HIERARCHY                             │
│                                                                      │
│  Level 0 Table (covers huge regions, e.g., 512 GB per entry)        │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Entry 0 → Points to Level 1 Table                          │   │
│  │  Entry 1 → Points to Level 1 Table                          │   │
│  │  Entry 2 → Points to a BLOCK (large physical region)        │   │
│  └──────────────────────────────────────────────────────────────┘   │
│           │                                                          │
│           ▼                                                          │
│  Level 1 Table (covers 1 GB per entry with 4KB granule)             │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Entry 0 → Points to Level 2 Table                          │   │
│  │  Entry 1 → Points to a BLOCK (1 GB physical region)         │   │
│  └──────────────────────────────────────────────────────────────┘   │
│           │                                                          │
│           ▼                                                          │
│  Level 2 Table (covers 2 MB per entry with 4KB granule)             │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Entry 0 → Points to Level 3 Table                          │   │
│  │  Entry 1 → Points to a BLOCK (2 MB physical region)         │   │
│  └──────────────────────────────────────────────────────────────┘   │
│           │                                                          │
│           ▼                                                          │
│  Level 3 Table (covers 4 KB per entry — the smallest page)          │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Entry 0 → PAGE (4 KB physical region)                      │   │
│  │  Entry 1 → PAGE (4 KB physical region)                      │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ARMv8-A: Maximum 4 levels (Level 0 to Level 3)                     │
└──────────────────────────────────────────────────────────────────────┘
```

### Large vs Small Blocks Trade-off

| Block Size | Levels Needed | TLB Efficiency | Flexibility |
|------------|---------------|----------------|-------------|
| **Large** (e.g., 1 GB) | Fewer | ✅ More efficient | ❌ Less flexible |
| **Small** (e.g., 4 KB) | More | ❌ Less efficient | ✅ Fine-grained control |

---

## 11. Translation Table Formats

Each table entry is **64 bits wide**. The bottom 2 bits determine the **type of entry**:

```
┌──────────────────────────────────────────────────────────────────────┐
│              TRANSLATION TABLE ENTRY FORMATS                         │
│                                                                      │
│  Bits [1:0]  │  Entry Type          │  Valid At Levels              │
│  ────────────┼──────────────────────┼───────────────────────────    │
│  0b00 / 0b10 │  FAULT (Invalid)     │  All levels                   │
│  0b01        │  BLOCK Descriptor    │  Levels 1, 2                  │
│  0b11        │  TABLE Descriptor    │  Levels 0, 1, 2               │
│  0b11        │  PAGE Descriptor     │  Level 3 only                 │
│                                                                      │
│  TABLE Descriptor (Levels 0, 1, 2):                                 │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ [63] Upper Attrs │ [47:12] Next Table Address │ [11:2] Attrs │   │
│  │                                               │ [1:0] = 0b11 │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  BLOCK Descriptor (Levels 1, 2):                                    │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ [63] Upper Attrs │ [47:n] Block Address │ [n-1:2] Lower Attrs│   │
│  │                                         │ [1:0] = 0b01       │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  PAGE Descriptor (Level 3):                                         │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ [63] Upper Attrs │ [47:12] Page Address │ [11:2] Lower Attrs │   │
│  │                                         │ [1:0] = 0b11       │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

> 💡 **Why no Block descriptor at Level 0?** Level 0 entries cover enormous regions (512 GB with 4KB granule). It doesn't make sense to map a 512 GB block directly.

---

## 12. Translation Granule

The **translation granule** is the **smallest block of memory** that can be described. Everything is a multiple of this size.

### Supported Granule Sizes

| Granule | Supported By |
|---------|-------------|
| **4 KB** | All Arm Cortex-A processors |
| **16 KB** | Implementation defined |
| **64 KB** | All Arm Cortex-A processors |

### Block Sizes Per Level (4KB Granule)

```
┌──────────────────────────────────────────────────────────────────────┐
│              BLOCK SIZES WITH 4KB GRANULE                            │
│                                                                      │
│  Level │  Block Size  │  VA Bits Used  │  Notes                     │
│  ──────┼──────────────┼────────────────┼──────────────────────────  │
│    0   │   512 GB     │   [47:39]      │  Table only, no blocks     │
│    1   │    1 GB      │   [38:30]      │  Block or Table            │
│    2   │    2 MB      │   [29:21]      │  Block or Table            │
│    3   │    4 KB      │   [20:12]      │  Page only (smallest)      │
│                                                                      │
│  Virtual Address Bit Layout (4KB granule, 48-bit VA):               │
│  ┌────────┬────────┬────────┬────────┬────────────────────────┐    │
│  │[47:39] │[38:30] │[29:21] │[20:12] │      [11:0]            │    │
│  │ L0 idx │ L1 idx │ L2 idx │ L3 idx │  Offset within page    │    │
│  └────────┴────────┴────────┴────────┴────────────────────────┘    │
│    9 bits   9 bits   9 bits   9 bits        12 bits                  │
└──────────────────────────────────────────────────────────────────────┘
```

### Starting Level of Translation

The starting level depends on the **virtual address size** (`TnSZ`):

```
Example with 4KB granule:

  T0SZ = 32  →  64 - 32 = 32-bit VA  →  Start at Level 1
  T0SZ = 34  →  64 - 34 = 30-bit VA  →  Start at Level 2
  T0SZ = 25  →  64 - 25 = 39-bit VA  →  Start at Level 0
```

---

## 13. Key Registers That Control Address Translation

### `SCTLR_ELx` — System Control Register

```
┌──────────────────────────────────────────────────────────────────────┐
│  SCTLR_ELx (System Control Register)                                 │
│                                                                      │
│  Bit  │  Name  │  Function                                          │
│  ─────┼────────┼──────────────────────────────────────────────────  │
│   0   │   M    │  MMU Enable: 0=disabled, 1=enabled                 │
│   2   │   C    │  Data/Unified Cache Enable                         │
│  25   │   EE   │  Endianness of translation table walks             │
│                                                                      │
│  When M=0 (MMU disabled):                                           │
│  → All addresses are flat-mapped (VA = PA)                          │
│  → All data accesses treated as Device-nGnRnE                       │
└──────────────────────────────────────────────────────────────────────┘
```

### `TTBR0_ELx` / `TTBR1_ELx` — Translation Table Base Registers

```
┌──────────────────────────────────────────────────────────────────────┐
│  TTBR0_EL1 / TTBR1_EL1 (Translation Table Base Registers)           │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ [63:48] ASID  │  [47:1] BADDR (Table Base Address)  │ [0] CnP│  │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Field  │  Function                                                  │
│  ───────┼────────────────────────────────────────────────────────   │
│  BADDR  │  Physical address of the start of the translation table   │
│  ASID   │  Address Space Identifier for Non-Global translations     │
│  CnP    │  Common not Private (Armv8.2-A+)                         │
│                                                                      │
│  TTBR0_EL1 → Points to User Space (lower VA) translation table     │
│  TTBR1_EL1 → Points to Kernel Space (upper VA) translation table   │
└──────────────────────────────────────────────────────────────────────┘
```

### `TCR_ELx` — Translation Control Register

```
┌──────────────────────────────────────────────────────────────────────┐
│  TCR_ELx (Translation Control Register)                              │
│                                                                      │
│  Field   │  Function                                                 │
│  ────────┼─────────────────────────────────────────────────────────  │
│  T0SZ    │  Size of lower VA space (user space)                     │
│  T1SZ    │  Size of upper VA space (kernel space)                   │
│  TG0     │  Granule size for TTBR0 (user space)                    │
│  TG1     │  Granule size for TTBR1 (kernel space)                  │
│  PS/IPS  │  Size of PA or IPA output address space                  │
│  SH0/SH1 │  Shareability for table walks                            │
│  IRGN0/1 │  Inner cacheability for table walks                      │
│  ORGN0/1 │  Outer cacheability for table walks                      │
│  TBI0/1  │  Top Byte Ignore (for tagged pointers)                   │
│  HA      │  Hardware Access flag update (Armv8.1-A+)               │
│  HD      │  Hardware Dirty state management (Armv8.1-A+)           │
│                                                                      │
│  ⚠️  TG0 and TG1 have DIFFERENT encodings — be careful!            │
└──────────────────────────────────────────────────────────────────────┘
```

### `MAIR_ELx` — Memory Attribute Indirection Register

```
┌──────────────────────────────────────────────────────────────────────┐
│  MAIR_ELx (Memory Attribute Indirection Register)                    │
│                                                                      │
│  Contains 8 attribute slots (Attr0 to Attr7), each 8 bits wide:     │
│                                                                      │
│  ┌────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┐ │
│  │ Attr7  │ Attr6  │ Attr5  │ Attr4  │ Attr3  │ Attr2  │ Attr1  │ Attr0  │ │
│  │ [63:56]│ [55:48]│ [47:40]│ [39:32]│ [31:24]│ [23:16]│ [15:8] │ [7:0]  │ │
│  └────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┘ │
│                                                                      │
│  Each AttrN encodes: Memory Type + Cacheability                     │
│                                                                      │
│  Translation table entries use AttrIndx[2:0] to SELECT which        │
│  of the 8 slots to use for that page/block.                         │
│                                                                      │
│  Why indirect? Saves bits! 3 bits (index) vs 8 bits (full attr)    │
└──────────────────────────────────────────────────────────────────────┘
```

### `VTCR_EL2` — Virtualization Translation Control Register

| Field | Function |
|-------|----------|
| `T0SZ` | Size of IPA space for Stage 2 |
| `SL0` | Starting level for Stage 2 table walk |
| `TG0` | Granule size for Stage 2 |

---

## 14. Translation Lookaside Buffer (TLB) Maintenance

### What is a TLB?

The TLB is a **cache of recently used translations**. It stores VA→PA mappings so the MMU doesn't have to re-read the translation tables every time.

```
┌──────────────────────────────────────────────────────────────────────┐
│              TLB vs TRANSLATION TABLES                               │
│                                                                      │
│  Translation Tables:  Stored in RAM, managed by software (OS)       │
│  TLB:                 Hardware cache of translations, managed by MMU │
│                                                                      │
│  Important: TLB stores the INTERPRETATION of table entries          │
│  (given the register configuration at walk time), NOT the raw       │
│  table entries themselves.                                           │
└──────────────────────────────────────────────────────────────────────┘
```

### When Must You Invalidate the TLB?

```
┌──────────────────────────────────────────────────────────────────────┐
│              TLB INVALIDATION RULES                                  │
│                                                                      │
│  ✅ NO TLB invalidate needed when:                                  │
│     • Mapping an address for the FIRST TIME                         │
│       (faulting entries cannot be cached in TLB)                    │
│                                                                      │
│  ❌ TLB invalidate IS needed when:                                  │
│     • UNMAPPING an address (marking it as faulting)                 │
│     • CHANGING the output PA of a mapping                           │
│     • CHANGING attributes (e.g., read-only → read-write)           │
│     • CHANGING how tables are interpreted (e.g., granule size)      │
└──────────────────────────────────────────────────────────────────────┘
```

### The `TLBI` Instruction

```
Syntax:  TLBI <type><level>{IS|OS} {, <Xt>}

Examples:
  TLBI ALLE1IS        → Invalidate ALL EL0/1 entries, Inner Shareable
  TLBI VAAE1IS, X0   → Invalidate entry matching VA in X0, any ASID
  TLBI ALLE3          → Invalidate ALL EL3 entries (local only)
  TLBI ASIDE1IS, X0  → Invalidate all entries with ASID in X0
```

| Parameter | Options | Meaning |
|-----------|---------|---------|
| `<type>` | `ALL`, `VA`, `VAA`, `ASID`, ... | Which entries to invalidate |
| `<level>` | `E1`, `E2`, `E3` | Which address space |
| `IS` / `OS` | Inner/Outer Shareable | Broadcast to other cores |
| `<Xt>` | Register | VA or ASID to target |

### Typical TLB Invalidation Sequence

```asm
STR  X1, [X5]        // 1. Write new value to translation table entry
DSB  ISH             // 2. Ensure the write is visible to all observers
TLBI VAAE1IS, X0     // 3. Invalidate the TLB entry for VA in X0
DSB  ISH             // 4. Ensure TLB invalidation is complete
ISB                  // 5. Synchronize instruction stream
```

---

## 15. Address Translation (AT) Instructions

The `AT` instruction lets software **query the translation** for a specific address without actually accessing memory:

```
Syntax:  AT <operation>, <Xn>

Examples:
  AT S1E1R, X0    → Translate X0 using Stage 1 EL1 read permissions
  AT S12E0R, X0   → Translate X0 using Stage 1+2 EL0 read permissions

Result is written to: PAR_EL1 (Physical Address Register)
```

```
┌──────────────────────────────────────────────────────────────────────┐
│  PAR_EL1 (Physical Address Register)                                 │
│                                                                      │
│  On SUCCESS:                                                         │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ [47:12] PA  │  [11:0] Attributes  │  [0] F=0 (no fault)     │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  On FAULT:                                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Fault type encoded in PAR_EL1  │  [0] F=1 (fault occurred) │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ⚠️  EL1 CANNOT use AT to query EL2 translations (privilege breach) │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 16. Quick Reference: Glossary

| Term | Full Name | Meaning |
|------|-----------|---------|
| **VA** | Virtual Address | Address used by software |
| **PA** | Physical Address | Real hardware address |
| **IPA** | Intermediate Physical Address | Output of Stage 1 in virtualization |
| **MMU** | Memory Management Unit | Hardware that translates VA→PA |
| **TLB** | Translation Lookaside Buffer | Cache of recent translations |
| **ASID** | Address Space Identifier | Tags TLB entries per application |
| **VMID** | Virtual Machine Identifier | Tags TLB entries per VM |
| **EL** | Exception Level | Privilege level (EL0–EL3) |
| **PAS** | Physical Address Space | Secure/Non-secure/Realm/Root |
| **PE** | Processing Element | Generic term for a CPU core/thread |
| **CnP** | Common not Private | Allows TLB sharing across cores |
| **TTBR** | Translation Table Base Register | Points to start of page tables |
| **TCR** | Translation Control Register | Controls VA size, granule, etc. |
| **MAIR** | Memory Attribute Indirection Register | Stores memory type attributes |
| **SCTLR** | System Control Register | Enables/disables MMU, caches |
| **TLBI** | TLB Invalidate instruction | Invalidates TLB entries |
| **AT** | Address Translation instruction | Queries a translation |
| **PAR_EL1** | Physical Address Register | Stores result of AT instruction |

---

## 17. Knowledge Check Q&A

**Q: What is the difference between a stage and a level in address translation?**
> A **stage** is the full process of translating one address type to another (VA→IPA is Stage 1; IPA→PA is Stage 2). A **level** refers to the depth within a single stage's table hierarchy (Level 0, 1, 2, 3).

**Q: What is the maximum size of a physical address?**
> Up to **52 bits** (since Armv8.2-A). Implementation defined — check `ID_AA64MMFR0_EL1`.

**Q: Which register field controls the size of the virtual address space?**
> `TCR_ELx.TnSZ` (or `VTCR_EL2.T0SZ` for Stage 2).

**Q: What is a translation granule, and what are the supported sizes?**
> The smallest block of memory that can be described. Supported sizes: **4 KB, 16 KB, 64 KB**.

**Q: What does `TLBI ALLE3` do?**
> Invalidates **all TLB entries** for the EL3 virtual address space (local core only).

**Q: Can a translation table entry that causes a Translation Fault be cached in the TLBs?**
> **No.** Faulting entries cannot be cached. So you don't need a TLBI when mapping an address for the first time.

**Q: How are addresses mapped when the MMU is disabled?**
> **Flat-mapped** — the input address equals the output address (VA = PA).

**Q: What is an ASID and when does a TLB entry include an ASID?**
> An ASID (Address Space Identifier) identifies which application a translation belongs to. TLB entries include an ASID when the mapping is **Non-Global** (`nG=1`).

---

*📖 Next: Read [README_2_Memory_Attributes_and_Properties.md](README_2_Memory_Attributes_and_Properties.md) to learn about memory types, permissions, and attributes.*
