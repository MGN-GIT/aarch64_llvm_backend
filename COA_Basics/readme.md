# CPU Fundamentals: ISA, Microarchitecture, Pipelining & Memory System — A Beginner's Guide

This guide walks through the core concepts in computer architecture: **Instruction Set Architecture (ISA)**, **Microarchitecture**, **Instruction Pipelining**, and the **Memory System (Cache, TLB, Virtual Memory)**. Think of them as layers of the same onion — ISA is the *contract*, microarchitecture is the *implementation*, pipelining is the *assembly line* that implementation runs on, and the memory system is the *supply chain* feeding that assembly line.

---

## 1. Instruction Set Architecture (ISA)

### What is it, really?

Imagine you buy a universal remote control. The remote has buttons: Power, Volume Up, Channel Down. Any TV that "speaks" this remote's language will respond correctly to these buttons — regardless of whether it's a cheap TV or an expensive one, old or new.

The **ISA is that set of buttons** — the fixed vocabulary of commands a CPU understands. It defines:
- What **instructions** exist (add, subtract, load, store, jump...)
- What **data types** are supported
- What **registers** are available (small storage slots inside the CPU)
- How **memory** is addressed and organized

The key idea: the ISA is *abstract*. It says nothing about how the CPU actually performs these operations internally — only what the outcome must be. This is what lets your old software run on a brand-new processor: as long as the new chip implements the same ISA, it honors the same "contract."

**Analogy:** ISA = the sheet music. Any orchestra that can read this sheet music can play the piece — a school orchestra and a world-famous philharmonic will both play the "correct" notes, even though their performance (speed, richness, polish) differs wildly. That performance difference is the microarchitecture.

### CISC vs. RISC — the big classification

| | CISC (Complex Instruction Set Computer) | RISC (Reduced Instruction Set Computer) |
|---|---|---|
| Philosophy | Many specialized, powerful instructions | Few, simple instructions used very frequently |
| Instruction length | Often variable | Usually fixed |
| Example | x86 (Intel/AMD) | ARM, MIPS, RISC-V |
| Trade-off | Denser code, more complex hardware | Simpler hardware, needs more instructions to do the same job |

There are other, more exotic categories too:
- **VLIW/EPIC** (Very Long Instruction Word / Explicitly Parallel Instruction Computing) — the *compiler*, not the hardware, decides which instructions run in parallel.
- **MISC/OISC** (Minimal/One-Instruction Set Computer) — theoretical, extremely minimal designs, rarely used commercially.

### What's inside an instruction?

A typical instruction has:
- An **opcode** — the operation itself (e.g., "add")
- **Operands** — what to operate on (registers, constants, or memory addresses)
- **Addressing modes** — the "rules" for how to interpret those operands

### How many operands does an instruction take?

This is a fun way to compare ISAs. Consider computing `C = A + B`:

- **0-operand (stack machine):** `push A; push B; add; pop C` — everything happens via an implicit stack.
- **1-operand (accumulator machine):** `load A; add B; store C` — there's one implicit "working register."
- **2-operand:** `move A→C; add B→C` — common in CISC/RISC hybrids.
- **3-operand:** `add A, B → C` — one instruction does it all directly (common in modern RISC like ARM, MIPS, RISC-V), keeping A, B, and C all available in registers afterward for reuse.

More operands per instruction generally means fewer instructions needed, but each instruction becomes "wider" (needs more bits to encode).

### Instruction length & code density

- **Fixed-length instructions** (typical of RISC) are simpler for hardware to decode and pipeline efficiently.
- **Variable-length instructions** (typical of CISC, e.g., x86 instructions range up to 15 bytes) can pack more meaning into less memory space — this is called **code density** — but are trickier to decode quickly.

### Why does this matter to hardware designers?

The ISA choice ripples through everything downstream: how easy it is to pipeline instructions, how much power the chip uses, how big the cache needs to be, and how fast the whole system can run. This connects directly to the next topic.

---

