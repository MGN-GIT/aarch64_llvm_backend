# 📗 TableGen Beginner's Guide — Part 2: Advanced Features

> **Series Overview**
> - **Part 1:** Core concepts, syntax, types, values, and basic statements
> - **Part 2 (this file):** multiclass, defm, foreach, bang operators, DAGs, and more
> - **Part 3:** Backends, AArch64 `.td` file patterns, and real-world usage guide

---

## Table of Contents

1. [multiclass — Define Many Records at Once](#1-multiclass--define-many-records-at-once)
2. [defm — Instantiate a multiclass](#2-defm--instantiate-a-multiclass)
3. [Advanced multiclass Patterns](#3-advanced-multiclass-patterns)
4. [foreach — Loop Over a Range](#4-foreach--loop-over-a-range)
5. [Directed Acyclic Graphs (DAGs)](#5-directed-acyclic-graphs-dags)
6. [Bang Operators — TableGen's Built-in Functions](#6-bang-operators--tablegens-built-in-functions)
7. [deftype — Type Aliases](#7-deftype--type-aliases)
8. [dump — Debug Printing](#8-dump--debug-printing)
9. [Using Classes as Subroutines](#9-using-classes-as-subroutines)
10. [The Paste Operator `#` — Deep Dive](#10-the-paste-operator---deep-dive)
11. [Putting It All Together — A Complete Example](#11-putting-it-all-together--a-complete-example)
12. [Common Patterns and Idioms](#12-common-patterns-and-idioms)

---

## 1. `multiclass` — Define Many Records at Once

### The Problem

Suppose you have an architecture where every arithmetic instruction comes in two flavors:
- Register-Register: `ADD X0, X1, X2`
- Register-Immediate: `ADD X0, X1, #42`

Without `multiclass`, you'd write:

```tablegen
def ADD_rr : Instruction<0b111, "add $dst, $src1, $src2", (ops GPR:$dst, GPR:$src1, GPR:$src2)>;
def ADD_ri : Instruction<0b111, "add $dst, $src1, $src2", (ops GPR:$dst, GPR:$src1, Imm:$src2)>;
def SUB_rr : Instruction<0b101, "sub $dst, $src1, $src2", (ops GPR:$dst, GPR:$src1, GPR:$src2)>;
def SUB_ri : Instruction<0b101, "sub $dst, $src1, $src2", (ops GPR:$dst, GPR:$src1, Imm:$src2)>;
def MUL_rr : Instruction<0b100, "mul $dst, $src1, $src2", (ops GPR:$dst, GPR:$src1, GPR:$src2)>;
def MUL_ri : Instruction<0b100, "mul $dst, $src1, $src2", (ops GPR:$dst, GPR:$src1, Imm:$src2)>;
```

This is repetitive. With `multiclass`, you write the pattern once:

### Basic `multiclass` Syntax

```tablegen
multiclass ri_inst<int opc, string asmstr> {
  def _rr : Instruction<opc, asmstr # " $dst, $src1, $src2",
                        (ops GPR:$dst, GPR:$src1, GPR:$src2)>;
  def _ri : Instruction<opc, asmstr # " $dst, $src1, $src2",
                        (ops GPR:$dst, GPR:$src1, Imm:$src2)>;
}

// Instantiate for each operation:
defm ADD : ri_inst<0b111, "add">;
defm SUB : ri_inst<0b101, "sub">;
defm MUL : ri_inst<0b100, "mul">;
```

This produces exactly the same 6 records as before:
```
ADD_rr, ADD_ri
SUB_rr, SUB_ri
MUL_rr, MUL_ri
```

### How Names Are Formed

When a `defm` instantiates a `multiclass`, the `defm` name is **prepended** to each `def` name inside the multiclass:

```
defm ADD  +  def _rr  →  ADD_rr
defm ADD  +  def _ri  →  ADD_ri
```

The `def` names inside a multiclass typically start with `_` or some suffix to make the final names readable.

### The Implicit `NAME` in multiclass

Inside a multiclass, `NAME` refers to the name given in the `defm` statement:

```tablegen
multiclass MyMC {
  def _variant : SomeClass {
    string FullName = NAME # "_variant";   // NAME = whatever defm provides
  }
}

defm FOO : MyMC;
// Creates FOO_variant with FullName = "FOO_variant"
```

These two are equivalent inside a multiclass:
```tablegen
def Foo ...
def NAME # Foo ...   // Same thing — NAME is automatically prepended
```

---

## 2. `defm` — Instantiate a multiclass

### Basic Syntax

```tablegen
defm Name : MultiClass1<args>, MultiClass2<args>, RegularClass1, RegularClass2;
```

Rules:
- Multiclasses must come **before** regular classes in the list
- There must be at least one multiclass
- Regular classes add their fields to **all** records created by the multiclasses
- `defm` has **no body** (no `{ }`)

### `defm` with Regular Classes

```tablegen
class XD { bits<4> Prefix = 11; }
class XS { bits<4> Prefix = 12; }

multiclass R {
  def rr : I<4>;
  def rm : I<2>;
}

multiclass Y {
  defm SS : R, XD;   // R is multiclass, XD is regular class
  defm SD : R, XS;
}

defm Instr : Y;
```

This creates 4 records:
- `InstrSSrr` — has fields from `I<4>` and `XD` (Prefix=11)
- `InstrSSrm` — has fields from `I<2>` and `XD` (Prefix=11)
- `InstrSDrr` — has fields from `I<4>` and `XS` (Prefix=12)
- `InstrSDrm` — has fields from `I<2>` and `XS` (Prefix=12)

### Anonymous `defm`

```tablegen
defm    : SomeMultiClass<...>;   // Globally unique name (not empty string)
defm "" : SomeMultiClass<...>;   // Empty string name
```

These are different! The first gets a unique generated name; the second gets an empty name.

### `defm` Inside a `multiclass`

When `defm` appears inside a multiclass body, `NAME` is automatically prepended if not already present:

```tablegen
multiclass basic_s<bits<4> opc> {
  defm SS : basic_r<opc>;        // Becomes: defm NAME # SS : basic_r<opc>
  defm SD : basic_r<opc>;        // Becomes: defm NAME # SD : basic_r<opc>
  def X : Instruction<opc, "x">; // Becomes: def NAME # X : ...
}

defm ADD : basic_s<0xf>;
// Creates: ADDSSrr, ADDSSrm, ADDSDrr, ADDSDrm, ADDX
```

---

## 3. Advanced `multiclass` Patterns

### Nested multiclasses

```tablegen
class Instruction<bits<4> opc, string Name> {
  bits<4> opcode = opc;
  string name = Name;
}

multiclass basic_r<bits<4> opc> {
  def rr : Instruction<opc, "rr">;
  def rm : Instruction<opc, "rm">;
}

multiclass basic_s<bits<4> opc> {
  defm SS : basic_r<opc>;   // Expands basic_r inside basic_s
  defm SD : basic_r<opc>;
  def X : Instruction<opc, "x">;
}

multiclass basic_p<bits<4> opc> {
  defm PS : basic_r<opc>;
  defm PD : basic_r<opc>;
  def Y : Instruction<opc, "y">;
}

defm ADD : basic_s<0xf>, basic_p<0xf>;
```

This creates 10 records:
```
ADDSSrr, ADDSSrm   (from basic_s → basic_r)
ADDSDrr, ADDSDrm   (from basic_s → basic_r)
ADDX               (from basic_s)
ADDPSrr, ADDPSrm   (from basic_p → basic_r)
ADDPDrr, ADDPDrm   (from basic_p → basic_r)
ADDY               (from basic_p)
```

### `let` Inside `multiclass`

```tablegen
multiclass basic_r<bits<4> opc> {
  let Predicates = [HasSSE2] in {
    def rr : Instruction<opc, "rr">;
    def rm : Instruction<opc, "rm">;
  }
  let Predicates = [HasSSE3] in
    def rx : Instruction<opc, "rx">;
}
```

### `multiclass` Inheriting from Another `multiclass`

```tablegen
multiclass Base<int x> {
  def _a : SomeClass<x>;
}

multiclass Extended<int x> : Base<x> {
  // Inherits def _a from Base
  def _b : OtherClass<x>;
}

defm FOO : Extended<5>;
// Creates: FOO_a (from Base) and FOO_b (from Extended)
```

---

## 4. `foreach` — Loop Over a Range

### Basic Syntax

```tablegen
foreach variable = range in {
  // statements using variable
}

// Or single statement:
foreach variable = range in statement;
```

### Range Forms

```tablegen
// Integer range:
foreach i = 0...7 in { ... }       // i = 0, 1, 2, 3, 4, 5, 6, 7

// List of values:
foreach i = [0, 1, 4, 8] in { ... }

// Using !range bang operator:
foreach i = !range(8) in { ... }    // i = 0, 1, 2, 3, 4, 5, 6, 7
```

### Generating Registers

```tablegen
foreach i = 0...30 in {
  def X#i : Register<i> {
    string Name = "x" # i;
    bits<5> Encoding = i;
  }
}
// Creates X0, X1, X2, ..., X30
```

### Generating Instruction Variants

```tablegen
foreach sz = ["b", "h", "w", "x"] in {
  def LD#sz : LoadInst<sz>;
  def ST#sz : StoreInst<sz>;
}
// Creates: LDb, LDh, LDw, LDx, STb, STh, STw, STx
```

### `foreach` Inside a `multiclass`

```tablegen
multiclass LoadStoreGroup<string base> {
  foreach sz = ["8", "16", "32", "64"] in {
    def base # sz : LoadInst<sz>;
  }
}
```

### Scope Rules

Variables defined with `defvar` inside `foreach` are **local to each iteration**:

```tablegen
foreach i = 0...3 in {
  defvar name = "reg" # i;   // Fresh each iteration
  def R#i : Register<i> {
    string Name = name;
  }
}
```

> ⚠️ You **cannot** accumulate values across iterations. `defvar i = !add(i, 1)` does NOT work.

---

## 5. Directed Acyclic Graphs (DAGs)

DAGs are one of the most important concepts in LLVM TableGen, used to represent instruction operands and code patterns.

### DAG Syntax

```tablegen
(operator arg1, arg2, arg3)
(operator arg1:$name1, arg2:$name2)   // Arguments with names
```

- The **operator** must be a record (a `def`)
- **Arguments** can be any TableGen value, including other DAGs
- **Names** (prefixed with `$`) tag arguments for pattern matching

### Operand Lists

In instruction definitions, DAGs describe what operands an instruction takes:

```tablegen
// (outs ...) — output operands (results)
// (ins ...)  — input operands

dag OutOperandList = (outs GPR64:$dst);
dag InOperandList  = (ins GPR64:$src1, GPR64:$src2);
```

Here:
- `outs` and `ins` are records (operators)
- `GPR64` is a register class record
- `$dst`, `$src1`, `$src2` are names for pattern matching

### Selection Patterns

DAGs also describe how instructions map to IR operations:

```tablegen
list<dag> Pattern = [(set GPR64:$dst, (add GPR64:$src1, GPR64:$src2))];
```

This says: "This instruction implements `(add src1, src2)` and puts the result in `dst`."

### Nested DAGs

DAGs can be nested to represent complex patterns:

```tablegen
// Match: dst = (add src1, (shl src2, amount))
list<dag> Pattern = [(set GPR64:$dst,
                      (add GPR64:$src1,
                           (shl GPR64:$src2, i64imm:$amount)))];
```

### DAG Argument Formats

| Format | Meaning |
|--------|---------|
| `value` | Argument with a value, no name |
| `value:$name` | Argument with a value and a name |
| `$name` | Argument with a name but uninitialized value |

```tablegen
(ins GPR64:$src1, GPR64:$src2)   // Both have values and names
(outs GPR64:$dst)                 // Has value and name
(ops $unnamed)                    // Name only, no value
```

### Bang Operators for DAGs

```tablegen
// Get the operator of a DAG:
def op = !getdagop(myDag);

// Get an argument by index:
def arg0 = !getdagarg<GPR64>(myDag, 0);

// Get an argument by name:
def argDst = !getdagarg<GPR64>(myDag, "dst");

// Get an argument's name:
string name0 = !getdagname(myDag, 0);

// Set an argument:
dag newDag = !setdagarg(myDag, 0, newValue);

// Set the operator:
dag newDag = !setdagop(myDag, newOp);

// Concatenate two DAGs (operators must match):
dag combined = !con(dag1, dag2);

// Create a DAG from parts:
dag d = !dag(op, [arg1, arg2], ["name1", "name2"]);

// Get size (number of arguments, not counting operator):
int n = !size(myDag);
```

---

## 6. Bang Operators — TableGen's Built-in Functions

Bang operators are TableGen's way of doing computation. They all start with `!`.

### Arithmetic

```tablegen
!add(a, b, ...)    // Addition:        !add(3, 4) = 7
!sub(a, b)         // Subtraction:     !sub(10, 3) = 7
!mul(a, b, ...)    // Multiplication:  !mul(3, 4) = 12
!div(a, b)         // Division:        !div(10, 3) = 3 (signed, truncated)
!shl(a, n)         // Left shift:      !shl(1, 3) = 8
!sra(a, n)         // Arithmetic right shift
!srl(a, n)         // Logical right shift
!and(a, b, ...)    // Bitwise AND
!or(a, b, ...)     // Bitwise OR
!xor(a, b, ...)    // Bitwise XOR
!not(a)            // Logical NOT:     !not(0) = 1, !not(1) = 0
!logtwo(a)         // Floor log base 2: !logtwo(8) = 3
```

### Comparison

```tablegen
!eq(a, b)    // Equal:              !eq(3, 3) = 1
!ne(a, b)    // Not equal:          !ne(3, 4) = 1
!lt(a, b)    // Less than:          !lt(2, 5) = 1
!le(a, b)    // Less than or equal: !le(3, 3) = 1
!gt(a, b)    // Greater than:       !gt(5, 2) = 1
!ge(a, b)    // Greater or equal:   !ge(3, 3) = 1
```

Works on `bit`, `bits`, `int`, and `string` values.

### Conditional

```tablegen
// if-then-else:
!if(condition, thenValue, elseValue)

// Example:
int abs_x = !if(!lt(x, 0), !sub(0, x), x);

// Multi-way conditional (like switch):
!cond(cond1: val1, cond2: val2, ..., true: default)

// Example:
string sign = !cond(!lt(x, 0): "negative",
                    !eq(x, 0): "zero",
                    true:       "positive");

// Switch (compact form of !cond with !eq):
!switch(key, case1: val1, case2: val2, default)

// Example:
string regName = !switch(size, 1: "byte", 2: "half", 4: "word", "unknown");
```

### String Operations

```tablegen
!strconcat(s1, s2, ...)    // Concatenate strings
!substr(s, start)           // Substring from start to end
!substr(s, start, len)      // Substring of given length
!find(s1, s2)               // Find s2 in s1, returns position or -1
!find(s1, s2, start)        // Find starting from position
!size(s)                    // Length of string
!empty(s)                   // 1 if empty, 0 otherwise
!tolower(s)                 // Convert to lowercase
!toupper(s)                 // Convert to uppercase
!subst(target, repl, value) // Replace target with repl in value
!repr(value)                // Debug: represent value as string
!match(str, regex)          // 1 if str matches ERE regex
!interleave(list, delim)    // Join list elements with delimiter
```

```tablegen
// Examples:
string s = !strconcat("Hello", ", ", "World");   // "Hello, World"
string s = !substr("Hello World", 6);             // "World"
string s = !substr("Hello World", 0, 5);          // "Hello"
int pos = !find("Hello", "ll");                   // 2
int len = !size("Hello");                         // 5
string up = !toupper("hello");                    // "HELLO"
string joined = !interleave(["a","b","c"], ", "); // "a, b, c"
```

### List Operations

```tablegen
!listconcat(l1, l2, ...)    // Concatenate lists
!listflatten(l)             // Flatten list<list<T>> to list<T>
!listremove(l1, l2)         // Remove l2 elements from l1
!listsplat(val, count)      // Create list of count copies of val
!size(l)                    // Number of elements
!empty(l)                   // 1 if empty
!head(l)                    // First element
!tail(l)                    // All elements except first
!range(n)                   // [0, 1, ..., n-1]
!range(start, end)          // [start, ..., end-1]
!range(start, end, step)    // With step
!range(list)                // [0, 1, ..., size(list)-1]
```

```tablegen
// Examples:
list<int> a = [1, 2, 3];
list<int> b = [4, 5, 6];
list<int> c = !listconcat(a, b);          // [1, 2, 3, 4, 5, 6]
list<int> d = !listsplat(0, 4);           // [0, 0, 0, 0]
int first = !head(a);                     // 1
list<int> rest = !tail(a);                // [2, 3]
list<int> r = !range(5);                  // [0, 1, 2, 3, 4]
list<int> r2 = !range(2, 5);             // [2, 3, 4]
list<int> r3 = !range(0, 10, 2);         // [0, 2, 4, 6, 8]
```

### Functional Operations on Lists

```tablegen
// Map: apply expression to each element
!foreach(var, list, expr)

// Example: double each element
list<int> doubled = !foreach(x, [1, 2, 3], !mul(x, 2));
// = [2, 4, 6]

// Filter: keep elements where predicate is true
!filter(var, list, predicate)

// Example: keep only even numbers
list<int> evens = !filter(x, [1, 2, 3, 4, 5, 6], !eq(!and(x, 1), 0));
// = [2, 4, 6]

// Fold: reduce list to single value
!foldl(init, list, acc, var, expr)

// Example: sum all elements
int total = !foldl(0, [1, 2, 3, 4], sum, x, !add(sum, x));
// = 10

// Sort: sort list by key
!sort(var, list, key)

// Example: sort records by Name field
list<Thing> sorted = !sort(t, Things, t.Name);
```

### Type and Record Operations

```tablegen
// Cast a value to another type:
!cast<TargetType>(value)

// If value is a string, look up the record with that name:
Instruction inst = !cast<Instruction>("ADD32rr");

// Check if a record of given type with given name exists:
!exists<ClassName>("RecordName")   // Returns 1 or 0

// Check if value is a subtype of given type:
!isa<ClassName>(value)   // Returns 1 or 0

// Check if value is initialized (not ?):
!initialized(value)   // Returns 1 or 0

// Get all instances of a class:
!instances<ClassName>()
!instances<ClassName>("regex")   // Filtered by name regex
```

```tablegen
// Practical cast example in multiclass:
multiclass LoadPat<string suffix, ValueType vt> {
  def : Pat<(vt (load addr:$ptr)),
            (!cast<Instruction>("LD" # suffix # "_reg") addr:$ptr)>;
}
// !cast<Instruction>("LDB_reg") looks up the record named "LDB_reg"
// This is extremely common in AArch64 .td files!
```

---

## 7. `deftype` — Type Aliases

`deftype` creates an alias for a type:

```tablegen
deftype MyInt = int;
deftype RegList = list<Register>;

// Now use the alias:
MyInt x = 42;
RegList regs = [X0, X1, X2];
```

Currently only primitive types and type aliases are supported. `deftype` can only appear at the top level.

---

## 8. `dump` — Debug Printing

`dump` prints a message to stderr. Useful for debugging multiclass instantiations:

```tablegen
multiclass MC<dag s> {
  dump "Instantiating MC with s = " # !repr(s);
  def : SomeClass<s>;
}
```

- At top level: printed immediately
- Inside a record/class/multiclass: printed at each instantiation point

---

## 9. Using Classes as Subroutines

A powerful pattern: use a class to compute a value, then retrieve the result field.

### Pattern

```tablegen
class ComputeSomething<int input> {
  // Do computation using bang operators
  defvar intermediate = !mul(input, 2);
  int ret = !add(intermediate, 1);   // The "return value"
}

def MyRecord {
  int Value = ComputeSomething<5>.ret;   // = 11
}
```

### Real Example: Validating a Size

```tablegen
class isValidSize<int size> {
  bit ret = !cond(!eq(size,  1): 1,
                  !eq(size,  2): 1,
                  !eq(size,  4): 1,
                  !eq(size,  8): 1,
                  !eq(size, 16): 1,
                  true: 0);
}

class DataField<int sz> {
  int Size = sz;
  bit ValidSize = isValidSize<sz>.ret;
}

def Field4 : DataField<4>;    // ValidSize = 1
def Field3 : DataField<3>;    // ValidSize = 0
```

### Chaining Subroutines

```tablegen
class Step1<int x> {
  int ret = !mul(x, 2);
}

class Step2<int x> {
  int ret = !add(Step1<x>.ret, 10);
}

def R {
  int Result = Step2<5>.ret;   // Step1<5>.ret = 10, Step2<5>.ret = 20
}
```

---

## 10. The Paste Operator `#` — Deep Dive

The `#` operator has some subtle behaviors worth understanding.

### In Record Names

When used in a `def` or `defm` name, **global variable names on the right side are treated as literal strings**:

```tablegen
defvar suffix = "_suffstring";

def name # suffix { }
// Creates record named "namesuffix" (literal "suffix", not the variable value!)

foreach i = [1, 2] in {
  def rec # i { }
  // Creates "rec1" and "rec2" (i IS evaluated because it's a foreach variable)
}
```

### In Value Expressions

In regular value expressions, the **right-hand side global name is also literal**, but the left-hand side is evaluated normally:

```tablegen
defvar suffix = "_suffstring";
defvar nums = [0, 1, 2, 3];

def test {
  string s = suffix # suffix;
  // Left "suffix" is evaluated → "_suffstring"
  // Right "suffix" is literal → "suffix"
  // Result: "_suffstringsuffix"

  list<int> n = nums # [4, 5, 6];
  // Left "nums" is evaluated → [0, 1, 2, 3]
  // Right is a literal list → [4, 5, 6]
  // Result: [0, 1, 2, 3, 4, 5, 6]
}
```

### Trailing Paste

```tablegen
def name # { }   // Concatenates name with empty string
```

### Common Usage in AArch64

```tablegen
// Building instruction names dynamically:
multiclass LoadStore<string sz> {
  def LD#sz : LoadInst<sz>;    // LDB, LDH, LDW, LDX
  def ST#sz : StoreInst<sz>;   // STB, STH, STW, STX
}

// Building pattern names with !cast:
def : Pat<(i32 (load addr:$ptr)),
          (!cast<Instruction>("LDR" # "W" # "_reg") addr:$ptr)>;
```

---

## 11. Putting It All Together — A Complete Example

Here's a realistic example showing how all these features combine, similar to what you'd see in an AArch64 `.td` file:

```tablegen
// ============================================================
// Base classes
// ============================================================

class Instruction<bits<8> opc, string asm> {
  bits<8> Opcode   = opc;
  string  AsmStr   = asm;
  bit     isReturn = 0;
  bit     isCall   = 0;
  dag     OutOps   = (outs);
  dag     InOps    = (ins);
  list<dag> Pattern = [];
}

class ArithInst<bits<8> opc, string asm, dag outs, dag ins, list<dag> pat>
    : Instruction<opc, asm> {
  let OutOps   = outs;
  let InOps    = ins;
  let Pattern  = pat;
}

// ============================================================
// Register classes
// ============================================================

class Register<int num, string name> {
  int    Number = num;
  string Name   = name;
}

foreach i = 0...7 in
  def R#i : Register<i, "r" # i>;

// ============================================================
// Multiclass for register/immediate variants
// ============================================================

multiclass BinOp<bits<8> opc, string mnemonic, SDNode node> {
  // Register-Register variant
  def _rr : ArithInst<opc, mnemonic # " $dst, $src1, $src2",
                      (outs GPR:$dst),
                      (ins GPR:$src1, GPR:$src2),
                      [(set GPR:$dst, (node GPR:$src1, GPR:$src2))]>;

  // Register-Immediate variant
  def _ri : ArithInst<!add(opc, 1), mnemonic # " $dst, $src1, #$imm",
                      (outs GPR:$dst),
                      (ins GPR:$src1, i32imm:$imm),
                      [(set GPR:$dst, (node GPR:$src1, imm:$imm))]>;
}

// ============================================================
// Instantiate instructions
// ============================================================

let isCall = 0 in {
  defm ADD : BinOp<0x10, "add", add>;
  defm SUB : BinOp<0x20, "sub", sub>;
  defm AND : BinOp<0x30, "and", and>;
  defm ORR : BinOp<0x40, "orr", or>;
}

// ============================================================
// Validation
// ============================================================

class ValidateOpcode<bits<8> opc> {
  assert !lt(opc, 0xFF), "Opcode must be < 255, got: " # !repr(opc);
  bits<8> Value = opc;
}
```

This produces records: `ADD_rr`, `ADD_ri`, `SUB_rr`, `SUB_ri`, `AND_rr`, `AND_ri`, `ORR_rr`, `ORR_ri`.

---

## 12. Common Patterns and Idioms

### Pattern 1: Enum-like Records

```tablegen
class FPFormat<bits<3> val> {
  bits<3> Value = val;
}

def NotFP      : FPFormat<0>;
def ZeroArgFP  : FPFormat<1>;
def OneArgFP   : FPFormat<2>;
def TwoArgFP   : FPFormat<3>;
```

### Pattern 2: Bit-field Decomposition

```tablegen
class ModRefVal<bits<2> val> {
  bits<2> Value = val;
}

class ModRefBits<ModRefVal mrv> {
  bit isMod = mrv.Value{0};   // Extract bit 0
  bit isRef = mrv.Value{1};   // Extract bit 1
}

def None   : ModRefVal<0b00>;
def Mod    : ModRefVal<0b01>;
def Ref    : ModRefVal<0b10>;
def ModRef : ModRefVal<0b11>;

def foo : ModRefBits<Mod>;    // isMod=1, isRef=0
def bar : ModRefBits<Ref>;    // isMod=0, isRef=1
```

### Pattern 3: Dynamic Record Lookup with `!cast`

```tablegen
// Very common in AArch64 for building instruction names:
multiclass SignedLoad<string T, string Rm> {
  def : Pat<(i32 (!cast<SDNode>("sextload" # T) addr:$ptr)),
            (!cast<Instruction>("LDRS" # T # "w_" # Rm) addr:$ptr)>;
}
```

### Pattern 4: Collecting Records with `defset`

```tablegen
defset list<Instruction> AllInstructions = {
  defm ADD : BinOp<0x10, "add", add>;
  defm SUB : BinOp<0x20, "sub", sub>;
}
// AllInstructions = [ADD_rr, ADD_ri, SUB_rr, SUB_ri]
```

### Pattern 5: Conditional Features with `#ifdef`

```tablegen
#ifdef HAVE_NEON
  include "ARMInstrNEON.td"
#endif
```

### Pattern 6: Computing Values with `!foldl`

```tablegen
// Sum the sizes of all fields:
class FieldGroup<list<int> sizes> {
  int TotalSize = !foldl(0, sizes, acc, sz, !add(acc, sz));
}

def MyGroup : FieldGroup<[8, 16, 32]>;
// MyGroup.TotalSize = 56
```

### Pattern 7: Building Strings with `!foreach` and `!interleave`

```tablegen
// Create a comma-separated list of register names:
list<string> names = !foreach(r, [X0, X1, X2], r.Name);
// names = ["x0", "x1", "x2"]

string nameList = !interleave(names, ", ");
// nameList = "x0, x1, x2"
```

### Pattern 8: Guarded Instruction Sets

```tablegen
// Apply predicates to a group of instructions:
let Predicates = [HasNEON] in {
  defm VADD : SIMDInst<"vadd">;
  defm VMUL : SIMDInst<"vmul">;
}

let Predicates = [HasSVE] in {
  defm FADD : SVEInst<"fadd">;
}
```

---

## Bang Operator Quick Reference

| Operator | Purpose | Example |
|----------|---------|---------|
| `!add` | Add | `!add(3, 4)` → `7` |
| `!sub` | Subtract | `!sub(10, 3)` → `7` |
| `!mul` | Multiply | `!mul(3, 4)` → `12` |
| `!div` | Divide | `!div(10, 3)` → `3` |
| `!and` | Bitwise AND | `!and(0b1100, 0b1010)` → `0b1000` |
| `!or` | Bitwise OR | `!or(0b1100, 0b1010)` → `0b1110` |
| `!xor` | Bitwise XOR | `!xor(0b1100, 0b1010)` → `0b0110` |
| `!not` | Logical NOT | `!not(0)` → `1` |
| `!shl` | Left shift | `!shl(1, 3)` → `8` |
| `!sra` | Arith right shift | `!sra(-8, 1)` → `-4` |
| `!srl` | Logic right shift | `!srl(8, 1)` → `4` |
| `!eq` | Equal | `!eq(3, 3)` → `1` |
| `!ne` | Not equal | `!ne(3, 4)` → `1` |
| `!lt` | Less than | `!lt(2, 5)` → `1` |
| `!le` | Less or equal | `!le(3, 3)` → `1` |
| `!gt` | Greater than | `!gt(5, 2)` → `1` |
| `!ge` | Greater or equal | `!ge(3, 3)` → `1` |
| `!if` | Conditional | `!if(cond, a, b)` |
| `!cond` | Multi-way cond | `!cond(c1:v1, c2:v2, true:vd)` |
| `!switch` | Switch | `!switch(k, c1:v1, default)` |
| `!cast` | Type cast / lookup | `!cast<T>("name")` |
| `!isa` | Type check | `!isa<T>(val)` → `0` or `1` |
| `!exists` | Record exists? | `!exists<T>("name")` → `0` or `1` |
| `!initialized` | Is initialized? | `!initialized(val)` → `0` or `1` |
| `!instances` | All instances of class | `!instances<T>()` |
| `!strconcat` | Concat strings | `!strconcat("a", "b")` → `"ab"` |
| `!substr` | Substring | `!substr("hello", 1, 3)` → `"ell"` |
| `!find` | Find in string | `!find("hello", "ll")` → `2` |
| `!size` | Size of string/list/dag | `!size([1,2,3])` → `3` |
| `!empty` | Is empty? | `!empty([])` → `1` |
| `!tolower` | To lowercase | `!tolower("ABC")` → `"abc"` |
| `!toupper` | To uppercase | `!toupper("abc")` → `"ABC"` |
| `!subst` | Substitute | `!subst("x", "y", "fox")` → `"foy"` |
| `!match` | Regex match | `!match("foo", "f.*")` → `1` |
| `!interleave` | Join with delimiter | `!interleave(["a","b"], ",")` → `"a,b"` |
| `!repr` | Debug repr | `!repr(42)` → `"42"` |
| `!listconcat` | Concat lists | `!listconcat([1],[2])` → `[1,2]` |
| `!listflatten` | Flatten list | `!listflatten([[1,2],[3]])` → `[1,2,3]` |
| `!listremove` | Remove elements | `!listremove([1,2,3],[2])` → `[1,3]` |
| `!listsplat` | Repeat value | `!listsplat(0, 3)` → `[0,0,0]` |
| `!head` | First element | `!head([1,2,3])` → `1` |
| `!tail` | All but first | `!tail([1,2,3])` → `[2,3]` |
| `!range` | Integer range | `!range(3)` → `[0,1,2]` |
| `!foreach` | Map over list/dag | `!foreach(x, [1,2], !mul(x,2))` → `[2,4]` |
| `!filter` | Filter list | `!filter(x, [1,2,3], !gt(x,1))` → `[2,3]` |
| `!foldl` | Fold/reduce | `!foldl(0, [1,2,3], a, x, !add(a,x))` → `6` |
| `!sort` | Sort list | `!sort(x, list, x.Name)` |
| `!logtwo` | Log base 2 | `!logtwo(8)` → `3` |
| `!con` | Concat DAGs | `!con(dag1, dag2)` |
| `!dag` | Create DAG | `!dag(op, [a1,a2], ["n1","n2"])` |
| `!getdagop` | Get DAG operator | `!getdagop((foo 1, 2))` → `foo` |
| `!getdagarg` | Get DAG argument | `!getdagarg<T>(dag, 0)` |
| `!getdagname` | Get arg name | `!getdagname(dag, 0)` |
| `!setdagop` | Set DAG operator | `!setdagop(dag, newOp)` |
| `!setdagarg` | Set DAG argument | `!setdagarg(dag, 0, newVal)` |
| `!setdagname` | Set arg name | `!setdagname(dag, 0, "newName")` |

---

## What's Next?

In **Part 3**, we'll cover:
- The TableGen **backend system** — how `.inc` files are generated
- **AArch64-specific patterns** — how to read and understand real AArch64 `.td` files
- Key AArch64 classes: `Instruction`, `Register`, `RegisterClass`, `SubtargetFeature`, `Pat`, `Predicate`
- How instruction encoding works in AArch64 TableGen
- How selection patterns connect TableGen to the LLVM code generator
- Tips for navigating the AArch64 TableGen file hierarchy

---

*Part 2 of 3 — TableGen Beginner's Guide*
