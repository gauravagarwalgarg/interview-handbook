# C++ Coding & Output Questions

> Real interview questions from Google, Meta, HFT firms (Citadel, Jane Street, Tower Research).
> Focus: "What does this print?" / "Is this well-defined?" / "What's wrong?"

---

### Q1: Virtual Dispatch in Constructor Level: SDE 2

**Question:**
```cpp
#include <iostream>
struct Base {
    Base() { print(); }
    virtual void print() { std::cout << "Base\n"; }
};
struct Derived : Base {
    int x = 42;
    void print() override { std::cout << "Derived " << x << "\n"; }
};
int main() { Derived d; }
```
What does this print?

**Answer:**
```
Base
```
During `Base` constructor, the vtable still points to `Base::print`. The `Derived` object isn't constructed yet, so virtual dispatch resolves to `Base::print`.

**Follow-up:** What if `print()` is pure virtual in `Base`? → Undefined behavior (crash on most implementations).

---

### Q2: Move After std::move Level: SDE 2

**Question:**
```cpp
#include <iostream>
#include <string>
#include <vector>
int main() {
    std::string s = "hello";
    std::vector<std::string> v;
    v.push_back(std::move(s));
    std::cout << s.size() << "\n";
    std::cout << v[0] << "\n";
}
```

**Answer:**
```
0
hello
```
After `std::move`, `s` is in a valid-but-unspecified state. For `std::string`, most implementations leave it empty. `v[0]` holds the moved-from content.

**Follow-up:** Is it safe to call `s.clear()` after the move? → Yes, moved-from objects must be in a destructible/assignable state.

---

### Q3: RAII and Exception Safety Level: SDE 3

**Question:**
```cpp
#include <iostream>
#include <memory>
void process(std::unique_ptr<int> a, std::unique_ptr<int> b) {
    std::cout << *a + *b << "\n";
}
int main() {
    process(std::unique_ptr<int>(new int(1)), std::unique_ptr<int>(new int(2)));
}
```
Is this code safe pre-C++17?

**Answer:**
Pre-C++17: **potential memory leak**. The compiler could evaluate: `new int(1)`, `new int(2)`, then construct the `unique_ptr`s. If the second `new` throws, the first allocation leaks.

C++17+: Safe. Function arguments have indeterminate sequencing but each full-expression is sequenced.

Fix: Use `std::make_unique<int>(1)` allocation and `unique_ptr` construction are one expression.

**Follow-up:** Why does `make_unique` solve this? → It wraps allocation + construction into a single function call expression.

---

### Q4: Template SFINAE Puzzle Level: SDE 3

**Question:**
```cpp
#include <iostream>
#include <type_traits>
template<typename T>
typename std::enable_if<std::is_integral<T>::value, void>::type
foo(T t) { std::cout << "integral: " << t << "\n"; }

template<typename T>
typename std::enable_if<!std::is_integral<T>::value, void>::type
foo(T t) { std::cout << "other: " << t << "\n"; }

int main() {
    foo(42);
    foo(3.14);
    foo('A');
}
```

**Answer:**
```
integral: 42
other: 3.14
integral: 65
```
`char` is an integral type. SFINAE selects the integral overload for `char`, printing its integer value.

**Follow-up:** Rewrite using C++20 concepts. → `void foo(std::integral auto t)`.

---

### Q5: Smart Pointer Circular Reference Level: SDE 2

**Question:**
```cpp
#include <iostream>
#include <memory>
struct Node {
    std::shared_ptr<Node> next;
    ~Node() { std::cout << "~Node\n"; }
};
int main() {
    auto a = std::make_shared<Node>();
    auto b = std::make_shared<Node>();
    a->next = b;
    b->next = a;  // circular
    std::cout << "end of main\n";
}
```

**Answer:**
```
end of main
```
No destructors are called. Circular `shared_ptr` references prevent ref count from reaching zero → **memory leak**.