## 2. Microarchitecture

### The core idea

If ISA is the sheet music, **microarchitecture (µarch)** is *how* a specific chip actually plays it — the literal circuitry, the wiring, the internal "assembly line" that executes instructions.

> **Computer architecture = ISA + Microarchitecture**

Two chips can implement the *exact same ISA* (e.g., x86) with *completely different* internal designs — this is exactly the historical relationship between Intel Pentium and AMD Athlon.

### The basic instruction cycle

Every CPU, at heart, repeats these four steps over and over:

1. **Fetch** the instruction (read it from memory)
2. **Decode** it (figure out what it means)
3. **Execute** it (actually do the operation)
4. **Write back** the result

### Key techniques that make CPUs fast

#### a) Pipelining
Instead of finishing all 4 steps for one instruction before starting the next, pipelining overlaps them — like a factory assembly line. While instruction #2 is being decoded, instruction #1 is already executing, and instruction #3 is being fetched. No single instruction finishes faster, but many more finish *per second* overall.

RISC's simple, uniform instructions made pipelines easier to build efficiently — this was historically a major reason RISC chips outperformed CISC chips at the same clock speed. (Section 4 below covers pipelining in much more depth — it's important enough to deserve its own dedicated walkthrough.)

#### b) Branch Prediction & Speculative Execution
When a program hits an `if` statement (a conditional branch), the CPU doesn't actually know yet which path will be taken. Modern CPUs *guess* (based on past patterns) and start executing down the guessed path before they're sure — this is **speculative execution**. Guess right → free speed boost. Guess wrong → the CPU must throw away that work and start over (a "pipeline flush").

#### c) Superscalar Execution
Rather than having just one of each functional unit (one adder, one multiplier), superscalar CPUs have *multiple* — e.g., two integer units, two floating-point units — so multiple instructions can execute truly simultaneously, not just overlapped in a pipeline.

#### d) Out-of-Order Execution
If instruction #5 is stuck waiting on slow memory, but instruction #8 doesn't depend on it, the CPU can quietly execute #8 first, then reorder the results at the end so it *looks* like everything happened in the original order.

#### e) Register Renaming
Sometimes two unrelated pieces of code both want to use, say, "register 3," creating an artificial bottleneck. Register renaming secretly maps these to different physical registers internally, so the two operations can proceed in parallel instead of waiting on each other.

#### f) Multiprocessing & Multithreading
Instead of just making *one* stream of instructions faster, modern systems run *many* streams in parallel:
- **Multi-core:** multiple full CPUs on one chip.
- **Multithreading:** when one thread stalls (e.g., waiting on memory), the core switches to a different ready thread almost instantly — much cheaper than a full OS-level context switch.

### Why microarchitecture design is hard

Unlike pure "make it fast" goals, microarchitects juggle: chip area/cost, power consumption, heat, manufacturability, and testability — all while sticking strictly to the ISA "contract" they must support.

---

## 3. CPU Cache

### The core problem cache solves

Here's the crux of modern computing: **processors got fast much quicker than memory did.** A CPU today can execute hundreds of instructions in the time it takes to fetch one piece of data from main memory (RAM). If the CPU had to wait for RAM every single time, it would spend most of its life idle.

**Cache is the fix.** It's a small, very fast memory sitting physically close to the CPU core, storing copies of *recently or frequently used* data — so the CPU usually doesn't have to make the slow trip to RAM.

**Analogy:** Imagine writing a research paper. Your desk (cache) holds the 5 books you're actively using. The library (main memory) has everything, but walking there for every single fact would be painfully slow. You keep the books you keep re-using right on your desk.

### Why cache is fast

Cache uses **SRAM** (Static RAM) instead of the **DRAM** used for main memory. SRAM is much faster to access but needs more transistors per bit — making it more expensive and physically larger per byte, which is why you have a *little* fast cache and a *lot* of slower main memory, not the other way around.

### The cache hierarchy: L1, L2, L3

