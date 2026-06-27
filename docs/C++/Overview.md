# C++ Interview Questions

**Focus: Modern C++14/17/20 | Memory Management | RAII | Concurrency | OOP for Constrained Environments | Design Patterns**

---

## Section Index

| File | Topic |
|------|-------|
| [QuickFire.md](QuickFire.md) | Brief language-gauge questions (30s each) |
| [MemoryManagement.md](MemoryManagement.md) | Smart pointers, RAII, allocators, placement new |
| [Concurrency.md](Concurrency.md) | Threads, atomics, lock-free, memory ordering |
| [OOP_DesignPatterns.md](OOP_DesignPatterns.md) | Patterns for embedded/constrained systems (1-hr judgeable) |
| [ModernCpp.md](ModernCpp.md) | C++14/17/20 features relevant to firmware |
| [ConstrainedEnvironments.md](ConstrainedEnvironments.md) | No-heap, deterministic, real-time C++ |

---

## Quick-Fire Questions (Language Rating Gauge)

These are designed to be answered in under 30 seconds. They reveal depth of understanding quickly.

### Constructors & Destructors

⚡ **Can constructors be virtual?**
> **No.** The vtable pointer is initialized during construction. The object's type isn't fully established until construction completes, so virtual dispatch during construction is meaningless. Use the Factory pattern or clone idiom instead.

⚡ **Can destructors be virtual? Should they be?**
> **Yes.** If a class is designed for polymorphic use (base pointer deletion), the destructor **must** be virtual. Otherwise, deleting a derived object through a base pointer is undefined behavior.

⚡ **What happens if you throw an exception from a destructor?**
> If the destructor is called during stack unwinding (due to another exception), `std::terminate()` is called. Mark destructors `noexcept` (implicit since C++11).

⚡ **What is the order of construction in a class hierarchy with multiple inheritance?**
> Virtual bases (left-to-right, depth-first), then direct bases (left-to-right declaration order), then members (declaration order), then the constructor body.

### Memory & Ownership

⚡ **What's the difference between `std::unique_ptr` and `std::shared_ptr`?**
> `unique_ptr`: exclusive ownership, zero overhead, non-copyable, movable. `shared_ptr`: shared ownership via reference counting, thread-safe control block, ~2 pointer overhead.

⚡ **When would you use `std::weak_ptr`?**
> To break circular references in `shared_ptr` graphs (e.g., observer patterns, parent-child trees) and for non-owning cache references.

⚡ **What does `std::move` actually do?**
> It performs a `static_cast` to an rvalue reference (`T&&`). It doesn't move anythingit enables the move constructor/assignment to be called.

⚡ **What is placement new?**
> Constructs an object at a pre-allocated memory address without allocating new memory. Used in memory pools, arena allocators, and embedded systems with fixed memory maps.

### Type System & Templates

⚡ **What is SFINAE?**
> Substitution Failure Is Not An Error. If template argument substitution fails, the specialization is silently removed from the overload set rather than causing a compile error.

⚡ **What replaced SFINAE in modern C++?**
> `if constexpr` (C++17) for compile-time branching, and Concepts (C++20) for constraining templates with readable syntax.

⚡ **What's the difference between `constexpr` and `const`?**
> `const`: value cannot be modified after initialization (can be runtime). `constexpr`: value must be computable at compile time. `consteval` (C++20): function *must* be evaluated at compile time.

⚡ **What is `std::optional` and when do you use it?**
> A vocabulary type (C++17) representing a value that may or may not be present. Replaces sentinel values, nullable raw pointers, and out-parameters for "maybe" return types.

### Concurrency Quick-Fire

⚡ **What's the difference between `std::mutex` and `std::atomic`?**
> Mutex: blocks threads, protects critical sections. Atomic: lock-free operations on single values using hardware CAS/LL-SC instructions. Atomics are faster for simple shared state.

⚡ **What is a data race vs. a race condition?**
> Data race: two threads access the same memory location, at least one writes, no synchronization (undefined behavior). Race condition: program correctness depends on thread scheduling (logic bug, not UB).

⚡ **What memory orderings does C++ provide?**
> `memory_order_relaxed`, `memory_order_acquire`, `memory_order_release`, `memory_order_acq_rel`, `memory_order_seq_cst` (default). Sequential consistency is safest but most expensive on ARM/RISC-V.

---

*Detailed sections follow in separate files.*