**Follow-up:** Fix it. → Use `std::weak_ptr<Node> next;` for at least one direction.

---

### Q6: Undefined Behavior Signed Overflow Level: SDE 2

**Question:**
```cpp
#include <iostream>
int main() {
    int x = INT_MAX;
    x += 1;
    std::cout << x << "\n";
}
```

**Answer:**
**Undefined behavior.** Signed integer overflow is UB in C++. The compiler may optimize assuming it never happens. Output varies: could print `INT_MIN` (wrap), crash, or be optimized away entirely.

**Follow-up:** How to safely check for overflow before it happens?
```cpp
if (x > INT_MAX - 1) { /* overflow would occur */ }
```

---

### Q7: const Correctness Level: SDE 2

**Question:**
```cpp
#include <iostream>
struct Foo {
    mutable int count = 0;
    void bar() const {
        count++;
        std::cout << count << "\n";
    }
};
int main() {
    const Foo f;
    f.bar();
    f.bar();
}
```

**Answer:**
```
1
2
```
`mutable` allows modification even in a `const` method on a `const` object. Common for caching/counting.

**Follow-up:** Is this thread-safe? → No. `mutable` doesn't imply synchronization. Need `mutable std::atomic<int> count` or a mutex.

---

### Q8: Static Initialization Order Fiasco Level: SDE 3

**Question:**
```cpp
// file1.cpp
#include <iostream>
extern int y;
int x = y + 1;

// file2.cpp
extern int x;
int y = x + 1;

// main.cpp
#include <iostream>
extern int x, y;
int main() { std::cout << x << " " << y << "\n"; }
```

**Answer:**
Undefined depends on linking order. If `file2.cpp` initializes first: `y = 0 + 1 = 1`, then `x = 1 + 1 = 2`. Output: `2 1`. If reversed: `1 2`.

**Fix:** Use the "Construct on First Use" idiom:
```cpp
int& getX() { static int x = getY() + 1; return x; }
```

**Follow-up:** Does C++11 guarantee thread-safe static local initialization? → Yes (§6.7).

---

### Q9: Object Slicing Level: SDE 2

**Question:**
```cpp
#include <iostream>
struct Base {
    virtual void who() { std::cout << "Base\n"; }
};
struct Derived : Base {
    int data = 99;
    void who() override { std::cout << "Derived " << data << "\n"; }
};
int main() {
    Derived d;
    Base b = d;  // slicing
    b.who();

    Base& ref = d;
    ref.who();
}
```

**Answer:**
```
Base
Derived 99
```
`Base b = d` slices copies only the `Base` part, vtable is `Base`'s. Through a reference/pointer, polymorphism works.

**Follow-up:** How does `std::vector<Base>` cause slicing? → Store `std::unique_ptr<Base>` instead.

---

### Q10: Multiple Inheritance Diamond Level: SDE 3

**Question:**
```cpp
#include <iostream>
struct A {
    int x = 10;
    virtual void foo() { std::cout << "A::foo " << x << "\n"; }
};
struct B : virtual A { void foo() override { std::cout << "B::foo\n"; } };
struct C : virtual A { void foo() override { std::cout << "C::foo\n"; } };
struct D : B, C { void foo() override { std::cout << "D::foo " << x << "\n"; } };

int main() {
    D d;
    d.foo();
    std::cout << d.x << "\n";  // unambiguous due to virtual inheritance
}
```

**Answer:**
```
D::foo 10
10
```
Virtual inheritance ensures a single `A` subobject. `D::foo` overrides both `B::foo` and `C::foo`. Without virtual inheritance, `d.x` would be ambiguous.

**Follow-up:** What's the memory layout overhead of virtual inheritance? → Extra vptr(s) + offset table for virtual base.

---

### Q11: Dangling Reference from Lambda Level: SDE 3

**Question:**
```cpp
#include <iostream>
#include <functional>
std::function<int()> make_counter() {
    int count = 0;
    return [&count]() { return ++count; };
}
int main() {
    auto counter = make_counter();
    std::cout << counter() << "\n";
}
```

