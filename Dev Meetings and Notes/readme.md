# LLVM Backend Learning Resources
Architecture & Platform Information for Compiler Writers - https://www.llvm.org/docs/CompilerWriterInfo.html

## 1. LLVM IR

- [LLVM IR](https://www.youtube.com/watch?v=m8G_S5LwlTo)
- [LLVM Language Reference](https://llvm.org/docs/LangRef.html)

## 2. LLVM IR Optimization

- [Optimization Passes in LLVM IR](https://www.youtube.com/watch?v=7GHXDEIMGIY)
- [LLVM Passes and Pass Manager](https://llvm.org/docs/NewPassManager.html)
- [LLVM Transformation Passes](https://llvm.org/docs/Passes.html)

## 3. Target-Aware Optimization / Cost Model

- [Cost Modeling (TTI)](https://www.youtube.com/watch?v=uvOiF0RtaGs)
- [Target Transform Info](https://llvm.org/docs/TargetTransformInfo.html)
- [TargetTransformInfo — LLVM Documentation](https://llvm.org/doxygen/classllvm_1_1TargetTransformInfo.html)

## 4. LLVM Code Generator

- [LLVM Code Generator](https://llvm.org/docs/CodeGenerator.html)
- [Writing an LLVM Backend](https://llvm.org/docs/WritingAnLLVMBackend.html)

## 5. Machine IR

- [Machine IR (MIR) Language Reference](https://llvm.org/docs/MIRLangRef.html)
- [LLVM Code Generator](https://llvm.org/docs/CodeGenerator.html)

## 6. Instruction Selection — Overview

- [GI vs SD](https://www.youtube.com/watch?v=F6GGbYtae3g&list=PL_R5A0lGi1AA4Lv2bBFSwhgDaHvvpVU21&index=3)
- [The State of the Art at Instruction Selection](https://discourse.llvm.org/t/the-state-of-art-at-instruction-selection/57674/3)

## 7. SelectionDAG

- [SelectionDAG](https://youtu.be/nNQ6AF6i5FI)
- [SelectionDAG Instruction Selection](https://llvm.org/docs/CodeGenerator.html#selectiondag-instruction-selection)

## 8. GlobalISel

- [GlobalISel](https://llvm.org/docs/GlobalISel.html)
- [GlobalISel Pipeline](https://llvm.org/docs/GlobalISel/Pipeline.html)
- [Porting GlobalISel to a New Target](https://llvm.org/docs/GlobalISel/Porting.html)
- [GlobalISel MIR Patterns](https://llvm.org/docs/GlobalISel/MIRPatterns.html)

## 9. TableGen

- [TableGen — AArch64](https://youtu.be/vkVjIAlzdMw)
- [TableGen Programmer's Reference](https://llvm.org/docs/TableGen/ProgRef.html)
- [TableGen Backends](https://llvm.org/docs/TableGen/Backends.html)

## 10. Instruction Scheduling

- [Instruction Scheduling Model — LLVM](https://www.youtube.com/watch?v=YZHhlmOTG0g)
- [Machine Instruction Scheduling](https://llvm.org/docs/CodeGenerator.html#machine-instruction-scheduling)

## 11. Register Allocation

- [Register Allocation](https://www.youtube.com/watch?v=IK8TMJf3G6U)
- [Register Allocation](https://llvm.org/docs/CodeGenerator.html#register-allocation)

## 12. Prologue / Epilogue / Frame Lowering

- [LLVM Machine Representation — Prologue/Epilogue and Frame Lowering](https://llvm.org/devmtg/2017-10/slides/Braun-Welcome%20to%20the%20Back%20End.pdf)
- [Building an LLVM Backend — Frame Lowering](https://llvm.org/devmtg/2014-04/PDFs/Talks/Building%20an%20LLVM%20backend.pdf)
- [Prolog/Epilog Code Insertion](https://llvm.org/docs/CodeGenerator.html#prolog-epilog-code-insertion)
- [TargetFrameLowering](https://llvm.org/doxygen/classllvm_1_1TargetFrameLowering.html)

## 13. Machine-Level Optimizations

- [Machine Code Optimizations](https://llvm.org/docs/CodeGenerator.html#machine-code-optimizations)
- [SSA-Based Machine Code Optimizations](https://llvm.org/docs/CodeGenerator.html#ssa-based-machine-code-optimizations)

## 14. MC Layer / Object Code Emission

- [Object Code Emission](https://www.youtube.com/watch?v=VPyZBi39Ymw&t=236s)
- [The MC Layer](https://llvm.org/docs/CodeGenerator.html#the-mc-layer)

## 15. Complete LLVM Backend

- [Creating an LLVM Backend](https://youtu.be/b53WqCbLEYg)
- [Writing an LLVM Backend](https://llvm.org/docs/WritingAnLLVMBackend.html)

## 16. LLVM AArch64 Backend

- [AArch64 Backend — LLVM Source](https://github.com/llvm/llvm-project/tree/main/llvm/lib/Target/AArch64)
- [AArch64 Target Documentation](https://llvm.org/docs/AArch64.html)

## 17. C++ Prerequisites

- [Modern C++](https://youtube.com/playlist?list=PLgnQpQtFTOGRM59sr3nSL8BmeMZR9GCIA)
- [Concurrency in C++](https://youtube.com/playlist?list=PLvv0ScY6vfd_ocTP2ZLicgqKnvq50OCXM)

## 18. Introducing Scalable Vector Extensions(SVE) to LLVM

- [SVE](https://llvm.org/devmtg/2016-11/Slides/Emerson-ScalableVectorizationinLLVMIR.pdf)
- [Dev Team Meeting](https://youtu.be/0up2hJk7k94)

## 19. Introduction to JIT for MCJIT target

- [JIT Compilation tutorial slides](https://llvm.org/devmtg/2022-11/slides/Tutorial2-JITLink.pdf)
- [Dev Team Meeting] (https://www.youtube.com/watch?v=UwHgCqQ2DDA)

```
1. LLVM IR
       ↓
2. LLVM IR Optimization
       ↓
3. TTI / Cost Model
       ↓
4. Pass Manager
       ↓
5. LLVM Code Generator
       ↓
6. Machine IR / MIR
       ↓
7. GI vs SelectionDAG
       ↓
8. SelectionDAG
       ↓
9. GlobalISel
       ↓
10. TableGen
       ↓
11. MachineInstr / MachineFunction
       ↓
12. Instruction Scheduling
       ↓
13. Register Allocation
       ↓
14. Post-RA Machine Passes
       ↓
15. MC Layer / Object Emission
       ↓
16. Assembly / Machine Code

```
