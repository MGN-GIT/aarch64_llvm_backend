# LLVM IR Basics — What the Backend Is Lowering From
> Primary source files: `AArch64ISelLowering.cpp`, `AArch64ISelLowering.h`,
> `AArch64ISelDAGToDAG.cpp`, `AArch64TargetMachine.cpp`, `AArch64Subtarget.h`

---

## Table of Contents

- [Why This Matters](#why-this-matters)
- [The IR Object Model — Four Things to Know](#the-ir-object-model--four-things-to-know)
  - [Function](#function)
  - [BasicBlock](#basicblock)
  - [Instruction](#instruction)
  - [Value](#value)
- [IR Types and Their MVT Equivalents](#ir-types-and-their-mvt-equivalents)
  - [Scalar Integer Types](#scalar-integer-types)
  - [Floating-Point Types](#floating-point-types)
  - [Vector Types](#vector-types)
  - [Pointer Types](#pointer-types)
- [IR Intrinsics — llvm.aarch64.*](#ir-intrinsics--llvmaarch64)
  - [What an Intrinsic Is](#what-an-intrinsic-is)
  - [How the Backend Handles Them](#how-the-backend-handles-them)
  - [Concrete Examples from AArch64ISelDAGToDAG.cpp](#concrete-examples-from-aarch64iseldagtodag-cpp)
- [DataLayout — Endianness, Pointer Size, and Alignment](#datalayout--endianness-pointer-size-and-alignment)
  - [What DataLayout Is](#what-datalayout-is)
  - [Where It Is Set](#where-it-is-set)
  - [How the Backend Reads It](#how-the-backend-reads-it)
  - [Endianness in Practice](#endianness-in-practice)
- [Function Attributes — Per-Function Subtarget Selection](#function-attributes--per-function-subtarget-selection)
  - [What Function Attributes Are](#what-function-attributes-are)
  - [getSubtargetImpl — The Key Function](#getsubtargetimpl--the-key-function)
  - [The Streaming Mode Attributes](#the-streaming-mode-attributes)
  - [SVE Vector Length Attributes](#sve-vector-length-attributes)
  - [Why Each Function Gets Its Own Subtarget](#why-each-function-gets-its-own-subtarget)
- [How Everything Connects — End to End](#how-everything-connects--end-to-end)

---

## Why This Matters

The AArch64 backend's entire job is to take LLVM IR and produce AArch64 machine
code. Every pass in the backend — instruction selection, register allocation,
frame lowering, code emission — operates on a representation that was derived
from IR.

If you do not understand what IR looks like, you cannot understand:
- Why `AArch64ISelLowering.cpp` has hundreds of `LowerXxx` functions — each one
  handles a specific IR operation or type.
- Why `AArch64ISelDAGToDAG.cpp` checks `MVT::i32` vs `MVT::i64` everywhere —
  it is matching the IR type to the right instruction variant.
- Why `AArch64TargetMachine.cpp::getSubtargetImpl()` reads function attributes
  — the IR carries per-function target configuration.

---

## The IR Object Model — Four Things to Know

LLVM IR is structured as a hierarchy. The four objects you will encounter
constantly in the backend are:

```
Module
  └── Function          ← one per C/C++ function
        └── BasicBlock  ← one per control-flow block
              └── Instruction ← one per operation (add, load, call, ...)
                    └── Value ← everything that produces a result
```

### Function

A `Function` is the top-level unit of compilation. The backend processes one
`Function` at a time. It carries:

- The function's **name** and **type signature** (argument types, return type)
- A list of **BasicBlocks**
- **Function attributes** — key/value pairs that control code generation

In `AArch64ISelLowering.cpp`, the `Function` is accessed through
`MachineFunction::getFunction()`:

```cpp
// AArch64ISelLowering.cpp — LowerFormalArguments
// The IR Function is the source of truth for argument types and calling conv
SDValue AArch64TargetLowering::LowerFormalArguments(
    SDValue Chain, CallingConv::ID CallConv, bool isVarArg,
    const SmallVectorImpl<ISD::InputArg> &Ins,
    const SDLoc &DL, SelectionDAG &DAG,
    SmallVectorImpl<SDValue> &InVals) const {
  MachineFunction &MF = DAG.getMachineFunction();
  const Function &F = MF.getFunction();  // ← the IR Function
  // ...
}
```

In `AArch64TargetMachine.cpp`, the `Function` is used to read attributes and
select the right subtarget:

```cpp
// AArch64TargetMachine.cpp — getSubtargetImpl
const AArch64Subtarget *
AArch64TargetMachine::getSubtargetImpl(const Function &F) const {
  Attribute CPUAttr = F.getFnAttribute("target-cpu");
  Attribute FSAttr  = F.getFnAttribute("target-features");
  // ...
}
```

### BasicBlock

A `BasicBlock` is a straight-line sequence of instructions with a single entry
point and a single exit (a terminator instruction — `br`, `ret`, `switch`,
etc.). No branches can jump into the middle of a basic block.

In the backend, each IR `BasicBlock` becomes a `MachineBasicBlock`. The
`SelectionDAG` is built one basic block at a time.

```
IR BasicBlock:
  %a = add i32 %x, 1
  %b = mul i32 %a, 2
  br i1 %cond, label %true, label %false

                │
                ▼ SelectionDAG

MachineBasicBlock:
  ADDWri  W8, W0, #1
  LSLWri  W8, W8, #1    ; mul by 2 = shift left by 1
  CBNZ    W1, .LBB0_1
```

### Instruction

An `Instruction` is a single IR operation. Every instruction has:
- An **opcode** (e.g., `add`, `load`, `call`, `getelementptr`)
- **Operands** — references to other `Value`s
- A **type** — the type of the result it produces

The backend's instruction selector (`AArch64ISelDAGToDAG.cpp`) matches IR
instructions (represented as `SDNode`s in the SelectionDAG) to AArch64 machine
instructions.

```
IR instruction:          %result = add i32 %a, %b
                                   ^^^  ^^^
                                   opcode  type

SelectionDAG node:       (add:i32 %a, %b)
                          ^^^  ^^^
                          ISD::ADD  MVT::i32

AArch64 machine instr:   ADDWrr  W0, W1, W2
```

### Value

`Value` is the base class of everything in IR that produces a result. This
includes:
- `Instruction` — the result of an operation
- `Argument` — a function parameter
- `Constant` — a compile-time constant (`i32 42`, `float 1.0`)
- `GlobalValue` — a global variable or function

Every `Value` has a **type**. The type determines which machine register class
and which instruction variants the backend will use.

```cpp
// The type of a Value is queried constantly in the backend
// AArch64FastISel.cpp
EVT CEVT = TLI.getValueType(DL, C->getType(), true);
//                                ^^^^^^^^^^^
//                                IR type → EVT (Extended Value Type)
```

---

## IR Types and Their MVT Equivalents

IR types are the language of the frontend. The backend works with `MVT`
(Machine Value Type) and `EVT` (Extended Value Type). The translation happens
in `TargetLowering::getValueType()`.

### Scalar Integer Types

| IR Type | MVT | AArch64 Register | Notes |
|---------|-----|-----------------|-------|
| `i1` | `MVT::i1` | W register (32-bit) | Booleans; zero-extended to 32 bits |
| `i8` | `MVT::i8` | W register (32-bit) | Byte; zero/sign-extended to 32 bits |
| `i16` | `MVT::i16` | W register (32-bit) | Halfword; extended to 32 bits |
| `i32` | `MVT::i32` | W register (32-bit) | Native 32-bit integer |
| `i64` | `MVT::i64` | X register (64-bit) | Native 64-bit integer |
| `i128` | `MVT::i128` | Two X registers | Split across X register pair |

The backend checks these types constantly to pick the right instruction:

```cpp
// AArch64FastISel.cpp — choosing between W and X register variants
const TargetRegisterClass *RC =
    (VT == MVT::i64) ? &AArch64::GPR64RegClass
                     : &AArch64::GPR32RegClass;
unsigned ZeroReg = (VT == MVT::i64) ? AArch64::XZR : AArch64::WZR;
```

```cpp
// AArch64FastISel.cpp — size in bytes from MVT
case MVT::i8:  return 1;
case MVT::i16: return 2;
case MVT::i32: return 4;
case MVT::i64: return 8;
```

**Why `i8` and `i16` use 32-bit registers:**

AArch64 has no 8-bit or 16-bit general-purpose registers. All arithmetic
operates on 32-bit (W) or 64-bit (X) registers. When an `i8` value is loaded
from memory, it is zero-extended or sign-extended to fill the W register. The
backend inserts these extensions automatically.

```
IR:   %b = load i8, ptr %p
      %c = add i8 %b, 1

AArch64:
  LDRB  W0, [X1]       ; load byte, zero-extend to 32 bits
  ADD   W0, W0, #1     ; add in 32-bit register
  AND   W0, W0, #0xFF  ; truncate back to 8 bits (if needed)
```

### Floating-Point Types

| IR Type | MVT | AArch64 Register | Notes |
|---------|-----|-----------------|-------|
| `half` / `f16` | `MVT::f16` | H register (16-bit FP) | Requires FEAT_FP16 |
| `bfloat` / `bf16` | `MVT::bf16` | H register | Brain float; requires FEAT_BF16 |
| `float` / `f32` | `MVT::f32` | S register (32-bit FP) | Single precision |
| `double` / `f64` | `MVT::f64` | D register (64-bit FP) | Double precision |
| `fp128` / `f128` | `MVT::f128` | Q register (128-bit) | Quad precision |

```cpp
// AArch64FastISel.cpp — FP type check
if (VT != MVT::f32 && VT != MVT::f64)
  return Register();  // FastISel only handles f32 and f64
```

### Vector Types

| IR Type | MVT | AArch64 Register | Notes |
|---------|-----|-----------------|-------|
| `<8 x i8>` | `MVT::v8i8` | D register (64-bit NEON) | 8 bytes |
| `<16 x i8>` | `MVT::v16i8` | Q register (128-bit NEON) | 16 bytes |
| `<4 x i16>` | `MVT::v4i16` | D register | 4 halfwords |
| `<8 x i16>` | `MVT::v8i16` | Q register | 8 halfwords |
| `<2 x i32>` | `MVT::v2i32` | D register | 2 words |
| `<4 x i32>` | `MVT::v4i32` | Q register | 4 words |
| `<2 x i64>` | `MVT::v2i64` | Q register | 2 doublewords |
| `<4 x float>` | `MVT::v4f32` | Q register | 4 single-precision floats |
| `<2 x double>` | `MVT::v2f64` | Q register | 2 double-precision floats |
| `<vscale x 4 x i32>` | `MVT::nxv4i32` | Z register (SVE) | Scalable vector |

The backend distinguishes 64-bit ("D-register") and 128-bit ("Q-register")
NEON vectors constantly:

```cpp
// AArch64ISelLowering.cpp — choosing between D and Q register operations
bool Is128Bit = VT.getSizeInBits() == 128;
SDValue RegSeq = Is128Bit ? createQTuple(Regs) : createDTuple(Regs);
```

```cpp
// AArch64ISelLowering.cpp — register class from vector type
else if (RegVT == MVT::f64 || RegVT.is64BitVector())
  RC = &AArch64::FPR64RegClass;   // D register
else if (RegVT == MVT::f128 || RegVT.is128BitVector())
  RC = &AArch64::FPR128RegClass;  // Q register
```

### Pointer Types

In IR, pointers are typed (`ptr` in opaque pointer mode, or `i32*`/`i64*` in
typed pointer mode). In the AArch64 backend, pointers are always represented
as `MVT::i64` in the SelectionDAG — even in ILP32 mode, the DAG uses 64-bit
pointers internally:

```cpp
// AArch64ISelLowering.h — getPointerTy
MVT getPointerTy(const DataLayout &DL, uint32_t AS = 0) const override {
  if ((AS == ARM64AS::PTR32_SPTR) || (AS == ARM64AS::PTR32_UPTR)) {
    return MVT::i32;  // __ptr32 extension: 32-bit pointer
  } else {
    // Returning i64 unconditionally here (i.e. even for ILP32) means that
    // the *DAG* representation of pointers will always be 64-bits.
    return MVT::i64;
  }
}
```

---

## IR Intrinsics — llvm.aarch64.*

### What an Intrinsic Is

An IR intrinsic is a special function call in LLVM IR that the backend
recognises by name and lowers to specific machine instructions. Intrinsics
look like function calls in IR but are never actually called — the backend
replaces them with machine code during instruction selection.

```llvm
; IR — an AArch64-specific intrinsic call
%result = call i32 @llvm.aarch64.crc32b(i32 %crc, i32 %data)
```

This is not a real function call. The backend sees `Intrinsic::aarch64_crc32b`
and emits a single `CRC32Brr` instruction.

AArch64 intrinsics are declared in `llvm/IR/IntrinsicsAArch64.h` (generated
from `IntrinsicsAArch64.td`). They cover:
- **CRC** — `llvm.aarch64.crc32b`, `llvm.aarch64.crc32h`, etc.
- **NEON** — `llvm.aarch64.neon.ld2`, `llvm.aarch64.neon.st4`, etc.
- **SVE** — `llvm.aarch64.sve.ld2.sret`, `llvm.aarch64.sve.st1`, etc.
- **SME** — `llvm.aarch64.sme.read.hor.vg2`, etc.
- **Pointer authentication** — `llvm.aarch64.irg.sp`, `llvm.aarch64.tagp`, etc.
- **Atomics** — `llvm.aarch64.ldxp`, `llvm.aarch64.stlxp`, etc.

### How the Backend Handles Them

Intrinsics arrive in the SelectionDAG as `ISD::INTRINSIC_WO_CHAIN` (no
side-effects), `ISD::INTRINSIC_W_CHAIN` (has side-effects / memory access),
or `ISD::INTRINSIC_VOID` (no return value).

The `AArch64DAGToDAGISel::Select()` function in `AArch64ISelDAGToDAG.cpp`
dispatches on the intrinsic ID:

```cpp
// AArch64ISelDAGToDAG.cpp — intrinsic dispatch
case ISD::INTRINSIC_W_CHAIN: {
  unsigned IntNo = Node->getConstantOperandVal(1);
  switch (IntNo) {
  default:
    break;
  case Intrinsic::aarch64_ldaxp:
  case Intrinsic::aarch64_ldxp: {
    // Load-exclusive pair: maps to LDAXPX or LDXPX
    unsigned Op = (IntNo == Intrinsic::aarch64_ldaxp)
                      ? AArch64::LDAXPX
                      : AArch64::LDXPX;
    SDValue MemAddr = Node->getOperand(2);
    // ... emit machine node
  }
  case Intrinsic::aarch64_neon_ld2: {
    // NEON interleaved load: maps to LD2 instruction
    if (VT == MVT::v8i8)
      SelectLoad(Node, 2, AArch64::LD2Twov8b, AArch64::dsub0);
    // ...
  }
  case Intrinsic::aarch64_sve_ld2_sret: {
    // SVE predicated load: maps to LD2B_IMM or LD2B
    SelectPredicatedLoad(Node, 2, 0, AArch64::LD2B_IMM, AArch64::LD2B, true);
    return;
  }
  }
}
```

### Concrete Examples from AArch64ISelDAGToDAG.cpp

**CRC intrinsics** (handled in `AArch64FastISel.cpp`):

```cpp
// AArch64FastISel.cpp
case Intrinsic::aarch64_crc32b:  Opc = AArch64::CRC32Brr;  break;
case Intrinsic::aarch64_crc32h:  Opc = AArch64::CRC32Hrr;  break;
case Intrinsic::aarch64_crc32w:  Opc = AArch64::CRC32Wrr;  break;
case Intrinsic::aarch64_crc32x:  Opc = AArch64::CRC32Xrr;  break;
case Intrinsic::aarch64_crc32cb: Opc = AArch64::CRC32CBrr; break;
// ...
```

**Pointer authentication** (handled in `AArch64ISelDAGToDAG.cpp`):

```cpp
// AArch64ISelDAGToDAG.cpp — llvm.aarch64.irg.sp
SDValue IRG_SP = N->getOperand(2);
if (IRG_SP->getOpcode() != ISD::INTRINSIC_W_CHAIN ||
    IRG_SP->getConstantOperandVal(1) != Intrinsic::aarch64_irg_sp) {
  return false;
}
```

**SME tile reads** (handled in `AArch64ISelDAGToDAG.cpp`):

```cpp
// AArch64ISelDAGToDAG.cpp
case Intrinsic::aarch64_sme_read_hor_vg2: {
  if (VT == MVT::nxv16i8)
    SelectMultiVectorMove<14, 2>(Node, 2, AArch64::ZAB0, ...);
}
```

The pattern is always the same:
1. The IR intrinsic call arrives as an `ISD::INTRINSIC_*` node.
2. The backend reads the intrinsic ID from operand 0 (or 1 for `W_CHAIN`).
3. A `switch` dispatches to the right selection function.
4. The selection function emits a `MachineNode` with the correct AArch64 opcode.

---

## DataLayout — Endianness, Pointer Size, and Alignment

### What DataLayout Is

`DataLayout` is an IR-level object that describes the memory model of the
target. It answers questions like:
- Is the target little-endian or big-endian?
- How wide are pointers?
- What is the natural alignment of `i32`? Of `i64`?
- What is the stack alignment?

It is encoded as a string in the IR module, for example:

```
; Little-endian AArch64 (Linux ELF)
target datalayout = "e-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128"

; Big-endian AArch64
target datalayout = "E-m:e-i8:8:32-i16:16:32-i64:64-i128:128-n32:64-S128"
```

Breaking down the string:

```
e          → little-endian  (E = big-endian)
m:e        → ELF mangling
i8:8:32    → i8  has ABI alignment 8 bits, preferred alignment 32 bits
i16:16:32  → i16 has ABI alignment 16 bits, preferred alignment 32 bits
i64:64     → i64 has ABI alignment 64 bits
i128:128   → i128 has ABI alignment 128 bits
n32:64     → native integer widths are 32 and 64 bits
S128       → stack is 128-bit (16-byte) aligned
```

### Where It Is Set

The `DataLayout` string is computed by `Triple::computeDataLayout()` (in LLVM's
`TargetParser` library) and passed to the `TargetMachine` constructor:

```cpp
// AArch64TargetMachine.cpp — constructor
AArch64TargetMachine::AArch64TargetMachine(
    const Target &T, const Triple &TT, ...)
    : CodeGenTargetMachineImpl(
          T,
          TT.computeDataLayout(),  // ← DataLayout string from Triple
          TT, CPU, FS, ...)
```

The `Triple` encodes the OS, architecture, and ABI. The `computeDataLayout()`
function uses this to produce the correct string — for example, macOS uses
`m:o` (Mach-O mangling) while Linux uses `m:e` (ELF mangling).

### How the Backend Reads It

The `DataLayout` is accessed through `SelectionDAG::getDataLayout()` and
`MachineFunction::getDataLayout()`. It is used in many places:

```cpp
// AArch64ISelDAGToDAG.cpp — pointer type from DataLayout
const DataLayout &DL = CurDAG->getDataLayout();
const TargetLowering *TLI = getTargetLowering();
Base = CurDAG->getTargetFrameIndex(
    FI, TLI->getPointerTy(DL));  // ← pointer size from DataLayout
```

```cpp
// AArch64CallingConvention.cpp — stack alignment from DataLayout
const MaybeAlign StackAlign =
    State.getMachineFunction().getDataLayout().getStackAlignment();
```

```cpp
// AArch64CallingConvention.td — endianness check
class CCIfBigEndian<CCAction A> :
  CCIf<"State.getMachineFunction().getDataLayout().isBigEndian()", A>;

class CCIfILP32<CCAction A> :
  CCIf<"State.getMachineFunction().getDataLayout().getPointerSize() == 4", A>;
```

### Endianness in Practice

AArch64 supports both little-endian (the common case) and big-endian. The
backend checks endianness in many places because vector lane ordering and
load/store behaviour differ:

```cpp
// AArch64Subtarget.h
bool isLittleEndian() const { return IsLittle; }
```

```cpp
// AArch64ISelDAGToDAG.cpp — indexed load: different opcode for big-endian vectors
} else if (VT.is64BitVector() && Subtarget->isLittleEndian()) {
  Opcode = IsPre ? AArch64::LDRDpre : AArch64::LDRDpost;
} else if (VT.is128BitVector() && Subtarget->isLittleEndian()) {
  Opcode = IsPre ? AArch64::LDRQpre : AArch64::LDRQpost;
} else if (VT.is64BitVector()) {
  // Big-endian: use LD1 with XZR offset instead
  Opcode = AArch64::LD1Onev8b_POST;
}
```

```cpp
// AArch64ISelDAGToDAG.cpp — LD1 offset register differs by endianness
SDValue Offset = (VT.isVector() && !Subtarget->isLittleEndian())
                     ? CurDAG->getRegister(AArch64::XZR, MVT::i64)
                     : CurDAG->getTargetConstant(OffsetVal, dl, MVT::i64);
```

The two concrete `TargetMachine` subclasses encode endianness at construction:

```cpp
// AArch64TargetMachine.cpp
AArch64leTargetMachine::AArch64leTargetMachine(...)
    : AArch64TargetMachine(T, TT, CPU, FS, Options, RM, CM, OL, JIT,
                           /*LittleEndian=*/true) {}

AArch64beTargetMachine::AArch64beTargetMachine(...)
    : AArch64TargetMachine(T, TT, CPU, FS, Options, RM, CM, OL, JIT,
                           /*LittleEndian=*/false) {}
```

---

## Function Attributes — Per-Function Subtarget Selection

### What Function Attributes Are

Function attributes are key/value pairs attached to an IR `Function`. They
carry information that the frontend (Clang) computed about the function and
wants to communicate to the backend.

```llvm
; IR — function with target attributes
define void @foo() #0 {
  ret void
}

attributes #0 = {
  "target-cpu"="apple-m1"
  "target-features"="+neon,+sve,+sme"
  "aarch64_pstate_sm_enabled"
}
```

The backend reads these attributes in `getSubtargetImpl()` to build a
per-function `AArch64Subtarget`.

### getSubtargetImpl — The Key Function

`AArch64TargetMachine::getSubtargetImpl()` is called once per function. It
reads the function's attributes and constructs (or retrieves from a cache) the
correct `AArch64Subtarget` for that function:

```cpp
// AArch64TargetMachine.cpp
const AArch64Subtarget *
AArch64TargetMachine::getSubtargetImpl(const Function &F) const {
  // Step 1: Read the target-cpu attribute
  Attribute CPUAttr  = F.getFnAttribute("target-cpu");
  Attribute TuneAttr = F.getFnAttribute("tune-cpu");
  Attribute FSAttr   = F.getFnAttribute("target-features");

  // Step 2: Fall back to the global target CPU/features if not set per-function
  StringRef CPU     = CPUAttr.isValid()  ? CPUAttr.getValueAsString()  : TargetCPU;
  StringRef TuneCPU = TuneAttr.isValid() ? TuneAttr.getValueAsString() : CPU;
  StringRef FS      = FSAttr.isValid()   ? FSAttr.getValueAsString()   : TargetFS;

  // Step 3: Read streaming mode attributes
  bool IsStreaming =
      ForceStreaming ||
      F.hasFnAttribute("aarch64_pstate_sm_enabled") ||
      F.hasFnAttribute("aarch64_pstate_sm_body");
  bool IsStreamingCompatible =
      ForceStreamingCompatible ||
      F.hasFnAttribute("aarch64_pstate_sm_compatible");

  // Step 4: Read SVE vector length constraints
  unsigned MinSVEVectorSize = 0;
  unsigned MaxSVEVectorSize = 0;
  if (F.hasFnAttribute(Attribute::VScaleRange)) {
    ConstantRange CR = getVScaleRange(&F, 64);
    MinSVEVectorSize = CR.getUnsignedMin().getZExtValue() * 128;
    MaxSVEVectorSize = CR.getUnsignedMax().getZExtValue() * 128;
  }

  // Step 5: Build a cache key from all the above
  SmallString<512> Key;
  Key += "SVEMin"; Key += utostr(MinSVEVectorSize);
  Key += "SVEMax"; Key += utostr(MaxSVEVectorSize);
  Key += "IsStreaming="; Key += utostr(IsStreaming);
  Key += CPU; Key += TuneCPU; Key += FS;
  // ...

  // Step 6: Look up or create the subtarget
  auto &I = SubtargetMap[Key];
  if (!I) {
    I = std::make_unique<AArch64Subtarget>(
        TargetTriple, CPU, TuneCPU, FS, *this, isLittle,
        MinSVEVectorSize, MaxSVEVectorSize,
        IsStreaming, IsStreamingCompatible, ...);
  }
  return I.get();
}
```

**Key insight:** There is no single global subtarget. Every unique combination
of `target-cpu`, `target-features`, and streaming attributes gets its own
`AArch64Subtarget` instance. This is how a single compilation unit can contain
functions targeting different CPU variants.

### The Streaming Mode Attributes

SME (Scalable Matrix Extension) introduces a "streaming mode" for the CPU.
Three function attributes control this:

| Attribute | Meaning | Effect on Subtarget |
|-----------|---------|---------------------|
| `aarch64_pstate_sm_enabled` | Function runs entirely in streaming mode | `IsStreaming = true` |
| `aarch64_pstate_sm_body` | Function body runs in streaming mode | `IsStreaming = true` |
| `aarch64_pstate_sm_compatible` | Function can run in either mode | `IsStreamingCompatible = true` |

These are read in `AArch64SMEAttributes.cpp`:

```cpp
// AArch64SMEAttributes.cpp
if (Attrs.hasFnAttr("aarch64_pstate_sm_enabled"))
  Bitmask |= SM_Enabled;
if (Attrs.hasFnAttr("aarch64_pstate_sm_compatible"))
  Bitmask |= SM_Compatible;
if (Attrs.hasFnAttr("aarch64_pstate_sm_body"))
  Bitmask |= SM_Body;
```

The streaming mode affects which instructions are legal. In streaming mode,
standard NEON and SVE instructions are disabled; only the streaming-compatible
subset of SVE (via SME) is available:

```cpp
// AArch64Subtarget.h
bool isNeonAvailable() const {
  return hasNEON() &&
         (hasSMEFA64() || (!isStreaming() && !isStreamingCompatible()));
  //                       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  //                       NEON is disabled in streaming mode
}

bool isSVEAvailable() const {
  return hasSVE() &&
         (hasSMEFA64() || (!isStreaming() && !isStreamingCompatible()));
  //                       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  //                       Full SVE is disabled in streaming mode
}
```

The `AArch64ISelLowering.cpp` constructor uses these flags to set which
operations are legal for the current function:

```cpp
// AArch64ISelLowering.cpp — SVE operations only legal when SVE is available
if (Subtarget->hasSVE2() ||
    (Subtarget->hasSME() && Subtarget->isStreaming()))
  setOperationAction(ISD::OR, VT, Custom);
```

### SVE Vector Length Attributes

The `vscale_range` attribute tells the backend the minimum and maximum SVE
vector length for a function. This allows the backend to generate more
efficient code when the vector length is known:

```llvm
; IR — function with known SVE vector length (512-bit = vscale * 128 * 4)
define void @foo() "vscale_range"="4,4" { ... }
```

```cpp
// AArch64TargetMachine.cpp — reading vscale_range
if (F.hasFnAttribute(Attribute::VScaleRange)) {
  ConstantRange CR = getVScaleRange(&F, 64);
  MinSVEVectorSize = CR.getUnsignedMin().getZExtValue() * 128;
  MaxSVEVectorSize = CR.getUnsignedMax().getZExtValue() * 128;
}
```

When `MinSVEVectorSize == MaxSVEVectorSize`, the backend knows the exact vector
length and can use fixed-length SVE code paths.

### Why Each Function Gets Its Own Subtarget

Consider a translation unit with two functions:

```c
// Compiled with -mcpu=cortex-a55
void normal_func(int *a, int *b) { ... }

// Compiled with __attribute__((target("cpu=apple-m1,+sve")))
void optimized_func(float *a, float *b) { ... }
```

Clang emits different `target-cpu` and `target-features` attributes for each
function. The backend calls `getSubtargetImpl()` for each function and gets
back a different `AArch64Subtarget` — one for Cortex-A55 (no SVE), one for
Apple M1 (with SVE). The instruction selector then uses the correct subtarget
to decide which instructions are legal.

This is also why the `AArch64TargetMachine` has no `getSubtargetImpl()` with
no arguments — it is explicitly deleted:

```cpp
// AArch64TargetMachine.h
// DO NOT IMPLEMENT: There is no such thing as a valid default subtarget,
// subtargets are per-function entities based on the target-specific
// attributes of each function.
const AArch64Subtarget *getSubtargetImpl() const = delete;
```

---

## How Everything Connects — End to End

Here is how IR concepts flow through the backend to produce machine code:

```
IR Function (with attributes):
  define void @foo()
      "target-cpu"="cortex-a55"
      "target-features"="+neon"
      "aarch64_pstate_sm_compatible" {
    %a = add i32 %x, %y          ; Instruction, type i32
    %b = load <4 x float>, ptr %p ; Instruction, type <4 x float>
    call void @llvm.aarch64.neon.st4(...)  ; Intrinsic
    ret void
  }

         │
         ▼ AArch64TargetMachine::getSubtargetImpl()

Reads attributes → builds AArch64Subtarget:
  CPU = "cortex-a55"
  Features = "+neon"
  IsStreamingCompatible = true   ← from "aarch64_pstate_sm_compatible"
  IsStreaming = false
  isNeonAvailable() = false      ← NEON disabled in streaming-compatible mode!

         │
         ▼ SelectionDAG construction

IR types → MVT:
  i32           → MVT::i32   → W register
  <4 x float>   → MVT::v4f32 → Q register (128-bit NEON)
  ptr           → MVT::i64   → X register (pointer)

         │
         ▼ AArch64ISelDAGToDAG::Select()

Intrinsic dispatch:
  ISD::INTRINSIC_VOID
    → Intrinsic::aarch64_neon_st4
    → SelectStore(Node, 4, AArch64::ST4Fourv4s)
    → MachineNode: ST4Fourv4s

Arithmetic:
  (add:i32 %x, %y)
    → MVT::i32 → W register variant
    → ADDWrr W0, W1, W2

Load:
  (load:v4f32 %p)
    → MVT::v4f32 → Q register
    → LDRQui Q0, [X0]

         │
         ▼ MC Layer → binary output

  0x0E0C0000   ST4 {V0.4S, V1.4S, V2.4S, V3.4S}, [X0]
  0x0B010000   ADD W0, W0, W1
  0x3DC00000   LDR Q0, [X0]
```

The IR type system, DataLayout, intrinsics, and function attributes are not
separate concerns — they are all inputs to the same pipeline. The backend reads
them all at the start of each function and uses them to make every instruction
selection decision.
 
