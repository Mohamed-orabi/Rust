# Concurrency Primitives and Lock-Free Structures
## The Exhaustive Edition — Every Claim Has an Example

---

## Table of Contents

1. [Foundations: What's Really Happening Inside the CPU](#1-foundations-whats-really-happening-inside-the-cpu)
2. [Memory Models and Memory Ordering, Mechanics-First](#2-memory-models-and-memory-ordering-mechanics-first)
3. [Atomic Operations, Instruction by Instruction](#3-atomic-operations-instruction-by-instruction)
4. [Lock-Based Primitives, Disassembled](#4-lock-based-primitives-disassembled)
5. [Higher-Level Synchronization, Step by Step](#5-higher-level-synchronization-step-by-step)
6. [Lock-Free Programming Theory, With Real Code Paths](#6-lock-free-programming-theory-with-real-code-paths)
7. [The ABA Problem, Walked Through Cycle by Cycle](#7-the-aba-problem-walked-through-cycle-by-cycle)
8. [Memory Reclamation, Mechanism by Mechanism](#8-memory-reclamation-mechanism-by-mechanism)
9. [Lock-Free Data Structures, Built From Scratch](#9-lock-free-data-structures-built-from-scratch)
10. [Wait-Free Structures, With Helping Mechanics](#10-wait-free-structures-with-helping-mechanics)
11. [Performance, Measured in Cycles and Cache Lines](#11-performance-measured-in-cycles-and-cache-lines)
12. [Interview Traps, Each With a Worked Scenario](#12-interview-traps-each-with-a-worked-scenario)

---

## 1. Foundations: What's Really Happening Inside the CPU

### 1.1 The lie that `x = x + 1` is one operation

**The surface claim:** `counter = counter + 1` is a single instruction.

**Between the lines:** It is *never* one instruction at the machine level on any general-purpose CPU when `counter` lives in memory. The CPU has registers; arithmetic happens only in registers. So even on a single-threaded program with no concurrency at all, the CPU performs three steps. The reason this matters for concurrency is that *another thread can be scheduled between any two of those steps*, and the OS scheduler can preempt at instruction boundaries.

**Concrete example — disassembling `counter++` in C#:**

```csharp
private static int counter;
counter++;
```

The JIT emits (x86-64):
```
mov  eax, [counter]    ; step 1: load
add  eax, 1            ; step 2: add
mov  [counter], eax    ; step 3: store
```

**Concrete example — the lost update, made painful:**

Imagine two threads each running `counter++` 1,000,000 times. You'd expect the final value to be 2,000,000. In practice you'll see numbers like 1,247,883. Try this yourself:

```csharp
class Program {
    static int counter = 0;
    static void Main() {
        var t1 = new Thread(() => { for (int i = 0; i < 1_000_000; i++) counter++; });
        var t2 = new Thread(() => { for (int i = 0; i < 1_000_000; i++) counter++; });
        t1.Start(); t2.Start(); t1.Join(); t2.Join();
        Console.WriteLine(counter);  // Almost never 2,000,000
    }
}
```

**Hidden detail:** The reason it's not always wildly wrong is that on many iterations, neither thread is preempted in the critical window. You only lose updates when both threads happen to execute their three steps in interleaved fashion. The probability per iteration is small, but at a million iterations it accumulates.

**The fix, also concrete:**
```csharp
Interlocked.Increment(ref counter);  // emits `lock inc [counter]` — single atomic instruction
```

### 1.2 Three layers of reordering — each with its own example

Your code does not run in source order. There are three independent reorderers, and you must understand each as a separate phenomenon.

#### Layer 1: Compiler reordering

**What it is:** The compiler can rearrange your code as long as a *single-threaded* observer can't tell.

**Concrete example — a real reordering you can observe:**

Source code in C:
```c
int x = 0, y = 0;

void writer() {
    x = 1;     // statement A
    y = 2;     // statement B
}
```

If you compile with `gcc -O2`, the compiler may emit `y` first because it has more registers free, or because it's filling a pipeline bubble. Look at the assembly with `gcc -O2 -S writer.c`:

```asm
writer:
    mov DWORD PTR y[rip], 2
    mov DWORD PTR x[rip], 1
    ret
```

**Hidden detail — why the compiler is "allowed" to do this:** the C abstract machine specifies that within a single thread, you can't observe the order of `x = 1; y = 2;` unless you read them. So the compiler treats them as commutative. The instant another thread reads `x` and `y`, this assumption breaks — but the compiler doesn't know about other threads unless you tell it (via `volatile`, `atomic`, or memory barriers).

#### Layer 2: CPU out-of-order execution

**What it is:** Even if the compiled instructions are in order A, B, the CPU executes them out of order using its reorder buffer (ROB).

**Concrete example — an Intel Skylake pipeline:**

The CPU dispatches up to 4 instructions per cycle into 8 execution ports. If instruction A is a memory load that misses L1 cache (300 cycles), and instruction B is an add on a register, the CPU executes B first while waiting for A. From a single-threaded perspective, the result is identical because the CPU retires instructions in order. From another core's perspective, B's *side effect* (a store to memory) became visible before A's.

**Concrete example you can reproduce — Dekker's anomaly on x86:**

```c
// Two threads, two flags
int x = 0, y = 0, r1, r2;

void thread_a() { x = 1; r1 = y; }  // store x, then load y
void thread_b() { y = 1; r2 = x; }  // store y, then load x
```

You'd think at least one of `r1` or `r2` must be 1 (since one store always precedes the other in the global order). But on x86, you can observe `r1 == 0 && r2 == 0`. Why? Each store sits in the **store buffer** before reaching the cache. The load on the same core can satisfy from the store buffer, but the *other core* doesn't see the store until it drains. So thread A reads `y` before `y = 1` is visible globally, and vice versa.

This is reproducible. Run a tight loop of this experiment and you'll see the (0,0) outcome at single-digit-percent rates.

**Hidden detail:** The fix is `mfence` between the store and the load on each thread, which drains the store buffer.

#### Layer 3: Cache coherency delays

**What it is:** Each core has its own L1 cache. When core 0 writes to address X, core 1's copy is invalidated via MESI protocol. The invalidation takes time — typically 30-100 cycles.

**Concrete example — a write that "doesn't happen" for 50ns:**

Core 0 executes `flag = 1`. The store enters core 0's store buffer. The cache line holding `flag` is in **Modified** state in core 0's L1. Core 1 is spinning on `while (!flag) {}`. Core 1's L1 has the line in **Shared** state (stale).

Sequence:
1. Cycle 0: Core 0 executes the store, writes to store buffer
2. Cycle 5: Store buffer drains to L1, line transitions to Modified, MESI sends Invalidate to core 1
3. Cycle 35: Core 1 receives Invalidate, marks line Invalid
4. Cycle 36+: Core 1's next load misses L1, must fetch from L2 or LLC, getting the new value

Until step 4, core 1 sees `flag == 0`. There's a window of ~30-40 cycles where core 0 has "written" but core 1 hasn't seen it. Multiplied across thousands of operations, this is why naive busy-waits and polling aren't free.

### 1.3 Why C# devs get bitten — a real bug pattern

**Surface claim:** "On x86, my code works without barriers."

**Between the lines:** x86 has a strong memory model that hides many bugs. ARM (your phone, M1/M2 Mac, AWS Graviton, Azure Ampere) does not. Code that ships fine on developer laptops fails in production on ARM cloud instances.

**Concrete example — a bug that survives x86 testing:**

```csharp
class Singleton {
    private static Singleton _instance;
    private int _initialized;

    public static Singleton Get() {
        if (_instance == null) {
            lock (typeof(Singleton)) {
                if (_instance == null) {
                    var s = new Singleton();
                    s._initialized = 42;
                    _instance = s;     // BUG: writes to s and write to _instance can be reordered
                }
            }
        }
        return _instance;
    }
}
```

**On x86:** Stores are released in program order (TSO). So writing `s._initialized = 42` always becomes visible before `_instance = s`. Other threads that see a non-null `_instance` always see `_initialized == 42`. The bug is invisible.

**On ARM:** No such guarantee. Another thread can see `_instance != null` while `_initialized` still reads as 0, because the writes to `s._initialized` and `_instance` aren't ordered in the global sense without a barrier.

**Fix:**
```csharp
private static Singleton _instance;
// ...
Volatile.Write(ref _instance, s);  // release semantics — prevents the reordering
```

Or simpler, use `Lazy<Singleton>` which handles this internally.

---

## 2. Memory Models and Memory Ordering, Mechanics-First

### 2.1 What a memory model actually specifies

**Surface claim:** "A memory model is a contract."

**Between the lines:** A memory model is the set of *legal executions* of a multithreaded program. Given source code, a memory model tells you which observations are possible and which are forbidden. It's a mathematical relation over operations: which operations can a thread "see" in which order.

**Concrete example — the C++ memory model in action:**

```cpp
std::atomic<int> x{0}, y{0};
int r1, r2;

// Thread 1
x.store(1, std::memory_order_relaxed);
r1 = y.load(std::memory_order_relaxed);

// Thread 2
y.store(1, std::memory_order_relaxed);
r2 = x.load(std::memory_order_relaxed);
```

**What the C++ memory model says:** `(r1=0, r2=0)` is *legal*. The relaxed ordering provides no constraint between operations on different variables. The compiler is free to reorder, the CPU is free to reorder — this is allowed.

**What it would forbid:** if both stores were `memory_order_seq_cst`, then `(0,0)` is forbidden — there must be a global total order in which one store precedes the other, and the corresponding load must see it.

This is the mathematics of memory models: which 4-tuples `(r1, r2)` are legal under which orderings.

### 2.2 Relaxed ordering — what it actually means

**Surface claim:** "Relaxed only guarantees atomicity."

**Between the lines:** Relaxed says: this read/write is atomic (no torn values), and the compiler/CPU may reorder it freely with surrounding operations. Two relaxed operations on the *same* atomic variable are still ordered (this is called "modification order" — every atomic has a single total order of modifications).

**Concrete example — relaxed counter, why it's safe:**

```rust
use std::sync::atomic::{AtomicU64, Ordering};

static REQUESTS: AtomicU64 = AtomicU64::new(0);

fn handle_request() {
    do_actual_work();
    REQUESTS.fetch_add(1, Ordering::Relaxed);  // pure counter
}

fn print_stats() {
    println!("requests: {}", REQUESTS.load(Ordering::Relaxed));
}
```

**Why relaxed is fine here:** You don't care about ordering relative to other variables. Each `fetch_add` is atomic — no two threads will both think they got the same number. The total count is correct. You might read a slightly-stale value in `print_stats`, but that's acceptable for a metric.

**Concrete example — relaxed is wrong:**

```rust
static FLAG: AtomicBool = AtomicBool::new(false);
static mut DATA: i32 = 0;

// Thread A
unsafe { DATA = 42; }
FLAG.store(true, Ordering::Relaxed);  // BUG

// Thread B
while !FLAG.load(Ordering::Relaxed) { }  // BUG
unsafe { println!("{}", DATA); }  // could print 0
```

**Why this is wrong:** Relaxed doesn't order `DATA = 42` with `FLAG.store`. The compiler could reorder them. The CPU could reorder them. Thread B could see `FLAG == true` while `DATA` is still 0.

### 2.3 Acquire and Release — the publish/subscribe pattern

**Surface claim:** "Acquire/release establishes happens-before."

**Between the lines:** Imagine a thread "publishes" a value with a release store. Every memory operation *prior* to the release is bundled with the store. When another thread does an acquire load and *sees that value*, it inherits the bundle — every prior write is now visible to it. This is called **synchronizes-with**: release-A synchronizes-with acquire-B if B reads the value A wrote.

**Concrete example — message passing through a flag:**

```rust
use std::sync::atomic::{AtomicBool, Ordering};
use std::cell::UnsafeCell;

struct Channel { data: UnsafeCell<i32>, ready: AtomicBool }
unsafe impl Sync for Channel {}

static CH: Channel = Channel {
    data: UnsafeCell::new(0),
    ready: AtomicBool::new(false),
};

// Producer
unsafe { *CH.data.get() = 42; }                 // ① non-atomic write
CH.ready.store(true, Ordering::Release);        // ② release: bundles ①

// Consumer
while !CH.ready.load(Ordering::Acquire) {}      // ③ acquire: sees ②, inherits bundle
unsafe { assert_eq!(*CH.data.get(), 42); }      // ④ guaranteed to see 42
```

**The mechanics:** The release at ② is a barrier *upward* for the producer — no operation above ② can sink below it. The acquire at ③ is a barrier *downward* for the consumer — no operation below ③ can rise above it. The synchronization happens because ③ sees the value written by ②.

**Hidden detail — the bundle isn't magic:** It's enforced by hardware barriers. On x86, a release store is just a regular store (TSO already gives release semantics). On ARM, it emits `STLR` (Store-Release Register), which is a special instruction. An acquire load on ARM emits `LDAR` (Load-Acquire Register). Look at the disassembly to confirm.

**Concrete example — what happens if the load doesn't see the store:**

```rust
// Suppose another producer overwrote ready to false, then back to true.
// The synchronization edge depends on which store the load reads.
// If the load reads the value from producer #2, it synchronizes with producer #2's release,
// not producer #1's.
```

This is critical: **synchronizes-with is per-value, not per-variable**. The acquire reader synchronizes with whichever release writer wrote the value it read.

### 2.4 AcqRel and SeqCst — when you need both directions

**Surface claim:** "AcqRel is for read-modify-write."

**Between the lines:** A read-modify-write atomically reads, computes, and writes. It needs to act like an acquire (for the read part — see prior writes) and a release (for the write part — make subsequent reads see this). AcqRel is exactly this combination.

**Concrete example — a CAS in a lock-free stack:**

```rust
loop {
    let head = self.head.load(Ordering::Acquire);    // see prior pushes
    unsafe { (*new_node).next = head; }
    if self.head.compare_exchange_weak(
        head, new_node,
        Ordering::AcqRel,                            // both: see priors AND publish
        Ordering::Acquire,                           // failure: still need to see priors
    ).is_ok() { return; }
}
```

**Why both:** When the CAS succeeds, we want the *write* to be visible to the next thread that reads `head` (release). We also want to *see* the up-to-date `head` value (acquire) — though the load already covered this. The AcqRel makes the success case correct.

**SeqCst — when you need a global total order:**

**Concrete example — Dekker's algorithm requires SeqCst:**

```rust
static FLAG_A: AtomicBool = AtomicBool::new(false);
static FLAG_B: AtomicBool = AtomicBool::new(false);

// Thread A
FLAG_A.store(true, Ordering::SeqCst);
if !FLAG_B.load(Ordering::SeqCst) { /* enter critical section */ }

// Thread B
FLAG_B.store(true, Ordering::SeqCst);
if !FLAG_A.load(Ordering::SeqCst) { /* enter critical section */ }
```

**Why SeqCst is needed:** With acquire/release alone, both threads can enter the critical section. The store and load on each thread aren't ordered against each other (they're on different variables, so release/acquire doesn't bind them). SeqCst forces a global total order across all SeqCst operations on all threads, so at least one thread will see the other's flag.

**Cost:** SeqCst on x86 emits a full `MFENCE` after the store (or uses `XCHG` which is implicitly a fence). On ARM, it uses `DMB ISH` (Data Memory Barrier, Inner Shareable). Both cost ~30 cycles.

### 2.5 C# memory model concretely

**Surface claim:** "C# `volatile` provides acquire/release."

**Between the lines:** C#'s `volatile` keyword on a field means: reads are acquire, writes are release. It does *not* make compound operations atomic. `volatile int x; x++;` is *not* atomic — it's a volatile read, a non-atomic increment, and a volatile write.

**Concrete example — a real C# bug from this misunderstanding:**

```csharp
class RateLimiter {
    private volatile int _count = 0;
    public void Increment() { _count++; }  // BUG: not atomic
}
```

Even though `_count` is volatile, `_count++` translates to:
```
ldfld _count    ; volatile load (acquire)
ldc.i4.1
add
stfld _count    ; volatile store (release)
```

Two threads can both load 5, both add 1, both store 6. Lost update.

**Fix:**
```csharp
private int _count = 0;  // doesn't need volatile
public void Increment() { Interlocked.Increment(ref _count); }
```

**Hidden detail:** `Interlocked` operations in C# are always full sequential consistency — there's no relaxed version. Java's `AtomicInteger` is similarly always SeqCst. Rust and C++ let you choose. This means C# code is "safer by default" but pays SeqCst cost everywhere.

### 2.6 x86 TSO vs ARM weak ordering — concretely

**Surface claim:** "x86 has strong memory order, ARM is weak."

**Between the lines:** x86's TSO allows exactly one reordering: a load can pass an earlier store to a different address. ARM allows almost any reordering except dependent ones (a load can't be reordered before a store it depends on through addressing).

**Concrete example — same code, different behaviors:**

```c
// Shared
int x = 0, y = 0;

// Thread 1
x = 1;
int r1 = y;

// Thread 2
y = 1;
int r2 = x;
```

**On x86:** `(r1=0, r2=0)` is observed at ~1% rate due to the store-buffer-bypass-load reordering.

**On ARM:** `(r1=0, r2=0)` is observed at ~30% rate. Many more reorderings are legal, including the store-store reorder *across* threads (one core sees stores in a different order than another).

**Real-world consequence:** the `litmus7` tool by Maranget et al. lets you run these tests. A 2012 paper "Synchronising C/C++ and POWER" found bugs in production code that had run for years on x86 because they only manifested on POWER (similar weak ordering to ARM).

---

## 3. Atomic Operations, Instruction by Instruction

### 3.1 What "atomic" means at the hardware level

**Surface claim:** "Atomic operations complete indivisibly."

**Between the lines:** Indivisibility is enforced by the hardware locking the cache line during the operation. On x86, the `LOCK` prefix tells the CPU to assert a lock on the cache line (or, on older CPUs, the entire bus). No other core can read or modify that line until the locked instruction completes.

**Concrete example — what `LOCK XADD` does:**

```asm
lock xadd dword [counter], eax
```

The CPU:
1. Acquires the cache line for `counter` in **Modified** state (evicting it from other caches via MESI)
2. Reads the current value into a temp
3. Adds `eax` (atomically, as part of the locked operation)
4. Writes the result back
5. Returns the old value in `eax`
6. Releases the cache line

During steps 1-5, no other core can touch this cache line. Other cores trying to access it stall.

**Cost:** ~25 cycles uncontended, ~100-500 cycles under contention. Compare to a regular `add`, which is 1 cycle.

### 3.2 Compare-And-Swap (CAS) — the universal primitive

**Surface claim:** "CAS is foundational for lock-free."

**Between the lines:** CAS solves the consensus problem. Given N threads and a shared atomic, CAS lets exactly one thread "win" a race. Herlihy's 1991 paper proved CAS has consensus number infinity — it can solve consensus for any number of threads. Atomic load/store has consensus number 1. This is *the* mathematical reason CAS is special.

**Concrete example — implementing a max():**

```rust
fn atomic_max(v: &AtomicU64, candidate: u64) {
    let mut current = v.load(Ordering::Relaxed);
    loop {
        if candidate <= current { return; }
        match v.compare_exchange_weak(current, candidate, Ordering::AcqRel, Ordering::Relaxed) {
            Ok(_) => return,
            Err(actual) => current = actual,  // someone else updated; check again
        }
    }
}
```

**Walk-through with values:**
- `v` starts at 5, three threads call `atomic_max` with 7, 9, 8 simultaneously.
- Thread (7) reads current=5, computes "7 > 5", attempts CAS(5, 7).
- Thread (9) reads current=5, computes "9 > 5", attempts CAS(5, 9). Both threads race the CAS.
- Suppose thread (7) wins: v=7. Thread (9)'s CAS fails, returns Err(7). Thread (9) loops, sees current=7, "9 > 7", tries CAS(7, 9). Wins: v=9.
- Thread (8) sees current=9, "8 ≤ 9", returns without writing.

Final `v = 9`. Correct.

### 3.3 `compare_exchange` vs `compare_exchange_weak` — when to use which

**Surface claim:** "Weak can fail spuriously."

**Between the lines:** On ARM and POWER, the native primitive is LL/SC (Load-Linked / Store-Conditional). The SC fails if the cache line was touched since the LL — even if the value matches. To implement strong CAS on ARM, you need a retry loop *inside* the CAS itself. Weak CAS exposes this retry to you, which is a win when *you're already in a retry loop*.

**Concrete example — weak inside an explicit loop:**

```rust
// GOOD: weak inside loop. ARM emits one LL/SC pair per iteration.
loop {
    let cur = atomic.load(Ordering::Relaxed);
    let new = compute(cur);
    if atomic.compare_exchange_weak(cur, new, AcqRel, Relaxed).is_ok() {
        break;
    }
}

// SUBOPTIMAL: strong inside loop. ARM emits an inner LL/SC retry inside compare_exchange,
// then your outer loop also retries. Two layers of retries.
loop {
    let cur = atomic.load(Ordering::Relaxed);
    let new = compute(cur);
    if atomic.compare_exchange(cur, new, AcqRel, Relaxed).is_ok() {
        break;
    }
}
```

**When to use strong:** when you're not in a retry loop and want a single boolean answer.

```rust
// Strong is correct here: try once, give up if conflict.
fn try_set_once(atomic: &AtomicBool) -> bool {
    atomic.compare_exchange(false, true, AcqRel, Relaxed).is_ok()
}
```

### 3.4 Fetch-and-add, fetch-or, fetch-and — the RMW family

**Surface claim:** "These are atomic read-modify-writes."

**Between the lines:** Each of these maps to a specific hardware instruction. On x86, you have `LOCK XADD`, `LOCK OR`, `LOCK AND`, `LOCK XCHG`. On ARM, all are implemented via LL/SC loops. Fetch-add is the most common because counters are everywhere.

**Concrete example — using fetch_or to claim a slot in a bitmap:**

```rust
static SLOTS: AtomicU64 = AtomicU64::new(0);

fn claim_slot() -> Option<u32> {
    loop {
        let cur = SLOTS.load(Ordering::Relaxed);
        if cur == u64::MAX { return None; }  // all slots taken
        let bit = (!cur).trailing_zeros();  // first free bit
        let mask = 1u64 << bit;
        let prev = SLOTS.fetch_or(mask, Ordering::AcqRel);
        if prev & mask == 0 {
            return Some(bit);  // we set the bit; we own this slot
        }
        // someone else set it; retry
    }
}
```

**Walk-through:** Suppose `SLOTS = 0b0011`. Thread wants slot. `(!cur).trailing_zeros() = 2`. Mask = `0b0100`. `fetch_or(0b0100)` returns previous `0b0011`. `0b0011 & 0b0100 == 0`, so we own slot 2. SLOTS is now `0b0111`.

If two threads race for slot 2: both compute mask `0b0100`. Both call fetch_or. The first returns prev `0b0011` (claims slot). The second returns prev `0b0111` (`0b0111 & 0b0100 != 0` — failed; loop). The second retries, finds slot 3.

### 3.5 LL/SC — the ARM/RISC-V primitive

**Surface claim:** "LL/SC is more flexible than CAS."

**Between the lines:** LL/SC operates on a *reservation* — when you LL, the cache line is marked reserved. Any *write* to that line by *any* core (including yourself, even via unrelated stores!) invalidates the reservation. Then SC fails. This naturally avoids ABA at the hardware level.

**Concrete example — ARM assembly for fetch_add:**

```asm
loop:
    ldaxr w1, [x0]        ; Load-Acquire Exclusive: reserve cache line
    add   w1, w1, #1
    stlxr w2, w1, [x0]    ; Store-Release Exclusive: succeed only if reservation valid
    cbnz  w2, loop        ; w2 != 0 means SC failed; retry
```

**Hidden detail — SC can fail spuriously:** Context switches, interrupts, cache evictions, or another core simply *reading* the line (in some implementations) can invalidate the reservation. So even if your value didn't change, SC may fail. This is why `compare_exchange_weak` exists — it exposes this.

**Concrete example — ABA-immunity at the LL/SC level:**

```
Initial: [A] (a heap-allocated node)
Thread 1: LL(head) -> reads A, reservation = (head, A)
Thread 2: pops A, pushes B, pushes A back. head = A again.
            But each operation touched the head's cache line.
            Thread 1's reservation was invalidated by thread 2's writes.
Thread 1: SC(head, X) fails (no spurious; reservation gone).
            Thread 1 retries, reads head again, sees the actual current state.
```

CAS would have succeeded because the *value* matches. LL/SC fails because the *cache line was touched*. This is why some lock-free algorithms are easier to write directly in ARM assembly.

---

## 4. Lock-Based Primitives, Disassembled

### 4.1 Mutex internals — the futex fast path

**Surface claim:** "Mutexes use futexes on Linux."

**Between the lines:** A modern mutex has two paths. The **fast path** is pure userspace: a CAS on an atomic. The **slow path** drops into the kernel via `futex(2)` to block. The fast path is ~25ns; the slow path is ~1-3μs (includes a syscall).

**Concrete example — pseudocode for a Linux pthread mutex (simplified):**

```c
struct mutex { atomic_int state; };  // 0=free, 1=locked, 2=locked-with-waiters

void lock(mutex *m) {
    int c;
    // Fast path: try to grab from free
    if ((c = cmpxchg(&m->state, 0, 1)) == 0) return;
    // Slow path: there's contention
    do {
        if (c == 2 || cmpxchg(&m->state, 1, 2) != 0) {
            // Mark as "locked with waiters" and sleep
            futex_wait(&m->state, 2);
        }
    } while ((c = cmpxchg(&m->state, 0, 2)) != 0);
}

void unlock(mutex *m) {
    if (atomic_dec(&m->state) != 1) {
        // We had waiters
        m->state = 0;
        futex_wake(&m->state, 1);  // wake one
    }
}
```

**Walk-through — uncontended:**
- Thread A: `lock` → CAS(0, 1) succeeds. State=1. Returns immediately. 25ns.
- Thread A: `unlock` → atomic_dec returns 1, no waiters. State=0. Returns immediately.

**Walk-through — contended:**
- Thread A: `lock` → CAS(0, 1) succeeds. State=1.
- Thread B: `lock` → CAS(0, 1) fails (state is 1). Tries CAS(1, 2) to mark waiters. State=2.
- Thread B: `futex_wait(state, 2)` → kernel checks state==2, blocks B.
- Thread A: `unlock` → atomic_dec returns 2 (had waiters). Sets state=0. `futex_wake` → kernel wakes B.
- Thread B: wakes from futex_wait, retries CAS(0, 2), succeeds. State=2 (B now owns, with no waiters tracked yet, but next lock will set 2 again).

**Hidden detail — the optimization:** The "state=2 means locked with waiters" is critical. It means uncontended unlock doesn't need a syscall. Only when there are waiters does unlock pay the syscall cost. This makes mutexes basically free in the uncontended case.

### 4.2 Spinlock — when blocking is too expensive

**Surface claim:** "Spinlocks busy-wait."

**Between the lines:** A spinlock burns CPU instead of blocking. This is a win only when (a) the critical section is very short (< ~1μs), and (b) the cost of a context switch (~3-10μs) would exceed the spin time. In userspace, spinning is dangerous because the kernel can preempt the lock holder, leaving spinners burning CPU forever.

**Concrete example — naive spinlock (don't use!):**

```rust
struct BadSpinLock { state: AtomicBool }

impl BadSpinLock {
    fn lock(&self) {
        while self.state.swap(true, Ordering::Acquire) {} 
    }
    fn unlock(&self) {
        self.state.store(false, Ordering::Release);
    }
}
```

**Why this is bad:** `swap` is a write. Every spinning thread is hammering the cache line with writes, causing constant MESI invalidation traffic. With 8 threads spinning, the cache line bounces 8 times per microsecond.

**Concrete example — TTAS (Test-and-Test-and-Set) spinlock:**

```rust
struct GoodSpinLock { state: AtomicBool }

impl GoodSpinLock {
    fn lock(&self) {
        loop {
            // First test: cheap atomic load (cache line stays Shared while unlocked)
            while self.state.load(Ordering::Relaxed) {
                std::hint::spin_loop();  // PAUSE on x86, YIELD on ARM
            }
            // Second test: actual claim attempt
            if self.state.compare_exchange_weak(
                false, true, Ordering::Acquire, Ordering::Relaxed
            ).is_ok() {
                return;
            }
        }
    }
}
```

**Why this is better:** 
- The spinning loop only does loads. A load on an already-cached line is free (1 cycle). The cache line stays in Shared state across all spinning cores — no MESI traffic.
- When the lock is released, the line transitions to Modified on the unlocker. Spinners' copies are invalidated. Each spinner reloads (one cache miss each). One of them wins the CAS; the rest spin again on the now-Modified line.
- `spin_loop` (x86 `PAUSE`) hints the CPU that we're in a spin loop, allowing it to slow down speculative execution and save power.

**Concrete example — when spinlocks deadlock on a single CPU:**

```c
// Single-core system. Two threads, T1 and T2. T1 is high-priority, T2 is low.
T2: spinlock.lock();      // acquires
T2: ... working ...
T1: spinlock.lock();      // spins
// Scheduler: T1 has higher priority, never yields to T2.
// T2 never gets CPU to release the lock. Permanent spin.
```

This is why kernels disable preemption while holding a spinlock. Userspace code shouldn't use spinlocks for this reason.

### 4.3 Reader-Writer Lock

**Surface claim:** "Multiple readers OR one writer."

**Between the lines:** Implementation is subtle. The state is typically a single integer: positive = N readers, negative = writer, 0 = free. Two policy questions:
1. **Reader-priority** vs **writer-priority** vs **fair**: who wins under contention?
2. **Write-bias**: should new readers wait for pending writers?

**Concrete example — a simple RwLock state:**

```rust
// state encoding:
//   0       = free
//   N > 0   = N readers
//   -1      = one writer

fn read_lock(state: &AtomicI32) {
    loop {
        let cur = state.load(Ordering::Acquire);
        if cur >= 0 {
            if state.compare_exchange_weak(cur, cur + 1, Ordering::AcqRel, Ordering::Relaxed).is_ok() {
                return;
            }
        } else {
            std::hint::spin_loop();
        }
    }
}

fn write_lock(state: &AtomicI32) {
    loop {
        if state.compare_exchange_weak(0, -1, Ordering::AcqRel, Ordering::Relaxed).is_ok() {
            return;
        }
        std::hint::spin_loop();
    }
}
```

**Walk-through — reader contention:**
- State = 0. Three threads call `read_lock`.
- All read state=0. All try CAS(0, 1). One wins (state=1). Two fail.
- Two retry: read state=1. CAS(1, 2). One wins (state=2). One fails.
- One retries: read state=2. CAS(2, 3). Wins.
- Final: 3 readers, state=3.

**Concrete example — writer starvation:**

If a writer arrives while readers keep arriving, with the above implementation, the writer can wait forever. New readers see state ≥ 0 and increment. Writer sees state ≥ 1 and spins.

**Fix — writer-bias:** When a writer is waiting, new readers should block. Implementations like `parking_lot::RwLock` use additional state to track pending writers.

**Hidden detail — RwLock is often slower than Mutex:** The reader path involves an atomic CAS *every read*. If your critical section is fast (e.g., reading a single int), the cost of the CAS dominates and you'd be faster with a plain Mutex. RwLock wins when read sections are long enough to amortize the lock overhead and contention is high.

**When to use RwLock:** Readers vastly outnumber writers AND read sections take more than ~1μs.

### 4.4 Recursive (reentrant) mutex

**Surface claim:** "Same thread can lock it twice."

**Between the lines:** Recursive mutexes track the owner thread ID and a recursion count. They cost more (one extra TLS read per lock) and *encourage bad design*. If a function holding a lock calls another function that locks the same mutex, that's usually a code smell — the inner function shouldn't need the lock if the outer already provides exclusivity.

**Concrete example — C# `lock` is recursive:**

```csharp
private readonly object _lock = new();

void Outer() {
    lock (_lock) {
        Inner();  // works because lock is recursive
    }
}

void Inner() {
    lock (_lock) {  // re-enters; recursion count goes to 2
        // ...
    }
}
```

**Why Rust doesn't have this by default:**

```rust
let m = Mutex::new(0);
let g1 = m.lock().unwrap();
let g2 = m.lock().unwrap();  // DEADLOCK — Rust's Mutex is not recursive
```

Rust forces you to either restructure (Inner takes a `&mut T` from the guard) or use `parking_lot::ReentrantMutex` explicitly.

**Hidden detail — recursive mutexes break invariant restoration patterns:** A common pattern is "function temporarily breaks an invariant, restores it before returning." If you reenter holding the lock, the inner caller sees the broken invariant. With non-recursive locks, this can't happen.

### 4.5 Deadlock — the four conditions and how to break them

**Surface claim:** "Coffman conditions are necessary for deadlock."

**Between the lines:** All four must hold simultaneously. Removing any one prevents deadlock. Most prevention strategies target condition 4 (circular wait) because it's the easiest to enforce.

**Concrete example — the classic two-lock deadlock:**

```rust
let a = Mutex::new(0);
let b = Mutex::new(0);

// Thread 1
let _ga = a.lock().unwrap();
std::thread::sleep(std::time::Duration::from_millis(10));
let _gb = b.lock().unwrap();  // waits for thread 2 to release b

// Thread 2
let _gb = b.lock().unwrap();
std::thread::sleep(std::time::Duration::from_millis(10));
let _ga = a.lock().unwrap();  // waits for thread 1 to release a

// Cycle: 1 waits on b (held by 2), 2 waits on a (held by 1). Forever.
```

**Breaking condition 1 (mutual exclusion):** Use lock-free or RCU. Often impossible to remove.

**Breaking condition 2 (hold and wait):** Acquire all locks at once. Hard with arbitrary code.

**Breaking condition 3 (no preemption):** Use try_lock with timeout. If can't get the second lock, release the first and retry.

```rust
loop {
    let ga = a.lock().unwrap();
    match b.try_lock() {
        Ok(gb) => { /* both held */ break; }
        Err(_) => { drop(ga); std::thread::sleep(some_time); }
    }
}
```

**Breaking condition 4 (circular wait):** Define a global lock order. Always acquire locks in that order.

```rust
fn transfer(from: &Account, to: &Account, amount: u64) {
    // Compare addresses to enforce a global order
    let (first, second) = if from.id < to.id { (from, to) } else { (to, from) };
    let _g1 = first.lock.lock().unwrap();
    let _g2 = second.lock.lock().unwrap();
    // ... do transfer
}
```

This prevents the cycle: there's no way for two threads to wait on each other in opposite order.

### 4.6 Priority inversion — the Mars Pathfinder bug

**Surface claim:** "High-priority threads can be blocked by low-priority threads."

**Between the lines:** It's worse than it sounds. The chain is: high-priority thread H waits on a lock held by low-priority thread L. Medium-priority thread M starts running (it's higher than L, so L is preempted), and M doesn't need the lock. M can run forever, starving L, which starves H. This is the **unbounded** form of priority inversion.

**Concrete example — Mars Pathfinder, 1997:**

The Pathfinder rover had a high-priority bus management task (H), a low-priority meteorological data task (L), and a medium-priority communications task (M), all on VxWorks RTOS.
- L acquires a mutex protecting an information bus.
- M (frequent communications work) preempts L.
- H needs the mutex, but it's held by L. H waits.
- M keeps running. L can't run. H is starved.
- Watchdog timer trips, system resets. This happened repeatedly on Mars.

**Fix — priority inheritance:**
When H tries to acquire a mutex held by L, the OS temporarily boosts L's priority to H's. Now L can run (M can no longer preempt it), L finishes its critical section, releases the mutex, and L's priority drops back. H acquires immediately.

**Concrete pseudocode for priority inheritance:**

```
mutex.lock():
    if owner == NULL:
        owner = current_thread
        return
    if current_thread.priority > owner.priority:
        owner.boosted_priority = current_thread.priority  // inherit
        scheduler.reschedule(owner)
    block until owner releases

mutex.unlock():
    owner.priority = owner.original_priority  // restore
    owner = next_waiter or NULL
```

**Hidden detail — semaphores can't inherit:** Semaphores have no owner (any thread can post). So priority inheritance is impossible with semaphores. This is one reason mutexes and semaphores are not interchangeable in real-time systems.

### 4.7 Lock convoys

**Surface claim:** "Many threads queue on a lock."

**Between the lines:** Once a convoy forms, even after the bottleneck is resolved, threads keep colliding because they wake up in lock-step. Each thread releases the lock, the next acquires, releases, the next acquires — but all the previously-waiting threads are now in tight contention on every release.

**Concrete example — a logging convoy:**

```csharp
private static readonly object _logLock = new();

void DoWork() {
    // ... compute ...
    lock (_logLock) {
        _logFile.WriteLine(...);  // disk I/O, ~5ms
    }
}
```

If 100 threads call `DoWork` per second, and each takes 5ms in the lock, you can only sustain 200 lock acquisitions per second. Throughput collapses. Worse: once the convoy forms, threads spend almost all their time waiting on the lock, even when there's "free" CPU.

**Fix:** Buffered logging — each thread writes to a thread-local buffer, a single dedicated thread flushes to disk. Or use a lock-free MPSC queue.

---

## 5. Higher-Level Synchronization, Step by Step

### 5.1 Semaphore — counting mutual exclusion

**Surface claim:** "A semaphore generalizes a mutex."

**Between the lines:** A semaphore tracks an integer count. `acquire()` decrements (waits if zero). `release()` increments (wakes a waiter if any). It has no owner — any thread can release. This is both the strength (flexibility) and weakness (no priority inheritance, easier to misuse).

**Concrete example — a connection pool:**

```rust
use tokio::sync::Semaphore;
use std::sync::Arc;

let pool = Arc::new(Semaphore::new(10));  // 10 db connections

async fn query(pool: Arc<Semaphore>) -> String {
    let _permit = pool.acquire().await.unwrap();
    // We hold a permit. If 10 queries are running, an 11th waits here.
    let result = run_query().await;
    result
    // _permit dropped here, releases back to pool
}
```

**Walk-through:**
- 10 tasks call `query` simultaneously. All acquire permits. Count = 0.
- Task 11 calls `query`. Acquires; count = -1 conceptually; tokio puts it in a wait queue.
- One task finishes, drops permit. Count back to 0. Tokio wakes task 11. Now task 11 holds the permit, 10 tasks total are running.

**Concrete example — semaphore as a binary mutex (don't!):**

```rust
let s = Semaphore::new(1);
let _p = s.acquire().await.unwrap();
// critical section
```

This *works* but lacks the safety properties of a mutex: any task can `add_permits(1)` to release, even if it didn't acquire. With Mutex, the guard ties release to scope.

**Concrete example — producer-consumer with bounded buffer:**

```rust
let empty_slots = Arc::new(Semaphore::new(BUFFER_SIZE));
let filled_slots = Arc::new(Semaphore::new(0));
let buffer = Arc::new(Mutex::new(VecDeque::new()));

// Producer
let _p = empty_slots.acquire().await.unwrap();  // wait for empty slot
buffer.lock().unwrap().push_back(item);
filled_slots.add_permits(1);  // signal consumer

// Consumer
let _p = filled_slots.acquire().await.unwrap();  // wait for filled slot
let item = buffer.lock().unwrap().pop_front().unwrap();
empty_slots.add_permits(1);  // signal producer
```

This is the textbook bounded-buffer pattern using two semaphores. The mutex protects the buffer's internal state; the semaphores manage flow.

### 5.2 Condition variable — wait until predicate

**Surface claim:** "Threads wait for a condition with a mutex."

**Between the lines:** A condvar does *not* check the predicate for you. It's a notification mechanism: it lets you sleep while atomically releasing a mutex, and wakes when signaled. *You* check the predicate. The mutex is required because the predicate involves shared state.

**The atomic release-and-wait is the magic:** if `cv.wait(lock)` released the mutex first, then waited, there'd be a race: between the release and the wait, another thread could check, signal, and the wait would miss it. The condvar API guarantees the release-and-wait is atomic from the perspective of `notify`.

**Concrete example — a simple work queue:**

```rust
use std::sync::{Arc, Mutex, Condvar};

struct Queue {
    mutex: Mutex<Vec<i32>>,
    cvar: Condvar,
}

let q = Arc::new(Queue { mutex: Mutex::new(vec![]), cvar: Condvar::new() });

// Consumer
fn consume(q: Arc<Queue>) {
    let mut v = q.mutex.lock().unwrap();
    while v.is_empty() {
        v = q.cvar.wait(v).unwrap();
        // wait: atomically releases mutex and sleeps
        // returns: re-acquires mutex, returns it
    }
    let item = v.pop().unwrap();
    // ...
}

// Producer
fn produce(q: Arc<Queue>, x: i32) {
    let mut v = q.mutex.lock().unwrap();
    v.push(x);
    q.cvar.notify_one();
    // notify happens while we still hold the mutex; consumer is on the cvar's queue;
    // when we drop our mutex, consumer wakes, re-acquires, and proceeds
}
```

**Walk-through:**
1. Consumer locks mutex, sees v empty. Calls cvar.wait. Releases mutex, sleeps.
2. Producer locks mutex (now free). Pushes 1. Calls notify_one (consumer marked wake-eligible).
3. Producer releases mutex.
4. Consumer wakes, re-acquires mutex, sees v=[1], pops.

**Hidden detail — why the `while`, not `if`:**

```rust
while v.is_empty() { v = q.cvar.wait(v).unwrap(); }
```

Two reasons:
- **Spurious wakeups:** The OS may wake the thread for no reason. POSIX explicitly allows this (it makes implementations easier on certain hardware).
- **Stolen wakeup:** If three consumers are waiting and a producer notifies one, but a *fourth* consumer arrives between notify and wake, the fourth might grab the item. The waking consumer must re-check.

If you write `if`, your code can pop from an empty queue. This is one of the most common condvar bugs.

**Concrete example — `notify_one` vs `notify_all`:**

```rust
// Single-item queue, multiple waiters: notify_one is correct.
// Each waiter only needs one item.
v.push(x);
q.cvar.notify_one();

// State change affects all waiters: notify_all.
*shutdown = true;
q.cvar.notify_all();
// Any number of threads might be waiting on different predicates,
// all of which become true when shutdown=true. Wake them all.
```

### 5.3 Barrier — synchronize N threads at a point

**Surface claim:** "All N threads must arrive."

**Between the lines:** Barriers are used in bulk-synchronous parallel (BSP) algorithms — phases where each thread does its part, then all sync, then next phase. The barrier is reusable: after all N arrive, the next call to `wait` starts a fresh round.

**Concrete example — parallel matrix multiplication with phases:**

```rust
use std::sync::{Arc, Barrier};

const N: usize = 4;
let barrier = Arc::new(Barrier::new(N));

let handles: Vec<_> = (0..N).map(|tid| {
    let b = barrier.clone();
    std::thread::spawn(move || {
        for phase in 0..3 {
            do_phase_work(tid, phase);
            let result = b.wait();  // returns BarrierWaitResult
            if result.is_leader() {
                println!("phase {} complete", phase);  // exactly one thread is leader
            }
        }
    })
}).collect();
```

**Walk-through:**
- All 4 threads start phase 0 work.
- Threads 0, 1 finish quickly, call `wait`. Barrier count goes from 4 down to 2. They sleep.
- Threads 2, 3 finish. Thread 2 calls wait: count → 1, sleeps. Thread 3 calls wait: count → 0, this is the *last* thread.
- Thread 3's wait wakes 0, 1, 2. Thread 3 (the last) is the "leader" — useful for one-time per-phase actions.
- All 4 proceed to phase 1.

**Hidden detail — barrier resets:** A barrier with N=4 always waits for groups of 4. After the first batch passes through, the count resets to 4 for the next batch.

### 5.4 CountdownEvent / Latch — single-use

**Surface claim:** "Counts down to zero."

**Between the lines:** Like a barrier but single-use, and the count can be decremented by *any* number of threads (not necessarily one each). Useful for "wait for N tasks to complete."

**Concrete example — wait for parallel tasks:**

```csharp
using System.Threading;

var latch = new CountdownEvent(5);
for (int i = 0; i < 5; i++) {
    int taskId = i;
    Task.Run(() => {
        DoWork(taskId);
        latch.Signal();  // decrement by 1
    });
}
latch.Wait();  // blocks until count reaches 0
Console.WriteLine("All tasks done");
```

**Concrete example — racing threads to complete:**

```csharp
var latch = new CountdownEvent(1);
var results = new ConcurrentBag<string>();

foreach (var endpoint in endpoints) {
    Task.Run(async () => {
        var data = await Fetch(endpoint);
        results.Add(data);
        latch.Signal();  // first signal opens the gate
    });
}
latch.Wait();
// Returns as soon as ANY one task signals. The "first response wins" pattern.
```

### 5.5 ManualResetEvent / AutoResetEvent

**Surface claim:** "Set and reset signals."

**Between the lines:** These are direct exposures of OS-level event objects. ManualResetEvent: when set, all waiters wake; stays set until you Reset. AutoResetEvent: when set, *exactly one* waiter wakes, then it auto-resets.

**Concrete example — manual reset for "system ready":**

```csharp
private static readonly ManualResetEventSlim _systemReady = new(false);

void StartSystem() {
    LoadConfig();
    OpenConnections();
    _systemReady.Set();  // all blocked threads release
}

void HandleRequest() {
    _systemReady.Wait();  // returns immediately if already set
    Process();
}
```

**Concrete example — auto-reset for one-at-a-time work:**

```csharp
private static readonly AutoResetEvent _workReady = new(false);
private static Queue<Work> _queue = new();

void Producer() {
    lock (_queue) _queue.Enqueue(new Work());
    _workReady.Set();  // wakes exactly one waiter, then resets
}

void Consumer() {  // multiple consumer threads
    while (true) {
        _workReady.WaitOne();  // exactly one will be released per Set
        Work w;
        lock (_queue) w = _queue.Dequeue();
        Process(w);
    }
}
```

This is essentially an asymmetric semaphore: `Set` is `release(1)` and `WaitOne` is `acquire`.

### 5.6 Channels — message passing

**Surface claim:** "Channels avoid shared state."

**Between the lines:** Channels move ownership of data between threads. The receiver gets exclusive access; the sender no longer has it. This eliminates an entire category of bugs — there's no shared mutable state to synchronize.

**Concrete example — Rust mpsc channel:**

```rust
use std::sync::mpsc;
use std::thread;

let (tx, rx) = mpsc::channel::<String>();

thread::spawn(move || {
    let s = String::from("hello");
    tx.send(s).unwrap();  // ownership transferred; tx no longer has s
    // tx.send(s);  // compile error: s was moved
});

let received = rx.recv().unwrap();
println!("{}", received);
```

**Hidden detail — bounded vs unbounded:**

```rust
// Unbounded: send never blocks; queue can grow unboundedly.
let (tx, rx) = mpsc::channel();

// Bounded: send blocks when full. Provides backpressure.
let (tx, rx) = mpsc::sync_channel(100);  // capacity 100
```

**Why backpressure matters:** Without it, a fast producer overwhelms a slow consumer, exhausting memory. With it, the producer naturally slows down to match the consumer's rate.

**Concrete example — channel patterns:**

```rust
// SPSC: single producer, single consumer. Fastest implementations possible.
// Used for: ringbuffers, audio.

// MPSC: multi-producer, single consumer. Common for "log all events to one writer."

// MPMC: multi-producer, multi-consumer. Work queue.

// SPMC: rare. Broadcast pattern.
```

C# `System.Threading.Channels.Channel<T>` supports both bounded and unbounded:

```csharp
var ch = Channel.CreateBounded<int>(100);
await ch.Writer.WriteAsync(42);
int v = await ch.Reader.ReadAsync();
```

---

## 6. Lock-Free Programming Theory, With Real Code Paths

### 6.1 Progress guarantees, formally

**Surface claim:** "Lock-free means the system always progresses."

**Between the lines:** Each progress guarantee is a quantified statement about *which* threads make progress and *when*.

**Blocking:**
> "A thread can be delayed indefinitely by another thread's actions or inaction."

**Concrete example:** Any code holding a mutex. If the mutex holder is preempted, sleeping, or crashed, every other thread waiting blocks indefinitely.

**Obstruction-free:**
> "A thread that runs in isolation (other threads pause) finishes in a bounded number of steps."

**Concrete example:** Software Transactional Memory (STM) without contention management. If only one thread is committing, it succeeds. With contention, threads can endlessly abort each other.

**Lock-free:**
> "At least one thread (in the entire system) makes progress in a bounded number of steps."

**Concrete example:** A CAS loop. Even if 100 threads are racing, at least one CAS succeeds per iteration. The system makes progress; individual threads can starve.

**Wait-free:**
> "Every thread makes progress in a bounded number of *its own* steps."

**Concrete example:** SPSC queue with a bounded ring buffer. Each producer call is `O(1)` regardless of consumer activity. No retries.

### 6.2 Why lock-free matters in production

**Surface claim:** "Lock-free has nicer properties."

**Between the lines:** Three concrete operational benefits beyond performance:

**1. No deadlocks possible.** There are no locks; the dependency graph is empty.

```rust
// Any sequence of operations on lock-free structures cannot deadlock.
// Even if a thread is suspended at any point, others continue.
```

**2. Robust to thread death.** If a thread is killed (signal, crash, panic), no one is "holding" anything blocking others.

**Concrete example:** In a real-time system using a lock-free SPSC queue, you can SIGKILL a producer and the consumer continues processing the queue. With a mutex-protected queue, killing a producer mid-lock leaves the lock held; consumer deadlocks.

**3. Predictable tail latencies under contention.** A lock-free algorithm has bounded worst-case path length; a mutex can have unbounded wait.

**Concrete example:** A web service with 99.9th percentile SLA. With a mutex on a hot path, occasional convoys cause latency spikes to 100+ms. Switching to a lock-free queue keeps p99.9 under 1ms even at peak load. (This is why HFT systems use lock-free data structures pervasively.)

**Hidden detail — when lock-free is *worse*:** Average case under low contention. A mutex's uncontended path is ~25ns. A CAS loop is also ~25ns *if it succeeds first try*. But under contention, the CAS loop retries; the mutex puts threads to sleep. With high contention and long critical sections, mutexes can outperform lock-free because spinning threads burn CPU while sleeping threads don't.

### 6.3 The CAS loop pattern, dissected

**Surface claim:** "Read, compute, CAS, retry."

**Between the lines:** The CAS loop is the lock-free equivalent of "lock, modify, unlock." But it's optimistic: assume no conflict, and only "commit" if the world hasn't changed.

**The skeleton:**
```rust
loop {
    let snapshot = atomic.load(Ordering::Acquire);   // ① observe
    let new = transform(snapshot);                    // ② compute (possibly expensive!)
    match atomic.compare_exchange_weak(
        snapshot, new,
        Ordering::AcqRel, Ordering::Relaxed
    ) {
        Ok(_) => break,
        Err(_) => continue,                          // ③ retry
    }
}
```

**Hidden detail — wasted work:** The computation at ② happens *before* the CAS. If the CAS fails, that work is discarded. If the computation is expensive (e.g., involves allocation), this is wasteful.

**Concrete example — a lock-free counter that may overflow:**

```rust
fn add_saturating(v: &AtomicU64, x: u64) {
    let mut cur = v.load(Ordering::Relaxed);
    loop {
        let new = cur.saturating_add(x);
        if new == cur { return; }  // already at max
        match v.compare_exchange_weak(cur, new, Ordering::AcqRel, Ordering::Relaxed) {
            Ok(_) => return,
            Err(actual) => cur = actual,
        }
    }
}
```

**Concrete example — when computation is expensive:**

```rust
// BAD: allocates inside loop. Allocation under contention = thrashing.
loop {
    let cur_vec = atomic_ptr.load(...);
    let mut new_vec = (*cur_vec).clone();   // expensive!
    new_vec.push(item);
    let new_box = Box::into_raw(Box::new(new_vec));
    if atomic_ptr.compare_exchange_weak(cur_vec, new_box, ...).is_ok() { break; }
    // else: drop new_box (wasted), retry
}
```

This is why "copy-on-write atomic vec" is generally not a great primitive.

### 6.4 Linearizability — the gold standard correctness condition

**Surface claim:** "Operations appear atomic."

**Between the lines:** Linearizability says: every concurrent operation has a single point in time (its **linearization point**) where it took effect. The set of operations across all threads can be totally ordered by their linearization points, and that ordering produces a valid sequential execution.

**Concrete example — finding the linearization point of stack push:**

```rust
fn push(&self, value: T) {
    let new = Box::into_raw(Box::new(Node { value, next: ptr::null_mut() }));
    loop {
        let head = self.head.load(Ordering::Relaxed);
        unsafe { (*new).next = head; }
        if self.head.compare_exchange_weak(
            head, new, Ordering::Release, Ordering::Relaxed
        ).is_ok() {
            return;  // ← LINEARIZATION POINT: the successful CAS instruction
        }
    }
}
```

**Why the successful CAS is the linearization point:** Before the CAS commits, the new node is invisible to other threads. After the CAS commits, it's visible. There's a single hardware instant where the change becomes visible — the cache-line write that succeeds.

**Concrete example — linearization point of dequeue when empty:**

```rust
fn pop(&self) -> Option<T> {
    let head = self.head.load(Ordering::Acquire);
    if head.is_null() {
        return None;  // ← LINEARIZATION POINT for the empty case: the load
    }
    // ... CAS for non-empty case ...
}
```

For pop-when-empty, the linearization point is the load that observed null. This is correct because observing null is the moment we decided "the queue is empty at this instant." If a push happens after our load, our linearization is before that push — consistent with returning None.

**Why this matters in interviews:** When you describe a lock-free algorithm, identify each operation's linearization point. It's the strongest evidence that you understand the algorithm's correctness.

---

## 7. The ABA Problem, Walked Through Cycle by Cycle

### 7.1 The setup, made concrete

**Surface claim:** "A value goes A → B → A and CAS doesn't notice."

**Between the lines:** ABA only matters when the *meaning* of the value depends on its history, not just its identity. Pointers are the canonical case: the address may match, but the object behind it might be different (deallocated and reallocated).

**Concrete example — full ABA-on-pointers walkthrough in a Treiber stack:**

Initial state:
```
head ───→ [A] ───→ [B] ───→ [C] ───→ null
```

Thread 1 starts a `pop`:
1. T1: `head_local = head.load()` → reads pointer A
2. T1: `next_local = A.next` → reads pointer B
3. T1: about to call `head.compare_exchange(A, B)` — at this point, T1 thinks head=A, will swap to B.

Now T1 is preempted (timer interrupt, scheduler picks T2).

Thread 2 runs and does:
1. T2: pop returns A. `head` is now B. Memory of A is freed.
2. T2: pop returns B. `head` is now C. Memory of B is freed.
3. T2: allocate a new node. Allocator returns the *same address* as old A (likely — allocators often reuse recently-freed memory). Let's call this new node A' (same address, different content).
4. T2: push A'. A'.next = C. `head` = address-of-A'.

Now state:
```
head ───→ [A' (new)] ───→ [C]
```
But A and B's memory is freed, and head has the address that A used to live at.

Thread 1 resumes:
5. T1: `head.compare_exchange(A, B)` — the comparison is on pointer values. Pointer A == pointer A' (same address). CAS succeeds!
6. T1: `head` is now set to B. But B was freed in step 2!

**The corruption:**
```
head ───→ [B (freed!)] ───→ [garbage]
```

The next operation on head reads freed memory. Use-after-free, undefined behavior.

### 7.2 Solution 1: Tagged pointers (DWCAS)

**Surface claim:** "Pack a counter with the pointer."

**Between the lines:** Use double-width CAS (DWCAS) to atomically compare-and-swap a (pointer, counter) pair. Every modification increments the counter, even if the pointer returns to its old value. The counter detects ABA.

**Concrete example — using `cmpxchg16b` on x86-64:**

```rust
#[repr(C, align(16))]
#[derive(Copy, Clone)]
struct TaggedPtr<T> {
    ptr: *mut T,
    tag: u64,
}

// On x86-64 with the cmpxchg16b feature, AtomicU128 is available
// (or use crates like atomic128, or Rust's portable_atomic).

fn pop(head: &AtomicU128) -> Option<T> {
    loop {
        let cur = head.load(Ordering::Acquire);
        let cur_tagged: TaggedPtr<Node<T>> = transmute(cur);
        if cur_tagged.ptr.is_null() { return None; }
        let next = unsafe { (*cur_tagged.ptr).next };
        let new = TaggedPtr { ptr: next, tag: cur_tagged.tag.wrapping_add(1) };
        if head.compare_exchange_weak(cur, transmute(new), Ordering::AcqRel, Ordering::Relaxed).is_ok() {
            return Some(unsafe { (*cur_tagged.ptr).value });  // still need reclamation!
        }
    }
}
```

**Walking through ABA prevention:**
- Initial: head = (ptr=A, tag=5).
- T1 reads (A, 5). Plans CAS((A,5), (B,6)).
- T2 pops A: head = (B, 6). Pops B: head = (null, 7). Pushes A': head = (A, 8).
- T1 attempts CAS((A,5), (B,6)) — fails! Current head is (A, 8). Tag 5 ≠ 8. T1 retries.

**Hidden detail — counter overflow:** With a 64-bit counter, you'd need 2^64 modifications to wrap. At 1GHz update rate, that's 580 years. With a 32-bit counter (older systems), it's ~4 seconds — still possible to ABA in practice. Always use 64-bit tags.

### 7.3 Solution 2: LL/SC (hardware-level immunity)

**Surface claim:** "LL/SC detects any cache line modification."

**Between the lines:** Already covered in section 3.5. The key point: ARM/POWER/RISC-V give you ABA-free CAS for free, because the reservation is invalidated by *any* write to the line, not just value changes.

**Concrete example — same ABA scenario on ARM:**
- T1: `LDAXR x1, [head]` → reads A, sets reservation on cache line containing head.
- T2: pops A → writes head, invalidating T1's reservation.
- T2: pops B → another write.
- T2: pushes A' → another write.
- T1: `STLXR x2, x1, [head]` → fails (reservation gone). T1 retries.

x86 doesn't have LL/SC, so you need DWCAS or another solution.

### 7.4 Solution 3: Hazard pointers / epoch reclamation

**Surface claim:** "Don't reuse memory while anyone might hold a pointer to it."

**Between the lines:** This avoids ABA by avoiding memory reuse. If A's memory is never reallocated until all readers are done, ABA can't happen — A' would be at a different address.

**Detailed mechanism in section 8.**

### 7.5 When ABA doesn't matter

**Concrete example — counter with ABA that's harmless:**

```rust
// Atomic increment is immune to ABA at the value level.
// Even if you read 5, someone increments to 6, decrements to 5, and you CAS(5, 6),
// the result is correct: counter is 6. The "lost" intermediate state of 6 → 5 → 6
// doesn't matter because increment is associative.
counter.fetch_add(1, Ordering::Relaxed);
```

ABA only bites when the value's *meaning* depends on history. For pure counters, max trackers, etc., it's fine. For pointers and tagged objects, it's deadly.

---

## 8. Memory Reclamation, Mechanism by Mechanism

In garbage-collected languages (C#, Java, Go), this section is mostly free — the GC handles it. In Rust/C++, you must answer: **when can I free a node I logically removed?**

### 8.1 Reference counting — `Arc<T>`

**Surface claim:** "Atomic refcount, free when zero."

**Between the lines:** `Arc::clone` increments atomically (relaxed); drop decrements (release) and checks for zero (acquire). The reads/writes are simple. The hard part: how do you safely *acquire* an `Arc` from a shared atomic pointer? You can race between "I read the pointer" and "I clone its Arc."

**Concrete example — the race that breaks naive shared-Arc:**

```rust
// BROKEN: storing Arc<T> in AtomicPtr without protection
static SHARED: AtomicPtr<Arc<Data>> = AtomicPtr::new(ptr::null_mut());

fn read() -> Arc<Data> {
    let p = SHARED.load(Ordering::Acquire);
    let arc = unsafe { Arc::clone(&*p) };  // RACE: between load and clone, the Arc could be dropped
    arc
}
```

**Why broken:** Between `load` and `Arc::clone`, another thread can store a different Arc and drop the old one (refcount may go to zero). Now `*p` is dangling; `Arc::clone` reads freed memory.

**Fix — use `arc-swap`:**

```rust
use arc_swap::ArcSwap;
use std::sync::Arc;

let shared: ArcSwap<Data> = ArcSwap::from(Arc::new(initial));

// Reader:
let snapshot = shared.load();  // returns Guard<Arc<Data>>
// snapshot is safe to use; ArcSwap uses hazard pointers internally
```

**How `arc-swap` works internally:** Each thread has hazard pointer slots. `load` publishes the pointer in the slot, then re-reads the shared pointer to verify, then increments the refcount, then clears the slot. The publication ensures the writer can't drop the Arc to zero while a reader is "in flight."

### 8.2 Hazard pointers — Maged Michael, 2004

**Surface claim:** "Each thread publishes pointers it's using."

**Between the lines:** Each thread has a small array of "hazard pointer" slots (typically 1-8). Before reading a pointer, write it to your slot. Before freeing a pointer, scan all threads' slots; if anyone has it, defer freeing.

**Concrete example — the read protocol:**

```rust
// Pseudocode for hazard-pointer-protected pop
fn pop_with_hp(head: &AtomicPtr<Node>, my_hp: &AtomicPtr<Node>) -> Option<T> {
    loop {
        let p = head.load(Ordering::Acquire);
        if p.is_null() { return None; }

        // Publish p in our hazard slot.
        my_hp.store(p, Ordering::SeqCst);  // SeqCst critical here

        // Re-validate: did head change between load and publish?
        if head.load(Ordering::Acquire) != p {
            continue;  // p might have been freed; retry
        }

        // Now p is protected: any thread about to free p will see our hazard slot.
        let next = unsafe { (*p).next };

        if head.compare_exchange(p, next, Ordering::AcqRel, Ordering::Relaxed).is_ok() {
            my_hp.store(ptr::null_mut(), Ordering::Release);
            return Some(unsafe { read_value_then_retire(p) });
        }
    }
}

fn retire(p: *mut Node, retired_list: &mut Vec<*mut Node>) {
    retired_list.push(p);
    if retired_list.len() > THRESHOLD {
        scan_and_free(retired_list);
    }
}

fn scan_and_free(retired: &mut Vec<*mut Node>) {
    // Collect all threads' hazard pointers.
    let in_use: HashSet<*mut Node> = all_thread_hps();
    retired.retain(|p| {
        if in_use.contains(p) { true }       // still hazard; keep in retired list
        else { unsafe { drop_in_place(*p); }; false }
    });
}
```

**Walk-through:**
1. T1 starts pop. Reads head=A. Stores A in its hazard slot.
2. T2 pops A. Now wants to free A. Calls `retire(A)`. Adds A to its retire list.
3. T2 eventually calls `scan_and_free`. Sees T1's slot contains A. Defers freeing A.
4. T1 finishes pop. Clears its hazard slot.
5. Next time T2 (or anyone) scans, A's slot is clear. A is freed.

**Hidden detail — why SeqCst:** The publish-then-revalidate idiom requires sequential consistency. With weaker orderings, the publish might be reordered after the revalidation read, breaking the guarantee.

**Cost:** Per-read overhead (one SeqCst store + one extra load). Bounded memory: at most `K * N` retired nodes, where K is per-thread retire threshold and N is thread count.

### 8.3 Epoch-based reclamation (EBR) — `crossbeam-epoch`

**Surface claim:** "Threads enter epochs; retired nodes wait for old epochs to drain."

**Between the lines:** A global epoch counter advances over time (e.g., 0, 1, 2, ...). Each thread, when accessing the structure, "pins" itself to the current epoch. Retired nodes are tagged with the epoch when they were retired. A node can be freed once *no thread is pinned to that epoch or earlier*.

**Concrete example — using `crossbeam-epoch`:**

```rust
use crossbeam_epoch::{self as epoch, Atomic, Owned, Shared};
use std::sync::atomic::Ordering;

struct Stack<T> { head: Atomic<Node<T>> }
struct Node<T> { value: T, next: Atomic<Node<T>> }

impl<T> Stack<T> {
    fn push(&self, value: T) {
        let guard = &epoch::pin();  // pin to current epoch
        let mut new = Owned::new(Node { value, next: Atomic::null() });
        loop {
            let head = self.head.load(Ordering::Acquire, guard);
            new.next.store(head, Ordering::Relaxed);
            match self.head.compare_exchange(head, new, Ordering::Release, Ordering::Relaxed, guard) {
                Ok(_) => return,
                Err(e) => new = e.new,
            }
        }
    }

    fn pop(&self) -> Option<T> {
        let guard = &epoch::pin();
        loop {
            let head = self.head.load(Ordering::Acquire, guard);
            match unsafe { head.as_ref() } {
                None => return None,
                Some(node) => {
                    let next = node.next.load(Ordering::Acquire, guard);
                    if self.head.compare_exchange(head, next, Ordering::AcqRel, Ordering::Acquire, guard).is_ok() {
                        unsafe {
                            guard.defer_destroy(head);  // schedule for free in a future epoch
                            return Some(ptr::read(&node.value));
                        }
                    }
                }
            }
        }
    }
}
```

**The epoch protocol internally:**
- Global epoch `E` (currently say 5).
- Each thread, when active, sets its local epoch = E.
- `defer_destroy(p)` adds p to a per-thread bag tagged with current epoch (5).
- Periodically, the runtime checks: "What's the smallest local epoch any thread has?" If all threads are at epoch ≥ 7, then any node in the epoch-5 bag is safe to free (no thread can possibly hold a reference from before epoch 5+1 = 6).
- Global epoch advances when there's pending garbage and threads have advanced.

**Hidden detail — why this works:** A pointer can only be obtained while pinned. Once pinned to epoch 5, you may hold pointers obtained during epoch 5. When you unpin and re-pin, you might be at epoch 7. The retired-at-epoch-5 nodes can only be referenced by threads pinned at epoch ≤ 5. Once all threads have advanced past, those pointers can't be used.

**Concrete example — failure mode of EBR:** A thread that pins and never unpins prevents reclamation forever. Memory grows unboundedly. Always pin briefly.

```rust
// BAD: long pin
let guard = epoch::pin();
loop {
    let head = self.head.load(Ordering::Acquire, &guard);
    do_something_slow();  // memory grows!
}

// GOOD: short pin per iteration
loop {
    let guard = epoch::pin();
    let head = self.head.load(Ordering::Acquire, &guard);
    do_something();
    drop(guard);  // explicit unpin
}
```

### 8.4 RCU (Read-Copy-Update) — Linux kernel's secret weapon

**Surface claim:** "Readers are zero overhead; writers copy and publish."

**Between the lines:** RCU's revolution is making readers truly zero-overhead — no atomics, no fences, no writes to shared data on the read path. Cost is paid by writers.

**Concrete example — RCU in the Linux kernel:**

```c
// Reader path: no atomics, no locks!
rcu_read_lock();
struct config *p = rcu_dereference(global_config);
use(p->field);
rcu_read_unlock();

// Writer path: copy, modify, publish, wait, free
struct config *new_p = kmalloc(...);
struct config *old_p = global_config;
*new_p = *old_p;
new_p->field = new_value;
rcu_assign_pointer(global_config, new_p);  // publish (release)
synchronize_rcu();  // wait for grace period: all pre-existing readers done
kfree(old_p);
```

**What `rcu_read_lock` and `rcu_read_unlock` actually do in the kernel:** they simply disable preemption (a single CPU register write). No atomic, no shared memory. This is the magic.

**How `synchronize_rcu` works:** It waits until every CPU has gone through a context switch (or another quiescent state). Since readers can't be preempted while in a read-side critical section, once every CPU has context-switched, all pre-existing readers must have finished.

**Concrete example — why this is fast:**

A reader on a 1GHz machine doing `rcu_read_lock; load; rcu_read_unlock` costs ~3 cycles. A read with a mutex costs ~25ns = 25 cycles uncontended, 100s under contention. RCU is roughly **10-100x faster** for read-heavy workloads.

**Trade-off:** Writers pay grace period delay (potentially milliseconds). Memory usage can grow during a grace period (multiple versions live).

**Userspace RCU (urcu):** Achieves similar guarantees without kernel-level preemption-disable, using more elaborate epoch tracking. Used by some HFT systems.

### 8.5 Quiescent State Based Reclamation (QSBR)

**Surface claim:** "Threads periodically declare they're idle."

**Between the lines:** Like EBR but lower per-operation overhead. Threads declare quiescence at natural boundaries (e.g., end of an event loop iteration). Memory can be reclaimed once all threads have been quiescent at least once since retirement.

**Concrete example — typical usage in a server:**

```c
while (running) {
    process_request();
    qsbr_quiescent_state();  // "I'm not in any critical section right now"
}
```

If all threads call `qsbr_quiescent_state` at least once between time T (when X was retired) and time T', then at T' it's safe to free X.

**Comparison to EBR:** EBR's pin/unpin happens per operation (more overhead, more granular). QSBR's quiescent state is sparse but requires explicit cooperation.


---

## 9. Lock-Free Data Structures, Built From Scratch

### 9.1 Treiber Stack — the simplest lock-free structure

**Surface claim:** "Push and pop with CAS on head."

**Between the lines:** The structure is a singly-linked list. The only shared mutable state is the head pointer. Every push or pop is a CAS on head. The simplicity hides the ABA risk (covered in section 7) and the memory reclamation challenge (covered in section 8).

**Concrete example — full implementation:**

```rust
use std::sync::atomic::{AtomicPtr, Ordering};
use std::ptr;

struct Stack<T> {
    head: AtomicPtr<Node<T>>,
}

struct Node<T> {
    value: T,
    next: *mut Node<T>,
}

impl<T> Stack<T> {
    fn new() -> Self {
        Stack { head: AtomicPtr::new(ptr::null_mut()) }
    }

    fn push(&self, value: T) {
        let new = Box::into_raw(Box::new(Node { value, next: ptr::null_mut() }));
        loop {
            let head = self.head.load(Ordering::Relaxed);
            unsafe { (*new).next = head; }
            // Release: ensure (*new).next write is visible before head update
            match self.head.compare_exchange_weak(head, new, Ordering::Release, Ordering::Relaxed) {
                Ok(_) => return,
                Err(_) => continue,
            }
        }
    }

    fn pop(&self) -> Option<T> {
        loop {
            let head = self.head.load(Ordering::Acquire);
            if head.is_null() { return None; }
            let next = unsafe { (*head).next };
            // AcqRel: acquire to see push's write to next; release to publish new head
            match self.head.compare_exchange_weak(head, next, Ordering::AcqRel, Ordering::Acquire) {
                Ok(_) => {
                    // SAFETY VIOLATION in real code: needs hazard pointers / EBR.
                    // For this example, assume single-threaded popping.
                    let node = unsafe { Box::from_raw(head) };
                    return Some(node.value);
                }
                Err(_) => continue,
            }
        }
    }
}
```

**Walk-through of concurrent pushes:**

Initial: `head = null`.
- T1: push(1). new=A. read head=null. set A.next=null. CAS(null, A). ✓ head=A.
- T2: push(2). new=B. read head=A. set B.next=A. CAS(A, B). ✓ head=B.
- T3: push(3). new=C. read head=B. set C.next=B. CAS(B, C). ✓ head=C.

State: `head → C → B → A → null`. Values [3, 2, 1].

**Walk-through of contention:**
- T1: push(1). new=A. read head=null. set A.next=null. About to CAS.
- T2: push(2). new=B. read head=null. set B.next=null. CAS(null, B). ✓ head=B.
- T1: CAS(null, A) fails (head is B, not null). Loop.
- T1: read head=B. set A.next=B. CAS(B, A). ✓ head=A.

State: `head → A → B → null`. Both pushes succeeded; ordering depended on CAS race.

**Hidden detail — Why `Release` on push:** The new node has `next` written non-atomically before the CAS. We need readers (who do `Acquire` on head) to see the `next` write. Release on the CAS ensures this.

**Hidden detail — Why `Acquire` on the failed CAS:** Even on failure, we may want to see updates from other threads' successful operations. In this code, the failed CAS uses `Acquire` for safety, though `Relaxed` would work since we just retry the load.

### 9.2 Michael-Scott Queue — the standard MPMC FIFO

**Surface claim:** "Two pointers (head, tail), CAS on each."

**Between the lines:** A queue is harder than a stack because both ends change. Naively, you'd need to atomically update head and tail together — but that's not possible with single-word CAS. Michael & Scott (PODC 1996) solved this with two key tricks:

1. **Sentinel (dummy) node**: head always points to a dummy. Real values live in nodes after head.
2. **Lazy tail**: tail can be one node behind. Any thread that notices a lagging tail helps advance it.

**Concrete example — full skeleton:**

```rust
struct Queue<T> {
    head: AtomicPtr<Node<T>>,
    tail: AtomicPtr<Node<T>>,
}

struct Node<T> {
    value: Option<T>,  // None for dummy
    next: AtomicPtr<Node<T>>,
}

impl<T> Queue<T> {
    fn new() -> Self {
        let dummy = Box::into_raw(Box::new(Node {
            value: None,
            next: AtomicPtr::new(ptr::null_mut()),
        }));
        Queue {
            head: AtomicPtr::new(dummy),
            tail: AtomicPtr::new(dummy),
        }
    }

    fn enqueue(&self, value: T) {
        let new = Box::into_raw(Box::new(Node {
            value: Some(value),
            next: AtomicPtr::new(ptr::null_mut()),
        }));
        loop {
            let tail = self.tail.load(Ordering::Acquire);
            let next = unsafe { (*tail).next.load(Ordering::Acquire) };
            // Tail might have advanced; re-check
            if tail != self.tail.load(Ordering::Acquire) { continue; }

            if next.is_null() {
                // Tail is the last node. Try to append.
                if unsafe { (*tail).next.compare_exchange(
                    ptr::null_mut(), new,
                    Ordering::Release, Ordering::Relaxed,
                ).is_ok() } {
                    // Successfully linked. Try to swing tail (best effort).
                    let _ = self.tail.compare_exchange(
                        tail, new,
                        Ordering::Release, Ordering::Relaxed,
                    );
                    return;
                }
            } else {
                // Tail was lagging. Help advance it.
                let _ = self.tail.compare_exchange(
                    tail, next,
                    Ordering::Release, Ordering::Relaxed,
                );
            }
        }
    }

    fn dequeue(&self) -> Option<T> {
        loop {
            let head = self.head.load(Ordering::Acquire);
            let tail = self.tail.load(Ordering::Acquire);
            let next = unsafe { (*head).next.load(Ordering::Acquire) };
            if head != self.head.load(Ordering::Acquire) { continue; }
            
            if head == tail {
                if next.is_null() {
                    return None;  // queue empty
                }
                // Tail is lagging, help advance.
                let _ = self.tail.compare_exchange(tail, next, Ordering::Release, Ordering::Relaxed);
            } else {
                // Read value before CAS, in case someone else dequeues
                let value = unsafe { ptr::read(&(*next).value) };
                if self.head.compare_exchange(head, next, Ordering::Release, Ordering::Relaxed).is_ok() {
                    // Free old head (sentinel becomes 'next', old becomes garbage)
                    // Real code: schedule via EBR; for simplicity:
                    // unsafe { drop(Box::from_raw(head)); }
                    return value;
                }
            }
        }
    }
}
```

**Walk-through of concurrent enqueue:**

Initial: `head → dummy ← tail`, dummy.next = null.

- T1: enqueue(1). new=A. read tail=dummy, next=null. CAS dummy.next: null→A. ✓
  - Now `dummy.next = A`, but tail still points to dummy.
  - T1 about to swing tail: dummy → A. Suspended (preempted).

- T2: enqueue(2). new=B. read tail=dummy, next=A.
  - next is not null! Tail is lagging. T2 *helps*: CAS tail dummy→A. ✓
  - T2 retries: read tail=A, next=null. CAS A.next: null→B. ✓
  - T2 swings tail: A→B. ✓

- T1 resumes: tries CAS tail dummy→A. Fails (tail is now B). T1 ignores (best-effort).

State: `head → dummy → A(1) → B(2) ← tail`.

**Hidden detail — why the consistency re-check:**

```rust
let tail = self.tail.load(...);
let next = unsafe { (*tail).next.load(...) };
if tail != self.tail.load(...) { continue; }
```

Between reading `tail` and reading `tail->next`, another thread might have dequeued and freed the node we read tail from. The re-check ensures tail is still valid. This is *not* sufficient for memory safety alone (we'd still have read freed memory in the second line); it's a consistency check. Memory safety requires hazard pointers or EBR.

**Hidden detail — why dequeue reads `value` before CAS:**

The CAS publishes the change "head moved." Once published, another dequeuer might also try to read the value and free the old head. We read first, then CAS, then free. There's still a subtle issue: two threads can both read the same `next->value` if they race; but only one wins the CAS and returns it. Need careful types (Option/move semantics) to prevent double-take.

In real implementations, the value is typically `MaybeUninit<T>` and the value is consumed only by the winning CAS thread.

### 9.3 SPSC Ring Buffer — the speed champion

**Surface claim:** "Single producer, single consumer, no CAS needed."

**Between the lines:** When there's exactly one producer and one consumer, you can use plain atomic load/store instead of CAS. The producer writes the tail; the consumer writes the head. Each only reads the other's index.

**Concrete example — full implementation:**

```rust
use std::cell::UnsafeCell;
use std::mem::MaybeUninit;
use std::sync::atomic::{AtomicUsize, Ordering};

struct SpscRing<T, const N: usize> {
    buf: [UnsafeCell<MaybeUninit<T>>; N],
    head: AtomicUsize,  // next read position; written by consumer only
    tail: AtomicUsize,  // next write position; written by producer only
}

unsafe impl<T: Send, const N: usize> Sync for SpscRing<T, N> {}

impl<T, const N: usize> SpscRing<T, N> {
    fn try_push(&self, value: T) -> Result<(), T> {
        let tail = self.tail.load(Ordering::Relaxed);  // we wrote it; relaxed OK
        let next = (tail + 1) % N;
        if next == self.head.load(Ordering::Acquire) {
            return Err(value);  // full
        }
        unsafe { (*self.buf[tail].get()).write(value); }
        self.tail.store(next, Ordering::Release);  // publish
        Ok(())
    }

    fn try_pop(&self) -> Option<T> {
        let head = self.head.load(Ordering::Relaxed);
        if head == self.tail.load(Ordering::Acquire) {
            return None;  // empty
        }
        let value = unsafe { (*self.buf[head].get()).assume_init_read() };
        self.head.store((head + 1) % N, Ordering::Release);
        Some(value)
    }
}
```

**Walk-through:**

Buffer N=4. Initial: head=0, tail=0 (empty).

- Producer: try_push(10). tail=0. next=1. head=0. next!=head. Write buf[0]=10. tail=1.
- Producer: try_push(20). tail=1. next=2. head=0. Write buf[1]=20. tail=2.
- Consumer: try_pop. head=0. tail=2. Read buf[0]=10. head=1. Returns 10.
- Producer: try_push(30). tail=2. next=3. head=1. Write buf[2]=30. tail=3.
- Producer: try_push(40). tail=3. next=0. head=1. Write buf[3]=40. tail=0.
- Producer: try_push(50). tail=0. next=1. head=1. next==head → full. Return Err.

**Why the orderings:**
- Producer's load of `head`: Acquire. We need to see the consumer's `head` update (which was Release). Without acquire, we might see stale head and write into a slot the consumer hasn't yet finished reading.
- Producer's store of `tail`: Release. We're publishing the data write at `buf[tail]`. The consumer's Acquire load of `tail` synchronizes with this; the consumer can then safely read `buf[tail]`.
- Producer's load of own `tail`: Relaxed. We wrote it last; we know the value.

**Why this is fast:** No CAS, no MFENCE on x86. Every operation is a single atomic load + a single atomic store. On x86, those are basically free (regular MOV instructions with implicit acquire/release).

**Real-world use:** DPDK packet queues between cores. Audio engines (one thread fills audio buffer, another plays it). Linux kernel-userspace shared memory rings (io_uring, perf events).

### 9.4 Vyukov's MPMC Bounded Queue — the gold standard

**Surface claim:** "Each slot has a sequence number."

**Between the lines:** Dmitry Vyukov designed a bounded MPMC queue where each slot tracks its own state via a sequence counter. Producers and consumers race independently, but per-slot CAS coordinates ownership.

**Concrete structure:**

```rust
struct Slot<T> {
    seq: AtomicUsize,
    data: UnsafeCell<MaybeUninit<T>>,
}

struct MpmcQueue<T, const N: usize> {
    buffer: [Slot<T>; N],
    enqueue_pos: AtomicUsize,  // CACHE PADDED
    dequeue_pos: AtomicUsize,  // CACHE PADDED
}

// Initialization: slot[i].seq = i for all i.
```

**Enqueue protocol:**

```rust
fn try_enqueue(&self, value: T) -> Result<(), T> {
    let mut pos = self.enqueue_pos.load(Ordering::Relaxed);
    loop {
        let slot = &self.buffer[pos % N];
        let seq = slot.seq.load(Ordering::Acquire);
        let diff = seq as i64 - pos as i64;

        if diff == 0 {
            // Slot is ready to be filled; try to claim position
            match self.enqueue_pos.compare_exchange_weak(
                pos, pos + 1, Ordering::Relaxed, Ordering::Relaxed
            ) {
                Ok(_) => {
                    // We own this position. Write data and update seq.
                    unsafe { (*slot.data.get()).write(value); }
                    slot.seq.store(pos + 1, Ordering::Release);
                    return Ok(());
                }
                Err(actual) => pos = actual,
            }
        } else if diff < 0 {
            return Err(value);  // queue full
        } else {
            pos = self.enqueue_pos.load(Ordering::Relaxed);  // tail moved; refresh
        }
    }
}
```

**Dequeue protocol:**

```rust
fn try_dequeue(&self) -> Option<T> {
    let mut pos = self.dequeue_pos.load(Ordering::Relaxed);
    loop {
        let slot = &self.buffer[pos % N];
        let seq = slot.seq.load(Ordering::Acquire);
        let diff = seq as i64 - (pos + 1) as i64;

        if diff == 0 {
            match self.dequeue_pos.compare_exchange_weak(
                pos, pos + 1, Ordering::Relaxed, Ordering::Relaxed
            ) {
                Ok(_) => {
                    let value = unsafe { (*slot.data.get()).assume_init_read() };
                    slot.seq.store(pos + N, Ordering::Release);  // mark slot as ready for next round
                    return Some(value);
                }
                Err(actual) => pos = actual,
            }
        } else if diff < 0 {
            return None;  // empty
        } else {
            pos = self.dequeue_pos.load(Ordering::Relaxed);
        }
    }
}
```

**Walk-through:**

N=4. Slots initialized: seq=[0,1,2,3]. enqueue_pos=0, dequeue_pos=0.

- T1: enqueue(10). pos=0. slot=0, seq=0, diff=0. CAS enqueue_pos 0→1. ✓ Write buf[0]=10. seq[0]=1.
- T2: enqueue(20). pos=1. slot=1, seq=1, diff=0. CAS 1→2. ✓ Write buf[1]=20. seq[1]=2.
- T3: dequeue. pos=0. slot=0, seq=1, diff=1-1=0. CAS dequeue_pos 0→1. ✓ Read buf[0]=10. seq[0]=4. Return 10.
- T4: enqueue(30). pos=2. slot=2, seq=2, diff=0. CAS 2→3. Write buf[2]=30. seq[2]=3.
- T5: enqueue. pos=3. slot=3, seq=3, diff=0. CAS 3→4. Write buf[3]=40. seq[3]=4.
- T6: enqueue. pos=4. slot=4%4=0. seq[0]=4. diff=4-4=0. CAS 4→5. Write buf[0]=50. seq[0]=5.
  - Slot 0 now has value 50; this is the "second round."

**Why sequence numbers work:** Each slot's `seq` encodes both "is this filled?" and "which round?" When a producer sees `seq == pos`, it means "this slot is ready for round (pos / N)." When a consumer sees `seq == pos + 1`, it means "this slot was filled in round (pos / N)."

**Why this is efficient:**
- Producers and consumers operate on different cache lines (head and tail are padded apart).
- Per-slot sequences avoid a global lock; multiple producers can be filling different slots simultaneously.
- No allocation; everything is in a fixed array.

**Used by:** `crossbeam::queue::ArrayQueue` (Rust), Cameron's `concurrentqueue` (C++), and many MPMC tokio channels.

### 9.5 Lock-free hash map — the migration challenge

**Surface claim:** "Hash maps need lock-free resizing."

**Between the lines:** A hash map's hard part isn't insert/lookup — those can use per-bucket CAS. The hard part is **resize**: doubling the table requires moving every entry, and you can't lock the world.

**Concrete example — the CHM resize protocol (simplified Java ConcurrentHashMap):**

When load factor crosses threshold:
1. Allocate a new table double the size. Mark old table as "resizing."
2. Concurrent inserters/lookups *help* migrate. Each thread that operates on the old table moves a few buckets to the new table before doing its operation.
3. A bucket being moved is marked with a "forwarding" sentinel. Lookups encountering this consult the new table.
4. Once all buckets are migrated, old table is freed (via EBR).

**Concrete walkthrough:**

Old table size 4, new table size 8. Bucket 0 has nodes [(key=A, value=1), (key=E, value=5)]. (Both hash to bucket 0 in old; A goes to bucket 0, E to bucket 4 in new.)

- T1: lookup(A). Sees bucket 0 has forwarding marker. Reads from new table bucket 0 → finds A. Returns 1.
- T2: insert(K, 11). Helps migrate bucket 0 first. Walks list: A → goes to new bucket 0 (CAS into new). E → goes to new bucket 4 (CAS into new). Sets old bucket 0 to forwarding marker. Then inserts K into new table.
- T3: lookup(E). Goes to new table directly (knows resize is in progress; hashes to bucket 4 in new size). Finds E.

**Crucial CAS:** The "publish forwarding marker" CAS on the old bucket atomically completes migration of that bucket. Until that CAS, all readers must use the old table; after, all use the new.

**This is genuinely complex.** Don't implement from scratch; use libraries:
- Rust: `flurry` (port of Java CHM), `papaya`
- C++: `folly::ConcurrentHashMap`, `junction`
- Java: `java.util.concurrent.ConcurrentHashMap`
- C#: `System.Collections.Concurrent.ConcurrentDictionary` (uses striped locking, not lock-free, but very fast)

### 9.6 Harris's Lock-Free Linked List

**Surface claim:** "Logical deletion via marked next pointers."

**Between the lines:** Tim Harris (2001) solved a hard problem: how to delete a node from a linked list lock-free. The naive approach has a race — between observing a node and CAS-removing it, another thread can insert after it, creating a lost update.

**The solution — two-phase deletion:**
1. **Logical delete:** mark the node's next pointer (steal a low bit, e.g., set bit 0 of `next` to indicate "deleted").
2. **Physical delete:** CAS the predecessor's next from "this node" to "this node's next."

The marking ensures that if anyone tries to insert after a marked node, the insert CAS fails (the next pointer they observed has the mark bit, but they're trying to CAS without it).

**Concrete pseudocode:**

```rust
// `next` is a pointer with the LSB used as a "marked for deletion" flag.
// Real Rust: use AtomicPtr with bit manipulation, or a tagged-pointer crate.

fn delete(prev: &Node, target: &Node) -> bool {
    // Phase 1: mark target as deleted
    loop {
        let next = target.next.load(Acquire);
        if is_marked(next) { return false; }  // already deleted
        if target.next.compare_exchange(next, mark(next), AcqRel, Relaxed).is_ok() {
            break;
        }
    }
    // Phase 2: try to physically remove (best-effort; another thread may help)
    let _ = prev.next.compare_exchange(
        target_unmarked, target.next.load(Acquire), AcqRel, Relaxed
    );
    true
}

fn insert_after(prev: &Node, new: *mut Node) -> bool {
    loop {
        let next = prev.next.load(Acquire);
        if is_marked(next) { return false; }  // prev is deleted; abort
        new.next.store(unmark(next), Release);
        if prev.next.compare_exchange(next, new, AcqRel, Relaxed).is_ok() {
            return true;
        }
    }
}
```

**Walk-through of the race that's now safe:**

List: A → B → C. T1 wants to delete B. T2 wants to insert X after B.

- T1: marks B.next (now B.next has the mark bit set).
- T2: reads prev=B, next=B.next (which has mark). T2 sees it's marked → aborts insert (or retries from a different position).

Without marking, T2's insert could succeed at the same time as T1's delete, and X would be in a list segment that's about to be unlinked.

### 9.7 Lock-Free Skip List

**Surface claim:** "Multi-level Harris lists with probabilistic balance."

**Between the lines:** A skip list is multiple linked lists at increasing levels of "skip distance." Each level is a Harris-style lock-free list. Insert/delete must update all levels of the tower.

**Concrete example — finding key 17 in a skip list:**

```
Level 3:  HEAD -----------------> 50 -----------> nil
Level 2:  HEAD -------> 20 ------> 50 -----> 80 -> nil
Level 1:  HEAD -> 5 -> 20 -> 35 -> 50 -> 65 -> 80 -> nil
Level 0:  HEAD -> 5 -> 12 -> 20 -> 35 -> 41 -> 50 -> 65 -> 80 -> nil
```

To find 17:
- L3: 50 > 17, go down.
- L2: 20 > 17, go down.
- L1: 5 < 17, advance. 20 > 17, go down.
- L0: 5 < 17, advance. 12 < 17, advance. 20 > 17, stop. 17 not found (would be between 12 and 20).

Insertion of 17 with random tower height:
1. Pick height (e.g., 2) randomly with geometric distribution (height 0 with p=0.5, height 1 with p=0.25, height 2 with p=0.125, etc.).
2. Find insertion point at each level.
3. CAS-insert at level 0. If succeeded, CAS-insert at level 1. etc.
4. If a CAS fails (concurrent insert/delete), retry that level.

Java's `ConcurrentSkipListMap` is the canonical implementation. Rust's `crossbeam::SkipMap` is excellent.

**Why skip lists vs trees?** Lock-free balanced trees are extremely hard (rebalancing needs multi-node atomic update). Skip lists' probabilistic balance avoids the issue — you don't need to rebalance, just choose a good height for new nodes.

---

## 10. Wait-Free Structures, With Helping Mechanics

### 10.1 The cost of wait-freedom

**Surface claim:** "Wait-free is the strongest progress guarantee."

**Between the lines:** Wait-freedom requires every operation to complete in a bounded number of *its own* steps. To achieve this in the presence of other threads, slow threads must be *helped*. The helping mechanism is what makes wait-free algorithms complex.

**Concrete example — why CAS loop alone isn't wait-free:**

In a CAS loop, if K other threads keep succeeding their CAS first, you retry K times. K can be unbounded. You're lock-free (some thread always progresses), not wait-free (you might never progress).

To make it wait-free, you need a bound on retries that doesn't depend on system size.

### 10.2 Universal Construction — Herlihy 1991

**Surface claim:** "Any sequential object can be made wait-free."

**Between the lines:** Herlihy showed any deterministic sequential object can be transformed into a wait-free concurrent version using CAS plus an unbounded log. The construction is theoretically beautiful but practically slow:

- Each thread maintains an announcement.
- Each operation: announce intent, then walk the announcement list helping any pending operation, finally apply your own operation.
- Result: every operation completes in O(N) time where N is thread count.

**Concrete example — the basic protocol:**

```c
Operation announce[N];  // announcement slot per thread

void apply(int my_id, Op op) {
    announce[my_id] = op;  // publish intent
    for (int i = 0; i < N; i++) {
        Op other = announce[i];
        if (other != null && !is_done(other)) {
            // Help complete this operation
            try_apply(other);
        }
    }
    return result_of(announce[my_id]);
}
```

Every operation does O(N) work helping others. Worth it for theoretical wait-freedom but rarely worth it in practice.

### 10.3 Practical wait-free structures

**Wait-free SPSC queue (Lamport):** Trivially wait-free. Each push/pop is a constant number of loads/stores.

**Wait-free MPSC queue (Vyukov's intrusive list):**

```rust
struct VyukovMpsc<T> {
    head: AtomicPtr<Node<T>>,  // producers append here
    tail: *mut Node<T>,        // consumer reads here (single consumer)
}

fn push(&self, node: *mut Node<T>) {
    unsafe { (*node).next.store(ptr::null_mut(), Relaxed); }
    let prev = self.head.swap(node, AcqRel);  // wait-free: bounded swap
    unsafe { (*prev).next.store(node, Release); }  // link forward
}

fn pop(&mut self) -> Option<T> {
    // Single consumer; no contention here.
    let tail = self.tail;
    let next = unsafe { (*tail).next.load(Acquire) };
    if next.is_null() { return None; }
    self.tail = next;
    Some(unsafe { (*next).value.read() })
}
```

**Why push is wait-free:** It uses `swap`, not CAS. `swap` always succeeds (no retry loop). It's a constant-time operation.

**Push has a transient inconsistency:** Between `swap` (which exposes `node` as new head) and `store` (which links the old head to node), there's a window where the queue is "broken" — the consumer can see the new head but its predecessor's `next` pointer is still null. The consumer must handle this by stalling. This means `pop` is *not* strictly wait-free; the consumer can wait for a producer to finish linking.

In practice, the wait is short (one cache miss). For most uses, "essentially wait-free" is good enough.

### 10.4 Helping mechanism — a concrete pattern

**Surface claim:** "Slow threads get help from fast threads."

**Between the lines:** The pattern: every thread, before doing its own work, scans an "announcements" array. If anyone has a pending operation (older than some threshold), help them complete it. Then do your own work. Bounded help time gives bounded latency.

**Concrete example — wait-free queue with helping (sketch):**

```
operation_slot[N]: per-thread pending operation

enqueue(value):
    slot = operation_slot[my_id]
    slot.value = value
    slot.state = PENDING
    for each thread t with pending op older than (now - epsilon):
        help_complete(t.slot)
    help_complete(slot)

help_complete(slot):
    // Atomically apply slot.value to the data structure
    // Multiple helpers race; CAS ensures one succeeds
    if slot.state == PENDING:
        try_apply_to_queue(slot.value)
        slot.state = DONE
```

The result: even if other threads are constantly racing, your operation eventually gets help from a thread that succeeds, and your latency is bounded.

**Real example:** Kogan & Petrank's wait-free MPMC queue (PPoPP 2011). Used in some real-time embedded systems. About 3-5x slower than Michael-Scott in average case, but with bounded worst case.

---

## 11. Performance, Measured in Cycles and Cache Lines

### 11.1 False sharing — concrete demonstration

**Surface claim:** "Different variables on the same cache line cause contention."

**Between the lines:** A cache line is 64 bytes (typical x86) or 128 bytes (Apple Silicon). If two threads modify variables that happen to fall on the same line, every modification by one invalidates the other's cached copy. Performance collapses.

**Concrete example — a benchmark you can run:**

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::thread;
use std::time::Instant;

// Version A: counters in adjacent fields (false sharing)
struct CountersBad {
    a: AtomicU64,  // bytes 0-7
    b: AtomicU64,  // bytes 8-15 — same cache line as a!
}

// Version B: counters padded to separate cache lines
#[repr(align(64))]
struct PaddedCounter { v: AtomicU64 }
struct CountersGood {
    a: PaddedCounter,  // line 0
    b: PaddedCounter,  // line 1
}

fn bench<F>(label: &str, f: F) where F: Fn() {
    let start = Instant::now();
    f();
    println!("{}: {:?}", label, start.elapsed());
}

fn main() {
    let bad = std::sync::Arc::new(CountersBad {
        a: AtomicU64::new(0), b: AtomicU64::new(0)
    });
    let good = std::sync::Arc::new(CountersGood {
        a: PaddedCounter { v: AtomicU64::new(0) },
        b: PaddedCounter { v: AtomicU64::new(0) },
    });

    bench("bad (false sharing)", || {
        let bad1 = bad.clone();
        let bad2 = bad.clone();
        let t1 = thread::spawn(move || {
            for _ in 0..10_000_000 { bad1.a.fetch_add(1, Ordering::Relaxed); }
        });
        let t2 = thread::spawn(move || {
            for _ in 0..10_000_000 { bad2.b.fetch_add(1, Ordering::Relaxed); }
        });
        t1.join().unwrap(); t2.join().unwrap();
    });

    bench("good (padded)", || {
        let good1 = good.clone();
        let good2 = good.clone();
        let t1 = thread::spawn(move || {
            for _ in 0..10_000_000 { good1.a.v.fetch_add(1, Ordering::Relaxed); }
        });
        let t2 = thread::spawn(move || {
            for _ in 0..10_000_000 { good2.b.v.fetch_add(1, Ordering::Relaxed); }
        });
        t1.join().unwrap(); t2.join().unwrap();
    });
}
```

**Typical results on modern x86:**
- bad: ~2.5 seconds
- good: ~250 ms

**~10x speedup just from padding.** Each fetch_add on the bad version causes a cache line bounce — the line ping-pongs between core 0's L1 and core 1's L1.

**Concrete example — finding false sharing in C#:**

```csharp
[StructLayout(LayoutKind.Explicit, Size = 128)]
struct PaddedLong {
    [FieldOffset(64)] public long Value;
}

class Counters {
    public PaddedLong A;  // each takes a 128-byte slot
    public PaddedLong B;
}
```

Or use `[StructLayout(LayoutKind.Sequential, Pack = 64)]` patterns.

**Hidden detail — intentional cache-line affinity:** Sometimes you *want* multiple variables on the same line (to read them all in one cache miss). Read-mostly data benefits from packing. Write-heavy contended data needs padding. Profile to know.

### 11.2 Cache line ping-pong — quantified

**Surface claim:** "Contended cache lines are slow."

**Between the lines:** Moving a cache line from one core's L1 to another's costs roughly 100-200ns on modern CPUs (depends on whether they share L2/L3 — within socket vs. across socket).

**Concrete numbers (Intel Xeon Skylake):**
- L1 hit: 4-5 cycles (~1.5ns at 3GHz)
- Same-socket cache line transfer: ~30-50ns
- Cross-socket cache line transfer (NUMA): ~100-300ns
- Memory access (DRAM miss): ~70-100ns local, 100-300ns remote

**Concrete example — a hot counter on a 16-core machine:**

```rust
// Naive: all 16 threads incrementing one counter
static COUNTER: AtomicU64 = AtomicU64::new(0);
// Each fetch_add: cache line bounces between cores. ~50ns per increment.
// 16 cores * 1 increment per 50ns = 320M ops/sec total. Per thread: 20M.

// Better: per-thread counters, summed periodically.
struct ShardedCounter { shards: Vec<PaddedAtomicU64> }
fn increment(c: &ShardedCounter, thread_id: usize) {
    c.shards[thread_id].v.fetch_add(1, Ordering::Relaxed);  // each thread hits its own line
}
fn read(c: &ShardedCounter) -> u64 {
    c.shards.iter().map(|s| s.v.load(Ordering::Relaxed)).sum()
}
// Each fetch_add: own cache line, no contention. ~1ns per increment.
// 16 cores * 1B ops/sec = 16B total. Per thread: 1B. 50x speedup.
```

This pattern (sharding counters) is universal in high-throughput systems.

### 11.3 Memory barrier costs — concrete numbers

**Surface claim:** "Barriers are expensive."

**Between the lines:** Different barriers have wildly different costs.

**On x86 (Skylake-class):**

| Operation | Approx cycles |
|---|---|
| Regular load (L1 hit) | 4 |
| Regular store | 1 (pipelined, hidden) |
| `mov` with implicit acquire (a regular load, since x86 is acquire-by-default) | 4 |
| `mov` with implicit release (regular store) | 1 |
| `lock xadd` (atomic increment, uncontended) | 25 |
| `lock cmpxchg` (CAS, uncontended) | 25 |
| `mfence` | 30 |
| `lock`-prefixed instruction under heavy contention | 100-1000 |

**On ARM (Apple M1):**

| Operation | Approx cycles |
|---|---|
| Regular load | 3 |
| `ldar` (load-acquire) | 3-5 |
| `stlr` (store-release) | 3-5 |
| `dmb ish` (full barrier) | 10-30 |
| `casa` (CAS with acquire) | 10-15 uncontended |

**Concrete example — a SeqCst load on x86 vs ARM:**

```rust
// Rust source
let v = atomic.load(Ordering::SeqCst);
```

**On x86:** Compiles to a regular `mov`. SeqCst loads on x86 are free. (SeqCst stores need `mfence` though.)

**On ARM:** Compiles to `ldar`. Slightly more expensive than relaxed load.

This is why "x86 hides bugs": Acquire and SeqCst loads have the same cost on x86 (free), so developers can't tell them apart by performance. On ARM, they compile differently.

### 11.4 Backoff strategies — what to do under contention

**Surface claim:** "Backoff reduces contention."

**Between the lines:** When CAS keeps failing, blind retry causes more contention. Better strategies:

**1. Spin-pause (microseconds):**

```rust
for _ in 0..32 {
    std::hint::spin_loop();  // x86: PAUSE; ARM: YIELD
}
```

`PAUSE` tells the CPU "I'm in a spin loop." Effects:
- Saves power (slows speculative execution).
- Hints the memory subsystem to prefer other cores' requests.
- ~5-100ns per pause.

**2. Exponential backoff:**

```rust
let mut backoff = 1;
loop {
    if try_op().is_ok() { return; }
    for _ in 0..backoff { std::hint::spin_loop(); }
    backoff = (backoff * 2).min(1024);
}
```

Doubles the wait each failure, capped. Naturally adapts to contention level.

**3. OS yield:**

```rust
std::thread::yield_now();  // tell scheduler "run something else"
```

Cost: ~1-3μs (kernel call). Use when spinning more than ~1μs.

**4. Park (block):**

```rust
std::thread::park();  // sleep until unparked
```

Cost: similar to mutex slow path, ~3μs sleep + wake. Use when waiting for a specific event.

**Concrete example — `crossbeam::utils::Backoff` does all four progressively:**

```rust
use crossbeam_utils::Backoff;
let backoff = Backoff::new();
loop {
    if try_op().is_ok() { return; }
    if backoff.is_completed() {
        std::thread::yield_now();
    } else {
        backoff.snooze();  // adaptive spin → yield → block
    }
}
```

### 11.5 NUMA — cross-socket considerations

**Surface claim:** "Multi-socket systems have non-uniform memory access."

**Between the lines:** A 2-socket server has memory attached to each CPU. Access to "your" socket's memory is fast (100ns); access to the other socket's memory is slow (300ns). Concurrency primitives that work great on a single socket can collapse across sockets.

**Concrete example — locking on NUMA:**

A mutex held by a thread on socket 0, contended by threads on socket 1, causes the cache line to ping-pong across sockets — each cycle costs 300ns instead of 50ns. Throughput drops 6x.

**Solutions:**
- **Pin threads:** `taskset` (Linux), `SetThreadAffinityMask` (Windows). Keep threads on one socket.
- **Per-socket data:** maintain a separate copy per NUMA node, sync periodically.
- **NUMA-aware allocators:** `numactl --localalloc` or `mbind` ensures memory is allocated on the local node.

**Concrete example — Linux `numactl`:**

```bash
numactl --cpunodebind=0 --membind=0 ./myserver
```

Pins the process to NUMA node 0 (CPU and memory). Eliminates cross-socket traffic.

---

## 12. Interview Traps, Each With a Worked Scenario

### 12.1 "Volatile" doesn't mean atomic

**Trap:** "Use volatile for thread safety."

**Reality:** In C/C++, `volatile` only prevents *compiler* optimization — no atomicity, no memory barriers across threads. In Java/C#, `volatile` does provide acquire/release but still doesn't make compound operations atomic.

**Concrete example — a real bug from 2010 in early C# code:**

```csharp
private volatile int _counter = 0;

void Increment() {
    _counter++;  // BUG: looks atomic, isn't.
}
```

The IL:
```
ldsfld _counter   // volatile read (acquire)
ldc.i4.1
add
stsfld _counter   // volatile write (release)
```

Two threads can both load the same value. Lost update. The fix: `Interlocked.Increment(ref _counter)`.

### 12.2 Double-checked locking — the broken pattern

**Trap:** Without proper memory barriers, double-checked locking publishes half-constructed objects.

**Concrete example — the bug in C++ pre-C++11:**

```cpp
class Singleton {
    static Singleton* instance;
    static std::mutex mtx;
public:
    static Singleton* get() {
        if (!instance) {  // ① check (no barrier)
            std::lock_guard<std::mutex> lock(mtx);
            if (!instance) {
                instance = new Singleton();  // ② BUG
            }
        }
        return instance;
    }
};
```

**The bug at ②:** `instance = new Singleton()` is three operations:
1. Allocate memory.
2. Construct object in memory.
3. Assign pointer to `instance`.

The compiler/CPU can reorder 2 and 3. So `instance` might point to allocated-but-uninitialized memory. Another thread doing the unsynchronized check at ① can see the non-null pointer and use the half-constructed object.

**Fix in C++11:**
```cpp
static std::atomic<Singleton*> instance;
// ...
Singleton* p = instance.load(std::memory_order_acquire);
if (!p) {
    std::lock_guard<std::mutex> lock(mtx);
    p = instance.load(std::memory_order_relaxed);
    if (!p) {
        p = new Singleton();
        instance.store(p, std::memory_order_release);
    }
}
return p;
```

The release store guarantees the construction happens-before the publication.

**Fix in C#:**
```csharp
private static volatile Singleton _instance;
// volatile gives release on write, acquire on read.
```

Or just use `Lazy<T>`.

**Fix in Java:**
```java
private static volatile Singleton instance;
// volatile in Java (since JMM v2, Java 5) gives the right semantics.
```

### 12.3 Mutex doesn't make all operations atomic

**Trap:** "I have a mutex; my code is thread-safe."

**Concrete example — race despite mutex:**

```rust
let map = Mutex::new(HashMap::new());
// Thread A
let len = map.lock().unwrap().len();
if len > 0 {
    let val = map.lock().unwrap().get(&key);  // map could be cleared between!
}
```

Each `lock` is independent. Between the two locks, another thread can modify the map. Solution: hold a single lock spanning both operations.

```rust
let guard = map.lock().unwrap();
let len = guard.len();
if len > 0 {
    let val = guard.get(&key);
}
```

### 12.4 Acquire/release on different variables doesn't synchronize

**Trap:** "I released X, you acquire Y, why don't you see my changes?"

**Concrete example:**

```rust
static FLAG_A: AtomicBool = AtomicBool::new(false);
static FLAG_B: AtomicBool = AtomicBool::new(false);
static mut DATA: i32 = 0;

// Thread 1
unsafe { DATA = 42; }
FLAG_A.store(true, Ordering::Release);

// Thread 2
while !FLAG_B.load(Ordering::Acquire) {}  // BUG: waiting on B, but A was released
unsafe { println!("{}", DATA); }
```

The release on A and the acquire on B don't synchronize. Thread 2 might see DATA=0.

**Fix:** Use the *same* variable for the synchronization:

```rust
// Thread 1
unsafe { DATA = 42; }
FLAG_A.store(true, Ordering::Release);

// Thread 2
while !FLAG_A.load(Ordering::Acquire) {}
unsafe { println!("{}", DATA); }
```

Or use `fence(SeqCst)` for cross-variable ordering (but it's broader/more expensive).

### 12.5 "Is x86 sequentially consistent?" — Trick question

**Trap:** Many people say yes.

**Reality:** x86 is **TSO** (Total Store Order), not SC. The one allowed reordering is store→load (a load can pass an earlier store *to a different address*).

**Concrete example — Dekker:**

```c
int x = 0, y = 0, r1, r2;

void thread_a() { x = 1; r1 = y; }
void thread_b() { y = 1; r2 = x; }
```

Under SC, `(r1=0, r2=0)` is impossible (one store must come first globally; the corresponding load must see it).

Under TSO (x86), `(r1=0, r2=0)` *is* observable. The store sits in the store buffer; the load goes ahead and reads the other variable. Both threads can observe stale values.

**Fix on x86:** `mfence` or any LOCK-prefixed instruction between the store and the load.

```c
void thread_a() {
    x = 1;
    asm volatile("mfence" ::: "memory");
    r1 = y;
}
```

### 12.6 "Can a spinlock deadlock?" — Yes, especially on a single CPU

**Trap:** "Spinlocks just spin; they can't deadlock."

**Reality:** On a single CPU (or with a high-priority spinner blocking the holder), spinlocks can deadlock indefinitely.

**Concrete example — single-core deadlock:**

```c
// Single-core embedded system. Two tasks, T_high and T_low.
T_low: spinlock.lock();        // acquires
T_low: doing work...
T_high: starts running.        // preempts T_low
T_high: spinlock.lock();       // spins forever!
// T_low never gets CPU because T_high has higher priority.
```

This is why kernel spinlocks always disable preemption while held. Userspace shouldn't use spinlocks except in very specific scenarios.

### 12.7 Mutex vs binary semaphore — what's the difference?

**Trap:** "They're the same."

**Reality:** Three key differences:

**1. Ownership:**
- Mutex has an owner (the thread that locked it). Only that thread can unlock.
- Semaphore has no owner. Any thread can post.

**Concrete example:** With a binary semaphore, you can implement signaling: thread A waits; thread B posts. Mutex can't do this — only the locker can unlock.

```rust
let sem = Semaphore::new(0);
// Thread A
sem.acquire().await;  // waits
// Thread B
sem.add_permits(1);  // signals A
```

**2. Priority inheritance:** Mutex can inherit (boost owner's priority). Semaphore cannot (no owner).

**3. Reentrancy:** A non-reentrant mutex can't be locked twice by the same thread (deadlock or error). A binary semaphore allows the same thread to "acquire twice" if you post in between.

### 12.8 "Lock-free means no locks?"

**Trap:** Lock-free means no syntactic locks.

**Reality:** Lock-free is a *progress guarantee*: at least one thread always makes progress in a bounded number of steps. A lock-free algorithm uses atomics, not mutexes, but individual threads can still starve. **Wait-free** is the stronger property (every thread makes progress).

**Concrete example — starvation under lock-free:**

```rust
// Hypothetical: 7 threads constantly succeeding their CAS, 1 thread always failing.
// The system makes progress (7 threads' worth). Lock-free property holds.
// The 8th thread starves. Not wait-free.
```

In practice, retry counts under lock-free are bounded by hardware effects (cache line contention has natural fairness over time). Pure starvation is rare but possible.

### 12.9 "Are atomic operations always faster than mutexes?"

**Trap:** "Atomics are lower level, so faster."

**Reality:** Depends entirely on workload.

**Concrete example — atomic loses:**

```rust
// Critical section: 100ns of work.
// Contention: 8 threads.

// Mutex: 8 threads serialize. Throughput: 1 op per 100ns = 10M ops/sec.
// CAS loop: average 4 retries per success. Each retry costs cache miss (~50ns) + work (100ns).
// Throughput: 1 op per 600ns per thread = 1.6M ops/sec.

// Mutex wins by 6x here.
```

**Concrete example — atomic wins:**

```rust
// Critical section: 5ns (single counter increment).
// Contention: 8 threads.

// Mutex: 25ns lock + 5ns work + 25ns unlock = 55ns per op.
// Atomic fetch_add: 25ns per op uncontended, 50ns under contention.
// Atomic ~2x faster.
```

The crossover depends on critical section length, contention, and architecture. Profile.

### 12.10 "Can `volatile` in C++ make my code thread-safe?"

**Trap:** It's a keyword for memory access; that must mean it's for threading.

**Reality:** In C++, `volatile` is for memory-mapped I/O — preventing the compiler from optimizing away accesses to hardware registers. It provides no thread safety. Use `std::atomic` for threading.

**Concrete example:**

```cpp
volatile int x = 0;  // for, e.g., a hardware register
// Thread A
x = 1;
// Thread B
while (x == 0) {}  // may compile to infinite loop on optimization, or read torn value
```

`volatile` prevents the compiler from caching x in a register. But:
- It doesn't make x = 1 atomic (might tear).
- It doesn't add memory barriers.
- It doesn't affect CPU reordering.

The fix is `std::atomic<int>`.

### 12.11 Async mutexes vs sync mutexes

**Trap:** "I'll use `tokio::sync::Mutex` everywhere; it's safer."

**Reality:** Async mutexes yield the *task* (cooperative); sync mutexes block the *thread*. Mixing them wrong has consequences.

**Concrete example — using sync mutex across .await (BAD):**

```rust
let m = std::sync::Mutex::new(data);
let guard = m.lock().unwrap();
some_async_operation().await;  // BUG: holds OS thread; blocks runtime
guard.modify();
```

If your async runtime has 8 worker threads and 8 tasks all do this, all threads are blocked, and *no* tasks can make progress.

**Fix:** Use `tokio::sync::Mutex` for cross-await:

```rust
let m = tokio::sync::Mutex::new(data);
let guard = m.lock().await;
some_async_operation().await;
guard.modify();
```

**When to still use sync mutex:** When the critical section doesn't await. Sync mutexes are faster (no async overhead).

### 12.12 The TOCTOU race

**Trap:** "I checked it; now I'll act on it."

**Concrete example — TOCTOU file race:**

```rust
if file_exists(path) {     // ① check
    read_file(path);       // ② use
}
```

Between ① and ②, the file could be deleted. The check is meaningless without locking or transactional semantics.

**Concrete example — TOCTOU in a HashMap:**

```rust
if !map.contains_key(&k) {
    map.insert(k, v);  // race: another thread may have inserted between
}
```

**Fix:** Use a single atomic operation:
```rust
map.entry(k).or_insert(v);  // atomic check-and-insert
```

Or in C#:
```csharp
dict.TryAdd(key, value);  // atomic
```

### 12.13 Coffman conditions for deadlock — know all four

**Trap:** "There must be a cycle."

**Reality:** Cycle (circular wait) is necessary but only one of four needed conditions.

1. **Mutual exclusion** (resources can't be shared)
2. **Hold and wait** (holding a resource while waiting for another)
3. **No preemption** (resources can't be forcibly released)
4. **Circular wait** (cycle in wait-for graph)

To prevent deadlock, break *any one*. Most prevention strategies break #4 (lock ordering) or #3 (try_lock with timeout).

### 12.14 Forgetting that `clone()` of `Arc` is atomic

**Trap:** Performance concern.

**Reality:** `Arc::clone` is `fetch_add(1, Relaxed)` — a single atomic op. ~25ns. Not free, not bad.

**When it matters:** Cloning Arc in a tight loop on a hot path can show up in profiles. Use `&Arc<T>` references when possible.

---

## Quick Reference: When to Use What — With Concrete Triggers

| If you're writing... | Use this | Because |
|---|---|---|
| Counter incremented from many threads | `AtomicU64::fetch_add(1, Relaxed)` | No lock needed; relaxed enough |
| Hot counter (millions/sec from many cores) | Sharded counters with periodic sum | Avoids cache line contention |
| One-time initialization | `OnceLock` / `Lazy` / `std::call_once` | Built-in; correctly handles DCL |
| Protecting a small struct, low contention | `Mutex<T>` | Simple, correct, fast uncontended |
| Read-heavy data, occasional writes | `RwLock` or `arc-swap` | Multiple readers don't block |
| Tasks waiting for completion | `CountdownEvent` / `WaitGroup` / `JoinSet` | Single-use, clear semantics |
| Producer/consumer with backpressure | Bounded channel | Slows producer to consumer rate |
| Fast IPC between two threads | SPSC ring buffer (`crossbeam::queue::ArrayQueue` SPSC) | Wait-free, no CAS |
| MPMC queue | `crossbeam::queue::ArrayQueue` (Vyukov) | Fastest practical MPMC |
| Concurrent hash map | `dashmap` (Rust), `ConcurrentHashMap` (Java), `ConcurrentDictionary` (C#) | Don't roll your own |
| Wait for predicate to become true | `Condvar` with `while`-loop check | Standard pattern |
| One signal to many waiters | `ManualResetEvent` / `tokio::sync::Notify` | Broadcast |
| Limit concurrency to N | `Semaphore::new(N)` | Counting limit |

---

## Final Mastery Tips for the Interview

1. **When asked about a race, don't say "race condition" — distinguish:**
   - **Data race:** unsynchronized memory access (UB in C++/Rust). Mechanically defined.
   - **Race condition:** logic bug due to timing (TOCTOU, etc.). Application-level concept.
   - The two overlap but aren't identical. Showing this distinction signals depth.

2. **Always identify the linearization point** of any concurrent operation. "The successful CAS instruction is the linearization point of push." This single sentence demonstrates you understand correctness reasoning.

3. **Reason from the memory model, not the hardware.** "Assuming weak ordering, this code is broken because there's no synchronizes-with edge between the producer and consumer." Don't say "on x86 this is fine" — that's brittle.

4. **Quantify when you can.** "Cache line bounce is ~100ns." "Mutex uncontended is ~25ns; contended is ~3μs." Numbers turn vague intuitions into concrete trade-offs.

5. **Mention real systems.** "Linux kernel uses RCU for the dentry cache." "Java's CHM uses sharding plus lock-free per-bucket." This shows you know theory in practice.

6. **For lock-free questions, mention ABA upfront.** Even if you don't implement protection, acknowledging it signals expertise.

7. **Know the Coffman conditions and how to break each.** Especially #4 (lock ordering) and #3 (try_lock + backoff).

8. **Remember: your mutex doesn't make compound operations atomic.** Multiple `lock` calls = multiple critical sections. Wrap the entire compound op in one lock.

9. **Async ≠ concurrency primitives.** Async/await schedules tasks; you still need mutexes, channels, atomics underneath.

10. **When in doubt, prefer message passing over shared state.** Channels eliminate whole classes of bugs. Use shared state only when channels can't express the pattern.

Good luck.
