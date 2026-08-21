### Build instructions 

* Configure CMD :

```
cmake -S <Path to llvm src> -B <Path to build out> -G Ninja -DCMAKE_BUILD_TYPE=Debug -DLLVM_TARGETS_TO_BUILD=AArch64 -DLLVM_INCLUDE_TESTS=ON -DLLVM_BUILD_TESTS=ON -DLLVM_INCLUDE_BENCHMARKS=OFF -DLLVM_INCLUDE_EXAMPLES=OFF -DLLVM_INCLUDE_DOCS=OFF -DLLVM_ENABLE_PROJECTS=
```

* Build only core Backend components :
```
cmake --build <Path to build out> --target llc FileCheck count not split-file llvm-config

```

### Running tests

* Change the path to build output bin dir -> Make sure you have the core llc(llvm static compiler), file check, count, not, etc

```
Generates asm directly: llc.exe -mtriple=aarch64-linux-gnu test.mir -o output.s
Generates mir after the pass but stop before asm printer/obj emission:  llc.exe -mtriple=aarch64-linux-gnu -run-pass=<mir pass name> -verify-machineinstrs test.mir -o output.mir
```
### Condition Codes - AArch64

| Condition |          Value | Meaning         | Common use                      |   |         |
| --------- | -------------: | --------------- | ------------------------------- | - | ------- |
| `EQ`      |  `0x0` / **0** | Equal           | `Z == 1`                        |   |         |
| `NE`      |  `0x1` / **1** | Not equal       | `Z == 0`                        |   |         |
| `HS`      |  `0x2` / **2** | Unsigned ≥      | `C == 1`                        |   |         |
| `LO`      |  `0x3` / **3** | Unsigned <      | `C == 0`                        |   |         |
| `MI`      |  `0x4` / **4** | Negative        | `N == 1`                        |   |         |
| `PL`      |  `0x5` / **5** | Positive / zero | `N == 0`                        |   |         |
| `VS`      |  `0x6` / **6** | Overflow        | `V == 1`                        |   |         |
| `VC`      |  `0x7` / **7** | No overflow     | `V == 0`                        |   |         |
| `HI`      |  `0x8` / **8** | Unsigned >      | `C == 1 && Z == 0`              |   |         |
| `LS`      |  `0x9` / **9** | Unsigned ≤      | `C == 0                         |   | Z == 1` |
| `GE`      | `0xA` / **10** | Signed ≥        | `N == V`                        |   |         |
| `LT`      | `0xB` / **11** | Signed <        | `N != V`                        |   |         |
| `GT`      | `0xC` / **12** | Signed >        | `Z == 0 && N == V`              |   |         |
| `LE`      | `0xD` / **13** | Signed ≤        | `Z == 1                         |   | N != V` |
| `AL`      | `0xE` / **14** | Always          | Unconditional                   |   |         |
| `NV`      | `0xF` / **15** | Always          | Encodes `1111`; executes always |   |         |


# MIR Manipulation — Practical APIs for Beginners

The table above is fairly abstract/legacy-flavored (from the manual itself). This section is the practical set of MIR-manipulation APIs a beginner writing a new pass would actually reach for day-to-day (mostly from `MachineInstr.h`, `MachineBasicBlock.h`, `MachineFunction.h`, `MachineRegisterInfo.h`, `MachineInstrBuilder.h`).

## Inspecting an instruction

| API | Returns | What it's for |
|---|---|---|
| `MI.getOpcode()` | `unsigned` | Get the opcode to switch/compare against |
| `MI.getDesc()` | `const MCInstrDesc&` | Get flags: `mayLoad()`, `mayStore()`, `isCall()`, `isBranch()`, `isCommutable()`, etc. |
| `MI.operands()` | iterator range | Range-for over all `MachineOperand`s |
| `MI.defs()` / `MI.uses()` / `MI.explicit_operands()` | iterator ranges | Filtered operand iteration |
| `MI.getOperand(i)` | `MachineOperand&` | Access a specific operand by index |
| `MI.getNumOperands()` | `unsigned` | Operand count |
| `MI.print(errs())` / `MI.dump()` | `void` | Print for debugging (huge time-saver) |
| `MI.isIdenticalTo(OtherMI)` | `bool` | Structural comparison |

