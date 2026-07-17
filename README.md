# LLVM AArch64 Backend — Contributor Learning Roadmap

> A structured guide from **basic → advanced** concepts needed to contribute to the LLVM AArch64 backend.
> Built from a thorough analysis of every major file in this codebase.

---

## Table of Contents

- [Level 1 — Foundations](#-level-1--foundations-prerequisites-before-touching-any-code)
  - [1.1 AArch64 ISA Fundamentals](#11--aarch64-isa-fundamentals)
  - [1.2 LLVM IR Basics](#12--llvm-ir-basics)
  - [1.3 LLVM Compilation Pipeline](#13--llvm-compilation-pipeline-big-picture)
- [Level 2 — Core Backend Infrastructure](#-level-2--core-backend-infrastructure)
  - [2.1 Target Machine & Subtarget](#21--target-machine--subtarget)
  - [2.2 TableGen & Target Description Files](#22--tablegen--target-description-files)
  - [2.3 Register Information](#23--register-information)
  - [2.4 Instruction Information](#24--instruction-information)
- [Level 3 — Instruction Selection](#-level-3--instruction-selection)
  - [3.1 SelectionDAG Concepts](#31--selectiondag-concepts)
  - [3.2 DAG-to-DAG Instruction Selection](#32--dag-to-dag-instruction-selection)
  - [3.3 Fast Instruction Selection (FastISel)](#33--fast-instruction-selection-fastisel)
  - [3.4 GlobalISel (Modern Instruction Selection)](#34--globalisel-modern-instruction-selection)
- [Level 4 — ABI, Calling Conventions & Frame Management](#-level-4--abi-calling-conventions--frame-management)
  - [4.1 Calling Conventions](#41--calling-conventions)
  - [4.2 Frame Lowering & Stack Layout](#42--frame-lowering--stack-layout)
  - [4.3 Machine Function Info](#43--machine-function-info)
- [Level 5 — Optimization Passes](#-level-5--optimization-passes)
  - [5.1 Pre-RA Optimization Passes](#51--pre-ra-optimization-passes)
  - [5.2 Post-RA Optimization Passes](#52--post-ra-optimization-passes)
  - [5.3 Security Hardening Passes](#53--security-hardening-passes)
- [Level 6 — Advanced Vector Extensions](#-level-6--advanced-vector-extensions)
  - [6.1 NEON (Advanced SIMD)](#61--neon-advanced-simd)
  - [6.2 SVE (Scalable Vector Extension)](#62--sve-scalable-vector-extension)
  - [6.3 SME (Scalable Matrix Extension)](#63--sme-scalable-matrix-extension)
- [Level 7 — Machine Code Emission & Scheduling](#-level-7--machine-code-emission--scheduling)
  - [7.1 MC Layer (Machine Code)](#71--mc-layer-machine-code)
  - [7.2 Machine Scheduling](#72--machine-scheduling)
  - [7.3 Disassembler](#73--disassembler)
- [Level 8 — Expert-Level Topics](#-level-8--expert-level-topics)
  - [8.1 Register Allocation Interaction](#81--register-allocation-interaction)
  - [8.2 Machine Outliner](#82--machine-outliner)
  - [8.3 Target Transform Info (TTI)](#83--target-transform-info-tti)
  - [8.4 TLS (Thread-Local Storage)](#84--tls-thread-local-storage)
  - [8.5 Pointer Authentication (PAC/BTI)](#85--pointer-authentication-pacbti)
- [Recommended Study Sequence](#-recommended-study-sequence)
- [Key Files to Read First](#-key-files-to-read-first-in-order)

---

## 🟢 Level 1 — Foundations (Prerequisites Before Touching Any Code)

These are non-negotiable prerequisites. Without them, nothing in the backend will make sense.

---

### 1.1 — AArch64 ISA Fundamentals

**Why:** Every file in this codebase assumes you know the hardware.

| Concept | Where You'll See It |
|---|---|
| Register file: `X0–X30`, `W0–W30`, `SP`, `XZR`, `LR`, `FP` | `AArch64RegisterInfo.td`, `AArch64RegisterInfo.h` |
| NEON registers: `V0–V31`, `D`, `S`, `H`, `B` sub-registers | `AArch64RegisterInfo.td`, `AArch64InstrInfo.h` |
| SVE registers: `Z0–Z31` (scalable), `P0–P15` (predicates) | `AArch64SVEInstrInfo.td`, `AArch64Subtarget.h` |
| SME registers: ZA tile array, ZT0 | `AArch64SMEInstrInfo.td`, `AArch64MachineFunctionInfo.h` |
| Instruction encoding: fixed 32-bit width | `AArch64InstrFormats.td` |
| Condition flags: `NZCV` | `AArch64InstrInfo.h` (`UsedNZCV` struct) |
| Addressing modes: base+imm, base+reg, pre/post-index | `AArch64ISelDAGToDAG.cpp` (`SelectAddrModeIndexed`, `SelectAddrModeWRO`) |
| Calling convention: AAPCS64 | `AArch64CallingConvention.td/.h` |

> **Study:** ARM Architecture Reference Manual (ARMv8/ARMv9), AAPCS64 spec.

---

### 1.2 — LLVM IR Basics

**Why:** The backend's job is to lower LLVM IR to machine code. You must know what you're lowering *from*.

| Concept | Where You'll See It |
|---|---|
| `Function`, `BasicBlock`, `Instruction`, `Value` | `AArch64ISelLowering.cpp` |
| Types: `i8`, `i32`, `i64`, `float`, `<4 x i32>` | `AArch64ISelLowering.h` |
| IR intrinsics (`llvm.aarch64.*`) | `AArch64ISelDAGToDAG.cpp` |
| `DataLayout` (endianness, pointer size) | `AArch64TargetMachine.cpp`, `AArch64Subtarget.h` |
| `Function` attributes (`target-cpu`, `target-features`, `aarch64_pstate_sm_enabled`) | `AArch64TargetMachine.cpp` → `getSubtargetImpl()` |

---

### 1.3 — LLVM Compilation Pipeline (Big Picture)

**Why:** You need to know *where* each file fits in the pipeline.

```
LLVM IR
   │
   ▼  [IR Passes: AArch64ISelLowering, SVEIntrinsicOpts, etc.]
SelectionDAG (or GlobalISel)
   │
   ▼  [Instruction Selection: AArch64ISelDAGToDAG / GISel/]
MachineInstr (MIR)
   │
   ▼  [Register Allocation, Scheduling, Peephole, etc.]
MCInst
   │
   ▼  [AArch64AsmPrinter, MCTargetDesc/]
Object File / Assembly
```

> **Key file:** `AArch64TargetMachine.cpp` — the `AArch64PassConfig` class shows the **exact ordered list** of every pass in the pipeline.

---

## 🔵 Level 2 — Core Backend Infrastructure

These are the foundational classes every backend must implement.

---

### 2.1 — Target Machine & Subtarget

**Files:** `AArch64TargetMachine.h/.cpp`, `AArch64Subtarget.h/.cpp`

| Concept | Key Detail |
|---|---|
| `TargetMachine` is the root object | `AArch64leTargetMachine` / `AArch64beTargetMachine` for endianness |
| `Subtarget` is **per-function** | `getSubtargetImpl(const Function &F)` — different CPUs per function via attributes |
| Feature bits | Auto-generated from `AArch64Features.td` via `GET_SUBTARGETINFO_MACRO` macros |
| CPU families | `ARMProcFamilyEnum` — Cortex-A53, Apple M-series, Neoverse, etc. |
| Code model | `Small`, `Tiny`, `Large` — affects addressing mode choices |
| Relocation model | `PIC_`, `Static` — affects global address lowering |
| SVE vector size range | `MinSVEVectorSizeInBits` / `MaxSVEVectorSizeInBits` — critical for scalable vectorization |

> **Key insight:** The subtarget is the central hub — it owns `AArch64InstrInfo`, `AArch64RegisterInfo`, `AArch64FrameLowering`, `AArch64TargetLowering`, and all GlobalISel components.

---

### 2.2 — TableGen & Target Description Files

**Files:** `AArch64.td`, `AArch64InstrInfo.td`, `AArch64InstrFormats.td`, `AArch64RegisterInfo.td`, `AArch64Features.td`, `AArch64Processors.td`

| Concept | Key Detail |
|---|---|
| `def` / `class` / `multiclass` | Building blocks of all `.td` files |
| `RegisterClass` | Groups registers by type (GPR64, FPR128, ZPR, PPR) |
| `SubtargetFeature` | Each CPU feature (e.g., `FeatureSVE`, `FeatureNEON`) |
| `Processor` / `ProcessorModel` | Maps CPU names to feature sets and scheduling models |
| `Instruction` records | Encode opcode, operands, encoding, scheduling info |
| `ComplexPattern` | Used in `AArch64ISelDAGToDAG.cpp` for addressing mode selection |
| `SDNode` / `SDNodeXForm` | Custom DAG nodes for AArch64-specific operations |
| Auto-generated `.inc` files | `AArch64GenInstrInfo.inc`, `AArch64GenRegisterInfo.inc`, etc. |

> **Key insight:** TableGen generates ~80% of the boilerplate. Understanding `.td` files is mandatory — they define what instructions exist and how they're selected.

---

### 2.3 — Register Information

**Files:** `AArch64RegisterInfo.h/.cpp`, `AArch64RegisterInfo.td`

| Concept | Key Detail |
|---|---|
| Physical vs virtual registers | Virtual regs exist pre-RA; physical regs after |
| Register classes | `GPR32`, `GPR64`, `FPR64`, `ZPR`, `PPR`, `ZPR2`, `ZPR4` tuples |
| Sub-registers | `W` regs are `sub_32` of `X` regs; `B/H/S/D` are sub-regs of `Q` |
| Reserved registers | `SP`, `XZR`, `FP`, `LR` — `getReservedRegs()` |
| Callee-saved registers | `getCalleeSavedRegs()` — differs by calling convention |
| Frame register | `getFrameRegister()` — returns `FP` (X29) or `SP` |
| Register scavenging | `requiresRegisterScavenging()` — needed for large frames |
| Allocation hints | `getRegAllocationHints()` — guides RA for better code |

---

### 2.4 — Instruction Information

**Files:** `AArch64InstrInfo.h/.cpp`

| Concept | Key Detail |
|---|---|
| `TargetInstrInfo` subclass | Provides all instruction-level queries |
| `copyPhysReg()` | How to emit a register copy (MOV, ORR, FMOV, etc.) |
| `storeRegToStackSlot()` / `loadRegFromStackSlot()` | Spill/reload code generation |
| `analyzeBranch()` / `insertBranch()` / `removeBranch()` | Branch manipulation for CFG transforms |
| `expandPostRAPseudo()` | Expands pseudo-instructions after register allocation |
| `getMachineCombinerPatterns()` | Machine combiner patterns (FMLA, MADD, etc.) |
| `getOutliningCandidateInfo()` | Machine outliner support |
| `AArch64MachineCombinerPattern` enum | All SIMD/FP fusion patterns (FMLA, MULADDW, etc.) |
| TSFlags | Per-instruction flags: `ElementSizeType`, `DestructiveInstType`, `SMEMatrixType` |

---

## 🟡 Level 3 — Instruction Selection

This is the heart of the backend — translating IR/DAG to machine instructions.

---

### 3.1 — SelectionDAG Concepts

**Files:** `AArch64ISelLowering.h/.cpp`, `AArch64SelectionDAGInfo.h/.cpp`

| Concept | Key Detail |
|---|---|
| `SDValue` / `SDNode` | Nodes in the DAG; values are typed |
| `ISD::` opcodes | Standard DAG opcodes (ADD, LOAD, BR_CC, etc.) |
| `AArch64ISD::` opcodes | Target-specific DAG nodes (ADDlow, CSEL, FMOV, etc.) |
| `LowerOperation()` | Entry point for custom lowering of ISD nodes |
| `PerformDAGCombine()` | DAG-level peephole optimizations |
| `LegalizeType` / `LegalizeOp` | Making illegal types/ops legal for the target |
| `MVT` / `EVT` | Machine value types (i32, v4i32, nxv4i32, etc.) |
| `CCValAssign` | Calling convention value assignment |

**Key methods to study in `AArch64ISelLowering.cpp`:**

| Method | What It Does |
|---|---|
| `LowerFormalArguments()` | How function arguments are received |
| `LowerCall()` | How function calls are emitted |
| `LowerReturn()` | How return values are passed |
| `LowerGlobalAddress()` | ADRP+ADD, GOT, TLS patterns |
| `LowerSELECT_CC()` | CSEL, CSINC, CSINV, CSNEG patterns |

---

### 3.2 — DAG-to-DAG Instruction Selection

**File:** `AArch64ISelDAGToDAG.cpp`

| Concept | Key Detail |
|---|---|
| `SelectionDAGISel` subclass | `AArch64DAGToDAGISel` |
| `Select(SDNode*)` | Main dispatch function — handles special cases |
| `#include "AArch64GenDAGISel.inc"` | TableGen-generated pattern matching |
| `ComplexPattern` selectors | `SelectArithImmed`, `SelectAddrModeIndexed`, `SelectAddrModeWRO/XRO` |
| Addressing mode selection | Indexed (scaled 12-bit), Unscaled (9-bit signed), Register+Register |
| Bitfield operations | `tryBitfieldExtractOp()`, `tryBitfieldInsertOp()` — UBFM/SBFM/BFM |
| Indexed loads | `tryIndexedLoad()` — pre/post-increment LDR |
| SVE addressing | `SelectAddrModeIndexedSVE`, `SelectSVERegRegAddrMode` |
| SME tile selection | `SelectSMETileSlice()`, `SelectSMETile()` |
| Pointer authentication | `SelectPtrauthAuth()`, `SelectPtrauthResign()` |

---

### 3.3 — Fast Instruction Selection (FastISel)

**File:** `AArch64FastISel.cpp`

| Concept | Key Detail |
|---|---|
| Used at `-O0` for speed | Bypasses full SelectionDAG |
| Handles common patterns only | Falls back to DAG for complex cases |
| `fastLowerArguments()` | Fast path for argument lowering |
| `fastLowerCall()` | Fast path for call lowering |

---

### 3.4 — GlobalISel (Modern Instruction Selection)

**Directory:** `GISel/`

| Concept | Key Detail |
|---|---|
| Enabled at `-O0` by default | `EnableGlobalISelAtO = 0` in `AArch64TargetMachine.cpp` |
| `IRTranslator` | Converts IR to generic MachineInstrs (G_ADD, G_LOAD, etc.) |
| `Legalizer` | Makes generic ops legal for AArch64 |
| `RegBankSelect` | Assigns register banks (GPR vs FPR) |
| `InstructionSelect` | Selects real AArch64 instructions |
| Pre/Post legalizer combiners | `AArch64PreLegalizerCombiner`, `AArch64PostLegalizerCombiner` |
| `AArch64Combine.td` | TableGen rules for GISel combiners |
| `AArch64RegisterBanks.td` | Defines GPR and FPR register banks |

---

## 🟠 Level 4 — ABI, Calling Conventions & Frame Management

---

### 4.1 — Calling Conventions

**Files:** `AArch64CallingConvention.td/.h/.cpp`

| Concept | Key Detail |
|---|---|
| AAPCS64 | Standard AArch64 ABI — `CC_AArch64_AAPCS` |
| Darwin PCS | Apple's variant — `CC_AArch64_DarwinPCS` |
| Win64 PCS | Windows ABI — `CC_AArch64_Win64PCS` |
| GHC calling convention | Haskell runtime — `CC_AArch64_GHC` |
| Arm64EC | Windows ARM64EC thunk calling convention |
| SVE vector call | `CallingConv::AArch64_SVE_VectorCall` |
| `CCAssignFnForCall()` | Selects the right CC function |
| Varargs handling | `saveVarArgRegisters()`, `LowerVASTART()` |
| Tail call optimization | `isEligibleForTailCallOptimization()` |

---

### 4.2 — Frame Lowering & Stack Layout

**Files:** `AArch64FrameLowering.h/.cpp`, `AArch64PrologueEpilogue.cpp`

| Concept | Key Detail |
|---|---|
| Stack grows downward | `StackGrowsDown`, 16-byte aligned |
| Frame record | `[FP, LR]` pair saved at top of frame |
| `emitPrologue()` / `emitEpilogue()` | Generate function entry/exit code |
| Callee-saved registers | `spillCalleeSavedRegisters()` — STP pairs |
| SVE stack regions | Separate ZPR and PPR stack areas (`getZPRStackSize()`, `getPPRStackSize()`) |
| Red zone | 128-byte area below SP for leaf functions |
| Frame index elimination | `eliminateFrameIndex()` — replaces FI with SP/FP+offset |
| Shrink wrapping | `enableShrinkWrapping()` — moves prologue/epilogue to optimal locations |
| Stack probing | `inlineStackProbe()` — Windows stack guard pages |
| Homogeneous prolog/epilog | `AArch64LowerHomogeneousPrologEpilog.cpp` — size optimization |

---

### 4.3 — Machine Function Info

**File:** `AArch64MachineFunctionInfo.h/.cpp`

| Concept | Key Detail |
|---|---|
| Per-function AArch64 state | Extends `MachineFunctionInfo` |
| Stack argument tracking | `BytesInStackArgArea`, `ArgumentStackToRestore` |
| Varargs frame indices | `VarArgsGPRIndex`, `VarArgsFPRIndex` |
| LOH directives | `MILOHContainer` — Linker Optimization Hints for Mach-O |
| PAC-RET signing | `SignCondition`, `SignWithBKey`, `SignInstrLabel` |
| SME state | `SMEFnAttrs`, `PStateSMReg`, `HasStreamingModeChanges` |
| SVE stack sizes | `StackSizeZPR`, `StackSizePPR` |
| Jump table info | `JumpTableEntryInfo` — compressed jump tables |

---

## 🔴 Level 5 — Optimization Passes

These are the AArch64-specific optimization passes that run throughout the pipeline.

---

### 5.1 — Pre-RA Optimization Passes

| Pass File | What It Does |
|---|---|
| `AArch64PromoteConstant.cpp` | Promotes constants to global variables to reduce code size |
| `AArch64ConditionOptimizer.cpp` | Optimizes condition code sequences |
| `AArch64ConditionalCompares.cpp` | Forms `CCMP`/`CCMN` instruction chains |
| `AArch64CondBrTuning.cpp` | Converts `CMP+Bcc` to `CBZ`/`CBNZ`/`TBZ`/`TBNZ` |
| `AArch64AdvSIMDScalarPass.cpp` | Uses scalar SIMD instructions when profitable |
| `AArch64DeadRegisterDefinitionsPass.cpp` | Replaces dead defs with `XZR`/`WZR` |
| `AArch64MIPeepholeOpt.cpp` | Machine-level peephole optimizations |
| `AArch64SIMDInstrOpt.cpp` | SIMD instruction optimizations (interleaving, etc.) |
| `AArch64StorePairSuppress.cpp` | Suppresses STP formation when unprofitable |
| `AArch64StackTagging.cpp` | MTE (Memory Tagging Extension) stack tagging |

---

### 5.2 — Post-RA Optimization Passes

| Pass File | What It Does |
|---|---|
| `AArch64LoadStoreOptimizer.cpp` | Forms LDP/STP pairs from adjacent loads/stores |
| `AArch64RedundantCopyElimination.cpp` | Removes redundant register copies |
| `AArch64RedundantCondBranchPass.cpp` | Removes redundant conditional branches |
| `AArch64A57FPLoadBalancing.cpp` | Balances FP register usage on Cortex-A57 |
| `AArch64PostCoalescerPass.cpp` | Post-coalescer cleanup |
| `AArch64ExpandPseudoInsts.cpp` | Expands pseudo-instructions to real ones |
| `AArch64CompressJumpTables.cpp` | Uses smallest possible jump table entries |
| `AArch64CollectLOH.cpp` | Collects Linker Optimization Hints (Mach-O only) |
| `AArch64CodeLayoutOpt.cpp` | Code layout optimizations |

---

### 5.3 — Security Hardening Passes

| Pass File | What It Does |
|---|---|
| `AArch64PointerAuth.cpp` | PAC (Pointer Authentication Code) instrumentation |
| `AArch64BranchTargets.cpp` | BTI (Branch Target Identification) insertion |
| `AArch64SLSHardening.cpp` | Straight-Line Speculation hardening |
| `AArch64SpeculationHardening.cpp` | Spectre-v1 mitigation |
| `AArch64StackTagging.cpp` | MTE stack tagging for memory safety |
| `AArch64A53Fix835769.cpp` | Cortex-A53 erratum 835769 workaround |

---

## 🟣 Level 6 — Advanced Vector Extensions

---

### 6.1 — NEON (Advanced SIMD)

**Files:** `AArch64InstrInfo.td`, `AArch64ISelLowering.cpp`

| Concept | Key Detail |
|---|---|
| Fixed-length 64/128-bit vectors | `v8i8`, `v4i16`, `v2i32`, `v1i64`, `v16i8`, `v8i16`, `v4i32`, `v2i64` |
| `addTypeForNEON()` | Registers all NEON vector types as legal |
| Shuffle lowering | `LowerVECTOR_SHUFFLE()`, `AArch64PerfectShuffle.cpp` |
| Interleaved access | `lowerInterleavedLoad/Store()` — LD2/ST2/LD3/ST3/LD4/ST4 |
| NEON intrinsics | Mapped via `LowerINTRINSIC_WO_CHAIN()` |
| `AArch64AdvSIMDScalarPass` | Promotes scalar ops to SIMD when beneficial |

---

### 6.2 — SVE (Scalable Vector Extension)

**Files:** `AArch64SVEInstrInfo.td`, `SVEInstrFormats.td`, `AArch64ISelLowering.cpp`

| Concept | Key Detail |
|---|---|
| Scalable types | `nxv16i8`, `nxv4i32`, `nxv2f64` — size unknown at compile time |
| `vscale` | Runtime multiplier for vector length |
| Predicate registers | `P0–P15` — control which lanes are active |
| `PTRUE` / `PFALSE` | Create all-true / all-false predicates |
| Destructive instructions | Source = destination (tracked via `DestructiveInstType` TSFlag) |
| Fixed-length SVE | `useSVEForFixedLengthVectors()` — use SVE for NEON-sized vectors |
| `LowerToScalableOp()` | Converts fixed-length ops to scalable SVE ops |
| `SVEIntrinsicOpts.cpp` | Optimizes SVE intrinsic sequences |
| `SVEShuffleOpts.cpp` | Converts shuffles to SVE TBL instructions |
| Tail folding | `DefaultSVETFOpts` — loop tail folding with predicates |

---

### 6.3 — SME (Scalable Matrix Extension)

**Files:** `AArch64SMEInstrInfo.td`, `SMEInstrFormats.td`, `AArch64SMEAttributes.h/.cpp`

| Concept | Key Detail |
|---|---|
| ZA register array | 2D tile storage for matrix operations |
| ZT0 register | Lookup table register (SME2) |
| Streaming mode | `PSTATE.SM` — `aarch64_pstate_sm_enabled/body/compatible` |
| `SMEAttrs` | Tracks ZA/ZT0 state and streaming mode per function |
| `MachineSMEABIPass.cpp` | Manages SME ABI transitions |
| `SMEPeepholeOpt.cpp` | SME-specific peephole optimizations |
| `AArch64Arm64ECCallLowering.cpp` | Arm64EC (Windows) call lowering with SME |
| Streaming hazards | `getStreamingHazardSize()` — padding between CPU/SME accesses |

---

## ⚫ Level 7 — Machine Code Emission & Scheduling

---

### 7.1 — MC Layer (Machine Code)

**Directory:** `MCTargetDesc/`

| Concept | Key Detail |
|---|---|
| `MCInst` | Final instruction representation before encoding |
| `AArch64MCInstLower.cpp` | Converts `MachineInstr` → `MCInst` |
| `AArch64AsmPrinter.cpp` | Drives assembly/object emission |
| `MCTargetDesc/AArch64MCTargetDesc.cpp` | Registers MC components |
| `MCTargetDesc/AArch64MCCodeEmitter.cpp` | Binary encoding of instructions |
| `MCTargetDesc/AArch64AsmBackend.cpp` | Fixups, relaxation, object writing |
| `MCTargetDesc/AArch64AddressingModes.h` | Encoding/decoding of immediate operands |
| LOH (Linker Optimization Hints) | `MCLOHDirective` — Mach-O ADRP+LDR optimization hints |

---

### 7.2 — Machine Scheduling

**Files:** `AArch64MachineScheduler.h/.cpp`, `AArch64Schedule.td`, `AArch64Sched*.td`

| Concept | Key Detail |
|---|---|
| `ScheduleDAGMILive` | Pre-RA instruction scheduler |
| `AArch64PostRASchedStrategy` | Custom post-RA scheduler |
| `SchedMachineModel` | Per-CPU scheduling model (issue width, latencies) |
| `WriteRes` / `ReadAdvance` | Resource usage and forwarding latencies |
| Macro fusion | `AArch64MacroFusion.cpp` — fuse instruction pairs (ADRP+ADD, CMP+Bcc) |
| Load/store clustering | `createLoadClusterDAGMutation`, `createStoreClusterDAGMutation` |
| CPU-specific models | `AArch64SchedA53.td`, `AArch64SchedNeoverseN1.td`, `AArch64SchedCyclone.td`, etc. |
| `AArch64PfmCounters.td` | Performance counter mappings for profiling |

---

### 7.3 — Disassembler

**Directory:** `Disassembler/`

| Concept | Key Detail |
|---|---|
| `AArch64Disassembler.cpp` | Decodes binary → `MCInst` |
| Used by `llvm-objdump`, debuggers | Reverse of the encoder |
| `DecodeStatus` | Success/Fail/SoftFail for partial decoding |

---

## 🏆 Level 8 — Expert-Level Topics

---

### 8.1 — Register Allocation Interaction

**Files:** `AArch64PBQPRegAlloc.h/.cpp`, `AArch64RegisterInfo.cpp`

| Concept | Key Detail |
|---|---|
| PBQP (Partitioned Boolean Quadratic Programming) | Custom RA constraints for NEON register pairing |
| `getCustomPBQPConstraints()` | Adds AArch64-specific PBQP cost nodes |
| `shouldCoalesce()` | Controls register coalescing decisions |
| `getRegAllocationHints()` | Hints for SME transposed register tuples |
| `AArch64SRLTDefineSuperRegs.cpp` | Subreg liveness tracking mitigation |

---

### 8.2 — Machine Outliner

**File:** `AArch64InstrInfo.cpp` (outliner methods)

| Concept | Key Detail |
|---|---|
| `getOutliningCandidateInfo()` | Identifies sequences safe to outline |
| `buildOutlinedFrame()` | Constructs the outlined function |
| `insertOutlinedCall()` | Replaces sequence with call |
| `getOutliningTypeImpl()` | Classifies each instruction for outlining |
| LR save/restore | Critical challenge — LR must be preserved across outlined calls |

---

### 8.3 — Target Transform Info (TTI)

**Files:** `AArch64TargetTransformInfo.h/.cpp`

| Concept | Key Detail |
|---|---|
| `AArch64TTIImpl` | Provides cost models to middle-end passes |
| Vectorization costs | `getArithmeticInstrCost()`, `getShuffleCost()` |
| Loop unrolling hints | `getUnrollingPreferences()` |
| Interleaving factors | `getMaxInterleaveFactor()` |
| Cache parameters | `getCacheLineSize()`, `getPrefetchDistance()` |
| SVE-specific costs | Scalable vector operation costs |

---

### 8.4 — TLS (Thread-Local Storage)

**Files:** `AArch64ISelLowering.cpp`, `AArch64CleanupLocalDynamicTLSPass.cpp`

| Concept | Key Detail |
|---|---|
| ELF TLS models | Local-exec, Initial-exec, Local-dynamic, General-dynamic |
| `LowerELFGlobalTLSAddress()` | ELF TLS lowering |
| `LowerDarwinGlobalTLSAddress()` | Darwin TLS via `_tlv_get_addr` |
| `LowerWindowsGlobalTLSAddress()` | Windows TLS via `__readgsqword` |
| TLSDESC | `LowerELFTLSDescCallSeq()` — fast TLS descriptor model |
| `LDTLSCleanup` pass | Combines multiple `_TLS_MODULE_BASE_` accesses |

---

### 8.5 — Pointer Authentication (PAC/BTI)

**Files:** `AArch64PointerAuth.h/.cpp`, `AArch64BranchTargets.cpp`

| Concept | Key Detail |
|---|---|
| PAC keys | IA, IB (instruction), DA, DB (data) |
| `PACIA` / `AUTIA` | Sign/authenticate with IA key |
| PAC-RET | Signs LR in prologue, authenticates in epilogue |
| PAuthLR | Uses PC as salt for LR signing |
| BTI | `BTIASYNC`, `BTICALL`, `BTIJUMP` — landing pad instructions |
| `AArch64FunctionInfo::SignCondition` | Controls when PAC-RET is applied |
| `fixupPtrauthDiscriminator()` | Optimizes discriminator operands |

---

## 📅 Recommended Study Sequence

| Timeframe | Focus |
|---|---|
| Week 1–2 | Level 1 — AArch64 ISA + LLVM IR + Pipeline overview |
| Week 3–4 | Level 2 — TargetMachine, Subtarget, TableGen basics |
| Week 5–6 | Level 2 continued — RegisterInfo, InstrInfo |
| Week 7–9 | Level 3 — SelectionDAG lowering + DAGToDAG selection |
| Week 10–11 | Level 4 — Calling conventions + Frame lowering |
| Week 12–13 | Level 5 — Optimization passes (pick 2–3 to study deeply) |
| Week 14–15 | Level 6 — NEON first, then SVE basics |
| Week 16+ | Levels 7–8 — MC layer, scheduling, advanced topics as needed |

---

## 📂 Key Files to Read First (In Order)

| Priority | File | Why |
|---|---|---|
| 1 | `AArch64TargetMachine.cpp` | Shows the entire pass pipeline |
| 2 | `AArch64Subtarget.h` | Central hub of all backend components |
| 3 | `AArch64RegisterInfo.td` | Defines all registers and classes |
| 4 | `AArch64InstrFormats.td` | Instruction encoding structure |
| 5 | `AArch64ISelLowering.cpp` | Largest file — core lowering logic |
| 6 | `AArch64ISelDAGToDAG.cpp` | Instruction selection patterns |
| 7 | `AArch64FrameLowering.cpp` | Stack frame management |
| 8 | `AArch64CallingConvention.td` | ABI rules |
| 9 | `AArch64LoadStoreOptimizer.cpp` | Good example of a real optimization pass |
| 10 | `GISel/AArch64InstructionSelector.cpp` | Modern GlobalISel path |
 
