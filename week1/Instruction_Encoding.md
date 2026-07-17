# AArch64 Instruction Encoding — Fixed 32-bit Width
> Source file: `AArch64InstrFormats.td`

---

## Table of Contents

- [What Does "Fixed 32-bit Width" Mean?](#what-does-fixed-32-bit-width-mean)
- [Width vs Address — What is the Difference?](#width-vs-address--what-is-the-difference)
- [What Lives Inside the 32 Bits?](#what-lives-inside-the-32-bits)
- [The Root Class — AArch64Inst](#the-root-class--aarch64inst)
- [Pseudo vs Real Instructions](#pseudo-vs-real-instructions)
- [How a Bit Field Becomes an Instruction](#how-a-bit-field-becomes-an-instruction)
- [Encoding Groups — Reading the Bit Layout](#encoding-groups--reading-the-bit-layout)
  - [Group 1 — Data Processing (Register)](#group-1--data-processing-register)
  - [Group 2 — Data Processing (Immediate)](#group-2--data-processing-immediate)
  - [Group 3 — Branches](#group-3--branches)
  - [Group 4 — Load / Store](#group-4--load--store)
- [Operand Encoding — Immediates](#operand-encoding--immediates)
  - [Unsigned Immediates](#unsigned-immediates)
  - [Signed Immediates](#signed-immediates)
  - [Scaled Immediates](#scaled-immediates)
  - [Logical Immediates — The Special Case](#logical-immediates--the-special-case)
  - [Floating-Point Immediates](#floating-point-immediates)
- [Operand Encoding — Shifts and Extends](#operand-encoding--shifts-and-extends)
  - [Arithmetic Shifted Register](#arithmetic-shifted-register)
  - [Logical Shifted Register](#logical-shifted-register)
  - [Extended Register](#extended-register)
- [Operand Encoding — Branches and PC-Relative Labels](#operand-encoding--branches-and-pc-relative-labels)
- [TSFlags — Per-Instruction Metadata](#tsflags--per-instruction-metadata)
- [The Unpredictable Field](#the-unpredictable-field)
- [ComplexPattern — Bridging TableGen and C++](#complexpattern--bridging-tablegen-and-c)
- [How Everything Connects — End to End](#how-everything-connects--end-to-end)

---

## What Does "Fixed 32-bit Width" Mean?

Every AArch64 instruction is **exactly 32 bits wide** — no exceptions. This is
fundamentally different from x86, where instructions can be 1 to 15 bytes long.

```
AArch64 instruction stream:
  [31:0]  [31:0]  [31:0]  [31:0]
  ──────  ──────  ──────  ──────
  4 bytes 4 bytes 4 bytes 4 bytes

x86 instruction stream:
  [8-bit] [8-bit][16-bit] [8-bit][8-bit][32-bit]
  ──────  ──────────────  ──────────────────────
  1 byte  2 bytes         6 bytes
```

**Why does this matter for a compiler?**

1. **Instruction fetch is simple** — the CPU always fetches exactly 4 bytes.
   No variable-length decoding needed.
2. **Every operand must fit in the 32 bits** — there is no room for a full
   64-bit immediate. Large constants must be loaded separately.
3. **Immediates are encoded in specific bit fields** — the compiler must know
   exactly which bits hold which operand, and whether the value fits.
4. **The disassembler is trivial** — read 4 bytes, decode fixed fields.

In LLVM, this 32-bit constraint is expressed in `AArch64InstrFormats.td` as:

```tablegen
// AArch64InstrFormats.td
class AArch64Inst<Format f, string cstr> : Instruction {
  field bits<32> Inst;   // ← the 32-bit encoding
  // ...
}
```

Every instruction in the AArch64 backend ultimately inherits from this class
and fills in specific bits of `Inst`.

---

## Width vs Address — What is the Difference?

These are two completely separate concepts that are easy to confuse.

```
Memory address    Content
──────────────    ──────────────────────────────────────
0x1000            [byte][byte][byte][byte]   ← instruction 1
0x1004            [byte][byte][byte][byte]   ← instruction 2
0x1008            [byte][byte][byte][byte]   ← instruction 3
0x100C            [byte][byte][byte][byte]   ← instruction 4
```

- **Address** (`0x1000`, `0x1004`, ...) — where in memory the instruction
  lives. This is just a location.
- **Width** (4 bytes = 32 bits) — how many bytes the instruction itself
  occupies. This is the size of the binary data.

Because every AArch64 instruction is exactly 4 bytes wide, addresses always
advance by exactly 4. That is why you see `0x1000`, `0x1004`, `0x1008` — never
`0x1001` or `0x1003`.

Compare with x86 where instructions have variable width:

```
Memory address    Content                      Width
──────────────    ───────────────────────────  ─────
0x1000            [byte]                       1 byte
0x1001            [byte][byte]                 2 bytes
0x1003            [byte][byte][byte][byte]      4 bytes
0x1007            [byte][byte][byte]            3 bytes
```

On x86 the CPU cannot know where the next instruction starts until it has
fully decoded the current one. On AArch64 it always knows: next instruction
is at `current_address + 4`.

---

## What Lives Inside the 32 Bits?

The entire instruction — **opcode + all operands** — is packed into those
4 bytes. Every piece of information the CPU needs to execute the instruction
must fit inside.

```
ADD  X0, X1, X2

 31      29 28    21 20    16 15    10 9      5 4      0
┌──────────┬────────┬────────┬────────┬────────┬────────┐
│  sf + op │ fixed  │   Rm   │  shift │   Rn   │   Rd   │
│ (opcode) │ (group)│  (X2)  │  (#0)  │  (X1)  │  (X0)  │
└──────────┴────────┴────────┴────────┴────────┴────────┘
  3 bits     8 bits   5 bits   6 bits   5 bits   5 bits
```

Breaking it down:

```
Bit  [31]     = 1          → 64-bit operation (X registers, not W)
Bit  [29]     = 0          → don't set flags (ADD not ADDS)
Bits [28:21]  = 11010110   → "data processing register" group identifier
Bits [20:16]  = 00010      → Rm = X2 (register number 2)
Bits [15:10]  = 000000     → shift amount = 0, shift type = LSL
Bits [9:5]    = 00001      → Rn = X1 (register number 1)
Bits [4:0]    = 00000      → Rd = X0 (register number 0)
```

The **opcode** is not a single field — it is spread across several fixed bit
ranges that together identify the instruction. The **operands** (register
numbers, immediates, shift amounts) fill the remaining bits.

**The constraint this creates — large constants cannot fit:**

Because everything must share 32 bits, there is no room for a full 64-bit
constant inside a single instruction.

```asm
; IMPOSSIBLE in one instruction:
MOV  X0, #0x1234567890ABCDEF   ; 64-bit constant — does not fit in 32 bits

; AArch64 splits it across four instructions:
MOVZ  X0, #0xCDEF              ; load bits [15:0]
MOVK  X0, #0x90AB, LSL #16     ; insert bits [31:16]
MOVK  X0, #0x5678, LSL #32     ; insert bits [47:32]
MOVK  X0, #0x1234, LSL #48     ; insert bits [63:48]
```

Each `MOVZ`/`MOVK` carries only 16 bits of the constant — that is all that
fits alongside the opcode and register fields in 32 bits. This is a direct
consequence of the fixed-width encoding.

**Summary:**

```
32 bits = opcode bits + register numbers + immediate value
          (what to do)   (which registers)  (constant, if any)

All three must share the same 32 bits.
The instruction format defines exactly which bits belong to which part.
```

---

## The Root Class — AArch64Inst

```tablegen
// AArch64InstrFormats.td
class AArch64Inst<Format f, string cstr> : Instruction {
  field bits<32> Inst;          // The 32-bit instruction encoding
  field bits<32> Unpredictable = 0; // Bits that make the instruction UNPREDICTABLE
  field bits<32> SoftFail = Unpredictable; // Alias for the disassembler

  let Namespace = "AArch64";
  Format F   = f;
  bits<2> Form = F.Value;

  // Per-instruction metadata flags (packed into TSFlags)
  bit isWhile = 0;
  bit isPTestLike = 0;
  FalseLanesEnum FalseLanes       = FalseLanesNone;
  DestructiveInstTypeEnum DestructiveInstType = NotDestructive;
  SMEMatrixTypeEnum SMEMatrixType = SMEMatrixNone;
  ElementSizeEnum ElementSize     = ElementSizeNone;

  // TSFlags bit layout:
  let TSFlags{13-11} = SMEMatrixType.Value;
  let TSFlags{10}    = isPTestLike;
  let TSFlags{9}     = isWhile;
  let TSFlags{8-7}   = FalseLanes.Value;
  let TSFlags{6-3}   = DestructiveInstType.Value;
  let TSFlags{2-0}   = ElementSize.Value;

  let Pattern     = [];
  let Constraints = cstr;
}
```

**Key fields explained:**

| Field | What it is |
|---|---|
| `bits<32> Inst` | The actual 32-bit binary encoding of the instruction |
| `bits<32> Unpredictable` | Bits that, if set differently from `Inst`, make the instruction UNPREDICTABLE (hardware may do anything) |
| `TSFlags` | Target-specific flags packed into a single integer — used by passes to query instruction properties without string comparisons |
| `Constraints` | Register constraints, e.g. `"$Rd = $Rn"` for tied operands |

---

## Pseudo vs Real Instructions

There are two kinds of instructions in the backend:

```tablegen
// Pseudo: no encoding, only exists during compilation
class Pseudo<dag oops, dag iops, list<dag> pattern, string cstr = "">
    : AArch64Inst<PseudoFrm, cstr> {
  let isCodeGenOnly = 1;  // never emitted as machine code
  let isPseudo      = 1;
}

// Real: has a 32-bit encoding, gets emitted as machine code
class EncodedI<string cstr, list<dag> pattern> : AArch64Inst<NormalFrm, cstr> {
  let Size = 4;  // always 4 bytes
}
```

**Pseudo instructions** are placeholders used during compilation that get
expanded into real instructions before the final binary is written. Examples:

```tablegen
// A tail call — no real encoding, expanded by AArch64FrameLowering
def TCRETURNri : Pseudo<(outs), (ins tcGPR64:$dst, i32imm:$FPDiff), []>;

// A function prologue placeholder
def ADJCALLSTACKDOWN : Pseudo<(outs), (ins i32imm:$amt1, i32imm:$amt2), []>;
```

**Real instructions** have every bit of `Inst` defined. The MC layer reads
these bit assignments to emit the final binary.

---

## How a Bit Field Becomes an Instruction

Let's trace a concrete example — the `ADD` instruction (register form):

```asm
ADD  X0, X1, X2     ; X0 = X1 + X2
```

In the AArch64 manual, this encodes as:

```
31 30 29 28    24 23 22 21 20    16 15    10 9     5 4     0
 1  0  0  01011  00  0  Rm[4:0]  000000   Rn[4:0]  Rd[4:0]
```

In TableGen, this is built up through a class hierarchy:

```tablegen
// Step 1: The base two-operand register class
class BaseTwoOperandRegReg<bit sf, bit S, bits<6> opc, ...>
  : I<...> {
  bits<5> Rd;
  bits<5> Rn;
  bits<5> Rm;

  let Inst{31}    = sf;        // 1 = 64-bit, 0 = 32-bit
  let Inst{30}    = 0b0;
  let Inst{29}    = S;         // 1 = set flags (ADDS), 0 = don't
  let Inst{28-21} = 0b11010110; // fixed opcode group bits
  let Inst{20-16} = Rm;        // 5-bit source register 2
  let Inst{15-10} = opc;       // 6-bit operation code
  let Inst{9-5}   = Rn;        // 5-bit source register 1
  let Inst{4-0}   = Rd;        // 5-bit destination register
}
```

When `ADD X0, X1, X2` is assembled:
- `sf = 1` (64-bit operation)
- `S = 0` (don't set flags)
- `Rd = 0` (X0 = register 0)
- `Rn = 1` (X1 = register 1)
- `Rm = 2` (X2 = register 2)
- `opc = 0b000000` (shift amount = 0, shift type = LSL)

Result:
```
Bit 31:    1         (sf = 64-bit)
Bit 30:    0
Bit 29:    0         (S = no flags)
Bits 28-21: 11010110 (fixed)
Bits 20-16: 00010    (Rm = X2 = 2)
Bits 15-10: 000000   (opc = LSL #0)
Bits 9-5:   00001    (Rn = X1 = 1)
Bits 4-0:   00000    (Rd = X0 = 0)

Binary: 1000 1010 0000 0010 0000 0000 0010 0000
Hex:    0x8A020020
```

---

## Encoding Groups — Reading the Bit Layout

AArch64 instructions are divided into groups based on bits `[28:25]`. This is
the first thing the hardware decoder looks at.

```
Bits [28:25]   Group
────────────   ──────────────────────────────────
  00xx         Reserved / Data Processing (misc)
  100x         Data Processing — Immediate
  101x         Branches, Exception, System
  x1x0         Loads and Stores
  x101         Data Processing — Register
  x111         Data Processing — SIMD/FP
```

### Group 1 — Data Processing (Register)

These instructions operate on registers only — no memory access, no immediates
(except shift amounts).

```tablegen
// Shift instruction — LSL, LSR, ASR, ROR
class BaseShift<bit size, bits<2> shift_type, RegisterClass regtype, ...>
  : BaseTwoOperandRegReg<size, 0b0, {0,0,1,0,?,?}, regtype, ...> {
  let Inst{11-10} = shift_type;  // 00=LSL, 01=LSR, 10=ASR, 11=ROR
}

// Multiply-accumulate — MADD, MSUB
class BaseMulAccum<bit isSub, bits<3> opc, ...> : I<...> {
  bits<5> Rd;
  bits<5> Rn;
  bits<5> Rm;
  bits<5> Ra;  // accumulate register
  let Inst{30-24} = 0b0011011;
  let Inst{23-21} = opc;
  let Inst{20-16} = Rm;
  let Inst{15}    = isSub;  // 0=MADD (add), 1=MSUB (subtract)
  let Inst{14-10} = Ra;
  let Inst{9-5}   = Rn;
  let Inst{4-0}   = Rd;
}
```

### Group 2 — Data Processing (Immediate)

These instructions have a register source and a constant value baked into the
instruction bits.

```tablegen
// ADD/SUB with immediate — the immediate is 12 bits, optionally shifted by 12
class addsub_shifted_imm<ValueType Ty>
    : Operand<Ty>, ComplexPattern<Ty, 2, "SelectArithImmed", [imm]> {
  let MIOperandInfo = (ops i32imm, i32imm);  // value + shift amount
  // Encoding: {imm12, shift}
  // shift = 0  → immediate is used as-is
  // shift = 12 → immediate is shifted left by 12 bits
}
```

Example:
```asm
ADD  X0, X1, #4096    ; 4096 = 0x1000 = 1 << 12
                       ; encoded as: imm12=1, shift=12
ADD  X0, X1, #1       ; encoded as: imm12=1, shift=0
```

### Group 3 — Branches

Branch instructions encode a **PC-relative offset** — the distance from the
current instruction to the target, in units of 4 bytes (since all instructions
are 4 bytes).

```tablegen
// Unconditional branch — 26-bit signed offset
class BImm<bit op, dag iops, string asm, list<dag> pattern> : I<...> {
  bits<26> addr;
  let Inst{31}    = op;       // 0=B, 1=BL
  let Inst{30-26} = 0b00101;  // fixed
  let Inst{25-0}  = addr;     // 26-bit PC-relative offset
}
```

The 26-bit field encodes `(target - PC) / 4`. Since instructions are always
4-byte aligned, the bottom 2 bits of the offset are always zero and are not
stored. This gives a range of ±128 MB.

```tablegen
// Conditional branch — 19-bit signed offset
class BranchCond<bit bit4, string mnemonic> : I<...> {
  bits<4>  cond;    // condition code (EQ, NE, LT, GT, etc.)
  bits<19> target;  // 19-bit PC-relative offset → ±1 MB range
  let Inst{31-24} = 0b01010100;
  let Inst{23-5}  = target;
  let Inst{4}     = bit4;
  let Inst{3-0}   = cond;
}
```

```tablegen
// Test-and-branch — 14-bit offset, plus a bit number to test
class BaseTestBranch<...> : I<...> {
  bits<5>  Rt;       // register to test
  bits<6>  bit_off;  // which bit to test (0-63)
  bits<14> target;   // 14-bit PC-relative offset → ±32 KB range
  let Inst{30-25} = 0b011011;
  let Inst{24}    = op;           // 0=TBZ, 1=TBNZ
  let Inst{23-19} = bit_off{4-0}; // lower 5 bits of bit number
  let Inst{18-5}  = target;
  let Inst{4-0}   = Rt;
}
```

### Group 4 — Load / Store

Load/store instructions encode the addressing mode in specific bit fields.

```tablegen
// Authenticated load (v8.3 PAC)
class BaseAuthLoad<bit M, bit W, ...> : I<...> {
  bits<10> offset;  // 10-bit signed scaled offset
  bits<5>  Rn;      // base register
  bits<5>  Rt;      // target register
  let Inst{31-24} = 0b11111000;
  let Inst{23}    = M;           // modifier (key A or B)
  let Inst{22}    = offset{9};   // sign bit of offset
  let Inst{21}    = 1;
  let Inst{20-12} = offset{8-0}; // lower 9 bits of offset
  let Inst{11}    = W;           // writeback
  let Inst{10}    = 1;
  let Inst{9-5}   = Rn;
  let Inst{4-0}   = Rt;
}
```

---

## Operand Encoding — Immediates

### Unsigned Immediates

```tablegen
// uimm6: unsigned 6-bit immediate, range [0, 63]
def uimm6 : Operand<i64>, ImmLeaf<i64, [{ return Imm >= 0 && Imm < 64; }]> {
  let ParserMatchClass = UImm6Operand;
}

// uimm8: unsigned 8-bit immediate, range [0, 255]
def uimm8_32b : Operand<i32>, ImmLeaf<i32, [{ return Imm >= 0 && Imm < 256; }]>;

// uimm16: unsigned 16-bit immediate, range [0, 65535]
def uimm16 : Operand<i16>, ImmLeaf<i16, [{return Imm >= 0 && Imm < 65536;}]>;
```

`ImmLeaf` is a TableGen class that defines both:
1. A **predicate** (the `[{ ... }]` block) — checked at instruction selection
   time to verify the constant fits
2. An **operand type** — used to match the right instruction variant

### Signed Immediates

```tablegen
// simm9: signed 9-bit immediate, range [-256, 255]
// Used for LDUR/STUR unscaled offset
def simm9 : Operand<i64>, ImmLeaf<i64, [{ return Imm >= -256 && Imm < 256; }]> {
  let DecoderMethod = "DecodeSImm<9>";
}

// simm7s4: signed 7-bit immediate, scaled by 4
// Range: [-256, 252] in steps of 4
// Used for LDP/STP (load/store pair)
def simm7s4 : Operand<i32> {
  let ParserMatchClass = SImm7s4Operand;
  let PrintMethod = "printImmScale<4>";
}
```

### Scaled Immediates

Many immediates are **scaled** — the value stored in the instruction bits is
the actual byte offset divided by the data size. This allows a small bit field
to cover a large byte range.

```tablegen
// The scale transform: divide by N before encoding, multiply by N when decoding
def UImmS2XForm : SDNodeXForm<imm, [{
  return CurDAG->getTargetConstant(N->getZExtValue() / 2, SDLoc(N), MVT::i64);
}]>;
def UImmS4XForm : SDNodeXForm<imm, [{
  return CurDAG->getTargetConstant(N->getZExtValue() / 4, SDLoc(N), MVT::i64);
}]>;
def UImmS8XForm : SDNodeXForm<imm, [{
  return CurDAG->getTargetConstant(N->getZExtValue() / 8, SDLoc(N), MVT::i64);
}]>;
```

`SDNodeXForm` is a TableGen transformation applied to an immediate value
**before** it is encoded into the instruction. The decoder applies the inverse
(multiply) when reading the binary back.

Example — `uimm5s4` (5-bit unsigned, scaled by 4):

```tablegen
def uimm5s4 : Operand<i64>, ImmLeaf<i64,
    [{ return Imm >= 0 && Imm < (32*4) && ((Imm % 4) == 0); }],
    UImmS4XForm> {
  let PrintMethod = "printImmScale<4>";
}
```

- Predicate: value must be in `[0, 128)` and a multiple of 4
- Transform: divide by 4 before encoding → stored as 5-bit value `[0, 31]`
- Print: multiply by 4 when printing → shows the original byte offset

### Logical Immediates — The Special Case

Logical instructions (`AND`, `ORR`, `EOR`, `TST`) use a special encoding for
their immediates. Not every 32-bit or 64-bit value can be encoded — only values
that consist of a **repeating bit pattern**.

```tablegen
// AArch64InstrFormats.td
def logical_imm32 : Operand<i32>, IntImmLeaf<i32, [{
  return AArch64_AM::isLogicalImmediate(Imm.getZExtValue(), 32);
}], logical_imm32_XFORM> {
  let PrintMethod = "printLogicalImm<int32_t>";
}

def logical_imm64 : Operand<i64>, IntImmLeaf<i64, [{
  return AArch64_AM::isLogicalImmediate(Imm.getZExtValue(), 64);
}], logical_imm64_XFORM> {
  let PrintMethod = "printLogicalImm<int64_t>";
}
```

The encoding transform:

```tablegen
def logical_imm32_XFORM : SDNodeXForm<imm, [{
  uint64_t enc = AArch64_AM::encodeLogicalImmediate(N->getZExtValue(), 32);
  return CurDAG->getTargetConstant(enc, SDLoc(N), MVT::i32);
}]>;
```

`AArch64_AM::encodeLogicalImmediate` converts the actual bit pattern into a
13-bit encoded form `{N, immr, imms}`. The hardware decodes this back to the
full 32/64-bit pattern.

**Why this restriction?** The 32-bit instruction only has 13 bits for the
immediate field. The encoding scheme allows representing useful patterns like
`0xFF00FF00`, `0xAAAAAAAA`, `0x0000FFFF` — all values that are useful for
masking operations.

### Floating-Point Immediates

```tablegen
// 8-bit FP immediate — encodes a limited set of float values
def fpimm32 : Operand<f32>,
    FPImmLeaf<f32, [{
      return AArch64_AM::getFP32Imm(Imm) != -1;
    }], fpimm32XForm> {
  let PrintMethod = "printFPImmOperand";
}

def fpimm32XForm : SDNodeXForm<fpimm, [{
  uint32_t Enc = AArch64_AM::getFP32Imm(N->getValueAPF());
  return CurDAG->getTargetConstant(Enc, SDLoc(N), MVT::i32);
}]>;
```

Only 256 specific float values can be encoded as an 8-bit immediate in FMOV.
The format is `±1.mantissa × 2^exponent` where mantissa is 4 bits and exponent
is 3 bits. Values like `1.0`, `2.0`, `0.5`, `-1.0` are encodable; arbitrary
floats like `3.14` are not and must be loaded from a constant pool.

---

## Operand Encoding — Shifts and Extends

### Arithmetic Shifted Register

```tablegen
// Encoding: {shift_type[1:0], shift_amount[5:0]} = 8 bits total
// shift_type: 00=LSL, 01=LSR, 10=ASR
class arith_shift<ValueType Ty, int width> : Operand<Ty> {
  let PrintMethod = "printShifter";
}

// A register + shift amount, used as a single operand
class arith_shifted_reg<ValueType Ty, RegisterClass regclass, int width>
    : Operand<Ty>,
      ComplexPattern<Ty, 2, "SelectArithShiftedRegister", []> {
  let MIOperandInfo = (ops regclass, !cast<Operand>("arith_shift" # width));
}
```

Used in instructions like:
```asm
ADD  X0, X1, X2, LSL #3    ; X0 = X1 + (X2 << 3)
SUB  X0, X1, X2, ASR #1    ; X0 = X1 - (X2 >> 1)  (arithmetic shift)
```

The shift type and amount are packed into a single 8-bit field in the
instruction encoding.

### Logical Shifted Register

```tablegen
// Encoding: {shift_type[1:0], shift_amount[5:0]} = 8 bits
// shift_type: 00=LSL, 01=LSR, 10=ASR, 11=ROR  (ROR allowed for logical ops)
class logical_shift<int width> : Operand<i32> {
  let PrintMethod = "printShifter";
}

class logical_shifted_reg<ValueType Ty, RegisterClass regclass, Operand shiftop>
    : Operand<Ty>,
      ComplexPattern<Ty, 2, "SelectLogicalShiftedRegister", []> {
  let MIOperandInfo = (ops regclass, shiftop);
}
```

Used in:
```asm
AND  X0, X1, X2, ROR #4    ; X0 = X1 & rotate_right(X2, 4)
ORR  X0, X1, X2, LSL #8    ; X0 = X1 | (X2 << 8)
```

### Extended Register

```tablegen
// Encoding: {extend_type[2:0], shift_amount[2:0]} = 6 bits
// extend_type: UXTB=000, UXTH=001, UXTW=010, UXTX=011
//              SXTB=100, SXTH=101, SXTW=110, SXTX=111
def arith_extend : Operand<i32> {
  let PrintMethod = "printArithExtend";
}

class arith_extended_reg32<ValueType Ty> : Operand<Ty>,
    ComplexPattern<Ty, 2, "SelectArithExtendedRegister", []> {
  let MIOperandInfo = (ops GPR32, arith_extend);
}
```

Used in:
```asm
ADD  X0, X1, W2, UXTW #2   ; X0 = X1 + zero_extend(W2) << 2
ADD  X0, SP, X1, LSL #3    ; X0 = SP + X1 * 8  (pointer arithmetic)
```

The extend type tells the hardware how to widen the source register before
adding. This is essential for pointer arithmetic where an array index (32-bit)
needs to be added to a 64-bit base address.

---

## Operand Encoding — Branches and PC-Relative Labels

```tablegen
// 26-bit branch target (B / BL)
def am_b_target : Operand<OtherVT> {
  let EncoderMethod = "getBranchTargetOpValue";
  let DecoderMethod = "DecodeUnconditionalBranch";
  let PrintMethod   = "printAlignedLabel";
  let OperandType   = "OPERAND_PCREL";
}

// 19-bit conditional branch target (B.cond, CBZ, CBNZ)
def am_brcond : Operand<OtherVT> {
  let EncoderMethod = "getCondBranchTargetOpValue";
  let DecoderMethod = "DecodePCRelLabel19";
  let PrintMethod   = "printAlignedLabel";
  let OperandType   = "OPERAND_PCREL";
}

// 14-bit test-and-branch target (TBZ, TBNZ)
def am_tbrcond : Operand<OtherVT> {
  let EncoderMethod = "getTestBranchTargetOpValue";
  let PrintMethod   = "printAlignedLabel";
  let OperandType   = "OPERAND_PCREL";
}
```

All branch targets are `OPERAND_PCREL` — they are encoded as a **signed
offset from the current PC**, divided by 4 (since instructions are 4-byte
aligned). The assembler computes `(target_address - current_PC) / 4` and
stores that value in the instruction bits.

```
Branch type    Bit field   Range (bytes)
────────────   ─────────   ─────────────
B / BL         26-bit      ±128 MB
B.cond         19-bit      ±1 MB
CBZ / CBNZ     19-bit      ±1 MB
TBZ / TBNZ     14-bit      ±32 KB
```

---

## TSFlags — Per-Instruction Metadata

`TSFlags` is a 64-bit integer attached to every `MachineInstr`. The AArch64
backend packs several properties into the low 14 bits:

```tablegen
// AArch64InstrFormats.td
let TSFlags{13-11} = SMEMatrixType.Value;     // which SME matrix type
let TSFlags{10}    = isPTestLike;             // SVE predicate test
let TSFlags{9}     = isWhile;                 // SVE while loop instruction
let TSFlags{8-7}   = FalseLanes.Value;        // SVE false lane behaviour
let TSFlags{6-3}   = DestructiveInstType.Value; // SVE destructive operand
let TSFlags{2-0}   = ElementSize.Value;       // element size (B/H/S/D/Q)
```

**Why TSFlags?**

Passes that process instructions need to query properties quickly. Instead of
doing string comparisons on instruction names, they read a bit field:

```cpp
// Example: checking if an instruction is a SVE "while" instruction
bool isWhileOpcode(unsigned Opcode) {
  const MCInstrDesc &Desc = TII->get(Opcode);
  return (Desc.TSFlags >> AArch64::IsWhileShift) & 1;
}
```

**The DestructiveInstType field** is particularly important for SVE. SVE
instructions are often "destructive" — the destination register is also one
of the source registers. The `MOVPRFX` instruction can be inserted before a
destructive instruction to break this dependency. The `DestructiveInstType`
field tells the `AArch64MovePrefixPass` which instructions need this treatment:

```tablegen
def NotDestructive               : DestructiveInstTypeEnum<0>;
def DestructiveOther             : DestructiveInstTypeEnum<1>;
def DestructiveUnary             : DestructiveInstTypeEnum<2>;
def DestructiveBinaryImm         : DestructiveInstTypeEnum<3>;
def DestructiveBinary            : DestructiveInstTypeEnum<5>;
def DestructiveBinaryComm        : DestructiveInstTypeEnum<6>;
def DestructiveBinaryCommWithRev : DestructiveInstTypeEnum<7>;
```

---

## The Unpredictable Field

```tablegen
// AArch64InstrFormats.td
field bits<32> Unpredictable = 0;
field bits<32> SoftFail = Unpredictable;
```

Some instruction encodings have **reserved bits** — bits that the architecture
says must be zero (or one), but if they are set differently, the hardware
behaviour is **unpredictable** (the CPU may do anything, including crash).

```tablegen
// Example: SMULH — the Ra field must be 0b11111 but is ignored
class MulHi<bits<3> opc, string asm, SDNode OpNode> : I<...> {
  let Inst{14-10} = 0b11111;
  let Unpredictable{14-10} = 0b11111;  // if these bits differ, UNPREDICTABLE
}
```

The `SoftFail` alias is used by the disassembler — if it encounters an
instruction where the `Unpredictable` bits are set, it marks the instruction
as a soft failure (decoded but flagged as potentially wrong) rather than a
hard failure (completely unrecognised).

---

## ComplexPattern — Bridging TableGen and C++

Some operand matching is too complex to express in TableGen's pattern language.
`ComplexPattern` is the escape hatch — it calls a C++ function to do the
matching.

```tablegen
// TableGen side: declare the pattern
def am_indexed32 : ComplexPattern<iPTR, 2, "SelectAddrModeIndexed32", []>;
//                                ^^^^  ^   ^^^^^^^^^^^^^^^^^^^^^^^^
//                                type  num  C++ function name
//                                      outputs
```

```cpp
// C++ side: implement the matching
bool AArch64DAGToDAGISel::SelectAddrModeIndexed32(SDValue N,
                                                   SDValue &Base,
                                                   SDValue &OffImm) {
  return SelectAddrModeIndexed(N, 4, Base, OffImm);
  //                              ^
  //                              Size = 4 bytes (32-bit)
}
```

The `2` in `ComplexPattern<iPTR, 2, ...>` means the pattern produces **2
output operands** — `Base` and `OffImm`. These become separate operands in the
`MachineInstr`.

Similarly for shifted registers:

```tablegen
def arith_shifted_reg32 : arith_shifted_reg<i32, GPR32, 32>;
// ComplexPattern<i32, 2, "SelectArithShiftedRegister", []>
// Produces 2 outputs: the register and the shift operand
```

---

## How Everything Connects — End to End

Here is the complete journey from C source code to a 32-bit instruction binary:

```
C source:
  int foo(int *arr, int i) { return arr[i]; }

         │
         ▼ Clang
LLVM IR:
  %ptr = getelementptr i32, ptr %arr, i64 %i
  %val = load i32, ptr %ptr

         │
         ▼ SelectionDAG (AArch64ISelDAGToDAG.cpp)

SelectAddrModeWRO<32> matches the address:
  Base   = %arr  (X0)
  Offset = %i    (W1)
  SignExtend = 0 (UXTW)
  DoShift    = 1 (scale by 4)

Selects instruction:
  LDR W0, [X0, W1, UXTW #2]

         │
         ▼ MachineInstr created

MachineInstr: LDRWroW
  Operands: W0 (def), X0 (base), W1 (offset), 0 (SignExtend), 1 (DoShift)

         │
         ▼ MC Layer (AArch64MCCodeEmitter.cpp)

Reads bit assignments from AArch64InstrFormats.td:
  Inst{31-30} = 0b10        (size = 32-bit)
  Inst{29-27} = 0b111       (load/store register)
  Inst{26}    = 0b0         (not SIMD)
  Inst{25-24} = 0b10        (register offset)
  Inst{23-22} = 0b01        (load, unsigned)
  Inst{21}    = 0b1         (register offset mode)
  Inst{20-16} = 0b00001     (W1 = register 1)
  Inst{15-13} = 0b010       (UXTW)
  Inst{12}    = 0b1         (shift = #2)
  Inst{11-10} = 0b10        (fixed)
  Inst{9-5}   = 0b00000     (X0 = register 0)
  Inst{4-0}   = 0b00000     (W0 = register 0)

         │
         ▼ Binary output

0xB8610800   (4 bytes, little-endian in the .o file)
```

Every step is driven by the bit field assignments in `AArch64InstrFormats.td`.
The C++ code in the instruction selector, MC emitter, and disassembler all read
from the same TableGen-generated tables — there is one source of truth for the
encoding.
