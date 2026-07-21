# AArch64 — Addressing Modes & Calling Convention (AAPCS64)
> Source files: `AArch64ISelDAGToDAG.cpp`, `AArch64InstrFormats.td`,
> `AArch64CallingConvention.td`

---

## Table of Contents

- [Part 1 — Addressing Modes](#part-1--addressing-modes)
  - [What is an Addressing Mode?](#what-is-an-addressing-mode)
  - [Mode 1 — Base + Scaled Immediate](#mode-1--base--scaled-immediate)
  - [Mode 2 — Base + Unscaled Immediate](#mode-2--base--unscaled-immediate)
  - [Mode 3 — Base + Register Offset (WRO / XRO)](#mode-3--base--register-offset-wro--xro)
  - [Mode 4 — Pre-Index](#mode-4--pre-index)
  - [Mode 5 — Post-Index](#mode-5--post-index)
  - [All Five Modes Side by Side](#all-five-modes-side-by-side)
  - [How the Compiler Picks a Mode](#how-the-compiler-picks-a-mode)
  - [How This Maps to Code in AArch64ISelDAGToDAG.cpp](#how-this-maps-to-code-in-aarch64iseldagtodag-cpp)
- [Part 2 — AAPCS64 Calling Convention](#part-2--aapcs64-calling-convention)
  - [What is a Calling Convention?](#what-is-a-calling-convention)
  - [Rule 1 — Integer Arguments: X0–X7](#rule-1--integer-arguments-x0x7)
  - [Rule 2 — Small Integers Are Promoted](#rule-2--small-integers-are-promoted)
  - [Rule 3 — Float Arguments: S0–S7 / D0–D7](#rule-3--float-arguments-s0s7--d0d7)
  - [Rule 4 — More Than 8 Arguments Go on the Stack](#rule-4--more-than-8-arguments-go-on-the-stack)
  - [Rule 5 — Return Values](#rule-5--return-values)
  - [Rule 6 — Callee-Saved Registers](#rule-6--callee-saved-registers)
  - [Rule 7 — Large Struct Return via X8 (SRet)](#rule-7--large-struct-return-via-x8-sret)
  - [Rule 8 — SVE / NEON Vector Arguments](#rule-8--sve--neon-vector-arguments)
  - [The Full Call Sequence — One Worked Example](#the-full-call-sequence--one-worked-example)
  - [How TableGen Encodes These Rules](#how-tablegen-encodes-these-rules)
  - [Variants of the Calling Convention](#variants-of-the-calling-convention)
  - [Connection to Callee-Saved and Tail Calls](#connection-to-callee-saved-and-tail-calls)

---

## Part 1 — Addressing Modes

### What is an Addressing Mode?

When a CPU executes a load or store instruction, it needs to know **which memory
location to access**. The CPU cannot just say "go to memory" — it needs a
concrete **address**. An **addressing mode** is the method used to compute that
address from the operands of the instruction.

Think of it like giving someone directions to a house:

```
"Go to house number 42"                → fixed address (immediate)
"Go 3 doors from where you are"        → base register + constant offset
"Go to the door number stored in your  → base register + register offset
 notebook"
"Walk forward, then read the door"     → post-index
"Read the door, then walk forward"     → pre-index
```

AArch64 has five addressing modes. Each one is a different way to express
`address = f(registers, constants)`.

---

### Mode 1 — Base + Scaled Immediate

```
Address = Base_Register + (Immediate × Data_Size_In_Bytes)
```

This is the most common mode. You have a **base register** holding a starting
address, and you add a **constant offset** to it. The offset is encoded in the
instruction as a 12-bit unsigned integer.

```asm
LDR  X0, [X1, #8]      ; address = X1 + 8
STR  X2, [X3, #16]     ; address = X3 + 16
LDR  W0, [X1, #100]    ; address = X1 + 100
LDR  Q0, [X1, #32]     ; address = X1 + 32  (128-bit NEON load)
```

**What does "scaled" mean?**

The 12-bit field in the instruction encoding does not store the raw byte offset.
It stores the offset **divided by the data size**. The CPU multiplies it back
when computing the address.

```
For a 64-bit (8-byte) load:
  Encoded value = byte_offset / 8
  Byte range    = 0 to 4095 × 8 = 0 to 32760

For a 32-bit (4-byte) load:
  Encoded value = byte_offset / 4
  Byte range    = 0 to 4095 × 4 = 0 to 16380

For an 8-bit (1-byte) load:
  Encoded value = byte_offset / 1
  Byte range    = 0 to 4095
```

This is why the offset must be a **multiple of the data size** — if it is not,
the division is not exact and the mode cannot be used.

**In the code — `isValidAsScaledImmediate`:**

```cpp
// AArch64ISelDAGToDAG.cpp
static bool isValidAsScaledImmediate(int64_t Offset, unsigned Range,
                                     unsigned Size) {
  if ((Offset & (Size - 1)) == 0 &&   // must be aligned to data size
       Offset >= 0 &&                  // must be non-negative
       (uint64_t)Offset < Range * Size) // must fit in 12-bit field
    return true;
  return false;
}
```

`(Offset & (Size - 1)) == 0` checks alignment — for a 64-bit load (`Size=8`),
this checks that the offset is a multiple of 8.

**In TableGen — `am_indexed8`, `am_indexed32`, etc.:**

```tablegen
// AArch64InstrFormats.td
def am_indexed8   : ComplexPattern<iPTR, 2, "SelectAddrModeIndexed8",  []>;
def am_indexed16  : ComplexPattern<iPTR, 2, "SelectAddrModeIndexed16", []>;
def am_indexed32  : ComplexPattern<iPTR, 2, "SelectAddrModeIndexed32", []>;
def am_indexed64  : ComplexPattern<iPTR, 2, "SelectAddrModeIndexed64", []>;
def am_indexed128 : ComplexPattern<iPTR, 2, "SelectAddrModeIndexed128",[]>;
```

Each `ComplexPattern` is a TableGen hook that calls the corresponding C++
function in `AArch64ISelDAGToDAG.cpp`. The number `2` means it produces two
output operands: `Base` and `OffImm`.

---

### Mode 2 — Base + Unscaled Immediate

```
Address = Base_Register + Immediate (raw bytes, no scaling)
```

Similar to Mode 1, but:
- The offset is **not scaled** — it is the exact byte count
- The offset can be **negative** (range: −256 to +255)
- Uses different instruction mnemonics: `LDUR` / `STUR` (the `U` = Unscaled)

```asm
LDUR  X0, [X1, #-8]    ; address = X1 - 8   (negative offset)
STUR  X2, [X3, #-16]   ; address = X3 - 16
LDUR  W0, [X1, #3]     ; address = X1 + 3   (not a multiple of 4 — can't use LDR)
```

**When is this used instead of Mode 1?**

Two situations:
1. The offset is **negative** — Mode 1 only handles non-negative offsets
2. The offset is **not aligned** to the data size — e.g., loading a 32-bit value
   at byte offset 3 (which is not a multiple of 4)

**In the code — `SelectAddrModeUnscaled`:**

```cpp
// AArch64ISelDAGToDAG.cpp
bool AArch64DAGToDAGISel::SelectAddrModeUnscaled(SDValue N, unsigned Size,
                                                  SDValue &Base,
                                                  SDValue &OffImm) {
  // Only matches ADD nodes (base + constant)
  if (N.getOpcode() != ISD::ADD) return false;

  if (ConstantSDNode *RHS = dyn_cast<ConstantSDNode>(N.getOperand(1))) {
    int64_t RHSC = RHS->getSExtValue();
    if (RHSC >= -256 && RHSC < 256) {   // 9-bit signed range
      Base   = N.getOperand(0);
      OffImm = CurDAG->getTargetConstant(RHSC, dl, MVT::i64);
      return true;
    }
  }
  return false;
}
```

The range `[-256, 255]` is a 9-bit signed integer — exactly what the `LDUR`
instruction encoding provides.

**Key insight — Mode 1 is tried first:**

```cpp
// AArch64ISelDAGToDAG.cpp — inside SelectAddrModeIndexed
// Before falling back to our general case, check if the unscaled
// instructions can handle this. If so, that's preferable.
if (SelectAddrModeUnscaled(N, Size, Base, OffImm))
    return false;   // let the unscaled path handle it
```

The compiler tries Mode 1 (scaled) first. If the offset doesn't fit Mode 1 but
fits Mode 2 (unscaled), it uses Mode 2. This avoids materializing the address
in a separate register.

---

### Mode 3 — Base + Register Offset (WRO / XRO)

```
Address = Base_Register + Offset_Register (optionally shifted or extended)
```

Instead of a constant, the offset is **another register**. This is used when
the offset is computed at runtime — for example, indexing into an array with a
variable index.

```asm
LDR  X0, [X1, X2]              ; address = X1 + X2
LDR  X0, [X1, X2, LSL #3]      ; address = X1 + (X2 << 3) = X1 + X2*8
LDR  X0, [X1, W2, UXTW]        ; address = X1 + zero_extend(W2)
LDR  X0, [X1, W2, SXTW #2]     ; address = X1 + sign_extend(W2) << 2
```

There are two variants based on the width of the offset register:

**WRO — W Register Offset:**
The offset is a 32-bit `W` register. It must be extended to 64-bit before
adding. The extension can be:
- `UXTW` — zero-extend (treat as unsigned)
- `SXTW` — sign-extend (treat as signed, so negative indices work)

**XRO — X Register Offset:**
The offset is a 64-bit `X` register. It can optionally be shifted left by
`0`, `1`, `2`, or `3` bits (i.e., multiplied by 1, 2, 4, or 8).

**A concrete C example:**

```c
int arr[100];
int load_element(int *arr, int index) {
    return arr[index];
}
```

`index` is a variable — its value is not known at compile time. The compiler
cannot use a constant offset. It uses Mode 3:

```asm
; arr is in X0, index is in W1
LDR  W0, [X0, W1, UXTW #2]
;         ^^  ^^  ^^^^  ^^
;         arr idx  zero  scale by 4 (int = 4 bytes)
;                  extend
```

`UXTW #2` means: zero-extend `W1` to 64-bit, then shift left by 2 (multiply
by 4, since `int` is 4 bytes). This computes `arr + index * 4` in one
instruction.

**In the code — `SelectAddrModeWRO` and `SelectAddrModeXRO`:**

```cpp
// AArch64ISelDAGToDAG.cpp
bool AArch64DAGToDAGISel::SelectAddrModeWRO(SDValue N, unsigned Size,
                                             SDValue &Base, SDValue &Offset,
                                             SDValue &SignExtend,
                                             SDValue &DoShift) {
  // Matches: base + (w_reg extended to 64-bit, optionally shifted)
  // Produces 4 operands: Base, Offset, SignExtend flag, DoShift flag
}

bool AArch64DAGToDAGISel::SelectAddrModeXRO(SDValue N, unsigned Size,
                                             SDValue &Base, SDValue &Offset,
                                             SDValue &SignExtend,
                                             SDValue &DoShift) {
  // Matches: base + (x_reg, optionally shifted by log2(Size))
  // Produces 4 operands: Base, Offset, SignExtend flag, DoShift flag
}
```

Notice these produce **4 operands** (not 2 like the immediate modes). The extra
two are flags: `SignExtend` (SXTW vs UXTW) and `DoShift` (whether to apply the
scale shift).

**In TableGen:**

```tablegen
// AArch64InstrFormats.td
def ro_Windexed32 : ComplexPattern<iPTR, 4, "SelectAddrModeWRO<32>", []>;
def ro_Xindexed32 : ComplexPattern<iPTR, 4, "SelectAddrModeXRO<32>", []>;
//                                      ^
//                                      4 output operands
```

---

### Mode 4 — Pre-Index

```
Address  = Base_Register + Offset
Base_Register is UPDATED to the new address BEFORE the memory access
```

```asm
LDR  X0, [X1, #8]!     ; 1. compute address = X1 + 8
                        ; 2. X1 = X1 + 8   ← update BEFORE load
                        ; 3. load from X1 (which is now X1_old + 8)
```

The `!` suffix means **writeback** — the base register is updated. The update
happens **before** the memory access.

**When is this useful?**

Walking through a data structure where you always want the base register to
point to the current element:

```c
// Advancing a pointer and reading the new location
*++ptr = value;   // increment first, then write — this is pre-index
```

```asm
STR  X0, [X1, #8]!     ; X1 += 8, then store X0 at new X1
```

**In the prologue — saving register pairs:**

Pre-index is used heavily in function prologues to save callee-saved registers
while simultaneously allocating stack space:

```asm
STP  X29, X30, [SP, #-16]!   ; SP -= 16, then store FP and LR at new SP
;                              ; this both allocates stack space AND saves regs
;                              ; in one instruction
```

---

### Mode 5 — Post-Index

```
Address  = Base_Register (original value, BEFORE any update)
Base_Register is UPDATED AFTER the memory access
```

```asm
LDR  X0, [X1], #8      ; 1. load from X1 (original value)
                        ; 2. X1 = X1 + 8   ← update AFTER load
```

No `!` here — the offset comes **after** the closing `]`. The memory access
uses the original base register value, then the register advances.

**When is this useful?**

Iterating through an array — read the current element, then advance the pointer
to the next one:

```c
while (n--) {
    process(*ptr);
    ptr++;             // advance after reading — this is post-index
}
```

```asm
loop:
    LDR  W2, [X0], #4  ; load *ptr (W2 = *X0), then X0 += 4
    BL   process
    SUBS X1, X1, #1
    B.NE loop
```

**In the epilogue — restoring register pairs:**

Post-index is used in function epilogues to restore callee-saved registers
while simultaneously deallocating stack space:

```asm
LDP  X29, X30, [SP], #16   ; load FP and LR from SP, then SP += 16
;                            ; this both restores regs AND deallocates stack
;                            ; in one instruction
```

---

### All Five Modes Side by Side

```
Mode              Assembly Example          Address Used       Base Updated?
──────────────────────────────────────────────────────────────────────────────
Base + Scaled     LDR X0, [X1, #8]          X1 + 8             No
Immediate

Base + Unscaled   LDUR X0, [X1, #-8]        X1 - 8             No
Immediate

Base + Register   LDR X0, [X1, X2, LSL #3]  X1 + X2*8          No

Pre-Index         LDR X0, [X1, #8]!         X1 + 8             Yes, BEFORE
                                                                X1 = X1 + 8

Post-Index        LDR X0, [X1], #8          X1 (original)      Yes, AFTER
                                                                X1 = X1 + 8
```

```
Offset range summary:
  Scaled immediate:   0 to 4095 × data_size  (must be aligned, non-negative)
  Unscaled immediate: -256 to +255           (any byte offset, can be negative)
  Register offset:    full 64-bit range      (runtime computed)
```

---

### How the Compiler Picks a Mode

The compiler sees LLVM IR like this:

```llvm
%val = load i64, ptr %ptr
```

It does not say which addressing mode to use. The instruction selector in
`AArch64ISelDAGToDAG.cpp` tries each mode in order and picks the first one
that fits:

```
Step 1: Is the offset a constant that fits in 12-bit scaled form?
        → use LDR [base, #imm]          (Mode 1 — SelectAddrModeIndexed)

Step 2: Is the offset a constant in [-256, +255]?
        → use LDUR [base, #imm]         (Mode 2 — SelectAddrModeUnscaled)

Step 3: Is the offset in a W register (with optional sign/zero extend)?
        → use LDR [base, Wreg, UXTW]    (Mode 3 — SelectAddrModeWRO)

Step 4: Is the offset in an X register (with optional shift)?
        → use LDR [base, Xreg, LSL #n]  (Mode 3 — SelectAddrModeXRO)

Step 5: Nothing fits
        → materialize the full address in a register first
        → use LDR [Xreg]
```

Pre-index and post-index (Modes 4 and 5) are matched separately by
`tryIndexedLoad()` — the compiler looks for patterns like
`load(ptr + offset); ptr = ptr + offset` and folds them into a single
indexed instruction.

---

### How This Maps to Code in AArch64ISelDAGToDAG.cpp

Every addressing mode has a corresponding C++ selector function and a
TableGen `ComplexPattern` that hooks them together:

```
TableGen ComplexPattern          C++ Selector Function
─────────────────────────────────────────────────────────────────────
am_indexed8                  →   SelectAddrModeIndexed8(N, Base, OffImm)
am_indexed16                 →   SelectAddrModeIndexed16(N, Base, OffImm)
am_indexed32                 →   SelectAddrModeIndexed32(N, Base, OffImm)
am_indexed64                 →   SelectAddrModeIndexed64(N, Base, OffImm)
am_indexed128                →   SelectAddrModeIndexed128(N, Base, OffImm)

am_unscaled8                 →   SelectAddrModeUnscaled8(N, Base, OffImm)
am_unscaled16                →   SelectAddrModeUnscaled16(N, Base, OffImm)
am_unscaled32                →   SelectAddrModeUnscaled32(N, Base, OffImm)
am_unscaled64                →   SelectAddrModeUnscaled64(N, Base, OffImm)
am_unscaled128               →   SelectAddrModeUnscaled128(N, Base, OffImm)

ro_Windexed8                 →   SelectAddrModeWRO<8>(N, Base, Off, SE, DS)
ro_Windexed32                →   SelectAddrModeWRO<32>(N, Base, Off, SE, DS)
ro_Windexed64                →   SelectAddrModeWRO<64>(N, Base, Off, SE, DS)

ro_Xindexed8                 →   SelectAddrModeXRO<8>(N, Base, Off, SE, DS)
ro_Xindexed32                →   SelectAddrModeXRO<32>(N, Base, Off, SE, DS)
ro_Xindexed64                →   SelectAddrModeXRO<64>(N, Base, Off, SE, DS)
```

The number suffix (8, 16, 32, 64, 128) is the **data size in bits**. The same
selector logic is reused for all sizes — only the scale factor changes.

---

## Part 2 — AAPCS64 Calling Convention

### What is a Calling Convention?

When function `foo` calls function `bar`, they need to agree on a set of rules:

1. **Where do arguments go?** — which registers, or the stack
2. **Where does the return value go?** — which register
3. **Which registers does `bar` promise not to destroy?** — callee-saved
4. **Who cleans up the stack?** — caller or callee

Without this agreement, `foo` might put an argument in `X0` but `bar` reads it
from `X1` — the program produces garbage. The **calling convention** is the
contract that both sides follow.

**AAPCS64** = **A**rchitecture **P**rocedure **C**all **S**tandard for
**64**-bit AArch64. It is the standard contract used on Linux, Android, and
most non-Apple platforms.

In LLVM, these rules are written in `AArch64CallingConvention.td` as a list of
`CCIfType` / `CCAssignToReg` / `CCAssignToStack` rules that are evaluated
top-to-bottom for each argument.

---

### Rule 1 — Integer Arguments: X0–X7

```tablegen
// AArch64CallingConvention.td
CCIfType<[i64], CCAssignToReg<[X0, X1, X2, X3, X4, X5, X6, X7]>>,
CCIfType<[i32], CCAssignToReg<[W0, W1, W2, W3, W4, W5, W6, W7]>>,
```

The first 8 integer arguments go into registers, left to right.

```c
void foo(int a, int b, int c, int d,
         int e, int f, int g, int h);
//           W0     W1     W2     W3
//           W4     W5     W6     W7
```

```asm
; calling foo(1, 2, 3, 4, 5, 6, 7, 8):
MOV  W0, #1
MOV  W1, #2
MOV  W2, #3
MOV  W3, #4
MOV  W4, #5
MOV  W5, #6
MOV  W6, #7
MOV  W7, #8
BL   foo
```

---

### Rule 2 — Small Integers Are Promoted

```tablegen
CCIfType<[i1, i8, i16], CCPromoteToType<i32>>,
```

`bool` (i1), `char` (i8), and `short` (i16) are too small for a 32-bit
register slot but they still occupy a full register. They are
**zero-extended to 32-bit** before being placed in `W0`–`W7`.

```c
void foo(char a, short b, bool c);
//            W0       W1      W2
//       (promoted  (promoted  (promoted
//        to i32)    to i32)    to i32)
```

This means the callee can safely read `W0` as a 32-bit value — the upper bits
are guaranteed to be zero.

---

### Rule 3 — Float Arguments: S0–S7 / D0–D7

```tablegen
CCIfType<[f16],  CCAssignToReg<[H0, H1, H2, H3, H4, H5, H6, H7]>>,
CCIfType<[f32],  CCAssignToReg<[S0, S1, S2, S3, S4, S5, S6, S7]>>,
CCIfType<[f64],  CCAssignToReg<[D0, D1, D2, D3, D4, D5, D6, D7]>>,
CCIfType<[f128, v2i64, v4i32, v8i16, v16i8, v4f32, v2f64, v8f16, v8bf16],
         CCAssignToReg<[Q0, Q1, Q2, Q3, Q4, Q5, Q6, Q7]>>,
```

Floating-point and NEON vector arguments use the FP/NEON registers —
**completely separate** from the integer registers. This means a function can
have up to 8 integer arguments AND 8 float arguments all in registers at the
same time.

```c
void foo(int a, float b, int c, double d, int e, float f);
//           X0      S0      X1       D1      X2      S2
```

Notice:
- `a` → `X0` (first integer slot)
- `b` → `S0` (first float slot — NOT X1)
- `c` → `X1` (second integer slot)
- `d` → `D1` (second float slot — NOT X2)
- `e` → `X2` (third integer slot)
- `f` → `S2` (third float slot)

Integer and float register slots are counted **independently**. Using a float
argument does not consume an integer register slot, and vice versa.

---

### Rule 4 — More Than 8 Arguments Go on the Stack

```tablegen
CCIfType<[i64, f64, ...],          CCAssignToStack<8,  8>>,
CCIfType<[f128, v2i64, v4i32, ...], CCAssignToStack<16, 16>>,
```

The 9th integer argument (and beyond) goes onto the stack. The caller allocates
the space and the callee reads from `[SP + offset]`.

```c
void foo(int a, int b, int c, int d,
         int e, int f, int g, int h,
         int i,   // 9th  → [SP + 0]
         int j);  // 10th → [SP + 8]
//           X0  X1  X2  X3  X4  X5  X6  X7
```

```asm
; caller sets up the 9th and 10th arguments on the stack:
MOV  W0, #1
; ... W1-W7 for args 2-8 ...
MOV  W9, #9
STR  W9, [SP, #0]    ; 9th argument on stack
MOV  W9, #10
STR  W9, [SP, #8]    ; 10th argument on stack
BL   foo
```

The `CCAssignToStack<8, 8>` means: allocate 8 bytes on the stack, aligned to
8 bytes.

---

### Rule 5 — Return Values

```tablegen
// RetCC_AArch64_AAPCS
CCIfType<[i32], CCAssignToReg<[W0, W1, W2, W3, W4, W5, W6, W7]>>,
CCIfType<[i64], CCAssignToReg<[X0, X1, X2, X3, X4, X5, X6, X7]>>,
CCIfType<[f32], CCAssignToReg<[S0, S1, S2, S3, S4, S5, S6, S7]>>,
CCIfType<[f64], CCAssignToReg<[D0, D1, D2, D3, D4, D5, D6, D7]>>,
```

A single return value goes in `X0` (integer) or `S0`/`D0` (float). Multiple
return values (e.g., returning a struct that fits in registers) can use
`X0`–`X7`.

```c
int   foo() { return 42;   }   // result in W0
float bar() { return 3.14; }   // result in S0
long  baz() { return 100L; }   // result in X0
```

---

### Rule 6 — Callee-Saved Registers

```tablegen
// AArch64CallingConvention.td
def CSR_AArch64_AAPCS : CalleeSavedRegs<(add
    X19, X20, X21, X22, X23, X24, X25, X26, X27, X28,
    LR, FP,
    D8, D9, D10, D11, D12, D13, D14, D15)>;
```

These are the registers that a function **must preserve**. If `bar` wants to
use `X19`, it must save it to the stack in the prologue and restore it in the
epilogue.

```
Callee-saved (bar MUST preserve):   X19–X28, FP (X29), LR (X30), D8–D15
Caller-saved (bar CAN destroy):     X0–X18, D0–D7, D16–D31
```

```asm
bar:
    ; prologue — save callee-saved registers we want to use
    STP  X19, X20, [SP, #-16]!   ; save X19 and X20

    MOV  X19, X0                  ; now safe to use X19
    ; ... do work ...

    ; epilogue — restore before returning
    LDP  X19, X20, [SP], #16     ; restore X19 and X20
    RET
```

**Why does this list matter for LLVM?**

The `CSR_AArch64_AAPCS` definition is used by:
- `AArch64RegisterInfo::getCalleeSavedRegs()` — tells the register allocator
  which registers need save/restore
- `AArch64FrameLowering::spillCalleeSavedRegisters()` — generates the actual
  STP/LDP instructions in the prologue/epilogue
- `tcGPR64` register class — excludes these registers from tail call targets
  (as discussed in the register file notes)

---

### Rule 7 — Large Struct Return via X8 (SRet)

```tablegen
CCIfSRet<CCIfType<[i64], CCAssignToReg<[X8]>>>,
```

When a function returns a large struct that does not fit in registers, the
**caller** allocates space for it and passes a **pointer to that space** in
`X8`. The callee writes the result there. This is called **Struct Return**
(SRet).

```c
struct Big { int a, b, c, d, e; };   // 20 bytes — too big for registers

struct Big make_big() {
    return (struct Big){1, 2, 3, 4, 5};
}

// Caller does:
struct Big result;                    // allocate space
make_big_with_hidden_ptr(&result);    // X8 = &result (hidden argument)
// callee writes into *X8
```

```asm
; caller:
ADD  X8, SP, #offset_of_result   ; X8 = pointer to result storage
BL   make_big                     ; callee writes to *X8

; callee (make_big):
MOV  W9, #1
STR  W9, [X8, #0]    ; result.a = 1
MOV  W9, #2
STR  W9, [X8, #4]    ; result.b = 2
; ... etc
RET
```

---

### Rule 8 — SVE / NEON Vector Arguments

```tablegen
// SVE scalable vectors → Z0–Z7
CCIfType<[nxv16i8, nxv8i16, nxv4i32, nxv2i64, ...],
         CCAssignToReg<[Z0, Z1, Z2, Z3, Z4, Z5, Z6, Z7]>>,

// SVE predicates → P0–P3
CCIfType<[nxv16i1, nxv8i1, nxv4i1, nxv2i1, nxv1i1, aarch64svcount],
         CCAssignToReg<[P0, P1, P2, P3]>>,

// NEON 128-bit vectors → Q0–Q7
CCIfType<[v2i64, v4i32, v8i16, v16i8, v4f32, v2f64, v8f16, v8bf16],
         CCAssignToReg<[Q0, Q1, Q2, Q3, Q4, Q5, Q6, Q7]>>,
```

SVE vector arguments use `Z0–Z7` (up to 8 scalable vectors in registers).
SVE predicate arguments use `P0–P3` (up to 4 predicates in registers). NEON
128-bit vectors use `Q0–Q7`.

If there are more SVE vectors than fit in `Z0–Z7`, they are passed indirectly:

```tablegen
// If no Z register is available, pass a pointer instead
CCIfType<[nxv16i8, ...], CCPassIndirect<i64>>,
```

---

### The Full Call Sequence — One Worked Example

```c
float compute(int a, float b, int c, float d) {
    return a + b + c + d;
}

// calling it:
float result = compute(10, 3.14f, 20, 2.71f);
```

**Step 1 — Caller assigns arguments:**

```
Argument   Type    Register   Rule
─────────────────────────────────────────────────────
a = 10     i32     W0         Rule 1: first integer slot
b = 3.14   f32     S0         Rule 3: first float slot
c = 20     i32     W1         Rule 1: second integer slot (W0 already used)
d = 2.71   f32     S1         Rule 3: second float slot (S0 already used)
```

**Step 2 — Caller emits setup code:**

```asm
MOV   W0, #10          ; a
FMOV  S0, #3.14        ; b
MOV   W1, #20          ; c
FMOV  S1, #2.71        ; d
BL    compute
; result comes back in S0
```

**Step 3 — Callee reads arguments:**

```asm
compute:
    ; a is in W0, b is in S0, c is in W1, d is in S1
    SCVTF  S2, W0        ; convert a (int) to float
    FADD   S2, S2, S0    ; S2 = a + b
    SCVTF  S3, W1        ; convert c (int) to float
    FADD   S2, S2, S3    ; S2 = a + b + c
    FADD   S0, S2, S1    ; S0 = a + b + c + d  (return value goes in S0)
    RET
```

**Step 4 — Caller reads return value:**

```asm
; after BL compute returns:
; S0 = result (float)
FSTR  S0, [SP, #result_offset]   ; store result
```

---

### How TableGen Encodes These Rules

The calling convention is a **list of rules evaluated top to bottom**. Each
rule has a condition and an action:

```tablegen
// AArch64CallingConvention.td — simplified view of CC_AArch64_AAPCS

// Rule: if type is i1/i8/i16, promote to i32 first
CCIfType<[i1, i8, i16], CCPromoteToType<i32>>,

// Rule: if type is i32, try to assign to W0, W1, ..., W7 in order
CCIfType<[i32], CCAssignToReg<[W0, W1, W2, W3, W4, W5, W6, W7]>>,

// Rule: if type is i64, try to assign to X0, X1, ..., X7 in order
CCIfType<[i64], CCAssignToReg<[X0, X1, X2, X3, X4, X5, X6, X7]>>,

// Rule: if type is f32, try to assign to S0, S1, ..., S7 in order
CCIfType<[f32], CCAssignToReg<[S0, S1, S2, S3, S4, S5, S6, S7]>>,

// Rule: if no register is available, put it on the stack
CCIfType<[i32, f32], CCAssignToStack<8, 8>>,
CCIfType<[i64, f64, ...], CCAssignToStack<8, 8>>,
```

LLVM evaluates these rules for each argument in order. When a rule matches and
a register is available, that register is assigned and marked as used. When all
registers for a type are exhausted, the stack rule fires.

There are two separate convention definitions:
- `CC_AArch64_AAPCS` — used when the function **receives** arguments
  (`LowerFormalArguments`)
- `RetCC_AArch64_AAPCS` — used when the function **returns** a value
  (`LowerReturn`) and when the caller **reads** the return value (`LowerCall`)

---

### Variants of the Calling Convention

The file defines several variants for different platforms and use cases:

```
CC_AArch64_AAPCS          Standard Linux/Android ABI
CC_AArch64_DarwinPCS      Apple's variant (stack slots sized as needed,
                           i128 doesn't need even registers)
CC_AArch64_Win64PCS       Windows ABI
CC_AArch64_GHC            Glasgow Haskell Compiler (uses X19–X28 for STG
                           machine registers — callee-saved become args!)
CC_AArch64_Preserve_None  Experimental: use ALL registers for arguments,
                           nothing is preserved
CC_AArch64_Arm64EC_Thunk  Windows Arm64EC x64-compatible thunk ABI
```

The callee-saved register lists also have variants:

```tablegen
CSR_AArch64_AAPCS          Standard: X19-X28, LR, FP, D8-D15
CSR_AArch64_AAVPCS         Vector PCS: adds Q8-Q23 (full NEON preservation)
CSR_AArch64_SVE_AAPCS      SVE: adds Z8-Z23, P4-P15
CSR_Darwin_AArch64_AAPCS   Darwin: same registers, different stack layout
                            (LR, FP at top of callee-save area)
CSR_AArch64_NoRegs         No registers saved (used for special stubs)
```

---

### Connection to Callee-Saved and Tail Calls

Everything in this document connects back to the tail call register classes
discussed in the register file notes:

```
AAPCS64 callee-saved:   X19–X28, FP, LR, D8–D15
                                ↓
These are the registers the epilogue RESTORES before returning
                                ↓
A tail call jump happens AFTER the epilogue
                                ↓
Any value stored in X19–X28/FP/LR is OVERWRITTEN by the epilogue
                                ↓
The jump target register must survive the epilogue
                                ↓
Only X0–X18 survive (caller-saved — epilogue never touches them)
                                ↓
tcGPR64 = GPR64 minus {X19–X28, FP, LR}
        = exactly {X0–X18}
```

The calling convention definition (`CSR_AArch64_AAPCS`) and the tail call
register class (`tcGPR64`) are two sides of the same coin — one defines what
must be preserved, the other defines what is safe to use after preservation
has happened.
