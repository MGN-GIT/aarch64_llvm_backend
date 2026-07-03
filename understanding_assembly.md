# ARM64 Assembly Notes

Complete guide: https://mariokartwii.com/arm64/index.html

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

  
