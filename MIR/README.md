# Writing MIR Tests — Beginner's Guide
---

## Table of Contents

- [What is a MIR Test and Why Write One?](#what-is-a-mir-test-and-why-write-one)
- [The Shape of a MIR File](#the-shape-of-a-mir-file)
- [Step 1 — Generate a MIR File from C](#step-1--generate-a-mir-file-from-c)
- [Step 2 — Understand What You Are Looking At](#step-2--understand-what-you-are-looking-at)
- [Step 3 — Write the RUN Line](#step-3--write-the-run-line)
- [Step 4 — Write the CHECK Lines](#step-4--write-the-check-lines)
- [Step 5 — Simplify the MIR File](#step-5--simplify-the-mir-file)
- [MIR Syntax You Need to Know](#mir-syntax-you-need-to-know)
  - [Registers](#registers)
  - [Basic Blocks](#basic-blocks)
  - [Instructions](#instructions)
  - [Register Flags](#register-flags)
  - [Memory Operands](#memory-operands)
- [A Complete Worked Example](#a-complete-worked-example)
- [Common Mistakes](#common-mistakes)
- [Quick Reference](#quick-reference)

---

## What is a MIR Test and Why Write One?

When you work on a backend pass — say `AArch64LoadStoreOptimizer` or
`AArch64MIPeepholeOpt` — you need a way to test it in isolation without
running the entire compiler pipeline from C source.

**MIR tests let you do exactly that.**

You write a `.mir` file that contains instructions at a specific point in the
pipeline, run a single pass on it, and check the output.

```
Without MIR tests:
  C source → full pipeline → check final assembly
  If something is wrong, which of the 30+ passes caused it?

With MIR tests:
  Hand-written MIR → single pass → check MIR output
  You know exactly which pass you are testing.
```

---

## The Shape of a MIR File

A MIR file has two parts separated by `---`:

```
Part 1: Embedded LLVM IR module (optional but usually needed)
        Written as a YAML block literal string

---     separator

Part 2: One or more machine functions
        Each one is a YAML document
```

Minimal example:

```yaml
# The embedded IR module
define void @foo() { ret void }
---
# The machine function
name: foo
body: |
  bb.0:
    RET_ReallyLR
...
```

The `name: foo` must match a function name in the IR module above.
The `body:` block contains the actual instructions.

---

## Step 1 — Generate a MIR File from C

You almost never write a MIR file from scratch. You generate one from C code
and then edit it.

**Write your C test case:**

```c
// test.c
int add(int a, int b) {
    return a + b;
}
```

**Stop the compiler just before the pass you want to test:**

```bash
# Stop just before the load/store optimizer
llc -mtriple=aarch64 -stop-before=aarch64-ldst-opt test.ll -o test.mir

# Stop just after a pass (to see what it produced)
llc -mtriple=aarch64 -stop-after=aarch64-ldst-opt test.ll -o test.mir
```

To get `test.ll` from `test.c`:

```bash
clang -O1 -target aarch64 -S -emit-llvm test.c -o test.ll
```

**Find the pass name** by listing all passes:

```bash
llc -mtriple=aarch64 --help-hidden 2>&1 | grep aarch64
```

Common AArch64 pass names:

```
aarch64-ldst-opt          AArch64LoadStoreOptimizer
aarch64-expand-pseudo     AArch64ExpandPseudoInsts
aarch64-mi-peephole       AArch64MIPeepholeOpt
aarch64-condopt           AArch64ConditionOptimizer
aarch64-dead-defs         AArch64DeadRegisterDefinitions
machine-cp                MachineCopyPropagation
postrapseudos             Post-RA pseudo instruction expansion
```

---

## Step 2 — Understand What You Are Looking At

After running `llc -stop-before=...` you will get a verbose MIR file. Here is
what the key parts mean:

```yaml
---
name:            add           # function name — must match IR above
alignment:       4             # function alignment in bytes
tracksRegLiveness: true        # pass needs to know which regs are live
                               # keep this — most passes require it
liveins:
  - { reg: '$w0' }             # W0 is live on entry (argument 'a')
  - { reg: '$w1' }             # W1 is live on entry (argument 'b')

body: |
  bb.0.entry:                  # basic block 0, named "entry"
    liveins: $w0, $w1          # which regs are live at start of this block

    %0:gpr32 = COPY $w0        # virtual reg %0 = copy of W0
    %1:gpr32 = COPY $w1        # virtual reg %1 = copy of W1
    %2:gpr32 = ADDWrr %0, %1   # %2 = %0 + %1
    $w0 = COPY %2              # return value in W0
    RET_ReallyLR implicit $w0  # return
...
```

---

## Step 3 — Write the RUN Line

The RUN line goes at the top of the `.mir` file as a comment. It tells `llc`
which pass to run and pipes the output to `FileCheck`.

```
# RUN: llc -mtriple=aarch64 -run-pass=<pass-name> %s -o - | FileCheck %s
```

Breaking it down:

```
llc                         the LLVM static compiler
-mtriple=aarch64            target architecture
-run-pass=aarch64-ldst-opt  run ONLY this one pass
%s                          the current file (substituted by lit)
-o -                        output to stdout
| FileCheck %s              pipe to FileCheck, patterns also in this file
```

**Running the same pass twice** (if a pass runs multiple times in the pipeline):

```
# RUN: llc -mtriple=aarch64 -run-pass=dead-mi-elimination,1 %s -o - | FileCheck %s
#                                                            ^
#                                                            run index (0-based)
```

---

## Step 4 — Write the CHECK Lines

`FileCheck` scans the output for patterns. You add `CHECK:` comments to your
`.mir` file:

```
# CHECK-LABEL: name: add        ← marks the start of a function to check
# CHECK:       ADDWrr            ← this instruction must appear
# CHECK-NOT:   SUBWrr            ← this instruction must NOT appear
# CHECK:       RET_ReallyLR      ← this must appear after the ADDWrr
```

**Key FileCheck directives:**

```
# CHECK:       pattern    must appear (in order)
# CHECK-NOT:   pattern    must NOT appear between previous and next CHECK
# CHECK-LABEL: pattern    resets position — use for function boundaries
# CHECK-NEXT:  pattern    must appear on the very next line
# CHECK-SAME:  pattern    must appear on the same line as previous CHECK
# CHECK-DAG:   pattern    must appear but order doesn't matter
```

**Practical example — testing that two STRs merge into STP:**

```
# CHECK-LABEL: name: store_pair
# CHECK:       STPXi
# CHECK-NOT:   STRXui
```

---

## Step 5 — Simplify the MIR File

The raw output from `llc -stop-before` is very verbose. Simplify it:

**Remove things you don't need:**

```yaml
# REMOVE these if your test doesn't depend on them:
alignment:       4             # remove — default is fine
exposesReturnsTwice: false     # remove — default
legalized:       true          # remove — default
regBankSelected: true          # remove — default
selected:        true          # remove — default

frameInfo:                     # remove entire section if no stack usage
  maxAlignment:  1

# KEEP this — most passes need it:
tracksRegLiveness: true
```

**Simplify the IR module:**

If your test doesn't depend on global variables or function attributes, replace
the full IR with a dummy:

```llvm
# Before (verbose):
define i32 @add(i32 %a, i32 %b) #0 {
entry:
  %add = add nsw i32 %a, %b
  ret i32 %add
}
attributes #0 = { nounwind optsize ... }

# After (minimal):
define i32 @add(i32, i32) { ret i32 0 }
```

**Simplify block successors:**

```yaml
# Before:
successors: %bb.1(0x40000000), %bb.2(0x40000000)

# After (drop branch weights if test doesn't need them):
successors: %bb.1, %bb.2
```

**Drop memory operand details if not testing alias analysis:**

```yaml
# Before:
STRXui %0, %1, 0 :: (store 8 from %ir.ptr, !alias.scope !9)

# After:
STRXui %0, %1, 0 :: (store (s64))
```

**Drop IR block name references:**

```yaml
# Before:
bb.42.myblock:

# After:
bb.42:
```

---

## MIR Syntax You Need to Know

### Registers

```
$w0, $x0, $sp    Physical registers — prefixed with $
%0, %1, %2       Virtual registers  — prefixed with %
_                Null register (no register / don't care)
```

Virtual registers can have a register class annotation:

```
%0:gpr32         virtual reg %0, must be in the GPR32 class
%1:gpr64         virtual reg %1, must be in the GPR64 class
%2:fpr64         virtual reg %2, must be in the FPR64 class
```

### Basic Blocks

```yaml
bb.0:                    # block with ID 0, no name
bb.0.entry:              # block with ID 0, named "entry"
bb.1.then:               # block with ID 1, named "then"
```

Referencing a block:

```yaml
B %bb.1              # branch to block 1
successors: %bb.1, %bb.2
```

Block with attributes:

```yaml
bb.0 (address-taken):    # this block's address is taken
bb.1 (landing-pad):      # this is an exception landing pad
bb.2 (align 16):         # align to 16 bytes
```

### Instructions

**Instruction with no output:**

```
RET_ReallyLR
B %bb.1
BL @some_function
```

**Instruction with one output:**

```
%0:gpr32 = ADDWrr %1, %2
$w0 = COPY %0
```

**Instruction with multiple outputs:**

```
$sp, $fp, $lr = LDPXpost $sp, 2
```

**The output register(s) come before the `=`, inputs come after.**

**Instruction flags:**

```
$fp = frame-setup ADDXri $sp, 0, 0    # part of prologue
$fp = frame-destroy LDRXui $sp, 0     # part of epilogue
```

### Register Flags

These go before the register name:

```
dead $eflags              result is never used
killed $w0                last use of this register
undef $w1                 value doesn't matter (uninitialized)
implicit $w0              not a real operand, just marks a use
implicit-def $w0          not a real operand, just marks a def
early-clobber %0          this def happens before the uses
renamable $w0             register allocator may rename this
```

Example:

```
dead $eax = XOR32rr undef $eax, undef $eax, implicit-def dead $eflags
```

### Memory Operands

Memory operands come at the end of an instruction after `::`:

```
STRXui %0, %1, 0 :: (store (s64))
LDRWui %0, %1, 0 :: (load (s32))
```

Full form with alignment:

```
STRXui %0, $sp, 0 :: (store (s64) into %stack.0, align 8)
```

Volatile (must not be reordered):

```
LDRWui %0, %1, 0 :: (volatile load (s32))
```

Atomic:

```
LDRWui %0, %1, 0 :: (load acquire (s32))
```

---

## A Complete Worked Example

**Goal:** Test that `AArch64LoadStoreOptimizer` merges two adjacent stores into
an STP instruction.

**Step 1 — Write the C:**

```c
void store_pair(long *p, long a, long b) {
    p[0] = a;
    p[1] = b;
}
```

**Step 2 — Generate MIR stopped before the optimizer:**

```bash
clang -O1 -target aarch64 -S -emit-llvm store_pair.c -o store_pair.ll
llc -mtriple=aarch64 -stop-before=aarch64-ldst-opt store_pair.ll -o store_pair.mir
```

**Step 3 — Edit and simplify into the final test file:**

```yaml
# RUN: llc -mtriple=aarch64 -run-pass=aarch64-ldst-opt %s -o - | FileCheck %s

# CHECK-LABEL: name: store_pair
# CHECK:       STPXi renamable $x1, renamable $x2, renamable $x0, 0
# CHECK-NOT:   STRXui

define void @store_pair(ptr %p, i64 %a, i64 %b) { ret void }
---
name:            store_pair
tracksRegLiveness: true
liveins:
  - { reg: '$x0' }
  - { reg: '$x1' }
  - { reg: '$x2' }
body: |
  bb.0:
    liveins: $x0, $x1, $x2

    STRXui renamable $x1, renamable $x0, 0    ; p[0] = a
    STRXui renamable $x2, renamable $x0, 1    ; p[1] = b  (offset 1 = 8 bytes)
    RET_ReallyLR
...
```

**Step 4 — Run the test:**

```bash
llvm-lit store_pair.mir
# or directly:
llc -mtriple=aarch64 -run-pass=aarch64-ldst-opt store_pair.mir -o - | FileCheck store_pair.mir
```

**What the optimizer does:**

```
Before:
  STRXui $x1, $x0, 0    ; store x1 at x0+0
  STRXui $x2, $x0, 1    ; store x2 at x0+8

After:
  STPXi $x1, $x2, $x0, 0  ; store pair: x1 at x0+0, x2 at x0+8
```

The two separate stores become one pair instruction. The CHECK line verifies
this happened. The CHECK-NOT verifies the old instructions are gone.

---

## Common Mistakes

**1. Forgetting `tracksRegLiveness: true`**

```yaml
# Wrong — many passes will crash or produce wrong output
name: foo
body: |
  ...

# Right
name: foo
tracksRegLiveness: true
body: |
  ...
```

**2. IR function name doesn't match machine function name**

```yaml
# Wrong — names don't match
define i32 @my_func() { ret i32 0 }
---
name: foo          # ← doesn't match @my_func
```

```yaml
# Right
define i32 @foo() { ret i32 0 }
---
name: foo          # ← matches
```

**3. Missing liveins on the block**

```yaml
# Wrong — pass doesn't know $w0 is live, may produce wrong output
bb.0:
  %0:gpr32 = COPY $w0

# Right
bb.0:
  liveins: $w0
  %0:gpr32 = COPY $w0
```

**4. Wrong offset units**

AArch64 load/store immediate offsets are **scaled** — they are in units of the
data size, not bytes:

```yaml
# STRXui stores 8 bytes
# offset 1 = 1 × 8 = 8 bytes from base
STRXui $x1, $x0, 1    ; stores at x0 + 8

# STRWui stores 4 bytes
# offset 1 = 1 × 4 = 4 bytes from base
STRWui $w1, $x0, 1    ; stores at x0 + 4
```

**5. Using the wrong instruction name**

Instruction names are case-sensitive and must match exactly what is in
`AArch64InstrInfo.td`. Use `llc -stop-after` output as your reference — copy
the instruction names directly from there.

---

## Quick Reference

```bash
# Generate MIR stopped before a pass
llc -mtriple=aarch64 -stop-before=<pass> input.ll -o out.mir

# Generate MIR stopped after a pass
llc -mtriple=aarch64 -stop-after=<pass> input.ll -o out.mir

# Run a single pass on a MIR file
llc -mtriple=aarch64 -run-pass=<pass> input.mir -o -

# Simplify the MIR output
llc -mtriple=aarch64 -stop-before=<pass> -simplify-mir input.ll -o out.mir

# List all available pass names
llc -mtriple=aarch64 --help-hidden 2>&1 | grep -i aarch64

# Run a lit test directly
llvm-lit path/to/test.mir
```

```
MIR file structure:
  <embedded IR>          define void @foo() { ret void }
  ---
  name:            foo
  tracksRegLiveness: true
  liveins:
    - { reg: '$x0' }
  body: |
    bb.0:
      liveins: $x0
      <instructions>
  ...

Register syntax:
  $x0, $w0, $sp    physical
  %0, %1           virtual
  %0:gpr64         virtual with register class
  _                null / noreg

Instruction syntax:
  INSTR op1, op2              no output
  %dst = INSTR op1, op2       one output
  $dst1, $dst2 = INSTR op1    multiple outputs

Common register flags:
  killed    last use
  dead      result unused
  implicit  side-effect use/def
  undef     value doesn't matter
  renamable register allocator may rename

FileCheck directives:
  # CHECK:       must appear
  # CHECK-NOT:   must not appear
  # CHECK-LABEL: function boundary
  # CHECK-NEXT:  must be next line
  # CHECK-DAG:   order doesn't matter
```
 
