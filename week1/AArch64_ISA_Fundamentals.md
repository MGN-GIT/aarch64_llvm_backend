# AArch64 Register File — Deep Dive
> Source of truth: `AArch64RegisterInfo.td`

---

## Table of Contents

- [How Registers Are Defined in TableGen](#how-registers-are-defined-in-tablegen)
- [Part 1 — General Purpose Registers (GPR)](#part-1--general-purpose-registers-gpr)
  - [The W/X Relationship — Sub-registers](#the-wx-relationship--sub-registers)
  - [Special GPRs: SP, XZR, FP, LR](#special-gprs-sp-xzr-fp-lr)
  - [GPR Register Classes](#gpr-register-classes)
  - [Condition Flags: NZCV](#condition-flags-nzcv)
- [Part 2 — NEON / FP Registers](#part-2--neon--fp-registers)
  - [The B/H/S/D/Q Sub-register Tower](#the-bhsdq-sub-register-tower)
  - [The _HI Artificial Registers](#the-_hi-artificial-registers)
  - [FPR Register Classes](#fpr-register-classes)
  - [NEON Vector Tuples: DD, QQ, etc.](#neon-vector-tuples-dd-qq-etc)
  - [The V Operands](#the-v-operands)
- [Part 3 — SVE Registers](#part-3--sve-registers)
  - [Z Registers — Scalable Vectors](#z-registers--scalable-vectors)
  - [P Registers — Predicates](#p-registers--predicates)
  - [PN Registers — Predicates-as-Counters](#pn-registers--predicates-as-counters)
  - [SVE Register Classes: ZPR, PPR](#sve-register-classes-zpr-ppr)
  - [SVE Multi-register Tuples](#sve-multi-register-tuples)
  - [SVE Strided Register Groups](#sve-strided-register-groups)
- [Part 4 — SME Registers](#part-4--sme-registers)
  - [The ZA Tile Hierarchy](#the-za-tile-hierarchy)
  - [ZA Sub-register Indices](#za-sub-register-indices)
  - [ZT0 — The Lookup Table Register](#zt0--the-lookup-table-register)
  - [SME Register Classes: MPR, ZTR](#sme-register-classes-mpr-ztr)
  - [SME Operand Types](#sme-operand-types)
- [Sub-register Index Reference](#sub-register-index-reference)
- [Key Patterns to Recognise](#key-patterns-to-recognise)

---

## How Registers Are Defined in TableGen

Every register in this file is an instance of the `AArch64Reg` class:

```tablegen
// AArch64RegisterInfo.td
class AArch64Reg<bits<16> enc, string n, list<Register> subregs = [],
                 list<string> altNames = []>
    : Register<n, altNames> {
  let HWEncoding = enc;   // The hardware encoding (0-31 for most regs)
  let Namespace = "AArch64";
  let SubRegs = subregs;  // Which smaller registers live inside this one
}
```

Three fields matter most:
- **`enc`** — the hardware encoding. `W0` and `X0` both encode as `0`. `WSP` and `WZR` both encode as `31` — they share an encoding, disambiguated by context.
- **`n`** — the assembly name string (`"x0"`, `"sp"`, `"z15"`, etc.)
- **`subregs`** — the list of smaller registers that are sub-parts of this one. This is what makes the sub-register hierarchy work.

---

## Part 1 — General Purpose Registers (GPR)

### The W/X Relationship — Sub-registers

The 64-bit `X` registers contain the 32-bit `W` registers as their lower half. This is expressed in TableGen using `SubRegIndices`:

```tablegen
// Step 1: Define the sub-register index
def sub_32   : SubRegIndex<32>;      // 32-bit wide, at bit offset 0
def sub_32_hi : SubRegIndex<32, 32>; // 32-bit wide, at bit offset 32

// Step 2: Define the W registers (no sub-registers themselves)
def W0 : AArch64Reg<0, "w0">, DwarfRegNum<[0]>;
// ...

// Step 3: Define the X registers WITH sub-registers
// W0_HI is an artificial register representing bits [63:32] of X0
def W0_HI : AArch64Reg<-1, "w0_hi"> { let isArtificial = 1; }

let SubRegIndices = [sub_32, sub_32_hi], CoveredBySubRegs = 1 in {
  def X0 : AArch64Reg<0, "x0", [W0, W0_HI]>, DwarfRegAlias<W0>;
  // ...
}
```

**What this means in practice:**
- `X0` contains two sub-registers: `W0` (bits 0–31, index `sub_32`) and `W0_HI` (bits 32–63, index `sub_32_hi`).
- `CoveredBySubRegs = 1` tells LLVM that the sub-registers together fully cover the parent — no bits are unaccounted for.
- `DwarfRegAlias<W0>` means `X0` shares the same DWARF debug number as `W0` (DWARF number 0).
- Writing to `W0` in AArch64 **zero-extends** into `X0` — the hardware clears bits 32–63. The `W0_HI` artificial register exists so LLVM's liveness analysis can model this correctly.

---

### Special GPRs: SP, XZR, FP, LR

```tablegen
def FP  : AArch64Reg<29, "x29", [W29, W29_HI]>, DwarfRegAlias<W29>;
def LR  : AArch64Reg<30, "x30", [W30, W30_HI]>, DwarfRegAlias<W30>;
def SP  : AArch64Reg<31, "sp",  [WSP, WSP_HI]>, DwarfRegAlias<WSP>;
def XZR : AArch64Reg<31, "xzr", [WZR, WZR_HI]>, DwarfRegAlias<WSP> {
  let isConstant = true;
}
```

Key observations:
- **`FP` (Frame Pointer)** is physically `X29`. It has the assembly name `"x29"` but the LLVM symbol name `FP`. In code you'll see `AArch64::FP`, not `AArch64::X29`.
- **`LR` (Link Register)** is physically `X30`. Same pattern — LLVM name is `AArch64::LR`.
- **`SP` and `XZR` share hardware encoding `31`**. The CPU distinguishes them by instruction context: memory instructions use `SP`; data instructions use `XZR`. `DwarfRegAlias<WSP>` means both share DWARF number 31.
- **`isConstant = true`** on `XZR`/`WZR` tells LLVM this register always reads as zero and writes are discarded. This enables optimizations like replacing dead definitions with `XZR`.

---

### GPR Register Classes

Register classes group registers that are interchangeable for a given purpose. The GPR classes form a careful hierarchy:

```tablegen
// Base: W0-W30 only (no WZR, no WSP)
def GPR32common : RegisterClass<"AArch64", [i32], 32,
                                (sequence "W%u", 0, 30)>;

// Base: X0-X28, FP, LR (no XZR, no SP)
def GPR64common : RegisterClass<"AArch64", [i64], 64,
                                (add (sequence "X%u", 0, 28), FP, LR)>;

// GPR32/GPR64: add the zero register (WZR/XZR)
def GPR32 : RegisterClass<"AArch64", [i32], 32, (add GPR32common, WZR)>;
def GPR64 : RegisterClass<"AArch64", [i64], 64, (add GPR64common, XZR)>;

// GPR32sp/GPR64sp: add the stack pointer (WSP/SP) instead
def GPR32sp : RegisterClass<"AArch64", [i32], 32, (add GPR32common, WSP)>;
def GPR64sp : RegisterClass<"AArch64", [i64], 64, (add GPR64common, SP)>;

// GPR32all/GPR64all: everything — used as a common super-class
def GPR32all : RegisterClass<"AArch64", [i32], 32, (add GPR32common, WZR, WSP)>;
def GPR64all : RegisterClass<"AArch64", [i64], 64, (add GPR64common, XZR, SP)>;
```

**Why the split?**
- `ADD X0, X1, X2` — uses `GPR64` (XZR allowed, SP not)
- `ADD X0, SP, #16` — uses `GPR64sp` (SP allowed, XZR not)
- `STR X0, [SP]` — the base register uses `GPR64sp`
- Instructions that accept either use `GPR64all`

**Argument registers:**
```tablegen
def GPR32arg : RegisterClass<"AArch64", [i32], 32, (sequence "W%u", 0, 7)>;
def GPR64arg : RegisterClass<"AArch64", [i64], 64, (sequence "X%u", 0, 7)>;
```
`X0–X7` are the argument/return registers per AAPCS64. These classes are used in calling convention definitions.

**Tail call registers:**
```tablegen
// Can't use callee-saved regs for indirect tail calls
def tcGPR64 : RegisterClass<"AArch64", [i64], 64,
    (sub GPR64common, X19, X20, X21, X22, X23, X24, X25, X26, X27, X28, FP, LR)>;

// BTI-restricted: only X16/X17 can branch to "BTI c" targets
def tcGPRx16x17 : RegisterClass<"AArch64", [i64], 64, (add X16, X17)>;
```

**Atomic CASP pairs:**
```tablegen
// For CASP instruction: requires even/odd register pairs
def WSeqPairs : RegisterTuples<[sube32, subo32], [...]>;
def XSeqPairs : RegisterTuples<[sube64, subo64], [...]>;
def WSeqPairsClass : RegisterClass<"AArch64", [untyped], 32, (add WSeqPairs)>;
def XSeqPairsClass : RegisterClass<"AArch64", [untyped], 64, (add XSeqPairs)>;
```
`CASP` requires `(Wn, Wn+1)` pairs where `n` is even. The `sube32`/`subo32` sub-register indices model the even/odd halves.

---

### Condition Flags: NZCV

```tablegen
def NZCV : AArch64Reg<0, "nzcv">;

def CCR : RegisterClass<"AArch64", [i32], 32, (add NZCV)> {
  let CopyCost = -1;   // Never copy this register
  let isAllocatable = 0; // Never allocate it
}
```

- `NZCV` is the processor flags register (Negative, Zero, Carry, oVerflow).
- `CopyCost = -1` means the register allocator will never try to copy it.
- `isAllocatable = 0` means it is never assigned to a virtual register.
- It appears as an implicit def/use on instructions like `ADDS`, `CMP`, `Bcc`.

Other special-purpose registers defined similarly:
```tablegen
def FFR  : AArch64Reg<0, "ffr">,  DwarfRegNum<[47]>; // SVE First Fault Register
def VG   : AArch64Reg<0, "vg">,   DwarfRegNum<[46]>; // Virtual Granule (SVE vector length)
def FPCR : AArch64Reg<0, "fpcr">;                    // FP Control Register
def FPSR : AArch64Reg<0, "fpsr">;                    // FP Status Register
def FPMR : AArch64Reg<0, "fpmr">;                    // FP Mode Register
```

---

## Part 2 — NEON / FP Registers

### The B/H/S/D/Q Sub-register Tower

All 32 floating-point/NEON registers share the same hardware register file. They are accessed at different widths using different names. The hierarchy from smallest to largest:

```
B (8-bit)  ──sub-reg of──▶  H (16-bit)  ──sub-reg of──▶  S (32-bit)
                                                               │
                                                         sub-reg of
                                                               │
                                                           D (64-bit)  ──sub-reg of──▶  Q (128-bit)
                                                                                              │
                                                                                        sub-reg of
                                                                                              │
                                                                                          Z (scalable)
```

In TableGen, this is built bottom-up:

```tablegen
// Sub-register indices for the FP tower
def bsub    : SubRegIndex<8,   0>;   // 8-bit  at offset 0
def hsub    : SubRegIndex<16,  0>;   // 16-bit at offset 0
def ssub    : SubRegIndex<32,  0>;   // 32-bit at offset 0
def dsub    : SubRegIndex<64,  0>;   // 64-bit at offset 0
def zsub    : SubRegIndex<128, 0>;   // 128-bit at offset 0

// _hi variants: the upper bits not covered by the lower sub-register
def bsub_hi : SubRegIndex<8,  8>;    // bits [15:8]  of H
def hsub_hi : SubRegIndex<16, 16>;   // bits [31:16] of S
def ssub_hi : SubRegIndex<32, 32>;   // bits [63:32] of D
def dsub_hi : SubRegIndex<64, 64>;   // bits [127:64] of Q

// Level 1: B registers (no sub-registers)
def B0 : AArch64Reg<0, "b0">, DwarfRegNum<[64]>;

// Level 2: H contains [B, B_HI]
let SubRegIndices = [bsub, bsub_hi] in {
  def H0 : AArch64Reg<0, "h0", [B0, B0_HI]>, DwarfRegAlias<B0>;
}

// Level 3: S contains [H, H_HI]
let SubRegIndices = [hsub, hsub_hi] in {
  def S0 : AArch64Reg<0, "s0", [H0, H0_HI]>, DwarfRegAlias<B0>;
}

// Level 4: D contains [S, S_HI], has alternate name "v0"
let SubRegIndices = [ssub, ssub_hi], RegAltNameIndices = [vreg, vlist1] in {
  def D0 : AArch64Reg<0, "d0", [S0, S0_HI], ["v0", ""]>, DwarfRegAlias<B0>;
}

// Level 5: Q contains [D, D_HI], also has alternate name "v0"
let SubRegIndices = [dsub, dsub_hi], RegAltNameIndices = [vreg, vlist1] in {
  def Q0 : AArch64Reg<0, "q0", [D0, D0_HI], ["v0", ""]>, DwarfRegAlias<B0>;
}
```

**Key insight:** All of `B0`, `H0`, `S0`, `D0`, `Q0` share hardware encoding `0` and all alias to DWARF register `64`. They are five different views of the same physical register. The `DwarfRegAlias<B0>` on H0/S0/D0/Q0 means they all share DWARF number 64.

**The `"v0"` alternate name:** `D0` and `Q0` both have the alternate name `"v0"`. In NEON assembly, `v0.8b` (8 bytes = 64-bit) refers to `D0`, and `v0.16b` (16 bytes = 128-bit) refers to `Q0`. The assembler uses the element type suffix to pick the right register.

---

### The _HI Artificial Registers

```tablegen
foreach i = 0-31 in {
  def B#i#_HI : AArch64Reg<-1, "b"#i#"_hi"> { let isArtificial = 1; }
  def H#i#_HI : AArch64Reg<-1, "h"#i#"_hi"> { let isArtificial = 1; }
  def S#i#_HI : AArch64Reg<-1, "s"#i#"_hi"> { let isArtificial = 1; }
  def D#i#_HI : AArch64Reg<-1, "d"#i#"_hi"> { let isArtificial = 1; }
  def Q#i#_HI : AArch64Reg<-1, "q"#i#"_hi"> { let isArtificial = 1; }
}
```

- `isArtificial = 1` means these registers do not correspond to any real hardware register.
- `HWEncoding = -1` confirms they have no hardware encoding.
- They exist purely so that LLVM's sub-register liveness tracking can account for the upper bits of each register.
- For example: writing to `S0` (32-bit) does NOT zero the upper 96 bits of `Q0` in NEON (unlike GPRs where writing `W0` zeros the upper 32 bits of `X0`). The `S0_HI` register models those upper bits as a separate entity so liveness analysis knows they are still live after a 32-bit write.

---

### FPR Register Classes

```tablegen
def FPR8   : RegisterClass<"AArch64", [i8, aarch64mfp8], 8,
                           (sequence "B%u", 0, 31)>;
def FPR16  : RegisterClass<"AArch64", [f16, bf16, i16], 16,
                           (sequence "H%u", 0, 31)>;
def FPR32  : RegisterClass<"AArch64", [f32, i32], 32,
                           (sequence "S%u", 0, 31)>;
def FPR64  : RegisterClass<"AArch64", [f64, i64, v2f32, v1f64,
                                       v8i8, v4i16, v2i32, v1i64,
                                       v4f16, v4bf16],
                           64, (sequence "D%u", 0, 31)>;
def FPR128 : RegisterClass<"AArch64",
                           [v16i8, v8i16, v4i32, v2i64, v4f32, v2f64,
                            f128, v8f16, v8bf16],
                           128, (sequence "Q%u", 0, 31)>;
```

Notice that `FPR64` accepts both scalar types (`f64`, `i64`) and vector types (`v8i8`, `v4i16`, `v2i32`, etc.). This is because a 64-bit NEON register can hold either a scalar or a packed vector — the hardware doesn't distinguish.

**Restricted classes** for instructions that only accept a subset of registers:
```tablegen
def FPR128_lo   : RegisterClass<..., (trunc FPR128, 16)>;  // Q0-Q15 only
def FPR128_0to7 : RegisterClass<..., (trunc FPR128, 8)>;   // Q0-Q7 only
def FPR16_lo    : RegisterClass<..., (trunc FPR16, 16)>;   // H0-H15 only
def FPR64_lo    : RegisterClass<..., (trunc FPR64, 16)>;   // D0-D15 only
```
Some NEON instructions (e.g., indexed multiply `FMLA Vd.4s, Vn.4s, Vm.s[lane]`) can only use `V0–V15` for the indexed operand because the lane index field is only 4 bits wide.

---

### NEON Vector Tuples: DD, QQ, etc.

NEON load/store instructions (`LD2`, `LD3`, `LD4`, `ST2`, etc.) operate on groups of consecutive registers. These are modelled as `RegisterTuples`:

```tablegen
// Pairs of D registers: {D0,D1}, {D1,D2}, ..., {D31,D0} (wrapping)
def DSeqPairs : RegisterTuples<[dsub0, dsub1],
                               [(rotl FPR64, 0), (rotl FPR64, 1)]>;
def DD   : RegisterClass<"AArch64", [untyped], 64, (add DSeqPairs)> {
  let Size = 128;  // Two 64-bit registers = 128 bits total
}

// Triples and quads similarly
def DDD  : RegisterClass<"AArch64", [untyped], 64, (add DSeqTriples)> { let Size = 192; }
def DDDD : RegisterClass<"AArch64", [untyped], 64, (add DSeqQuads)>   { let Size = 256; }

// Same for 128-bit Q registers
def QQ   : RegisterClass<"AArch64", [untyped], 128, (add QSeqPairs)>   { let Size = 256; }
def QQQ  : RegisterClass<"AArch64", [untyped], 128, (add QSeqTriples)> { let Size = 384; }
def QQQQ : RegisterClass<"AArch64", [untyped], 128, (add QSeqQuads)>   { let Size = 512; }
```

- `rotl FPR64, 1` means "rotate the FPR64 list left by 1" — so the second element of a pair is always the register one higher (with wrap-around).
- `dsub0`, `dsub1` are the sub-register indices for the first and second element of the tuple.
- `[untyped]` means the register class doesn't have a specific value type — it's just a container.

---

### The V Operands

```tablegen
def V64  : RegisterOperand<FPR64,  "printVRegOperand">;
def V128 : RegisterOperand<FPR128, "printVRegOperand">;
```

`RegisterOperand` wraps a `RegisterClass` with custom printing/parsing behaviour. `V64` and `V128` print registers using the `v0`–`v31` naming convention with element type suffixes (e.g., `v0.4s`, `v1.2d`). This is what you see in NEON assembly output.

---

## Part 3 — SVE Registers

### Z Registers — Scalable Vectors

```tablegen
// Sub-register indices for Z registers
def zsub    : SubRegIndex<128, 0>;    // The bottom 128 bits (= the Q register)
def zsub_hi : SubRegIndex<-1, 128>;   // The upper scalable portion (size unknown)

// Z registers contain [Q, Q_HI]
let SubRegIndices = [zsub, zsub_hi] in {
  def Z0 : AArch64Reg<0, "z0", [Q0, Q0_HI]>, DwarfRegNum<[96]>;
  def Z1 : AArch64Reg<1, "z1", [Q1, Q1_HI]>, DwarfRegNum<[97]>;
  // ... Z2-Z31 similarly, DWARF numbers 98-127
}
```

**Critical detail:** `zsub_hi` has size `-1`. This is how LLVM represents a sub-register of **unknown size** — the upper portion of a scalable vector register whose actual size depends on the hardware's SVE vector length (128 to 2048 bits in multiples of 128). The `-1` tells LLVM "this is scalable, don't try to compute a fixed size."

**The Z/Q relationship:** `Z0` contains `Q0` as its bottom 128 bits (via `zsub`). This means:
- A NEON instruction writing `Q0` also writes the bottom of `Z0`.
- An SVE instruction writing `Z0` writes all of it, including the part above `Q0`.
- LLVM's liveness analysis handles this correctly because of the sub-register structure.

**DWARF numbers:** Z registers are DWARF 96–127, separate from Q registers (which alias to B registers at DWARF 64–95).

---

### P Registers — Predicates

```tablegen
// PN registers are the "predicate-as-counter" base
def PN0 : AArch64Reg<0, "pn0">, DwarfRegNum<[48]>;
// ... PN1-PN15, DWARF 49-63

// P registers are sub-registers of PN registers
let SubRegIndices = [psub] in {
  def P0 : AArch64Reg<0, "p0", [PN0]>, DwarfRegAlias<PN0>;
  // ... P1-P15
}
```

- `P0–P15` are the SVE predicate registers. Each holds one bit per byte of the corresponding Z register — so for a 512-bit Z register, the predicate is 64 bits wide.
- `psub` is a sub-register index with size `-1` (scalable), since the predicate width scales with the vector length.
- `P0` is a sub-register of `PN0`. The `PN` (predicate-as-counter) registers are the same physical registers accessed in a different mode (SVE2 counting mode).
- DWARF numbers 48–63 for PN0–PN15; P registers alias to the same DWARF numbers.

---

### PN Registers — Predicates-as-Counters

```tablegen
def PN0 : AArch64Reg<0, "pn0">, DwarfRegNum<[48]>;
// ...

class PNRClass<int firstreg, int lastreg> : RegisterClass<
    "AArch64", [aarch64svcount], 16,
    (sequence "PN%u", firstreg, lastreg)> {
  let Size = 16;
}

def PNR        : PNRClass<0, 15>;
def PNR_3b     : PNRClass<0, 7>;   // Restricted to PN0-PN7
def PNR_p8to15 : PNRClass<8, 15>;  // Restricted to PN8-PN15
```

- `PN` registers are the same physical registers as `P` registers, but used as loop counters in SVE2 instructions like `WHILELT`.
- The value type `aarch64svcount` is a special LLVM type for predicate-as-counter values.
- `Size = 16` is the minimum size in bits (one bit per byte of a 128-bit minimum vector).

---

### SVE Register Classes: ZPR, PPR

```tablegen
// ZPR: SVE data vector register class
class ZPRClass<int firstreg, int lastreg, int step = 1>
    : RegisterClass<"AArch64",
                    [nxv16i8, nxv8i16, nxv4i32, nxv2i64,
                     nxv2f16, nxv4f16, nxv8f16,
                     nxv2bf16, nxv4bf16, nxv8bf16,
                     nxv2f32, nxv4f32, nxv2f64],
                    128, (sequence "Z%u", firstreg, lastreg, step)> {
  let Size = 128;  // Minimum size; actual size is vscale * 128 bits
}

def ZPR    : ZPRClass<0, 31>;        // All Z0-Z31
def ZPR_4b : ZPRClass<0, 15>;        // Z0-Z15 (4-bit encoding)
def ZPR_3b : ZPRClass<0, 7>;         // Z0-Z7  (3-bit encoding)
def ZPRMul2 : ZPRClass<0, 30, 2>;    // Even registers: Z0,Z2,...,Z30
def ZPRMul4 : ZPRClass<0, 28, 4>;    // Multiples of 4: Z0,Z4,...,Z28
```

The `nxv` prefix means "scalable vector" — `nxv4i32` is a vector of `4 * vscale` 32-bit integers. `Size = 128` is the minimum (when `vscale = 1`).

**Restricted classes** exist because some SVE instructions encode the register in fewer bits:
- `ZPR_4b` (Z0–Z15): used when the instruction has a 4-bit register field
- `ZPR_3b` (Z0–Z7): used when the instruction has a 3-bit register field

```tablegen
// PPR: SVE predicate register class
class PPRClass<int firstreg, int lastreg, int step = 1>
    : RegisterClass<"AArch64",
                    [nxv16i1, nxv8i1, nxv4i1, nxv2i1, nxv1i1],
                    16, (sequence "P%u", firstreg, lastreg, step)> {
  let Size = 16;
}

def PPR    : PPRClass<0, 15>;   // All P0-P15
def PPR_3b : PPRClass<0, 7>;    // P0-P7 (3-bit encoding)
def PPRMul2 : PPRClass<0, 14, 2>; // Even predicates: P0,P2,...,P14
```

The `nxv16i1` type means "scalable vector of 1-bit values, minimum 16 elements" — one bit per byte of the corresponding Z register.

---

### SVE Multi-register Tuples

SVE instructions can operate on groups of 2, 3, or 4 consecutive Z registers:

```tablegen
def ZSeqPairs   : RegisterTuples<[zsub0, zsub1],
                                 [(rotl ZPR, 0), (rotl ZPR, 1)]>;
def ZSeqTriples : RegisterTuples<[zsub0, zsub1, zsub2],
                                 [(rotl ZPR, 0), (rotl ZPR, 1), (rotl ZPR, 2)]>;
def ZSeqQuads   : RegisterTuples<[zsub0, zsub1, zsub2, zsub3],
                                 [(rotl ZPR, 0), ..., (rotl ZPR, 3)]>;

def ZPR2 : RegisterClass<"AArch64", [untyped], 128, (add ZSeqPairs)>  { let Size = 256; }
def ZPR3 : RegisterClass<"AArch64", [untyped], 128, (add ZSeqTriples)>{ let Size = 384; }
def ZPR4 : RegisterClass<"AArch64", [untyped], 128, (add ZSeqQuads)>  { let Size = 512; }
```

**SME2 alignment-constrained tuples:**
```tablegen
// Must start at a multiple of 2: Z0,Z2,Z4,...,Z30
def ZPR2Mul2 : RegisterClass<"AArch64", [untyped], 128,
                             (add (decimate ZSeqPairs, 2))> { let Size = 256; }

// Must start at a multiple of 4: Z0,Z4,Z8,...,Z28
def ZPR4Mul4 : RegisterClass<"AArch64", [untyped], 128,
                             (add (decimate ZSeqQuads, 4))> { let Size = 512; }
```
`decimate X, N` keeps every Nth element of the list. So `decimate ZSeqPairs, 2` keeps only the pairs starting at even-numbered registers.

---

### SVE Strided Register Groups

SME2 introduces strided register groups — registers that are not consecutive but spaced 8 apart:

```tablegen
// Strided pairs: {Z0,Z8}, {Z1,Z9}, ..., {Z7,Z15}  (low half)
//            and {Z16,Z24}, {Z17,Z25}, ..., {Z23,Z31} (high half)
def ZStridedPairsLo : RegisterTuples<[zsub0, zsub1], [
  (trunc (rotl ZPR, 0), 8),   // Z0-Z7
  (trunc (rotl ZPR, 8), 8)    // Z8-Z15
]>;
def ZStridedPairsHi : RegisterTuples<[zsub0, zsub1], [
  (trunc (rotl ZPR, 16), 8),  // Z16-Z23
  (trunc (rotl ZPR, 24), 8)   // Z24-Z31
]>;

// Strided quads: {Z0,Z4,Z8,Z12}, {Z1,Z5,Z9,Z13}, ...
def ZStridedQuadsLo : RegisterTuples<[zsub0, zsub1, zsub2, zsub3], [
  (trunc (rotl ZPR, 0),  4),  // Z0-Z3
  (trunc (rotl ZPR, 4),  4),  // Z4-Z7
  (trunc (rotl ZPR, 8),  4),  // Z8-Z11
  (trunc (rotl ZPR, 12), 4)   // Z12-Z15
]>;

def ZPR2Strided : RegisterClass<"AArch64", [untyped], 128,
                                (add ZStridedPairsLo, ZStridedPairsHi)> { let Size = 256; }
def ZPR4Strided : RegisterClass<"AArch64", [untyped], 128,
                                (add ZStridedQuadsLo, ZStridedQuadsHi)> { let Size = 512; }
```

These are used by SME2 outer-product and matrix multiply instructions that process multiple tiles simultaneously.

---

## Part 4 — SME Registers

### The ZA Tile Hierarchy

The SME ZA register is a 2D matrix of `SVL × SVL` bytes (where SVL = Streaming Vector Length). It is accessed as a hierarchy of tiles of different element sizes:

```
ZA (entire matrix, SVL×SVL bytes)
 └── ZAB0 (byte tile: 1 × SVL×SVL bits)
      ├── ZAH0 (half-word tile 0: half of ZAB0)
      │    ├── ZAS0 (single-word tile 0)
      │    │    ├── ZAD0 (double-word tile 0)
      │    │    │    ├── ZAQ0  (quad-word tile 0)
      │    │    │    ├── ZAQ1
      │    │    │    ...
      │    │    │    └── ZAQ7
      │    │    ├── ZAD1
      │    │    ...
      │    │    └── ZAD3
      │    └── ZAS1
      │         └── ZAD4..ZAD7
      └── ZAH1
           └── ZAS2, ZAS3
```

In TableGen, this is built bottom-up:

```tablegen
// Level 1: 16 quad-word tiles (128-bit each)
def ZAQ0  : AArch64Reg<0,  "za0.q">;
// ... ZAQ1-ZAQ15

// Level 2: 8 double-word tiles (256-bit each), each containing 2 quad tiles
let SubRegIndices = [zasubq0, zasubq1] in {
  def ZAD0 : AArch64Reg<0, "za0.d", [ZAQ0, ZAQ8]>;
  def ZAD1 : AArch64Reg<1, "za1.d", [ZAQ1, ZAQ9]>;
  // ... ZAD2-ZAD7
}

// Level 3: 4 single-word tiles (512-bit each), each containing 2 double tiles
let SubRegIndices = [zasubd0, zasubd1] in {
  def ZAS0 : AArch64Reg<0, "za0.s", [ZAD0, ZAD4]>;
  // ... ZAS1-ZAS3
}

// Level 4: 2 half-word tiles (1024-bit each), each containing 2 single tiles
let SubRegIndices = [zasubs0, zasubs1] in {
  def ZAH0 : AArch64Reg<0, "za0.h", [ZAS0, ZAS2]>;
  def ZAH1 : AArch64Reg<1, "za1.h", [ZAS1, ZAS3]>;
}

// Level 5: 1 byte tile (2048-bit), containing both half-word tiles
let SubRegIndices = [zasubh0, zasubh1] in {
  def ZAB0 : AArch64Reg<0, "za0.b", [ZAH0, ZAH1]>;
}

// Level 6: The entire ZA array, containing ZAB0
let SubRegIndices = [zasubb] in {
  def ZA : AArch64Reg<0, "za", [ZAB0]>;
}
```

**Why this structure?** An SME instruction like `FMOPA ZA0.S, P0/M, P1/M, Z0.S, Z1.S` operates on the `ZAS0` tile (32-bit element tile 0). The sub-register hierarchy lets LLVM know that writing `ZAS0` also writes part of `ZAB0` and all of `ZA`, enabling correct liveness analysis.

---

### ZA Sub-register Indices

```tablegen
def zasubb  : SubRegIndex<2048>;  // Byte tile:       16×16 bytes  = 2048 bits
def zasubh0 : SubRegIndex<1024>;  // Half-word tile 0: 16×16/2     = 1024 bits
def zasubh1 : SubRegIndex<1024>;  // Half-word tile 1: 16×16/2     = 1024 bits
def zasubs0 : SubRegIndex<512>;   // Single-word tile 0: 16×16/4   = 512 bits
def zasubs1 : SubRegIndex<512>;   // Single-word tile 1
def zasubd0 : SubRegIndex<256>;   // Double-word tile 0: 16×16/8   = 256 bits
def zasubd1 : SubRegIndex<256>;   // Double-word tile 1
def zasubq0 : SubRegIndex<128>;   // Quad-word tile 0: 16×16/16    = 128 bits
def zasubq1 : SubRegIndex<128>;   // Quad-word tile 1
```

These sizes are computed for the minimum SVL of 128 bits (16 bytes). At larger SVL values, the actual sizes scale accordingly.

---

### ZT0 — The Lookup Table Register

```tablegen
def ZT0 : AArch64Reg<0, "zt0">;

def ZTR : RegisterClass<"AArch64", [untyped], 512, (add ZT0)> {
  let Size = 512;
  let DiagnosticType = "InvalidLookupTable";
}
```

- `ZT0` is a 512-bit register introduced in SME2 for lookup table operations (`LUTI2`, `LUTI4`).
- It is a single register with no sub-registers.
- `ZTR` is its register class, used in instruction operand definitions.

---

### SME Register Classes: MPR, ZTR

```tablegen
let isAllocatable = 0 in {
  // The entire ZA array
  def MPR : RegisterClass<"AArch64", [untyped], 2048, (add ZA)> {
    let Size = 2048;
  }

  // Individual tile classes by element size
  def MPR8   : RegisterClass<"AArch64", [untyped], 2048, (add ZAB0)>       { let Size = 2048; }
  def MPR16  : RegisterClass<"AArch64", [untyped], 1024, (add ZAH0, ZAH1)> { let Size = 1024; }
  def MPR32  : RegisterClass<"AArch64", [untyped],  512, (add ZAS0..ZAS3)> { let Size = 512;  }
  def MPR64  : RegisterClass<"AArch64", [untyped],  256, (add ZAD0..ZAD7)> { let Size = 256;  }
  def MPR128 : RegisterClass<"AArch64", [untyped],  128, (add ZAQ0..ZAQ15)>{ let Size = 128;  }
}
```

**`isAllocatable = 0`** on all SME tile classes — the register allocator never assigns virtual registers to ZA tiles. ZA is managed explicitly by the SME ABI pass (`MachineSMEABIPass.cpp`).

**Index registers for ZA tile slices:**
```tablegen
// W8-W11: used as row/column index for ZA tile vector access
def MatrixIndexGPR32_8_11  : RegisterClass<"AArch64", [i32], 32,
                                           (sequence "W%u", 8, 11)>;
// W12-W15: alternative index registers
def MatrixIndexGPR32_12_15 : RegisterClass<"AArch64", [i32], 32,
                                           (sequence "W%u", 12, 15)>;
```
SME tile vector instructions like `LD1H {ZA0H.H[W8, 0]}, P0/Z, [X0]` use `W8–W11` or `W12–W15` as the slice index register.

---

### SME Operand Types

```tablegen
// Tile operands (e.g., za0.s, za1.d)
def TileOp16 : MatrixTileOperand<16, MPR16>;
def TileOp32 : MatrixTileOperand<32, MPR32>;
def TileOp64 : MatrixTileOperand<64, MPR64>;

// Tile vector operands — horizontal (H) and vertical (V)
def TileVectorOpH8   : MatrixTileVectorOperand<8,   MPR8,   0>;  // za0h.b
def TileVectorOpH16  : MatrixTileVectorOperand<16,  MPR16,  0>;  // za0h.h
def TileVectorOpV32  : MatrixTileVectorOperand<32,  MPR32,  1>;  // za0v.s
// ... etc.

// The entire ZA array
def MatrixOp : MatrixOperand<MPR, 0>;  // "za"

// SME2 typed ZA operands
def MatrixOp8  : MatrixOperand<MPR, 8>;   // "za" with byte elements
def MatrixOp16 : MatrixOperand<MPR, 16>;  // "za" with half-word elements
def MatrixOp32 : MatrixOperand<MPR, 32>;  // "za" with word elements
def MatrixOp64 : MatrixOperand<MPR, 64>;  // "za" with double-word elements

// Tile list (for SMSTART/SMSTOP)
def MatrixTileList : MatrixTileListOperand<>;  // e.g., {ZA0.D, ZA1.D}
```

---

## Sub-register Index Reference

| Index | Size (bits) | Offset (bits) | Used For |
|---|---|---|---|
| `sub_32` | 32 | 0 | W register inside X register |
| `sub_32_hi` | 32 | 32 | Upper 32 bits of X register |
| `bsub` | 8 | 0 | B inside H |
| `bsub_hi` | 8 | 8 | Upper 8 bits of H |
| `hsub` | 16 | 0 | H inside S |
| `hsub_hi` | 16 | 16 | Upper 16 bits of S |
| `ssub` | 32 | 0 | S inside D |
| `ssub_hi` | 32 | 32 | Upper 32 bits of D |
| `dsub` | 64 | 0 | D inside Q |
| `dsub_hi` | 64 | 64 | Upper 64 bits of Q |
| `zsub` | 128 | 0 | Q inside Z (bottom 128 bits) |
| `zsub_hi` | -1 | 128 | Upper scalable portion of Z |
| `psub` | -1 | 0 | P inside PN |
| `zsub0..zsub3` | -1 | — | Elements of ZPR2/3/4 tuples |
| `qsub0..qsub3` | 128 | — | Elements of QQ/QQQ/QQQQ tuples |
| `dsub0..dsub3` | 64 | — | Elements of DD/DDD/DDDD tuples |
| `psub0..psub1` | -1 | — | Elements of PPR2 pairs |
| `zasubb` | 2048 | — | ZAB0 inside ZA |
| `zasubh0/h1` | 1024 | — | ZAH tiles inside ZAB0 |
| `zasubs0/s1` | 512 | — | ZAS tiles inside ZAH |
| `zasubd0/d1` | 256 | — | ZAD tiles inside ZAS |
| `zasubq0/q1` | 128 | — | ZAQ tiles inside ZAD |

---

## Key Patterns to Recognise

**1. Shared hardware encoding, different contexts**
`SP` and `XZR` both encode as register `31`. The assembler and hardware use instruction context to distinguish them. LLVM models them as separate registers in separate register classes (`GPR64sp` vs `GPR64`).

**2. `_HI` artificial registers for liveness**
Every register that is a sub-register of a larger one has a corresponding `_HI` artificial register to model the upper bits. This is essential for correct liveness analysis — without it, LLVM would not know whether the upper bits of a register are live after a partial write.

**3. `isAllocatable = 0` for special registers**
`CCR` (NZCV), all SME tile classes (`MPR`, `MPR8`, etc.), and the dummy `_HI` classes are not allocatable. The register allocator will never assign a virtual register to them.

**4. `isConstant = true` for zero registers**
`XZR` and `WZR` have `isConstant = true`. This enables the `AArch64DeadRegisterDefinitionsPass` to replace dead register definitions with writes to `XZR`/`WZR`, saving the need to track liveness of the result.

**5. Size `-1` means scalable**
Any `SubRegIndex` or `RegisterClass` with `Size = -1` or a sub-register index of size `-1` represents a scalable quantity whose actual size is `vscale * minimum_size`. This is the foundation of SVE's scalable type system in LLVM.

**6. `DwarfRegAlias` collapses the hierarchy for debug info**
All of `B0`, `H0`, `S0`, `D0`, `Q0` share DWARF register number 64 via `DwarfRegAlias<B0>`. Debug information uses a single canonical register number per physical register, regardless of which view (width) is being used.
