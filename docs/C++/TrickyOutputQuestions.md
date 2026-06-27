# Tricky Input/Output Questions  C/C++

**Purpose: "What does this print?" questions that reveal deep language understanding in 30-60 seconds each. These expose knowledge of static variables, virtual dispatch, object lifetime, evaluation order, and undefined behavior.**

---

## Static Variables & Initialization

### Q1: Static local variable lifetime

```cpp
#include <iostream>

int& get_count() {
    static int count = 0;
    return ++count;
}

int main() {
    std::cout << get_count() << " ";
    std::cout << get_count() << " ";
    std::cout << get_count() << std::endl;
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** `1 2 3`

`static` local variables are initialized once (first call) and persist for the program lifetime. Each call increments the same variable.
</details>

---

### Q2: Static initialization order (TRICKY)

```cpp
#include <iostream>

struct A {
    A() { std::cout << "A"; }
    ~A() { std::cout << "~A"; }
};

struct B {
    B() { std::cout << "B"; }
    ~B() { std::cout << "~B"; }
};

B b;
A a;

int main() {
    std::cout << "M";
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** `BAM~A~B`

Global objects are constructed in declaration order (B before A). Destroyed in reverse order after `main()` returns.
</details>

---

### Q3: Static inside a loop

```cpp
#include <iostream>

void foo() {
    static int x = 0;
    int y = 0;
    std::cout << x++ << " " << y++ << " | ";
}

int main() {
    foo(); foo(); foo();
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** `0 0 | 1 0 | 2 0 | `

`x` is static  persists across calls. `y` is local  re-initialized to 0 every call.
</details>

---


### Q4: Static member variable (classic trap)

```cpp
#include <iostream>

class Counter {
public:
    Counter() { ++count_; }
    ~Counter() { --count_; }
    static int count_;
};

int Counter::count_ = 0;

int main() {
    Counter a;
    {
        Counter b;
        Counter c;
        std::cout << Counter::count_ << " ";
    }
    std::cout << Counter::count_ << std::endl;
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** `3 1`

After constructing `a`, `b`, `c`: count = 3. After `b` and `c` go out of scope (destroyed): count = 1. `a` still alive.
</details>

---

### Q5: Function-static with conditional initialization

```cpp
#include <iostream>

int init(int v) {
    std::cout << "init(" << v << ") ";
    return v;
}

void bar(bool first) {
    static int x = init(42);
    if (first) {
        static int y = init(99);
    }
    std::cout << x << " ";
}

int main() {
    bar(true);
    bar(true);
    bar(false);
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** `init(42) init(99) 42 42 42 `

`static int x` is initialized on first call to `bar`. `static int y` is initialized on first time the `if` branch is taken. Neither is re-initialized on subsequent calls.
</details>

---

## Virtual Functions & Polymorphism

### Q6: Virtual function called from constructor (CLASSIC TRAP)

```cpp
#include <iostream>

class Base {
public:
    Base() { init(); }
    virtual void init() { std::cout << "Base::init "; }
    virtual ~Base() = default;
};

class Derived : public Base {
public:
    Derived() { init(); }
    void init() override { std::cout << "Derived::init "; }
};

int main() {
    Derived d;
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** `Base::init Derived::init `

During `Base` constructor, the vtable still points to `Base`  virtual dispatch resolves to `Base::init`. By the time `Derived` constructor body runs, vtable is updated, so `init()` calls `Derived::init`.

**Key rule:** Virtual functions called from constructors/destructors do NOT dispatch to derived classes.
</details>

---

### Q7: Virtual destructor trap

```cpp
#include <iostream>

class Base {
public:
    Base() { std::cout << "B+"; }
    ~Base() { std::cout << "B-"; }  // NOT virtual!
};

class Derived : public Base {
public:
    Derived() { std::cout << "D+"; }
    ~Derived() { std::cout << "D-"; }
};

int main() {
    Base* p = new Derived();
    delete p;
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** `B+D+B-`

**This is undefined behavior!** Deleting a derived object through a base pointer when the destructor is not virtual. In practice, most compilers only call `~Base()`. `~Derived()` is never called  resource leak.

**Fix:** Make `~Base()` virtual.
</details>

---

### Q8: Pure virtual with implementation

```cpp
#include <iostream>

class Base {
public:
    virtual void foo() = 0;
    virtual ~Base() = default;
};

void Base::foo() {
    std::cout << "Base::foo ";
}

class Derived : public Base {
public:
    void foo() override {
        Base::foo();
        std::cout << "Derived::foo ";
    }
};

int main() {
    Derived d;
    d.foo();
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** `Base::foo Derived::foo `

A pure virtual function CAN have an implementation. It can be called explicitly via qualified name (`Base::foo()`). The class is still abstract  you cannot instantiate `Base` directly.
</details>

---

### Q9: Covariant return types + slicing

```cpp
#include <iostream>

class Animal {
public:
    virtual Animal* clone() const {
        std::cout << "Animal::clone ";
        return new Animal(*this);
    }
    virtual void speak() const { std::cout << "..."; }
    virtual ~Animal() = default;
};

class Dog : public Animal {
public:
    Dog* clone() const override {  // Covariant return type
        std::cout << "Dog::clone ";
        return new Dog(*this);
    }
    void speak() const override { std::cout << "Woof"; }
};

int main() {
    Dog d;
    Animal* a = &d;
    Animal* cloned = a->clone();
    cloned->speak();
    delete cloned;
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** `Dog::clone Woof`

Covariant return types allow `Dog::clone()` to return `Dog*` instead of `Animal*`. Virtual dispatch still works  `a->clone()` calls `Dog::clone`. The cloned object is a `Dog`, so `speak()` prints "Woof".
</details>

---

## Object Lifetime & Construction Order

### Q10: Member initialization order (TRICKY)

```cpp
#include <iostream>

class Widget {
public:
    Widget(int a, int b) : y_(b), x_(a + y_) {  // WARNING!
        std::cout << x_ << " " << y_ << std::endl;
    }
private:
    int x_;
    int y_;
};

int main() {
    Widget w(1, 2);
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** Undefined behavior (likely garbage for `x_`)

Members are initialized in **declaration order** (`x_` first, then `y_`), NOT in the order they appear in the initializer list. `x_` is initialized as `a + y_`, but `y_` hasn't been initialized yet  UB.

**Fix:** Reorder members: put `y_` before `x_` in the class declaration, or don't depend on `y_` when initializing `x_`.
</details>

---

### Q11: Copy elision / RVO

```cpp
#include <iostream>

class Obj {
public:
    Obj() { std::cout << "Ctor "; }
    Obj(const Obj&) { std::cout << "Copy "; }
    Obj(Obj&&) { std::cout << "Move "; }
    ~Obj() { std::cout << "Dtor "; }
};

Obj make() {
    return Obj();  // NRVO / copy elision
}

int main() {
    Obj o = make();
    std::cout << "End ";
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output (C++17 guaranteed):** `Ctor End Dtor `

Since C++17, copy elision is mandatory for this case (prvalue materialization). No copy or move constructor is called. The object is constructed directly in `o`.

Pre-C++17: compiler may elide (NRVO), but output *could* be `Ctor Move Dtor End Dtor`.
</details>

---

### Q12: Destruction order with inheritance

```cpp
#include <iostream>

struct A {
    A() { std::cout << "A+"; }
    virtual ~A() { std::cout << "A-"; }
};
struct B : A {
    B() { std::cout << "B+"; }
    ~B() override { std::cout << "B-"; }
};
struct C : B {
    C() { std::cout << "C+"; }
    ~C() override { std::cout << "C-"; }
};

int main() {
    A* p = new C();
    std::cout << " | ";
    delete p;
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** `A+B+C+ | C-B-A-`

Construction: base → derived. Destruction: derived → base. Virtual destructor ensures full cleanup through base pointer.
</details>

---

## Operator Overloading & Implicit Conversions

### Q13: Implicit conversion trap

```cpp
#include <iostream>

class Integer {
public:
    Integer(int v) : val_(v) { std::cout << "Ctor(" << v << ") "; }
    operator int() const { std::cout << "Conv "; return val_; }
    Integer operator+(const Integer& other) const {
        std::cout << "Add "; 
        return Integer(val_ + other.val_);
    }
private:
    int val_;
};

int main() {
    Integer a(3);
    Integer b = a + 5;
    std::cout << "= " << (int)b << std::endl;
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** `Ctor(3) Ctor(5) Add Ctor(8) Conv = 8`

`5` is implicitly converted to `Integer(5)` via the non-explicit constructor. Then `operator+` creates `Integer(8)`. Finally `(int)b` calls `operator int()`.

**Lesson:** Mark single-arg constructors `explicit` to prevent unintended conversions.
</details>

---

### Q14: Pre/post increment in expressions (C)

```cpp
#include <stdio.h>

int main() {
    int x = 5;
    int y = x++ + ++x;
    printf("%d %d\n", x, y);
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** UNDEFINED BEHAVIOR

Modifying `x` twice without a sequence point between modifications is UB in C and C++. Different compilers produce different results. GCC may give `7 12`, Clang may give `7 12`, MSVC may differ.

**Key rule:** Never modify a variable more than once in a single expression without sequencing.
</details>

---

## Templates & SFINAE

### Q15: Template specialization selection

```cpp
#include <iostream>

template <typename T>
void foo(T) { std::cout << "generic "; }

template <>
void foo<int>(int) { std::cout << "int-spec "; }

void foo(int) { std::cout << "overload "; }

int main() {
    foo(42);
    foo<int>(42);
    foo<>(42);
    foo(42.0);
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** `overload int-spec int-spec generic `

- `foo(42)`: Non-template overload preferred over template (exact match, non-template wins).
- `foo<int>(42)`: Explicitly requests template with `T=int`, finds specialization.
- `foo<>(42)`: Explicitly requests template (deduces `T=int`), finds specialization.
- `foo(42.0)`: `double` matches generic template `T=double`. Non-template `foo(int)` would require narrowing.
</details>

---

### Q16: sizeof and virtual (TRICKY)

```cpp
#include <iostream>

class Empty {};

class Base {
    virtual void foo() {}
};

class Derived : public Base {
    int x;
};

class Multi : public Base {
    virtual void bar() {}
    int y;
};

int main() {
    std::cout << sizeof(Empty) << " "
              << sizeof(Base) << " "
              << sizeof(Derived) << " "
              << sizeof(Multi) << std::endl;
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output (64-bit system, typical):** `1 8 16 16`

- `Empty`: 1 byte (C++ requires unique address)
- `Base`: 8 bytes (vptr only, pointer-sized on 64-bit)
- `Derived`: 16 bytes (vptr 8 + int 4 + padding 4)
- `Multi`: 16 bytes (still one vptr  both virtuals in same vtable  + int 4 + padding 4)

**Key insight:** A class has ONE vptr regardless of how many virtual functions it has. Multiple vtable entries, but one pointer.
</details>

---

## Pointers, References & Memory

### Q17: Dangling reference from temporary

```cpp
#include <iostream>
#include <string>

const std::string& get_name() {
    return "hello";  // WARNING: returning reference to temporary!
}

int main() {
    std::cout << get_name() << std::endl;
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** UNDEFINED BEHAVIOR (may print "hello", may crash, may print garbage)

A temporary `std::string` is constructed from `"hello"`, but the function returns a reference to it. The temporary is destroyed at the end of the return statement. The caller has a dangling reference.

**Fix:** Return by value: `std::string get_name() { return "hello"; }`
</details>

---

### Q18: Array decay and sizeof

```cpp
#include <iostream>

void foo(int arr[]) {
    std::cout << sizeof(arr) << " ";
}

int main() {
    int arr[10] = {};
    std::cout << sizeof(arr) << " ";
    foo(arr);
    int* p = arr;
    std::cout << sizeof(p) << std::endl;
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output (64-bit):** `40 8 8`

- `sizeof(arr)` in main: actual array size (10 × 4 = 40 bytes)
- `sizeof(arr)` in foo: array decays to pointer when passed to function  sizeof(pointer) = 8
- `sizeof(p)`: pointer size = 8

**Lesson:** Arrays are not pointers, but they decay to pointers when passed to functions.
</details>

---

### Q19: Pointer arithmetic across types

```cpp
#include <iostream>

struct S {
    int a;
    char b;
    double c;
};

int main() {
    S arr[3];
    std::cout << sizeof(S) << " ";
    std::cout << (char*)&arr[1] - (char*)&arr[0] << " ";
    
    int iarr[5] = {10, 20, 30, 40, 50};
    int* p = iarr + 2;
    std::cout << p[-1] << " " << *(p + 2) << std::endl;
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output (typical 64-bit, 8-byte aligned):** `16 16 20 50`

- `sizeof(S)`: int(4) + char(1) + padding(3) + double(8) = 16 (due to alignment)
- Byte distance between array elements = sizeof(S) = 16
- `p[-1]`: `*(p - 1)` = `iarr[1]` = 20
- `*(p + 2)`: `iarr[4]` = 50
</details>

---

## Tricky C-Specific Questions

### Q20: `const` pointer vs pointer to `const`

```cpp
#include <stdio.h>

int main() {
    int x = 10, y = 20;
    
    const int* p1 = &x;     // Pointer to const int
    int* const p2 = &x;     // Const pointer to int
    
    // p1 = &y;    // OK?    → YES (pointer can change)
    // *p1 = 30;   // OK?    → NO  (can't modify through pointer to const)
    // p2 = &y;    // OK?    → NO  (const pointer can't change)
    // *p2 = 30;   // OK?    → YES (value can be modified)
    
    *p2 = 30;
    p1 = &y;
    printf("%d %d\n", *p1, *p2);
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** `20 30`

- `p1` now points to `y` (value 20)
- `p2` still points to `x`, which was modified to 30 via `*p2 = 30`

**Rule:** Read declarations right-to-left: `const int*` = "pointer to int-that-is-const". `int* const` = "const-pointer to int".
</details>

---

### Q21: Comma operator (C/C++ trap)

```cpp
#include <iostream>

int main() {
    int x = (1, 2, 3, 4, 5);
    int y = 1, 2;  // What happens here?
    
    std::cout << x << std::endl;
    return 0;
}
```

<details>
<summary>Answer</summary>

**This doesn't compile!** `int y = 1, 2;` is a syntax error  the comma here is a declaration separator, not the comma operator. It tries to declare `2` as a variable name.

If the second line is removed, output is: `5`

The comma operator evaluates left-to-right and returns the rightmost value.
</details>

---

### Q22: `volatile` and optimization

```cpp
#include <iostream>

int main() {
    volatile int x = 10;
    int a = x;
    int b = x;
    std::cout << (a == b) << std::endl;
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** `1` (in practice, but NOT guaranteed!)

`volatile` forces the compiler to read `x` from memory both times (no caching in register). In a single-threaded program with no hardware mapping, both reads return 10. But semantically, the compiler cannot assume `a == b` because `x` is volatile  the value *could* change between reads (hardware register, ISR, etc.).

**Key insight:** `volatile` prevents optimization but does NOT provide atomicity or ordering.
</details>

---

## Multiple Inheritance & Diamond Problem

### Q23: Diamond inheritance

```cpp
#include <iostream>

class A {
public:
    A() { std::cout << "A(" << val_ << ") "; }
    int val_ = 1;
};

class B : public A {
public:
    B() { val_ = 2; std::cout << "B "; }
};

class C : public A {
public:
    C() { val_ = 3; std::cout << "C "; }
};

class D : public B, public C {
public:
    D() { std::cout << "D "; }
};

int main() {
    D d;
    // d.val_;  // AMBIGUOUS! Which A::val_?
    std::cout << d.B::val_ << " " << d.C::val_ << std::endl;
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** `A(1) B A(1) C D 2 3`

Without virtual inheritance, `D` has TWO copies of `A`. Construction order: B's A, then B body, C's A, then C body, then D body. `d.val_` is ambiguous  must qualify with `B::` or `C::`.

**Fix with virtual inheritance:**
```cpp
class B : virtual public A { ... };
class C : virtual public A { ... };
```
Now only ONE `A` subobject exists in `D`.
</details>

---

### Q24: Virtual inheritance construction order

```cpp
#include <iostream>

class A { public: A(int x) { std::cout << "A(" << x << ") "; } };
class B : virtual public A { public: B() : A(1) { std::cout << "B "; } };
class C : virtual public A { public: C() : A(2) { std::cout << "C "; } };
class D : public B, public C { public: D() : A(3) { std::cout << "D "; } };

int main() {
    D d;
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** `A(3) B C D `

With virtual inheritance, the MOST DERIVED class (`D`) is responsible for constructing the virtual base (`A`). `B::B()` and `C::C()`'s initializations of `A` are IGNORED when constructing `D`. Only `D`'s `A(3)` is called.
</details>

---

## Lambda & Capture Semantics

### Q25: Capture by value vs reference (TRICKY)

```cpp
#include <iostream>
#include <functional>

std::function<int()> make_counter() {
    int count = 0;
    return [count]() mutable { return ++count; };
}

int main() {
    auto c1 = make_counter();
    auto c2 = c1;  // Copy the lambda

    std::cout << c1() << " ";
    std::cout << c1() << " ";
    std::cout << c2() << " ";
    std::cout << c1() << std::endl;
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** `1 2 1 3`

`c1` captures `count` by value with `mutable`  it has its own copy. `c2 = c1` copies the lambda including its captured state (at that point `count = 0`). After `c1()` twice, c1's internal count is 2. `c2()` increments its own copy (was 0, now 1). `c1()` again gives 3.
</details>

---

### Q26: Capture by reference  dangling (BUG)

```cpp
#include <iostream>
#include <functional>

std::function<int()> make_adder(int x) {
    return [&x]() { return x + 1; };  // BUG!
}

int main() {
    auto add = make_adder(10);
    std::cout << add() << std::endl;
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** UNDEFINED BEHAVIOR (may print 11, may print garbage)

`x` is a local parameter. Capturing by reference (`&x`) creates a dangling reference after `make_adder` returns. The lambda holds a reference to a destroyed stack variable.

**Fix:** Capture by value: `[x]() { return x + 1; }`
</details>

---

## Evaluation Order & Sequence Points

### Q27: Function argument evaluation order

```cpp
#include <iostream>

int f() { std::cout << "f"; return 1; }
int g() { std::cout << "g"; return 2; }
int h() { std::cout << "h"; return 3; }

void print(int a, int b, int c) {
    std::cout << " = " << a << b << c << std::endl;
}

int main() {
    print(f(), g(), h());
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:** IMPLEMENTATION DEFINED order of `f`, `g`, `h`, then ` = 123`

The order in which function arguments are evaluated is **unspecified** (not undefined  all three will execute, but in any order). GCC typically evaluates right-to-left (`hgf`), Clang/MSVC may differ.

Result is always `= 123` regardless of evaluation order.
</details>

---

### Q28: `std::move` doesn't move (COMMON MISCONCEPTION)

```cpp
#include <iostream>
#include <string>

int main() {
    std::string a = "hello";
    std::string b = std::move(a);
    
    std::cout << "a='" << a << "' b='" << b << "'" << std::endl;
    std::cout << "a.size()=" << a.size() << std::endl;
    return 0;
}
```

<details>
<summary>Answer</summary>

**Output:**
```
a='' b='hello'
a.size()=0
```

`std::move` casts `a` to an rvalue reference, enabling the move constructor of `b`. After the move, `a` is in a "valid but unspecified state"  for `std::string`, it's typically empty.

**Key rule:** After `std::move`, the source object is valid (can be destroyed, assigned to) but its value is unspecified. Don't use it without reassigning.
</details>

---

## Scoring Guide

| Score | Performance |
|-------|------------|
| 22-28 correct | Expert  deep language lawyer knowledge |
| 16-21 correct | Proficient  senior level, knows the gotchas |
| 10-15 correct | Competent  mid-level, some gaps |
| 5-9 correct | Developing  needs more experience with edge cases |
| 0-4 correct | Below bar for senior role |

**Candidate Score:** __/28

**Notes on which questions tripped them up:**


