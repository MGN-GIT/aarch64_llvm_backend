# AArch64 LLVM Backend — Contribution Guide

This document catalogs all known `TODO`, `FIXME`, and pending implementation
items found across the AArch64 backend source files. Each entry includes the
source file, the function or context where it appears, a description of the
work needed, and an estimated difficulty level.

Use this as a starting point when looking for something to contribute.

---

## Table of Contents

1. [How to Contribute](#how-to-contribute)
2. [Difficulty Legend](#difficulty-legend)
3. [Recommended Starting Points](#recommended-starting-points)
4. [Frame Lowering](#1-frame-lowering--aarch64frameloweringcpph)
5. [Instruction Info](#2-instruction-info--aarch64instrinfocpp)
6. [Load/Store Optimizer](#3-loadstore-optimizer--aarch64loadstorecpp)
7. [Fast ISel](#4-fast-isel--aarch64fastiselectcpp)
8. [Conditional Compares](#5-conditional-compares--aarch64conditionalcomparescpp)
9. [Condition Optimizer](#6-condition-optimizer--aarch64conditionoptimizercpp)
10. [AdvSIMD Scalar Pass](#7-advsimd-scalar-pass--aarch64advsimdscalarpasscpp)
11. [A57 FP Load Balancing](#8-a57-fp-load-balancing--aarch64a57fploadbalancingcpp)
12. [Immediate Expansion](#9-immediate-expansion--aarch64expandimmcpp)
13. [Arm64EC Call Lowering](#10-arm64ec-call-lowering--aarch64arm64eccallloweringcpp)
14. [Collect LOH](#11-collect-loh--aarch64collectlohcpp)
15. [TableGen / Instruction Formats](#12-tablegen--instruction-formats--aarch64instrformatstd)
16. [Register Bank Info](#13-register-bank-info--aarch64genregisterbankinfoddef)
17. [Expand Pseudo Instructions](#14-expand-pseudo-instructions--aarch64expandpseudoinstscpp)
18. [Full Index Table](#full-index-table)

---

## How to Contribute

1. Pick an item from the tables below.
2. Find the relevant file and function using the **Location** column.
3. Search for the exact `TODO` or `FIXME` comment in the source to get full
   context.
4. Write a fix, add tests under `llvm/test/CodeGen/AArch64/`, and open a patch
   on [LLVM Phabricator](https://reviews.llvm.org) or a
   [GitHub pull request](https://github.com/llvm/llvm-project/pulls).
5. Reference this document's item number in your commit message for
   traceability.

---

## Difficulty Legend

| Symbol | Meaning |
|--------|---------|
| 🟢 Low | Self-contained change, minimal risk of regression, good for first-time contributors |
| 🟡 Medium | Requires understanding of surrounding infrastructure; moderate testing needed |
| 🔴 Hard | Touches multiple subsystems, requires deep knowledge of ABI/unwind/SVE semantics |

---

## Recommended Starting Points

These items are well-scoped, low-risk, and have clear expected outcomes.
They are ideal for first-time contributors to the AArch64 backend.

| # | File | What to Do |
|---|------|-----------|
| 15 | `AArch64InstrInfo.cpp` | Fold `x+1`, `-x`, `~x` into `CSEL` variants (`csinc`/`csneg`/`csinv`) |
| 16 | `AArch64InstrInfo.cpp` | Form `fabs`, `fmin`, `fmax` from `FCSEL` in `canInsertSelect` |
| 45 | `AArch64ConditionOptimizer.cpp` | Handle `TBNZ`/`TBZ` the same way as `CMP` for `a < 0` patterns |
| 46 | `AArch64ConditionOptimizer.cpp` | Handle `CSET` and other conditional instructions in cross-block optimization |
| 49 | `AArch64AdvSIMDScalarPass.cpp` | Add more opcode matches (many possibilities noted in comment) |
| 50 | `AArch64AdvSIMDScalarPass.cpp` | Avoid FPR64→GPR copy when the ultimate user already expects FPR64 |
| 51 | `AArch64AdvSIMDScalarPass.cpp` | Avoid GPR copy-back when all uses can use FPR64 directly |
| 55 | `AArch64ExpandImm.cpp` | Add more two-instruction MOV immediate sequences |
| 68 | `AArch64InstrFormats.td` | Roll out zero-register substitution to GPR32/GPR64 stores |
| 70 | `AArch64InstrFormats.td` | Fill in `Sched<[]>` scheduling details for `BaseSIMDInsDup` |
| 44 | `AArch64ConditionalCompares.cpp` | Clean up unexpected PHIs in CmpBB when they appear |
| 53 | `AArch64A57FPLoadBalancing.cpp` | Make `SizeFuzz` configurable instead of hardcoded to `1` |

---

## 1. Frame Lowering — `AArch64FrameLowering.cpp/.h`

| # | Type | Function | Description | Difficulty |
|---|------|----------|-------------|------------|
| 1 | TODO | `homogeneousPrologEpilog` | **Windows not yet supported** for homogeneous prolog/epilog size optimization. The function returns `false` early for Windows targets with a `TODO` comment. | 🟡 Medium |
| 2 | TODO | `homogeneousPrologEpilog` | **SVE not yet supported** for homogeneous prolog/epilog. The function returns `false` early when SVE stack objects are likely present. | 🔴 Hard |
| 3 | TODO | `isTargetWindows` | **UEFI target classification** — should `isTargetWindows` include UEFI targets that use Windows CFI? Currently no AArch64 UEFI support exists, but the predicate must stay in sync with `getCalleeSavedRegs`. | 🟡 Medium |
| 4 | TODO | `windowsRequiresStackProbe` | **Stack protector threshold** not factored into the Windows stack probe size decision. Comment says "TODO: When implementing stack protectors, take that into account for the probe threshold." | 🟡 Medium |
| 5 | FIXME | `estimateRSStackSizeLimit` | **Conservative spill slot estimate** — currently guesses based on unscaled indexing range, which causes unnecessary spill slot allocation. A more precise estimate would reduce stack usage. | 🟡 Medium |
| 6 | FIXME | `eliminateCallFramePseudoInstr` | **In-function stack adjustment limited to 24 bits** — there is no guaranteed temporary register available for adjustments larger than 24 bits. `ADD`/`SUB` immediate only supports LSL #0 and LSL #12. | 🔴 Hard |
| 7 | FIXME | `shouldSignReturnAddressEverywhere` | **WinCFI + PAC-RET instruction ordering** — `SEH_PACSignLR` and `SEH_EpilogEnd` must be placed in the correct order when WinCFI is used. Currently the function returns `false` early for Windows CFI targets. | 🟡 Medium |
| 8 | FIXME | `getFrameIndexReference` | **Debug info SP-relative references** can produce wrong offsets when simple call frames are not used. The comment says this is the same as the code-gen reference "for now". | 🟡 Medium |
| 9 | TODO | `getSEHFrameIndexOffset` | **Scalable vectors not supported** — the function does not work for SVE/scalable vector frame indices. A comment at the top of the function explicitly states this. | 🔴 Hard |
| 10 | FIXME | `determineStackHazardSlot` | **SVE object alignment > 16 bytes** — SVE vector length is not necessarily a power of two, so objects requiring alignment larger than 16 bytes would need dynamic runtime alignment. This is not yet implemented. | 🔴 Hard |
| 11 | FIXME | `mergeSetTagsInsns` (epilog) | **Conservative STG loop liveness check** — the current approach bails out of the merge even when STG loops are not present after the merge insert list. The liveness check is overly broad. | 🟡 Medium |
| 12 | FIXME | `AArch64FrameLowering.h` | **Stack realignment scratch register** — conservatively avoids using callee-save registers as a scratch for re-alignment. A smarter approach could use a callee-save reg if it is being saved anyway. | 🟡 Medium |
| 13 | FIXME | `determineCalleeSaves` | **Compact unwind pairing** — the current code forces register pairing even when unwinding is not needed. The non-paired format is actually better in that case. | 🟢 Low |

---

## 2. Instruction Info — `AArch64InstrInfo.cpp`

| # | Type | Function | Description | Difficulty |
|---|------|----------|-------------|------------|
| 14 | FIXME | `getInstSizeInBytes` | **Pseudo-instruction sizing** — only handles pseudos that do not expand before the assembler printer. Other pseudo-instructions are not covered and fall through to a default 4-byte size. | 🟡 Medium |
| 15 | FIXME | `canInsertSelect` (GPR path) | **Missing CSEL folds** — `x+1`, `-x`, and `~x` patterns are not yet folded into `csinc`, `csneg`, and `csinv`. The existing `canFoldIntoCSel` infrastructure is already in place and can be extended. | 🟢 Low |
| 16 | FIXME | `canInsertSelect` (FPR path) | **Missing FCSEL folds** — `fabs`, `fmin`, and `fmax` are not yet formed from `FCSEL`. | 🟢 Low |
| 17 | FIXME | `isAsCheapAsAMove` | **Micro-architecture dependent cost** — the implementation should use a micro-architecture target hook rather than a one-size-fits-all heuristic. | 🟡 Medium |
| 18 | FIXME | `analyzeCompare` | **Subregisters not passed out** of `analyzeCompare` — two separate FIXME comments note that subregister information is dropped, causing the function to return `false` when subregisters are present. | 🟢 Low |
| 19 | FIXME | `emitFrameOffset` | **Offset > 24-bit scratch register** — if the offset does not fit in 24 bits, it should be computed into a scratch register. This is currently unimplemented; the comment suggests using `DestReg` as the scratch if it is virtual. | 🟡 Medium |
| 20 | TODO | Machine combiner gather pattern | **More opcodes to match** — integer types, vectors, XOR, and OR are not yet handled in the gather pattern optimization. Comment says "There are many more machine instruction opcodes to match." | 🟡 Medium |
| 21 | FIXME | `getOutliningCandidateInfo` | **SP-modifying instructions in outliner** — instructions like `add x0, sp, #8` are not handled. The outliner bails out when it cannot fix up the offset. | 🟡 Medium |
| 22 | FIXME | `getOutliningCandidateInfo` | **LR value after outlined call** — no check is performed to determine whether the code after the outlined call uses the value of LR. | 🟡 Medium |
| 23 | FIXME | `isFunctionSafeToOutlineFrom` | **Streaming-mode changes in outliner** — it is unsafe to outline across `smstart`/`smstop` pairs. The outliner needs to ensure such pairs are outlined together and that async unwind info is handled correctly. | 🔴 Hard |
| 24 | FIXME | `isFunctionSafeToOutlineFrom` | **Windows unwind info in outliner** — the outliner cannot generate or handle Windows unwind info. Currently disabled entirely for Windows CFI targets. | 🔴 Hard |
| 25 | FIXME | `isFunctionSafeToOutlineFrom` | **CFI instructions in outliner** — cannot outline across CFI instructions because the proper offset fixups are not implemented. | 🟡 Medium |
| 26 | FIXME | `isFunctionSafeToOutlineFrom` | **Functions with stack frames in outliner** — calls that construct a stack frame are not yet allowed in the outliner. | 🔴 Hard |
| 27 | TODO | `getOutliningCandidateInfo` | **Bugzilla #46767 — multiple stack adjustments** — outlining when the stack is adjusted more than once is not yet safe or supported. | 🔴 Hard |
| 28 | FIXME | `isFunctionSafeToOutlineFrom` | **Noreturn functions in outliner** — liveness info is not fixed up for noreturn functions, so outlining is always disabled for them. A targeted fix could re-enable outlining in safe cases. | 🟡 Medium |
| 29 | FIXME | `isFunctionSafeToOutlineFrom` | **Section-marked functions in outliner** — outlining from multiple functions with the same section marking is disabled. Could be allowed when the sections match. | 🟢 Low |

---

## 3. Load/Store Optimizer — `AArch64LoadStoreOptimizer.cpp`

| # | Type | Function | Description | Difficulty |
|---|------|----------|-------------|------------|
| 30 | FIXME | `mergeSetTagsInsns` | **Conservative STG loop liveness bail-out** — the liveness check bails out even when STG loops are not present after the merge point. The check should be narrowed to only apply when STG loops are actually present in the relevant range. | 🟡 Medium |

---

## 4. Fast ISel — `AArch64FastISel.cpp`

| # | Type | Function | Description | Difficulty |
|---|------|----------|-------------|------------|
| 31 | TODO | ADRP emission | **Duplicate ADRP logic** — there is duplicate logic between `AArch64FastISel` and `AArch64ExpandPseudoInsts.cpp` for building `ADRP + MOVK + ADD`. The operands are not 1:1 so abstraction is non-trivial but worthwhile. | 🟢 Low |
| 32 | FIXME | `addMemOperand` | **Frame index size/alignment** — should use VT-based size and alignment rather than `getObjectSize`/`getObjectAlignment`. | 🟢 Low |
| 33 | FIXME | Live-in copy emission | **Unnecessary live-in copy** — `EmitLiveInCopies` may eliminate the live-in if its only use is a bitcast, which is not turned into an instruction. The copy is emitted defensively but may be avoidable. | 🟢 Low |
| 34 | FIXME | `fastLowerCall` | **Custom argument lowering** — the `VA.needsCustom()` path returns `false` without handling custom arguments. | 🟡 Medium |
| 35 | TODO | `fastLowerCall` | **Big-endian vector results** — vector return values are not handled for big-endian targets; the function returns `false`. | 🟡 Medium |
| 36 | FIXME | `fastLowerCall` | **ILP32 target support** — ILP32 is disabled at `-O0` for correctness reasons. Proper support should be implemented. | 🟡 Medium |
| 37 | FIXME | `fastLowerCall` | **Large code model ELF** — large code model is not supported for ELF in FastISel; only MachO is supported. | 🟡 Medium |
| 38 | FIXME | `fastLowerIntrinsicCall` | **More intrinsics** — only a small subset of intrinsics is handled; most fall through to `return false` and are not fast-selected. | 🟡 Medium |
| 39 | FIXME | `fastLowerIntrinsicCall` | **SExt i1 to i64** — sign-extension of `i1` to `i64` returns an empty register instead of being handled. | 🟢 Low |
| 40 | FIXME | `fastLowerIntrinsicCall` | **`MachineMemOperand` for cmpxchg** — `MachineMemOperand` does not support cmpxchg yet, so the memory operand is not attached to the instruction. | 🟡 Medium |

---

## 5. Conditional Compares — `AArch64ConditionalCompares.cpp`

| # | Type | Function | Description | Difficulty |
|---|------|----------|-------------|------------|
| 41 | FIXME | `canSpeculateBlock` | **Critical edge speculation** — it should be possible to speculate a block on the critical edge between Head and Tail (diamond if-conversion), but this is not yet implemented. | 🔴 Hard |
| 42 | FIXME | `canSpeculateBlock` | **PHIs in Tail block** — PHI nodes in the Tail block could be if-converted to selects, but this is not yet handled. | 🟡 Medium |
| 43 | FIXME | `canSpeculateBlock` | **Real PHIs in CmpBB** — PHIs in CmpBB could be handled if the CmpBB values are defined before the ccmp clobbers the flags. Alternatively, sinking the ccmp past the PHI would always be safe. | 🟡 Medium |
| 44 | FIXME | `canSpeculateBlock` | **PHI cleanup in CmpBB** — CmpBB should never have PHIs since Head is its only predecessor, but if it does, they should be cleaned up rather than causing a silent bail-out. | 🟢 Low |

---

## 6. Condition Optimizer — `AArch64ConditionOptimizer.cpp`

| # | Type | Function | Description | Difficulty |
|---|------|----------|-------------|------------|
| 45 | TODO | `optimizeCrossBlock` | **TBNZ/TBZ handling** — `TBNZ`/`TBZ` are not handled the same way as `CMP` for `a < 0` patterns. | 🟢 Low |
| 46 | TODO | `optimizeCrossBlock` | **Other conditional instructions** — instructions like `CSET` are not handled in cross-block optimization. | 🟢 Low |
| 47 | TODO | `optimizeCrossBlock` | **Flexible second branch** — the second branch could be any instruction that does not require adjusting, but currently only specific forms are handled. | 🟡 Medium |

---

## 7. AdvSIMD Scalar Pass — `AArch64AdvSIMDScalarPass.cpp`

| # | Type | Function | Description | Difficulty |
|---|------|----------|-------------|------------|
| 48 | TODO | Pass design | **Graph-based predicate heuristics** — the current linear instruction list walk misses cases where instructions have interdependencies. A graph-based analysis would give more thorough coverage. | 🔴 Hard |
| 49 | FIXME | Opcode matching | **Many more opcodes** — only a small set of opcodes is handled. The comment says "Lots more possibilities." Many arithmetic and logical opcodes could be added. | 🟡 Medium |
| 50 | FIXME | Destination register allocation | **Avoid FPR64→GPR copy** — there is no need to copy to a GPR if the ultimate user already expects an FPR64. The pass should check for this and avoid the copy. | 🟢 Low |
| 51 | FIXME | Result copy-back | **Avoid GPR copy-back** — the result copy back to a GPR could be avoided if all uses could use the FPR64 directly. | 🟢 Low |

---

## 8. A57 FP Load Balancing — `AArch64A57FPLoadBalancing.cpp`

| # | Type | Function | Description | Difficulty |
|---|------|----------|-------------|------------|
| 52 | FIXME | Chain detection | **Interdependent chains** — chains with interdependencies (e.g. `mul r0, r1, r2` followed by `mul r3, r0, r1`) are not handled. The pass needs to track other uses of registers it wants to rewrite. | 🔴 Hard |
| 53 | FIXME | `SizeFuzz` constant | **Hardcoded size fuzz** — `SizeFuzz = 1` is hardcoded. The comment asks "Does this need to be configurable?" It should be exposed as a command-line option or target hook. | 🟢 Low |
| 54 | FIXME | Non-kill operand rewrite | **Non-kill register rewrite** — the pass only handles the kill case. Non-kill cases require tracking other uses of the registers being rewritten. | 🟡 Medium |

---

## 9. Immediate Expansion — `AArch64ExpandImm.cpp`

| # | Type | Function | Description | Difficulty |
|---|------|----------|-------------|------------|
| 55 | FIXME | `expandMOVImm` | **More two-instruction MOV sequences** — the comment says "Add more two-instruction sequences" before falling through to three-instruction sequences. Adding more two-instruction patterns would reduce code size and improve performance. | 🟢 Low |

---

## 10. Arm64EC Call Lowering — `AArch64Arm64ECCallLowering.cpp`

| # | Type | Function | Description | Difficulty |
|---|------|----------|-------------|------------|
| 56 | FIXME | Varargs thunk | **x5 not used by x64 side** — x5 is pushed into the x64 argument list but is not actually used by the x64 side. Needs revisiting once proper isel for varargs is implemented. | 🔴 Hard |
| 57 | FIXME | Argument size info | **`getParamArm64ECArgSizeBytes` missing info** — the argument size information needed for proper thunk generation is not available. The relevant code is `#if 0`'d out pending resolution of D132926. | 🔴 Hard |
| 58 | FIXME | Return value size info | **`getRetArm64ECArgSizeBytes` missing info** — same issue as above for return values. | 🔴 Hard |
| 59 | FIXME | sret type validation | **sret type sanity check** — the sret type should be validated; integer or pointer sret types cause incorrect mangling and codegen. | 🟢 Low |
| 60 | FIXME | Large struct sret | **Large struct sret mangling** — large struct sret should be mangled as an integer argument and integer return to allow more thunk reuse, instead of using `m` syntax. | 🟡 Medium |
| 61 | FIXME | Type canonicalization | **`Arm64Ty` not thoroughly canonicalized** — the type is not fully canonicalized before mangling, which may produce suboptimal or incorrect thunk names. | 🟢 Low |
| 62 | FIXME | Attribute transfer | **Missing attribute copying** — only `sret` is copied to the thunk; other attributes that may be necessary for correctness are not transferred. | 🟢 Low |
| 63 | FIXME | Weak linkage | **Weak linkage functions** — functions with weak linkage are not handled in the metadata name emission path. | 🟡 Medium |
| 64 | FIXME | Bitcast calls | **`getCalledFunction()` fails with bitcast** — `getCalledFunction()` returns null for unprototyped C functions that have an implicit bitcast, causing the optimization to be skipped. | 🟢 Low |

---

## 11. Collect LOH — `AArch64CollectLOH.cpp`

| # | Type | Function | Description | Difficulty |
|---|------|----------|-------------|------------|
| 65 | FIXME | LOH emission | **Proper liveness tracking** — the current implementation bails out if any instruction exists between the `ADD` and the `LDR`, even if those instructions do not clobber the relevant register. Full register liveness tracking would allow more LOH hints to be emitted. | 🟡 Medium |

---

## 12. TableGen / Instruction Formats — `AArch64InstrFormats.td`

| # | Type | Location | Description | Difficulty |
|---|------|----------|-------------|------------|
| 66 | TODO | `v2f32`/`v4f32` type map | **Other vector types missing** — only `v2f32` and `v4f32` are mapped in the type conversion table. Other vector types should be added. | 🟢 Low |
| 67 | FIXME | System register instructions | **NZCV def modeling** — some system register instructions define NZCV and others do not, but this is not properly modeled in the instruction definitions. Explicitly modeling each system register as a register class would fix this but may be overkill. | 🟡 Medium |
| 68 | TODO | `StoreUIz` multiclass | **Zero-register substitution for GPR stores** — zero-register substitution is implemented for `StoreUIz` but should be rolled out to GPR32/GPR64 stores and folded back into the base `StoreUI` multiclass. | 🟢 Low |
| 69 | FIXME | TableGen expanded types | **Element count change in expanded types** — TableGen cannot handle expanded types that also change the element count (e.g. placing results in the high elements of a register rather than the low elements). | 🔴 Hard |
| 70 | FIXME | `BaseSIMDInsDup` scheduling | **Missing scheduling details** — `Sched<[]>` is a placeholder. Actual scheduling information needs to be filled in once the details are known. | 🟢 Low |
| 71 | FIXME | SIMD instruction class factoring | **Poor code factoring** — the comment says "There has got to be a better way to factor these." The SIMD instruction class hierarchy could be refactored for clarity and maintainability. | 🟢 Low |

---

## 13. Register Bank Info — `AArch64GenRegisterBankInfo.def`

| # | Type | Location | Description | Difficulty |
|---|------|----------|-------------|------------|
| 72 | TODO | File header | **Should be TableGen-generated** — the static objects in this file are hand-written but the file header explicitly says "This should be generated by TableGen." Automating this would reduce maintenance burden and the risk of hand-written errors. | 🔴 Hard |

---

## 14. Expand Pseudo Instructions — `AArch64ExpandPseudoInsts.cpp`

| # | Type | Function | Description | Difficulty |
|---|------|----------|-------------|------------|
| 73 | FIXME | `expandMI` loop | **Extra liveness pass necessity unclear** — an extra pass is performed in the loop to get loop-carried dependencies right, but the comment asks "is this necessary?" Investigating and removing it if unneeded would simplify the code. | 🟢 Low |

---

## Full Index Table

| # | Type | File | Function | Description | Difficulty |
|---|------|------|----------|-------------|------------|
| 1 | TODO | `AArch64FrameLowering.cpp` | `homogeneousPrologEpilog` | Windows not supported for homogeneous prolog/epilog | 🟡 |
| 2 | TODO | `AArch64FrameLowering.cpp` | `homogeneousPrologEpilog` | SVE not supported for homogeneous prolog/epilog | 🔴 |
| 3 | TODO | `AArch64FrameLowering.cpp` | `isTargetWindows` | UEFI target classification | 🟡 |
| 4 | TODO | `AArch64FrameLowering.cpp` | `windowsRequiresStackProbe` | Stack protector threshold not factored in | 🟡 |
| 5 | FIXME | `AArch64FrameLowering.cpp` | `estimateRSStackSizeLimit` | Conservative spill slot estimate | 🟡 |
| 6 | FIXME | `AArch64FrameLowering.cpp` | `eliminateCallFramePseudoInstr` | In-function stack adjustment limited to 24 bits | 🔴 |
| 7 | FIXME | `AArch64FrameLowering.cpp` | `shouldSignReturnAddressEverywhere` | WinCFI + PAC-RET instruction ordering | 🟡 |
| 8 | FIXME | `AArch64FrameLowering.cpp` | `getFrameIndexReference` | Debug info SP-relative references can be wrong | 🟡 |
| 9 | TODO | `AArch64FrameLowering.cpp` | `getSEHFrameIndexOffset` | Scalable vectors not supported | 🔴 |
| 10 | FIXME | `AArch64FrameLowering.cpp` | `determineStackHazardSlot` | SVE object alignment > 16 bytes not supported | 🔴 |
| 11 | FIXME | `AArch64FrameLowering.cpp` | `mergeSetTagsInsns` | Conservative STG loop liveness check | 🟡 |
| 12 | FIXME | `AArch64FrameLowering.h` | Stack realignment | Conservative callee-save scratch register avoidance | 🟡 |
| 13 | FIXME | `AArch64FrameLowering.cpp` | `determineCalleeSaves` | Compact unwind forces pairing even when not needed | 🟢 |
| 14 | FIXME | `AArch64InstrInfo.cpp` | `getInstSizeInBytes` | Pseudo-instruction sizing incomplete | 🟡 |
| 15 | FIXME | `AArch64InstrInfo.cpp` | `canInsertSelect` | Missing CSEL folds for x+1, -x, ~x | 🟢 |
| 16 | FIXME | `AArch64InstrInfo.cpp` | `canInsertSelect` | Missing FCSEL folds for fabs, fmin, fmax | 🟢 |
| 17 | FIXME | `AArch64InstrInfo.cpp` | `isAsCheapAsAMove` | Should use micro-architecture target hook | 🟡 |
| 18 | FIXME | `AArch64InstrInfo.cpp` | `analyzeCompare` | Subregisters not passed out of analyzeCompare | 🟢 |
| 19 | FIXME | `AArch64InstrInfo.cpp` | `emitFrameOffset` | Offset > 24-bit scratch register unimplemented | 🟡 |
| 20 | TODO | `AArch64InstrInfo.cpp` | Machine combiner | More opcodes to match (int, vector, XOR, OR) | 🟡 |
| 21 | FIXME | `AArch64InstrInfo.cpp` | `getOutliningCandidateInfo` | SP-modifying instructions not handled in outliner | 🟡 |
| 22 | FIXME | `AArch64InstrInfo.cpp` | `getOutliningCandidateInfo` | LR value after outlined call not checked | 🟡 |
| 23 | FIXME | `AArch64InstrInfo.cpp` | `isFunctionSafeToOutlineFrom` | Streaming-mode changes unsafe to outline | 🔴 |
| 24 | FIXME | `AArch64InstrInfo.cpp` | `isFunctionSafeToOutlineFrom` | Windows unwind info not supported in outliner | 🔴 |
| 25 | FIXME | `AArch64InstrInfo.cpp` | `isFunctionSafeToOutlineFrom` | CFI instructions block outlining | 🟡 |
| 26 | FIXME | `AArch64InstrInfo.cpp` | `isFunctionSafeToOutlineFrom` | Functions with stack frames not outlinable | 🔴 |
| 27 | TODO | `AArch64InstrInfo.cpp` | `getOutliningCandidateInfo` | Bugzilla #46767 — multiple stack adjustments | 🔴 |
| 28 | FIXME | `AArch64InstrInfo.cpp` | `isFunctionSafeToOutlineFrom` | Noreturn functions always disabled for outlining | 🟡 |
| 29 | FIXME | `AArch64InstrInfo.cpp` | `isFunctionSafeToOutlineFrom` | Section-marked functions disabled for outlining | 🟢 |
| 30 | FIXME | `AArch64LoadStoreOptimizer.cpp` | `mergeSetTagsInsns` | Conservative STG loop liveness bail-out | 🟡 |
| 31 | TODO | `AArch64FastISel.cpp` | ADRP emission | Duplicate ADRP logic with ExpandPseudoInsts | 🟢 |
| 32 | FIXME | `AArch64FastISel.cpp` | `addMemOperand` | Frame index size/alignment should use VT | 🟢 |
| 33 | FIXME | `AArch64FastISel.cpp` | Live-in copy | Unnecessary live-in copy emission | 🟢 |
| 34 | FIXME | `AArch64FastISel.cpp` | `fastLowerCall` | Custom argument lowering not handled | 🟡 |
| 35 | TODO | `AArch64FastISel.cpp` | `fastLowerCall` | Big-endian vector results not handled | 🟡 |
| 36 | FIXME | `AArch64FastISel.cpp` | `fastLowerCall` | ILP32 target disabled at -O0 | 🟡 |
| 37 | FIXME | `AArch64FastISel.cpp` | `fastLowerCall` | Large code model ELF not supported | 🟡 |
| 38 | FIXME | `AArch64FastISel.cpp` | `fastLowerIntrinsicCall` | More intrinsics needed | 🟡 |
| 39 | FIXME | `AArch64FastISel.cpp` | `fastLowerIntrinsicCall` | SExt i1 to i64 returns empty register | 🟢 |
| 40 | FIXME | `AArch64FastISel.cpp` | `fastLowerIntrinsicCall` | MachineMemOperand missing for cmpxchg | 🟡 |
| 41 | FIXME | `AArch64ConditionalCompares.cpp` | `canSpeculateBlock` | Critical edge speculation not implemented | 🔴 |
| 42 | FIXME | `AArch64ConditionalCompares.cpp` | `canSpeculateBlock` | PHIs in Tail not if-converted to selects | 🟡 |
| 43 | FIXME | `AArch64ConditionalCompares.cpp` | `canSpeculateBlock` | Real PHIs in CmpBB not handled | 🟡 |
| 44 | FIXME | `AArch64ConditionalCompares.cpp` | `canSpeculateBlock` | PHI cleanup in CmpBB missing | 🟢 |
| 45 | TODO | `AArch64ConditionOptimizer.cpp` | `optimizeCrossBlock` | TBNZ/TBZ not handled like CMP for a < 0 | 🟢 |
| 46 | TODO | `AArch64ConditionOptimizer.cpp` | `optimizeCrossBlock` | CSET and other conditional instrs not handled | 🟢 |
| 47 | TODO | `AArch64ConditionOptimizer.cpp` | `optimizeCrossBlock` | Second branch could be more flexible | 🟡 |
| 48 | TODO | `AArch64AdvSIMDScalarPass.cpp` | Pass design | Graph-based predicate heuristics needed | 🔴 |
| 49 | FIXME | `AArch64AdvSIMDScalarPass.cpp` | Opcode matching | Many more opcodes to handle | 🟡 |
| 50 | FIXME | `AArch64AdvSIMDScalarPass.cpp` | Destination register | Avoid FPR64→GPR copy when user expects FPR64 | 🟢 |
| 51 | FIXME | `AArch64AdvSIMDScalarPass.cpp` | Result copy-back | Avoid GPR copy-back when all uses can use FPR64 | 🟢 |
| 52 | FIXME | `AArch64A57FPLoadBalancing.cpp` | Chain detection | Interdependent chains not handled | 🔴 |
| 53 | FIXME | `AArch64A57FPLoadBalancing.cpp` | `SizeFuzz` | Hardcoded size fuzz should be configurable | 🟢 |
| 54 | FIXME | `AArch64A57FPLoadBalancing.cpp` | Non-kill rewrite | Non-kill register rewrite not handled | 🟡 |
| 55 | FIXME | `AArch64ExpandImm.cpp` | `expandMOVImm` | More two-instruction MOV sequences needed | 🟢 |
| 56 | FIXME | `AArch64Arm64ECCallLowering.cpp` | Varargs thunk | x5 not used by x64 side; needs revisiting | 🔴 |
| 57 | FIXME | `AArch64Arm64ECCallLowering.cpp` | Argument size | getParamArm64ECArgSizeBytes info missing | 🔴 |
| 58 | FIXME | `AArch64Arm64ECCallLowering.cpp` | Return size | getRetArm64ECArgSizeBytes info missing | 🔴 |
| 59 | FIXME | `AArch64Arm64ECCallLowering.cpp` | sret validation | sret type not sanity-checked | 🟢 |
| 60 | FIXME | `AArch64Arm64ECCallLowering.cpp` | Large struct sret | Should mangle as integer for thunk reuse | 🟡 |
| 61 | FIXME | `AArch64Arm64ECCallLowering.cpp` | Type canonicalization | Arm64Ty not thoroughly canonicalized | 🟢 |
| 62 | FIXME | `AArch64Arm64ECCallLowering.cpp` | Attribute transfer | Only sret copied; other attrs not transferred | 🟢 |
| 63 | FIXME | `AArch64Arm64ECCallLowering.cpp` | Weak linkage | Weak linkage functions not handled | 🟡 |
| 64 | FIXME | `AArch64Arm64ECCallLowering.cpp` | Bitcast calls | getCalledFunction() fails with bitcast | 🟢 |
| 65 | FIXME | `AArch64CollectLOH.cpp` | LOH emission | Proper liveness tracking not implemented | 🟡 |
| 66 | TODO | `AArch64InstrFormats.td` | Type map | Other vector types missing from map | 🟢 |
| 67 | FIXME | `AArch64InstrFormats.td` | System registers | NZCV def not properly modeled | 🟡 |
| 68 | TODO | `AArch64InstrFormats.td` | `StoreUIz` | Zero-register substitution not rolled out to GPR stores | 🟢 |
| 69 | FIXME | `AArch64InstrFormats.td` | Expanded types | TableGen can't handle element count change | 🔴 |
| 70 | FIXME | `AArch64InstrFormats.td` | `BaseSIMDInsDup` | Scheduling details not filled in | 🟢 |
| 71 | FIXME | `AArch64InstrFormats.td` | SIMD factoring | SIMD instruction class hierarchy needs refactoring | 🟢 |
| 72 | TODO | `AArch64GenRegisterBankInfo.def` | File | Should be TableGen-generated, currently hand-written | 🔴 |
| 73 | FIXME | `AArch64ExpandPseudoInsts.cpp` | `expandMI` loop | Extra liveness pass may be unnecessary | 🟢 |

---

## Count by Difficulty

| Difficulty | Count |
|------------|-------|
| 🟢 Low     | 24    |
| 🟡 Medium  | 34    |
| 🔴 Hard    | 15    |
| **Total**  | **73** |

---

## Count by File

| File | Items |
|------|-------|
| `AArch64InstrInfo.cpp` | 16 |
| `AArch64FrameLowering.cpp/.h` | 13 |
| `AArch64FastISel.cpp` | 10 |
| `AArch64Arm64ECCallLowering.cpp` | 9 |
| `AArch64InstrFormats.td` | 6 |
| `AArch64ConditionalCompares.cpp` | 4 |
| `AArch64AdvSIMDScalarPass.cpp` | 4 |
| `AArch64ConditionOptimizer.cpp` | 3 |
| `AArch64A57FPLoadBalancing.cpp` | 3 |
| `AArch64LoadStoreOptimizer.cpp` | 1 |
| `AArch64ExpandImm.cpp` | 1 |
| `AArch64CollectLOH.cpp` | 1 |
| `AArch64GenRegisterBankInfo.def` | 1 |
| `AArch64ExpandPseudoInsts.cpp` | 1 |

---

*Last updated by automated source scan. Re-run the grep search for `TODO\|FIXME`
across the AArch64 backend directory to refresh this list after upstream changes.*
