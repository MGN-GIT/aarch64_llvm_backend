
## Xtensa Backend Issues and TODOs

The following items were identified by reviewing the Xtensa backend:

| Issues/TODO | Detailed Description | Difficulty | Component |
|---|---|---|---|
| Big-endian support | The machine-code emitter and assembler backend reject big-endian mode, so Xtensa big-endian assembly/object generation is not supported by these paths. | High | [MC code emitter](llvm/lib/Target/Xtensa/MCTargetDesc/XtensaMCCodeEmitter.cpp), [assembler backend](llvm/lib/Target/Xtensa/MCTargetDesc/XtensaAsmBackend.cpp) |
| Tail-call optimization | `LowerCall` disables ordinary tail calls and fails compilation for `musttail` calls. Implementing this requires respecting the Xtensa calling convention and frame teardown. | High | [ISel lowering](llvm/lib/Target/Xtensa/XtensaISelLowering.cpp) |
| Hardware loops with text-section literals | Hardware-loop profitability is disabled when literals are placed in the text section. Constant-island support is needed to allow loop formation in this mode. | High | [Target transform info](llvm/lib/Target/Xtensa/XtensaTargetTransformInfo.cpp), ConstantIslands pass |
| Mask source in BREG fixup | The `MOVBA` expansion shifts and ORs the source into BREG without masking it first. Confirm the source constraints and mask it if bits outside the intended field can be present. | Medium | [BREG fixup pass](llvm/lib/Target/Xtensa/XtensaBRegFixupPass.cpp) |
| Out-of-range immediate handling | `buildLoadImmediate` handles values up to 32 bits via `L32R`; larger or smaller values reach a fatal error. Confirm whether such values can reach this helper and define the intended handling. | Medium | [Instruction info](llvm/lib/Target/Xtensa/XtensaInstrInfo.cpp) |
| Floating-point operation selection | `FSQRT`, `FSIN`, and `FCOS` are currently marked `Expand`; the TODO suggests enabling target-specific instruction handling when available. | Medium | [ISel lowering](llvm/lib/Target/Xtensa/XtensaISelLowering.cpp) |
| Branch-swap optimization | Branch analysis lacks the X86-style branch swap mentioned in the TODO. This is an optimization opportunity for branch layout and code quality, not an apparent correctness blocker. | Low | [Instruction info](llvm/lib/Target/Xtensa/XtensaInstrInfo.cpp) |
| Scavenging spill-slot efficiency | Windowed ABI frame lowering notes that scavenging spill slots may be allocated more efficiently, potentially reducing stack and frame overhead. | Low | [Frame lowering](llvm/lib/Target/Xtensa/XtensaFrameLowering.cpp) |
