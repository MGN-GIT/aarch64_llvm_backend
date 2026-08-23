# SelectionDAG Deep Dive — From Optimized IR to Machine Instructions

> Companion to `02_llvm_compilation_pipeline.md`. That file gives the big
> picture of all five pipeline stages; this file zooms into **Stage 2** and
> answers, with exact function/file names from this checkout:
>
> - Who generates DAG nodes from LLVM IR?
> - What exactly happens in `AArch64ISelLowering.cpp`?
> - What happens after lowering (legalize/combine/schedule)?
> - What does `AArch64ISelDAGToDAG.cpp` do?
> - Who actually invokes the TableGen-generated patterns?
> - **Who drives all of this once the middle-end has produced optimized
>   IR, and who "hands" the DAG to the backend?**

---

## Table of Contents

- [0. Where This Fits After Optimization](#0-where-this-fits-after-optimization)
- [1. Who Creates the MachineFunction Container?](#1-who-creates-the-machinefunction-container)
- [2. Who Generates DAG Nodes From LLVM IR?](#2-who-generates-dag-nodes-from-llvm-ir)
- [3. What Happens in AArch64ISelLowering.cpp](#3-what-happens-in-aarch64iselloweringcpp)
- [4. What Happens After Lowering — CodeGenAndEmitDAG](#4-what-happens-after-lowering--codegenandemitdag)
- [5. What AArch64ISelDAGToDAG.cpp Does](#5-what-aarch64iseldagtodagcpp-does)
- [6. Who Actually Uses the TableGen Patterns?](#6-who-actually-uses-the-tablegen-patterns)
- [7. DAG Lifetime — It's Transient, Not Persistent](#7-dag-lifetime--its-transient-not-persistent)
- [8. Full Hand-off Table](#8-full-hand-off-table)

---

## 0. Where This Fits After Optimization

By the time codegen starts, the middle-end (`opt` pipeline, or clang's
`-O2` IR pass pipeline) has already finished and produced a **Module** full
of **optimized LLVM IR functions**. Codegen begins when a driver (`llc`, or
clang's backend action) calls into the target machine. Below is the *exact*
call chain, traced through the real source in this checkout — no invented
class names.

```
opt-level IR passes (InstCombine, GVN, LICM, Inliner, ...)
        │
        ▼
   Optimized LLVM IR  (a Module, still 100% target-independent)
        │
        ▼  llc.cpp: compileModule()
   Target->addPassesToEmitFile(PM, Out, ..., FileType, NoVerify, MMIWP)
        │        (Target is a TargetMachine* — for AArch64, an
        │         AArch64leTargetMachine/AArch64beTargetMachine instance)
        ▼
   CodeGenTargetMachineImpl::addPassesToEmitFile()      (generic — llvm/lib/CodeGen/CodeGenTargetMachineImpl.cpp)
        │  calls the static helper in the SAME file:
        ▼
   addPassesToGenerateCode(TM, PM, DisableVerify, MMIWP)  (generic, file-static helper)
        │
        ├─ PassConfig = TM.createPassConfig(PM)
        │      -> AArch64TargetMachine::createPassConfig()        (AArch64TargetMachine.cpp)
        │      -> returns `new AArch64PassConfig(*this, PM)`
        │
        ├─ PassConfig->addISelPasses()      <-- generic, TargetPassConfig.cpp, but calls virtual AArch64 hooks (see below)
        └─ PassConfig->addMachinePasses()   <-- generic, TargetPassConfig.cpp, calls virtual AArch64 hooks (addPreRegAlloc, addPostRegAlloc, addPreSched2, addPreEmitPass, ...)
```

`AArch64PassConfig` (defined in `AArch64TargetMachine.cpp`) is a subclass of
the generic `TargetPassConfig` (`llvm/lib/CodeGen/TargetPassConfig.cpp`).
`TargetPassConfig::addISelPasses()` is **not** overridden by AArch64 — it's
the generic driver that decides the overall order and, at specific points,
calls virtual hooks that AArch64 *does* override. Here is its real body
(`TargetPassConfig.cpp`):

```cpp
bool TargetPassConfig::addISelPasses() {
  if (TM->useEmulatedTLS())
    addPass(createLowerEmuTLSPass());

  PM->add(createTargetTransformInfoWrapperPass(TM->getTargetIRAnalysis()));
  if (getOptLevel() != CodeGenOptLevel::None)
    addPass(createObjCARCContractPass());
  addPass(createPreISelIntrinsicLoweringPass());
  addPass(createExpandIRInstsPass(getOptLevel()));

  addIRPasses();              // <-- virtual! dispatches to AArch64PassConfig::addIRPasses
  addCodeGenPrepare();        // <-- virtual! dispatches to AArch64PassConfig::addCodeGenPrepare
  addPassesToHandleExceptions();   // generic, NOT overridden by AArch64 (SjLj/DwarfEH/WinEH/Wasm/None)
  addISelPrepare();            // generic; internally calls addPreISel() -- see below

  return addCoreISelPasses();  // generic; decides FastISel/GlobalISel/SelectionDAG,
                               // and near the end calls addInstSelector() -- see below
}
```

`addISelPrepare()` (also generic, `TargetPassConfig.cpp`) is where
`addPreISel()` is actually invoked — it is **not** a sibling of
`addIRPasses()`, it's nested one level deeper:

```cpp
void TargetPassConfig::addISelPrepare() {
  addPreISel();                       // <-- virtual! dispatches to AArch64PassConfig::addPreISel
  if (requiresCodeGenSCCOrder()) addPass(new DummyCGSCCPass);
  addPass(createInlineAsmPreparePass());
  addPass(createSafeStackPass());
  addPass(createStackProtectorPass());
  ...
  addPass(createVerifierPass());      // IR verifier — last chance to catch a bad IR pass
}
```

And `addCoreISelPasses()` (generic) is where `addInstSelector()` is finally
invoked, near the very end, once FastISel/GlobalISel/SelectionDAG has been
chosen:

```cpp
bool TargetPassConfig::addCoreISelPasses() {
  // ... selects Selector = SelectionDAG / FastISel / GlobalISel ...
  if (Selector != SelectorType::GlobalISel || !isGlobalISelAbortEnabled())
    if (addInstSelector())            // <-- virtual! dispatches to AArch64PassConfig::addInstSelector
      return true;
  addPass(&FinalizeISelID);
  ...
}
```

### So, precisely: where is each hook called *for AArch64*?

All three hooks below live in `AArch64TargetMachine.cpp` as overrides on
`AArch64PassConfig`. None of them is called directly by AArch64 code —
AArch64 never calls `addISelPasses()` or `addISelPrepare()` itself. Instead,
the **generic** `TargetPassConfig` machinery calls these virtual methods
through the vtable, at the fixed points shown above:

| AArch64 override | Defined in | Invoked by (generic) | When |
|---|---|---|---|
| `AArch64PassConfig::addIRPasses()` | `AArch64TargetMachine.cpp` | `TargetPassConfig::addISelPasses()` | early, right after `ExpandIRInstsPass` |
| `AArch64PassConfig::addPreISel()` | `AArch64TargetMachine.cpp` | `TargetPassConfig::addISelPrepare()` | inside `addISelPrepare()`, before SafeStack/StackProtector |
| `AArch64PassConfig::addCodeGenPrepare()` | `AArch64TargetMachine.cpp` | `TargetPassConfig::addISelPasses()` | right after `addIRPasses()` returns |
| `AArch64PassConfig::addInstSelector()` | `AArch64TargetMachine.cpp` | `TargetPassConfig::addCoreISelPasses()` | last step of `addISelPasses()`, once selector kind is chosen |

Each override typically adds its own AArch64-specific passes and then (for
`addIRPasses`/`addCodeGenPrepare`) explicitly calls the base-class version to
still get the generic passes:

```cpp
// AArch64TargetMachine.cpp
void AArch64PassConfig::addIRPasses() {
  addPass(createAtomicExpandLegacyPass());
  if (EnableSVEIntrinsicOpts && TM->getOptLevel() != CodeGenOptLevel::None)
    addPass(createSVEIntrinsicOptsPass());
  ... // several more AArch64-only IR passes ...

  TargetPassConfig::addIRPasses();   // <-- don't forget the generic ones!
                                     //     (Verifier, AA passes, LSR, GC lowering,
                                     //      ConstantHoisting, ExpandReductions, ...)

  if (getOptLevel() == CodeGenOptLevel::Aggressive && EnableSelectOpt)
    addPass(createSelectOptimizePass());
  addPass(createAArch64StackTaggingPass(...));
  if (... EnableSVEShuffleOpt) addPass(createSVEShuffleOptsPass());
  if (TM->getOptLevel() >= CodeGenOptLevel::Default)
    addPass(createComplexDeinterleavingPass(TM));
  ...
}

bool AArch64PassConfig::addPreISel() {
  if (TM->getOptLevel() != CodeGenOptLevel::None && EnablePromoteConstant)
    addPass(createAArch64PromoteConstantPass());
  if (/* global merge enabled */)
    addPass(createGlobalMergePass(TM, 4095, ...));
  return false;
}

void AArch64PassConfig::addCodeGenPrepare() {
  if (getOptLevel() != CodeGenOptLevel::None)
    addPass(createTypePromotionLegacyPass());
  TargetPassConfig::addCodeGenPrepare();   // <-- adds the generic CodeGenPrepare pass
}

bool AArch64PassConfig::addInstSelector() {
  addPass(createAArch64ISelDag(getAArch64TargetMachine(), getOptLevel()));
  if (TM->getTargetTriple().isOSBinFormatELF() &&
      getOptLevel() != CodeGenOptLevel::None)
    addPass(createAArch64CleanupLocalDynamicTLSPass());
  return false;
}
```

Everything up through `addISelPrepare()` (i.e. `addIRPasses`, `addPreISel`,
`addCodeGenPrepare`, and exception-handling prep) runs **entirely on LLVM
IR** — no `MachineFunction` or DAG exists yet for any of it. The very first
`MachineFunctionPass` in the whole pipeline is the one added inside
`addInstSelector()`: `createAArch64ISelDag()` (declared/defined in
`AArch64ISelDAGToDAG.cpp`), which wraps `AArch64DAGToDAGISel` inside a
`SelectionDAGISelLegacy` **MachineFunctionPass**. This is the pass that,
once it actually runs (once per `Function`), is responsible for the entire
DAG-build → lower → legalize → select → schedule → emit sequence described
in the rest of this document.

---

## 1. Who Creates the MachineFunction Container?

`SelectionDAGISelLegacy` is a `MachineFunctionPass`, **not** a `FunctionPass`.
The generic LLVM `MachineFunctionPass` infrastructure guarantees that before
*any* `MachineFunctionPass` runs on a given IR `Function`, there is already
an (empty) `MachineFunction` object for it — one basic block skeleton per IR
basic block, no instructions yet. This object is produced by
`MachineFunctionAnalysis` (generic, target-independent), which is required
by the pass manager as soon as the first `MachineFunctionPass` (which is our
`AArch64DAGToDAGISelLegacy`) is scheduled.

So, in order:
1. `MachineFunctionAnalysis` creates an **empty `MachineFunction`** that
   mirrors the CFG shape of the IR `Function` (empty `MachineBasicBlock`s,
   correctly linked as CFG successors/predecessors).
2. `SelectionDAGISelLegacy::runOnMachineFunction(MF)` is invoked with this
   empty container **and** a reference to the original IR `Function`
   (`MF.getFunction()`).
3. Everything from here on — DAG building, lowering, legalizing, selecting,
   scheduling — happens *inside this single pass invocation*, and the result
   (real `MachineInstr`s) is inserted directly into the `MachineBasicBlock`s
   of that same `MachineFunction`.

This is an important point: **the DAG is not some separate IR that flows
between passes.** It is built, consumed, and destroyed entirely inside the
`AArch64DAGToDAGISel` (well, generic `SelectionDAGISel`) pass, one function
at a time, one basic block at a time. See [§7](#7-dag-lifetime--its-transient-not-persistent).

---

## 2. Who Generates DAG Nodes From LLVM IR?

`SelectionDAGISel::runOnMachineFunction()` (generic,
`llvm/lib/CodeGen/SelectionDAG/SelectionDAGISel.cpp`) calls
`SelectAllBasicBlocks(Fn)`, which iterates the IR function's basic blocks in
reverse-post-order and, for each one, calls:

```cpp
// SelectionDAGISel.cpp
void SelectionDAGISel::SelectBasicBlock(...) {
  for (BasicBlock::const_iterator I = Begin; I != End && !SDB->HasTailCall; ++I)
    SDB->visit(*I);                       // <-- walks every IR instruction

  CurDAG->setRoot(SDB->getControlRoot());
  ...
  CodeGenAndEmitDAG();                    // <-- everything downstream happens here
}
```

`SDB` is a `SelectionDAGBuilder` (`llvm/lib/CodeGen/SelectionDAG/SelectionDAGBuilder.cpp`,
target-independent). It has one `visit*` method per IR instruction kind
(`visitAdd`, `visitLoad`, `visitCall`, `visitGetElementPtr`, `visitBr`, ...),
each of which calls `DAG.getNode(...)` to build generic `ISD::` opcodes.
This builder doesn't know anything about AArch64 — it only starts touching
the target through `TargetLowering` hooks for arguments/calls/returns
(`LowerFormalArguments`, `LowerCall`, `LowerReturn`), which is the very first
point AArch64-specific code gets involved.

---

## 3. What Happens in `AArch64ISelLowering.cpp`

`AArch64TargetLowering` (subclass of the generic `TargetLowering`) is
consulted in two ways:

**a) ABI / calling-convention lowering** — used *while* the DAG builder is
running:
- `LowerFormalArguments()` — incoming AAPCS64 registers/stack → DAG values.
- `LowerCall()` — builds the call sequence with correct calling convention.
- `LowerReturn()` — builds the return sequence.

**b) Custom operation lowering** — for any generic `ISD::` opcode marked
`Custom` via `setOperationAction(..., Custom)` in the `AArch64TargetLowering`
constructor. Whenever the legalizer (see next section) encounters one of
these, it calls:

```cpp
SDValue AArch64TargetLowering::LowerOperation(SDValue Op, SelectionDAG &DAG) const {
  switch (Op.getOpcode()) {
  case ISD::GlobalAddress:   return LowerGlobalAddress(Op, DAG);
  case ISD::SETCC:           return LowerSETCC(Op, DAG);
  case ISD::SELECT_CC:       return LowerSELECT_CC(Op, DAG);
  case ISD::BR_CC:           return LowerBR_CC(Op, DAG);
  ...
  }
}
```

This turns generic IR-ish concepts into **`AArch64ISD::`** target-specific
pseudo-nodes (`AArch64ISD::ADDlow`, `AArch64ISD::CSEL`, `AArch64ISD::CALL`,
...) that map naturally onto real instructions later.

`AArch64TargetLowering::PerformDAGCombine()` is the target's hook into the
generic DAG combiner, letting AArch64 do its own peephole-style DAG rewrites
beyond what the target-independent combiner already knows.

**Important:** `ISelLowering` never picks real machine opcodes. It only
makes the DAG legal (legal types + legal ops). The DAG at this point still
consists of `SDValue`/`ISD::`/`AArch64ISD::` nodes — not `MachineSDNode`s.

---

## 4. What Happens After Lowering — `CodeGenAndEmitDAG`

`SelectionDAGISel::CodeGenAndEmitDAG()` (generic,
`llvm/lib/CodeGen/SelectionDAG/SelectionDAGISel.cpp`) is the orchestrator
for everything from "raw DAG" to "scheduled MachineInstrs in the block":

```cpp
void SelectionDAGISel::CodeGenAndEmitDAG() {
  CurDAG->Combine(BeforeLegalizeTypes, ...);   // DAG combine #1
  CurDAG->LegalizeTypes();                     // type legalization
  CurDAG->Combine(AfterLegalizeTypes, ...);     // DAG combine #2
  CurDAG->LegalizeVectors();
  CurDAG->LegalizeTypes();
  CurDAG->Combine(AfterLegalizeVectorOps, ...);
  CurDAG->Legalize();                           // op legalization (calls LowerOperation)
  CurDAG->Combine(AfterLegalizeDAG, ...);        // DAG combine #3

  DoInstructionSelection();    // <-- AArch64ISelDAGToDAG.cpp takes over here

  ScheduleDAGSDNodes *Scheduler = CreateScheduler();
  Scheduler->Run(CurDAG, FuncInfo->MBB);
  FuncInfo->MBB = Scheduler->EmitSchedule(...);  // emits real MachineInstrs
}
```

Order: **combine → legalize types → combine → legalize vectors → legalize
ops (may re-enter `LowerOperation`) → combine → instruction-select →
schedule → emit MIR.** All of this is generic/target-independent machinery;
AArch64 only supplies the legality tables and the combine/select callbacks.

---

## 5. What `AArch64ISelDAGToDAG.cpp` Does

`DoInstructionSelection()` (generic) walks the now-legal DAG in reverse
topological order and calls the target's `Select()` on every node:

```cpp
void SelectionDAGISel::DoInstructionSelection() {
  PreprocessISelDAG();                 // AArch64-specific pre-pass
  while (ISelPosition != CurDAG->allnodes_begin()) {
    SDNode *Node = &*--ISelPosition;
    if (Node->use_empty()) continue;
    Select(Node);                      // <-- virtual dispatch
  }
  PostprocessISelDAG();
}
```

`AArch64DAGToDAGISel::Select()`:

```cpp
void AArch64DAGToDAGISel::Select(SDNode *Node) {
  if (Node->isMachineOpcode()) { Node->setNodeId(-1); return; }

  switch (Node->getOpcode()) {
    case ISD::ATOMIC_CMP_SWAP:        return SelectCMP_SWAP(Node);
    case AArch64ISD::PTRAUTH_AUTH:    return SelectPtrauthAuth(Node);
    // ...dozens of hand-written cases: SVE multi-vector loads/stores,
    //    register-tuple formation, pointer authentication, bitfield
    //    extract/insert, indexed loads, SME tile selection, etc.
  }

  SelectCode(Node);   // <----- TableGen-generated pattern matcher!
}
```

So this file has two jobs:
1. **Hand-written C++ selection** for patterns too complex/multi-output for
   the declarative TableGen matcher (SVE loads, register tuples, pointer
   auth) plus **`ComplexPattern` callback functions** referenced by name
   from `.td` files (`SelectAddrModeIndexed`, `SelectArithImmed`,
   `SelectShiftedRegister`, ...).
2. **Falls back to `SelectCode(Node)`** for everything expressible as a
   simple declarative tree pattern.

---

## 6. Who Actually Uses the TableGen Patterns?

At the bottom of the class:

```cpp
// AArch64ISelDAGToDAG.cpp
#include "AArch64GenDAGISel.inc"
```

Generated by (`CMakeLists.txt`):

```
tablegen(LLVM AArch64GenDAGISel.inc -gen-dag-isel)
```

`llvm-tblgen -gen-dag-isel` scans every `def Pat<...>` and every
`Instruction`'s `Pattern` field across `AArch64InstrInfo.td`,
`AArch64InstrFormats.td`, `AArch64SVEInstrInfo.td`,
`AArch64InstrAtomics.td`, etc., and emits:

- **`SelectCode(SDNode *N)`** — an opcode-indexed byte-code matcher that
  walks the DAG, matches every `Pattern`/`Pat`, and replaces matched
  subtrees with a `MachineSDNode` for the matching `Instruction` record.
- Trampoline calls into `ComplexPattern` callbacks — this is exactly why
  `AArch64DAGToDAGISel` has methods like `SelectArithImmed`. In
  `AArch64InstrInfo.td`:
  ```tablegen
  def addsub_shifted_imm : ComplexPattern<i32, 2, "SelectArithImmed", []>;
  ```
  TableGen emits a direct call to `SelectArithImmed()` inside the
  generated `SelectCode()`.

So the chain is:

```
.td Pattern/Pat/ComplexPattern defs
        │  llvm-tblgen -gen-dag-isel
        ▼
AArch64GenDAGISel.inc  (defines SelectCode())
        │  #include'd into
        ▼
AArch64DAGToDAGISel  (class body)
        │  called from
        ▼
AArch64DAGToDAGISel::Select()
        │  called from
        ▼
SelectionDAGISel::DoInstructionSelection()   (generic)
```

Nobody else calls `SelectCode()` — it's a private implementation detail of
each target's `Select()` override.

---

## 7. DAG Lifetime — It's Transient, Not Persistent

Unlike LLVM IR (which persists across the whole optimization pipeline as
`Module`/`Function`), the **SelectionDAG has no persistent existence between
passes**:

- One `SelectionDAG` object (`CurDAG`) is owned by the `SelectionDAGISel`
  pass itself, reused for every basic block of every function it processes.
- It is (re)built fresh **per basic block** inside `SelectBasicBlock()`.
- It is fully consumed — combined, legalized, selected, scheduled, and
  lowered to `MachineInstr`s — before the next basic block starts.
- At the end of `CodeGenAndEmitDAG()`, `CurDAG->clear()` wipes it.

So there is exactly **one pass** in the whole `AArch64PassConfig` pipeline
that ever sees a `SelectionDAG`: the instruction-selection
`MachineFunctionPass` added by `addInstSelector()`. Every pass before it
only ever sees IR; every pass after it (`addPreRegAlloc`, `addPostRegAlloc`,
`addPreSched2`, `addPreEmitPass`, ...) only ever sees `MachineInstr`s/MIR.
The DAG is purely an internal implementation detail of that one pass.

---

## 8. Full Hand-off Table

| Boundary | Who hands off | Who receives | Representation crossing the boundary |
|---|---|---|---|
| Optimizer → Codegen driver | `opt`/clang IR pipeline | `llc.cpp: compileModule()` → `TargetMachine::addPassesToEmitFile` | Optimized `Module` (LLVM IR) |
| Codegen driver → Pass pipeline builder | `CodeGenTargetMachineImpl::addPassesToEmitFile` | file-static `addPassesToGenerateCode()` → `AArch64PassConfig` (via `createPassConfig`) → `TargetPassConfig::addISelPasses`/`addMachinePasses` | still IR, now inside a `PassManager` |
| IR passes → MachineFunction creation | `TargetPassConfig` scheduling (first `MachineFunctionPass` added, i.e. the one from `addInstSelector`) | `MachineFunctionAnalysis` (generic) | Empty `MachineFunction` (CFG skeleton only) |
| Container → DAG builder | `SelectionDAGISelLegacy::runOnMachineFunction` | `SelectionDAGBuilder::visit()` (generic) | IR `Instruction`s → generic `ISD::` `SDNode`s |
| DAG builder → Target lowering | `SelectionDAGBuilder` (via `TargetLowering` hooks) | `AArch64TargetLowering::LowerFormalArguments/LowerCall/LowerReturn/LowerOperation` | generic `ISD::` ↔ `AArch64ISD::` `SDNode`s |
| Lowering → Combine/Legalize | `AArch64TargetLowering` | `SelectionDAG::Combine/LegalizeTypes/Legalize` (generic) | legal, target-aware DAG |
| Legal DAG → Instruction selection | `SelectionDAGISel::DoInstructionSelection` (generic) | `AArch64DAGToDAGISel::Select()` → `SelectCode()` (TableGen) | `SDNode`s → `MachineSDNode`s (real AArch64 opcodes) |
| Selected DAG → Scheduler | `SelectionDAGISel::CodeGenAndEmitDAG` | `ScheduleDAGSDNodes::Run/EmitSchedule` (generic, AArch64 supplies scheduling model via `.td`) | `MachineSDNode`s → `MachineInstr`s |
| MIR → rest of MIR pipeline | `SelectionDAGISelLegacy` pass finishes | All later `MachineFunctionPass`es (RA, `AArch64LoadStoreOptimizer`, `AArch64ExpandPseudoInsts`, scheduling, `AArch64AsmPrinter`, ...) | `MachineInstr`s only, DAG is gone |

**One-line summary:** the optimizer hands off a `Module` of optimized IR to
`TargetPassConfig`; the very first `MachineFunctionPass` it schedules
(`AArch64DAGToDAGISelLegacy`, created by `createAArch64ISelDag`) is the
*only* place in the entire backend where a `SelectionDAG` is built at all —
it builds one per basic block, lowers/combines/legalizes it via
`AArch64ISelLowering.cpp` + generic `SelectionDAG` machinery, selects real
AArch64 instructions via `AArch64ISelDAGToDAG.cpp`'s hand-written code and
its TableGen-generated `SelectCode()`, schedules them, emits `MachineInstr`s
into the `MachineFunction`, and then throws the DAG away — forever — before
moving to the next basic block.