Most modern CPUs use multiple layers, trading off size vs. speed:

| Level | Size (typical) | Speed | Location |
|---|---|---|---|
| **L1** | ~32–192 KB | Fastest | Closest to each core, often split into L1i (instructions) and L1d (data) |
| **L2** | few hundred KB – few MB | Slower than L1 | Per-core or per small cluster |
| **L3** | several MB – tens of MB | Slower still, but way faster than RAM | Usually shared across all cores |

The CPU checks L1 first. If the data isn't there (a **miss**), it checks L2, then L3, and only as a last resort goes to main memory — each step being progressively slower but larger.

### Hit vs. Miss

- **Cache hit:** the data the CPU wants is already in the cache → fast!
- **Cache miss:** it's not there → the CPU must fetch it from a slower level (or from RAM), and while it waits, it may **stall** (sit idle) unless techniques like out-of-order execution give it other useful work to do meanwhile.

### How data gets placed in the cache: Associativity

When new data arrives from memory, where in the cache does it go?

- **Direct-mapped:** each memory location can only go in *one* specific cache slot. Simple and fast, but if two frequently-used pieces of data happen to map to the same slot, they keep evicting each other (a "conflict miss").
- **Fully associative:** data can go *anywhere* in the cache. Best hit rate, but expensive to search (every possible slot must be checked).
- **N-way set associative:** the practical middle ground — data can go into any one of N possible slots. Most real CPUs use this (commonly 4-way, 8-way, or more).

### Cache lines, tags, and addresses

Data isn't cached byte-by-byte — it's cached in chunks called **cache lines** (commonly 64 bytes). When your CPU reads even a single byte, it actually pulls in the whole 64-byte neighborhood, betting that nearby data will likely be needed soon too (this bet is called *locality of reference*, and it usually pays off).

Every cache entry also stores a **tag** (which "remembers" which memory address this data came from) and **flag bits**:
- **Valid bit:** is this cache slot actually holding real data yet?
- **Dirty bit:** has this data been changed since it was loaded, meaning it doesn't match main memory anymore and needs to be written back eventually?

### Write policies — what happens when the CPU changes data?

- **Write-through:** every write immediately updates both cache *and* main memory. Simple, but generates a lot of memory traffic.
- **Write-back:** writes only update the cache immediately (marking the line "dirty"); the update reaches main memory later, only when that cache line gets evicted. Faster, but more complex to keep consistent — especially with multiple CPU cores.

### Replacement policy — what gets kicked out?

When the cache is full and new data needs to come in, something must be evicted. The most common strategy is **LRU (Least Recently Used)** — throw out whatever hasn't been touched in the longest time, betting it's least likely to be needed again soon.

### Virtual memory, page tables & the TLB

Programs don't see real physical RAM addresses. Instead, each program gets its own clean, private-looking **virtual address space** — as if it had the whole memory to itself, starting from address zero. This is **virtual memory**, and it's managed by the operating system together with a piece of hardware called the **MMU (Memory Management Unit)**.

**Why bother?** Virtual memory gives you:
- **Isolation:** one program can't accidentally (or maliciously) read/write another program's memory.
- **Simplicity:** programmers/compilers don't need to know or care where in physical RAM their data actually lives.
- **Flexibility:** the OS can move data around in physical memory, or even temporarily push it out to disk, without the program noticing.

**How the translation works — the page table.** Virtual memory is chopped into fixed-size chunks called **pages** (commonly 4 KB). Physical memory is chopped into the same-size chunks, called **frames**. The **page table** is essentially a lookup dictionary maintained by the OS that says "virtual page #17 currently lives in physical frame #402." Every single memory access a program makes must first be translated through this table: virtual address → page table lookup → physical address.

**The problem:** the page table itself lives in main memory, so a naive translation would require *an extra trip to slow RAM before every single memory access* — effectively doubling memory latency for everything.

