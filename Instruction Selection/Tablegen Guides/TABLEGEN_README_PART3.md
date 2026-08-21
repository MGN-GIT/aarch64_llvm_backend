# 📙 TableGen Beginner's Guide — Part 3: Backends & AArch64 `.td` Files

> **Series Overview**
> - **Part 1:** Core concepts, syntax, types, values, and basic statements
> - **Part 2:** multiclass, defm, foreach, bang operators, DAGs
> - **Part 3 (this file):** Backends, AArch64 `.td` file patterns, and real-world usage guide

---

## Table of Contents

1. [How TableGen Backends Work](#1-how-tablegen-backends-work)
2. [The Major LLVM Backends](#2-the-major-llvm-backends)
3. [How Generated `.inc` Files Are Used](#3-how-generated-inc-files-are-used)
4. [The AArch64 TableGen File Hierarchy](#4-the-aarch64-tablegen-file-hierarchy)
5. [Core AArch64 Classes You Must Know](#5-core-aarch64-classes-you-must-know)
6. [Registers and Register Classes in AArch64](#6-registers-and-register-classes-in-aarch64)
7. [Instruction Definitions in AArch64](#7-instruction-definitions-in-aarch64)
8. [Instruction Encoding in AArch64](#8-instruction-encoding-in-aarch64)
9. [Selection Patterns (`Pat` and `PatFrag`)](#9-selection-patterns-pat-and-patfrag)
10. [Subtarget Features and Predicates](#10-subtarget-features-and-predicates)
11. [Scheduling Information](#11-scheduling-information)
12. [Searchable Tables — System Registers](#12-searchable-tables--system-registers)
13. [Reading Real AArch64 `.td` Files](#13-reading-real-aarch64-td-files)
14. [Backend Developer's Quick Guide](#14-backend-developers-quick-guide)
15. [Debugging TableGen Files](#15-debugging-tablegen-files)
16. [Complete AArch64 Instruction Walkthrough](#16-complete-aarch64-instruction-walkthrough)

---

## 1. How TableGen Backends Work

### The Pipeline

```
AArch64.td
    │
    ▼
llvm-tblgen (parser)
    │  Builds in-memory record database
    ▼
Backend (e.g., RegisterInfo, InstrInfo, AsmMatcher)
    │  Reads records, generates C++ code
    ▼
AArch64GenRegisterInfo.inc
AArch64GenInstrInfo.inc
AArch64GenAsmMatcher.inc
... etc.
```

### What a Backend Does

A backend is a C++ program that:
1. Receives the `RecordKeeper` (the database of all parsed records)
2. Queries records by class (e.g., "give me all records derived from `Register`")
3. Reads fields from those records
4. Writes C++ code to an output stream

### The `#ifdef` Guard Pattern

Every generated `.inc` file uses `#ifdef` guards so you can include the same file multiple times to get different pieces of code:

```cpp
// In AArch64RegisterInfo.cpp:
#define GET_REGINFO_TARGET_DESC
#include "AArch64GenRegisterInfo.inc"

// In AArch64RegisterInfo.h:
#define GET_REGINFO_HEADER
#include "AArch64GenRegisterInfo.inc"
```

The macros are automatically `#undef`'d after use inside the `.inc` file.

---

## 2. The Major LLVM Backends

Here are the backends most relevant to understanding AArch64 `.td` files:

### `RegisterInfo` Backend (`-gen-register-info`)

**Input:** Records derived from `Register`, `RegisterClass`, `RegisterTuples`

**Output:** `AArch64GenRegisterInfo.inc`

Generates:
- Enum of all register numbers (`AArch64::X0`, `AArch64::X1`, etc.)
- Register alias tables (which registers overlap)
- Register class membership tables
- Callee-saved register lists

### `InstrInfo` Backend (`-gen-instr-info`)

**Input:** Records derived from `Instruction`

**Output:** `AArch64GenInstrInfo.inc`

Generates:
- Enum of all instruction opcodes
- Instruction property tables (isReturn, isBranch, mayLoad, etc.)
- Operand type tables

### `AsmWriter` Backend (`-gen-asm-writer`)

**Input:** Records with `AsmString` fields

**Output:** `AArch64GenAsmWriter.inc`

Generates:
- `printInstruction()` function that converts a `MachineInstr` to assembly text

### `AsmMatcher` Backend (`-gen-asm-matcher`)

**Input:** Records with `AsmString` and operand info

**Output:** `AArch64GenAsmMatcher.inc`

Generates:
- Assembly parser that converts text like `"add x0, x1, x2"` into a `MCInst`

### `DAGISel` Backend (`-gen-dag-isel`)

**Input:** Records with `Pattern` fields (DAG patterns)

**Output:** `AArch64GenDAGISel.inc`

Generates:
- Instruction selection table used by `SelectionDAGISel`
- Matches IR patterns to machine instructions

### `CodeEmitter` Backend (`-gen-emitter`)

**Input:** Records with `Inst` (encoding) fields

**Output:** `AArch64GenMCCodeEmitter.inc`

Generates:
- `getBinaryCodeForInstr()` — converts a `MCInst` to its binary encoding

### `Subtarget` Backend (`-gen-subtarget`)

**Input:** Records derived from `SubtargetFeature`, `Processor`

**Output:** `AArch64GenSubtargetInfo.inc`

Generates:
- Feature flags and CPU feature tables
- `ParseSubtargetFeatures()` function

### `SearchableTables` Backend (`-gen-searchable-tables`)

**Input:** Records derived from `GenericTable`, `GenericEnum`, `SearchIndex`

**Output:** Various `.inc` files

Generates:
- Searchable C++ tables (used for system registers in AArch64)

---

## 3. How Generated `.inc` Files Are Used

### Example: Register Info

```cpp
// AArch64GenRegisterInfo.inc (generated)
#ifdef GET_REGINFO_ENUM
namespace AArch64 {
enum {
  NoRegister,
  FP = 1,
  LR = 2,
  NZCV = 3,
  SP = 4,
  WSP = 5,
  WZR = 6,
  XZR = 7,
  B0 = 8,
  // ... hundreds more
  X0 = 309,
  X1 = 310,
  // ...
};
} // end namespace AArch64
#endif // GET_REGINFO_ENUM
```

```cpp
// AArch64RegisterInfo.h
#define GET_REGINFO_ENUM
#include "AArch64GenRegisterInfo.inc"
```

### Example: Instruction Info

```cpp
// AArch64GenInstrInfo.inc (generated)
#ifdef GET_INSTRINFO_ENUM
namespace AArch64 {
enum {
  PHI = 0,
  INLINEASM = 1,
  // ...
  ADDWri = 1234,
  ADDWrr = 1235,
  ADDXri = 1236,
  ADDXrr = 1237,
  // ...
};
} // end namespace AArch64
#endif
```

---

## 4. The AArch64 TableGen File Hierarchy

Understanding which file defines what is essential for navigating AArch64 `.td` files.

```
llvm/lib/Target/AArch64/
├── AArch64.td                    ← Root file, includes everything
├── AArch64RegisterInfo.td        ← Register definitions
├── AArch64InstrInfo.td           ← Top-level instruction includes
├── AArch64InstrFormats.td        ← Instruction encoding formats
├── AArch64InstrArithmetic.td     ← ADD, SUB, MUL, etc.
├── AArch64InstrBranch.td         ← B, BL, CBZ, etc.
├── AArch64InstrLoad.td           ← LDR, LDRB, LDRSW, etc.
├── AArch64InstrStore.td          ← STR, STRB, etc.
├── AArch64InstrFP.td             ← Floating-point instructions
├── AArch64InstrSIMD.td           ← NEON/SIMD instructions
├── AArch64InstrSVE.td            ← SVE instructions
├── AArch64InstrAtomics.td        ← Atomic instructions
├── AArch64InstrSystem.td         ← System instructions (MSR, MRS, etc.)
├── AArch64SchedA53.td            ← Cortex-A53 scheduling
├── AArch64SchedA57.td            ← Cortex-A57 scheduling
├── AArch64SchedFalkor.td         ← Falkor scheduling
├── AArch64Features.td            ← SubtargetFeature definitions
├── AArch64Processors.td          ← CPU definitions
└── AArch64SystemOperands.td      ← System registers (searchable tables)

llvm/include/llvm/Target/
├── Target.td                     ← Core LLVM target classes
├── TargetSchedule.td             ← Scheduling classes
├── TargetSelectionDAG.td         ← DAG node definitions
└── TargetInstrInfo.td            ← Instruction base classes
```

### The Include Chain

```tablegen
// AArch64.td
include "llvm/Target/Target.td"
include "AArch64RegisterInfo.td"
include "AArch64Features.td"
include "AArch64InstrInfo.td"
include "AArch64Processors.td"
```

```tablegen
// AArch64InstrInfo.td
include "AArch64InstrFormats.td"
include "AArch64InstrArithmetic.td"
include "AArch64InstrBranch.td"
// ... etc.
```

---

## 5. Core AArch64 Classes You Must Know

### From `llvm/include/llvm/Target/Target.td`

#### `Register`

```tablegen
class Register<string n, list<string> altNames = []> {
  string Namespace = "";
  string AsmName = n;
  list<string> AltNames = altNames;
  list<Register> Aliases = [];
  list<Register> SubRegs = [];
  list<SubRegIndex> SubRegIndices = [];
  list<RegAltNameIndex> RegAltNameIndices = [];
  list<int> DwarfNumbers = [];
  int CostPerUse = 0;
  bit CoveredBySubRegs = false;
  bit isArtificial = false;
  bit isConstant = false;
}
```

#### `RegisterClass`

```tablegen
class RegisterClass<string namespace, list<ValueType> regTypes,
                    int alignment, dag regList, RegAltNameIndex idx = NoRegAltName> {
  string Namespace = namespace;
  list<ValueType> RegTypes = regTypes;
  int Size = 0;
  int Alignment = alignment;
  int CopyCost = 1;
  dag MemberList = regList;
  // ... more fields
}
```

#### `Instruction` (the most important class!)

```tablegen
class Instruction {
  string Namespace = "";
  dag OutOperandList;          // Output operands
  dag InOperandList;           // Input operands
  string AsmString = "";       // Assembly syntax
  list<dag> Pattern = [];      // Selection patterns
  list<Register> Uses = [];    // Implicitly used registers
  list<Register> Defs = [];    // Implicitly defined registers
  list<Predicate> Predicates = [];  // Feature guards

  // Instruction properties:
  bit isReturn = 0;
  bit isBranch = 0;
  bit isIndirectBranch = 0;
  bit isCompare = 0;
  bit isMoveImm = 0;
  bit isBarrier = 0;
  bit isCall = 0;
  bit isAdd = 0;
  bit isTrap = 0;
  bit canFoldAsLoad = 0;
  bit mayLoad = ?;
  bit mayStore = ?;
  bit isCommutable = 0;
  bit isTerminator = 0;
  bit isReMaterializable = 0;
  bit isPredicable = 0;
  bit hasDelaySlot = 0;
  bit usesCustomInserter = 0;
  bit hasSideEffects = ?;
  bit isPseudo = 0;
  // ... many more
}
```

#### `SubtargetFeature`

```tablegen
class SubtargetFeature<string n, string a, string v, string d,
                       list<SubtargetFeature> i = []> {
  string Name = n;           // Feature name (e.g., "neon")
  string Attribute = a;      // C++ attribute name (e.g., "HasNEON")
  string Value = v;          // Value to set (usually "true")
  string Desc = d;           // Human-readable description
  list<SubtargetFeature> Implies = i;  // Implied features
}
```

#### `Processor`

```tablegen
class Processor<string n, ProcessorItineraries itin,
                list<SubtargetFeature> f> {
  string Name = n;
  ProcessorItineraries ProcItin = itin;
  list<SubtargetFeature> Features = f;
}
```

---

## 6. Registers and Register Classes in AArch64

### How AArch64 Registers Are Defined

```tablegen
// AArch64RegisterInfo.td (simplified)

// Define sub-register indices
def sub_32 : SubRegIndex<32>;   // 32-bit sub-register of 64-bit register

// Define a 64-bit general-purpose register
class AArch64Reg<bits<16> enc, string n, list<string> alt = []>
    : Register<n, alt> {
  let HWEncoding = enc;
  let Namespace = "AArch64";
}

// Define 32-bit sub-register
class AArch64RegWithSubs<bits<16> enc, string n,
                          list<Register> subregs = [],
                          list<string> alt = []>
    : AArch64Reg<enc, n, alt> {
  let SubRegs = subregs;
  let SubRegIndices = [sub_32];
}

// The actual register definitions:
def W0  : AArch64Reg<0,  "w0">,  DwarfRegNum<[0]>;
def W1  : AArch64Reg<1,  "w1">,  DwarfRegNum<[1]>;
// ... W2 through W30

def X0  : AArch64RegWithSubs<0,  "x0",  [W0]>,  DwarfRegNum<[0]>;
def X1  : AArch64RegWithSubs<1,  "x1",  [W1]>,  DwarfRegNum<[1]>;
// ... X2 through X30

def XZR : AArch64Reg<31, "xzr">;   // Zero register (64-bit)
def WZR : AArch64Reg<31, "wzr">;   // Zero register (32-bit)
def SP  : AArch64Reg<31, "sp">;    // Stack pointer
def WSP : AArch64Reg<31, "wsp">;   // Stack pointer (32-bit)
```

### Register Classes

```tablegen
// 64-bit general-purpose registers
def GPR64 : RegisterClass<"AArch64", [i64], 64, (add
  X0, X1, X2, X3, X4, X5, X6, X7,
  X8, X9, X10, X11, X12, X13, X14, X15,
  X16, X17, X18, X19, X20, X21, X22, X23,
  X24, X25, X26, X27, X28, X29, X30, XZR
)>;

// 32-bit general-purpose registers
def GPR32 : RegisterClass<"AArch64", [i32], 32, (add
  W0, W1, W2, W3, W4, W5, W6, W7,
  W8, W9, W10, W11, W12, W13, W14, W15,
  W16, W17, W18, W19, W20, W21, W22, W23,
  W24, W25, W26, W27, W28, W29, W30, WZR
)>;

// 64-bit FP/SIMD registers
def FPR64 : RegisterClass<"AArch64", [f64, v2f32, v2i32, v4i16, v8i8], 64, (add
  D0, D1, D2, D3, D4, D5, D6, D7,
  D8, D9, D10, D11, D12, D13, D14, D15,
  D16, D17, D18, D19, D20, D21, D22, D23,
  D24, D25, D26, D27, D28, D29, D30, D31
)>;
```

### Reading Register Definitions

When you see in AArch64 `.td` files:

```tablegen
def ADDXrr : I<0b10001011000, (outs GPR64:$Rd), (ins GPR64:$Rn, GPR64:$Rm), ...>
```

- `GPR64` is the register class (64-bit general-purpose registers)
- `:$Rd` is the name of this operand (used in patterns and assembly strings)
- `(outs ...)` means this is an output (destination) operand
- `(ins ...)` means these are input (source) operands

---

## 7. Instruction Definitions in AArch64

### The AArch64 Instruction Class Hierarchy

```
Instruction                    (from Target.td)
  └── InstAArch64              (AArch64-specific base)
        └── I                  (common AArch64 instruction)
              ├── InstReg      (register operand instructions)
              ├── InstImm      (immediate operand instructions)
              ├── InstMem      (memory access instructions)
              └── ...
```

### A Simplified AArch64 Instruction Class

```tablegen
// From AArch64InstrFormats.td (simplified)
class InstAArch64<dag outs, dag ins, string asmstr, list<dag> pattern>
    : Instruction {
  let Namespace = "AArch64";
  let OutOperandList = outs;
  let InOperandList  = ins;
  let AsmString      = asmstr;
  let Pattern        = pattern;
  field bits<32> Inst;   // The 32-bit encoding
  field bits<32> SoftFail = 0;
}
```

### A Real AArch64 ADD Instruction

```tablegen
// From AArch64InstrArithmetic.td (simplified)

// The base class for ADD/SUB with shifted register
class AddSubShiftedReg<bit isSub, bit setFlags, RegisterClass regtype,
                       string asm, list<dag> pattern>
    : I<(outs regtype:$Rd),
        (ins regtype:$Rn, regtype:$Rm, arith_shift64:$shift),
        asm # "\t$Rd, $Rn, $Rm$shift",
        pattern> {
  bits<5> Rd;
  bits<5> Rn;
  bits<5> Rm;
  bits<8> shift;

  let Inst{31}    = regtype == GPR64 ? 1 : 0;  // sf bit: 1=64-bit, 0=32-bit
  let Inst{30}    = isSub;                       // op bit: 1=SUB, 0=ADD
  let Inst{29}    = setFlags;                    // S bit: 1=set flags
  let Inst{28-24} = 0b01011;                     // Fixed encoding
  let Inst{23-22} = shift{7-6};                  // Shift type
  let Inst{21}    = 0;
  let Inst{20-16} = Rm;
  let Inst{15-10} = shift{5-0};                  // Shift amount
  let Inst{9-5}   = Rn;
  let Inst{4-0}   = Rd;
}

// Concrete definitions using multiclass:
multiclass AddSub<bit isSub, string mnemonic, string cond_mnemonic,
                  SDNode OpNode = ?> {
  // 64-bit register-register with shift
  def Xrs : AddSubShiftedReg<isSub, 0, GPR64, mnemonic,
    [(set GPR64:$Rd, (OpNode GPR64:$Rn, (shift_reg GPR64:$Rm, i32imm:$shift)))]>;

  // 32-bit register-register with shift
  def Wrs : AddSubShiftedReg<isSub, 0, GPR32, mnemonic,
    [(set GPR32:$Rd, (OpNode GPR32:$Rn, (shift_reg GPR32:$Rm, i32imm:$shift)))]>;

  // 64-bit register-immediate
  def Xri : AddSubImmInst<isSub, 0, GPR64, mnemonic,
    [(set GPR64:$Rd, (OpNode GPR64:$Rn, addsub_shifted_imm64:$imm))]>;

  // 32-bit register-immediate
  def Wri : AddSubImmInst<isSub, 0, GPR32, mnemonic,
    [(set GPR32:$Rd, (OpNode GPR32:$Rn, addsub_shifted_imm32:$imm))]>;
}

defm ADD : AddSub<0, "add", "eq", add>;
defm SUB : AddSub<1, "sub", "ne", sub>;
```

This creates: `ADDXrs`, `ADDWrs`, `ADDXri`, `ADDWri`, `SUBXrs`, `SUBWrs`, `SUBXri`, `SUBWri`

---

## 8. Instruction Encoding in AArch64

### How Encoding Works

AArch64 instructions are always 32 bits. The encoding is specified using `bits<32> Inst` and `let Inst{...} = ...` assignments.

### Bit Field Assignment

```tablegen
class MyInst : InstAArch64<...> {
  bits<5>  Rd;      // Destination register (5 bits = 32 registers)
  bits<5>  Rn;      // Source register 1
  bits<5>  Rm;      // Source register 2
  bits<12> imm12;   // 12-bit immediate

  // Assign bits to the 32-bit encoding:
  let Inst{31}    = 1;        // sf = 1 (64-bit operation)
  let Inst{30}    = 0;        // op = 0 (ADD, not SUB)
  let Inst{29}    = 0;        // S = 0 (don't set flags)
  let Inst{28-24} = 0b10001;  // Fixed opcode bits
  let Inst{23-22} = 0b00;     // Shift = LSL
  let Inst{21-10} = imm12;    // 12-bit immediate value
  let Inst{9-5}   = Rn;       // Source register
  let Inst{4-0}   = Rd;       // Destination register
}
```

### Reading Encoding Patterns

When you see in AArch64 `.td` files:

```tablegen
let Inst{31} = sf;
let Inst{30} = op;
let Inst{29} = S;
let Inst{28-23} = 0b100010;
let Inst{22} = shift;
let Inst{21-10} = imm12;
let Inst{9-5} = Rn;
let Inst{4-0} = Rd;
```

This directly maps to the AArch64 Architecture Reference Manual's instruction encoding diagrams. Each bit range corresponds to a field in the binary encoding.

### The `SoftFail` Field

```tablegen
field bits<32> SoftFail = 0;
```

`SoftFail` is used by the disassembler. Bits set in `SoftFail` are "don't care" bits — if they don't match, the instruction is still decoded but flagged as potentially invalid.

---

## 9. Selection Patterns (`Pat` and `PatFrag`)

### What Selection Patterns Do

Selection patterns tell the LLVM instruction selector: "When you see this IR pattern, use this instruction."

### The `Pat` Class

```tablegen
// Syntax: def : Pat<(IRPattern), (InstructionPattern)>;

// Match a 64-bit add and use ADDXrr:
def : Pat<(i64 (add GPR64:$Rn, GPR64:$Rm)),
          (ADDXrr GPR64:$Rn, GPR64:$Rm)>;

// Match a zero-extended load and use LDRB:
def : Pat<(i64 (zextloadi8 GPR64sp:$Rn)),
          (LDRBBroX GPR64sp:$Rn, XZR, 0, 0)>;
```

### `PatFrag` — Reusable Pattern Fragments

```tablegen
// Define a reusable pattern fragment:
def load_acquire : PatFrag<(ops node:$ptr),
                            (load node:$ptr),
                            [{ return cast<LoadSDNode>(N)->getOrdering() ==
                                      AtomicOrdering::Acquire; }]>;

// Use it in a pattern:
def : Pat<(i64 (load_acquire GPR64sp:$Rn)),
          (LDARX GPR64sp:$Rn)>;
```

### Complex Patterns with `!cast`

A very common AArch64 pattern uses `!cast` to build instruction names dynamically:

```tablegen
multiclass ro_signed_pats<string T, string Rm, dag Base, dag Offset,
                          dag Extend, dag address, ValueType sty> {
  // Match sign-extending load to 32-bit result:
  def : Pat<(i32 (!cast<SDNode>("sextload" # sty) address)),
            (!cast<Instruction>("LDRS" # T # "w_" # Rm # "_RegOffset")
              Base, Offset, Extend)>;

  // Match sign-extending load to 64-bit result:
  def : Pat<(i64 (!cast<SDNode>("sextload" # sty) address)),
            (!cast<Instruction>("LDRS" # T # "x_" # Rm # "_RegOffset")
              Base, Offset, Extend)>;
}
```

Here:
- `"sextload" # sty` builds a string like `"sextloadi8"` or `"sextloadi16"`
- `!cast<SDNode>(...)` looks up the DAG node with that name
- `"LDRS" # T # "w_" # Rm # "_RegOffset"` builds an instruction name like `"LDRSBw_GPR64_RegOffset"`
- `!cast<Instruction>(...)` looks up that instruction record

### The `(set ...)` DAG Operator

In instruction patterns, `set` is used to indicate which operand receives the result:

```tablegen
list<dag> Pattern = [(set GPR64:$Rd, (add GPR64:$Rn, GPR64:$Rm))];
//                   ^^^             ^^^^^^^^^^^^^^^^^^^^^^^^^^^
//                   Output          IR operation to match
```

---

## 10. Subtarget Features and Predicates

### Defining Features

```tablegen
// AArch64Features.td (simplified)

def FeatureNEON : SubtargetFeature<"neon", "HasNEON", "true",
    "Enable Advanced SIMD instructions (NEON)">;

def FeatureSVE : SubtargetFeature<"sve", "HasSVE", "true",
    "Enable Scalable Vector Extension (SVE) instructions",
    [FeatureNEON]>;   // SVE implies NEON

def FeatureCRC : SubtargetFeature<"crc", "HasCRC", "true",
    "Enable ARMv8 CRC-32 checksum instructions">;

def FeatureCrypto : SubtargetFeature<"crypto", "HasCrypto", "true",
    "Enable cryptographic instructions",
    [FeatureNEON]>;   // Crypto implies NEON
```

### Defining Predicates

```tablegen
// Predicates are used to guard instruction availability:
def HasNEON : Predicate<"Subtarget->hasNEON()">,
              AssemblerPredicate<(all_of FeatureNEON), "NEON">;

def HasSVE  : Predicate<"Subtarget->hasSVE()">,
              AssemblerPredicate<(all_of FeatureSVE), "SVE">;

def HasCRC  : Predicate<"Subtarget->hasCRC()">,
              AssemblerPredicate<(all_of FeatureCRC), "CRC">;
```

### Using Predicates

```tablegen
// Guard a single instruction:
def FADDP : FPInst<...> {
  let Predicates = [HasNEON];
}

// Guard a group of instructions:
let Predicates = [HasNEON] in {
  defm VADD : SIMDThreeSameVector<0, 0, 0b11010, "fadd", fadd>;
  defm VSUB : SIMDThreeSameVector<1, 0, 0b11010, "fsub", fsub>;
}

// Guard with multiple predicates (all must be true):
let Predicates = [HasSVE, HasBF16] in {
  def BFMLA_ZPmZZ : SVEInst<...>;
}
```

### Defining CPU Processors

```tablegen
// AArch64Processors.td (simplified)

def ProcA53 : SubtargetFeature<"a53", "ARMProcFamily", "CortexA53",
    "Cortex-A53 ARM processors">;

def TuneA53 : SubtargetFeature<"tune-a53", "ARMProcFamily", "CortexA53",
    "Cortex-A53 tuning">;

def ProcessorFeatures {
  list<SubtargetFeature> A53 = [
    FeatureFPARMv8, FeatureNEON, FeatureCRC, FeatureCrypto,
    FeaturePerfMon, FeaturePostRAScheduler, FeatureSlowMisaligned128Store,
    FeatureUseRSqrt, ProcA53
  ];
}

def : ProcessorModel<"cortex-a53", CortexA53Model,
                     ProcessorFeatures.A53>;
```

---

## 11. Scheduling Information

### Why Scheduling Matters

The scheduler needs to know how long each instruction takes and what CPU resources it uses. This is described in TableGen.

### Scheduling Classes

```tablegen
// Define a scheduling write (latency of producing a result):
def WriteALU : SchedWrite;
def WriteLD  : SchedWrite;
def WriteST  : SchedWrite;
def WriteFPALU : SchedWrite;

// Define a scheduling read (when an operand is consumed):
def ReadALU : SchedRead;
```

### Processor Resource Model

```tablegen
// AArch64SchedA53.td (simplified)

def CortexA53Model : SchedMachineModel {
  let IssueWidth = 2;          // 2 instructions per cycle
  let MicroOpBufferSize = 0;   // In-order processor
  let LoadLatency = 3;
  let MispredictPenalty = 8;
}

// Define execution units:
def A53UnitALU  : ProcResource<2>;   // 2 ALU units
def A53UnitMul  : ProcResource<1>;   // 1 multiply unit
def A53UnitLoad : ProcResource<1>;   // 1 load unit

// Define write latencies for this processor:
def : WriteRes<WriteALU, [A53UnitALU]> {
  let Latency = 1;
}

def : WriteRes<WriteLD, [A53UnitLoad]> {
  let Latency = 3;
}
```

### Attaching Scheduling to Instructions

```tablegen
// In instruction definitions:
def ADDXrr : I<...> {
  let SchedRW = [WriteALU, ReadALU, ReadALU];
  //             ^^^^^^^^  ^^^^^^^^  ^^^^^^^^
  //             Output    Input 1   Input 2
}
```

---

## 12. Searchable Tables — System Registers

AArch64 has hundreds of system registers (accessed via `MSR`/`MRS` instructions). These are defined using the `SearchableTables` backend.

### How System Registers Are Defined

```tablegen
// AArch64SystemOperands.td (simplified)

// Define the enum type for system register encodings:
def SysRegValues : GenericEnum {
  let FilterClass = "SysReg";
  let NameField = "Name";
  let ValueField = "Encoding";
}

// Define the searchable table:
def SysRegsList : GenericTable {
  let FilterClass = "SysReg";
  let Fields = ["Name", "Encoding", "Readable", "Writeable", "FeaturesRequired"];
  string TypeOf_FeaturesRequired = "code";
  let PrimaryKey = ["Encoding"];
  let PrimaryKeyName = "lookupSysRegByEncoding";
}

// Define a secondary search index by name:
def lookupSysRegByName : SearchIndex {
  let Table = SysRegsList;
  let Key = ["Name"];
}

// Base class for system registers:
class SysReg<string name, bits<16> enc> {
  string Name = name;
  bits<16> Encoding = enc;
  bit Readable = 1;
  bit Writeable = 1;
  code FeaturesRequired = [{}];
}

// Actual system register definitions:
def NZCV    : SysReg<"nzcv",    0xDA10>;
def FPCR    : SysReg<"fpcr",    0xDA20>;
def FPSR    : SysReg<"fpsr",    0xDA21>;
def DAIF    : SysReg<"daif",    0xDA11>;
def CurrentEL : SysReg<"currentel", 0xD212> {
  let Writeable = 0;   // Read-only
}
def SP_EL0  : SysReg<"sp_el0",  0xC208>;
def SP_EL1  : SysReg<"sp_el1",  0xC210>;
// ... hundreds more
```

### Generated C++ Code

The `SearchableTables` backend generates:

```cpp
// Lookup by encoding (binary search):
const SysReg *lookupSysRegByEncoding(uint16_t Encoding);

// Lookup by name (binary search on uppercase name):
const SysReg *lookupSysRegByName(StringRef Name);
```

This is used by the AArch64 assembler/disassembler to validate and print system register names.

---

## 13. Reading Real AArch64 `.td` Files

### Step-by-Step: Decoding an Instruction Definition

Let's decode a real AArch64 instruction definition. Here's a simplified version of `LDRXui` (Load Register, unsigned immediate offset):

```tablegen
// Step 1: Find the encoding format class
class LoadStoreUI<bits<2> sz, bit V, bits<2> opc, dag outs, dag ins,
                  string asmstr, list<dag> pattern>
    : I<outs, ins, asmstr, pattern> {
  bits<5>  Rt;      // Target register
  bits<5>  Rn;      // Base register
  bits<12> offset;  // Unsigned offset (scaled)

  let Inst{31-30} = sz;      // Size: 11=64-bit, 10=32-bit, 01=16-bit, 00=8-bit
  let Inst{29-27} = 0b111;   // Fixed
  let Inst{26}    = V;       // V=0: integer, V=1: FP/SIMD
  let Inst{25-24} = 0b01;    // Fixed
  let Inst{23-22} = opc;     // Operation: 01=load, 00=store
  let Inst{21-10} = offset;  // 12-bit unsigned offset
  let Inst{9-5}   = Rn;      // Base register
  let Inst{4-0}   = Rt;      // Target register
}

// Step 2: Instantiate for 64-bit load
def LDRXui : LoadStoreUI<0b11, 0, 0b01,
    (outs GPR64:$Rt),
    (ins GPR64sp:$Rn, uimm12s8:$offset),
    "ldr\t$Rt, [$Rn, $offset]",
    [(set (i64 GPR64:$Rt), (load (am_indexed64 GPR64sp:$Rn, uimm12s8:$offset)))]>;
```

**Reading this:**
- `sz = 0b11` → 64-bit operation
- `V = 0` → integer register
- `opc = 0b01` → load operation
- `(outs GPR64:$Rt)` → one output: a 64-bit register named `$Rt`
- `(ins GPR64sp:$Rn, uimm12s8:$offset)` → two inputs: base register and scaled immediate
- `"ldr\t$Rt, [$Rn, $offset]"` → assembly syntax
- The `Pattern` matches `(load (am_indexed64 ...))` IR pattern

### Step-by-Step: Decoding a multiclass

```tablegen
// From AArch64InstrLoad.td (simplified)

// Multiclass for all load sizes:
multiclass Load<bits<2> sz, bit V, bits<2> opc, RegisterClass regtype,
                string asm, ValueType Ty, PatFrag loadop> {
  // Unsigned immediate offset:
  def ui : LoadStoreUI<sz, V, opc,
      (outs regtype:$Rt),
      (ins GPR64sp:$Rn, uimm12s8:$offset),
      asm # "\t$Rt, [$Rn, $offset]",
      [(set (Ty regtype:$Rt), (loadop (am_indexed64 GPR64sp:$Rn, uimm12s8:$offset)))]>;

  // Register offset:
  def roX : LoadStoreRO<sz, V, opc, regtype, asm, Ty, loadop>;

  // Pre-indexed:
  def pre : LoadStorePreIdx<sz, V, opc, regtype, asm, Ty, loadop>;

  // Post-indexed:
  def post : LoadStorePostIdx<sz, V, opc, regtype, asm, Ty, loadop>;
}

// Instantiate for each size:
defm LDRB  : Load<0b00, 0, 0b01, GPR32, "ldrb",  i32, zextloadi8>;
defm LDRH  : Load<0b01, 0, 0b01, GPR32, "ldrh",  i32, zextloadi16>;
defm LDRW  : Load<0b10, 0, 0b01, GPR32, "ldr",   i32, load>;
defm LDRX  : Load<0b11, 0, 0b01, GPR64, "ldr",   i64, load>;
```

This creates: `LDRBui`, `LDRBroX`, `LDRBpre`, `LDRBpost`, `LDRHui`, ..., `LDRXpost`

### Common Patterns to Recognize

| Pattern | What it means |
|---------|---------------|
| `let Predicates = [HasNEON] in` | Only available when NEON is enabled |
| `let isCommutable = 1 in` | Operands can be swapped (e.g., ADD) |
| `let mayLoad = 1 in` | Instruction reads memory |
| `let mayStore = 1 in` | Instruction writes memory |
| `let Defs = [NZCV] in` | Instruction sets condition flags |
| `let Uses = [SP] in` | Instruction reads the stack pointer |
| `!cast<Instruction>("LD" # sz)` | Dynamic instruction lookup by name |
| `(outs GPR64:$Rd)` | Output: 64-bit register |
| `(ins GPR64:$Rn, GPR64:$Rm)` | Inputs: two 64-bit registers |
| `[(set GPR64:$Rd, (add ...))]` | Selection pattern |
| `def : Pat<...>` | Anonymous pattern record |

---

## 14. Backend Developer's Quick Guide

### Data Structures

| C++ Class | What it represents |
|-----------|-------------------|
| `RecordKeeper` | The entire database of all records |
| `Record` | A single class or def record |
| `RecordVal` | A single field within a record |
| `RecTy` | The type of a field |
| `Init` | The value of a field |
| `DefInit` | A value that is a reference to a record |
| `IntInit` | An integer value |
| `StringInit` | A string value |
| `ListInit` | A list value |
| `DagInit` | A DAG value |
| `BitsInit` | A bits<n> value |

### Getting Records in a Backend

```cpp
// Get all records derived from "Instruction":
auto Instructions = Records.getAllDerivedDefinitions("Instruction");
for (Record *Inst : Instructions) {
  StringRef Name = Inst->getName();
  StringRef AsmStr = Inst->getValueAsString("AsmString");
  bool isReturn = Inst->getValueAsBit("isReturn");
  // ...
}

// Get a specific record by name:
Record *AddInst = Records.getDef("ADDXrr");

// Get a field value:
int64_t Opcode = Rec->getValueAsInt("Opcode");
std::string Name = Rec->getValueAsString("Name");
bool Flag = Rec->getValueAsBit("isCall");
std::vector<Record*> Uses = Rec->getValueAsListOfDefs("Uses");
```

### Writing a Simple Backend

```cpp
// MyBackend.cpp
#include "llvm/TableGen/Record.h"
#include "llvm/TableGen/TableGenBackend.h"

using namespace llvm;

namespace {

class MyEmitter {
  RecordKeeper &Records;
public:
  MyEmitter(RecordKeeper &RK) : Records(RK) {}

  void run(raw_ostream &OS) {
    emitSourceFileHeader("My Generated File", OS);

    OS << "#ifdef GET_MY_DATA\n";

    auto Defs = Records.getAllDerivedDefinitions("MyClass");
    for (Record *R : Defs) {
      OS << "  { \"" << R->getName() << "\", "
         << R->getValueAsInt("Value") << " },\n";
    }

    OS << "#endif // GET_MY_DATA\n";
  }
};

} // anonymous namespace

// Register the backend:
static TableGen::Emitter::OptClass<MyEmitter>
    X("gen-my-data", "Generate my data tables");
```

---

## 15. Debugging TableGen Files

### Tool 1: Print All Records

```bash
# See what all records expand to:
llvm-tblgen AArch64.td --print-records 2>&1 | grep -A 20 "def ADDXrr"
```

### Tool 2: Print Detailed Records

```bash
# See records with source locations:
llvm-tblgen AArch64.td --print-detailed-records 2>&1 | less
```

### Tool 3: JSON Output

```bash
# Get all records as JSON for scripting:
llvm-tblgen AArch64.td --dump-json > aarch64.json
python3 -c "
import json
data = json.load(open('aarch64.json'))
# Find all instructions:
for name in data['!instanceof']['Instruction']:
    inst = data[name]
    print(name, inst.get('AsmString', ''))
"
```

### Tool 4: `dump` Statement

```tablegen
// Add temporary debug output to your .td file:
multiclass MyMC<int x> {
  dump "Instantiating MyMC with x = " # !repr(x);
  def : SomeClass<x>;
}
```

### Tool 5: `assert` for Validation

```tablegen
class CheckedInst<bits<8> opc> {
  assert !lt(opc, 256), "Opcode out of range: " # !repr(opc);
  bits<8> Opcode = opc;
}
```

### Tool 6: `-print-enums`

```bash
# List all records of a specific class:
llvm-tblgen AArch64.td -print-enums -class=Register
llvm-tblgen AArch64.td -print-enums -class=Instruction
```

### Common Debugging Scenarios

**Problem:** "I don't know what fields a record has"
```bash
llvm-tblgen AArch64.td --print-records 2>&1 | grep -A 100 "^def ADDXrr {"
```

**Problem:** "I don't know which class defines a field"
```bash
llvm-tblgen AArch64.td --print-detailed-records 2>&1 | grep -A 5 "AsmString"
```

**Problem:** "My multiclass isn't generating the right records"
```tablegen
// Add dump statements:
multiclass MyMC<string name> {
  dump "Creating record: " # NAME # "_" # name;
  def _#name : SomeClass;
}
```

**Problem:** "I want to see the full expansion of a specific instruction"
```bash
llvm-tblgen AArch64.td --print-records 2>&1 | \
  awk '/^def LDRXui \{/,/^\}/' 
```

---

## 16. Complete AArch64 Instruction Walkthrough

Let's trace a complete example: the AArch64 `ADD X0, X1, X2` instruction.

### Step 1: The `.td` Definition

```tablegen
// In AArch64InstrArithmetic.td:
defm ADD : AddSub<0, "add", "eq", add>;
```

This invokes the `AddSub` multiclass with `isSub=0`, `mnemonic="add"`, `OpNode=add`.

### Step 2: The multiclass Expansion

The `AddSub` multiclass creates (among others):

```tablegen
def ADDXrs : AddSubShiftedReg<0, 0, GPR64, "add",
    [(set GPR64:$Rd, (add GPR64:$Rn, (shift_reg GPR64:$Rm, i32imm:$shift)))]>;
```

### Step 3: The Fully Expanded Record

After TableGen processes everything, `ADDXrs` looks like:

```
def ADDXrs {   // Instruction InstAArch64 I AddSubShiftedReg
  string Namespace = "AArch64";
  dag OutOperandList = (outs GPR64:$Rd);
  dag InOperandList = (ins GPR64:$Rn, GPR64:$Rm, arith_shift64:$shift);
  string AsmString = "add\t$Rd, $Rn, $Rm$shift";
  list<dag> Pattern = [(set GPR64:$Rd, (add GPR64:$Rn, (shift_reg GPR64:$Rm, i32imm:$shift)))];
  list<Register> Uses = [];
  list<Register> Defs = [];
  list<Predicate> Predicates = [];
  bit isReturn = 0;
  bit isBranch = 0;
  bit isCall = 0;
  bit isCommutable = 1;
  bit mayLoad = 0;
  bit mayStore = 0;
  bit hasSideEffects = 0;
  // ... many more fields ...
  bits<32> Inst = { 1, 0, 0, 0, 1, 0, 1, 1, 0, 0, 0, ... };
  //               ^sf ^op ^S  ^^^^^^^^^^^^ fixed bits
}
```

### Step 4: What Each Backend Does With It

| Backend | What it reads | What it generates |
|---------|--------------|-------------------|
| `InstrInfo` | `Namespace`, properties | `AArch64::ADDXrs` enum value, property table |
| `AsmWriter` | `AsmString`, operand info | `printInstruction()` case for ADDXrs |
| `AsmMatcher` | `AsmString`, operand types | Parser for `"add x0, x1, x2"` |
| `DAGISel` | `Pattern` | Selector: match `(add GPR64, GPR64)` → `ADDXrs` |
| `CodeEmitter` | `Inst` bit fields | Binary encoding function |

### Step 5: The Generated C++ (Simplified)

```cpp
// From AArch64GenInstrInfo.inc:
namespace AArch64 {
  enum { ADDXrs = 1234, ... };
}

// From AArch64GenAsmWriter.inc:
void AArch64InstPrinter::printInstruction(const MCInst *MI, ...) {
  switch (MI->getOpcode()) {
  case AArch64::ADDXrs:
    OS << "add\t";
    printOperand(MI, 0, OS);  // $Rd
    OS << ", ";
    printOperand(MI, 1, OS);  // $Rn
    OS << ", ";
    printOperand(MI, 2, OS);  // $Rm
    printShiftOperand(MI, 3, OS);  // $shift
    return;
  }
}

// From AArch64GenMCCodeEmitter.inc:
uint64_t AArch64MCCodeEmitter::getBinaryCodeForInstr(const MCInst &MI, ...) {
  switch (MI.getOpcode()) {
  case AArch64::ADDXrs: {
    uint64_t Value = 0x8B000000;  // Fixed bits
    Value |= (getMachineOpValue(MI, MI.getOperand(0)) & 0x1F);       // Rd
    Value |= (getMachineOpValue(MI, MI.getOperand(1)) & 0x1F) << 5;  // Rn
    Value |= (getMachineOpValue(MI, MI.getOperand(2)) & 0x1F) << 16; // Rm
    // ... shift encoding
    return Value;
  }
  }
}
```

---

## Summary: Key Takeaways for AArch64 `.td` Files

### The Big Picture

```
AArch64.td
  ├── Registers (AArch64RegisterInfo.td)
  │     └── def X0, X1, ..., W0, W1, ..., D0, D1, ...
  │     └── def GPR64, GPR32, FPR64, ... (register classes)
  │
  ├── Features (AArch64Features.td)
  │     └── def FeatureNEON, FeatureSVE, FeatureCRC, ...
  │
  ├── Instructions (AArch64InstrInfo.td → many files)
  │     └── Encoding formats (AArch64InstrFormats.td)
  │     └── Arithmetic: ADD, SUB, MUL, ...
  │     └── Loads/Stores: LDR, STR, ...
  │     └── Branches: B, BL, CBZ, ...
  │     └── FP/SIMD: FADD, FMUL, ...
  │     └── SVE: FADD_ZPmZ, ...
  │
  ├── Patterns (Pat<> records throughout instruction files)
  │     └── Connect IR operations to machine instructions
  │
  ├── Scheduling (AArch64SchedA53.td, etc.)
  │     └── Latencies and resource usage per CPU
  │
  └── Processors (AArch64Processors.td)
        └── CPU definitions with feature lists
```

### Reading Checklist

When you encounter an unfamiliar construct in an AArch64 `.td` file:

1. **Is it a `class`?** → It's a template/blueprint. Look at its fields and template args.
2. **Is it a `def`?** → It's a concrete record. Look at what classes it inherits from.
3. **Is it a `defm`?** → It's instantiating a multiclass. Find the multiclass definition.
4. **Is it a `multiclass`?** → It defines a group of records. Look at the `def`s inside.
5. **Is it `let ... in { }`?** → It's overriding fields for a group of records.
6. **Is it `!cast<T>("name")`?** → It's looking up a record by a dynamically-built name.
7. **Is it `(outs ...)` / `(ins ...)`?** → These are instruction operand lists.
8. **Is it `[(set ..., (...))]`?** → This is an instruction selection pattern.
9. **Is it `let Inst{...} = ...`?** → This is instruction binary encoding.
10. **Is it `let Predicates = [...]`?** → This guards the instruction behind a feature flag.

---

*Part 3 of 3 — TableGen Beginner's Guide*

---

## Appendix: Useful Commands for AArch64 TableGen Exploration

```bash
# See all AArch64 registers:
llvm-tblgen llvm/lib/Target/AArch64/AArch64.td \
  -I llvm/include \
  -print-enums -class=Register

# See all AArch64 instructions:
llvm-tblgen llvm/lib/Target/AArch64/AArch64.td \
  -I llvm/include \
  -print-enums -class=Instruction | wc -l

# Dump a specific instruction's full record:
llvm-tblgen llvm/lib/Target/AArch64/AArch64.td \
  -I llvm/include \
  --print-records 2>&1 | grep -A 80 "^def ADDXrr {"

# Get all records as JSON and query with Python:
llvm-tblgen llvm/lib/Target/AArch64/AArch64.td \
  -I llvm/include \
  --dump-json > aarch64.json

python3 << 'EOF'
import json
data = json.load(open('aarch64.json'))
# Find all load instructions:
for name in data['!instanceof']['Instruction']:
    inst = data[name]
    if inst.get('mayLoad') == 1:
        print(name, ":", inst.get('AsmString', ''))
EOF

# Generate register info:
llvm-tblgen llvm/lib/Target/AArch64/AArch64.td \
  -I llvm/include \
  -gen-register-info -o /tmp/AArch64GenRegisterInfo.inc

# Generate instruction info:
llvm-tblgen llvm/lib/Target/AArch64/AArch64.td \
  -I llvm/include \
  -gen-instr-info -o /tmp/AArch64GenInstrInfo.inc
```