**Answer:**
**Undefined behavior.** `count` is a local variable captured by reference. After `make_counter()` returns, the reference dangles.

**Fix:** Capture by value `[count]() mutable { return ++count; }`.

**Follow-up:** What if we want shared state between copies? → Use `std::shared_ptr<int>` captured by value.

---

### Q12: Exception in Destructor Level: SDE 3

**Question:**
```cpp
#include <iostream>
struct Bad {
    ~Bad() noexcept(false) { throw std::runtime_error("oops"); }
};
int main() {
    try {
        Bad b;
        throw std::runtime_error("first");
    } catch (const std::exception& e) {
        std::cout << e.what() << "\n";
    }
}
```

**Answer:**
**`std::terminate()` is called.** During stack unwinding from "first", `~Bad()` throws a second exception. Two simultaneous exceptions → terminate.

**Follow-up:** Why are destructors `noexcept` by default since C++11? → To prevent this exact scenario.

---

### Q13: Operator Overloading Implicit Conversion Level: SDE 2

**Question:**
```cpp
#include <iostream>
struct Integer {
    int val;
    Integer(int v) : val(v) {}  // implicit conversion
    Integer operator+(const Integer& rhs) const {
        return Integer(val + rhs.val);
    }
};
std::ostream& operator<<(std::ostream& os, const Integer& i) {
    return os << i.val;
}
int main() {
    Integer a(5);
    std::cout << a + 3 << "\n";   // works?
    std::cout << 3 + a << "\n";   // works?
}
```

**Answer:**
```
8
```
Then **compilation error** on `3 + a`. The member `operator+` requires `Integer` on the left. `a + 3` works because `3` implicitly converts to `Integer`.

**Fix:** Make `operator+` a free function: `Integer operator+(const Integer& l, const Integer& r)`.

**Follow-up:** How does `explicit` on the constructor change behavior? → `a + 3` also fails.

---

### Q14: Memory Layout and Padding Level: SDE 2 (HFT)

**Question:**
```cpp
#include <iostream>
struct A { char a; int b; char c; };
struct B { int b; char a; char c; };
int main() {
    std::cout << sizeof(A) << "\n";
    std::cout << sizeof(B) << "\n";
}
```

**Answer (typical x86-64):**
```
12
8
```
`A`: `char(1) + pad(3) + int(4) + char(1) + pad(3)` = 12.
`B`: `int(4) + char(1) + char(1) + pad(2)` = 8.
Struct members are ordered to minimize padding.

**Follow-up:** How to enforce no padding? → `#pragma pack(1)` or `__attribute__((packed))` but beware unaligned access penalties.

---

### Q15: volatile vs atomic Level: SDE 3 (HFT)

**Question:**
```cpp
#include <thread>
#include <iostream>
volatile int flag = 0;  // NOT atomic!
int data = 0;
void writer() {
    data = 42;
    flag = 1;
}
void reader() {
    while (flag == 0) {}
    std::cout << data << "\n";
}
int main() {
    std::thread t1(writer), t2(reader);
    t1.join(); t2.join();
}
```

**Answer:**
**Undefined behavior / data race.** `volatile` prevents compiler reordering but provides **no** memory ordering guarantees between threads. The reader might see `flag == 1` but `data == 0` due to CPU reordering.

**Fix:** Use `std::atomic<int> flag` with `std::memory_order_release` / `acquire`.

**Follow-up:** When is `volatile` appropriate? → Memory-mapped I/O, signal handlers never for thread synchronization.

---

### Q16: Perfect Forwarding Failure Level: SDE 3

**Question:**
```cpp
#include <iostream>
#include <utility>
void process(int& x) { std::cout << "lvalue: " << x << "\n"; }
void process(int&& x) { std::cout << "rvalue: " << x << "\n"; }

template<typename T>
void wrapper(T&& arg) {
    process(arg);  // Bug: always calls lvalue overload!
}
int main() {
    int a = 5;
    wrapper(a);
    wrapper(10);
}
```