**The fix — TLB (Translation Lookaside Buffer):** the TLB is a small, very fast, specialized cache — just like a data or instruction cache, but instead of caching *data*, it caches *recent virtual-to-physical translations*. Since programs tend to keep reusing the same handful of pages for a while (locality of reference again), the TLB has a very high hit rate, and most translations happen in a single fast cycle instead of a slow memory round-trip.

- **TLB hit:** the translation is already cached → instant physical address, memory access proceeds normally.
- **TLB miss:** the CPU has to "walk" the page table in memory to find the translation, which is much slower — then it caches the result in the TLB for next time.

**Where the TLB fits with the cache:** whether a CPU checks the cache using the virtual address or the already-translated physical address (and in what order relative to the TLB lookup) leads to a few cache design flavors — **PIPT**, **VIPT**, **VIVT** — each with different speed/complexity trade-offs. The short version: doing the TLB lookup and the cache-index lookup *in parallel* (rather than one after another) is a common trick to hide translation latency, since the cache index bits and the physical tag bits can often be resolved independently.

### Multi-core caching

With multiple cores on one chip, designers must decide what's shared vs. private:
- L1 is almost always **private** per core (sharing it would slow every core down).
- L2 is sometimes private, sometimes shared between a couple of cores.
- L3 is usually **shared** across all cores — useful for cores that are cooperating on the same task, and it keeps overall chip area efficient.

---

## 4. Instruction Pipelining — Under the Hood

This section zooms into the technique briefly introduced in Section 2, since it's central enough to most architecture interviews and discussions to deserve a full walkthrough of its own.

### The classic five-stage RISC pipeline

Classic RISC designs (MIPS, SPARC, early ARM, the educational DLX) popularized a clean five-stage pipeline that's still the reference model taught today:

