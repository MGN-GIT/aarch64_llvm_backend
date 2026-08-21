# 📘 TableGen Beginner's Guide — Part 1: Core Concepts & Syntax

> **Series Overview**
> - **Part 1 (this file):** Core concepts, syntax, types, values, and basic statements
> - **Part 2:** Advanced features — multiclass, defm, foreach, bang operators, DAGs
> - **Part 3:** Backends, AArch64 `.td` file patterns, and real-world usage guide

---

## Table of Contents

1. [What is TableGen and Why Does It Exist?](#1-what-is-tablegen-and-why-does-it-exist)
2. [How TableGen Works — The Big Picture](#2-how-tablegen-works--the-big-picture)
3. [Running TableGen](#3-running-tablegen)
4. [File Structure and Lexical Rules](#4-file-structure-and-lexical-rules)
5. [Types in TableGen](#5-types-in-tablegen)
6. [Values and Expressions](#6-values-and-expressions)
7. [The `class` Statement](#7-the-class-statement)
8. [The `def` Statement](#8-the-def-statement)
9. [The `let` Statement](#9-the-let-statement)
10. [The `defvar` Statement](#10-the-defvar-statement)
11. [The `defset` Statement](#11-the-defset-statement)
12. [The `if` Statement](#12-the-if-statement)
13. [The `assert` Statement](#13-the-assert-statement)
14. [Include Files and Preprocessor](#14-include-files-and-preprocessor)
15. [How Records Are Built — Step by Step](#15-how-records-are-built--step-by-step)
16. [Quick Reference Cheat Sheet](#16-quick-reference-cheat-sheet)

---

## 1. What is TableGen and Why Does It Exist?

### The Problem TableGen Solves

Imagine you are writing a compiler backend for a CPU architecture like AArch64. That CPU has **thousands of instructions**. For each instruction, you need to describe:

- Its name and opcode
- What operands it takes (registers, immediates, memory addresses)
- How it is encoded in binary
- What assembly syntax it uses
- What it does semantically (for the optimizer)
- Whether it reads/writes flags, memory, etc.

Writing all of this by hand in C++ would mean thousands of lines of repetitive, error-prone code. If you need to change one property shared by 500 instructions, you'd have to edit 500 places.

**TableGen solves this** by letting you write a compact, declarative description of all this information in `.td` files. TableGen then generates the C++ code automatically.

### What TableGen Actually Is

TableGen is a **domain-specific language (DSL)** and tool that:

1. **Reads** `.td` source files containing class and record definitions
2. **Instantiates** all the records (resolving inheritance, template arguments, etc.)
3. **Passes** the resulting records to a **backend** that generates output files (usually `.inc` C++ files)

```
  YourTarget.td  ──►  llvm-tblgen  ──►  Backend  ──►  TargetGenInstrInfo.inc
                       (parser)                         TargetGenRegisterInfo.inc
                                                        TargetGenAsmMatcher.inc
                                                        ... etc.
```

### Real Example: AArch64

In the LLVM AArch64 backend, a single `defm` line like:

```tablegen
defm ADD : ArithLogicReg<0b0001011, "add", add, 1>;
```

...expands into dozens of concrete records covering all variants of the ADD instruction (32-bit, 64-bit, with/without shift, etc.), each with all their fields properly filled in. Without TableGen, each variant would need to be written out manually.

---

## 2. How TableGen Works — The Big Picture

### Two Key Concepts: Classes and Records

TableGen has two primary building blocks:

| Concept | Keyword | What it is |
|---------|---------|------------|
| **Class** | `class` | An *abstract* template — like a blueprint. Cannot be used directly. |
| **Record** (def) | `def` | A *concrete* instance — the actual data. |

Think of it like this:
- A `class` is like a C++ class definition
- A `def` is like creating an object (instance) of that class

```tablegen
// Abstract class — defines the shape
class Register<int num> {
  int Number = num;
  string Name = ?;   // ? means "uninitialized, must be set later"
}

// Concrete record — fills in the data
def X0 : Register<0> {
  string Name = "x0";
}

def X1 : Register<1> {
  string Name = "x1";
}
```

After TableGen processes this, `X0` and `X1` are fully-specified records with all fields filled in.

### The Flow of Information

```
.td files  →  TableGen Parser  →  In-memory Records  →  Backend  →  .inc files
```

The backend is what gives meaning to the records. TableGen itself doesn't know what "Register" or "Instruction" means — that's up to the backend (e.g., the RegisterInfo backend knows to look for records derived from the `Register` class).

---

## 3. Running TableGen

### Basic Usage

```bash
# Parse a file and print all records (default behavior)
llvm-tblgen MyTarget.td

# Use a specific backend
llvm-tblgen MyTarget.td -gen-register-info -o MyTargetGenRegisterInfo.inc

# Print all records as JSON
llvm-tblgen MyTarget.td -dump-json

# Print all definitions of a specific class
llvm-tblgen X86.td -print-enums -class=Register

# Define a preprocessor macro
llvm-tblgen MyTarget.td -Dmacro1
```

### Useful Debugging Options

```bash
# Print all classes and records (default)
llvm-tblgen MyTarget.td --print-records

# Print detailed records with source locations
llvm-tblgen MyTarget.td --print-detailed-records

# Time each phase
llvm-tblgen MyTarget.td --time-phases
```

### What the Output Looks Like

When you run `llvm-tblgen` on a file with `def ADD32rr`, you get a fully expanded record showing every field and its value — even fields inherited from parent classes many levels up. This is very useful for debugging.

---

## 4. File Structure and Lexical Rules

### File Extension

TableGen files use the `.td` extension.

### Comments

```tablegen
// This is a single-line comment (BCPL style)

/* This is a
   multi-line comment (C style) */

/* Nested /* comments */ are supported */
```

### Numeric Literals

```tablegen
42          // Decimal integer
-7          // Negative decimal (sign is part of the literal!)
0xFF        // Hexadecimal
0b1010      // Binary
```

> ⚠️ **Important:** The `-` sign is *part of* the integer literal in TableGen, not a unary operator. So `1-5` is parsed as two tokens: `1` and `-5`.

### String Literals

```tablegen
"Hello, World!"          // Regular string

[{
  This is a code literal.
  It can span multiple lines.
  Line breaks are preserved.
}]                       // Code literal (also a string, just multi-line)
```

Adjacent string literals are concatenated (like in C):
```tablegen
"Hello, " "World!"   // Same as "Hello, World!"
```

### Identifiers

```tablegen
MyClass       // Valid identifier
_helper       // Valid (underscore allowed)
2ndPass       // Valid! TableGen allows leading digits
```

### Reserved Keywords

These cannot be used as identifiers:

```
assert    bit       bits      class     code
dag       def       dump      else      false
foreach   defm      defset    defvar    field
if        in        include   int       let
list      multiclass string   then      true
```

> ⚠️ `field` is deprecated except in the CodeEmitterGen backend.

---

## 5. Types in TableGen

TableGen is **statically typed**. Every value must have a type. Here are all the available types:

### Primitive Types

| Type | Description | Example |
|------|-------------|---------|
| `bit` | A single boolean bit: 0 or 1 | `bit isReturn = 0;` |
| `int` | A 64-bit signed integer | `int Opcode = 42;` |
| `string` | A sequence of characters | `string Name = "add";` |
| `code` | Alias for `string` (for code snippets) | `code Body = [{ return x; }];` |

### Composite Types

#### `bits<n>` — Fixed-width bit sequence

```tablegen
bits<8>  Opcode = 0b00000001;   // 8-bit value
bits<32> Encoding;               // 32-bit value, uninitialized

// Individual bits can be accessed:
bit bit7 = Opcode{7};           // Get bit 7
bits<4> nibble = Opcode{7...4}; // Get bits 7 down to 4
```

This is extremely common in AArch64 `.td` files for encoding instruction fields.

#### `list<T>` — A list of elements

```tablegen
list<int>    Costs = [1, 2, 3];
list<string> Names = ["x0", "x1", "x2"];
list<Register> Uses = [X0, X1];   // List of records!

// Access by index (0-based):
int first = Costs[0];   // = 1

// List slice:
list<int> sub = Costs[0...1];  // = [1, 2]
```

#### `dag` — Directed Acyclic Graph

```tablegen
// Syntax: (operator arg1:$name1, arg2:$name2, ...)
dag OutOps = (outs GPR64:$dst);
dag InOps  = (ins GPR64:$src1, GPR64:$src2);
dag Pattern = [(set GPR64:$dst, (add GPR64:$src1, GPR64:$src2))];
```

DAGs are used extensively to represent instruction operands and selection patterns. We'll cover them in depth in Part 2.

#### `ClassID` — A record of a specific class type

```tablegen
class Instruction { ... }

// A field that must hold a record derived from Instruction:
Instruction MyInstr = ADD32rr;

// A list of records derived from Register:
list<Register> Defs = [EFLAGS, RSP];
```

### Type Conversion

TableGen performs implicit conversions where possible. For example, you can assign an integer `7` to a `bits<4>` field — TableGen will convert it automatically.

---

## 6. Values and Expressions

### Literal Values

```tablegen
42          // Integer
"hello"     // String
true        // Same as integer 1
false       // Same as integer 0
?           // Uninitialized value
```

### Bit Sequences (using braces)

```tablegen
bits<4> val = {1, 0, 1, 1};   // Binary 1011 = 11
```

### List Initializers (using brackets)

```tablegen
list<int> nums = [1, 2, 3, 4];
list<int> empty = []<int>;     // Empty list — type annotation needed
```

### Record References

```tablegen
def MyReg : Register<5>;

// Use the record as a value:
Register r = MyReg;
list<Register> regs = [MyReg, OtherReg];
```

### Suffixed Values — Accessing Sub-parts

```tablegen
// Accessing bits of an integer or bits<n>:
bits<8> opcode = 0b10110001;
bit     msb    = opcode{7};        // Bit 7 (most significant)
bits<4> upper  = opcode{7...4};    // Bits 7 down to 4 = 0b1011
bits<4> lower  = opcode{3...0};    // Bits 3 down to 0 = 0b0001

// Accessing list elements:
list<int> nums = [10, 20, 30, 40];
int  first = nums[0];              // = 10
list<int> slice = nums[1...2];     // = [20, 30]

// Accessing a field of a record:
def MyReg : Register<5>;
int regNum = MyReg.Number;         // Access the Number field
```

### The Paste Operator `#` — String/Name Concatenation

The `#` operator concatenates strings or lists. It's especially useful for building record names dynamically:

```tablegen
// Building record names:
foreach i = [0, 1, 2, 3] in {
  def R#i : Register<i>;   // Creates R0, R1, R2, R3
}

// Concatenating strings in values:
defvar prefix = "ADD";
string fullName = prefix # "_rr";   // = "ADD_rr"

// Concatenating lists:
list<int> a = [1, 2];
list<int> b = [3, 4];
list<int> c = a # b;   // = [1, 2, 3, 4]
```

> ⚠️ **Quirk:** When `#` is used in a record name, global variable names on the *right* side are treated as literal strings, not evaluated. So `def name # suffix` produces `namesuffix` (the literal text "suffix"), not the value of the `suffix` variable.

### Anonymous Records (Class Invocation as Value)

You can create an anonymous record inline by invoking a class:

```tablegen
class isValidSize<int size> {
  bit ret = !cond(!eq(size, 1): 1,
                  !eq(size, 2): 1,
                  !eq(size, 4): 1,
                  true: 0);
}

def Data {
  int Size = 4;
  bit ValidSize = isValidSize<4>.ret;   // Creates anonymous record, gets .ret field
}
```

This is a powerful "subroutine" pattern — use a class to compute a value and retrieve the result.

---

## 7. The `class` Statement

### Basic Syntax

```tablegen
class ClassName <TemplateArgs> : ParentClass1, ParentClass2 {
  // Field definitions
  Type fieldName = value;

  // Override inherited fields
  let inheritedField = newValue;
}
```

### Simple Class (No Template Args)

```tablegen
class Instruction {
  string Namespace = "MyTarget";
  bit isReturn     = 0;
  bit isBranch     = 0;
  bit isCall       = 0;
  dag OutOperandList = (outs);
  dag InOperandList  = (ins);
}
```

### Parameterized Class (With Template Args)

```tablegen
class FPFormat<bits<3> val> {
  bits<3> Value = val;   // val is the template argument
}

// Instantiate with different values:
def NotFP     : FPFormat<0>;
def ZeroArgFP : FPFormat<1>;
def OneArgFP  : FPFormat<2>;
```

### Template Arguments with Defaults

```tablegen
class InstBase<int opc, string asm, bit hasSideEffects = 0> {
  int     Opcode        = opc;
  string  AsmString     = asm;
  bit     HasSideEffects = hasSideEffects;
}

// Required args only (hasSideEffects defaults to 0):
def NOP : InstBase<0x00, "nop">;

// Override the default:
def SYSCALL : InstBase<0x0F05, "syscall", 1>;
```

### Class Inheritance

```tablegen
class Base {
  int X = 0;
  int Y = 0;
}

class Derived : Base {
  let X = 10;    // Override X from Base
  int Z = 20;    // Add new field Z
}

def MyRecord : Derived;
// MyRecord has: X=10, Y=0, Z=20
```

### Multiple Inheritance

```tablegen
class HasName {
  string Name = ?;
}

class HasOpcode {
  bits<8> Opcode = ?;
}

class Instruction<string n, bits<8> op> : HasName, HasOpcode {
  let Name   = n;
  let Opcode = op;
}

def ADD : Instruction<"add", 0x01>;
// ADD has: Name="add", Opcode=0x01
```

### The Implicit `NAME` Template Argument

Every class has an implicit template argument called `NAME` (uppercase). It is automatically bound to the name of the `def` or `defm` that inherits from the class:

```tablegen
class MyClass {
  string RecordName = NAME;   // Will be set to the def's name
}

def Foo : MyClass;
// Foo.RecordName = "Foo"

def Bar : MyClass;
// Bar.RecordName = "Bar"
```

### Using `let append` and `let prepend` in Class Bodies

```tablegen
class Base {
  list<int> items = [2, 3];
}

class Middle : Base {
  let append items = [4];    // items = [2, 3, 4]
}

def Concrete : Middle {
  let prepend items = [1];   // items = [1, 2, 3, 4]
}
```

### Using `defvar` Inside a Class Body

```tablegen
class Compute<int x> {
  defvar doubled = !mul(x, 2);   // Local variable, NOT a field
  int Result = doubled;           // Use the local variable
}

def R : Compute<5>;
// R.Result = 10
// R does NOT have a "doubled" field
```

---

## 8. The `def` Statement

### Basic Syntax

```tablegen
def RecordName : ParentClass1<args>, ParentClass2 {
  // Override or add fields
  let fieldName = value;
  Type newField = value;
}
```

### Simple Record

```tablegen
class C {
  bit V = true;
}

def X : C;           // Inherits V = true, no overrides

def Y : C {
  let V = false;     // Override V
  string Greeting = "Hello!";   // Add new field
}
```

### Record with Multiple Parent Classes

```tablegen
def ADD32rr : Instruction<0x01, MRMDestReg, (outs GR32:$dst),
                                             (ins GR32:$src1, GR32:$src2),
                          "add{l}\t{$src2, $dst|$dst, $src2}",
                          [(set GR32:$dst, (add GR32:$src1, GR32:$src2))]>;
```

### Anonymous Records

```tablegen
// No name given — TableGen generates a unique name
def : SomeClass<args>;
```

Anonymous records are common in AArch64 `.td` files for `Pat` (pattern) definitions:

```tablegen
// This creates an anonymous record for a selection pattern
def : Pat<(i64 (zextloadi8 addr:$ptr)), (LDRB addr:$ptr)>;
```

### Self-Referential Records

A record can reference itself in its own definition:

```tablegen
class A<dag d> {
  dag the_dag = d;
}

def rec1 : A<(ops rec1)>;   // rec1 appears in its own template argument
```

---

## 9. The `let` Statement

### Top-Level `let` — Apply to Multiple Records

The top-level `let` applies field overrides to all records defined within its scope:

```tablegen
// Single statement scope:
let isTerminator = true, isReturn = true in
  def RET : I<0xC3, RawFrm, (outs), (ins), "ret", []>;

// Block scope (multiple records):
let isCall = true in {
  let Defs = [EAX, ECX, EDX, EFLAGS] in {
    def CALL32r : I<0xFF, MRM2r, (outs), (ins GR32:$dst), "call\t{*}$dst", []>;
    def CALL32m : I<0xFF, MRM2m, (outs), (ins i32mem:$dst), "call\t{*}$dst", []>;
  }
}
```

> ⚠️ **Key rule:** Top-level `let` only overrides *inherited* fields. It cannot override fields defined directly in the record's own body.

### `let` Inside a Record Body

```tablegen
class Base {
  int X = 5;
  int Y = 10;
}

def MyRec : Base {
  let X = 99;   // Override inherited X
  int Z = 20;   // New field
}
// MyRec: X=99, Y=10, Z=20
```

### `let append` and `let prepend`

```tablegen
let append Defs = [EFLAGS] in {
  def ADD : ...;   // ADD.Defs gets EFLAGS appended to whatever it already has
}
```

---

## 10. The `defvar` Statement

`defvar` defines a **variable** (not a record field). Variables are used for temporary values during processing.

### Global Variables

```tablegen
defvar MaxRegs = 32;
defvar RegPrefix = "X";

def X0 : Register<0>;
// Can use MaxRegs and RegPrefix in expressions below
```

### Variables in Record Bodies

```tablegen
class Foo<int x> {
  defvar doubled = !mul(x, 2);   // Local variable
  int Value = doubled;            // Use it
  // "doubled" is NOT a field of Foo
}
```

### Variables in `foreach`

```tablegen
foreach i = 0...7 in {
  defvar regName = "X" # i;   // Local to each iteration
  def X#i : Register<i> {
    string Name = regName;
  }
}
```

> ⚠️ Variables defined in `foreach` go out of scope at the end of each iteration. You cannot accumulate values across iterations.

---

## 11. The `defset` Statement

`defset` collects all records defined within its scope into a named global list:

```tablegen
defset list<Register> AllRegisters = {
  def X0 : Register<0>;
  def X1 : Register<1>;
  def X2 : Register<2>;
  // ... etc.
}

// AllRegisters is now [X0, X1, X2, ...]
// X0, X1, X2 are also defined as normal records
```

This is useful for backends that need to iterate over all records of a certain type.

### Nested `defset`

```tablegen
defset list<Register> AllRegs = {
  defset list<Register> IntRegs = {
    def X0 : Register<0>;
    def X1 : Register<1>;
  }
  defset list<Register> FPRegs = {
    def D0 : Register<32>;
    def D1 : Register<33>;
  }
}
// IntRegs = [X0, X1]
// FPRegs  = [D0, D1]
// AllRegs = [X0, X1, D0, D1]
```

---

## 12. The `if` Statement

```tablegen
// Basic if-then
if condition then {
  def A : SomeClass;
}

// if-then-else
if condition then {
  def A : ClassA;
} else {
  def A : ClassB;
}

// Single-statement form (no braces)
if condition then def A : SomeClass;
```

### Example: Conditional Record Definition

```tablegen
defvar HasFP = true;

if HasFP then {
  def FP_ADD : FPInstruction<"fadd">;
  def FP_MUL : FPInstruction<"fmul">;
}
```

### `if` Inside a Record Body

```tablegen
class Foo<int x> {
  if !gt(x, 0) then {
    int Positive = x;
  } else {
    int Positive = 0;
  }
}
```

---

## 13. The `assert` Statement

`assert` checks a condition and prints an error if it's false. Great for validating your TableGen files:

```tablegen
// Syntax: assert condition, "error message";

class PersonName<string name> {
  assert !le(!size(name), 32), "Name is too long: " # name;
  string Name = name;
}

class Person<string name, int age> : PersonName<name> {
  assert !and(!ge(age, 1), !le(age, 120)), "Invalid age: " # age;
  int Age = age;
}

def Knuth : Person<"Donald Knuth", 60>;   // OK
// def Bad : Person<"X", 200>;            // Would trigger assert!
```

### Where Assertions Are Checked

| Location | When checked |
|----------|-------------|
| Top level | Immediately when parsed |
| In a record body | After the record is fully built |
| In a class | When each subclass/record inheriting from it is built |
| In a multiclass | Each time the multiclass is instantiated with `defm` |

---

## 14. Include Files and Preprocessor

### Including Other Files

```tablegen
include "llvm/Target/Target.td"
include "llvm/Target/TargetSchedule.td"
```

The included file's content is lexically inserted at the `include` point.

### Preprocessor Directives

```tablegen
#define HAVE_FEATURE_X

#ifdef HAVE_FEATURE_X
  def FeatureX : SubtargetFeature<"feature-x", "HasFeatureX", "true", "Enable Feature X">;
#endif

#ifndef HAVE_FEATURE_Y
  // Feature Y not available
#endif
```

You can also define macros from the command line:

```bash
llvm-tblgen MyTarget.td -DHAVE_FEATURE_X
```

---

## 15. How Records Are Built — Step by Step

Understanding this helps you predict what a record will look like after TableGen processes it.

Given:
```tablegen
class C<int x> {
  int Y = x;
  int Yplus1 = !add(Y, 1);
}

let Y = 10 in {
  def rec1 : C<5>;
}
```

TableGen builds `rec1` in this order:

1. **Create empty record** named `rec1`
2. **Process parent class `C<5>`:**
   - Add field `Y = 5` (from template arg `x=5`)
   - Add field `Yplus1 = !add(Y, 1)` (not yet resolved)
3. **Apply top-level `let Y = 10`:**
   - Override `Y = 10`
4. **Parse record body** (empty in this case)
5. **Resolve inter-field references:**
   - `Yplus1 = !add(Y, 1) = !add(10, 1) = 11`
6. **Add to record list**

Result:
```
def rec1 {   // C
  int Y = 10;
  int Yplus1 = 11;
}
```

> 💡 **Key insight:** The `let` override happens *before* field references are resolved. So `Yplus1` sees the overridden value of `Y`, not the original template argument value.

---

## 16. Quick Reference Cheat Sheet

### Types

```tablegen
bit           // 0 or 1
int           // 64-bit integer
string        // Text
code          // Alias for string
bits<n>       // n-bit integer (e.g., bits<8>)
list<T>       // List of type T
dag           // Directed acyclic graph
ClassName     // Record of that class type
```

### Statements

```tablegen
class Foo<args> : Parents { ... }    // Define abstract class
def Bar : Foo<vals> { ... }          // Define concrete record
let field = val in { ... }           // Override fields in scope
defvar x = expr;                     // Define variable
defset list<T> name = { ... }        // Collect records into list
foreach i = range in { ... }         // Loop
if cond then { ... } else { ... }    // Conditional
assert cond, "msg";                  // Validation
include "file.td"                    // Include another file
multiclass MC<args> { ... }          // Define multi-record template (Part 2)
defm Name : MC<vals>;                // Instantiate multiclass (Part 2)
```

### Value Suffixes

```tablegen
val{n}          // Bit n of val
val{n...m}      // Bits n down to m
val[i]          // Element i of list
val[i...j]      // Slice of list
val.field       // Field of record
```

### Special Values

```tablegen
?               // Uninitialized
true / false    // Boolean (= 1 / 0)
NAME            // Implicit: name of the enclosing def/defm
```

---

## What's Next?

In **Part 2**, we'll cover:
- `multiclass` and `defm` — the most powerful TableGen feature
- `foreach` — generating many records at once
- **Bang operators** — TableGen's built-in functions (`!add`, `!if`, `!foreach`, `!cast`, etc.)
- **DAGs** in depth — how instruction patterns work
- `deftype` — type aliases
- The `dump` statement for debugging

These are the features you'll see *constantly* in AArch64 `.td` files.

---

*Part 1 of 3 — TableGen Beginner's Guide*
