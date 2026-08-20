enum CondCode {  // Meaning (integer)          Meaning (floating-point)
  EQ = 0x0,      // Equal                      Equal
  NE = 0x1,      // Not equal                  Not equal, or unordered
  HS = 0x2,      // Unsigned higher or same    >, ==, or unordered
  LO = 0x3,      // Unsigned lower             Less than
  MI = 0x4,      // Minus, negative            Less than
  PL = 0x5,      // Plus, positive or zero     >, ==, or unordered
  VS = 0x6,      // Overflow                   Unordered
  VC = 0x7,      // No overflow                Not unordered
  HI = 0x8,      // Unsigned higher            Greater than, or unordered
  LS = 0x9,      // Unsigned lower or same     Less than or equal
  GE = 0xa,      // Greater than or equal      Greater than or equal
  LT = 0xb,      // Less than                  Less than, or unordered
  GT = 0xc,      // Greater than               Greater than
  LE = 0xd,      // Less than or equal         <, ==, or unordered
  AL = 0xe,      // Always (unconditional)     Always (unconditional)
  NV = 0xf,      // Always (unconditional)     Always (unconditional)
  // Note the NV exists purely to disassemble 0b1111. Execution is "always".
  Invalid,

  // Common aliases used for SVE.
  ANY_ACTIVE   = NE, // (!Z)
  FIRST_ACTIVE = MI, // ( N)
  LAST_ACTIVE  = LO, // (!C)
  NONE_ACTIVE  = EQ  // ( Z)
};
