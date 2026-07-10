# README: AND + CMEQ to CMTST Optimization

A complete guide for implementing your first LLVM optimization!

---

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Concepts Explained](#concepts-explained)
3. [Understanding DAG](#understanding-dag)
4. [Where To Add Code](#where-to-add-code)
5. [Important APIs](#important-apis)
6. [Implementation Steps](#implementation-steps)
7. [Code Template](#code-template)
8. [Testing](#testing)
9. [Difficulty Assessment](#difficulty-assessment)

---

## Problem Statement

### The Issue

When you compile ARM code with pattern `(value & mask) == mask`, LLVM generates **inefficient code** with **2 instructions** when it could use **1 instruction**.

### Current (Bad) ❌

```asm
and     v2.16b, v1.16b, v2.16b    # Instruction 1: AND
cmeq    v2.16b, v2.16b, #0        # Instruction 2: CMEQ
```

**2 instructions needed**

### Desired (Good) ✅

```asm
cmtst   v2.16b, v1.16b, v2.16b    # Single instruction does everything!
```

**1 instruction needed** - More efficient!

### Your Task

```
IF you find: AND + CMEQ together
THEN: Replace them with CMTST
```

---

## Concepts Explained

### What is AND?

Bitwise AND operation - keeps only bits that are 1 in BOTH inputs.

```
Value A:    1010 1100
Value B:    0110 1001
AND Result: 0010 1000  ← Common 1 bits
```

### What is CMEQ?

Compare Equal - checks if two values are equal.

```
(result == 0) ?
    If YES → all 1s (1111 1111)
    If NO  → all 0s (0000 0000)
```

### What is CMTST?

Compare Test - checks if two values share ANY common 1 bits.

```
(value1 & value2) != 0 ?
    If YES → all 1s (1111 1111)
    If NO  → all 0s (0000 0000)
```

**CMTST = AND + CMEQ combined into ONE instruction!**

### Why They're Equivalent

```
AND + CMEQ does:
    Step 1: temp = A AND B
    Step 2: IF (temp == 0) → result = FALSE ELSE result = TRUE

CMTST does:
    IF (A AND B != 0) → result = TRUE ELSE result = FALSE

SAME LOGIC! Just one instruction instead of two!
```

### Visual Comparison

```
OLD (2 instructions):              NEW (1 instruction):
┌─────────────────┐               ┌──────────────────┐
│ Input A         │               │ Input A          │
│ Input B  ──┬──→ [AND] ──→       │ Input B  ──┬──→ [CMTST] ──→ Result
└─────────────┤                   └──────────────┘
              │
              └──→ [CMEQ] ──→ Result
                  (Compare with 0)
```

---

## Understanding DAG

### What is a Node?

A **node** = one operation in your code.

```
Example: result = (a & b) == 0

Creates 2 nodes:
    Node 1: AND
        - Opcode: ISD::AND
        - Operand[0]: a
        - Operand[1]: b

    Node 2: SETCC (comparison)
        - Opcode: ISD::SETCC
        - Operand[0]: AND result
        - Operand[1]: 0
        - Operand[2]: EQ (condition code)
```

### SDValue vs SDNode

```cpp
SDNode *node;   // The actual node object
                // Has opcode, operands, type info

SDValue value;  // A reference to a node + output index
                // This is what you usually work with
```

### Node Structure

```
Every node has:
┌──────────────────────────┐
│ 1. Opcode                │ What operation? (AND, MUL, SETCC)
│ 2. Operands (Inputs)     │ What goes into it? (up to N inputs)
│ 3. Value Type            │ What type is result? (i32, i64, vector)
│ 4. Metadata              │ Debug info, flags, etc.
└──────────────────────────┘
```

### DAG Example

```
If you write in C:
    result = (a & b) == 0

DAG looks like:
    a ──┐
        ├─→ [AND] ─→ andResult
    b ──┘

    andResult ──┐
                ├─→ [SETCC] ─→ finalResult
    0 ──────────┘
    (with EQ condition)
```

---

## Where To Add Code

### File Location

```
llvm/lib/Target/AArch64/AArch64ISelLowering.cpp
```

This is the **ONLY file** you need to edit!

### Function Name

Search for:
```
PerformDAGCombine
```

You'll find:
```cpp
SDValue AArch64TargetLowering::PerformDAGCombine(SDNode *N, DAGCombinerInfo &DCI) const {
    switch (N->getOpcode()) {
        // Existing cases here
    }
    return SDValue();
}
```

### Where To Insert

Inside the `switch` statement, add a **new case**:

```cpp
switch (N->getOpcode()) {
    // ... existing cases ...

    case ISD::SETCC: {
        // ⭐ YOUR CODE GOES HERE ⭐
    }
    break;

    // ... more cases ...
}
```

### Visual Structure

```
File: AArch64ISelLowering.cpp
│
└─ Function: PerformDAGCombine
   │
   └─ switch (N->getOpcode())
      │
      ├─ case ISD::ADD
      ├─ case ISD::SUB
      ├─ case ISD::XOR
      ├─ case ISD::MUL
      │
      ├─ ⭐ case ISD::SETCC (YOUR CODE HERE)
      │
      └─ case ISD::INTRINSIC_VOID
```

---

## Important APIs

### Node Operations

#### Get Opcode (What operation is this?)
```cpp
N->getOpcode()

Returns: The operation type
Example:
    if (N->getOpcode() == ISD::SETCC) { ... }
    if (N->getOpcode() == ISD::AND) { ... }
```

#### Get Operand (Get inputs to operation)
```cpp
N->getOperand(index)

Returns: SDValue (the input)
Example:
    SDValue firstInput = N->getOperand(0);
    SDValue secondInput = N->getOperand(1);
    SDValue thirdInput = N->getOperand(2);
```

#### Get Value Type (What type is the result?)
```cpp
N->getValueType(0)

Returns: EVT (type like i8, i32, vector types)
Example:
    EVT resultType = N->getValueType(0);
    if (resultType.isVector()) { ... }
```

### Type Checking

#### isa<> (Check if something is a type)
```cpp
isa<ConstantSDNode>(operand)

Returns: true or false
Example:
    if (isa<ConstantSDNode>(N->getOperand(1))) {
        // It's a constant!
    }
```

#### cast<> (Convert to specific type)
```cpp
cast<ConstantSDNode>(operand)

Returns: The converted object
Example:
    uint64_t value = cast<ConstantSDNode>(N->getOperand(1))->getZExtValue();
```

### Getting Values

#### Get Constant Value
```cpp
cast<ConstantSDNode>(operand)->getZExtValue()

Returns: uint64_t value
Example:
    if (cast<ConstantSDNode>(N->getOperand(1))->getZExtValue() == 0) {
        // Right side is zero!
    }
```

#### Get Condition Code (For comparisons)
```cpp
cast<CondCodeSDNode>(N->getOperand(2))->get()

Returns: ISD::CondCode (EQ, NE, LT, etc.)
Example:
    ISD::CondCode CC = cast<CondCodeSDNode>(N->getOperand(2))->get();
    if (CC == ISD::SETEQ) {
        // It's an equality comparison!
    }
```

### Building New Nodes

#### Get Debug Location
```cpp
SDLoc DL(N)

Gets debug location from existing node
Use when creating new nodes
```

#### Get DAG Reference
```cpp
SelectionDAG &DAG = DCI.DAG;

DCI = DAGCombinerInfo parameter
Use DAG to create new nodes
```

#### Create AND Node
```cpp
DAG.getNode(ISD::AND, DL, VT, operand0, operand1)

Returns: SDValue (new AND node)
```

#### Create CMTST Node
```cpp
DAG.getNode(AArch64ISD::CMTST, DL, VT, operand0, operand1)

Returns: SDValue (new CMTST node)
```

### Key Opcodes

```cpp
ISD::AND        // Bitwise AND
ISD::SETCC      // Comparison (==, !=, <, >, etc.)
ISD::SETEQ      // Equality comparison (==)
ISD::SETNE      // Not equal comparison (!=)

AArch64ISD::CMTST  // AArch64-specific CMTST instruction
```

### Return Values

```cpp
return SDValue();        // No optimization (leave as is)
return newNode;          // Return optimized version
```

---

## Implementation Steps

### Step 1: Check if SETCC

```cpp
case ISD::SETCC: {
    // We're inside a SETCC (comparison) node
    // Proceed to step 2
}
```

### Step 2: Get Operands

```cpp
SDValue lhs = N->getOperand(0);  // Left side of comparison
SDValue rhs = N->getOperand(1);  // Right side of comparison
ISD::CondCode cc = cast<CondCodeSDNode>(N->getOperand(2))->get();  // EQ, NE, etc.
```

### Step 3: Check Condition Code (Must be SETEQ)

```cpp
if (cc != ISD::SETEQ) {
    return SDValue();  // Not an equality comparison, skip
}
```

### Step 4: Check if LHS is AND

```cpp
if (lhs.getOpcode() != ISD::AND) {
    return SDValue();  // Left side is not AND, skip
}
```

### Step 5: Check if RHS is Zero Constant

```cpp
if (!isa<ConstantSDNode>(rhs)) {
    return SDValue();  // Right side is not a constant, skip
}

uint64_t rhsValue = cast<ConstantSDNode>(rhs)->getZExtValue();
if (rhsValue != 0) {
    return SDValue();  // Not comparing with zero, skip
}
```

### Step 6: Extract AND Operands

```cpp
SDValue andOp0 = lhs.getOperand(0);  // First operand of AND
SDValue andOp1 = lhs.getOperand(1);  // Second operand of AND
```

### Step 7: Create CMTST Node

```cpp
SDLoc DL(N);
EVT VT = N->getValueType(0);
SelectionDAG &DAG = DCI.DAG;

SDValue cmtstNode = DAG.getNode(AArch64ISD::CMTST, DL, VT, andOp0, andOp1);
```

### Step 8: Return the New Node

```cpp
return cmtstNode;
```

---

## Code Template

### Complete Template

```cpp
case ISD::SETCC: {
    // Step 1: Get operands
    SDValue lhs = N->getOperand(0);
    SDValue rhs = N->getOperand(1);
    ISD::CondCode cc = cast<CondCodeSDNode>(N->getOperand(2))->get();

    // Step 2: Check condition code
    if (cc != ISD::SETEQ)
        return SDValue();

    // Step 3: Check if LHS is AND
    if (lhs.getOpcode() != ISD::AND)
        return SDValue();

    // Step 4: Check if RHS is zero
    if (!isa<ConstantSDNode>(rhs))
        return SDValue();

    uint64_t rhsValue = cast<ConstantSDNode>(rhs)->getZExtValue();
    if (rhsValue != 0)
        return SDValue();

    // Pattern found! (AND x, y) == 0
    // Replace with CMTST(x, y)

    // Step 5: Extract AND operands
    SDValue andOp0 = lhs.getOperand(0);
    SDValue andOp1 = lhs.getOperand(1);

    // Step 6: Create CMTST node
    SDLoc DL(N);
    EVT VT = N->getValueType(0);
    SelectionDAG &DAG = DCI.DAG;

    return DAG.getNode(AArch64ISD::CMTST, DL, VT, andOp0, andOp1);
}
break;
```

### Minimal Template (Just Pattern Match)

```cpp
case ISD::SETCC: {
    SDValue lhs = N->getOperand(0);
    SDValue rhs = N->getOperand(1);
    ISD::CondCode cc = cast<CondCodeSDNode>(N->getOperand(2))->get();

    // Pattern: (AND x, y) == 0 with SETEQ condition
    if (cc == ISD::SETEQ && 
        lhs.getOpcode() == ISD::AND &&
        isa<ConstantSDNode>(rhs) &&
        cast<ConstantSDNode>(rhs)->getZExtValue() == 0) {

        // Replace with CMTST
        SDLoc DL(N);
        return DCI.DAG.getNode(AArch64ISD::CMTST, DL, N->getValueType(0),
                               lhs.getOperand(0), lhs.getOperand(1));
    }

    return SDValue();
}
break;
```

---

## Testing

### Step 1: Create Test File

Create: `llvm/test/CodeGen/AArch64/cmtst-combine.ll`

```llvm
; RUN: llc -mtriple=aarch64-unknown-linux %s -o - | FileCheck %s

define <16 x i8> @test_cmtst_v16i8(<16 x i8> %v) {
  %and = and <16 x i8> %v, <i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2>
  %cmp = icmp eq <16 x i8> %and, <i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2, i8 2>
  %ext = zext <16 x i1> %cmp to <16 x i8>
  ret <16 x i8> %ext
}

; CHECK-LABEL: test_cmtst_v16i8:
; CHECK: cmtst v0.16b, v0.16b, v1.16b
```

### Step 2: Run Test

```bash
cd llvm-project
mkdir build && cd build
cmake -G Ninja ..
ninja
cd ..

# Run your test
./build/bin/llvm-lit test/CodeGen/AArch64/cmtst-combine.ll
```

### Step 3: Check Output

If test passes:
```
✓ PASSED
```

If test fails:
```
✗ FAILED
```

### Step 4: Verify Assembly

Compile the test file and check assembly:

```bash
./build/bin/clang -target aarch64-linux-gnu -S test.ll -O2

# Check if it contains:
# cmtst
```

---

## Difficulty Assessment

### Difficulty Scale: **EASY-MODERATE** ⭐⭐

| Aspect | Difficulty | Time |
|--------|-----------|------|
| Finding location | ⭐ Easy | 10 min |
| Understanding APIs | ⭐⭐ Moderate | 1 hour |
| Writing code | ⭐⭐ Moderate | 1-2 hours |
| Testing | ⭐⭐ Moderate | 1-2 hours |
| **Total** | **⭐⭐** | **3-7 hours** |

### Easy Parts

- ✅ One file to edit
- ✅ One function to modify
- ✅ ~20 lines of code
- ✅ Clear requirements
- ✅ CMTST already exists

### Challenging Parts

- ⚠️ Understanding DAG concepts
- ⚠️ Learning new APIs
- ⚠️ Type system
- ⚠️ Debugging compilation
- ⚠️ LLVM compilation is slow

### Why It's Good First Task

- ✅ Well-scoped problem
- ✅ Clear success criteria
- ✅ Great for learning DAG
- ✅ Real contribution
- ✅ Existing infrastructure

---

## Quick Reference Checklist

### Before Coding
- [ ] Understand the problem (AND + CMEQ → CMTST)
- [ ] Know what AND, CMEQ, CMTST do
- [ ] Locate AArch64ISelLowering.cpp
- [ ] Find PerformDAGCombine function
- [ ] Review code template

### While Coding
- [ ] Add `case ISD::SETCC:` block
- [ ] Check condition code (must be SETEQ)
- [ ] Check LHS is AND
- [ ] Check RHS is zero constant
- [ ] Extract AND operands
- [ ] Create CMTST node
- [ ] Return the node

### After Coding
- [ ] Compile LLVM
- [ ] Write test case
- [ ] Run test
- [ ] Check assembly output
- [ ] Verify CMTST appears

---

## Common Mistakes To Avoid

```cpp
❌ Forgetting to break; after case
❌ Returning wrong node type
❌ Not checking all conditions
❌ Wrong operand indices
❌ Not handling vector types
❌ Type mismatch in DAG.getNode()
❌ Using wrong ISD::CondCode
```

---

## Pseudocode One More Time

```cpp
if (N is SETCC) {
    if (N compares with == 0) {
        if (N's input is AND) {
            // PATTERN FOUND!
            Get AND's two operands
            Create CMTST with those operands
            Return CMTST
        }
    }
}
Return nothing (no optimization)
```

---

## Resources

### Official Documentation
- LLVM SelectionDAG: https://llvm.org/docs/CodeGenerator/#selectiondag
- ISDOpcodes: `llvm/include/llvm/CodeGen/ISDOpcodes.h`
- SelectionDAGNodes: `llvm/include/llvm/CodeGen/SelectionDAGNodes.h`

### Files to Study
- `llvm/lib/Target/AArch64/AArch64ISelLowering.cpp` (Your target)
- `llvm/lib/Target/AArch64/AArch64InstrInfo.td` (CMTST instruction)
- `llvm/test/CodeGen/AArch64/` (Test examples)

### Key Files in LLVM
```
llvm/include/llvm/CodeGen/SelectionDAGNodes.h  (API reference)
llvm/include/llvm/CodeGen/ISDOpcodes.h         (All opcodes)
llvm/lib/CodeGen/SelectionDAG/DAGCombiner.cpp  (How combining works)
```

---

## Getting Help

If stuck, check:

1. **Existing cases** in PerformDAGCombine
2. **Similar optimizations** in the file
3. **Error messages** (they're usually helpful)
4. **Compile with -DLLVM_DEBUG** for more info
5. **Use `-mllvm -debug` flag** to trace execution

---

## Next Steps

1. **Read** this README completely
2. **Understand** each concept
3. **Find** the code location
4. **Copy** the code template
5. **Implement** step-by-step
6. **Compile** LLVM
7. **Test** your code
8. **Debug** if needed
9. **Submit** to LLVM!

---

## Good Luck! 🚀

You've got this! Start with understanding, then code, then test.

**Remember:** If you get stuck, re-read this README. The answer is usually here!
 