**Answer:**
```
lvalue: 5
lvalue: 10
```
Named rvalue references are lvalues. `arg` has a name, so `process(arg)` always resolves to `process(int&)`.

**Fix:** `process(std::forward<T>(arg));`

**Follow-up:** Explain how `T&&` with template deduction becomes a "forwarding reference" via reference collapsing.

---

### Q17: decltype and Expression Categories Level: SDE 3

**Question:**
```cpp
#include <iostream>
int x = 0;
int& foo() { return x; }

int main() {
    decltype(x) a = 0;        // type?
    decltype((x)) b = a;      // type?
    decltype(foo()) c = a;    // type?
    b = 42;
    std::cout << a << " " << x << "\n";
}
```

**Answer:**
- `decltype(x)` → `int` (named entity → declared type)
- `decltype((x))` → `int&` (parenthesized → expression, lvalue → reference)
- `decltype(foo())` → `int&`
```
42 42
```
`b` is a reference to `a`, and modifying `b` changes `a`.

**Follow-up:** What does `decltype(std::move(x))` yield? → `int&&` (xvalue → rvalue reference).

---

### Q18: Aggregate Initialization Trap Level: SDE 2

**Question:**
```cpp
#include <iostream>
struct Point { int x; int y; int z; };
int main() {
    Point p1{1, 2};
    Point p2 = {1};
    std::cout << p1.z << " " << p2.y << " " << p2.z << "\n";
}
```

**Answer:**
```
0 0 0
```
Aggregate initialization zero-initializes missing members.

**Follow-up:** What happens if `Point` has a user-declared constructor? → No longer an aggregate; this syntax may not compile.

---

### Q19: auto and Brace Initialization Level: SDE 2

**Question:**
```cpp
#include <iostream>
#include <typeinfo>
int main() {
    auto a = {1, 2, 3};   // C++17
    // auto b{1, 2, 3};   // Error in C++17
    auto c{42};            // C++17
    std::cout << typeid(a).name() << "\n";  // implementation-defined
    std::cout << typeid(c).name() << "\n";
}
```

**Answer:**
- `auto a = {1,2,3}` → `std::initializer_list<int>`
- `auto c{42}` → `int` (since C++17; was `initializer_list<int>` in C++11/14)

**Follow-up:** Why was the rule changed for single-element braces? → `auto x{5}` being `initializer_list` was surprising and rarely intended.

---

### Q20: constexpr and Compile-Time Evaluation Level: SDE 3

**Question:**
```cpp
#include <iostream>
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}
int main() {
    constexpr int a = factorial(5);
    int n = 5;
    int b = factorial(n);  // compiles?
    std::cout << a << " " << b << "\n";
    static_assert(factorial(5) == 120, "");
}
```

**Answer:**
```
120 120
```
`constexpr` functions CAN be evaluated at runtime when the argument isn't constexpr. `b = factorial(n)` compiles fine but runs at runtime.

**Follow-up:** How to force compile-time only? → C++20: `consteval int factorial(int n)`.

---

### Q21: String Literal Lifetime Level: SDE 2

**Question:**
```cpp
#include <iostream>
const char* greet() {
    return "Hello";  // safe?
}
const char* greet2() {
    char arr[] = "Hello";
    return arr;  // safe?
}
int main() {
    std::cout << greet() << "\n";
    std::cout << greet2() << "\n";  // UB
}
```

**Answer:**
`greet()`: **Safe.** String literals have static storage duration.
`greet2()`: **Undefined behavior.** `arr` is a local array; returning a pointer to it is dangling.

**Follow-up:** Where are string literals stored? → Typically in `.rodata` section (read-only data).

---

### Q22: std::variant and std::visit Level: SDE 3