## Inserting / building

| API | Returns | What it's for |
|---|---|---|
| `BuildMI(MBB, InsertPt, DL, TII.get(Opc), DestReg)` | `MachineInstrBuilder` | The main way to create+insert an instruction |
| `BuildMI(MF, DL, TII.get(Opc))` | `MachineInstrBuilder` | Build detached, insert manually later with `MBB.insert(...)` |
| `.addReg(Reg, Flags)` / `.addImm(V)` / `.addMBB(BB)` / `.addFrameIndex(FI)` / `.addGlobalAddress(GV)` | `MachineInstrBuilder&` | Chainable operand-adding methods |
| `MBB.insert(It, MI)` | iterator | Insert an already-built `MachineInstr*` at a position |
| `MBB.insertAfter(It, MI)` | iterator | Insert right after a given instruction |

## Removing / replacing

| API | Returns | What it's for |
|---|---|---|
| `MI.eraseFromParent()` | `void` | The standard way to delete an instruction — beginners often forget this exists and try `delete MI` (don't) |
| `MBB.erase(It)` | iterator | Erase by iterator, returns iterator to next instr |
| `MI.removeFromParent()` | `MachineInstr*` | Unlink without deleting (if you're going to reinsert it elsewhere) |
| `TII.replaceRegWith` / manual `MO.setReg()` loop | — | Common "rewrite all uses of reg X to reg Y" pattern |

## Working with operands/registers

| API | Returns | What it's for |
|---|---|---|
| `MO.isReg()` / `MO.isImm()` / `MO.isMBB()` / `MO.isFI()` etc. | `bool` | Type-check an operand before reading it |
| `MO.setIsKill(bool)` / `MO.isKill()` | `void`/`bool` | Mark/query the last use of a register in its live range |
| `MO.setIsDead(bool)` | `void` | Mark a def as unused afterward |
| `MRI.getRegClass(VReg)` | `const TargetRegisterClass*` | What register class a virtual register belongs to |
| `MRI.setRegClass(VReg, RC)` | `void` | Constrain/change a vreg's class |
| `MRI.def_instructions(VReg)` / `MRI.use_instructions(VReg)` | iterator ranges | Walk all defs/uses of a virtual register — very commonly used |
| `MRI.getVRegDef(VReg)` | `MachineInstr*` | Get the single defining instruction (valid pre-SSA-deconstruction) |
| `MRI.replaceRegWith(FromReg, ToReg)` | `void` | Rewrite every use/def of one vreg to another — extremely handy |

## Blocks and functions

| API | Returns | What it's for |
|---|---|---|
| `MBB.begin()/end()`, `MBB.instrs()` | iterators | Standard iteration; `instrs()` also walks inside bundles |
| `MBB.terminators()` | iterator range | Just the branch/return instructions at block end |
| `MBB.successors()` / `MBB.predecessors()` | iterator ranges | CFG edges |
| `MF.getFrameInfo()` | `MachineFrameInfo&` | Query/create stack objects (`CreateStackObject`, `getObjectSize`, etc.) |
| `MF.print(errs())` / `-print-after-all` (llc flag) | — | Dump the whole function's MIR — the single best debugging tool when starting out |

## Beginner tips that save real time

- `-print-after-all` / `-print-after=<pass-name>` on `llc` dumps MIR text after each pass, so you can literally see what your pass did.
- You can write and load `.mir` files directly (`llc -run-pass=<yourpass> foo.mir`) to unit-test a single pass without going through the whole IR→MIR pipeline — this is the standard way LLVM's own test suite tests codegen passes.
- Almost every "modify an instruction" task in a beginner pass is really one of: build a new `MachineInstr` with `BuildMI`, then `eraseFromParent()` the old one — rather than trying to mutate operands in place. It's more verbose but much harder to get subtly wrong.

