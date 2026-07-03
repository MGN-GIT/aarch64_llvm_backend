Complete guide: https://mariokartwii.com/arm64/index.html

The below is just a summary of above tutorial:

* AArch64 Registers :
  * General-Purpose Registers
    * X0-X30   : 64-bit General-Purpose Registers [Extended Registers]
    * W0-W30   : Lower 32 bits of X registers [Non-extended Registers]
    * SP        : Stack Pointer [Points to lower address of stack]
    * PC        : Program Counter (not directly accessible as a GPR)
    * PSTATE    : Processor state flags
    * X29 : Frame Pointer(FP)
    * X30 : Link register (Stores return address)
    * Zero: Special Register holding Zero:
      * wzr : Use this name for Non-Extended purposes
      * xzr : Use this name for Extended purposes

  * Floating-Point / SIMD Registers
    * V0-V31    : 128-bit SIMD & FP registers
    * Views of each V register:
      * B0-B31    : 8-bit
      * H0-H31    : 16-bit
      * S0-S31    : 32-bit
      * D0-D31    : 64-bit
      * Q0-Q31    : 128-bit
     
* Memory:
  * Memory is region where Instructions(Code) and Data are generally stored.
  * Location of Memory is called Memory addresses.
  * RAM (main memory) stores both a program's instructions and its data while the program is running.
  * Each process has its own virtual address space, which is divided into regions such as the text (code) segment, read-only data, initialized data, BSS, heap, and stack. The text segment contains the program's executable instructions, while the heap is used for dynamically allocated memory and the stack stores function call frames and local variables.
  * Each byte in memory has a unique memory address, which the CPU uses to access instructions and data.
  * On modern operating systems, techniques such as ASLR may cause the absolute addresses of these regions to change between program executions, but their relative layout remains the same within a process.
  * Generally, Memory addresses are always represented in hexadecimal because it is a compact and convenient way to represent binary values.
  * Consider this:
    * Binary:
      * 00000000000000000000000000010000
    * In terms of hex:
      * 0x00000010
     
* Basics of writing Assembly:
  * Comments:
    * As similar to C++ comments, Assembly supports both single and multi-line comments
    * Single Line comment:
    
      ```// This is a comment.```
      
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
      *  Lets understand these terms:
        * ins = Instruction Mnemonic
        * rD = Destination Register
        * rA = 1st Source Register
        * rB = 2nd Source Register
        * VALUE = Immediate Value
      * An immediate value is a constant encoded directly inside the instruction itself:
       ```mov w0, #10```
      * 10 is an immediate value - It is not read from a register - It is not read from memory -The CPU gets the value directly from the instruction.
        
    * Signed vs Unsigned Representation:
      * Bit 63 ............. Bit 0
      * The leftmost bit(bit 63) is the sign bit only when interpreting the value as signed.
        *   MSB = 0 → non-negative
        *   MSB = 1 → negative (using two's complement representation)

* Basic ARM64 Instructions:
  *  ADD instruction:
      *  The add instruction has two "versions". One for extended register use and one for non-extended use:
        * add xD, xA, xB
        * add wD, wA, wB
      * In the add instruction, the value in xA is added with the value in xB. The result of this addition is placed into xD.
      * Whatever value that was in xD beforehand, is overwritten.
  * Add Immediate Instruction:
    ```
    add xD, xA, aimm
    add wD, wA, aimm
    ```
    * aimm stands for Arithmetic Immediate Value. It covers the following Immediate Value Types:
      * UIMM12 – A 12-bit unsigned immediate value (range: 0 to 4095).
      * UIMM24 – A 24-bit unsigned immediate value (range: 0 to 16,777,215).
      * NIMM25 – A 25-bit signed immediate value (encoded in two's complement, allowing both positive and negative values).
    * Immediate values in ARM64 are conventionally written with a # prefix
  * Sub instruction:
    * sub subtracts one register or an immediate value from another:
      ```
      sub xD, xA, xB     // xD = xA - xB
      sub xD, xA, #imm   // xD = xA - immediate
      ```
  * Division instruction:
    ```
    sdiv xD, xA, xB //Signed Division. xD = xA/ xB
    sdiv wD, wA, wB //Signed Division. wD = wA / wB
    udiv xD, xA, xB //Unsigned Division. xD = xA / xB
    udiv wD, wA, wB //Unsigned Division. wD = wA / wB
    ```
    * NOTE: If a division by zero occurs, then the result is always 0.
    * NOTE: If a result is partial, then standard rounding is applied (5.7 rounds to 6)
    * NOTE: Cannot use a constant (immediate value) directly as an operand in ARM64 multiply or divide instructions. Both operands must be registers.
  * Negate instruction:
    ```
    neg wD, wA
    neg xD, xA
    ```
    * This will simply flip a positive number to be negative or vice-versa. Instances of zero remain as zero.
  * Mov Instruction:
    ```
    mov xD, SIMM32 //0xFFFFFFFFFFFF8000 thru 0x0000000000007FFF
    mov wD, SIMM32 //0xFFFF8000 thru 0x00007FFF
    mov xD, xA
    mov wD, xA
    ```
    * Mov instruction (with the Immediate Value option) is what you can use to write values from 'scratch' into a Register.

*  Writing Imm 64-bit Values to Registers:
    * A single mov cannot represent every 64-bit constant.
    * To load arbitrary constants, you either:
      * Use ldr =constant, letting the assembler load it from a literal pool:
        ``` ldr x0, =0x123456789ABCDEF0 ```
      *  Use movz + movk to construct the value 16 bits at a time:
        ```
            movz x8, #0x1234, lsl #48
            movk x8, #0x5678, lsl #32
            movk x8, #0x9ABC, lsl #16
            movk x8, #0xDEF0
        ```
         * movz:
           ```
           movz x8, #0x1234, lsl #48
                x8  1234 0000 0000 0000 #Everything else becomes zero.
           ```
         * movk:
           ```
           movk x8, #0x5678, lsl #32
                x8  1234 5678 000 0000  #1234 stayed the same (Keep) && 5678 was inserted.
           ```
         * The lsl chooses which 16-bit chunk to modify:
         ```
         +------+------+------+------+
         |63-48 |47-32 |31-16 |15-0  |
         +------+------+------+------+
         ```
         * movk writes to one box at a time. Useful way to remember it is:
      
          | `lsl` | Places the 16-bit immediate in |
          | ----: | ------------------------------ |
          |  `#0` | Bits 15–0 (lowest 16 bits)     |
          | `#16` | Bits 31–16                     |
          | `#32` | Bits 47–32                     |
          | `#48` | Bits 63–48 (highest 16 bits)   |

  

 
