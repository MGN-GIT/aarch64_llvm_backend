
## 🔎 Case Study: Where is CCMN defined, and how does the backend decide to emit it?

This is a worked example of tracing a single instruction (`CCMN`, the "conditional compare negative" instruction) end-to-end through the backend — useful as a template for investigating any other instruction.

### 1. SDNode definition

`AArch64InstrInfo.td` (line 911):
```tablegen
// Conditional compares. Operands: left,right,falsecc,cc,flags
def AArch64ccmn      : SDNode<"AArch64ISD::CCMN",  SDT_AArch64CCMP>;
```
The `AArch64ISD::CCMN` opcode is declared in the `NodeType` enum in `AArch64ISelLowering.h`, alongside `CCMP`/`FCCMP`.

### 2. Instruction definition / TableGen pattern

`AArch64InstrInfo.td` (line 3584):
```tablegen
defm CCMN : CondComparison<0, "ccmn", AArch64ccmn>;
defm CCMP : CondComparison<1, "ccmp", AArch64ccmp>;
```
The `CondComparison` multiclass (in `AArch64InstrFormats.td`, ~line 3626) expands to four instruction variants: `CCMNWi`, `CCMNXi` (register + immediate form) and `CCMNWr`, `CCMNXr` (register + register form). The actual isel pattern lives in the base classes `BaseCondComparisonImm` / `BaseCondComparisonReg` (`AArch64InstrFormats.td`, lines ~3575–3624):
```tablegen
[(set NZCV, (OpNode regtype:$Rn, immtype:$imm, (i32 imm:$nzcv), (i32 imm:$cond), NZCV))]
```
So there **is** a plain TableGen pattern that matches an `AArch64ccmn` DAG node straight to the `CCMN` machine instruction. That's the easy part — the interesting question is *how the `AArch64ISD::CCMN` node gets created in the first place*, since there's no generic IR node for "conditional compare". That happens via custom C++ lowering / DAG combining, not a `Pat<>`.

### 3. Where CCMN nodes actually get created (custom lowering + DAG combine)

All in `AArch64ISelLowering.cpp`:

- **`emitConditionalComparison()`** (~line 4110, part of the `\defgroup AArch64CCMP` CMP;CCMP chain-forming machinery documented at line 4058). Chooses `CCMN` over `CCMP` in three cases:
  - RHS is a constant that's negative and `> -32` (fits `CCMN`'s inverted immediate range) → negate the immediate and switch opcode.
  - `isCMN(RHS, CC, DAG)` is true, i.e. RHS is `(sub 0, op2)` and it's safe to fold into CMN (same helper used for the plain `CMP`→`CMN` fold, see `isCMN()` at line 3980).
  - LHS is `(sub 0, op1)` with an equality (`EQ`/`NE`) compare → commute operands to use `CCMN`.

- **`performANDORCSELCombine()`** (~line 21946) — a genuine target **DAG combine** on `AND`/`OR` of two `CSEL(0, 1, cc)` nodes:
  ```
  (AND (CSET cc0 cmp0) (CSET cc1 (CMP x1 y1)))  =>  (CSET cc1 (CCMP/CCMN x1 y1 ...))
  ```
  Around line 21996–22004: if the immediate operand of the inner `SUBS` is negative and within `(-31, -1]`, it emits `AArch64ISD::CCMN` (using the absolute value of the immediate) instead of `CCMP`, avoiding an extra `mov` to materialize the negated constant. This combine is invoked from `performORCombine`/`performANDCombine` (call sites at lines ~22133, 22342, 29563).

- **`isLegalCondCmpImmediate()`** (~line 12589) documents the rationale in a comment: *"A CCMP folds in only a 5-bit unsigned immediate (or its negation, via CCMN)."*

### Takeaway

CCMN is never matched directly from generic IR — it's synthesized by custom C++ combine/lowering logic that recognizes when a `CCMP` would need a negated immediate (or a `(sub 0, x)`-shaped operand) and substitutes `CCMN` instead, purely as a code-size/instruction-count optimization. Once the `AArch64ISD::CCMN` node exists, ordinary TableGen `Pat<>`/instruction-embedded patterns take over to select it to the real `CCMNWi/Xi/Wr/Xr` machine instructions.

**Files involved:**
| File | Role |
|---|---|
| `AArch64InstrInfo.td` (L911, L3584) | `SDNode` decl + `defm CCMN : CondComparison<...>` |
| `AArch64InstrFormats.td` (~L3575–3641) | `BaseCondComparisonImm/Reg`, `CondComparison` multiclass — instruction encodings + embedded patterns |
| `AArch64ISelLowering.cpp` (`emitConditionalComparison`, `performANDORCSELCombine`, `isCMN`, `isLegalCondCmpImmediate`) | Custom lowering/DAG-combine logic that decides when to emit `CCMN` vs `CCMP` |
 
