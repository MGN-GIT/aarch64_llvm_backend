# ARM64 Assembly Notes

Complete guide: https://mariokartwii.com/arm64/index.html

Refer to the AArch64 All Assembly instructions guide for know about all instructions

The below is a summary of the above tutorial.

---

## AArch64 Registers

* General-Purpose Registers
  * X0-X30 : 64-bit General-Purpose Registers [Extended Registers]
  * W0-W30 : Lower 32 bits of X registers [Non-extended Registers]
  * SP     : Stack Pointer [Points to lower address of stack]
  * PC     : Program Counter (not directly accessible as a GPR)
  * PSTATE : Processor state flags
  * X29 : Frame Pointer (FP)
  * X30 : Link register (Stores return address)
  * Zero: Special Register holding Zero:
    * wzr : Use this name for Non-Extended purposes
    * xzr : Use this name for Extended purposes

* Floating-Point / SIMD Registers
  * V0-V31 : 128-bit SIMD & FP registers
  * Views of each V register:
    * B0-B31 : 8-bit
    * H0-H31 : 16-bit
    * S0-S31 : 32-bit
    * D0-D31 : 64-bit
    * Q0-Q31 : 128-bit

---

## Memory

* Memory is the region where Instructions (Code) and Data are generally stored.
* Location of Memory is called Memory addresses.
* RAM (main memory) stores both a program's instructions and its data while the program is running.
* Each process has its own virtual address space, divided into regions: text (code) segment, read-only data, initialized data, BSS, heap, and stack.
  * The text segment contains the program's executable instructions.
  * The heap is used for dynamically allocated memory.
  * The stack stores function call frames and local variables.
* Each byte in memory has a unique memory address, which the CPU uses to access instructions and data.
* On modern operating systems, techniques such as ASLR may cause the absolute addresses of these regions to change between program executions, but their relative layout remains the same within a process.
* Generally, Memory addresses are always represented in hexadecimal because it is a compact and convenient way to represent binary values.
* Consider this:
  * Binary:
    ```
    00000000000000000000000000010000
    ```
  * In terms of hex:
    ```
    0x00000010
    ```

---

## Basics of writing Assembly

* Comments:
  * As similar to C++ comments, Assembly supports both single and multi-line comments.
  * Single Line comment:
    ```
    // This is a comment.
    ```
  * Multi Line comment:
    ```
    /*
    this
    is a
    comment
    */
    ```

* Instruction Format:
  * In most instructions, the Destination Register is the Register that holds the result of an executed instruction.
  * The Source Register is the Register that is used to compute the result for the Destination Register.
  * Some instructions will have one source register, while others will have two.
  * There are generally 4 formats that an Instruction can be represented:
    * Ins rD, rA, rB
    * Ins rD, rA
    * Ins rD, rA, VALUE
    * Ins rD, VALUE
  * Let's understand these terms:
    * ins = Instruction Mnemonic
    * rD = Destination Register
    * rA = 1st Source Register
    * rB = 2nd Source Register
    * VALUE = Immediate Value
  * An immediate value is a constant encoded directly inside the instruction itself:
    ```
    mov w0, #10
    ```
    * 10 is an immediate value:
      * It is not read from a register.
      * It is not read from memory.
      * The CPU gets the value directly from the instruction.

