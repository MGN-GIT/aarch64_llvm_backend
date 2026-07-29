# LLVM AArch64 Compilation Pipeline — Big Picture
> Key file: `llvm/lib/Target/AArch64/AArch64TargetMachine.cpp` — the `AArch64PassConfig` class

---

## Table of Contents

- [What Is a Compilation Pipeline?](#what-is-a-compilation-pipeline)
- [The Five Stages at a Glance](#the-five-stages-at-a-glance)
- [Stage 1 — LLVM IR](#stage-1--llvm-ir)
  - [What IR Looks Like](#what-ir-looks-like)
  - [IR-Level Passes for AArch64](#ir-level-passes-for-aarch64)
- [Stage 2 — SelectionDAG](#stage-2--selectiondag)
  - [What a DAG Is](#what-a-dag-is)
  - [The Three Sub-Stages Inside SelectionDAG](#the-three-sub-stages-inside-selectiondag)
  - [GlobalISel — The Alternative Path](#globalisel--the-alternative-path)
- [Stage 3 — MachineInstr (MIR)](#stage-3--machineinstr-mir)
  - [What MIR Looks Like](#what-mir-looks-like)
  - [Passes That Run on MIR](#passes-that-run-on-mir)
- [Stage 4 — MCInst](#stage-4--mcinst)
  - [MachineInstr vs MCInst — What Is the Difference?](#machineinstr-vs-mcinst--what-is-the-difference)
- [Stage 5 — Object File / Assembly](#stage-5--object-file--assembly)
- [AArch64PassConfig — The Master Ordered List](#aarch64passconfig--the-master-ordered-list)
- [Where Each Source File Lives in the Pipeline](#where-each-source-file-lives-in-the-pipeline)
- [How to Read a Pass Name](#how-to-read-a-pass-name)
- [End-to-End Worked Example](#end-to-end-worked-example)

---

## What Is a Compilation Pipeline?

A compiler does not translate your C code directly into machine code in one
step. It transforms the program through a series of **intermediate
representations (IRs)** — each one closer to the hardware than the last.

Think of it like a factory assembly line:

```
Raw material                  Finished product
(C / C++ source)              (ARM machine code)

  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  LLVM IR │───▶│  DAG     │───▶│  MIR     │───▶│  MCInst  │───▶│  .o file │
  └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
  "what to do"   "how to do it"  "real instrs"   "raw bytes"    "on disk"
  (abstract)     (target-aware)  (with regs)     (no metadata)  (linked)
```

Each stage has a specific job:

| Stage | Representation | Job |
|---|---|---|
| 1 | LLVM IR | Language-independent, target-independent optimisation |
| 2 | SelectionDAG | Map IR operations to real AArch64 instructions |
| 3 | MachineInstr (MIR) | Allocate registers, schedule, peephole optimise |
| 4 | MCInst | Strip compiler metadata, prepare raw instruction data |
| 5 | Object file | Encode bits, write ELF/Mach-O to disk |

**Why so many stages?**

Each stage can only see what it needs to. IR passes do not need to know about
ARM register names. Register allocation does not need to know about C types.
The MC layer does not need to know about optimisation. Keeping them separate
makes each stage simpler, testable, and reusable across targets.

---

## The Five Stages at a Glance

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                          LLVM IR                                    │
  │   %add = add i32 %a, %b                                             │
  │   store i32 %add, ptr %p                                            │
  └───────────────────────────┬─────────────────────────────────────────┘
                              │
              ┌───────────────▼───────────────┐
              │       IR-Level Passes          │
              │  AArch64ISelLowering           │  ← lowers intrinsics,
              │  SVEIntrinsicOpts              │    calling conventions,
              │  AArch64StackTagging           │    memory tagging, etc.
              └───────────────┬───────────────┘
                              │
  ┌───────────────────────────▼─────────────────────────────────────────┐
  │                       SelectionDAG                                  │
  │                                                                     │
  │      (add)                                                          │
  │      /   \                                                          │
  │   (%a)   (%b)          ← a graph of operations, not a list         │
  │                                                                     │
  └───────────────────────────┬─────────────────────────────────────────┘
                              │
              ┌───────────────▼───────────────┐
              │    Instruction Selection       │
              │  AArch64ISelDAGToDAG.cpp       │  ← picks real AArch64
              │  (or GlobalISel/)              │    instructions
              └───────────────┬───────────────┘
                              │
  ┌───────────────────────────▼─────────────────────────────────────────┐
  │                   MachineInstr (MIR)                                │
  │                                                                     │
  │   ADD  %vreg0, %vreg1, %vreg2                                       │
  │   STR  %vreg0, [%vreg3]        ← real instructions, fake registers  │
  │                                                                     │
  └───────────────────────────┬─────────────────────────────────────────┘
                              │
              ┌───────────────▼───────────────┐
              │   MIR Passes                   │
              │  Register Allocation           │  ← %vreg0 → X0
              │  Instruction Scheduling        │  ← reorder for speed
              │  Peephole Optimisation         │  ← local pattern fixes
              │  AArch64ExpandPseudo           │  ← expand pseudo instrs
              └───────────────┬───────────────┘
                              │
  ┌───────────────────────────▼─────────────────────────────────────────┐
  │                         MCInst                                      │
  │                                                                     │
  │   ADD  X0, X1, X2      ← real registers, no compiler metadata      │
  │   STR  X0, [X3]                                                     │
  │                                                                     │
  └───────────────────────────┬─────────────────────────────────────────┘
                              │
              ┌───────────────▼───────────────┐
              │  AArch64AsmPrinter             │  ← emits text or binary
              │  MCTargetDesc/                 │  ← encodes bits
              └───────────────┬───────────────┘
                              │
  ┌───────────────────────────▼─────────────────────────────────────────┐
  │              Object File (.o) or Assembly (.s)                      │
  │                                                                     │
  │   0x8B020020  0xF9000060  ...   ← 32-bit words, ELF/Mach-O format  │
  │                                                                     │
  └─────────────────────────────────────────────────────────────────────┘
```

---

## Stage 1 — LLVM IR

LLVM IR is the **common language** that all LLVM frontends (Clang, Rust, Swift,
etc.) produce. It is completely target-independent — it does not know about
ARM registers, instruction widths, or calling conventions.

### What IR Looks Like

```llvm
; C source:
;   int add(int a, int b) { return a + b; }

define i32 @add(i32 %a, i32 %b) {
entry:
  %result = add i32 %a, %b   ; add two 32-bit integers
  ret i32 %result             ; return the result
}
```

Key properties of IR:

```
┌─────────────────────────────────────────────────────────┐
│  IR is in SSA form (Static Single Assignment)           │
│                                                         │
│  Every value is assigned exactly once:                  │
│    %x = add i32 %a, %b    ← %x defined here            │
│    %y = mul i32 %x, 2     ← %x used here, never        │
│                              reassigned                 │
│                                                         │
│  This makes optimisation much easier — you always       │
│  know exactly where a value comes from.                 │
└─────────────────────────────────────────────────────────┘
```

IR uses **virtual registers** (written `%name`) — there are unlimited of them.
The real, limited AArch64 registers (X0–X30, etc.) are not assigned until
Stage 3.

### IR-Level Passes for AArch64

Before instruction selection even begins, several AArch64-specific passes run
on the IR to prepare it:

```
LLVM IR
  │
  ├─▶  AArch64ISelLowering        Converts IR constructs that have no direct
  │                               AArch64 equivalent into ones that do.
  │                               Examples:
  │                                 • Lowers calling conventions (which args
  │                                   go in X0, X1, ... vs the stack)
  │                                 • Lowers varargs, returns, tail calls
  │                                 • Lowers memory intrinsics (memcpy, etc.)
  │
  ├─▶  SVEIntrinsicOpts           Optimises SVE (Scalable Vector Extension)
  │                               intrinsic calls. For example, removes
  │                               redundant predicate operations.
  │
  ├─▶  AArch64StackTagging        Inserts MTE (Memory Tagging Extension)
  │                               tag instructions for heap/stack safety.
  │
  └─▶  AArch64PromoteConstant     Moves large constants out of the function
                                  body into a constant pool so they can be
                                  loaded with a single ADRP+LDR pair.
```

> **Beginner tip:** You do not need to understand all these passes right away.
> The key idea is that IR passes fix up things that are legal in IR but
> awkward or impossible to express directly in AArch64 instructions.

---

## Stage 2 — SelectionDAG

This is where the compiler first becomes **target-aware**. The IR is converted
into a **Directed Acyclic Graph (DAG)** of operations, and then real AArch64
instructions are selected to implement those operations.

### What a DAG Is

A DAG is a graph where:
- **Nodes** are operations (add, load, store, branch, ...)
- **Edges** are data dependencies (this node's output feeds that node's input)
- **Directed** means edges have a direction (data flows one way)
- **Acyclic** means there are no loops (no circular dependencies)

```
IR:
  %sum  = add  i32 %a, %b
  %prod = mul  i32 %sum, %c
  store i32 %prod, ptr %p

SelectionDAG:
                  [store]
                     │
                  [mul]
                  /    \
              [add]    [%c]
              /   \
           [%a]  [%b]
```

The DAG makes **data flow explicit** — you can see at a glance that `store`
depends on `mul`, which depends on `add`. This makes it easy to combine
multiple operations into a single instruction (instruction combining).

### The Three Sub-Stages Inside SelectionDAG

```
SelectionDAG
  │
  ├─▶  1. DAG Lowering (AArch64ISelLowering.cpp)
  │       Converts target-independent DAG nodes into AArch64-specific ones.
  │       Example: a generic "call" node becomes an AArch64 "BL" node with
  │       the correct argument registers set up.
  │
  ├─▶  2. DAG Combining
  │       Merges multiple DAG nodes into one where possible.
  │       Example:
  │         [add] + [shift]  →  [add with shift operand]
  │         This maps to a single AArch64 instruction:
  │           ADD X0, X1, X2, LSL #3
  │
  └─▶  3. Instruction Selection (AArch64ISelDAGToDAG.cpp)
          Walks the DAG and picks a real AArch64 instruction for each node.
          Driven by patterns in AArch64InstrInfo.td:
            (add GPR64:$Rn, GPR64:$Rm)  →  ADD Xd, Xn, Xm
```

**Key file:** `AArch64ISelDAGToDAG.cpp`

This file contains the `AArch64DAGToDAGISel` class. Its `Select()` method is
called for every DAG node and returns the chosen `MachineInstr`. Most of the
matching is done automatically from TableGen patterns, but complex addressing
modes (like `[X0, W1, UXTW #2]`) are handled by hand-written C++ methods like
`SelectAddrModeIndexed`.

### GlobalISel — The Alternative Path

LLVM has a newer instruction selection framework called **GlobalISel** that
works directly on IR-like instructions rather than a DAG. It lives in:

```
llvm/lib/Target/AArch64/GISel/
  ├── AArch64CallLowering.cpp
  ├── AArch64GlobalISelUtils.cpp
  ├── AArch64InstructionSelector.cpp   ← the main selector
  ├── AArch64LegalizerInfo.cpp
  └── AArch64RegisterBankInfo.cpp
```

```
SelectionDAG path (classic):
  IR → DAG → Combine → Select → MIR

GlobalISel path (newer):
  IR → Generic MIR → Legalise → RegBankSelect → Select → MIR
       (G_ADD, etc.)
```

For most AArch64 code today, SelectionDAG is still the default. GlobalISel is
used for some fast-compile scenarios (e.g. `-O0`).

> **Beginner tip:** When reading the AArch64 backend, you will see both paths.
> Focus on SelectionDAG first — it is more mature and more of the interesting
> optimisation work happens there.

---

## Stage 3 — MachineInstr (MIR)

After instruction selection, the program is represented as a list of
**MachineInstr** objects — one per instruction. This is called **MIR**
(Machine IR).

### What MIR Looks Like

```
; MIR after instruction selection (virtual registers, not real ones yet)
bb.0.entry:
  %0:gpr64 = COPY $x0          ; argument 'a' arrives in X0
  %1:gpr64 = COPY $x1          ; argument 'b' arrives in X1
  %2:gpr64 = ADDXrr %0, %1     ; %2 = a + b  (virtual register)
  $x0 = COPY %2                ; return value goes in X0
  RET_ReallyLR                 ; return

; MIR after register allocation (virtual registers replaced with real ones)
bb.0.entry:
  ADDXrr $x0, $x0, $x1         ; X0 = X0 + X1
  RET_ReallyLR
```

The key difference between MIR and IR:
- MIR uses **real instruction names** (`ADDXrr`, `LDRXui`, ...)
- MIR uses **virtual registers** at first (`%0`, `%1`, ...) then **real
  registers** (`$x0`, `$x1`, ...) after register allocation
- MIR is a **flat list** of instructions per basic block, not a graph

### Passes That Run on MIR

This is where the bulk of the backend work happens. The passes run in order:

```
MachineInstr (MIR)
  │
  ├─▶  AArch64ExpandPseudo
  │       Expands pseudo-instructions into real instruction sequences.
  │       Example:
  │         MOVaddr %dst, @global_var
  │         ──────────────────────────▶
  │         ADRP %dst, @global_var     ; load page address
  │         ADD  %dst, %dst, :lo12:@global_var  ; add page offset
  │
  ├─▶  Register Allocation (RegAlloc)
  │       Assigns real AArch64 registers (X0–X30, W0–W30, etc.) to
  │       virtual registers. If there are not enough registers, spills
  │       values to the stack.
  │
  │       Before:  ADD %vreg5, %vreg2, %vreg3
  │       After:   ADD X2, X0, X1
  │
  ├─▶  Prologue / Epilogue Insertion (AArch64FrameLowering)
  │       Inserts the function prologue (save callee-saved registers,
  │       allocate stack frame) and epilogue (restore registers, return).
  │       Example prologue:
  │         STP X29, X30, [SP, #-16]!   ; save frame pointer + link register
  │         MOV X29, SP                 ; set up frame pointer
  │
  ├─▶  Instruction Scheduling
  │       Reorders instructions to avoid pipeline stalls and hide memory
  │       latency. On AArch64, a load result is not available for 4–5
  │       cycles — the scheduler tries to put other work in between.
  │
  ├─▶  AArch64A57FPLoadBalancing
  │       Cortex-A57 specific: balances FP/SIMD instructions across the
  │       two FP pipelines to avoid bottlenecks.
  │
  ├─▶  Peephole Optimisation (AArch64MIPeepholeOpt)
  │       Looks at small windows of instructions and replaces them with
  │       shorter/faster equivalents.
  │       Example:
  │         MOV W0, W0    ← redundant, remove it
  │         UBFX + ORR    ← can sometimes become a single BFI
  │
  ├─▶  AArch64MovePrefixPass (MOVPRFX for SVE)
  │       Inserts MOVPRFX instructions before SVE destructive operations
  │       to break false register dependencies.
  │
  └─▶  AArch64BranchRelaxation
          If a branch target is too far away for the instruction's
          bit field (e.g. a conditional branch only has ±1 MB range),
          inserts a trampoline:
            B.EQ  far_target          ← too far, does not fit in 19 bits
            ──────────────────────▶
            B.NE  skip                ← inverted condition, short jump
            B     far_target          ← unconditional, 26-bit range (±128 MB)
          skip:
```

---

## Stage 4 — MCInst

`MCInst` is a **stripped-down version of MachineInstr**. It contains only what
is needed to encode the instruction as bits — no compiler metadata, no
liveness information, no virtual registers.

### MachineInstr vs MCInst — What Is the Difference?

```
MachineInstr                          MCInst
────────────────────────────────────  ────────────────────────────────────
Knows about basic blocks              No concept of basic blocks
Has virtual registers                 Only real registers (by number)
Has debug info, flags, metadata       No metadata
Used by all MIR passes                Used only for emission
Lives in MachineFunction              Lives in MC layer (target-independent)
~200 bytes                            ~20 bytes
```

The conversion happens in `AArch64AsmPrinter.cpp`:

```
MachineInstr  ──▶  AArch64AsmPrinter::emitInstruction()
                         │
                         ▼
                   MCInst  ──▶  MCStreamer  ──▶  .o or .s
```

The `AsmPrinter` calls `lowerToMCInst()` which copies the opcode and operands
from `MachineInstr` into a fresh `MCInst`, converting register numbers and
resolving any remaining pseudo-operands.

---

## Stage 5 — Object File / Assembly

The final stage writes the actual bytes to disk.

```
MCInst
  │
  ├─▶  Text output path (.s file):
  │       MCInstPrinter (AArch64InstPrinter.cpp)
  │       Converts MCInst back to human-readable assembly text:
  │         MCInst{opcode=ADD, X0, X1, X2}  →  "add x0, x1, x2\n"
  │
  └─▶  Binary output path (.o file):
          MCCodeEmitter (AArch64MCCodeEmitter.cpp)
          Reads the bit field assignments from AArch64InstrFormats.td
          (via TableGen-generated tables) and packs them into 4 bytes:
            MCInst{opcode=ADD, X0, X1, X2}  →  0x8B020000

          MCObjectWriter wraps the bytes in ELF or Mach-O format,
          adds relocation entries, and writes the .o file.
```

The relevant files all live in:

```
llvm/lib/Target/AArch64/MCTargetDesc/
  ├── AArch64MCCodeEmitter.cpp    ← encodes MCInst → 32-bit words
  ├── AArch64AsmBackend.cpp       ← applies relocations, fixups
  ├── AArch64ELFObjectWriter.cpp  ← ELF-specific output
  └── AArch64MachObjectWriter.cpp ← Mach-O specific output
```

---

## AArch64PassConfig — The Master Ordered List

All of the passes described above are registered in one place:

```
llvm/lib/Target/AArch64/AArch64TargetMachine.cpp
```

The `AArch64PassConfig` class inherits from `TargetPassConfig` and overrides
several methods to add passes at specific points in the pipeline:

```cpp
// Simplified view of AArch64PassConfig
class AArch64PassConfig : public TargetPassConfig {

  // IR-level passes (run before SelectionDAG)
  bool addPreISel() override {
    addPass(createAArch64StackTaggingPass());
    addPass(createAArch64PromoteConstantPass());
    addPass(createSVEIntrinsicOptsPass());
    // ...
    return false;
  }

  // Instruction selection
  bool addInstSelector() override {
    addPass(createAArch64ISelDag(/* ... */));  // SelectionDAG path
    return false;
  }

  // Passes after instruction selection, before register allocation
  void addPreRegAlloc() override {
    addPass(createAArch64A57FPLoadBalancingPass());
    addPass(createAArch64ExpandPseudoPass());
    // ...
  }

  // Passes after register allocation
  void addPostRegAlloc() override {
    addPass(createAArch64LoadStoreOptimizationPass());
    // ...
  }

  // Late passes, just before emission
  void addPreEmitPass() override {
    addPass(createAArch64BranchRelaxation());
    addPass(createAArch64MIPeepholeOptPass());
    addPass(createAArch64MovePrefixPass());
    // ...
  }
};
```

**Why is this file important?**

If you want to know **exactly** which passes run and in what order, this is the
single source of truth. Every pass that touches AArch64 code is registered
here. When you are debugging a miscompilation or a missed optimisation, you
start here to find which pass is responsible.

---

## Where Each Source File Lives in the Pipeline

```
Pipeline Stage          Source File(s)
──────────────────────  ──────────────────────────────────────────────────────
IR Passes               AArch64ISelLowering.cpp
                        AArch64ISelLowering.h
                        SVEIntrinsicOpts.cpp
                        AArch64StackTagging.cpp
                        AArch64PromoteConstant.cpp

Instruction Selection   AArch64ISelDAGToDAG.cpp        ← SelectionDAG path
(SelectionDAG)          AArch64ISelDAGToDAG.h
                        AArch64InstrInfo.td             ← patterns (TableGen)
                        AArch64InstrFormats.td          ← encodings (TableGen)

Instruction Selection   GISel/AArch64InstructionSelector.cpp  ← GlobalISel
(GlobalISel)            GISel/AArch64CallLowering.cpp
                        GISel/AArch64LegalizerInfo.cpp
                        GISel/AArch64RegisterBankInfo.cpp

MIR Passes              AArch64ExpandPseudo.cpp
                        AArch64FrameLowering.cpp        ← prologue/epilogue
                        AArch64LoadStoreOptimizer.cpp
                        AArch64A57FPLoadBalancing.cpp
                        AArch64MIPeepholeOpt.cpp
                        AArch64BranchRelaxation.cpp
                        AArch64MovePrefixPass.cpp

Emission (MCInst)       AArch64AsmPrinter.cpp           ← MachineInstr → MCInst
                        MCTargetDesc/AArch64MCCodeEmitter.cpp  ← MCInst → bits
                        MCTargetDesc/AArch64AsmBackend.cpp
                        AArch64InstPrinter.cpp          ← MCInst → text

Pipeline Registration   AArch64TargetMachine.cpp        ← AArch64PassConfig
```

---

## How to Read a Pass Name

Pass names in LLVM follow a loose convention. Once you know the pattern, you
can guess what a pass does from its name alone:

```
AArch64  ExpandPseudo
───────  ────────────
  │           │
  │           └── What it does
  └── Target (always AArch64 for backend passes)

AArch64  LoadStore  Optimizer
───────  ─────────  ─────────
  │          │          │
  │          │          └── Role (Optimizer, Pass, Lowering, ...)
  │          └── What it operates on
  └── Target

AArch64  A57  FP  LoadBalancing
───────  ───  ──  ─────────────
  │       │    │       │
  │       │    │       └── What it does
  │       │    └── Domain (FP = floating point)
  │       └── Microarchitecture (Cortex-A57 specific)
  └── Target
```

Common suffixes:

| Suffix | Meaning |
|---|---|
| `Lowering` | Converts high-level constructs to lower-level ones |
| `ExpandPseudo` | Replaces pseudo-instructions with real sequences |
| `Optimizer` | Improves existing instruction sequences |
| `Peephole` | Local pattern-matching optimisation (small window) |
| `Relaxation` | Fixes up instructions that are out of range |
| `FrameLowering` | Handles stack frame setup and teardown |
| `ISelDAGToDAG` | Instruction selection via SelectionDAG |

---

## End-to-End Worked Example

Let's trace a single C function through every stage:

```c
// Source
int multiply_add(int a, int b, int c) {
    return a * b + c;
}
```

**Stage 1 — LLVM IR**

```llvm
define i32 @multiply_add(i32 %a, i32 %b, i32 %c) {
entry:
  %mul = mul i32 %a, %b
  %add = add i32 %mul, %c
  ret i32 %add
}
```

No AArch64 knowledge here — just abstract integer operations.

---

**Stage 2 — SelectionDAG**

The DAG for the function body:

```
         [ret]
           │
         [add]
         /    \
      [mul]   [%c / X2]
      /    \
[%a / X0]  [%b / X1]
```

The instruction selector sees `[add [mul X0, X1], X2]` and recognises this
matches the AArch64 `MADD` instruction (multiply-accumulate):

```
MADD Xd, Xn, Xm, Xa   →   Xd = Xn * Xm + Xa
```

So the entire `mul + add` collapses into **one instruction**.

---

**Stage 3 — MachineInstr (MIR)**

After instruction selection (virtual registers):

```
%0:gpr32 = COPY $w0
%1:gpr32 = COPY $w1
%2:gpr32 = COPY $w2
%3:gpr32 = MADDWrrr %0, %1, %2    ; %3 = %0 * %1 + %2
$w0 = COPY %3
RET_ReallyLR
```

After register allocation (real registers):

```
$w0 = MADDWrrr $w0, $w1, $w2      ; W0 = W0 * W1 + W2
RET_ReallyLR
```

The three `COPY` instructions were eliminated because the allocator was able
to assign the virtual registers directly to the argument registers.

---

**Stage 4 — MCInst**

```
MCInst {
  opcode = AArch64::MADDWrrr,
  operands = [ W0 (def), W0, W1, W2 ]
}
MCInst {
  opcode = AArch64::RET,
  operands = [ X30 ]
}
```

---

**Stage 5 — Object File**

`MADDWrrr W0, W0, W1, W2` encodes as:

```
 31 30 29 28    24 23 21 20    16 15 14    10 9     5 4     0
  0  0  0  11011  000  Rm[4:0]  0  Ra[4:0]  Rn[4:0]  Rd[4:0]
  0  0  0  11011  000  00001    0  00010     00000    00000

Binary: 0001 1011 0000 0001 0000 1000 0000 0000
Hex:    0x1B010800
```

Written to the `.o` file as 4 bytes: `00 08 01 1B` (little-endian).

---

**Summary of the full journey:**

```
int multiply_add(int a, int b, int c) { return a * b + c; }
         │
         ▼  Clang
%mul = mul i32 %a, %b
%add = add i32 %mul, %c
         │
         ▼  SelectionDAG (pattern matching: mul+add → MADD)
MADDWrrr %vreg3, %vreg0, %vreg1, %vreg2
         │
         ▼  Register Allocation
MADDWrrr W0, W0, W1, W2
         │
         ▼  MCCodeEmitter
0x1B010800
         │
         ▼  ELF Writer
.o file: [ELF header][.text: 00 08 01 1B ...][.symtab: multiply_add @ 0x0]
```

Four source lines of C became one 4-byte AArch64 instruction. Every stage of
the pipeline contributed to making that happen.
