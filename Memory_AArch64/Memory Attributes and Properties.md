# 📗 AArch64 Memory Attributes and Properties
### *A Beginner-Friendly Guide to Memory Types, Permissions, and Attributes in ARMv8-A*

> **Source:** Learn the Architecture – AArch64 Memory Attributes and Properties (102376_0200_01_en, Version 2.0)  
> **Architecture:** ARMv8-A / ARMv9-A | **State:** AArch64

---

## 📚 Table of Contents

1. [What Are Memory Attributes and Why Are They Needed?](#1-what-are-memory-attributes-and-why-are-they-needed)
2. [How Attributes Are Assigned in AArch64](#2-how-attributes-are-assigned-in-aarch64)
3. [Hierarchical Attributes](#3-hierarchical-attributes)
4. [Memory Types Overview](#4-memory-types-overview)
5. [Normal Memory](#5-normal-memory)
6. [Device Memory](#6-device-memory)
7. [Device Memory Sub-Types](#7-device-memory-sub-types)
8. [Describing the Memory Type — MAIR_ELx](#8-describing-the-memory-type--mair_elx)
9. [Cacheability and Shareability Attributes](#9-cacheability-and-shareability-attributes)
10. [Access Permissions (AP bits)](#10-access-permissions-ap-bits)
11. [Privileged Access Never (PAN)](#11-privileged-access-never-pan)
12. [Execution Permissions (UXN / PXN)](#12-execution-permissions-uxn--pxn)
13. [Permission Indirection (Armv8.9-A+)](#13-permission-indirection-armv89-a)
14. [Permission Overlays (Armv8.9-A+)](#14-permission-overlays-armv89-a)
15. [Access Flag (AF bit)](#15-access-flag-af-bit)
16. [Dirty State (DBM bit)](#16-dirty-state-dbm-bit)
17. [Alignment](#17-alignment)
18. [Endianness](#18-endianness)
19. [Memory Aliasing and Mismatched Types](#19-memory-aliasing-and-mismatched-types)
20. [Quick Reference: All Attributes at a Glance](#20-quick-reference-all-attributes-at-a-glance)
21. [Knowledge Check Q&A](#21-knowledge-check-qa)

---

## 1. What Are Memory Attributes and Why Are They Needed?

Memory attributes are a set of **rules and properties** that define how the processor should interact with a specific region of memory. Different parts of the address space need to be treated differently:

```
┌──────────────────────────────────────────────────────────────────────┐
│              EXAMPLE ADDRESS MAP WITH ATTRIBUTES                     │
│                                                                      │
│  Address Space:                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  0xFFFF_FFFF  ┐                                              │   │
│  │  Kernel Code  │  → Normal, Cacheable, Executable (EL1 only) │   │
│  │  Kernel Data  │  → Normal, Cacheable, Read/Write            │   │
│  │  0xC000_0000  ┘                                              │   │
│  │  ─────────────────────────────────────────────────────────   │   │
│  │  0xBFFF_FFFF  ┐                                              │   │
│  │  Peripherals  │  → Device, Non-cacheable, Non-executable     │   │
│  │  (UART, GPIO) │    (side-effects on access!)                 │   │
│  │  0xA000_0000  ┘                                              │   │
│  │  ─────────────────────────────────────────────────────────   │   │
│  │  0x9FFF_FFFF  ┐                                              │   │
│  │  App Code     │  → Normal, Cacheable, Executable (EL0)      │   │
│  │  App Data     │  → Normal, Cacheable, Read/Write            │   │
│  │  0x0000_0000  ┘                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Without attributes, the processor can't know:                      │
│  • Can this region be cached?                                        │
│  • Can instructions be executed from here?                           │
│  • Can user apps write to this region?                               │
│  • Are there side-effects when reading this address?                 │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 2. How Attributes Are Assigned in AArch64

Attributes come from the **translation table entries** (also called page table entries or descriptors). Each entry describes a block or page of virtual addresses and contains:

```
┌──────────────────────────────────────────────────────────────────────┐
│              BLOCK/PAGE DESCRIPTOR ATTRIBUTE FIELDS                  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Bit 63                                              Bit 0   │   │
│  │  ┌──────┬─────┬─────┬────┬────┬──────────┬──────────┬────┐  │   │
│  │  │ UXN  │ PXN │ DBM │ nG │ AF │  AP[2:1] │ AttrIndx │ ...│  │   │
│  │  └──────┴─────┴─────┴────┴────┴──────────┴──────────┴────┘  │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Key Attribute Fields:                                               │
│  ┌──────────┬──────────────────────────────────────────────────┐    │
│  │  Field   │  Meaning                                         │    │
│  ├──────────┼──────────────────────────────────────────────────┤    │
│  │ AttrIndx │ Index into MAIR_ELx (selects memory type)        │    │
│  │ AP[2:1]  │ Access Permissions (read/write, privilege)       │    │
│  │ UXN      │ User Execute Never (EL0 cannot execute)          │    │
│  │ PXN      │ Privileged Execute Never (EL1/2 cannot execute)  │    │
│  │ AF       │ Access Flag (has this page been accessed?)       │    │
│  │ DBM      │ Dirty Bit Modifier (hardware dirty tracking)     │    │
│  │ nG       │ Non-Global (tagged with ASID in TLB)             │    │
│  └──────────┴──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 3. Hierarchical Attributes

Some attributes can be set in **higher-level table descriptors** (not just the final page/block descriptor). These are called **hierarchical attributes** and they **override** lower-level entries.

Hierarchical attributes apply to:
- **Access Permissions** (AP)
- **Execution Permissions** (PXN, UXN)
- **Physical Address Space** (NS bit)

```
┌──────────────────────────────────────────────────────────────────────┐
│              HIERARCHICAL ATTRIBUTE OVERRIDE                         │
│                                                                      │
│  Level 1 Table Descriptor:                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  PXNTable = 1  (set at Level 1)                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│           │                                                          │
│           ▼  Overrides all lower levels!                             │
│  Level 2 Table Descriptor:                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  PXN = 0  (set at Level 2, but OVERRIDDEN by Level 1)        │   │
│  └──────────────────────────────────────────────────────────────┘   │
│           │                                                          │
│           ▼                                                          │
│  Level 3 Page Descriptor:                                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  PXN = 0  (set at Level 3, but OVERRIDDEN by Level 1)        │   │
│  │  → Effective PXN = 1 (privileged execute never)              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Rule: If hierarchical bit is SET → overrides lower levels          │
│        If hierarchical bit is CLEAR → lower levels used as-is       │
│                                                                      │
│  ⚠️  From Armv8.1-A: HPD (Hierarchical Permission Disable) bits    │
│      in TCR_ELx can DISABLE hierarchical attributes entirely.       │
│      When disabled, those bits are free for software use.           │
└──────────────────────────────────────────────────────────────────────┘
```

### When the MMU is Disabled

| Condition | Behavior |
|-----------|----------|
| Stage 1 MMU disabled | All data accesses → **Device-nGnRnE** |
| Stage 1 MMU disabled | Instruction fetches → depends on `SCTLR_ELx.I` |
| Stage 1 MMU disabled | All addresses → **read/write/executable** |
| Stage 2 disabled | Stage 1 attributes used **unmodified** |

---

## 4. Memory Types Overview

Every address in the system is assigned one of **two memory types**:

```
┌──────────────────────────────────────────────────────────────────────┐
│              TWO MEMORY TYPES IN ARMv8-A                             │
│                                                                      │
│  ┌─────────────────────────────┐  ┌─────────────────────────────┐  │
│  │       NORMAL MEMORY         │  │       DEVICE MEMORY         │  │
│  │                             │  │                             │  │
│  │  Used for: RAM, Flash, ROM  │  │  Used for: Peripherals,     │  │
│  │  Code should only be in     │  │  Memory-Mapped I/O (MMIO)   │  │
│  │  Normal regions.            │  │                             │  │
│  │                             │  │  Has SIDE-EFFECTS on access │  │
│  │  No side-effects on access  │  │  (e.g., reading a FIFO      │  │
│  │  → Processor may:           │  │   advances it!)             │  │
│  │    • Merge accesses         │  │                             │  │
│  │    • Speculate accesses     │  │  → Processor MUST:          │  │
│  │    • Reorder accesses       │  │    • Respect exact count    │  │
│  │    • Cache data             │  │    • Respect exact order    │  │
│  │                             │  │    • Never cache            │  │
│  └─────────────────────────────┘  └─────────────────────────────┘  │
│                                                                      │
│  Note: Armv6/v7 had a third type "Strongly Ordered"                 │
│        In Armv8, this maps to Device-nGnRnE                         │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 5. Normal Memory

Normal memory is used for **anything that behaves like memory**: RAM, Flash, ROM. This is the most common type in a system.

### Key Properties of Normal Memory

```
┌──────────────────────────────────────────────────────────────────────┐
│              NORMAL MEMORY PROPERTIES                                │
│                                                                      │
│  ✅ Can be CACHED                                                   │
│  ✅ Processor may REORDER accesses (for performance)               │
│  ✅ Processor may MERGE accesses                                    │
│  ✅ Processor may SPECULATIVELY access (prefetch)                  │
│  ✅ Code CAN be placed here                                         │
│                                                                      │
│  Why can the processor reorder Normal memory accesses?              │
│  Because reading a Normal location has NO SIDE-EFFECTS.             │
│  Reading address 0x1000 just returns data — it doesn't change       │
│  anything or trigger any process.                                    │
│                                                                      │
│  Example of what the processor might do:                            │
│  Program order:    STR [0x1000]   ← write to memory                │
│                    LDR [0x2000]   ← read (cache miss)               │
│                    LDR [0x3000]   ← read (cache hit)                │
│                                                                      │
│  Actual execution: LDR [0x3000]   ← completes first (cache hit)    │
│                    LDR [0x2000]   ← completes second (cache fill)   │
│                    STR [0x1000]   ← completes last (write buffer)   │
│                                                                      │
│  This is ALLOWED for Normal memory!                                 │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 6. Device Memory

Device memory is used for **peripherals** (UART, GPIO, timers, etc.) — anything accessed via Memory-Mapped I/O (MMIO).

### Key Properties of Device Memory

```
┌──────────────────────────────────────────────────────────────────────┐
│              DEVICE MEMORY PROPERTIES                                │
│                                                                      │
│  ❌ NEVER cached                                                    │
│  ❌ NO speculative DATA accesses                                    │
│  ❌ Processor MUST respect exact access count                       │
│  ❌ Processor MUST respect exact access order                       │
│  ⚠️  Code should NOT be placed here                                 │
│                                                                      │
│  Why no speculative access?                                          │
│  Reading a FIFO register ADVANCES the FIFO pointer.                 │
│  If the processor speculatively reads it, the FIFO advances         │
│  even though the program didn't intend to read it yet!              │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  FIFO Register (Device Memory)                               │   │
│  │                                                              │   │
│  │  Before read:  [A] [B] [C] [D]  ← A is at front            │   │
│  │  After read:   [B] [C] [D]      ← A consumed, B at front    │   │
│  │                                                              │   │
│  │  If processor speculatively reads this, A is LOST!          │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ⚠️  IMPORTANT DISTINCTION:                                         │
│  • Device type → prevents speculative DATA accesses                 │
│  • Execute Never (XN) → prevents speculative INSTRUCTION accesses   │
│  • To prevent ALL speculative accesses: mark as BOTH Device AND XN  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 7. Device Memory Sub-Types

Device memory has **four sub-types** with varying levels of restrictions. The letters after "Device" describe what is allowed:

```
┌──────────────────────────────────────────────────────────────────────┐
│              DEVICE MEMORY SUB-TYPES EXPLAINED                       │
│                                                                      │
│  Letter  │  Meaning                                                  │
│  ────────┼──────────────────────────────────────────────────────    │
│  G       │  Gathering ALLOWED (merge multiple accesses into one)    │
│  nG      │  No Gathering (each access must be separate)             │
│  R       │  Re-ordering ALLOWED (accesses to same peripheral)       │
│  nR      │  No Re-ordering (strict order must be maintained)        │
│  E       │  Early Write Acknowledgement ALLOWED                     │
│  nE      │  No Early Write Acknowledgement (must reach destination) │
└──────────────────────────────────────────────────────────────────────┘
```

### The Four Sub-Types

```
┌──────────────────────────────────────────────────────────────────────┐
│              DEVICE SUB-TYPE COMPARISON                              │
│                                                                      │
│  Most Permissive ──────────────────────────── Most Restrictive      │
│                                                                      │
│  Device-GRE    Device-nGRE    Device-nGnRE    Device-nGnRnE         │
│      │               │              │               │               │
│  Gathering ✅    Gathering ❌   Gathering ❌   Gathering ❌          │
│  Reorder   ✅    Reorder   ✅   Reorder   ❌   Reorder   ❌          │
│  Early Ack ✅    Early Ack ✅   Early Ack ✅   Early Ack ❌          │
│                                                                      │
│  Allowed behaviors (subset relationship):                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Device-GRE  ⊇  Device-nGRE  ⊇  Device-nGnRE  ⊇  nGnRnE   │   │
│  │  (all behaviors of nGnRnE are also allowed in GRE)           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  This means: A processor is ALLOWED to treat Device-GRE as         │
│  Device-nGnRnE (more restrictive is always safe).                   │
└──────────────────────────────────────────────────────────────────────┘
```

### Detailed Explanation of Each Property

#### Gathering (G / nG)
```
G  (Gathering allowed):
   Multiple accesses to the same/nearby addresses can be MERGED
   into a single bus transaction.
   Example: Two 32-bit writes to adjacent addresses → one 64-bit write

nG (No Gathering):
   Every access must be a separate bus transaction.
   Use for registers where the number of accesses matters.
```

#### Re-ordering (R / nR)
```
R  (Re-ordering allowed):
   Accesses to the same peripheral can be reordered for performance.
   Similar to Normal memory ordering rules.

nR (No Re-ordering):
   Accesses must reach the peripheral in program order.
   Use for registers where order matters (e.g., command sequences).
```

#### Early Write Acknowledgement (E / nE)
```
E  (Early Acknowledgement allowed):
   A write is considered "complete" once it's visible to other
   observers (e.g., reaches a write buffer in the interconnect),
   even if the peripheral hasn't received it yet.

nE (No Early Acknowledgement):
   A write is only "complete" when the DESTINATION peripheral
   sends back a write response.
   Use for critical registers where you must confirm the write arrived.
```

### Common Use Cases

| Sub-Type | Typical Use Case |
|----------|-----------------|
| `Device-nGnRnE` | PCIe config space, critical control registers |
| `Device-nGnRE` | Most MMIO peripherals (UART, GPIO, timers) |
| `Device-nGRE` | Peripherals where order within device doesn't matter |
| `Device-GRE` | Closest to Normal Non-cacheable; rarely used |

> ⚠️ **Important:** `Normal Non-cacheable` and `Device-GRE` look similar but are NOT the same! Normal Non-cacheable still allows **speculative data accesses**; Device-GRE does **not**.

---

## 8. Describing the Memory Type — MAIR_ELx

The memory type is **not directly encoded** in the translation table entry. Instead, a 3-bit index (`AttrIndx`) points to one of 8 slots in the `MAIR_ELx` register:

```
┌──────────────────────────────────────────────────────────────────────┐
│              MAIR_ELx INDIRECTION MECHANISM                          │
│                                                                      │
│  Translation Table Entry:                                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  ...  │  AttrIndx[2:0] = 0b011  │  ...                       │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                    │                                                  │
│                    │ Index = 3                                        │
│                    ▼                                                  │
│  MAIR_EL1 Register (64 bits = 8 slots × 8 bits each):               │
│  ┌────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┐ │
│  │ Attr7  │ Attr6  │ Attr5  │ Attr4  │ Attr3  │ Attr2  │ Attr1  │ Attr0  │ │
│  │        │        │        │        │   ◄────│────────│────────│────────│ │
│  └────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┘ │
│                                          ↑                                  │
│                                    Selected slot (index 3)                  │
│                                    Contains: memory type + cacheability     │
│                                                                      │
│  Why use indirection?                                                │
│  • Direct encoding needs 8 bits per table entry                     │
│  • Index only needs 3 bits per table entry                          │
│  • Saves 5 bits per entry × millions of entries = significant!      │
└──────────────────────────────────────────────────────────────────────┘
```

### Example MAIR_EL1 Configuration

```
Typical OS setup:
  Attr0 = 0xFF  → Normal, Write-Back, Read-Allocate, Write-Allocate (DRAM)
  Attr1 = 0x04  → Device-nGnRE (most peripherals)
  Attr2 = 0x00  → Device-nGnRnE (critical peripherals)
  Attr3 = 0x44  → Normal, Non-cacheable
  Attr4–7 = available for other uses
```

---

## 9. Cacheability and Shareability Attributes

For **Normal memory**, two additional attributes control caching behavior:

### Cacheability

```
┌──────────────────────────────────────────────────────────────────────┐
│              CACHEABILITY ATTRIBUTES                                  │
│                                                                      │
│  Non-cacheable (NC):                                                 │
│  → Data is NEVER stored in cache                                     │
│  → Every access goes directly to main memory                        │
│  → Slower, but always sees latest data                              │
│                                                                      │
│  Write-Through (WT):                                                 │
│  → Reads may be cached                                               │
│  → Writes go to BOTH cache AND main memory simultaneously           │
│  → Cache always matches memory                                       │
│                                                                      │
│  Write-Back (WB):                                                    │
│  → Reads AND writes go to cache                                      │
│  → Writes to memory happen LATER (when cache line is evicted)       │
│  → Fastest, but cache may differ from memory temporarily            │
│                                                                      │
│  ARM Recommendation: Mark DRAM as Normal Write-Back cacheable       │
└──────────────────────────────────────────────────────────────────────┘
```

### Shareability

```
┌──────────────────────────────────────────────────────────────────────┐
│              SHAREABILITY DOMAINS                                     │
│                                                                      │
│  Non-shareable:                                                      │
│  → Only the local core needs to see a coherent copy                 │
│  → No cache coherency with other cores                              │
│                                                                      │
│  Inner Shareable (IS):                                               │
│  → All cores in the INNER SHAREABLE DOMAIN see coherent data        │
│  → Typically: all cores in the same cluster/chip                    │
│  → Most common for OS-managed DRAM                                  │
│                                                                      │
│  Outer Shareable (OS):                                               │
│  → All cores in the OUTER SHAREABLE DOMAIN see coherent data        │
│  → Typically: multiple chips/sockets                                │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    System                                    │   │
│  │  ┌──────────────────────────────────────────────────────┐   │   │
│  │  │              Outer Shareable Domain                  │   │   │
│  │  │  ┌────────────────────┐  ┌────────────────────────┐  │   │   │
│  │  │  │  Inner Shareable   │  │  Inner Shareable       │  │   │   │
│  │  │  │  ┌────┐  ┌────┐   │  │  ┌────┐  ┌────┐       │  │   │   │
│  │  │  │  │Core│  │Core│   │  │  │Core│  │Core│       │  │   │   │
│  │  │  │  └────┘  └────┘   │  │  └────┘  └────┘       │  │   │   │
│  │  │  └────────────────────┘  └────────────────────────┘  │   │   │
│  │  └──────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ARM Recommendation: Mark DRAM as Normal Write-Back, Inner Shareable│
└──────────────────────────────────────────────────────────────────────┘
```

---

## 10. Access Permissions (AP bits)

The `AP[2:1]` field in the translation table entry controls **who can read/write** a memory region:

```
┌──────────────────────────────────────────────────────────────────────┐
│              ACCESS PERMISSION (AP) BIT TABLE                        │
│                                                                      │
│  AP[2:1]  │  EL0 (User/App)  │  EL1/2/3 (Kernel/Hypervisor/FW)    │
│  ─────────┼──────────────────┼──────────────────────────────────    │
│    00     │   No access      │   Read / Write                       │
│    01     │   Read / Write   │   Read / Write                       │
│    10     │   No access      │   Read only                          │
│    11     │   Read only      │   Read only                          │
│                                                                      │
│  Visual Summary:                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  AP=00: Kernel RW, User NO ACCESS  (kernel-only data)        │   │
│  │  AP=01: Kernel RW, User RW         (shared data)             │   │
│  │  AP=10: Kernel RO, User NO ACCESS  (kernel-only read-only)   │   │
│  │  AP=11: Kernel RO, User RO         (shared read-only)        │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  If an access violates permissions → PERMISSION FAULT exception     │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 11. Privileged Access Never (PAN)

### The Problem: Confused Deputy Attack

A malicious app might trick the OS into reading/writing data on the app's behalf that the app shouldn't be able to access directly. This is called a **confused deputy attack**.

### The Solution: PAN (Armv8.1-A+)

```
┌──────────────────────────────────────────────────────────────────────┐
│              PSTATE.PAN — PRIVILEGED ACCESS NEVER                    │
│                                                                      │
│  When PSTATE.PAN = 1:                                                │
│  → EL1 (or EL2 when E2H=1) CANNOT access unprivileged regions      │
│  → Any such access generates a PERMISSION FAULT                     │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  EL1 (OS Kernel)                                             │   │
│  │                                                              │   │
│  │  LDR X0, [user_buffer]  ← BLOCKED by PAN! ❌               │   │
│  │  (generates Permission Fault)                                │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  But sometimes the OS NEEDS to access user data (e.g., to write     │
│  to a buffer owned by an app). Solution: LDTR / STTR instructions   │
│                                                                      │
│  LDTR / STTR = Unprivileged Load/Store:                             │
│  → Checked against EL0 permissions even when run at EL1/EL2        │
│  → NOT blocked by PAN (because they're explicitly unprivileged)     │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  EL1 (OS Kernel)                                             │   │
│  │                                                              │   │
│  │  LDTR X0, [user_buffer]  ← ALLOWED! ✅                      │   │
│  │  (explicitly unprivileged, not blocked by PAN)               │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Historical note: "T" in LDTR = "Translation" — early ARM CPUs      │
│  only translated user-mode addresses, so the OS needed a special    │
│  "load with translation" instruction to access user data.           │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 12. Execution Permissions (UXN / PXN)

In addition to data access permissions, there are **execution permissions** that control whether code can be run from a region:

```
┌──────────────────────────────────────────────────────────────────────┐
│              EXECUTE NEVER BITS                                       │
│                                                                      │
│  UXN (User Execute Never):                                           │
│  → When UXN = 1: EL0 (user apps) CANNOT execute code from here     │
│  → When UXN = 0: EL0 CAN execute code from here                    │
│                                                                      │
│  PXN (Privileged Execute Never):                                     │
│  → When PXN = 1: EL1/EL2 (kernel/hypervisor) CANNOT execute here   │
│  → When PXN = 0: EL1/EL2 CAN execute code from here                │
│  → Called "XN" at EL3 and EL2 (when HCR_EL2.E2H=0)                │
│                                                                      │
│  Typical Configuration:                                              │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Region          │  UXN  │  PXN  │  Reason                  │   │
│  ├──────────────────┼───────┼───────┼──────────────────────────┤   │
│  │  App Code        │   0   │   1   │  App runs it, kernel      │   │
│  │                  │       │       │  should NOT               │   │
│  ├──────────────────┼───────┼───────┼──────────────────────────┤   │
│  │  Kernel Code     │   1   │   0   │  Kernel runs it, apps     │   │
│  │                  │       │       │  should NOT               │   │
│  ├──────────────────┼───────┼───────┼──────────────────────────┤   │
│  │  Data (heap/     │   1   │   1   │  Nobody should execute    │   │
│  │  stack/MMIO)     │       │       │  data regions!            │   │
│  └──────────────────┴───────┴───────┴──────────────────────────┘   │
│                                                                      │
│  ⚠️  ARM STRONGLY RECOMMENDS: Always mark Device regions as XN!    │
│      (Both UXN=1 and PXN=1)                                         │
│                                                                      │
│  Additional control: SCTLR_ELx has bits to make ALL writable        │
│  addresses non-executable (Write XOR Execute policy).               │
│                                                                      │
│  Rule: A location with EL0 write permissions is NEVER executable    │
│        at EL1.                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 13. Permission Indirection (Armv8.9-A+)

> **Introduced in:** Armv8.9-A / Armv9.4-A

Traditional permissions use the `AP` bits directly in the translation table entry. **Permission Indirection** adds a level of indirection — similar to how `MAIR_ELx` works for memory types.

```
┌──────────────────────────────────────────────────────────────────────┐
│              PERMISSION INDIRECTION MECHANISM                        │
│                                                                      │
│  Traditional (Direct):                                               │
│  Translation Table Entry → AP bits → Permissions                    │
│                                                                      │
│  With Permission Indirection:                                        │
│  Translation Table Entry → PIIndex → PIR_ELx → Permissions          │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Translation Table Entry:                                    │   │
│  │  ┌──────────────────────────────────────────────────────┐   │   │
│  │  │  ...  │  PIIndex[3:0] = 0b0011 (index 3)  │  ...    │   │   │
│  │  └──────────────────────────────────────────────────────┘   │   │
│  │                    │                                         │   │
│  │                    │ Index = 3                               │   │
│  │                    ▼                                         │   │
│  │  PIR_EL1 (Permission Indirection Register):                  │   │
│  │  ┌────────┬────────┬────────┬────────┬────────┬────────┐   │   │
│  │  │ Perm15 │  ...   │ Perm4  │ Perm3  │ Perm2  │ Perm1  │   │   │
│  │  │        │        │        │  RW+OV │        │        │   │   │
│  │  └────────┴────────┴────────┴────────┴────────┴────────┘   │   │
│  │                              ↑                               │   │
│  │                    Selected: Read/Write + Overlay applied    │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Permission Indirection Registers:                                   │
│  ┌──────────────┬──────────────────────────────────────────────┐    │
│  │  Register    │  Purpose                                     │    │
│  ├──────────────┼──────────────────────────────────────────────┤    │
│  │  PIRE0_EL1   │  Unprivileged EL1&0 base permissions         │    │
│  │  PIRE0_EL2   │  Unprivileged EL2&0 base permissions         │    │
│  │  PIR_EL1     │  Privileged EL1 base permissions             │    │
│  │  PIR_EL2     │  Privileged EL2 base permissions             │    │
│  │  PIR_EL3     │  Privileged EL3 base permissions             │    │
│  └──────────────┴──────────────────────────────────────────────┘    │
│                                                                      │
│  Benefits:                                                           │
│  ✅ More permission types available (16 slots vs 4 AP combinations) │
│  ✅ Change permissions for many pages by updating ONE register      │
│  ✅ More efficient use of translation table bits                    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 14. Permission Overlays (Armv8.9-A+)

> **Introduced in:** Armv8.9-A / Armv9.4-A

Permission overlays allow **further restricting** permissions **without a TLB invalidate** — a major performance win!

```
┌──────────────────────────────────────────────────────────────────────┐
│              PERMISSION OVERLAY MECHANISM                            │
│                                                                      │
│  Base permissions (from PIR_ELx) can be FURTHER RESTRICTED          │
│  by the Permission Overlay Register (POR_ELx).                      │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Translation Table Entry:                                    │   │
│  │  ┌──────────────────────────────────────────────────────┐   │   │
│  │  │  PIIndex = 3  │  POIndex = 6  │  ...                 │   │   │
│  │  └──────────────────────────────────────────────────────┘   │   │
│  │       │                  │                                   │   │
│  │       ▼                  ▼                                   │   │
│  │  PIR_EL1[3]          POR_EL0[6]                             │   │
│  │  = Read/Write        = Read Only                            │   │
│  │  + Overlay applied                                          │   │
│  │       │                  │                                   │   │
│  │       └──────────────────┘                                   │   │
│  │                  │                                           │   │
│  │                  ▼                                           │   │
│  │         Effective Permission = Read Only                     │   │
│  │         (overlay restricts base RW → RO)                    │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Permission Overlay Registers:                                       │
│  ┌──────────────┬──────────────────────────────────────────────┐    │
│  │  Register    │  Purpose                                     │    │
│  ├──────────────┼──────────────────────────────────────────────┤    │
│  │  POR_EL0     │  Unprivileged EL0 stage 1 overlay            │    │
│  │  POR_EL1     │  Privileged EL1 stage 1 overlay              │    │
│  │  POR_EL2     │  Privileged EL2 stage 1 overlay              │    │
│  │  POR_EL3     │  Privileged EL3 stage 1 overlay              │    │
│  └──────────────┴──────────────────────────────────────────────┘    │
│                                                                      │
│  KEY ADVANTAGE:                                                      │
│  • Base permissions (PIR) CAN be cached in TLB                     │
│  • Overlay permissions (POR) CANNOT be cached in TLB               │
│  → Changing overlay = NO TLB invalidate needed! 🚀                 │
│  → Huge performance win for frequently changing permissions         │
└──────────────────────────────────────────────────────────────────────┘
```

### Complete Example: Using Both Together

```
Scenario: OS (EL1) sets up memory for an app (EL0)

Step 1 — Set base permissions via PIR:
  OS selects index 3 in PIR_EL1
  Sets PIR_EL1[index 3] = 0b0101 = "Read/Write + Overlay applied"
  Sets PIIndex = 3 in all relevant TTDs

Step 2 — Set overlay permissions via POR:
  OS selects index 6 in POR_EL1
  Sets POR_EL1[index 6] = 0b0001 = "Read Only"

Step 3 — App requests restriction:
  App (EL0) calls OS via syscall: "restrict these pages to read-only"
  OS sets POIndex = 6 in those pages' TTDs

Result:
  Base permission: Read/Write
  Overlay:         Read Only
  Effective:       Read Only (overlay restricts base)

  To restore write access: just change POR_EL1[6] — NO TLBI needed!
```

---

## 15. Access Flag (AF bit)

The **Access Flag** tracks whether a page has been accessed (read or written):

```
┌──────────────────────────────────────────────────────────────────────┐
│              ACCESS FLAG (AF BIT)                                    │
│                                                                      │
│  AF = 0  →  Page has NOT been accessed                              │
│  AF = 1  →  Page HAS been accessed                                  │
│                                                                      │
│  Use Case: OS Page Replacement (Paging/Swapping)                    │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  OS needs to free RAM → which pages to evict to disk?        │   │
│  │                                                              │   │
│  │  Pages with AF=0 → not recently used → good candidates!     │   │
│  │  Pages with AF=1 → recently used → keep in RAM              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Note: In bare-metal (no OS), AF is usually pre-set to 1.           │
│        You only need AF management when doing page swapping.        │
└──────────────────────────────────────────────────────────────────────┘
```

### Two Ways to Update the AF Bit

```
┌──────────────────────────────────────────────────────────────────────┐
│              AF UPDATE METHODS                                       │
│                                                                      │
│  Method 1: SOFTWARE UPDATE (default)                                 │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  1. Table created with AF = 0                                │   │
│  │  2. App accesses the page                                    │   │
│  │  3. Hardware generates ACCESS FLAG FAULT exception           │   │
│  │  4. OS exception handler sets AF = 1 in the table entry     │   │
│  │  5. OS returns from exception, app continues                 │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Method 2: HARDWARE UPDATE (Armv8.1-A+)                             │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Enable: Set TCR_ELx.HA = 1                                  │   │
│  │                                                              │   │
│  │  1. Table created with AF = 0                                │   │
│  │  2. App accesses the page                                    │   │
│  │  3. Hardware AUTOMATICALLY sets AF = 1                       │   │
│  │  4. No exception generated — transparent to software!       │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Hardware update is faster (no exception overhead).                 │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 16. Dirty State (DBM bit)

The **Dirty Bit Modifier (DBM)** tracks whether a page has been **written to** (modified):

```
┌──────────────────────────────────────────────────────────────────────┐
│              DIRTY STATE TRACKING (DBM BIT)                          │
│                                                                      │
│  Why is dirty state important?                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  1. File loaded from disk into RAM                           │   │
│  │  2. User edits the file (RAM is now "dirty")                 │   │
│  │  3. OS needs to free RAM → evict this page                   │   │
│  │  4. Is the RAM copy newer than the disk copy?                │   │
│  │     • YES (dirty) → must write RAM back to disk first!      │   │
│  │     • NO (clean)  → can just discard RAM copy               │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  How DBM works (Armv8.1-A+):                                        │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  1. OS creates page with AP = Read-Only, DBM = 1            │   │
│  │  2. App tries to WRITE to the page                           │   │
│  │  3. Hardware detects write to Read-Only + DBM=1 page        │   │
│  │  4. Hardware AUTOMATICALLY changes AP to Read-Write          │   │
│  │     (this signals the page is now "dirty")                   │   │
│  │  5. No exception needed!                                     │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  When DBM=1: AP[2] / S2AP[1] bits change meaning:                  │
│  → Instead of recording write permission, they record DIRTY STATE  │
│  → These bits no longer cause permission faults                     │
│                                                                      │
│  Without DBM (software approach):                                   │
│  → Page marked Read-Only → write causes Permission Fault           │
│  → Exception handler marks page Read-Write and returns             │
│  → Still used for Copy-on-Write (CoW) scenarios                    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 17. Alignment

An access is **aligned** if the address is a multiple of the element size:

```
┌──────────────────────────────────────────────────────────────────────┐
│              ALIGNMENT RULES                                         │
│                                                                      │
│  Instruction  │  Size  │  Address must be multiple of               │
│  ─────────────┼────────┼──────────────────────────────────────────  │
│  LDRB / STRB  │  1 B   │  1 (any address — always aligned)         │
│  LDRH / STRH  │  2 B   │  2                                         │
│  LDR  / STR   │  4 B   │  4                                         │
│  LDR  / STR   │  8 B   │  8 (X registers)                          │
│  LDP  / STP   │  8 B   │  8 (NOT 16! aligned to element, not pair) │
│                                                                      │
│  Example:                                                            │
│  LDP X0, X1, [X2]   ← loads two 64-bit values (128 bits total)     │
│  X2 must be aligned to 8 bytes (64 bits), NOT 16 bytes!            │
│                                                                      │
│  Unaligned Access Rules:                                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Normal Memory:  Unaligned access ALLOWED by default         │   │
│  │                  (can be trapped by setting SCTLR_ELx.A=1)  │   │
│  │                                                              │   │
│  │  Device Memory:  Unaligned access ALWAYS FAULTS             │   │
│  │                  (generates Alignment Fault exception)       │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ⚠️  IMPORTANT: When MMU is disabled, all accesses are treated as  │
│      Device memory. So unaligned accesses ALWAYS fault when        │
│      MMU is off! This catches many startup code bugs.              │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 18. Endianness

```
┌──────────────────────────────────────────────────────────────────────┐
│              ENDIANNESS IN ARMv8-A                                   │
│                                                                      │
│  Little-Endian: Least significant byte at lowest address            │
│  Big-Endian:    Most significant byte at lowest address             │
│                                                                      │
│  Example: Value 0x12345678 stored at address 0x1000                 │
│                                                                      │
│  Little-Endian:                                                      │
│  ┌────────┬────────┬────────┬────────┐                              │
│  │  0x78  │  0x56  │  0x34  │  0x12  │                              │
│  │ 0x1000 │ 0x1001 │ 0x1002 │ 0x1003 │                              │
│  └────────┴────────┴────────┴────────┘                              │
│                                                                      │
│  Big-Endian:                                                         │
│  ┌────────┬────────┬────────┬────────┐                              │
│  │  0x12  │  0x34  │  0x56  │  0x78  │                              │
│  │ 0x1000 │ 0x1001 │ 0x1002 │ 0x1003 │                              │
│  └────────┴────────┴────────┴────────┘                              │
│                                                                      │
│  ARMv8-A Rules:                                                      │
│  • Instruction fetches: ALWAYS little-endian                        │
│  • Data accesses: IMPLEMENTATION DEFINED (both or one)              │
│  • Arm Cortex-A: Supports BOTH big-endian and little-endian         │
│  • Endianness configured PER EXCEPTION LEVEL                        │
│  • Controlled by SCTLR_ELx.E0E (EL0) and SCTLR_ELx.EE (EL1+)     │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 19. Memory Aliasing and Mismatched Types

**Aliasing** occurs when a single physical address has **multiple virtual addresses** pointing to it:

```
┌──────────────────────────────────────────────────────────────────────┐
│              MEMORY ALIASING                                         │
│                                                                      │
│  Virtual Address Space          Physical Address Space               │
│  ┌──────────────────────┐       ┌──────────────────────┐           │
│  │  VA 0x1000 (Normal)  │──────►│                      │           │
│  │                      │       │  PA 0x8000           │           │
│  │  VA 0x5000 (Normal)  │──────►│  (Location A)        │           │
│  └──────────────────────┘       └──────────────────────┘           │
│  ✅ COMPATIBLE aliases (same type) — OK!                            │
│                                                                      │
│  ┌──────────────────────┐       ┌──────────────────────┐           │
│  │  VA 0x2000 (Normal)  │──────►│                      │           │
│  │                      │       │  PA 0x9000           │           │
│  │  VA 0x6000 (Device)  │──────►│  (Location B)        │           │
│  └──────────────────────┘       └──────────────────────┘           │
│  ❌ INCOMPATIBLE aliases (Normal + Device) — DANGEROUS!             │
│                                                                      │
│  Compatible means:                                                   │
│  • Same memory TYPE (both Normal, or both Device)                   │
│  • For Device: same SUB-TYPE (both nGnRE, etc.)                    │
│  • For Normal: same cacheability AND shareability                   │
│                                                                      │
│  Incompatible aliases can cause:                                     │
│  • Cache coherency problems                                          │
│  • Unexpected behavior (data corruption)                            │
│  • Performance degradation                                           │
│                                                                      │
│  ARM STRONGLY RECOMMENDS: Never create incompatible aliases!        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 20. Quick Reference: All Attributes at a Glance

### Translation Table Entry Fields

| Field | Bits | Meaning |
|-------|------|---------|
| `AttrIndx` | [4:2] | Index into MAIR_ELx (selects memory type) |
| `AP[2:1]` | [7:6] | Access permissions (read/write, privilege) |
| `AF` | [10] | Access Flag (0=not accessed, 1=accessed) |
| `nG` | [11] | Non-Global (1=tagged with ASID in TLB) |
| `DBM` | [51] | Dirty Bit Modifier (hardware dirty tracking) |
| `PXN` | [53] | Privileged Execute Never |
| `UXN` | [54] | User Execute Never |

### Memory Type Summary

| Type | Cacheable | Speculative | Side-Effects | Code Allowed |
|------|-----------|-------------|--------------|--------------|
| Normal | ✅ Yes | ✅ Yes | ❌ No | ✅ Yes |
| Device-GRE | ❌ No | ❌ No | ✅ Yes | ❌ No (use XN) |
| Device-nGRE | ❌ No | ❌ No | ✅ Yes | ❌ No (use XN) |
| Device-nGnRE | ❌ No | ❌ No | ✅ Yes | ❌ No (use XN) |
| Device-nGnRnE | ❌ No | ❌ No | ✅ Yes | ❌ No (use XN) |

### Key Registers

| Register | Purpose |
|----------|---------|
| `MAIR_EL1` | Memory type attributes for EL1 stage 1 |
| `MAIR_EL2` | Memory type attributes for EL2 stage 1 |
| `MAIR_EL3` | Memory type attributes for EL3 stage 1 |
| `SCTLR_ELx` | System control (MMU enable, cache enable, endianness) |
| `PIR_EL1/2/3` | Permission Indirection Registers (Armv8.9-A+) |
| `PIRE0_EL1/2` | Unprivileged Permission Indirection Registers (Armv8.9-A+) |
| `POR_EL0/1/2/3` | Permission Overlay Registers (Armv8.9-A+) |

---

## 21. Knowledge Check Q&A

**Q: Where do the attributes for an address location come from?**
> From the **translation tables** — typically the Block/Page descriptor. Hierarchical attributes in higher-level Table descriptors can override lower-level entries.

**Q: What are the two memory types in Armv8-A?**
> **Normal** and **Device**.

**Q: What does `-nGnRE` mean in `Device-nGnRE`?**
> - `nG` = No Gathering (accesses cannot be merged)
> - `nR` = No Re-ordering (accesses must be in program order)
> - `E` = Early Write Acknowledgement **allowed** (write can complete before reaching the peripheral)

**Q: Why might a page be marked as `PXN=1, UXN=0`?**
> Application code needs to be **executable in User space (EL0)** but should **never be executed with kernel privileges (EL1/EL2)**. Setting `PXN=1` prevents kernel execution; `UXN=0` allows user execution.

**Q: What is the AF bit typically used for?**
> Tracking which pages have been **recently accessed**, so the OS can identify candidates for **paging out** (evicting to disk).

**Q: What endianness are instruction fetches?**
> Always **little-endian** in Armv8-A.

**Q: Before enabling the MMU, some startup code makes an unaligned access, causing an alignment fault. Why?**
> When the MMU is disabled, all accesses are treated as **Device memory**. Unaligned accesses to Device regions **always generate an alignment fault**.

**Q: What is the key advantage of Permission Overlays over traditional permission changes?**
> Overlay settings are **not cached in the TLB**, so changing them does **not require a TLB invalidate (TLBI)**. This avoids the significant performance cost of TLB maintenance.

---

*📖 Previous: [README_1_Memory_Management_Guide.md](README_1_Memory_Management_Guide.md)*  
*📖 Next: [README_3_Memory_Systems_Ordering_and_Barriers.md](README_3_Memory_Systems_Ordering_and_Barriers.md)*
