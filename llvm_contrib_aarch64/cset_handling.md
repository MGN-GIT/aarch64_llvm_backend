# Condition optimizer contribution

TODO 1: Handling Condition Select instructions family for Cross-Block optimization

* Cross Block vs Inter Block optimizations:
   * Cross-block optimizer (global optimizer):
      * Operates across multiple basic blocks using the function's control flow graph.
      * Understands branches, merges, and loops.
      * Performs more advanced optimizations such as global constant propagation, global common subexpression elimination, loop-invariant code motion, code motion, and global dead code elimination.
   * Intra-block optimizer (local optimizer):
      * Operates within a single basic block.
      * Does not consider branches or loops.
      * Performs simple, fast optimizations like constant folding, local common subexpression elimination, algebraic simplification, and local dead code elimination.

* Our Goal:
   * Consider the following example:
     ```
     int check(int a) {
          volatile int sink;
          int flag = (a > 4);   // CSET
          sink = flag;           // volatile store forces block split
          if (a < 6) { // CSEL
              return 1;
          }
          return 0;
      }
     ```
   * The LLVM compiler currently converts this into the following assembly:
   ```
   Before check(int):
        sub     sp, sp, #16
        cmp     w0, #4 <- Look here P1 
        cset    w8, gt
        cmp     w0, #6 <- Look here P2
        cset    w0, lt
        str     w8, [sp, #12]
        add     sp, sp, #16
        ret
   ```
   * Look at P1 and P2 clearly, both cmp instructions compare with imm value 5.
   * Ideally compiler should have optimized the imm val to #5 through cross-block optimizer for conditional select family instructions.
   * But, LLVM currently support cross-block optimizer for branch instructions causing LLVM backend to generate 2 different CMP instructions.
   * So, adding cross-block optimizer support for conditional select family would help CSE(Common subexpression elimination) to eliminate duplicate CMPs.
   * Note: The cross-block or intra-block optimizer applies when the immediate values of the two CMPs differ by 1 or 2
   ```
   After check(int):
        sub     sp, sp, #16
        cmp     w0, #5 <- Cross-block optimizer changes imm val to #5
        cset    w8, gt  
        cset    w0, lt
        str     w8, [sp, #12]
        add     sp, sp, #16
        ret
   ```
   * To achieve this, we have two commits: one for the head basic block (HBB), and another for the true basic block (TBB).

Our Approach:

Patch: Detects a valid conditional select family instruction in AArch64 for both Head and True-successor blocks:

1. Lets create a helper function named `findSelectConsumer` that takes Machine Basic Block returns a pair of values{Machine Instruction, Condition code}.
2. The `findSelectConsumer` return two outputs[Traverse through MBB in reverse]:
   * Hit Case: A Select machine code found(Reads NZCV flags and set value) comes after a CMP instruction -> Returns{machine code, Condition code - cc}
     
   * Example for Hit Case:
     
   ```
   CMP  x0, x1        ; (1) writes NZCV -> Sets NZCV flags
   CSET x2, EQ        ; (2) reads NZCV, writes x2 -> Reads NZCV flags
   findSelectConsumer() -> Returns {CSET, EQ}
   ```
   
   * Miss Case: Found adjacent instructions that reads NZCV flags before a CMP instruction -> Returns {nullptr, Invalid}

   * Example for Miss Case:

   ```
   CMP   x0, x1       ; writes NZCV
   ADDS  x4, x5, x6   ; also writes NZCV! ← overwrites the sticky note
   CSET  x2, EQ       ; reads NZCV — but whose flags is it actually reading?
   ```
   
   

   

  




























