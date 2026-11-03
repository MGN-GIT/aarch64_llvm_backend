
## Xtensa Backend Issues and TODOs

The following items were identified by reviewing the Xtensa backend:

| Issues/TODO | Detailed Description | Difficulty | Component |
|---|---|---|---|
| Big-endian support | The machine-code emitter, assembler backend, and disassembler reject big-endian mode, so Xtensa big-endian assembly, object generation, and disassembly are not supported by these paths. | High | [MC code emitter](llvm/lib/Target/Xtensa/MCTargetDesc/XtensaMCCodeEmitter.cpp), [assembler backend](llvm/lib/Target/Xtensa/MCTargetDesc/XtensaAsmBackend.cpp), [disassembler](llvm/lib/Target/Xtensa/Disassembler/XtensaDisassembler.cpp) |
| Tail-call optimization | `LowerCall` disables ordinary tail calls and fails compilation for `musttail` calls. Implementing this requires respecting the Xtensa calling convention and frame teardown. | High | [ISel lowering](llvm/lib/Target/Xtensa/XtensaISelLowering.cpp) |
| Hardware loops with text-section literals | Hardware-loop profitability is disabled when literals are placed in the text section. Constant-island support is needed to allow loop formation in this mode. | High | [Target transform info](llvm/lib/Target/Xtensa/XtensaTargetTransformInfo.cpp), ConstantIslands pass |
| Mask source in BREG fixup | The `MOVBA` expansion shifts and ORs the source into BREG without masking it first. Confirm the source constraints and mask it if bits outside the intended field can be present. | Medium | [BREG fixup pass](llvm/lib/Target/Xtensa/XtensaBRegFixupPass.cpp) |
| Out-of-range immediate handling | `buildLoadImmediate` handles values up to 32 bits via `L32R`; larger or smaller values reach a fatal error. Confirm whether such values can reach this helper and define the intended handling. | Medium | [Instruction info](llvm/lib/Target/Xtensa/XtensaInstrInfo.cpp) |
| Floating-point operation selection | `FSQRT`, `FSIN`, and `FCOS` are currently marked `Expand`; the TODO suggests enabling target-specific instruction handling when available. | Medium | [ISel lowering](llvm/lib/Target/Xtensa/XtensaISelLowering.cpp) |
| Branch-swap optimization | Branch analysis lacks the X86-style branch swap mentioned in the TODO. This is an optimization opportunity for branch layout and code quality, not an apparent correctness blocker. | Low | [Instruction info](llvm/lib/Target/Xtensa/XtensaInstrInfo.cpp) |
| Scavenging spill-slot efficiency | Windowed ABI frame lowering notes that scavenging spill slots may be allocated more efficiently, potentially reducing stack and frame overhead. | Low | [Frame lowering](llvm/lib/Target/Xtensa/XtensaFrameLowering.cpp) |
| NOP padding for unsupported byte counts | `writeNopData` emits a zero byte when one byte remains after writing full-width NOPs. The TODO questions whether non-instruction-sized padding should instead return an error; verify the assembler's alignment callers and define safe behavior. | Medium | [Assembler backend](llvm/lib/Target/Xtensa/MCTargetDesc/XtensaAsmBackend.cpp) |
| Empty constant-island blocks | Removing an unused constant-pool entry can leave an empty block behind. The FIXME asks whether those blocks should be removed after the constant-island work; doing so could simplify generated control flow/layout. | Low | [ConstantIslands pass](llvm/lib/Target/Xtensa/XtensaConstantIsland.cpp) |
| Constant-island alignment option cleanup | A hidden alignment option is marked for removal after sufficient testing. Confirm whether the default behavior is now established and remove the temporary option if it is no longer needed. | Low | [ConstantIslands pass](llvm/lib/Target/Xtensa/XtensaConstantIsland.cpp) |


# Xtensa SelectionDAG Code-Generation Opportunities

This note records candidate SelectionDAG improvements identified by comparing
the Xtensa ISA Summary for Xtensa LX processors with the instruction definitions,
lowering, and code-generation tests in this backend. Xtensa options are
configurable, so each instruction must remain guarded by its corresponding
subtarget feature. The ISA page numbers below refer to the supplied Cadence
Xtensa ISA Summary.

## Recommended candidates

