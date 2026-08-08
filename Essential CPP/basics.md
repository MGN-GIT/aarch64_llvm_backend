# Core C/C++ Fundamentals — ARM/Embedded Interview Prep

A quick-reference guide to the C/C++ concepts that come up most often in ARM and embedded systems interviews. Each section covers the concept, why it matters in embedded/ARM contexts, a code example, and likely interview questions.

---

## Table of Contents
1. [Pointers](#1-pointers)
2. [`volatile`](#2-volatile)
3. [`static`](#3-static)
4. [Struct Padding & Alignment](#4-struct-padding--alignment)
5. [Bitfields & Bitwise Operators](#5-bitfields--bitwise-operators)
6. [Memory Management](#6-memory-management)
7. [Smart Pointers](#7-smart-pointers)
8. [Multithreading Primitives](#8-multithreading-primitives)

---

## 1. Pointers

**What it is:** A pointer is a variable that stores the memory address of another variable. Function pointers store the address of executable code; pointer-to-pointer stores the address of another pointer.

**Why it matters:** In embedded/ARM work, pointers are how you talk directly to hardware — memory-mapped I/O, DMA buffers, and register access all go through raw pointers.

```c
int x = 42;
int *p = &x;        // pointer to int
int **pp = &p;       // pointer to pointer to int

// Function pointer: points to a function taking two ints, returning int
int (*op)(int, int);

// Common ARM pattern: pointer to a hardware register
#define GPIO_BASE 0x40020000
volatile uint32_t *gpio_reg = (volatile uint32_t *)GPIO_BASE;
```

**Pointer arithmetic:** `p + 1` advances by `sizeof(*p)` bytes, not 1 byte — critical to know for array traversal and buffer math.

**Likely interview questions:**
- What's the difference between `int *const p` and `const int *p`?
- How would you implement a callback mechanism in C using function pointers?
- What happens if you dereference a `NULL` or dangling pointer?
- Explain how a jump table (array of function pointers) could replace a large `switch` statement.

---

## 2. `volatile`

**What it is:** A type qualifier telling the compiler that a variable's value can change at any time, outside the normal flow of the program — so it must **never** cache the value in a register or optimize away "redundant" reads/writes.

**Why it matters:** Essential for memory-mapped hardware registers, variables shared with an ISR, and anything touched outside the compiler's visibility. Without it, the compiler may optimize away a loop that polls a status register, because from its point of view nothing in the loop changes the value.

```c
// WRONG: compiler may optimize this into an infinite loop
// or a single read, since it doesn't see status changing.
uint32_t status = *status_reg;
while (status == 0) {
    status = *status_reg;
}

// CORRECT
volatile uint32_t *status_reg = (volatile uint32_t *)STATUS_ADDR;
while (*status_reg == 0) {
    // wait for hardware to update the register
}
```

**Likely interview questions:**
- Why doesn't `volatile` guarantee thread safety or atomicity?
- Can a variable be both `const` and `volatile`? (Yes — e.g., a read-only hardware status register.)
- What's the difference between `volatile` and a memory barrier?
- Give an example of a bug caused by forgetting `volatile` in an ISR-shared variable.

---

## 3. `static`

**What it is:** Meaning depends entirely on context — this is a favorite "explain all the meanings" interview question.

| Context | Effect |
|---|---|
| Local variable inside a function | Persists across function calls; initialized once |
| Global variable / function at file scope | Internal linkage — not visible outside that translation unit |
| Class member variable (C++) | Shared across all instances of the class |
| Class member function (C++) | Callable without an instance; no access to `this` |

```c
void counter(void) {
    static int calls = 0;   // persists between calls, initialized once
    calls++;
    printf("%d\n", calls);
}

// file-scope static: only visible within this .c file
static int internal_helper(int x) { return x * 2; }
```

```cpp
class Sensor {
public:
    static int instance_count;   // shared across all Sensor objects
    static void reset_count() { instance_count = 0; }  // no 'this'
};
int Sensor::instance_count = 0;
```

**Likely interview questions:**
- What's the difference between a `static` global and a plain global variable?
- Why would you use `static` for a helper function in a `.c` file?
- Is a `static` local variable thread-safe? (No, not by default — needs synchronization.)
- What's a "Meyers' singleton" and how does `static` make it thread-safe in C++11+?

---

## 4. Struct Padding & Alignment

**What it is:** Compilers insert padding bytes between struct members so each member sits at an address matching its natural alignment (e.g., a 4-byte `int` at an address divisible by 4). This is a direct consequence of how the ARM memory system and instruction set access data efficiently.

**Why it matters:** Matters enormously for embedded work — packet/register layouts, DMA buffers, and cross-compiler struct compatibility all depend on understanding (and sometimes disabling) padding.

```c
struct Example {
    char  a;   // 1 byte
    // 3 bytes padding inserted here
    int   b;   // 4 bytes
    char  c;   // 1 byte
    // 3 bytes padding at the end (struct size must be multiple of largest alignment)
};
// sizeof(struct Example) == 12, not 6

// Force no padding — common for matching a hardware register layout
#pragma pack(push, 1)
struct PackedExample {
    char  a;
    int   b;
    char  c;
};
#pragma pack(pop)
// sizeof(struct PackedExample) == 6
```

**Likely interview questions:**
- Why might you avoid `#pragma pack(1)` on ARM even though it saves memory? (Unaligned access can be slower or, on some cores/configurations, fault.)
- How would you reorder struct members to minimize padding? (Largest members first, generally.)
- What's the difference between alignment and padding?
- How does struct layout affect binary compatibility between code compiled by different compilers?

---

## 5. Bitfields & Bitwise Operators

**What it is:** Bitwise operators (`&`, `|`, `^`, `~`, `<<`, `>>`) manipulate individual bits directly. Bitfields let you declare struct members that occupy a specific number of bits rather than a full byte/word.

**Why it matters:** Register-level programming is bit-level programming — setting a control bit, checking a status flag, or packing several small values into one word are everyday embedded tasks.

```c
// Set, clear, toggle, check bit N
#define SET_BIT(reg, n)    ((reg) |=  (1U << (n)))
#define CLEAR_BIT(reg, n)  ((reg) &= ~(1U << (n)))
#define TOGGLE_BIT(reg, n) ((reg) ^=  (1U << (n)))
#define CHECK_BIT(reg, n)  (((reg) >> (n)) & 1U)

// Bitfield struct — common for register descriptions
struct StatusReg {
    unsigned ready    : 1;
    unsigned error    : 1;
    unsigned mode     : 2;
    unsigned reserved : 4;
};  // fits in 1 byte
```

**Likely interview questions:**
- Live-coding: "Check if a number is a power of two" (`n && !(n & (n-1))`).
- Live-coding: "Count the number of set bits in an integer" (Brian Kernighan's algorithm).
- Why is bitfield layout not portable across compilers/platforms?
- How would you swap two variables without a temp variable using XOR — and why is that usually a bad idea in real code?

---

## 6. Memory Management

**What it is:** Understanding where variables live (stack vs heap) and how to allocate/free them correctly and safely, especially in memory-constrained embedded environments.

| | Stack | Heap |
|---|---|---|
| Allocation | Automatic (function scope) | Manual (`malloc`/`new`) |
| Speed | Fast (pointer bump) | Slower (allocator bookkeeping) |
| Size | Limited, fixed at compile/link time | Larger, but fragmentable |
| Lifetime | Ends when scope exits | Until explicitly freed |
| Embedded concern | Stack overflow (deep recursion, large locals) | Fragmentation, leaks — often avoided entirely in hard-real-time code |

```c
void stack_example(void) {
    int local[100];   // on the stack, freed automatically on return
}

void heap_example(void) {
    int *buf = malloc(100 * sizeof(int));
    if (buf == NULL) {
        // always check — embedded heaps are small
        return;
    }
    // ... use buf ...
    free(buf);
    buf = NULL;   // avoid dangling pointer / double-free
}
```

```cpp
// C++ equivalent — prefer new/delete pairing, or better, RAII (see smart pointers)
int *buf = new int[100];
delete[] buf;
```

**Likely interview questions:**
- What causes a memory leak, and how would you detect one in a long-running embedded system?
- What's a dangling pointer vs a wild pointer?
- Why is dynamic allocation (`malloc`/`new`) often banned or restricted in hard-real-time/safety-critical embedded code? (Non-deterministic timing, fragmentation risk.)
- What's a double-free, and what usually causes it?
- How does a memory pool / fixed-size allocator avoid fragmentation?

---

## 7. Smart Pointers

**What it is:** C++11+ RAII wrappers around raw pointers that automatically manage object lifetime, eliminating most manual `delete` calls and the leaks/double-frees that come with them.

| Type | Ownership | Use case |
|---|---|---|
| `std::unique_ptr` | Exclusive | Default choice — zero overhead vs raw pointer |
| `std::shared_ptr` | Shared (reference-counted) | Object needs multiple owners |
| `std::weak_ptr` | Non-owning reference to a `shared_ptr` | Break reference cycles, observe without owning |

```cpp
#include <memory>

void unique_example() {
    std::unique_ptr<Sensor> s = std::make_unique<Sensor>();
    // automatically deleted when s goes out of scope — no manual delete
}

void shared_example() {
    std::shared_ptr<Sensor> s1 = std::make_shared<Sensor>();
    std::shared_ptr<Sensor> s2 = s1;   // reference count now 2
    // object deleted only when the last shared_ptr goes out of scope
}
```

**Note on embedded use:** `shared_ptr`'s reference counting and heap allocation make it a poor fit for hard-real-time/resource-constrained code; `unique_ptr` is generally safe since it has no runtime overhead over a raw pointer. This distinction itself is a common interview probe.

**Likely interview questions:**
- Why would you choose `unique_ptr` over `shared_ptr` on an embedded target?
- How does `shared_ptr` implement reference counting, and is it thread-safe? (The control block's refcount is atomic; the pointed-to object's data is not automatically protected.)
- What problem does `weak_ptr` solve? (Breaking `shared_ptr` reference cycles that would otherwise leak.)
- What's the overhead of `unique_ptr` vs a raw pointer? (Effectively none with the default deleter — same size, calls destructor automatically.)

---

## 8. Multithreading Primitives

**What it is:** Mechanisms for coordinating concurrent execution — critical for ISR/main-loop interaction and RTOS task synchronization.

**Deadlock vs Livelock:**
- **Deadlock:** two or more threads are each waiting on a resource the other holds — nobody makes progress, and everything is blocked.
- **Livelock:** threads are actively responding to each other (not blocked) but still make no real progress — e.g., two threads each repeatedly yielding to avoid a collision, forever stepping around each other.

```c
// Classic deadlock: two mutexes acquired in opposite order by two threads
// Thread A:                    Thread B:
lock(mutex1);                   lock(mutex2);
lock(mutex2);  // waits for B   lock(mutex1);  // waits for A
                                 // -> deadlock
```

**Mutex vs Semaphore:**

| | Mutex | Semaphore |
|---|---|---|
| Purpose | Mutual exclusion (1 owner) | Signaling / resource counting |
| Ownership | Owned by the locking thread; only that thread unlocks it | No ownership — any thread can post/signal |
| Typical use | Protect a shared variable/critical section | Signal an ISR-to-task event, limit concurrent access to N resources |

```c
// Mutex: protect a shared counter
pthread_mutex_lock(&lock);
shared_counter++;
pthread_mutex_unlock(&lock);

// Semaphore: classic ISR-to-task signaling pattern
void ISR_Handler(void) {
    sem_post(&data_ready);   // signal from ISR — non-blocking
}
void task(void) {
    sem_wait(&data_ready);   // task blocks until ISR signals
    process_data();
}
```

**Priority Inversion:** occurs when a low-priority task holds a resource (mutex) that a high-priority task needs, and a medium-priority task preempts the low-priority one — effectively blocking the high-priority task indefinitely behind a task of lower priority than itself. Classic fix: **priority inheritance** (the low-priority task temporarily inherits the high-priority task's priority while holding the lock).

**Likely interview questions:**
- Give a concrete example of deadlock and how you'd prevent it (lock ordering, timeout-based locks, `std::lock`/`try_lock`).
- What's the difference between livelock and deadlock, in terms of thread state (blocked vs. actively running)?
- Explain priority inversion and how priority inheritance solves it — this is a very common ARM/RTOS-specific question (famously the Mars Pathfinder bug).
- When would you use a semaphore instead of a mutex? (Signaling across ISR/task boundary — mutexes generally shouldn't be taken inside an ISR since they can block.)
- How would you protect a variable shared between an ISR and the main loop on a single-core MCU? (Disable interrupts briefly, or use `volatile` + atomic operations — a full mutex is often unsuitable inside an ISR.)

---

## Quick Self-Check
Before the interview, make sure you can, without notes:
- [ ] Explain `int *const p` vs `const int *p` vs `const int *const p`
- [ ] Explain why `volatile` alone doesn't make code thread-safe
- [ ] List all four meanings of `static` in C/C++
- [ ] Compute the `sizeof` a struct given its member order, and reorder it to minimize padding
- [ ] Write set/clear/toggle/check-bit macros from memory
- [ ] Explain the tradeoffs of `malloc` in a hard-real-time system
- [ ] Explain when `unique_ptr` has zero overhead vs a raw pointer
- [ ] Diagram a deadlock scenario and two ways to prevent it
- [ ] Explain priority inversion and priority inheritance with a concrete example



# Advanced C/C++ Concepts — ARM/Embedded Interview Prep (Part 2)

Companion to the first README. Covers OOP mechanics, compile-time concepts, low-level/embedded specifics, error handling, and modern C++ features that show up in ARM/embedded interviews.

---

## Table of Contents
1. [Virtual Functions & Vtables](#1-virtual-functions--vtables)
2. [Rule of Three/Five/Zero](#2-rule-of-threefivezero)
3. [Move Semantics & Rvalue References](#3-move-semantics--rvalue-references)
4. [Diamond Problem & Virtual Inheritance](#4-diamond-problem--virtual-inheritance)
5. [Casts: static/dynamic/const/reinterpret](#5-casts-staticdynamicconstreinterpret)
6. [`constexpr` vs `const` vs `#define`](#6-constexpr-vs-const-vs-define)
7. [`inline` vs Macros](#7-inline-vs-macros)
8. [Templates](#8-templates)
9. [Fixed-Width Integer Types](#9-fixed-width-integer-types)
10. [Strict Aliasing & Type Punning](#10-strict-aliasing--type-punning)
11. [Undefined Behavior](#11-undefined-behavior)
12. [Sequence Points](#12-sequence-points)
13. [`extern "C"` & Name Mangling](#13-extern-c--name-mangling)
14. [Compiler vs Memory Barriers](#14-compiler-vs-memory-barriers)
15. [Reentrancy](#15-reentrancy)
16. [Linker Sections](#16-linker-sections)
17. [Exceptions in Embedded C++](#17-exceptions-in-embedded-c)
18. [Lambdas & Captures](#18-lambdas--captures)

---

## 1. Virtual Functions & Vtables

**What it is:** A `virtual` function enables runtime polymorphism — the actual function called is resolved based on the object's real type, not the static type of the pointer/reference. The compiler implements this with a **vtable** (virtual table): a hidden array of function pointers, one per class with virtual functions, plus a hidden pointer (`vptr`) in each object pointing to its class's vtable.

**Why it matters in embedded:** Every virtual call costs an extra pointer dereference (object → vtable → function), and every polymorphic object carries the extra `vptr` overhead. In interrupt handlers or tight loops, this indirection and the loss of inlining/branch-prediction certainty is often reason enough to avoid heavy virtual dispatch.

```cpp
class Sensor {
public:
    virtual void read() { /* default */ }
    virtual ~Sensor() {}   // virtual destructor — required for polymorphic deletion
};

class TempSensor : public Sensor {
public:
    void read() override { /* temp-specific */ }
};

Sensor *s = new TempSensor();
s->read();     // resolved at runtime via vtable -> TempSensor::read()
delete s;      // virtual destructor ensures TempSensor::~TempSensor() runs
```

**Likely interview questions:**
- Why must a base class destructor be `virtual` if you delete through a base pointer?
- What's the memory cost of adding one virtual function to a class? (A `vptr` per object, typically one machine word.)
- How does the compiler resolve a virtual call at the assembly level?
- Why might you use `final` on a class/method in performance-critical embedded code?

---

## 2. Rule of Three/Five/Zero

**What it is:** Guidance on which special member functions a class needs to define once it manages a resource (raw pointer, file handle, etc.):
- **Rule of Three (C++98):** if you define one of {destructor, copy constructor, copy assignment}, you almost certainly need all three.
- **Rule of Five (C++11+):** add move constructor and move assignment to the set.
- **Rule of Zero (modern best practice):** don't manage raw resources yourself at all — wrap them in RAII types (`unique_ptr`, containers) so the compiler-generated specials are correct by default and you define none of them.

```cpp
class Buffer {
    uint8_t *data;
    size_t   size;
public:
    Buffer(size_t n) : data(new uint8_t[n]), size(n) {}
    ~Buffer() { delete[] data; }                                   // 1: destructor
    Buffer(const Buffer &o) : data(new uint8_t[o.size]), size(o.size) {
        std::copy(o.data, o.data + size, data);                    // 2: copy ctor
    }
    Buffer &operator=(const Buffer &o) { /* ... */ return *this; } // 3: copy assign
    Buffer(Buffer &&o) noexcept : data(o.data), size(o.size) {
        o.data = nullptr;                                          // 4: move ctor
    }
    Buffer &operator=(Buffer &&o) noexcept { /* ... */ return *this; } // 5: move assign
};

// Rule of Zero equivalent — let unique_ptr manage the resource
class BufferRuleOfZero {
    std::unique_ptr<uint8_t[]> data;
    size_t size;
public:
    BufferRuleOfZero(size_t n) : data(std::make_unique<uint8_t[]>(n)), size(n) {}
    // no destructor, copy, or move needed — compiler generates correct ones
};
```

**Likely interview questions:**
- What breaks if a class holding a raw pointer relies on the compiler-generated (shallow) copy constructor? (Double-free on destruction — both copies point at the same memory.)
- Why prefer Rule of Zero in new code?
- What does `= delete` do, and when would you use it (e.g., to make a class non-copyable)?

---

## 3. Move Semantics & Rvalue References

**What it is:** `&&` denotes an rvalue reference — a reference to a temporary/about-to-be-destroyed object. `std::move` doesn't move anything itself; it's a cast that tells the compiler "treat this as an rvalue," enabling a move constructor/assignment to steal the resource instead of deep-copying it.

**Why it matters:** Moving a `unique_ptr` is just a pointer swap (cheap, O(1)); copying a `shared_ptr` requires an atomic increment; copying a container deep-copies every element. Understanding which operation you're triggering matters for performance-sensitive embedded code.

```cpp
std::unique_ptr<Sensor> a = std::make_unique<Sensor>();
std::unique_ptr<Sensor> b = std::move(a);   // ownership transferred, a is now null
// a.get() == nullptr here

std::vector<int> make_large_vector() {
    std::vector<int> v(10000);
    return v;   // move (or elided) on return — no deep copy
}
```

**Likely interview questions:**
- What does `std::move` actually do under the hood? (Nothing at runtime — it's just a `static_cast` to an rvalue reference.)
- What state should a moved-from object be left in? (Valid but unspecified — safe to destroy or reassign.)
- What is copy elision / RVO (return value optimization)?
- Why is `std::move`-ing a `const` object effectively a no-op (falls back to copy)?

---

## 4. Diamond Problem & Virtual Inheritance

**What it is:** When a class inherits from two classes that both inherit from a common base, the derived class ends up with two copies of the base's members — ambiguous and wasteful. `virtual` inheritance makes the base shared (a single copy) instead of duplicated.

```cpp
class Device        { public: int id; };
class Sensor  : virtual public Device {};
class Actuator: virtual public Device {};
class SmartValve : public Sensor, public Actuator {};
// virtual inheritance -> SmartValve has exactly ONE Device::id, not two

SmartValve v;
v.id = 5;   // unambiguous, thanks to virtual inheritance
```

**Likely interview questions:**
- What happens if you don't use `virtual` inheritance in this scenario? (Ambiguous access to `id` — must qualify as `Sensor::id` or `Actuator::id`.)
- What's the runtime cost of virtual inheritance? (Extra indirection to locate the shared base subobject — generally avoided in performance-critical embedded code.)

---

## 5. Casts: static/dynamic/const/reinterpret

| Cast | Purpose | Embedded relevance |
|---|---|---|
| `static_cast` | Compile-time-checked conversion (numeric types, related class pointers) | General-purpose safe conversions |
| `dynamic_cast` | Runtime-checked downcast, requires polymorphic type (RTTI) | Rarely used in embedded — RTTI often disabled to save flash/avoid overhead |
| `const_cast` | Add/remove `const`/`volatile` qualifiers | Sometimes needed to call a legacy non-const API on const data |
| `reinterpret_cast` | Reinterpret the raw bits as an unrelated type | **Very common** in embedded — mapping a raw address to a register struct |

```cpp
// Classic embedded reinterpret_cast: raw address -> hardware register struct
struct GpioRegs { uint32_t mode, data, dir; };
GpioRegs *gpio = reinterpret_cast<GpioRegs *>(0x40020000);
gpio->mode = 1;
```

**Likely interview questions:**
- Why is `reinterpret_cast` dangerous, and why is it still used so heavily in register-access code?
- Why is `dynamic_cast` often avoided/disabled in embedded C++? (Requires RTTI, adds code size and runtime cost.)
- When would you legitimately need `const_cast`? (Interfacing with an old C API that isn't const-correct, when you know the data won't actually be modified.)

---

## 6. `constexpr` vs `const` vs `#define`

| | Evaluated | Type-checked | Scoped |
|---|---|---|---|
| `#define` | Text substitution, preprocessor | No | No (global, no namespace) |
| `const` | Runtime or compile-time (compiler's choice) | Yes | Yes |
| `constexpr` | Guaranteed compile-time (if used in a constant context) | Yes | Yes |

```cpp
#define BUFFER_SIZE 64            // no type checking, just text substitution
const int buffer_size = 64;       // typed, but not guaranteed compile-time
constexpr int BUFFER_SIZE2 = 64;  // guaranteed compile-time -> usable as array bound, template arg

uint8_t buffer[BUFFER_SIZE2];     // valid: constexpr is a true compile-time constant
```

**Likely interview questions:**
- Why is `constexpr` preferred over `#define` for constants in modern C++?
- Can a `constexpr` function also run at runtime? (Yes — if its arguments aren't compile-time constants, it just becomes a normal function call.)
- What debugging problems does `#define` cause that `const`/`constexpr` avoid? (No type info, no scoping, can't set a breakpoint or watch on a macro.)

---

## 7. `inline` vs Macros

**What it is:** `inline` is a *hint* to the compiler that it may substitute the function body at the call site to avoid call overhead — it is not a guarantee (the compiler can ignore it, and modern compilers make their own inlining decisions regardless of the keyword). A `#define` macro is blind text substitution with no type checking, no scoping, and classic pitfalls around operator precedence and multiple evaluation.

```c
#define SQUARE(x) ((x) * (x))
int a = 5;
int r = SQUARE(a++);   // expands to ((a++) * (a++)) -- a incremented TWICE, undefined behavior

inline int square(int x) { return x * x; }
int r2 = square(a++);  // a incremented exactly once, as expected
```

**Likely interview questions:**
- Why can a macro like `SQUARE(x)` produce a bug that an `inline` function wouldn't?
- Does the `inline` keyword guarantee the function is actually inlined? (No — it's advisory; today it mainly affects linkage, allowing the definition in a header without ODR violations.)
- Why must you always parenthesize macro arguments and the whole expansion?

---

## 8. Templates

**What it is:** Compile-time generic programming — the compiler generates a separate instantiation of a template for each distinct type it's used with.

**Why it matters in embedded:** Powerful for writing reusable, type-safe code (e.g., a generic circular buffer), but each distinct instantiation is separately compiled code — heavy template use can cause **code bloat**, a real concern on flash-constrained MCUs.

```cpp
template <typename T, size_t N>
class CircularBuffer {
    T data[N];
    size_t head = 0, tail = 0;
public:
    void push(const T &item) { data[head] = item; head = (head + 1) % N; }
    T pop() { T item = data[tail]; tail = (tail + 1) % N; return item; }
};

CircularBuffer<uint8_t, 32> uart_buf;   // one instantiation
CircularBuffer<float, 16>   adc_buf;    // a second, separate instantiation
```

**Likely interview questions:**
- What is "code bloat" in the context of templates, and how would you mitigate it? (Share common logic in a non-template base, template only the type-specific part.)
- What's the difference between a template and a macro for generic code? (Templates are type-checked and scoped; macros are blind text substitution.)
- What is template specialization, and when would you use it?

---

## 9. Fixed-Width Integer Types

**What it is:** Types from `<stdint.h>` (`uint8_t`, `int16_t`, `uint32_t`, etc.) that guarantee an exact bit width, unlike `int`/`long`/`short`, whose sizes are implementation-defined and vary across platforms/compilers.

**Why it matters:** When you're describing a hardware register or a wire protocol, you need to know *exactly* how many bits/bytes you're dealing with — `int` might be 16, 32, or 64 bits depending on the target.

```c
#include <stdint.h>

typedef struct {
    uint8_t  status;   // exactly 1 byte, guaranteed
    uint16_t count;    // exactly 2 bytes, guaranteed
    uint32_t timestamp;// exactly 4 bytes, guaranteed
} __attribute__((packed)) Packet;
```

**Likely interview questions:**
- Why is `int` a poor choice for a struct that mirrors a hardware register layout?
- What's the difference between `uint32_t`, `uint_fast32_t`, and `uint_least32_t`?
- Why might `size_t` differ between a 32-bit and 64-bit ARM target?

---

## 10. Strict Aliasing & Type Punning

**What it is:** The strict aliasing rule says the compiler may assume that pointers of different (unrelated) types never point to the same memory — which lets it make aggressive optimizations, but means casting, say, a `float*` to a `uint32_t*` and dereferencing it is technically **undefined behavior**, even though it "works" in practice on many compilers/settings.

```c
// UB under strict aliasing — compiler may optimize incorrectly
float f = 3.14f;
uint32_t bits = *(uint32_t *)&f;   // dereferencing through unrelated pointer type

// SAFE alternative #1: memcpy (compiler recognizes and optimizes this)
uint32_t bits2;
memcpy(&bits2, &f, sizeof(bits2));

// SAFE alternative #2: union-based type punning (well-defined in C, common in embedded)
union FloatBits {
    float    f;
    uint32_t u;
};
union FloatBits fb;
fb.f = 3.14f;
uint32_t bits3 = fb.u;   // well-defined in C (note: technically UB in strict C++, though widely supported)
```

**Likely interview questions:**
- Why does casting between unrelated pointer types risk undefined behavior even if it "looks fine" in testing?
- What compiler flag relaxes strict aliasing, and why would you avoid relying on it? (`-fno-strict-aliasing` — masks the bug rather than fixing it, and isn't portable.)
- Why is `memcpy` often the *fastest* safe option, given compilers recognize and optimize it away entirely?

---

## 11. Undefined Behavior

**What it is:** Behavior the C/C++ standard places no requirements on — the compiler is free to do *anything*, including something that looks correct today and breaks with a compiler upgrade or optimization level change. Classic sources:
- Signed integer overflow (`INT_MAX + 1`)
- Dereferencing a `NULL`/wild/dangling pointer
- Reading an uninitialized variable
- Out-of-bounds array access
- Multiple unsequenced modifications to the same variable (see Sequence Points below)

```c
int x = INT_MAX;
x = x + 1;   // signed overflow -> UB (NOT guaranteed to wrap to INT_MIN)

int arr[5];
int y = arr[10];   // out-of-bounds read -> UB, may return garbage or crash
```

**Likely interview questions:**
- Why is signed overflow UB while unsigned overflow is well-defined (wraps modulo 2^n)?
- Give an example of UB that "worked" in a debug build but broke in a release/optimized build.
- Why do embedded/safety-critical teams use static analysis tools (e.g., MISRA C checkers) specifically to catch UB?

---

## 12. Sequence Points

**What it is:** Points in program execution where all side effects of prior expressions are guaranteed complete before moving on. Between sequence points, if a variable is modified more than once (or modified and read for something other than computing the new value), the result is undefined.

```c
int i = 1;
int r = i++ + ++i;   // UB: i is modified twice with no sequence point between —
                       // result is compiler/platform-dependent, not "5" or any fixed value

// Function call arguments: order of evaluation is unspecified (not necessarily UB,
// but still a common source of bugs)
printf("%d %d\n", f(), g());  // order f() and g() are called in is unspecified
```

**Likely interview questions:**
- Why is `i = i++ + ++i` undefined rather than just "implementation-defined"?
- What guarantees do the `,` (comma), `&&`, `||`, and `?:` operators provide about evaluation order that plain arithmetic operators don't?
- How did C++17 tighten these rules compared to C++14? (Many previously-unsequenced operations, like function argument evaluation relative to the call, became better defined — though argument-to-argument order is still unspecified.)

---

## 13. `extern "C"` & Name Mangling

**What it is:** C++ compilers mangle function names (encoding parameter types into the symbol name) to support overloading — but C compilers don't. `extern "C"` tells the C++ compiler to use C-style (unmangled) linkage for a function, so C and C++ object files can link together.

**Why it matters:** Extremely relevant to ARM toolchain work — mixing a C-based HAL/driver library with C++ application code, or exposing a C++ library to a C caller, requires this.

```cpp
// header shared between C and C++ code
#ifdef __cplusplus
extern "C" {
#endif

void hal_gpio_init(void);   // C linkage — callable from plain C, no name mangling

#ifdef __cplusplus
}
#endif
```

**Likely interview questions:**
- What does a mangled C++ symbol look like, and why does the linker need this? (Encodes namespace, class, and parameter types so overloaded functions get distinct symbols — e.g., `_Z3fooi` for `foo(int)`.)
- Why would a linker error like "undefined reference to `foo`" occur when mixing C and C++ without `extern "C"`?
- Can you overload a function declared `extern "C"`? (No — C linkage means no mangling, so no overload resolution by signature.)

---

## 14. Compiler vs Memory Barriers

**What it is:** Two distinct reordering problems:
- **Compiler reordering:** the compiler may reorder/optimize instructions as long as single-threaded observable behavior is unchanged. `volatile` prevents the compiler from doing this to that specific variable's accesses.
- **CPU/hardware reordering:** on multi-core or out-of-order CPUs, the *processor itself* may execute/complete memory operations out of program order. `volatile` does **not** prevent this — you need an actual memory barrier (`std::atomic` with the right memory order, or an explicit barrier instruction like ARM's `DMB`/`DSB`/`ISB`).

```c
// volatile prevents compiler reordering/caching, but NOT CPU reordering on multicore
volatile int flag = 0;
int data = 0;

// Thread A (or ISR)
data = 42;
flag = 1;         // on a multicore/OoO CPU, another core might observe flag==1
                   // before data==42 without a proper barrier

// Correct multicore-safe version uses atomics with explicit memory ordering
#include <stdatomic.h>
atomic_int flag2 = 0;
data = 42;
atomic_store_explicit(&flag2, 1, memory_order_release);
// consumer: atomic_load_explicit(&flag2, memory_order_acquire)
```

**Likely interview questions:**
- Why is `volatile` sufficient for single-core ISR-to-main-loop communication but not for multicore synchronization?
- What do ARM's `DMB` (Data Memory Barrier), `DSB` (Data Synchronization Barrier), and `ISB` (Instruction Synchronization Barrier) each do?
- What's the difference between acquire and release memory ordering semantics?

---

## 15. Reentrancy

**What it is:** A function is **reentrant** if it can be safely interrupted partway through and called again ("re-entered") — by an ISR, another thread, or recursively — before the first call finishes, without corrupting shared state. This is distinct from (but related to) thread safety.

**Requirements for reentrancy:**
- No use of non-local static/global variables without protection
- Doesn't call non-reentrant functions
- Doesn't return a pointer to static data
- Only operates on data passed to it (or local/stack data)

```c
// NOT reentrant — static buffer shared across all calls
char *int_to_str_bad(int n) {
    static char buf[12];   // shared state -> reentrant call corrupts it
    sprintf(buf, "%d", n);
    return buf;
}

// Reentrant — caller-provided buffer, no shared state
void int_to_str_good(int n, char *buf, size_t buflen) {
    snprintf(buf, buflen, "%d", n);
}
```

**Likely interview questions:**
- Is `strtok()` reentrant? (No — classic example; `strtok_r()` is the reentrant version.)
- Can a function be thread-safe but not reentrant, or reentrant but not thread-safe? (Reentrancy is about interruption/recursion safety; thread-safety is about concurrent access — mutex-protected code can be thread-safe yet non-reentrant if called recursively by the same thread and it self-deadlocks.)
- Why must ISRs generally only call reentrant functions?

---

## 16. Linker Sections

**What it is:** The linker places compiled code and data into named sections, each with different properties, defined by the linker script:

| Section | Contents | Notes |
|---|---|---|
| `.text` | Executable code | Typically placed in flash/ROM |
| `.rodata` | Read-only data (string literals, `const` globals) | Also usually in flash |
| `.data` | Initialized global/static variables | Stored in flash, **copied to RAM at startup** by the reset handler |
| `.bss` | Uninitialized (zero-initialized) global/static variables | Lives only in RAM, **zeroed at startup**, occupies no flash space |

```c
int initialized_global = 5;      // .data — has a nonzero initial value, copied from flash to RAM at boot
int uninitialized_global;        // .bss  — zero-initialized, no flash space consumed
const char *msg = "hello";       // "hello" itself -> .rodata; the pointer variable -> .data
static int counter;              // .bss (zero by default)
```

**Likely interview questions:**
- Why does startup code need to copy `.data` from flash to RAM before `main()` runs?
- Why doesn't `.bss` take up space in the flash image, even though it might be large?
- What goes wrong if you access a global variable before the startup code has finished the `.data`/`.bss` initialization? (Common in very early boot/reset-handler code.)
- Where would a large lookup table declared `const` end up, and why does that matter for RAM-constrained MCUs?

---

## 17. Exceptions in Embedded C++

**What it is:** C++ exceptions (`try`/`catch`/`throw`) provide structured error handling, but many embedded/real-time codebases build with `-fno-exceptions` and disable them entirely.

**Why they're often disabled:**
- Exception unwinding has non-deterministic timing — unacceptable for hard-real-time guarantees
- The exception-handling runtime support adds meaningful code size (flash) overhead, even in the no-exception path
- Stack unwinding through interrupt context is generally not well-defined

**Common alternatives:**
- Return codes / error enums, checked explicitly by the caller
- `std::optional` / `std::expected` (C++23) for "value or failure" returns without exceptions
- `assert()` + halt/reset for truly unrecoverable conditions
- Error callback/status-register patterns mirroring how hardware reports faults

```cpp
// Exception-free error handling pattern
enum class SensorError { None, Timeout, NotResponding };

SensorError read_sensor(int &out_value) {
    if (!sensor_ready()) return SensorError::NotResponding;
    out_value = read_raw();
    return SensorError::None;
}

int value;
if (read_sensor(value) != SensorError::None) {
    // handle failure explicitly, no unwinding involved
}
```

**Likely interview questions:**
- Why does exception unwinding conflict with hard-real-time guarantees?
- What compiler flag disables exceptions, and what happens if a `throw` is reached anyway with that flag set? (`-fno-exceptions`; typically calls `std::terminate`/aborts.)
- How would you design an error-handling scheme for a library that must work both with and without exceptions enabled?

---

## 18. Lambdas & Captures

**What it is:** An anonymous inline function object (C++11+). Increasingly common even in embedded C++ for callbacks (timer handlers, HAL completion callbacks) since they're often more efficient than `std::function` and easier to read than free-function-plus-context-pointer patterns.

```cpp
int threshold = 100;

// Capture by reference [&] or by value [=] — critical distinction
auto callback = [&threshold](int reading) {
    if (reading > threshold) trigger_alarm();
};

// Capture by value is safer for ISR-registered callbacks —
// avoids a dangling reference if the enclosing scope has exited by the time it runs
auto safe_callback = [threshold](int reading) {
    if (reading > threshold) trigger_alarm();
};
```

**Likely interview questions:**
- What's the danger of capturing by reference (`[&]`) in a callback that might outlive the enclosing scope — e.g., one registered with an ISR or a timer? (Dangling reference — classic use-after-scope bug.)
- What is a lambda under the hood? (Compiler-generated unnamed class with an `operator()` — a closure, not magic.)
- Why might `std::function` be avoided in embedded code despite being convenient? (Type erasure typically requires heap allocation and adds call overhead vs. a plain function pointer or templated callable.)

---

## Quick Self-Check (Part 2)
- [ ] Explain why a base class needs a virtual destructor
- [ ] State the Rule of Three/Five/Zero and why Rule of Zero is preferred today
- [ ] Explain what `std::move` actually does (a cast, not a physical move)
- [ ] Know when `reinterpret_cast` is appropriate vs dangerous
- [ ] Explain `constexpr` vs `const` vs `#define`
- [ ] Explain why `int` is a bad choice for a hardware register struct
- [ ] Explain the strict aliasing rule and a safe way around it
- [ ] Name three sources of undefined behavior from memory
- [ ] Explain the difference between `.data` and `.bss`
- [ ] Explain why exceptions are often disabled in embedded C++, and one alternative pattern
- [ ] Explain the capture-by-reference danger in a lambda used as an ISR callback
