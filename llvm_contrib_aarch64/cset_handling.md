# Contributing to `AArch64ConditionOptimizer.cpp`

## Table of Contents

1. [What this file is](#1-what-this-file-is)
2. [The problem it solves](#2-the-problem-it-solves)
3. [How the optimization works](#3-how-the-optimization-works)
   - [The core idea](#31-the-core-idea)
   - [The two transformation cases](#32-the-two-transformation-cases)
4. [Where in the compiler this runs](#4-where-in-the-compiler-this-runs)
5. [Key data structures](#5-key-data-structures)
6. [Key helper functions — what each one does](#6-key-helper-functions--what-each-one-does)
7. [The two optimization modes](#7-the-two-optimization-modes)
   - [Intra-block](#71-intra-block-optimizeintrablock)
   - [Cross-block](#72-cross-block-optimizecrossblock)
8. [Safety invariants the pass enforces](#8-safety-invariants-the-pass-enforces)
9. [What is missing — the two TODOs](#9-what-is-missing--the-two-todos)
   - [TODO 1 — CSET and other conditional instructions in cross-block](#91-todo-1--cset-and-other-conditional-instructions-in-cross-block)
   - [TODO 2 — Allow second block to be anything if no adjustment needed](#92-todo-2--allow-second-block-to-be-anything-if-no-adjustment-needed)
10. [Background knowledge you need](#10-background-knowledge-you-need)
    - [LLVM Machine IR](#101-llvm-machine-ir)
    - [AArch64 flag-setting instructions](#102-aarch64-flag-setting-instructions)
    - [AArch64 conditional instructions](#103-aarch64-conditional-instructions)
    - [Condition codes](#104-condition-codes)
    - [The dominator tree](#105-the-dominator-tree)
11. [Files and headers to read first](#11-files-and-headers-to-read-first)
12. [How to build and test](#12-how-to-build-and-test)
13. [Implementation plan for TODO 1](#13-implementation-plan-for-todo-1)
14. [Implementation plan for TODO 2](#14-implementation-plan-for-todo-2)
15. [Writing tests](#15-writing-tests)
16. [Pitfalls and edge cases](#16-pitfalls-and-edge-cases)

---

## 1. What this file is

`AArch64ConditionOptimizer.cpp` is an LLVM **machine function pass** that
lives in the AArch64 backend. It runs after instruction selection, directly
on `MachineInstr` objects — the low-level representation of assembly
instructions inside the compiler.

Its single purpose is to rewrite pairs of nearby comparison instructions so
they use **identical operands**, enabling the later CSE (Common Subexpression
Elimination) pass to delete the duplicate.

---

## 2. The problem it solves

Consider this C code:

```c
if ((a < 5 && ...) || (a > 5 && ...)) { ... }
```

Both sub-expressions compare `a` against `5`. Ideally the compiler would
emit one `CMP` and reuse the flags. But due to canonicalization in
`SelectionDAGBuilder`, `DAGCombine`, and target-specific lowering, the
generated assembly ends up as:

```asm
    cmp   w8, #4
    b.gt  .LBB0_3       ; branch if w8 > 4  (i.e. w8 >= 5)
    ...
.LBB0_3:
    cmp   w8, #6
    b.lt  .LBB0_6       ; branch if w8 < 6  (i.e. w8 <= 5)
```

The two `CMP` instructions compare against different immediates (`#4` and
`#6`) so CSE cannot remove either one. However, the two comparisons are
mathematically equivalent to comparing against `#5` with adjusted condition
codes:

```
w8 > 4   ≡   w8 >= 5      (GT #4  →  GE #5)
w8 < 6   ≡   w8 <= 5      (LT #6  →  LE #5)
```

After this pass:

```asm
    cmp   w8, #5
    b.ge  .LBB0_3
    ...
.LBB0_3:
    cmp   w8, #5        ; ← CSE removes this
    b.le  .LBB0_6
```

---

## 3. How the optimization works

### 3.1 The core idea

A `CMP` on AArch64 is an alias for `SUBS` or `ADDS` with a dead destination
register. It sets the `NZCV` flags register. A conditional instruction
(`Bcc`, `CSET`, `CSEL`, etc.) then reads those flags.

The pass looks for two `CMP + conditional` pairs that:
- Compare the **same register**
- Use **adjustable** condition codes (GT, GE, LT, LE, HI, HS, LO, LS)
- Have immediates that differ by exactly **1** or **2**

When those conditions hold, one or both immediates can be nudged by ±1 and
the condition code flipped (GT ↔ GE, LT ↔ LE) to make both `CMP`
instructions identical. CSE then removes the duplicate.

### 3.2 The two transformation cases

**Case A — Opposite directions, immediates differ by 2**

```
cmp w8, #4   b.gt   →   cmp w8, #5   b.ge
cmp w8, #6   b.lt   →   cmp w8, #5   b.le
```

Both are adjusted to meet in the middle. The condition codes flip from
exclusive to inclusive.

**Case B — Same direction, immediates differ by 1**

```
cmp w8, #10   b.gt   →   cmp w8, #10   b.gt   (unchanged)
cmp w8, #9    b.gt   →   cmp w8, #10   b.ge   (adjusted)
```

Only one side is adjusted. The adjusted side shifts its immediate to match
the other and flips its condition code.

---

## 4. Where in the compiler this runs

```
C source
   ↓  clang / Sema / CodeGen
LLVM IR  (llvm::Function, llvm::Instruction)
   ↓  SelectionDAG / GlobalISel
Machine IR  (MachineFunction, MachineInstr)   ← THIS PASS RUNS HERE
   ↓  Register allocation
   ↓  Prologue/epilogue insertion
   ↓  AsmPrinter
Assembly output
```

The pass is registered as `"aarch64-condopt"` and runs as part of the
AArch64 pre-regalloc optimization pipeline. It requires the
`MachineDominatorTree` analysis.

---

## 5. Key data structures

### `CmpInfo`

```cpp
struct CmpInfo {
  int Imm;           // The new immediate value after adjustment
  unsigned Opc;      // The new opcode (may change SUBS ↔ ADDS for CMN)
  AArch64CC::CondCode CC;  // The new condition code after adjustment
};
```

Returned by `getAdjustedCmpInfo()`. Describes what a `CMP` instruction
should look like after the ±1 adjustment.

### `CmpCondPair`

```cpp
struct CmpCondPair {
  MachineInstr *CmpMI;       // The CMP instruction
  MachineInstr *CondMI;      // The conditional instruction consuming it
  AArch64CC::CondCode CC;    // The condition code in use

  int getImm() const;        // Shortcut: CmpMI->getOperand(2).getImm()
  unsigned getOpc() const;   // Shortcut: CmpMI->getOpcode()
};
```

The fundamental unit the pass works with. One `CmpCondPair` represents a
single `CMP` instruction and the one conditional instruction that reads its
flags.

---

## 6. Key helper functions — what each one does

| Function | Purpose |
|---|---|
| `isCmpInstruction(Opc)` | Returns true for `SUBSWri`, `SUBSXri`, `ADDSWri`, `ADDSXri` — the four opcodes that implement `CMP`/`CMN` |
| `canAdjustCmp(CmpMI)` | Checks the CMP is safe to modify: immediate is a literal (not symbolic), immediate is in range, and the destination register is dead |
| `registersMatch(A, B)` | Checks both CMPs compare the same register, tracing through copy instructions |
| `nzcvLivesOut(MBB)` | Returns true if any successor block has `NZCV` in its live-in set — if so, modifying the CMP would corrupt the successor's flag use |
| `getBccTerminator(MBB)` | Returns the `Bcc` terminator of a block, or `nullptr` if the block does not end with a conditional branch |
| `findAdjustableCmp(CondMI)` | Walks backward from a conditional instruction to find the `CMP` that sets its flags, ensuring no other instruction reads or writes `NZCV` in between |
| `getAdjustedCmpInfo(CmpMI, CC)` | Computes the new `{Imm, Opc, CC}` triple after a ±1 adjustment. Handles signed/unsigned and CMN (ADDS) edge cases |
| `getComplementOpc(Opc)` | Swaps `ADDS ↔ SUBS` (needed when adjusting `CMN 1` to `CMP 0`) |
| `getAdjustedCmp(CC)` | Flips a condition code between inclusive and exclusive: `GT↔GE`, `LT↔LE`, `HI↔HS`, `LO↔LS` |
| `updateCmpInstr(CmpMI, NewImm, NewOpc)` | Writes the new immediate and opcode into the `MachineInstr` |
| `updateCondInstr(CondMI, NewCC)` | Writes the new condition code into the conditional instruction's operand |
| `applyCmpAdjustment(Pair, Info)` | Calls both `updateCmpInstr` and `updateCondInstr` and updates `Pair.CC` |
| `tryOptimizePair(First, Second)` | The core decision function: given two `CmpCondPair`s, decides whether and how to adjust them, then calls `applyCmpAdjustment` |
| `commitPendingPair(...)` | Used by intra-block: flushes a pending pair into the register map and tries to match it against a prior pair for the same register |
| `parseCondCode(Cond)` | Extracts an `AArch64CC::CondCode` from the operand array returned by `analyzeBranch` |

---

## 7. The two optimization modes

### 7.1 Intra-block (`optimizeIntraBlock`)

Scans a single `MachineBasicBlock` from top to bottom, tracking:

- `ActiveCmp` — the most recent `CMP` instruction seen
- `PendingPair` — a `CmpCondPair` waiting to be matched (a `CMP` followed
  by exactly one select-family conditional)
- `PairsByReg` — a map from register → last committed `CmpCondPair` for
  that register

**State machine per instruction:**

```
CMP instruction
  → commit any PendingPair, set ActiveCmp = this CMP

Non-CMP NZCV writer (e.g. ADDS used for arithmetic, not comparison)
  → commit PendingPair, clear ActiveCmp, clear PairsByReg

Select-family conditional (CSEL, CSINC, CSINV, CSNEG, CSET) — not a branch
  → if PendingPair exists: two conditionals share one CMP → unsafe, clear both
  → if ActiveCmp exists: form PendingPair = {ActiveCmp, this MI, CC}

Any other NZCV reader
  → clear ActiveCmp and PendingPair (flags consumed by unknown instruction)
```

At end of block: commit the final `PendingPair` only if `NZCV` does not
live out (a successor block might read the flags).

### 7.2 Cross-block (`optimizeCrossBlock`)

Looks at a pair of blocks in the dominator tree: the **head block** (HBB)
and its **true successor** (TBB, the block branched to when the head's
condition is true).

**Current flow:**

```
1. analyzeBranch(HBB) → get HeadCondOperands
2. analyzeBranch(TBB) → get TrueCondOperands
3. getBccTerminator(HBB) → HeadBrMI   (must be Bcc)
4. getBccTerminator(TBB) → TrueBrMI   (must be Bcc)
5. nzcvLivesOut check on both blocks
6. findAdjustableCmp(HeadBrMI) → HeadCmpMI
7. findAdjustableCmp(TrueBrMI) → TrueCmpMI
8. registersMatch(HeadCmpMI, TrueCmpMI)
9. parseCondCode → HeadCondCode, TrueCondCode
10. tryOptimizePair({HeadCmpMI, HeadBrMI, HeadCC}, {TrueCmpMI, TrueBrMI, TrueCC})
```

The entire pass entry point (`run`) visits every block in dominator-tree
pre-order and calls both `optimizeIntraBlock` and `optimizeCrossBlock` on
each block.

---

## 8. Safety invariants the pass enforces

These must hold before any modification is made. They are checked by the
helper functions described above.

| Invariant | Checked by | Why it matters |
|---|---|---|
| The CMP immediate is a literal integer, not a symbol | `canAdjustCmp` | Cannot do arithmetic on a symbolic operand |
| The adjusted immediate stays within the 12-bit AArch64 immediate range (`< 0xfff`) | `canAdjustCmp` | Encoding constraint |
| The CMP destination register is dead | `canAdjustCmp` | `SUBS`/`ADDS` write a result register; if anything reads it, changing the opcode or immediate would change that result |
| No instruction reads `NZCV` between the CMP and its conditional consumer | `findAdjustableCmp` | Another reader would be silently affected by the flag change |
| No instruction writes `NZCV` between the CMP and its conditional consumer | `findAdjustableCmp` | The CMP we found might not actually be the one setting the flags the conditional reads |
| Both CMPs compare the same register (tracing through copies) | `registersMatch` | Adjusting immediates only makes sense if both comparisons are about the same value |
| `NZCV` does not live out of the block being modified | `nzcvLivesOut` | A successor block reading `NZCV` would see the modified flags |
| Only one conditional instruction consumes a given CMP (intra-block) | `optimizeIntraBlock` state machine | Modifying the CMP would silently change what both consumers compare against |

---

## 9. What is missing — the two TODOs

The file header lists:

```
// TODO: For cross-block:
//   - handle other conditional instructions (e.g. CSET)
//   - allow second branching to be anything if it doesn't require adjusting
```

### 9.1 TODO 1 — CSET and other conditional instructions in cross-block

**The gap:**

`optimizeCrossBlock` calls `getBccTerminator` on both the head block and
the true-successor block. `getBccTerminator` returns `nullptr` for anything
that is not a `Bcc` instruction, causing the function to bail out
immediately.

This means the following pattern is completely missed:

```asm
    cmp   w8, #4
    cset  w9, gt        ; w9 = (w8 > 4) ? 1 : 0   ← not a Bcc, so bail
    ; fall-through
.LBB_next:
    cmp   w8, #6
    b.lt  .LBB_far
```

The head block ends with a `CSET` (a select-family instruction), not a
branch. The cross-block path never sees it.

**What needs to change:**

The cross-block path needs a second way to find the "conditional consumer"
in the head block: scan backward from the end of the block for a
select-family instruction that reads `NZCV`. The helper
`findCondCodeUseOperandIdxForBranchOrSelect` (already in `AArch64InstrInfo`)
returns the operand index of the condition code for any branch or
select-family instruction, or `-1` otherwise. This is exactly what
`optimizeIntraBlock` already uses.

A new helper — call it `findSelectConsumer` — would walk backward from the
last non-terminator instruction looking for a `CSEL`/`CSET`/`CSINC`/
`CSINV`/`CSNEG` that reads `NZCV`, with the same `NZCV`-interference check
that `findAdjustableCmp` uses.

The condition code is then extracted using
`findCondCodeUseOperandIdxForBranchOrSelect` instead of `parseCondCode`
(which only works for `analyzeBranch` output).

The rest of the cross-block logic — `findAdjustableCmp`, `registersMatch`,
`tryOptimizePair` — is reused unchanged.

### 9.2 TODO 2 — Allow second block to be anything if no adjustment needed

**The gap:**

`optimizeCrossBlock` currently requires **both** blocks to have a `Bcc`
terminator. But if the second comparison already has the right immediate
(no adjustment needed), the second block's terminator type is irrelevant —
only the first block's `CMP` needs to change.

Example: if the head block has `cmp w8, #4 / b.gt` and the true-successor
has `cmp w8, #5 / <anything>`, the head can be adjusted to `cmp w8, #5 /
b.ge` without touching the successor at all. The successor's terminator
type does not matter.

**What needs to change:**

In `tryOptimizePair`, the "same-direction" case (Case B) already only
adjusts one side. The cross-block path just needs to stop requiring the
second block to have a `Bcc` when `tryOptimizePair` would not need to
modify it. This requires either:

- Passing a flag to `tryOptimizePair` indicating the second consumer is
  not modifiable, and having it succeed only when the second side needs no
  change, or
- Doing a pre-check in `optimizeCrossBlock`: compute `getAdjustedCmpInfo`
  for the first pair and see if it already matches the second pair's
  immediate and opcode before requiring the second block to have a `Bcc`.

---

## 10. Background knowledge you need

### 10.1 LLVM Machine IR

Machine IR is the compiler's internal representation of assembly
instructions. The four classes you will work with constantly:

| Class | What it represents |
|---|---|
| `MachineFunction` | One compiled function |
| `MachineBasicBlock` | One basic block (straight-line sequence of instructions) |
| `MachineInstr` | One instruction |
| `MachineOperand` | One operand of an instruction (register, immediate, etc.) |

Operand indexing for `SUBSWri` (the `CMP` alias):

```
SUBSWri  Wd, Wn, #imm, shift
          ^   ^    ^      ^
          0   1    2      3
```

- `getOperand(0)` — destination register (always dead for a pure `CMP`)
- `getOperand(1)` — the register being compared
- `getOperand(2)` — the immediate value
- `getOperand(3)` — the shift amount (usually 0)

### 10.2 AArch64 flag-setting instructions

`CMP` on AArch64 is not a real instruction. It is an assembler alias:

```
CMP  Wn, #imm   →   SUBS  WZR, Wn, #imm    (SUBSWri with dead dest)
CMN  Wn, #imm   →   ADDS  WZR, Wn, #imm    (ADDSWri with dead dest)
```

`SUBS` computes `Wn - imm` and sets `NZCV`. `ADDS` computes `Wn + imm`
and sets `NZCV`. The result is discarded (written to `WZR`/`XZR`).

`NZCV` is a physical register in LLVM's model of AArch64. It is not a
real architectural register you can name in assembly, but LLVM tracks it
as a register for liveness purposes.

### 10.3 AArch64 conditional instructions

These all read `NZCV` and are the "consumers" the pass works with:

| Instruction | Meaning |
|---|---|
| `Bcc` | Conditional branch |
| `CSET Wd, cond` | `Wd = (cond) ? 1 : 0` |
| `CSEL Wd, Wn, Wm, cond` | `Wd = (cond) ? Wn : Wm` |
| `CSINC Wd, Wn, Wm, cond` | `Wd = (cond) ? Wn : Wm+1` |
| `CSINV Wd, Wn, Wm, cond` | `Wd = (cond) ? Wn : ~Wm` |
| `CSNEG Wd, Wn, Wm, cond` | `Wd = (cond) ? Wn : -Wm` |

`CSET` is the most common non-branch conditional. It materialises a boolean
into a register and is frequently generated for comparisons whose result is
stored in a variable.

### 10.4 Condition codes

The adjustable condition codes and their inclusive/exclusive pairs:

| Exclusive | Inclusive | Meaning |
|---|---|---|
| `GT` | `GE` | Signed greater-than / greater-or-equal |
| `LT` | `LE` | Signed less-than / less-or-equal |
| `HI` | `HS` | Unsigned higher / higher-or-same |
| `LO` | `LS` | Unsigned lower / lower-or-same |

The key algebraic identity the pass exploits:

```
a > N   ≡   a >= N+1      (GT #N  →  GE #(N+1))
a < N   ≡   a <= N-1      (LT #N  →  LE #(N-1))
```

### 10.5 The dominator tree

Block A **dominates** block B if every path from the function entry to B
passes through A. The pass visits blocks in dominator-tree pre-order so
that when it processes a head block, it has already processed all blocks
that dominate it.

For cross-block optimization, the pass looks at a head block and its
**true-branch successor** (the block jumped to when the condition is true).
The dominator tree is used only for traversal order, not for the
optimization logic itself.

---

## 11. Files and headers to read first

Read these before touching the implementation:

| File | Why |
|---|---|
| `AArch64ConditionOptimizer.cpp` | The file itself — read top to bottom |
| `AArch64InstrInfo.h` / `.cpp` | Contains `findCondCodeUseOperandIdxForBranchOrSelect`, `analyzeBranch`, and other instruction utilities |
| `Utils/AArch64BaseInfo.h` | Defines `AArch64CC::CondCode` and `getCondCodeName` |
| `MCTargetDesc/AArch64AddressingModes.h` | Defines `AArch64_AM::getShiftValue` used in `canAdjustCmp` |
| `llvm/CodeGen/MachineInstr.h` | Core `MachineInstr` API |
| `llvm/CodeGen/MachineBasicBlock.h` | `isLiveIn`, `successors`, `getFirstTerminator` |
| `llvm/CodeGen/MachineRegisterInfo.h` | `use_nodbg_empty`, `lookThruCopyLike` |
| `llvm/CodeGen/MachineDominators.h` | `MachineDominatorTree` |
| `test/CodeGen/AArch64/aarch64-condopt.ll` | Existing test cases — read these to understand what the pass already handles |

---

## 12. How to build and test

**Build just the AArch64 backend and the optimizer test tool:**

```bash
cmake -S llvm -B build \
  -DCMAKE_BUILD_TYPE=Debug \
  -DLLVM_TARGETS_TO_BUILD=AArch64 \
  -DLLVM_ENABLE_ASSERTIONS=ON \
  -G Ninja

ninja -C build llc FileCheck
```

**Run the existing tests:**

```bash
# Run all condopt tests
llvm-lit build/test/CodeGen/AArch64/ -k -j4 \
  --filter="condopt"

# Run a single test file
./build/bin/llc -mtriple=aarch64 \
  -run-pass=aarch64-condopt \
  -o - test/CodeGen/AArch64/aarch64-condopt.ll | \
  FileCheck test/CodeGen/AArch64/aarch64-condopt.ll
```

**Enable debug output to see what the pass is doing:**

```bash
./build/bin/llc -mtriple=aarch64 \
  -run-pass=aarch64-condopt \
  -debug-only=aarch64-condopt \
  -o - input.ll
```

**Dump Machine IR before and after the pass:**

```bash
./build/bin/llc -mtriple=aarch64 \
  -stop-before=aarch64-condopt \
  -o before.mir input.ll

./build/bin/llc -mtriple=aarch64 \
  -stop-after=aarch64-condopt \
  -o after.mir input.ll

diff before.mir after.mir
```

---

## 13. Implementation plan for TODO 1

The goal is to let `optimizeCrossBlock` recognise a head block that ends
with a select-family instruction (`CSET`, `CSEL`, `CSINC`, `CSINV`,
`CSNEG`) instead of a `Bcc`.

### Step 1 — Add `findSelectConsumer`

Add a new private method to `AArch64ConditionOptimizerImpl`:

```cpp
// Searches backward from the end of MBB (excluding terminators) for a
// select-family instruction that reads NZCV, with no intervening NZCV
// interference. Returns the instruction and its condition code, or
// {nullptr, Invalid} if none is found.
std::pair<MachineInstr *, AArch64CC::CondCode>
findSelectConsumer(MachineBasicBlock *MBB);
```

Implementation sketch:

```cpp
std::pair<MachineInstr *, AArch64CC::CondCode>
AArch64ConditionOptimizerImpl::findSelectConsumer(MachineBasicBlock *MBB) {
  for (MachineBasicBlock::reverse_iterator It = MBB->rbegin(),
                                           E = MBB->rend();
       It != E; ++It) {
    MachineInstr &MI = *It;
    if (MI.isDebugInstr()) continue;
    if (MI.isTerminator()) continue;

    // Stop if something writes NZCV — the select below this point
    // is not reading the flags we care about.
    if (MI.modifiesRegister(AArch64::NZCV, /*TRI=*/nullptr))
      return {nullptr, AArch64CC::Invalid};

    int CCOpIdx =
        AArch64InstrInfo::findCondCodeUseOperandIdxForBranchOrSelect(MI);
    if (CCOpIdx >= 0 && !MI.isBranch()) {
      AArch64CC::CondCode CC =
          (AArch64CC::CondCode)(int)MI.getOperand(CCOpIdx).getImm();
      return {&MI, CC};
    }

    // Any other NZCV reader means the select we want is not the last consumer.
    if (MI.readsRegister(AArch64::NZCV, /*TRI=*/nullptr))
      return {nullptr, AArch64CC::Invalid};
  }
  return {nullptr, AArch64CC::Invalid};
}
```

### Step 2 — Extend `optimizeCrossBlock`

After the existing `getBccTerminator` call for the head block fails,
fall back to `findSelectConsumer`:

```cpp
MachineInstr *HeadCondMI = getBccTerminator(&HBB);
AArch64CC::CondCode HeadCondCode;

if (HeadCondMI) {
  // Existing path: head ends with Bcc
  HeadCondCode = parseCondCode(HeadCondOperands);
  if (HeadCondCode == AArch64CC::Invalid) return false;
} else {
  // New path: head ends with a select-family instruction
  auto [SelectMI, SelectCC] = findSelectConsumer(&HBB);
  if (!SelectMI) return false;
  HeadCondMI = SelectMI;
  HeadCondCode = SelectCC;
}
```

The rest of the function (`findAdjustableCmp`, `registersMatch`,
`tryOptimizePair`) is unchanged — it already works with any
`CmpCondPair`, regardless of whether `CondMI` is a branch or a select.

### Step 3 — Update `findAdjustableCmp` if needed

`findAdjustableCmp` currently asserts `!I.isTerminator()`. When the
consumer is a select instruction (not a terminator), this is fine. But
verify that the backward scan starts from the right position — it should
start from `HeadCondMI`, not from the block's terminator.

### Step 4 — Add tests

See [Section 15](#15-writing-tests).

---

## 14. Implementation plan for TODO 2

The goal is to allow the second block's terminator to be something other
than `Bcc` when the second `CMP` already has the right immediate and no
adjustment to the second block is needed.

### Step 1 — Understand when the second block needs no change

In `tryOptimizePair`, the "same-direction" case (Case B) only adjusts one
side — the one whose immediate needs to change. If the second pair is the
"target" (the one that stays unchanged), then the second block's
terminator type is irrelevant.

### Step 2 — Pre-check in `optimizeCrossBlock`

Before requiring `TrueBrMI` to be a `Bcc`, compute whether the first
pair's adjustment would converge to the second pair's immediate:

```cpp
// Try to find a Bcc terminator in TBB. If not found, check whether
// the optimization would only need to modify HBB (not TBB).
MachineInstr *TrueCondMI = getBccTerminator(TBB);
if (!TrueCondMI) {
  // Fall back: find any conditional consumer in TBB
  auto [SelectMI, SelectCC] = findSelectConsumer(TBB);
  if (!SelectMI) return false;
  TrueCondMI = SelectMI;
  TrueCondCode = SelectCC;
  // Only proceed if tryOptimizePair would not need to modify TBB.
  // This is checked inside tryOptimizePair via the Adj.Imm == Target.getImm()
  // guard — no additional change needed here.
}
```

Alternatively, add a `bool SecondIsReadOnly` parameter to
`tryOptimizePair` and have it return false if it would need to modify the
second pair when `SecondIsReadOnly` is true.

---

## 15. Writing tests

Tests live in `llvm/test/CodeGen/AArch64/`. The existing condopt tests are
in `aarch64-condopt.ll`. Add new test cases in the same file or a new file
`aarch64-condopt-cset.ll`.

**Test format** — use `llc` with `-run-pass=aarch64-condopt` and
`FileCheck`:

```llvm
; RUN: llc -mtriple=aarch64 -run-pass=aarch64-condopt -o - %s | FileCheck %s

; Test: CSET in head block, Bcc in successor — cross-block with CSET
define i32 @test_cset_crossblock(i32 %a) {
entry:
  %cmp1 = icmp sgt i32 %a, 4
  %sel = zext i1 %cmp1 to i32   ; generates CSET
  br label %next
next:
  %cmp2 = icmp slt i32 %a, 6
  br i1 %cmp2, label %taken, label %fallthrough
taken:
  ret i32 1
fallthrough:
  ret i32 0
}

; CHECK-LABEL: test_cset_crossblock
; CHECK:       cmp {{w[0-9]+}}, #5
; CHECK-NEXT:  cset {{w[0-9]+}}, ge
; CHECK:       cmp {{w[0-9]+}}, #5   // This should be removed by CSE
; CHECK-NEXT:  b.le
```

Write tests for:
- `CSET` as head consumer, `Bcc` as successor consumer
- `CSEL` as head consumer
- `CSINC` as head consumer
- Second block has no `Bcc` but first block's adjustment already matches
- Negative cases: `NZCV` interference between `CMP` and `CSET` (should not optimize)
- Negative cases: `NZCV` lives out of the head block (should not optimize)

---

## 16. Pitfalls and edge cases

| Situation | What goes wrong | How the code handles it |
|---|---|---|
| `CMN 0` (ADDS with immediate 0) | `0 + (-1)` wraps; the transformation is not valid | `getAdjustedCmpInfo` returns the original `CmpInfo` unchanged |
| Unsigned wrap: `CMP 0, b.lo` adjusted to `CMN 1, b.ls` | Unsigned semantics change across 0 | `getAdjustedCmpInfo` returns original for unsigned cases |
| Two conditionals sharing one `CMP` | Modifying the `CMP` would silently change both | `optimizeIntraBlock` clears `PendingPair` and `ActiveCmp` when a second conditional is seen |
| `NZCV` lives into a successor | Modifying the `CMP` corrupts the successor's flag use | `nzcvLivesOut` check before any modification |
| The `CMP` destination is not dead | `SUBS`/`ADDS` write a result; changing the opcode changes that result | `canAdjustCmp` checks `MRI->use_nodbg_empty(dest)` |
| Adjusted immediate exceeds 12-bit range | Invalid encoding | `canAdjustCmp` checks `imm << shift >= 0xfff` |
| Copy chains between the two compared registers | `registersMatch` would return false for the same logical value | `TRI->lookThruCopyLike` traces through copies |
| A `CSET` with an intervening `NZCV` reader between it and the `CMP` | The `CSET` is not reading the flags from the `CMP` we found | `findAdjustableCmp` returns `nullptr` if any `NZCV` read is found between the two |
| Loop back-edges (head block == successor) | Would create an infinite optimization loop | `optimizeCrossBlock` checks `TBB == &HBB` and returns false |
 