| Priority | Candidate | Current backend behavior | Proposed direction | Important constraints |
|---|---|---|---|---|
| High | `ctlz` to `NSAU` | `XtensaInstrInfo.td` already defines an `NSAU` pattern for `ctlz`, but `XtensaISelLowering.cpp` unconditionally sets `ISD::CTLZ` to `Expand`. The `ctlz-cttz-ctpop.ll` test consequently expects a long software sequence. The ISA Summary specifies that `NSAU` returns 32 for zero and the count of leading zero bits otherwise, matching `ctlz` on `i32`. | Make `CTLZ` legal when `HasNSA` is enabled and retain expansion otherwise. Add a matching `ctlz_zero_undef` pattern if that node can reach instruction selection. | Gate the fast path on `HasNSA`. Test zero and nonzero inputs, CPUs with and without NSA, and the zero-undef intrinsic. |
| Medium | `rotl` / `rotr` through `SRC` | `ROTL` and `ROTR` are expanded. The backend already lowers `FSHL`/`FSHR` to `XtensaISD::SRCL`/`SRCR`, which the DAG selector implements with the `SRC` instruction. | Lower `rotl(x, n)` to `fshl(x, x, n)` and `rotr(x, n)` to `fshr(x, x, n)`, reusing the existing funnel-shift path instead of duplicating `SRC` selection. | Preserve LLVM's modulo-width shift semantics, including zero and variable shifts. Compare generated code and test all relevant shift-count boundaries. |
| Medium | Selects to conditional moves | `MOVEQZ`, `MOVNEZ`, `MOVLTZ`, and `MOVGEZ` are defined for the base core but have no patterns. Integer `SELECT` is expanded and `SELECT_CC` uses a custom inserter that creates branches and a merge. | Add a target lowering for selects based on `x == 0`, `x != 0`, `x < 0`, or `x >= 0`: copy the false value to the destination and conditionally move the true value with the matching instruction. | Restrict to exact zero/sign predicates. Preserve the false value when the move predicate is false, account for tied destination semantics, and compare branch versus move costs; a conditional move is not always faster. |
| Medium | Widened 16-bit multiply to `MUL16S` / `MUL16U` | `MUL16S` and `MUL16U` are defined for `HasMul16`, but have no SelectionDAG patterns. The ISA Summary confirms they multiply the low 16 bits of both operands and write the full 32-bit product (pp. 87, 523-524). | Investigate patterns for `i32` multiplication of sign-extended or zero-extended `i16` operands, selecting the signed or unsigned instruction as appropriate. | Gate on `HasMul16`. Compare generated code against `MULL` or expanded multiply on supported cores; test negative and boundary operands and both feature-enabled and feature-disabled configurations. |
| Medium | Signed saturation to `CLAMPS` | `CLAMPS` is defined for `HasCLAMPS`, but has no generic SelectionDAG pattern. The ISA specifies saturation to a signed N-bit range, with immediate values 7-22, and specifically notes use for saturating arithmetic and narrowing MAC16 results (pp. 94, 384). | Match canonical signed clamp idioms with constant bounds, especially saturating to signed 16-bit before truncating/storing, to `CLAMPS` with the corresponding immediate. | Only match bounds exactly representable by the immediate and preserve the original type's overflow semantics. Gate on `HasCLAMPS`; test both saturation boundaries and feature-disabled fallback. |
| Medium, ISA support missing | Bitfield insertion with `DEPBITS` | The ISA has a `DEPBITS` instruction that inserts 1-16 low source bits at a constant bit position, but the backend currently has no `DEPBITS` instruction definition or subtarget feature. The ISA describes the option and instruction at pp. 96, 394. | Add the option feature and MC/assembler/disassembler support, then select constant-width/constant-position insert idioms such as masked shift-and-OR into an existing word. | This needs instruction encoding and feature plumbing before a DAG pattern. Match only valid widths (1-16), positions (0-31), and ranges where width + position is at most 32; gate on the new feature. |

## Existing patterns to preserve

Several apparent opportunities are already covered and should not be duplicated:

- `ADDX2`, `ADDX4`, `ADDX8`, `SUBX2`, `SUBX4`, and `SUBX8` match scaled
  add/subtract DAG shapes in `XtensaInstrInfo.td`.
- `SEXT` already matches sign extension in-register from `i8` and `i16`.
- `MIN`, `MAX`, `MINU`, and `MAXU` are selected for the corresponding
  min/max operations when `HasMINMAX` is enabled.
- The backend has patterns for `FMA` and Xtensa single-precision fused
  multiply-add/subtract intrinsics.
- The `SRC` instruction is already used by the backend's funnel-shift lowering;
  rotate patterns should reuse that path rather than add another `SRC` selector.
- Xtensa `MAC16` operations use accumulator state and are currently exposed
  through target intrinsics. Do not map ordinary LLVM `mul`/`add` DAGs to these
  instructions unless accumulator behavior, side effects, and data types are
  represented explicitly.
- `NSA` counts redundant sign bits, unlike generic `ctlz`; it should not be
  selected for `ctlz`. `NSAU` is the matching unsigned leading-zero operation.
- Cache, TLB, interrupt, and special-register instructions have architectural
  side effects and are better represented by target intrinsics or explicit
  assembly than by ordinary arithmetic DAG patterns.

## Validation guidance

For each implemented pattern, add or update an LLVM IR code-generation test
under `llvm/test/CodeGen/Xtensa/`. Check both the intended instruction sequence
and feature-disabled fallback behavior. Run the focused Xtensa CodeGen tests
and verify the generated machine instructions, not just successful compilation.