**Question:**
```cpp
#include <iostream>
#include <variant>
#include <string>
int main() {
    std::variant<int, std::string> v = "hello";
    std::cout << v.index() << "\n";
    v = 42;
    std::cout << v.index() << "\n";
    std::cout << std::get<int>(v) << "\n";
    try { std::cout << std::get<std::string>(v) << "\n"; }
    catch (const std::bad_variant_access&) { std::cout << "bad access\n"; }
}
```

**Answer:**
```
1
0
42
bad access
```
`"hello"` is `const char*` which converts to `std::string` (index 1). After assigning 42, index becomes 0.

**Follow-up:** How is `std::variant` implemented without dynamic allocation? → Aligned storage + index tag. Size = max(alternatives) + tag.

---

### Q23: Thread-Safe Singleton (Meyers' Singleton) Level: SDE 3

**Question:**
```cpp
#include <iostream>
#include <thread>
struct Singleton {
    static Singleton& instance() {
        static Singleton s;  // thread-safe since C++11?
        return s;
    }
    Singleton() { std::cout << "constructed\n"; }
};
int main() {
    std::thread t1([]{ Singleton::instance(); });
    std::thread t2([]{ Singleton::instance(); });
    t1.join(); t2.join();
}
```

**Answer:**
```
constructed
```
Prints exactly once. C++11 guarantees thread-safe initialization of function-local statics (§6.7). The second thread blocks until construction completes.

**Follow-up:** What's the cost? → Typically an atomic check + mutex on first call. Zero cost after initialization on most ABIs.

---

### Q24: Structured Bindings and References Level: SDE 2

**Question:**
```cpp
#include <iostream>
#include <map>
int main() {
    std::map<std::string, int> m = {{"a", 1}, {"b", 2}};
    for (auto& [key, value] : m) {
        value *= 10;  // modifies map?
    }
    for (const auto& [key, value] : m) {
        std::cout << key << ":" << value << " ";
    }
}
```

**Answer:**
```
a:10 b:20
```
`auto&` binds by reference. Modifying `value` directly modifies the map's values.

**Follow-up:** What if we use `auto` without `&`? → Creates copies; map isn't modified.

---

### Q25: coroutines basics (C++20) Level: SDE 3

**Question:**
```cpp
#include <iostream>
#include <coroutine>
struct Generator {
    struct promise_type {
        int current;
        Generator get_return_object() { return {std::coroutine_handle<promise_type>::from_promise(*this)}; }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        std::suspend_always yield_value(int v) { current = v; return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };
    std::coroutine_handle<promise_type> h;
    int next() { h.resume(); return h.promise().current; }
    ~Generator() { if (h) h.destroy(); }
};
Generator counter() {
    for (int i = 0; ; i++) co_yield i;
}
int main() {
    auto gen = counter();
    std::cout << gen.next() << " " << gen.next() << " " << gen.next() << "\n";
}
```

**Answer:**
```
0 1 2
```
Each `co_yield` suspends the coroutine, storing `i` in `promise_type::current`. `next()` resumes from the suspension point.

**Follow-up:** What's the heap allocation cost? → One coroutine frame per coroutine. Compiler may elide (HALO optimization).

---

### Q26: Placement New and Manual Destruction Level: SDE 3 (HFT)

**Question:**
```cpp
#include <iostream>
#include <new>
struct Widget {
    int id;
    Widget(int i) : id(i) { std::cout << "ctor " << id << "\n"; }
    ~Widget() { std::cout << "dtor " << id << "\n"; }
};
int main() {
    alignas(Widget) char buf[sizeof(Widget)];
    Widget* w = new (buf) Widget(1);
    w->~Widget();  // manual destruction
    Widget* w2 = new (buf) Widget(2);
    w2->~Widget();
}
```

**Answer:**
```
ctor 1
dtor 1
ctor 2
dtor 2
```
Placement new constructs in existing memory. Must manually call destructor. No `delete` memory isn't from the free store.

**Follow-up:** Why is this used in HFT? → Avoids allocator latency; objects live in pre-allocated pools or shared memory.
