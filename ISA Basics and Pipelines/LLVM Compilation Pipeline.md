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
- [Files Not Yet Covered](#files-not-yet-covered)

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
  ├─▶  AArch64ISelLowering.cpp      Converts IR constructs that have no direct
  │                                 AArch64 equivalent into ones that do.
  │                                 Examples:
  │                                   • Lowers calling conventions (which args
  │                                     go in X0, X1, ... vs the stack)
  │                                   • Lowers varargs, returns, tail calls
  │                                   • Lowers memory intrinsics (memcpy, etc.)
  │
  ├─▶  SVEIntrinsicOpts.cpp         Optimises SVE (Scalable Vector Extension)
  │                                 intrinsic calls. Removes redundant predicate
  │                                 operations and simplifies SVE patterns.
  │
  ├─▶  SVEShuffleOpts.cpp           Optimises SVE shuffle operations.
  │                                 Recognises patterns that can be lowered to
  │                                 cheaper SVE instructions.
  │
  ├─▶  AArch64StackTagging.cpp      Inserts MTE (Memory Tagging Extension)
  │                                 tag instructions for heap/stack safety.
  │                                 Runs before register allocation so tags
  │                                 can be assigned to stack slots.
  │
  ├─▶  AArch64StackTaggingPreRA.cpp Second MTE pass — runs just before register
  │                                 allocation to finalise tag assignments.
  │
  ├─▶  AArch64PromoteConstant.cpp   Moves large constants out of the function
  │                                 body into a constant pool so they can be
  │                                 loaded with a single ADRP+LDR pair instead
  │                                 of multiple MOVZ/MOVK instructions.
  │
  ├─▶  AArch64TargetTransformInfo.cpp  Answers cost-model questions for the
  │                                    generic IR optimisers:
  │                                    "How expensive is this operation on
  │                                    AArch64?" Used by vectoriser, inliner,
  │                                    loop unroller to make decisions.
  │
  └─▶  AArch64Arm64ECCallLowering.cpp  Lowers calls for the Arm64EC ABI
                                       (Windows x64-compatible ABI for AArch64).
                                       Inserts thunks for cross-ABI calls.
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
GISel/
  ├── AArch64CallLowering.cpp           Lowers calls/returns to real instructions
  ├── AArch64CallLowering.h
  ├── AArch64GlobalISelUtils.cpp        Shared utility functions
  ├── AArch64GlobalISelUtils.h
  ├── AArch64InstructionSelector.cpp    The main selector — matches generic
  │                                    MIR ops (G_ADD, G_LOAD...) to real instrs
  ├── AArch64LegalizerInfo.cpp          Defines which generic ops are legal on
  ├── AArch64LegalizerInfo.h            AArch64 and how to make illegal ones legal
  ├── AArch64O0PreLegalizerCombiner.cpp Combines patterns at -O0 before legalise
  ├── AArch64PostLegalizerCombiner.cpp  Combines patterns after legalisation
  ├── AArch64PostLegalizerLowering.cpp  Lowers remaining generic ops after legalise
  ├── AArch64PostSelectOptimize.cpp     Cleans up after instruction selection
  ├── AArch64PreLegalizerCombiner.cpp   Combines patterns before legalisation
  ├── AArch64RegisterBankInfo.cpp       Maps virtual regs to register banks
  └── AArch64RegisterBankInfo.h        (GPR bank vs FPR bank)
```

```
SelectionDAG path (classic):
  IR → DAG → Combine → Select → MIR

GlobalISel path (newer):
  IR → Generic MIR → Legalise → RegBankSelect → Select → MIR
       (G_ADD, etc.)  (make legal)  (GPR or FPR?)  (real instrs)
```

The GlobalISel pipeline has more explicit stages:

```
Generic MIR
  │
  ├─▶  PreLegalizerCombiner     Simplify before legalisation
  │    O0PreLegalizerCombiner   (simpler version for -O0)
  │
  ├─▶  Legalizer                Make all ops legal for AArch64
  │                             e.g. G_ADD i128 → split into two G_ADD i64
  │
  ├─▶  PostLegalizerCombiner    Simplify after legalisation
  │    PostLegalizerLowering    Lower remaining generic ops
  │
  ├─▶  RegBankSelect            Assign each vreg to GPR or FPR bank
  │
  ├─▶  InstructionSelector      Match to real AArch64 instructions
  │
  └─▶  PostSelectOptimize       Clean up after selection
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
  ├─▶  AArch64ExpandPseudoInsts.cpp
  │       Expands pseudo-instructions into real instruction sequences.
  │       Example:
  │         MOVaddr %dst, @global_var
  │         ──────────────────────────▶
  │         ADRP %dst, @global_var
  │         ADD  %dst, %dst, :lo12:@global_var
  │       Also expands atomic pseudo-instructions (CMP_SWAP, etc.) into
  │       LDXR/STXR exclusive load-store loops.
  │
  ├─▶  AArch64ExpandImm.cpp
  │       Expands large immediate values that cannot fit in a single
  │       instruction into MOVZ + MOVK sequences.
  │         MOV X0, #0x1234567890ABCDEF
  │         ──────────────────────────▶
  │         MOVZ X0, #0xCDEF
  │         MOVK X0, #0x90AB, LSL #16
  │         MOVK X0, #0x5678, LSL #32
  │         MOVK X0, #0x1234, LSL #48
  │
  ├─▶  AArch64A57FPLoadBalancing.cpp
  │       Cortex-A57 specific: balances FP/SIMD instructions across the
  │       two FP pipelines to avoid bottlenecks. Reassigns FP register
  │       colours to spread work evenly.
  │
  ├─▶  AArch64AdvSIMDScalarPass.cpp
  │       Promotes scalar FP operations to use the AdvSIMD scalar
  │       register file when profitable. Avoids costly GPR↔FPR moves.
  │
  ├─▶  AArch64SIMDInstrOpt.cpp
  │       Optimises SIMD instruction sequences. For example, replaces
  │       suboptimal vector patterns with more efficient equivalents.
  │
  ├─▶  AArch64StorePairSuppress.cpp
  │       Analyses store patterns and sets the MOSuppressPair flag on
  │       stores that should NOT be merged into STP by the load/store
  │       optimizer. Prevents harmful pairing on some microarchitectures.
  │
  ├─▶  Register Allocation (RegAlloc)
  │       Assigns real AArch64 registers (X0–X30, W0–W30, etc.) to
  │       virtual registers. If there are not enough registers, spills
  │       values to the stack.
  │         Before:  ADD %vreg5, %vreg2, %vreg3
  │         After:   ADD X2, X0, X1
  │
  ├─▶  AArch64PBQPRegAlloc.cpp
  │       AArch64-specific hints for the PBQP (Partitioned Boolean
  │       Quadratic Programming) register allocator. Provides cost
  │       hints to avoid anti-dependencies on specific microarchitectures
  │       (e.g. Cortex-A57 FP pipeline pairing).
  │
  ├─▶  AArch64PostCoalescerPass.cpp
  │       Runs after register coalescing. Cleans up redundant copies
  │       that the coalescer could not eliminate, particularly for
  │       FP/SIMD register moves.
  │
  ├─▶  Prologue / Epilogue Insertion
  │    AArch64PrologueEpilogue.cpp + AArch64FrameLowering.cpp
  │       Inserts the function prologue (save callee-saved registers,
  │       allocate stack frame) and epilogue (restore registers, return).
  │         STP X29, X30, [SP, #-16]!   ; save FP + LR
  │         MOV X29, SP                 ; set up frame pointer
  │         SUB SP, SP, #local_size     ; allocate locals
  │
  ├─▶  AArch64LowerHomogeneousPrologEpilog.cpp
  │       Replaces repeated identical prologue/epilogue sequences with
  │       calls to a shared helper function. Reduces code size when
  │       many functions save the same set of callee-saved registers.
  │
  ├─▶  Instruction Scheduling
  │    AArch64MachineScheduler.cpp + AArch64MacroFusion.cpp
  │       Reorders instructions to avoid pipeline stalls and hide memory
  │       latency. AArch64MacroFusion.cpp teaches the scheduler about
  │       instruction pairs that fuse into a single micro-op on specific
  │       CPUs (e.g. ADRP+ADD, CMP+B.cond).
  │
  ├─▶  AArch64LoadStoreOptimizer.cpp
  │       Merges adjacent loads/stores into LDP/STP pair instructions.
  │         STR X0, [SP, #0]     ─▶   STP X0, X1, [SP, #0]
  │         STR X1, [SP, #8]
  │       Also converts post-index patterns and promotes narrow loads.
  │
  ├─▶  AArch64CodeLayoutOpt.cpp
  │       Reorders basic blocks to improve instruction cache utilisation.
  │       Moves cold (rarely executed) blocks away from hot paths.
  │
  ├─▶  AArch64CondBrTuning.cpp
  │       Converts conditional branch sequences into more efficient forms.
  │       Example: replaces CMP+B.cond with CBZ/CBNZ where possible.
  │
  ├─▶  AArch64ConditionalCompares.cpp
  │       Converts if-else chains into conditional compare sequences
  │       (CCMP/CCMN) to eliminate branches.
  │         if (a == 0 || b == 0)  →  CMP a, #0
  │                                   CCMP b, #0, #4, NE
  │                                   B.EQ target
  │
  ├─▶  AArch64ConditionOptimizer.cpp
  │       Optimises condition code usage. Removes redundant flag-setting
  │       instructions when the flags are already set correctly.
  │
  ├─▶  AArch64RedundantCondBranchPass.cpp
  │       Removes conditional branches whose condition is always true
  │       or always false based on preceding instructions.
  │
  ├─▶  AArch64RedundantCopyElimination.cpp
  │       Removes register copies that are provably redundant.
  │       Example: if X0 is known to already equal X1, removes MOV X0, X1.
  │
  ├─▶  AArch64DeadRegisterDefinitionsPass.cpp
  │       Replaces destination registers of instructions whose results
  │       are never used with XZR/WZR (the zero register). This can
  │       enable further optimisations and avoids false dependencies.
  │
  ├─▶  AArch64CollectLOH.cpp
  │       Collects Linker Optimisation Hints (LOH) — annotations that
  │       tell the linker it can optimise ADRP+LDR/ADD pairs into
  │       shorter sequences when the final address is known at link time.
  │
  ├─▶  AArch64CompressJumpTables.cpp
  │       Compresses jump table entries from 4 bytes to 1 or 2 bytes
  │       when the table entries are close enough to the table base.
  │       Reduces code size for switch statements.
  │
  ├─▶  AArch64CleanupLocalDynamicTLSPass.cpp
  │       Cleans up redundant TLS (Thread Local Storage) descriptor
  │       loads in the local-dynamic TLS model. Merges multiple
  │       __tls_get_addr calls for the same TLS block into one.
  │
  ├─▶  AArch64FalkorHWPFFix.cpp
  │       Falkor (Qualcomm) specific: fixes hardware prefetcher
  │       conflicts. Renames registers to prevent the Falkor prefetcher
  │       from incorrectly predicting strided access patterns.
  │
  ├─▶  AArch64A53Fix835769.cpp
  │       Cortex-A53 erratum 835769 workaround. Inserts a NOP before
  │       certain multiply-accumulate instructions that follow a memory
  │       instruction to avoid a CPU bug.
  │
  ├─▶  AArch64SLSHardening.cpp
  │       Straight-Line Speculation hardening. Inserts speculation
  │       barriers (DSB+ISB or SB) after indirect branches and returns
  │       to prevent speculative execution past those points.
  │
  ├─▶  AArch64SpeculationHardening.cpp
  │       Broader speculation hardening (Spectre mitigations). Masks
  │       register values that are derived from speculatively loaded
  │       data to prevent information leakage.
  │
  ├─▶  AArch64PointerAuth.cpp
  │       Inserts pointer authentication instructions (PACIA, AUTIA,
  │       etc.) for PAC (Pointer Authentication Code) security feature.
  │       Signs return addresses and function pointers.
  │
  ├─▶  AArch64BranchTargets.cpp
  │       Inserts BTI (Branch Target Identification) instructions at
  │       valid indirect branch targets. Part of the landing pad
  │       security feature.
  │
  ├─▶  MachineSMEABIPass.cpp
  │       Handles SME (Scalable Matrix Extension) ABI requirements.
  │       Inserts SMSTART/SMSTOP instructions at function boundaries
  │       when calling between streaming and non-streaming functions.
  │
  ├─▶  AArch64MIPeepholeOpt.cpp
  │       Looks at small windows of instructions and replaces them with
  │       shorter/faster equivalents.
  │         MOV W0, W0    ← redundant, remove it
  │         UBFX + ORR    ← can become a single BFI
  │
  ├─▶  SMEPeepholeOpt.cpp
  │       SME-specific peephole optimisations. Removes redundant
  │       SMSTART/SMSTOP pairs and streaming mode transitions.
  │
  ├─▶  AArch64MovePrefixPass (MOVPRFX for SVE)
  │       Inserts MOVPRFX instructions before SVE destructive operations
  │       to break false register dependencies.
  │
  └─▶  AArch64BranchRelaxation (via generic BranchRelaxation pass)
          If a branch target is too far away for the instruction's
          bit field (e.g. a conditional branch only has ±1 MB range),
          inserts a trampoline:
            B.EQ  far_target     ← too far, does not fit in 19 bits
            ──────────────────▶
            B.NE  skip           ← inverted condition, short jump
            B     far_target     ← unconditional, 26-bit range (±128 MB)
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
MCTargetDesc/
  ├── AArch64MCCodeEmitter.cpp      Encodes MCInst → 32-bit words
  │                                 Reads TableGen-generated bit field tables
  ├── AArch64AsmBackend.cpp         Applies relocations and fixups after
  │                                 encoding. Handles PC-relative fixups
  │                                 whose final value is only known at
  │                                 link time.
  ├── AArch64FixupKinds.h           Defines the relocation types AArch64
  │                                 uses (e.g. R_AARCH64_CALL26)
  ├── AArch64ELFObjectWriter.cpp    ELF-specific output (Linux, Android)
  ├── AArch64ELFStreamer.cpp        ELF-specific MCStreamer — handles
  ├── AArch64ELFStreamer.h          mapping symbols (.text, $x, $d)
  ├── AArch64MachObjectWriter.cpp   Mach-O specific output (Apple)
  ├── AArch64WinCOFFObjectWriter.cpp  COFF output (Windows)
  ├── AArch64WinCOFFStreamer.cpp    Windows-specific MCStreamer
  ├── AArch64WinCOFFStreamer.h
  ├── AArch64InstPrinter.cpp        Converts MCInst → human-readable text
  ├── AArch64InstPrinter.h          e.g. MCInst{ADD,X0,X1,X2} → "add x0,x1,x2"
  ├── AArch64MCAsmInfo.cpp          Defines assembler syntax rules
  ├── AArch64MCAsmInfo.h            (comment chars, directive names, etc.)
  ├── AArch64MCExpr.cpp             Handles AArch64-specific MC expressions
  │                                 like :lo12:, :got:, :tlsdesc: modifiers
  ├── AArch64MCTargetDesc.cpp       Registers all MC components with LLVM
  ├── AArch64MCTargetDesc.h
  ├── AArch64TargetStreamer.cpp      AArch64-specific MCStreamer directives
  ├── AArch64TargetStreamer.h        (.arch, .cpu, .inst directives)
  ├── AArch64MCLFIRewriter.cpp      Rewrites LFI (Landing Pad) annotations
  ├── AArch64MCLFIRewriter.h        in the MC layer for BTI support
  └── AArch64AddressingModes.h      Shared header: encodes/decodes addressing
                                    mode immediates (scaled, unscaled, etc.)
```

Also involved in emission:

```
AArch64AsmPrinter.cpp     MachineInstr → MCInst conversion
                          Handles special pseudo-instructions that need
                          custom lowering (e.g. TLSDESC calls, BTI)
AArch64MCInstLower.cpp    Helper for AsmPrinter: lowers MachineOperands
AArch64MCInstLower.h      (symbols, frame indices) to MCOperands
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
Pipeline Stage            Source File(s)
────────────────────────  ────────────────────────────────────────────────────
Target Registration       AArch64TargetMachine.cpp / .h
                          AArch64Subtarget.cpp / .h       CPU feature detection
                          AArch64.td                      Top-level TableGen
                          AArch64Features.td              CPU feature flags
                          AArch64Processors.td            CPU model definitions
                          AArch64PassRegistry.def         Pass registration
                          TargetInfo/                     LLVM target registration

IR Passes                 AArch64ISelLowering.cpp / .h    Main IR lowering
(pre-ISel)                AArch64TargetTransformInfo.cpp  Cost model
                          AArch64TargetTransformInfo.h
                          AArch64TargetObjectFile.cpp     Section assignment
                          AArch64TargetObjectFile.h
                          SVEIntrinsicOpts.cpp            SVE intrinsic opts
                          SVEShuffleOpts.cpp              SVE shuffle opts
                          AArch64StackTagging.cpp         MTE stack tagging
                          AArch64StackTaggingPreRA.cpp    MTE pre-regalloc
                          AArch64PromoteConstant.cpp      Constant pool
                          AArch64Arm64ECCallLowering.cpp  Arm64EC ABI
                          AArch64SelectionDAGInfo.cpp     DAG memory ops
                          AArch64SelectionDAGInfo.h

TableGen Definitions      AArch64InstrInfo.td             Instruction patterns
                          AArch64InstrFormats.td          Bit encodings
                          AArch64InstrAtomics.td          Atomic instructions
                          AArch64InstrGISel.td            GlobalISel patterns
                          AArch64SVEInstrInfo.td          SVE instructions
                          SVEInstrFormats.td              SVE encoding formats
                          AArch64SMEInstrInfo.td          SME instructions
                          SMEInstrFormats.td              SME encoding formats
                          AArch64LFI.td                   Landing pad instrs
                          AArch64RegisterInfo.td          Register definitions
                          AArch64RegisterBanks.td         Register bank defs
                          AArch64CallingConvention.td     Calling conventions
                          AArch64Combine.td               GlobalISel combines
                          AArch64Schedule.td              Scheduling base
                          AArch64SchedA53.td              Cortex-A53 sched
                          AArch64SchedA55.td              Cortex-A55 sched
                          AArch64SchedA57.td              Cortex-A57 sched
                          AArch64SchedNeoverseN1.td       Neoverse N1 sched
                          AArch64SchedNeoverseV1.td       Neoverse V1 sched
                          (+ many more AArch64Sched*.td)

Instruction Selection     AArch64ISelDAGToDAG.cpp         SelectionDAG selector
(SelectionDAG)            AArch64ISelDAGToDAG.h
                          AArch64FastISel.cpp             Fast ISel (-O0)
                          AArch64InstrInfo.cpp / .h       Instruction utilities

Instruction Selection     GISel/AArch64InstructionSelector.cpp
(GlobalISel)              GISel/AArch64CallLowering.cpp / .h
                          GISel/AArch64LegalizerInfo.cpp / .h
                          GISel/AArch64RegisterBankInfo.cpp / .h
                          GISel/AArch64GlobalISelUtils.cpp / .h
                          GISel/AArch64PreLegalizerCombiner.cpp
                          GISel/AArch64O0PreLegalizerCombiner.cpp
                          GISel/AArch64PostLegalizerCombiner.cpp
                          GISel/AArch64PostLegalizerLowering.cpp
                          GISel/AArch64PostSelectOptimize.cpp

Pre-RegAlloc MIR          AArch64ExpandPseudoInsts.cpp    Pseudo expansion
                          AArch64ExpandImm.cpp / .h       Immediate expansion
                          AArch64A57FPLoadBalancing.cpp   A57 FP balancing
                          AArch64AdvSIMDScalarPass.cpp    SIMD scalar promo
                          AArch64SIMDInstrOpt.cpp         SIMD optimisation
                          AArch64StorePairSuppress.cpp    STP suppression
                          AArch64PBQPRegAlloc.cpp / .h    PBQP hints

Register Allocation       AArch64RegisterInfo.cpp / .h    Register constraints
                          AArch64CallingConvention.cpp / .h  CC implementation
                          AArch64MachineFunctionInfo.cpp  Per-function state
                          AArch64MachineFunctionInfo.h
                          AArch64GenRegisterBankInfo.def
                          AArch64SRLTDefineSuperRegs.cpp  Super-reg definitions

Post-RegAlloc MIR         AArch64PrologueEpilogue.cpp / .h  Prologue/epilogue
                          AArch64FrameLowering.cpp / .h   Frame layout
                          AArch64LowerHomogeneousPrologEpilog.cpp
                          AArch64PostCoalescerPass.cpp    Post-coalesce cleanup
                          AArch64LoadStoreOptimizer.cpp   LDP/STP formation
                          AArch64MachineScheduler.cpp / .h  Scheduling
                          AArch64MacroFusion.cpp / .h     Macro-fusion hints
                          AArch64CodeLayoutOpt.cpp        Block reordering
                          AArch64CondBrTuning.cpp         Branch tuning
                          AArch64ConditionalCompares.cpp  CCMP formation
                          AArch64ConditionOptimizer.cpp   Flag optimisation
                          AArch64RedundantCondBranchPass.cpp
                          AArch64RedundantCopyElimination.cpp
                          AArch64DeadRegisterDefinitionsPass.cpp
                          AArch64CollectLOH.cpp           Linker hints
                          AArch64CompressJumpTables.cpp   Jump table compress
                          AArch64CleanupLocalDynamicTLSPass.cpp
                          AArch64FalkorHWPFFix.cpp        Falkor prefetch fix
                          AArch64A53Fix835769.cpp         A53 erratum fix

Security Hardening        AArch64SLSHardening.cpp         SLS barriers
                          AArch64SpeculationHardening.cpp Spectre mitigations
                          AArch64PointerAuth.cpp / .h     PAC instructions
                          AArch64BranchTargets.cpp        BTI instructions

SME / SVE Late Passes     MachineSMEABIPass.cpp           SME ABI transitions
                          AArch64SMEAttributes.cpp / .h   SME function attrs
                          SMEPeepholeOpt.cpp              SME peephole

Pre-Emit Passes           AArch64MIPeepholeOpt.cpp        MI peephole
                          AArch64MovePrefixPass           SVE MOVPRFX
                          AArch64PerfectShuffle.cpp / .h  Shuffle tables
                          (BranchRelaxation via generic pass)

Emission                  AArch64AsmPrinter.cpp           MachineInstr→MCInst
                          AArch64MCInstLower.cpp / .h     Operand lowering
                          MCTargetDesc/AArch64MCCodeEmitter.cpp  Bit encoding
                          MCTargetDesc/AArch64AsmBackend.cpp     Fixups
                          MCTargetDesc/AArch64FixupKinds.h
                          MCTargetDesc/AArch64ELFObjectWriter.cpp
                          MCTargetDesc/AArch64ELFStreamer.cpp / .h
                          MCTargetDesc/AArch64MachObjectWriter.cpp
                          MCTargetDesc/AArch64WinCOFFObjectWriter.cpp
                          MCTargetDesc/AArch64WinCOFFStreamer.cpp / .h
                          MCTargetDesc/AArch64InstPrinter.cpp / .h
                          MCTargetDesc/AArch64MCAsmInfo.cpp / .h
                          MCTargetDesc/AArch64MCExpr.cpp
                          MCTargetDesc/AArch64MCTargetDesc.cpp / .h
                          MCTargetDesc/AArch64TargetStreamer.cpp / .h
                          MCTargetDesc/AArch64MCLFIRewriter.cpp / .h
                          MCTargetDesc/AArch64AddressingModes.h

Assembler / Disassembler  AsmParser/                      .s file parsing
                          Disassembler/                   binary → MCInst

Utilities                 Utils/                          Shared AArch64 utils
                          AArch64.h                       Pass declarations
                          AArch64FMV.td                   Function Multi-Versioning
                          AArch64PfmCounters.td           Perf counter defs
                          AArch64SystemOperands.td        System reg operands
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

## Files Not Yet Covered

These files exist in the source tree but are not discussed in detail elsewhere
in these notes yet. Brief descriptions for reference:

```
File                                  What it does
────────────────────────────────────  ──────────────────────────────────────────
AArch64PerfectShuffle.cpp / .h        Pre-computed lookup table for optimal
                                      NEON shuffle instruction sequences.
                                      Used during SIMD lowering.

AArch64FMV.td                         Function Multi-Versioning definitions.
                                      Allows one function to have multiple
                                      implementations selected at runtime
                                      based on CPU features (e.g. SVE vs NEON).

AArch64PfmCounters.td                 Maps LLVM scheduling resources to
                                      hardware performance counter events.
                                      Used by llvm-mca for cycle-accurate
                                      simulation.

AArch64SystemOperands.td              Defines named system register operands
                                      (SPSR_EL1, TTBR0_EL1, etc.) used by
                                      MRS/MSR instructions.

AArch64SchedPredicates.td             Scheduling predicates — conditions that
AArch64SchedPredNeoverse.td           must be true for a scheduling rule to
AArch64SchedPredExynos.td             apply (e.g. "is this a load that hits
                                      the L1 cache?").

AArch64SRLTDefineSuperRegs.cpp        Defines super-register relationships
                                      for the SRLT (Sub-Register Liveness
                                      Tracking) framework.

AArch64SelectionDAGInfo.cpp / .h      Provides AArch64-specific information
                                      to the SelectionDAG about memory
                                      operations (memcpy, memset, memmove).
                                      Can select optimised inline sequences.

MCTargetDesc/AArch64MCLFIRewriter     Rewrites LFI (Landing Pad / BTI)
                                      annotations in the MC layer. Ensures
                                      BTI instructions are correctly placed
                                      at indirect branch targets.

Utils/                                Shared utility code used across
                                      multiple AArch64 passes.

AsmParser/                            Parses AArch64 assembly text (.s files)
                                      into MCInst objects. Used by the
                                      integrated assembler.

Disassembler/                         Converts binary machine code back into
                                      MCInst objects. Used by llvm-objdump
                                      and debuggers.
```

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