* Signed vs Unsigned Representation:
  * Bit 63 ............. Bit 0
  * The leftmost bit (bit 63) is the sign bit only when interpreting the value as signed.
    * MSB = 0 → non-negative
    * MSB = 1 → negative (using two's complement representation)

---

## Basic ARM64 Instructions

* ADD instruction:
  * The add instruction has two "versions" — one for extended register use, one for non-extended use:
    ```
    add xD, xA, xB
    add wD, wA, wB
    ```
  * In the add instruction, the value in xA is added with the value in xB. The result is placed into xD.
  * Whatever value that was in xD beforehand is overwritten.

* Add Immediate Instruction:
  ```
  add xD, xA, aimm
  add wD, wA, aimm
  ```
  * aimm stands for Arithmetic Immediate Value. It covers the following Immediate Value Types:
    * UIMM12 – A 12-bit unsigned immediate value (range: 0 to 4095).
    * UIMM24 – A 24-bit unsigned immediate value (range: 0 to 16,777,215).
    * NIMM25 – A 25-bit signed immediate value (encoded in two's complement, allowing both positive and negative values).
  * Immediate values in ARM64 are conventionally written with a `#` prefix.

* Sub instruction:
  * sub subtracts one register or an immediate value from another:
    ```
    sub xD, xA, xB     // xD = xA - xB
    sub xD, xA, #imm   // xD = xA - immediate
    ```

* Division instruction:
  ```
  sdiv xD, xA, xB   // Signed Division. xD = xA / xB
  sdiv wD, wA, wB   // Signed Division. wD = wA / wB
  udiv xD, xA, xB   // Unsigned Division. xD = xA / xB
  udiv wD, wA, wB   // Unsigned Division. wD = wA / wB
  ```
  * NOTE: If a division by zero occurs, the result is always 0.
  * NOTE: If a result is partial, standard rounding is applied (5.7 rounds to 6).
  * NOTE: Cannot use a constant (immediate value) directly as an operand in ARM64 multiply or divide instructions. Both operands must be registers.

* Negate instruction:
  ```
  neg wD, wA
  neg xD, xA
  ```
  * This simply flips a positive number to be negative or vice-versa. Instances of zero remain as zero.

* Mov Instruction:
  ```
  mov xD, SIMM32   // 0xFFFFFFFFFFFF8000 thru 0x0000000000007FFF
  mov wD, SIMM32   // 0xFFFF8000 thru 0x00007FFF
  mov xD, xA
  mov wD, xA
  ```
  * Mov instruction (with the Immediate Value option) is what you use to write values from 'scratch' into a Register.

* Writing Imm 64-bit Values to Registers:
  * A single mov cannot represent every 64-bit constant.
  * To load arbitrary constants, you either:
    * Use `ldr =constant`, letting the assembler load it from a literal pool:
      ```
      ldr x0, =0x123456789ABCDEF0
      ```
    * Use movz + movk to construct the value 16 bits at a time:
      ```
      movz x8, #0x1234, lsl #48
      movk x8, #0x5678, lsl #32
      movk x8, #0x9ABC, lsl #16
      movk x8, #0xDEF0
      ```
      * movz:
        ```
        movz x8, #0x1234, lsl #48
             x8  1234 0000 0000 0000   // Everything else becomes zero.
        ```
      * movk:
        ```
        movk x8, #0x5678, lsl #32
             x8  1234 5678 0000 0000  // 1234 stayed the same (Keep) && 5678 was inserted.
        ```
      * The `lsl` chooses which 16-bit chunk to modify:
        ```
        +------+------+------+------+
        |63-48 |47-32 |31-16 |15-0  |
        +------+------+------+------+
        ```
      * movk writes to one box at a time. Useful way to remember it:

        | `lsl` | Places the 16-bit immediate in |
        | ----: | ------------------------------ |
        |  `#0` | Bits 15–0 (lowest 16 bits)     |
        | `#16` | Bits 31–16                     |
        | `#32` | Bits 47–32                     |
        | `#48` | Bits 63–48 (highest 16 bits)   |

---

## Basic Loads and Stores

* How to interact with Memory?
  * Store = Value copied from Registers to Memory
  * Load = Value copied from Memory to Registers
  * For *any* load or store instruction, a Memory Address must first be calculated via the addition of the two source registers, or the addition of the source register with the Immediate Value. The result of this equation is known as the *Effective Address*.

### Store Double-Word (64-bit)

* ```
  str xD, [xA, UIMM12]
  str xD, [xA, sSIMM12]***
  str xD, [xA, xB]

  Effective Address:
  xA + UIMM12 = Effective Address
  xA + sSIMM12 = Effective Address
  xA + xB = Effective Address

  [] -> treat as location of address
  ```
* ***sSIMM12 stands for scaled 12-bit Signed Offset.
* Scaled means that the Immediate Offset must be of a certain multiple.
* The size of the multiple is dependent on the data size used within the Destination Register.
* Since the Destination Register is for a Double-Word, the sSIMM12 must be a value that is a multiple of 8.
* Store = *Copy-Paste* to Memory.

### Store Word (32-bit)

* ```
  str wD, [xA, UIMM12]
  str wD, [xA, sSIMM12]   // Offset must be a multiple of 4
  str wD, [xA, xB]
  ```
* This stores the ENTIRE 32-bits of the non-extended register wD to the Effective Address.
* Example: w10 = 0x777C010A stored at the Effective Address becomes `0A017C77` in memory (because of Little Endian).

### Store Halfword (16-bit)

* ```
  strh wD, [xA, UIMM12]
  strh wD, [xA, sSIMM12]   // Offset must be a multiple of 2
  strh wD, [xA, xB]
  ```
* This stores the LOWER 16-bits of the non-extended register wD to the Effective Address.
* Example:
  ```
  strh w8, [x5, #0x108]
  w8 = 0x7FFF
  x5 = 0x55008004B0

  EA = 0x55008004B0 + 0x108 = 0x55008005B8
  ```
* Due to little endian, the halfword value is modified to `0xFF7F` when stored.

### Store Byte

* ```
  strb wD, [xA, UIMM12]
  strb wD, [xA, SIMM12]   // Scaled offset isn't applicable, multiple of 1
  strb wD, [xA, xB]
  ```
* This stores the LOWER 8-bits of the non-extended register wD to the Effective Address.
* Example:
  ```
  strb w20, [x26, #0x8F]
  w20 = 0x80343E80
  x26 = 0x550080074C

  EA = 0x550080074C + 0x8F = 0x55008007DB
  ```
* Only the lower 8 bits (`0x80`) are stored. Because it's a single byte, the reverse-mechanism has no visible effect — `0x80` is stored as `0x80`.

### Little Endian

* ARM64 uses Little Endian. Any load/store on a non-byte value splits the value into bytes and stores/loads those bytes in **reverse order**.
* Reference table:

  | Instruction                  | Register Value       | Memory               |
  |-------------------------------|-----------------------|------------------------|
  | str xD (store doubleword)     | 0x0123456789ABCDEF    | 0xEFCDAB8967452301     |
  | str wD (store word)           | 0x01234567             | 0x67452301              |
  | strh wD (store halfword)      | 0x0123                 | 0x2301                  |
  | strb wD (store byte)          | 0x01                   | 0x01                    |

* IMPORTANT: Because of Little Endian, if you want to see the true/unconverted contents of Memory in GDB, you must always use the **Byte** unit type (`b`) — see GDB Memory Commands below.

### Load instructions

* Load = *Copy-Paste* from Memory to Register. Whatever was in the Register beforehand is now overwritten.
* Load instructions calculate the Effective Address the same way Store instructions do.

#### Load Double-Word

* ```
  ldr xD, [xA, UIMM12]
  ldr xD, [xA, sSIMM12]   // Offset must be multiple of 8
  ldr xD, [xA, xB]
  ```
* The double-word value located at the Effective Address is copy-pasted into xD.
* Example:
  ```
  ldr x11, [x13, #0xF0]
  x13 = 0x5500800574
  EA = 0x5500800574 + 0xF0 = 0x5500800664
  ```
* Due to little endian, the double-word value is split into bytes and reversed when placed into x11.

#### Load Word

* ```
  ldr wD, [xA, UIMM12]
  ldr wD, [xA, sSIMM12]   // Offset must be multiple of 4
  ldr wD, [xA, xB]
  ```
* The upper 32-bits (Extended-only portion) of the Destination Register (xD) are ALWAYS set to zero. True for EVERY load instruction where the Destination Register is non-extended (wD).
* Example:
  ```
  ldr w10, [x0, #0x4]
  x0 = 0x5500800474
  EA = 0x5500800474 + 0x4 = 0x5500800478
  ```
* Due to little endian, `0x0B000000` at the EA becomes `0x0000000B` when placed into w10. The extended portion (upper 32 bits of x10) is nulled.

#### Load Halfword

* ```
  ldrh wD, [xA, UIMM12]
  ldrh wD, [xA, sSIMM12]   // Offset must be multiple of 2
  ldrh wD, [xA, xB]
  ```
* The Extended-only portion bits (upper 32-bits of xD) are set to zero. The upper 16-bits of wD are also set to zero.
* Result: wD always ends up as `0x0000XXXX`.
* Example:
  ```
  ldrh w24, [x19, #0x500]
  x19 = 0x5500800136
  EA = 0x5500800136 + 0x500 = 0x5500800636
  ```
* Due to little endian, the halfword value is split into bytes and reversed when placed into w24.

#### Load Byte

* ```
  ldrb wD, [xA, UIMM12]
  ldrb wD, [xA, SIMM12]   // Scaled offset isn't applicable, multiple of 1
  ldrb wD, [xA, xB]
  ```
* The Extended-only portion bits (upper 32-bits of xD) are set to zero. The upper 24-bits of wD are also set to zero. Result: wD always ends up as `0x000000XX`.
* Example:
  ```
  ldrb w0, [x16, #0x2C]
  x16 = 0x55007FFEEB
  EA = 0x55007FFEEB + 0x2C = 0x55007FFF17
  ```
* The byte value at the EA is loaded into w0. Upper 32 bits (x0) and upper 24 bits (w0) are nulled.

### Same-register loads/stores

* Example 1:
  ```
  str x5, [x5, #0xF24]
  ```
  Destination and Source register use the same GPR. Works exactly like any other basic store — calculate EA (x5 + 0xF24), then store x5 to it.

* Example 2:
  ```
  ldrb w3, [x3]
  ```
  Destination and Source Register use the same GPR but different forms (non-extended vs extended). The byte value at the address in x3 is loaded into w3 — so x3 is no longer its original value.

### Final Notes

* If an Immediate Value is 0, you don't need to include it:
  ```
  str x9, [x22, #0]
  ```
  ...can be written as...
  ```
  str x9, [x22]
  ```
---
## Compares and Branches

Compares and Branches let you create conditional paths in your program (if/else logic).

### Unconditional Branch

```
b SIMM
b label
```

- "Unconditional" means the branch always executes, regardless of any condition.
- SIMM determines how far to jump. Any instructions jumped over do **NOT** execute.
- Since every ARM64 instruction is 4 bytes, a SIMM of `0x4` is useless — it just goes to the very next instruction.
- Branch instructions use a SIMM value, so backward branches are possible.
- Labels let the Assembler calculate the SIMM for you. Any name is fine as long as special characters (e.g. `$`) are omitted.
- When you supply a label, its "landing spot" must repeat the exact label name, appended with a colon:

```
b jump_here

add w13, w14, w15

jump_here:
mov x1, #1
```

### Conditional Branches

Conditional branches only execute based on an 'if' — they require a comparison instruction beforehand.

**Compare instruction:**

```
cmp xD, aimm
cmp wD, aimm
cmp xD, xA
cmp wD, wA
```

Compares the Destination Register against the Immediate Value / Source Register and sets condition flags for the next conditional branch to read.

**Example — Branch If Equal:**

```
cmp x0, #0
beq jump_here

add w1, w2, w3

jump_here:
str x7, [x7]
```

If x0 is 0:
1. `cmp` executes, sets condition flags.
2. `beq` is taken (x0 IS equal to 0).
3. Execution jumps over the `add` and lands at `str`.
4. `str` executes.

If x0 is NOT 0:
1. `cmp` executes, sets condition flags.
2. `beq` is NOT taken.
3. Execution falls through normally.
4. `add` executes, then `str` executes.

**A slightly larger example:**

```
cmp x0, #0
beq condition_met

mov x1, #1
b the_end

condition_met:
strh w1, [x2]

the_end:
str x3, [x4, #0x40]
```

### Signed vs Unsigned Treatment of Values

Conditional branches determine whether the values in the most recent `cmp` are treated as Signed or Unsigned — the register itself doesn't decide this, the branch mnemonic does.

```
bhs label   // Branch if Greater than or Equal to (Unsigned)
bgt label   // Branch if Greater than (Signed)
```

**Full list of conditional branch instructions:**

```
beq = Equal
bne = Not Equal
bgt = Greater Than (signed)
blt = Less Than (signed)
bge = Greater Than or Equal (signed)
ble = Less Than or Equal (signed)
bhs = Unsigned Higher or Same (aka Unsigned Greater Than or Equal)   // same as bcs
blo = Unsigned Lower Than (aka Unsigned Less Than)                    // same as bcc
bmi = Negative
bpl = Positive or Zero (aka Not Negative)
bvs = Signed Overflow
bvc = Not Signed Overflow
bhi = Unsigned Higher (aka Unsigned Greater Than)
bls = Unsigned Lower or Same (aka Unsigned Less Than or Equal)
bcs = Same as bhs (branch Carry Set)
bcc = Same as blo (branch Carry Clear)
bal = Always (same as an unconditional branch)
```

**Worked example — signed vs unsigned changes the outcome:**

```
w2  = 0xFFFFFFFF
w19 = 0x000000A0

cmp w2, w19   // Compare w2 vs w19

bgt somewhere   // Signed: w2 = -1, w19 = 160 -> -1 is NOT > 160 -> branch NOT taken
bhi somewhere   // Unsigned: w2 = 4294967295, w19 = 160 -> branch IS taken
```

### Exercise: conditional multiply

Goal: load a word, multiply it by 13 only if it's 0 or positive, then store it back. Assume the word value is at the address in x12. Use w6 and w7 for the multiplication.

```
ldr w6, [x12]     // Load our value from memory into w6
cmp w6, #0        // Compare value in w6 against Zero
blt done          // If w6 < 0 (signed), branch to "done:" - skip the multiply
mov w7, #13       // Place the multiple into w7 (mul can't use an immediate directly)
mul w6, w6, w7    // w6 = w6 * w7
str w6, [x12]     // Store new value back to where we loaded it from
done:
```

**Full source example:**

```
section .data
data_pointer
.long 0x00000000   // EDIT THIS TO YOUR LIKING BEFORE ASSEMBLING

.section .text
.globl _start
_start:
nop                     // For possible GDB Registers Unavailable bug
adr x12, data_pointer
ldr w6, [x12]     // Load our value from memory into w6
cmp w6, #0        // Compare value in w6 against Zero
blt done          // If w6 < 0 (signed), branch to "done:" - skip the multiply
mov w7, #13       // Place the multiple into w7
mul w6, w6, w7    // w6 = w6 * w7
str w6, [x12]     // Store new value back to where we loaded it from
done:
```

Once you step past the `str` instruction, the program will fault.

### Final Notes

- A conditional branch always analyzes the results from the **MOST RECENT** compare instruction.
- Official ARM format requires a dot after the `b` (e.g. `b.ne`). Most assemblers let you omit it, but GDB will still display the dot when disassembling conditional branches.

  ---
  ## Pre and Post Index of Loads/Stores

Loads and Stores can have additional operations done to them to update the Source Register's Address immediately *after* the instruction has executed. This is useful for things like loops.

- **Pre-Indexing** = Load/Store to the Effective Address, then increase/decrease the Source Register Address/Value by the Immediate Value present in the instruction.
- **Post-Indexing** = Load/Store to the Source Register Address, then increase/decrease the Source Register Address/Value by the Immediate Value present in the instruction.

### Pre-Index Store

```
str x0, [x1, #0x4]!
```

The `!` at the end marks this as a pre-index store.

What occurs:
1. `x1 + 0x4` = Effective Address
2. x0 (entire double-word) is stored at the Effective Address
3. x1 is then incremented by `0x4` — it holds the new value if the instruction executes again

**Worked example:**

```
str x0, [x1, #0x4]!

x0 = 5
x1 = 0x40008002B0
```

- Effective Address = `x1 + 4` = `0x40008002B4`
- x0 is stored to `0x40008002B4`
- x1 is updated to `0x40008002B4` after the store
- Because of Little Endian, the value in memory becomes `0x0500000000000000`

### Post-Index Store

```
str x0, [x1], #0x4
```

Note: only the Source Register is inside the brackets; the Immediate Value comes after a second comma.

What occurs:
1. `x1` = Effective Address (no offset added yet)
2. x0 (entire double-word) is stored at the Effective Address
3. x1 is then incremented by `0x4` — it holds the new value if the instruction executes again

**Worked example:**

```
str x0, [x1], #0x4

x0 = 5
x1 = 0x40008002B0
```

- Effective Address = `x1` = `0x40008002B0`
- x0 is stored at `0x40008002B0` **before** x1 is incremented
- x1 is then incremented by 4, becoming `0x40008002B4`
- Because of Little Endian, the value in memory becomes `0x0500000000000000`

### Pre-Index Load

```
ldr w5, [x16, #0x3C]!
```

What occurs:
1. `x16 + 0x3C` = Effective Address
2. Word value at the Effective Address is loaded into w5
3. x16 is then incremented by `0x3C` — it holds the new value if the instruction executes again

### Post-Index Load

```
ldr w5, [x16], #0x3C
```

What occurs:
1. `x16` = Effective Address (no offset added yet)
2. Word value at the Effective Address is loaded into w5
3. x16 is then incremented by `0x3C` — it holds the new value if the instruction executes again

### Note:
* Post-index: Access first, then update.
* Pre-index: Update first, then access.
---

## Loops

Loops transfer (copy-paste) a chunk of data from one area of memory to another. You need 4 items to write a loop:

1. Start Address where Contents are originally at (Source Address)
2. Start Address where you want Contents copied to (Destination Address)
3. Length of the Contents, tracked via a GPR ("loop tracker" — how many times the loop executes)
4. A Conditional Branch

Consider the size of the contents when deciding transfer granularity. If it divides evenly by 8, transferring double-words at a time makes sense. If it's an odd size like 29 bytes, transfer byte-per-byte.

### Building a byte-copy loop (29 bytes)

Source Address = `0x40007F0000`, Destination Address = `0x4000800000`. Place Source in x1, Destination in x2:

```
// Write x1 (Source) Address
ldr x1, =0x40007F0000

// Write x2 (Destination) Address
ldr x2, =0x4000800000
```

Set a Loop Tracker register (w3) to 29, since we're transferring 29 bytes one at a time:

```
mov w3, #29
```

Load and store one byte at a time, using post-indexed addressing (offset `#1`) so both addresses auto-increment after each transfer. w4 is used as the scratch register:

```
ldrb w4, [x1], #1   // Load byte
strb w4, [x2], #1   // Store byte
```

Decrement the tracker and branch back while it's not zero:

```
sub w3, w3, #1   // Decrement Loop Tracker
cmp w3, #0       // Check when Tracker hits 0
bne loop         // When not zero, we still have bytes to transfer
```

The `loop` label must land at the `ldrb` instruction — **not** at the top where the addresses are set. Landing at the top would reset the post-indexed addresses every iteration, causing an infinite loop:

```
loop:
ldrb w4, [x1], #1   // Load byte
strb w4, [x2], #1   // Store byte
sub w3, w3, #1      // Decrement Loop Tracker for every Byte Transferred
cmp w3, #0          // Check when Tracker hits 0
bne loop            // When not zero, we still have bytes to transfer
```

**Shortcut:** appending `s` to an instruction (e.g. `subs`) gives you a free `cmp rD, #0` on the result, letting you drop the separate `cmp`:

```
subs w3, w3, #1   // Perform the sub, then perform "cmp w3, #0"
bne loop
```

**Full loop:**

```
// Write x1 (Source) Address
ldr x1, =0x40007F0000

// Write x2 (Destination) Address
ldr x2, =0x4000800000

// Set Loop Tracker
mov w3, #29

// Loop
loop:
ldrb w4, [x1], #1   // Load byte
strb w4, [x2], #1   // Store byte
subs w3, w3, #1     // Decrement Loop Tracker, and Compare w3 to 0
bne loop            // When not zero, we still have bytes to transfer
```

**What happens per iteration:**
1. `ldrb` loads a byte into w4; x1 post-increments by 1.
2. `strb` stores the byte from w4; x2 post-increments by 1.
3. `subs` decrements w3 by 1 and sets flags as if `cmp w3, #0` ran.
4. `bne` branches back to `loop` if w3 isn't 0 yet — otherwise the loop ends.

### Exercise: Fibonacci sequence (stop above 1000)

Requirements:
- A register for addition variable #1, starting at 0
- A register for addition variable #2, starting at 1
- A register to hold the addition result
- A backward conditional branch (loop)

```
mov w0, #0        // Variable #1
mov w1, #1        // Variable #2
do_fibonacci:
add w2, w0, w1    // Perform the addition. Result goes into w2
cmp w2, #1000     // Compare result to 1,000
bhi done          // If greater than 1000, *stop* the Fibonacci sequence
mov w0, w1        // Previous result now becomes Variable #1
mov w1, w2        // Latest result now becomes Variable #2
b do_fibonacci    // Do the Fibonacci sequence again
done:             // End of code
```

**Why `bhi` instead of `bgt`:** the running Fibonacci result is always positive (it starts at 1 and only grows), so negative numbers are never possible — treat the comparison as **Unsigned**. Default to `bhi`/`blo` over `bgt`/`blt` unless you specifically know negative numbers are possible and want them treated as negative.

**Why the label placement matters:** `do_fibonacci:` sits right before the `add`, not at the very top of the source. If it were at the top, `w0`/`w1` would reset to 0/1 every iteration, w2 would never exceed 1000, and you'd get an infinite loop.

**Why two `mov`s before looping back:** the next addition needs to be *previous result + latest result*. `mov w0, w1` shifts the old latest result into variable #1; `mov w1, w2` makes the newest result variable #2 for the next pass.

**Full source example:**

```
.section .text
.globl _start
_start:
nop                    // For possible GDB Registers Unavailable bug
mov w0, #0             // Variable #1
mov w1, #1             // Variable #2
do_fibonacci:
add w2, w0, w1         // Perform the addition. Result goes into w2
cmp w2, #1000          // Compare result to 1,000
bhi done               // If greater than 1000, stop the Fibonacci sequence
mov w0, w1             // Previous result now becomes Variable #1
mov w1, w2             // Latest result now becomes Variable #2
b do_fibonacci         // Do the Fibonacci sequence again
done:                  // End of code
```

Once w2 exceeds 1000, continuing to step will fault the program.

---
## Logical Operations

### Fundamentals

- "High" or "true" = a bit with value 1. "Low" or "false" = a bit with value 0.
- The far left-hand bit is the **Sign Bit** / **Most Significant Bit (MSB)**. The far right-hand bit is the **Least Significant Bit (LSB)**.
- Extended registers: MSB = bit 63, LSB = bit 0. Non-extended registers: MSB = bit 31, LSB = bit 0. Bit numbers descend left to right.
- Logical operations happen **bit-by-bit**. For `w3` op `w27`: bit 31 of w3 with bit 31 of w27, bit 30 with bit 30, and so on down to bit 0 with bit 0.

There are 6 conceptual logical operations (OR, AND, XOR, NOR, NAND, XNOR/EQV), but ARM64 provides **7 instructions** — not a 1:1 mapping:

```
orr   // Logical OR
and   // Logical AND
eor   // Logical XOR (Exclusive-OR)
bic   // Logical AND w/ Complement (Bit Instruction Clear)
orn   // Logical OR w/ Complement
eon   // Logical XOR w/ Complement
mvn   // Logical NOT (Logical NOR w/ itself)
```

**Instruction formats:**

`orr`, `and`, `eor` allow an immediate:
```
wD, wA, wB
wD, wA, bimm32*
xD, xA, xB
xD, xA, bimm64*
```

`bic`, `orn`, `eon` do **not** allow an immediate:
```
wD, wA, wB
xD, xA, xB
```

*`bimm32`/`bimm64` immediates don't follow a simple numeric range — they're generated by a bitmask-rotation encoding. Rather than deriving the formula by hand, refer to a precomputed list of valid immediate values.

### Logical OR (orr)

Truth table:

| Input | Input | Result |
|:-----:|:-----:|:------:|
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 1 |

If any bit is high, the result is true.

Example: `w3 = 0x00000001`, `w27 = 0x80000001`

```
w3   0000 0000 0000 0000 0000 0000 0000 0001
w27  1000 0000 0000 0000 0000 0000 0000 0001
```

Each bit is OR'd with its corresponding bit → result:

```
1000 0000 0000 0000 0000 0000 0000 0001   // = 0x80000001
```

`orr` is a great way to set a specific bit high while leaving all other bits alone — a bit that's already high stays high:

```
ldr w0, [x1]                 // Load value from memory
orr w0, w0, #0x00008000      // Set bit 15 high, leave all other bits alone
str w0, [x1]                 // Store back new modified value to memory
```

### Logical AND (and)

Truth table:

| Input | Input | Result |
|:-----:|:-----:|:------:|
| 0 | 0 | 0 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 1 |

Both bits must be high for the result to be true.

**Checking even/odd:**

```
ands w0, w13, #0x00000001   // Wipes all bits except bit 0 (LSB); w0 is a scratch register
beq odd                      // Jump if w13 is even (result was 0)
```

`s`-suffixed instructions (like `ands`) give a free `cmp wD, #0` on the result — but **only `and` and `bic`** support this among the logical operations.

**Checking address alignment** (lower N bits null = aligned):

```
ands w0, w23, #0x3F   // 64-byte alignment check (divisible by 0x40)
bne unaligned

ands w0, w23, #0x1F   // 32-byte alignment check (divisible by 0x20)
bne unaligned

ands w0, w23, #0xF    // 16-byte alignment check (divisible by 0x10)
bne unaligned
```

### Logical XOR (eor)

Truth table:

| Input | Input | Result |
|:-----:|:-----:|:------:|
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

True only when the two bits differ.

`eor` is a great way to flip a single bit without affecting others:

```
ldr w0, [x1]              // Load value from memory
eor w0, w0, #0x0008        // Flip bit 3
str w0, [x1]              // Store value back to memory
```

XOR'ing a value with itself always produces 0. XOR is heavily used in encryption/hashing.

### Complements (Bitwise Negation)

A **Complement** is a Logical NOT applied to a value (each bit flips: 0↔1). Example: `0x0000FFFF` → `0xFFFF0000`; `0x8000FFF2` → `0x7FFF000E`.

ARM64 has no dedicated NOR instruction, and `0xFFFFFFFF`/`0xFFFFFFFFFFFFFFFF` can't be used as an XOR immediate — instead, use `mvn` (Move with Bitwise Negate):

```
mvn x4, x7    // Logical NOT of x7 placed into x4
mvn w0, w0    // Logical NOT of w0, in place
mvn w7, 0xFF  // Logical NOT of 0xFF (= 0xFFFFFF00) placed into w7
```

### bic, orn, eon (Complement variants)

**`bic` (AND w/ Complement, "Bit Instruction Clear")** — used to clear specific bits:

```
mov w8, 0x0010    // bic can't take an immediate directly
bic w7, w7, w8    // Clears bit 4 in w7, leaves everything else alone
```

What actually happens:
1. Logical NOT on `0x00000010` → `0xFFFFFFEF`
2. Logical AND of w7 with `0xFFFFFFEF`
3. Result placed back into w7

**`orn` (OR w/ Complement):**

```
orn w16, w2, w14
```
1. Logical NOT applied to w14
2. Result OR'd with w2
3. Result placed into w16

**`eon` (XOR w/ Complement):**

```
eon w5, w0, w27
```
1. Logical NOT applied to w27
2. Result XOR'd with w0
3. Result placed into w5

`eon` happens to produce identical results to Logical XNOR / EQV:

| Input | Input | Result |
|:-----:|:-----:|:------:|
| 0 | 0 | 1 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 1 |

Note: `bic` does **not** match NAND, and `orn` does **not** match NOR — but `and`, `orr`, `eor`, `bic`, `orn`, `eon`, and `mvn` together cover every logical scenario a program realistically needs, which is why ARM64 stops there.

### ANDing vs TESTing

`tst` performs a Logical AND, updates condition flags for a subsequent conditional branch, and discards the result — no scratch register needed:

```
tst w13, #0x00000001
beq odd
```

**Test-and-branch combined instructions** (test a single specific bit only):

```
// Test then Branch if Zero
tbz wD, imm, label
tbz xD, imm, label

// Test then Branch if not Zero
tbnz wD, imm, label
tbnz xD, imm, label
```

`imm` is the bit index: 0–31 for non-extended (`wD`), 0–63 for extended (`xD`).

Optimized even/odd check:

```
tbnz w13, 0, odd   // Test bit 0 (LSB) directly
```

### Final Note

Immediate values can be written in binary by prepending `0b`:

```
and w26, w26, 0b10
```
