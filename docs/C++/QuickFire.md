# C++ Quick-Fire Questions  Complete Reference

**Purpose:** Rapidly gauge candidate's depth across C++ fundamentals in 5–7 minutes. Each question expects a 15–30 second answer. Score: correct = 1 point, partially correct = 0.5, wrong/no answer = 0.

---

## Language Fundamentals (10 questions)

| # | Question | Expected Answer | Red Flag If... |
|---|----------|----------------|----------------|
| 1 | Can constructors be virtual? | No  vtable not set up during construction | Says "yes" |
| 2 | Can destructors be virtual? Should they be? | Yes. Must be virtual if class is used polymorphically. | Doesn't mention UB from non-virtual base dtor |
| 3 | What does `= delete` do? | Explicitly disables a function (copy ctor, assignment, etc.) | Confuses with `delete` operator |
| 4 | What is the Rule of Five? | If you define one of: dtor, copy ctor, copy assign, move ctor, move assign  define all five | Says "Rule of Three" without mentioning C++11 |
| 5 | What is `nullptr` vs `NULL` vs `0`? | `nullptr`: type-safe null pointer literal (C++11). `NULL`: macro, often `0` or `(void*)0`. `0`: integer. | Treats them as identical |
| 6 | What is an rvalue reference (`&&`)? | Reference to a temporary/expiring object. Enables move semantics. | Cannot explain when/why to use it |
| 7 | What is `std::forward`? | Perfect forwarding  preserves value category (lvalue/rvalue) in template forwarding references | Confuses with `std::move` |
| 8 | What is undefined behavior? Give 3 examples. | UB: compiler can do anything. Examples: null deref, signed overflow, use-after-free, data race, out-of-bounds | Cannot name examples |
| 9 | Difference between `struct` and `class`? | Only default access: struct=public, class=private. Functionally identical. | Thinks structs can't have methods |
| 10 | What is ODR (One Definition Rule)? | Each entity must have exactly one definition across all translation units (except inline/template) | Never heard of it |

---

## Memory & Ownership (8 questions)

| # | Question | Expected Answer | Red Flag If... |
|---|----------|----------------|----------------|
| 11 | What does `std::move` do? | `static_cast` to rvalue reference. Doesn't move anything  enables move semantics. | Thinks it physically moves data |
| 12 | `unique_ptr` vs `shared_ptr`  when each? | unique: sole ownership, zero overhead. shared: multiple owners, ref-counted. | Always defaults to shared_ptr |
| 13 | What is a dangling pointer? | Pointer to freed/invalid memory. | Cannot name causes |
| 14 | Stack vs heap  who's faster? | Stack: O(1) bump allocation. Heap: complex allocator, fragmentation risk. | Doesn't know why |
| 15 | What is placement new? | Constructs object at pre-allocated address without allocating. | Never heard of it |
| 16 | What is `std::weak_ptr` for? | Break circular references; non-owning observation of shared objects. | Cannot give a use case |
| 17 | Can `unique_ptr` be in a `std::vector`? | Yes  vector uses move semantics internally (C++11). | Says no |
| 18 | What is RAII? Give one example. | Resource Acquisition Is Initialization. Example: `lock_guard`, `unique_ptr`, file handles. | Cannot explain the concept |

---

## Concurrency (6 questions)

| # | Question | Expected Answer | Red Flag If... |
|---|----------|----------------|----------------|
| 19 | Data race vs race condition? | Data race: UB (unsynchronized concurrent access, one write). Race condition: logic bug from scheduling. | Treats as synonyms |
| 20 | What is `std::atomic`? | Lock-free operations on single values. Hardware CAS/LL-SC. | Thinks it's just a mutex wrapper |
| 21 | What is a spurious wakeup? | `condition_variable::wait` can return without notification. Always use predicate loop. | Never heard of it |
| 22 | `std::mutex` vs `std::shared_mutex`? | mutex: exclusive. shared_mutex: multiple readers OR one writer (reader-writer lock). | Cannot explain |
| 23 | What is `std::scoped_lock` (C++17)? | RAII lock for multiple mutexes with deadlock avoidance. Replaces `std::lock` + `lock_guard`. | Doesn't know it exists |
| 24 | Can you use `std::mutex` in an ISR? | No. Blocking in interrupt context is forbidden. Use lock-free mechanisms. | Says yes |

---

## Templates & Type System (6 questions)

| # | Question | Expected Answer | Red Flag If... |
|---|----------|----------------|----------------|
| 25 | What is SFINAE? | Substitution Failure Is Not An Error  invalid template substitution silently discards specialization. | Cannot explain at all |
| 26 | What are Concepts (C++20)? | Named constraints on template parameters. Readable alternative to SFINAE. | Hasn't heard of them |
| 27 | `constexpr` vs `const`? | const: can't modify. constexpr: must be computable at compile time. | Treats as identical |
| 28 | What is `decltype`? | Deduces the type of an expression. Used in trailing return types and template meta. | Confuses with `auto` |
| 29 | What is a variadic template? | Template accepting arbitrary number of type/value params via `...` parameter pack. | Cannot show syntax |
| 30 | What is `std::any` vs `std::variant`? | `any`: type-erased (any type, heap-possible). `variant`: closed set of types, stack-allocated. | Doesn't know either |

---

## Scoring Summary

| Range | Interpretation |
|-------|---------------|
| 25–30 | Expert  deep fluency |
| 18–24 | Proficient  solid senior-level |
| 12–17 | Competent  mid-level, knows core well |
| 6–11 | Developing  junior, significant gaps |
| 0–5 | Below bar for this role |

**Candidate Score: __/30**

---

## Interviewer Notes

**Strongest area:**

**Weakest area:**

**Notable responses (good or bad):**

**Time taken for 30 questions:** ___ minutes (target: 7–10 min)