| Stage | Name | What happens |
|---|---|---|
| **IF** | Instruction Fetch | Read the next instruction from the instruction cache/memory, using the Program Counter (PC) as the address. |
| **ID** | Instruction Decode & Register Fetch | Figure out what the instruction means, and read up to two source registers from the register file. Branch targets are often computed here too. |
| **EX** | Execute | The ALU does the actual math/logic, or computes a memory address for loads/stores. |
| **MEM** | Memory Access | Loads read from data memory/cache here; stores write to it here. (Instructions that don't touch memory just pass through this stage unused.) |
| **WB** | Write Back | The result is written back into the register file. |

At any given moment, five *different* instructions can be sitting in these five stages simultaneously — one being fetched, one being decoded, one executing, one accessing memory, and one writing back — like five cars on different sections of the same assembly line.

**A quick note on RISC vs. CISC decode:** RISC pipelines have essentially *no microcode* — decoding is simple combinational logic straight off the instruction bits. This is only possible because RISC instructions are simple and uniform to begin with; it's also *why* fewer bits are left over for things like register indices in CISC, since CISC instructions spend more bits describing complex operations.

### Why pipelining works — and its trade-offs

- **Throughput vs. latency:** any single instruction still takes 5 cycles start-to-finish (its *latency* doesn't improve) — but because stages overlap, the *processor as a whole* can finish roughly one instruction every cycle once the pipeline is full (its *throughput* improves dramatically).
- **A pipelined processor is usually more complex** than an equivalent non-pipelined ("multicycle") one — more registers between stages, more control logic — but it's typically far more energy- and gate-efficient *per instruction*, because the same execution hardware stays busy almost continuously instead of sitting idle for most of each instruction's lifetime.
- **Deeper pipelines (more stages)** let each individual stage do less work, so the logic can run at a higher clock speed — famous examples include the Pentium 4's 20-stage pipeline, and later "Prescott"/"Cedar Mill" cores which pushed to 31 stages. The trade-off: more stages means a bigger penalty whenever something goes wrong and the pipeline has to be flushed (see Control Hazards below).

### Hazards — when pipelining goes wrong

A **hazard** is any situation where overlapping instruction execution would produce an incorrect result if nothing were done about it. There are three classic categories:

#### 1. Structural hazards
Two instructions in the pipeline at the same time want to use the *same piece of hardware* simultaneously (e.g., both wanting the ALU in the same cycle). **Fix:** duplicate the contended hardware — e.g., give the decode stage its own dedicated adder for computing branch targets, so it doesn't have to fight the Execute stage's ALU for the same cycle.

#### 2. Data hazards
An instruction needs a value that a *previous, still-in-flight* instruction hasn't finished producing yet. Classic example:
```
SUB r3, r4 -> r10     ; computes r10
AND r10, r3 -> r11    ; needs r10 right away
```
By the time `AND` reaches Decode and wants to read `r10`, `SUB` hasn't written it back yet — the value in the register file is still stale. Two main fixes:

- **Bypassing / operand forwarding:** wire the *just-computed* result directly from later pipeline stages back into the ALU's input for the next instruction, skipping the register file entirely for that one cycle. This handles most cases with zero performance loss.
- **Pipeline interlock (stalling):** when forwarding isn't possible — classically, right after a **load** instruction, since the loaded value isn't available until the Memory stage completes — the pipeline detects the hazard and **stalls**, inserting a **bubble** (an idle "do-nothing" cycle) until the data is actually ready. This is where the historical MIPS acronym comes from: *Microprocessor without Interlocked Pipeline Stages* — early MIPS designs pushed this problem onto the *compiler* instead, which had to insert `NOP` instructions itself to guarantee correctness.

#### 3. Control hazards
Caused by branches and jumps. The pipeline doesn't actually know whether a conditional branch will be taken (or where an indirect jump goes) until that instruction has been resolved — but by then, the pipeline has often already fetched one or more *following* instructions speculatively. If the guess was wrong, that fetched work is wasted and must be flushed.

Common strategies to deal with this:
- **Predict Not Taken:** always keep fetching sequentially; if the branch *is* taken, throw away the wrongly-fetched instruction (small penalty).
- **Branch Likely:** only actually execute the instruction right after the branch if the branch is taken (used with delay slots).
- **Branch Delay Slot:** the ISA explicitly defines that the instruction immediately after a branch always executes regardless of the branch outcome — shifting the burden onto the compiler to fill that slot usefully. (Controversial in hindsight — see below.)
- **Branch Prediction:** guess (using history of past outcomes) whether the branch will be taken and where it goes, and speculatively fetch down that path — flushing and restarting if the guess turns out wrong. This is what virtually all modern high-performance CPUs use, since delay slots don't scale well to superscalar, multi-issue designs.

**Why delay slots fell out of favor:** they complicate the ISA's semantics (a jump takes effect *after* the next instruction, not immediately), they're hard for compilers to fill usefully (often forcing wasted `NOP`s anyway), and they interact badly with exceptions — an excepting instruction sitting in a delay slot creates two different "addresses" (where the exception happened vs. where execution should resume), which has historically been a persistent source of subtle bugs.

### Exceptions and precise interrupts

Exceptions (like arithmetic overflow or a TLB miss) are resolved later than branches — typically at the Write-back stage, not Decode — because the CPU needs a **precise exception**: a guarantee that *every* instruction before the excepting one has fully completed, and *nothing* from the excepting instruction onward has taken effect. Achieving this cleanly is one of the reasons results are committed to the register file (and store buffers) strictly in program order, even in a pipelined design.

### Cache misses inside the pipeline

When an instruction or data cache miss happens mid-pipeline, the CPU has to pause everything until the needed data arrives from a slower level. Two common implementation strategies:
- A **global stall signal** that freezes every pipeline stage in place — simple conceptually, but the signal has to reach a lot of hardware very quickly, which can itself become a speed bottleneck.
- **Reusing the exception mechanism** — treat the cache miss like a special exception, invalidate everything currently in flight, and restart the offending instruction once the data has arrived.

---

## 5. Interview-Ready "Why" Answers

Short, direct answers to the classic conceptual questions:

**Why do modern CPUs need branch prediction?**
Because pipelines are deep, and a conditional branch's outcome isn't known until it's been resolved partway down the pipeline — by which point several following instructions have already been fetched. Without a prediction, the CPU would have to *stall and wait* for every single branch before fetching anything further, which (since roughly 1 in 5 instructions is a branch) would waste a huge fraction of the pipeline's potential throughput. A good predictor lets the CPU keep speculatively fetching and executing down the *likely* path, so it's usually right and rarely pays the flush penalty.

**Why do pipeline stalls happen?**
Stalls happen when the next instruction genuinely cannot proceed correctly yet — most often a **data hazard** (it needs a value a prior in-flight instruction hasn't produced/written back yet, and that value can't be forwarded in time, as with a load-then-use), a **structural hazard** (contention for the same hardware unit), or a **control hazard** (waiting to resolve a branch, or recovering from a mispredicted one). The pipeline inserts a "bubble" — an idle cycle — until it's safe to continue.

**Why does out-of-order execution improve IPC (instructions per cycle)?**
Because in-order pipelines are forced to stall on a stuck instruction even when *later, independent* instructions are perfectly ready to run. Out-of-order execution lets the CPU look ahead in the instruction stream, find instructions with no unmet dependencies, and execute *those* while the stuck one waits (e.g., on a cache miss or a long-latency multiply) — then reorders the results at the end so the program's observable behavior is unchanged. This keeps the execution units busy far more of the time, directly raising the number of instructions completed per cycle.

---

## 6. Tying It All Together

Here's the mental model to walk away with:

1. **ISA** defines *what* the CPU must be able to do — the instructions, registers, and rules — so that software written once can run on many different chips.
2. **Microarchitecture** is *how* a specific chip actually implements that ISA internally — using techniques like pipelining, branch prediction, out-of-order execution, and multiple cores to go as fast as possible while obeying the ISA's rules.
3. **Pipelining** is the specific assembly-line technique that lets a CPU work on several instructions at once — and the hazards (structural, data, control) are exactly what happens when that overlap breaks the illusion of one-instruction-at-a-time execution, requiring bypassing, stalling, or prediction to fix.
4. **The memory system — cache, TLB, and virtual memory** — is microarchitecture's answer to the fact that memory is far slower than the CPU itself: caches hide RAM latency for data and instructions, while the TLB does the same trick specifically for address translation.

A helpful way to remember the relationship: **the ISA is the "what," the microarchitecture is the "how," pipelining is the assembly line that "how" runs on, and the memory system (cache + TLB) is what keeps that assembly line fed fast enough to be worth having.**

### Quick reference: core vocabulary

| Term | One-line definition |
|---|---|
| ISA vs. Microarchitecture | ISA = the instruction "contract"; microarchitecture = the actual circuit implementation of it |
| RISC vs. CISC | RISC = few, simple, fixed-length instructions; CISC = many, powerful, variable-length instructions |
| Pipeline hazard | Any situation where overlapping instructions would give a wrong answer if left unhandled |
| Branch prediction | Guessing a branch's outcome ahead of time to avoid stalling the pipeline |
| Superscalar execution | Issuing more than one instruction per cycle using duplicated execution units |
| Out-of-order execution | Executing ready instructions ahead of stalled earlier ones, then reordering results |
| Register renaming | Mapping the same "named" register to different physical registers to remove false dependencies |
| Speculative execution | Executing down a predicted path before knowing for certain it's correct |
| Cache line | The fixed-size chunk (commonly 64 B) that memory is cached in as a unit |
| Cache miss | Requested data isn't in the cache, forcing a slower fetch from a lower level or main memory |
| Write-through / Write-back | Update memory immediately on every write, vs. only when the cache line is evicted |
| Associativity | How many possible cache slots a given memory address is allowed to map to |
| TLB | A small cache of recent virtual-to-physical address translations |
| Virtual memory / Page table | The private per-program address space, and the OS-maintained lookup table that maps it to real physical memory |
