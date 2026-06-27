# C Coding & Output Questions

> Low-level C questions for embedded, silicon, kernel, and HFT roles.
> Focus: pointers, memory layout, undefined behavior, bit manipulation, and hardware interaction.

---

### Q1: Pointer Arithmetic Output Level: SDE 1

**Question:**
```c
#include <stdio.h>
int main() {
    int arr[] = {10, 20, 30, 40, 50};
    int *p = arr;
    printf("%d\n", *(p + 2));
    printf("%d\n", *p + 2);
    printf("%d\n", *(arr + 4));
    printf("%ld\n", (long)(&arr[4] - &arr[0]));
    return 0;
}
```

**Answer:**
```
30
12
50
4
```
- `*(p+2)`: pointer moves 2 ints forward → arr[2] = 30.
- `*p + 2`: dereference first (*p=10), then add 2 → 12.
- Pointer subtraction yields **element count**, not byte count.

**Follow-up:** What's `sizeof(arr)` vs `sizeof(p)`? → `5*sizeof(int)=20` vs `sizeof(int*)=8` (64-bit).

---

### Q2: Array Decay Behavior Level: SDE 1

**Question:**
```c
#include <stdio.h>
void print_size(int arr[]) {
    printf("param: %lu\n", sizeof(arr));
}
int main() {
    int arr[10];
    printf("local: %lu\n", sizeof(arr));
    print_size(arr);
    return 0;
}
```

**Answer:**
```
local: 40
param: 8
```
Arrays decay to pointers when passed to functions. `sizeof(arr)` in function = pointer size (8 on 64-bit). Only in declaring scope does `sizeof` give the full array size.

**Follow-up:** How to pass array with size info? → Pass pointer + length, or use `int (*arr)[10]` (pointer-to-array).

---

### Q3: Struct Padding and Alignment Level: SDE 2

**Question:**
```c
#include <stdio.h>
#include <stddef.h>

struct A { char a; int b; char c; };
struct B { char a; char c; int b; };
struct C { char a; double b; char c; };

int main() {
    printf("A: %lu (offset b=%lu, c=%lu)\n", sizeof(struct A), 
           offsetof(struct A, b), offsetof(struct A, c));
    printf("B: %lu\n", sizeof(struct B));
    printf("C: %lu\n", sizeof(struct C));
    return 0;
}
```

**Answer (x86-64):**
```
A: 12 (offset b=4, c=8)
B: 8
C: 24
```
- `A`: char(1) + pad(3) + int(4) + char(1) + pad(3) = 12
- `B`: char(1) + char(1) + pad(2) + int(4) = 8
- `C`: char(1) + pad(7) + double(8) + char(1) + pad(7) = 24

Rule: Each member aligns to its own size. Struct size rounds up to largest member alignment.

**Follow-up:** How to pack? → `__attribute__((packed))` but unaligned access may crash on ARM or be slow on x86.

---

### Q4: volatile Keyword Correctness Level: SDE 2 (Embedded)

**Question:**
```c
#include <stdio.h>

// Hardware register at fixed address
#define STATUS_REG (*(volatile unsigned int *)0x40001000)
#define DATA_REG   (*(volatile unsigned int *)0x40001004)

// Bug: What's wrong with this?
int read_sensor_bad() {
    unsigned int status = STATUS_REG;
    if (status & 0x01) {
        unsigned int data = DATA_REG;
        return data;
    }
    return -1;
}

// Why volatile is needed
void wait_for_ready() {
    while (!(STATUS_REG & 0x01)) {
        // Without volatile, compiler optimizes to:
        // if (!(STATUS_REG & 0x01)) while(1);
    }
}
```

**Answer:**
Without `volatile`, the compiler may:
1. Cache register reads in a CPU register.
2. Optimize away the loop (hoist the check).
3. Reorder reads between STATUS and DATA.

`volatile` prevents compiler optimization but NOT CPU reordering. For multi-core, need memory barriers.

**Follow-up:** Is volatile sufficient for multi-threaded code? → No. Use `_Atomic` (C11) or compiler barriers + hardware barriers.

---

### Q5: const Pointer Declarations Level: SDE 1

**Question:**
```c
#include <stdio.h>
int main() {
    int x = 10, y = 20;
    
    const int *p1 = &x;      // A
    int const *p2 = &x;      // B
    int *const p3 = &x;      // C
    const int *const p4 = &x; // D
    
    // Which are legal?
    // *p1 = 5;    // ?
    // p1 = &y;    // ?
    // *p3 = 5;    // ?
    // p3 = &y;    // ?
    // *p4 = 5;    // ?
    // p4 = &y;    // ?
    
    p1 = &y;   // OK
    *p3 = 5;   // OK
    printf("%d %d\n", *p1, *p3);
    return 0;
}
```

**Answer:**
```
20 5
```
- `const int *p` (A, B): pointer to const int can't modify `*p`, can reassign `p`.
- `int *const p` (C): const pointer to int can modify `*p`, can't reassign `p`.
- `const int *const p` (D): both const nothing modifiable.

Rule: Read right-to-left. `const` left of `*` = data is const. `const` right of `*` = pointer is const.

**Follow-up:** Can you cast away const? → `*(int*)p1 = 5` compiles but is UB if the original object was declared const.

---

### Q6: Bit Manipulation Set/Clear/Toggle Level: SDE 1

**Question:**
```c
#include <stdio.h>
int main() {
    unsigned int reg = 0xA5; // 1010 0101
    
    // Set bit 4
    reg |= (1U << 4);
    printf("set:    0x%02X\n", reg);
    
    // Clear bit 5
    reg &= ~(1U << 5);
    printf("clear:  0x%02X\n", reg);
    
    // Toggle bit 0
    reg ^= (1U << 0);
    printf("toggle: 0x%02X\n", reg);
    
    // Check bit 7
    printf("bit7:   %d\n", (reg >> 7) & 1);
    return 0;
}
```

**Answer:**
```
set:    0xB5
clear:  0x95
toggle: 0x94
bit7:   1
```
- 0xA5 = 1010_0101 → set bit4 → 1011_0101 = 0xB5
- 0xB5 = 1011_0101 → clear bit5 → 1001_0101 = 0x95
- 0x95 = 1001_0101 → toggle bit0 → 1001_0100 = 0x94
- Bit 7 of 0x94 = 1

**Follow-up:** Write a macro to set N consecutive bits starting at position P.
```c
#define SET_BITS(reg, pos, n) ((reg) | (((1U << (n)) - 1) << (pos)))
```

---

### Q7: Endianness Detection Level: SDE 2

**Question:**
```c
#include <stdio.h>
#include <stdint.h>

int is_little_endian() {
    uint32_t x = 1;
    return *(uint8_t *)&x == 1;
}

// Alternative using union (technically UB in C++ but well-defined in C)
int is_little_endian_union() {
    union { uint32_t i; uint8_t c[4]; } u = {.i = 1};
    return u.c[0] == 1;
}

int main() {
    printf("Little endian: %d\n", is_little_endian());
    
    uint32_t val = 0x12345678;
    uint8_t *bytes = (uint8_t *)&val;
    printf("Byte order: %02X %02X %02X %02X\n",
           bytes[0], bytes[1], bytes[2], bytes[3]);
    return 0;
}
```

**Answer (x86, little-endian):**
```
Little endian: 1
Byte order: 78 56 34 12
```
Little-endian: LSB at lowest address. The cast to `uint8_t*` examines individual bytes.

**Follow-up:** How to convert between host and network byte order? → `htonl()`, `ntohl()` etc.

---

### Q8: Function Pointer Syntax Level: SDE 2

**Question:**
```c
#include <stdio.h>

int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }

// Function pointer typedef
typedef int (*Operation)(int, int);

// Function that returns a function pointer
Operation get_operation(char op) {
    switch(op) {
        case '+': return add;
        case '-': return sub;
        default:  return NULL;
    }
}

int main() {
    Operation op = get_operation('+');
    printf("%d\n", op(10, 3));
    
    // Array of function pointers
    Operation ops[] = {add, sub};
    printf("%d\n", ops[1](10, 3));
    return 0;
}
```

**Answer:**
```
13
7
```
Function pointers enable polymorphism in C. Common in: callback APIs, vtable implementations, plugin systems.

**Follow-up:** Declare without typedef: `int (*get_op(char))(int, int)` return type is a function pointer.

---

### Q9: Memory-Mapped I/O Level: SDE 2 (Embedded)

**Question:**
```c
#include <stdint.h>

// Typical embedded register definitions
typedef struct {
    volatile uint32_t CR;     // Control register (offset 0x00)
    volatile uint32_t SR;     // Status register  (offset 0x04)
    volatile uint32_t DR;     // Data register    (offset 0x08)
    volatile uint32_t RESERVED[5]; // gap
    volatile uint32_t ISR;    // Interrupt status (offset 0x20)
} UART_TypeDef;

#define UART1_BASE  0x40011000
#define UART1       ((UART_TypeDef *)UART1_BASE)

void uart_send(uint8_t byte) {
    while (!(UART1->SR & (1 << 7))) {}  // wait for TXE
    UART1->DR = byte;
}

// What's the address of UART1->ISR?
// Answer: 0x40011000 + 0x20 = 0x40011020
```

**Answer:**
`UART1->ISR` is at address `0x40011020`. The struct layout maps directly to hardware register addresses. `volatile` ensures every read/write hits the hardware.

**Follow-up:** Why must the struct be `packed` or naturally aligned? → Padding would shift register offsets. ARM Cortex-M registers are typically word-aligned, so natural alignment works.

---

### Q10: Undefined Behavior Signed Overflow, Sequence Points Level: SDE 2

**Question:**
```c
#include <stdio.h>
#include <limits.h>

int main() {
    // UB 1: signed overflow
    int x = INT_MAX;
    // x + 1;  // UB! Compiler may assume this never happens
    
    // UB 2: modification and read without sequence point
    int i = 0;
    // int j = i++ + i++;  // UB!
    
    // UB 3: null dereference
    int *p = NULL;
    // *p = 5;  // UB!
    
    // UB 4: use after free
    // int *q = malloc(4); free(q); *q = 5;  // UB!
    
    // Well-defined alternatives:
    unsigned int ux = UINT_MAX;
    ux = ux + 1;  // defined: wraps to 0
    printf("%u\n", ux);
    
    return 0;
}
```

**Answer:**
```
0
```
Unsigned overflow wraps (defined). Signed overflow is UB compiler exploits this for optimization (e.g., assuming `x + 1 > x` is always true).

**Follow-up:** How does UB affect optimization?
```c
// Compiler can optimize this to just "return 1"
int always_true(int x) { return x + 1 > x; }
```

---

### Q11: Preprocessor Macro Traps Level: SDE 2

**Question:**
```c
#include <stdio.h>

#define SQUARE(x) x * x
#define SQUARE_SAFE(x) ((x) * (x))
#define MAX(a, b) ((a) > (b) ? (a) : (b))

int main() {
    printf("%d\n", SQUARE(3 + 1));      // Bug!
    printf("%d\n", SQUARE_SAFE(3 + 1));
    
    int a = 1, b = 2;
    int c = MAX(a++, b++);              // Bug!
    printf("c=%d a=%d b=%d\n", c, a, b);
    return 0;
}
```

**Answer:**
```
7
16
c=3 a=2 b=4
```
- `SQUARE(3+1)` → `3+1 * 3+1` = `3+3+1` = 7 (operator precedence).
- `MAX(a++, b++)` → `((a++) > (b++) ? (a++) : (b++))` `b` incremented twice.

**Fix:** Use `static inline` functions instead of macros for type safety and single evaluation.

**Follow-up:** When are macros still necessary? → Include guards, stringification, token pasting, conditional compilation.

---

### Q12: Stack Overflow Detection Level: SDE 2 (Embedded)

**Question:**
```c
#include <stdio.h>
#include <stdint.h>

#define STACK_SIZE 4096
#define STACK_CANARY 0xDEADBEEF

static uint32_t stack[STACK_SIZE / sizeof(uint32_t)];

void init_stack_canary() {
    stack[0] = STACK_CANARY;  // bottom of stack (grows down)
}

int check_stack_overflow() {
    return stack[0] != STACK_CANARY;
}

// Recursive function that will blow the stack
void recursive(int n) {
    char buffer[256];  // consumes stack
    buffer[0] = n;
    if (n > 0) recursive(n - 1);
}

int main() {
    init_stack_canary();
    // recursive(100);  // would overflow
    printf("overflow: %d\n", check_stack_overflow());
    return 0;
}
```

**Answer:**
```
overflow: 0
```
Stack canary is intact (no overflow occurred). In RTOS environments, canary words at stack boundaries detect overflow between context switches.

**Follow-up:** What's `-fstack-protector` in GCC? → Adds canaries to function return addresses to detect buffer overflows.

---

### Q13: malloc/free Patterns Level: SDE 2

**Question:**
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// Bug hunt: find all issues
char* create_string(const char *src) {
    char *s = malloc(strlen(src));  // Bug 1
    strcpy(s, src);
    return s;
}

void double_free_example() {
    int *p = malloc(sizeof(int));
    *p = 42;
    free(p);
    // printf("%d\n", *p);  // Bug 2: use after free
    // free(p);              // Bug 3: double free
}

int main() {
    char *s = create_string("hello");
    printf("%s\n", s);
    free(s);
    return 0;
}
```

**Answer:**
Bug 1: `malloc(strlen(src))` missing +1 for null terminator. Should be `malloc(strlen(src) + 1)`.
Bug 2: Use after free undefined behavior, may print 42 or crash.
Bug 3: Double free heap corruption.

Output (with bug): `hello` (works by luck writes one byte past allocation).

**Fix pattern:**
```c
free(p);
p = NULL;  // prevent use-after-free and double-free
```

**Follow-up:** What tool detects these? → Valgrind, AddressSanitizer (`-fsanitize=address`).

---

### Q14: Signal Handler Restrictions Level: SDE 3

**Question:**
```c
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

volatile sig_atomic_t got_signal = 0;

void handler(int sig) {
    got_signal = 1;
    // printf("signal!\n");  // Bug: not async-signal-safe!
    write(STDOUT_FILENO, "sig\n", 4);  // OK: write is async-signal-safe
}

int main() {
    signal(SIGINT, handler);
    printf("Waiting for Ctrl+C...\n");
    while (!got_signal) {
        pause();  // sleep until signal
    }
    printf("Got signal, exiting\n");
    return 0;
}
```

**Answer:**
Only async-signal-safe functions can be called in signal handlers. `printf` uses internal locks → deadlock if signal interrupts another `printf`. Only ~30 functions are safe (write, _exit, signal, etc.).

Variables shared with signal handlers must be `volatile sig_atomic_t`.

**Follow-up:** What's the self-pipe trick? → Write to a pipe in the handler; main loop uses `select`/`poll` on the read end.

---

### Q15: setjmp/longjmp Level: SDE 3

**Question:**
```c
#include <stdio.h>
#include <setjmp.h>

jmp_buf env;

void dangerous_function(int val) {
    printf("entering with %d\n", val);
    if (val == 0) {
        longjmp(env, 1);  // "throw"
    }
    printf("normal exit\n");
}

int main() {
    int ret = setjmp(env);  // "try"
    if (ret == 0) {
        printf("first time\n");
        dangerous_function(0);
        printf("unreachable\n");
    } else {
        printf("caught: %d\n", ret);  // "catch"
    }
    printf("continuing\n");
    return 0;
}
```

**Answer:**
```
first time
entering with 0
caught: 1
continuing
```
`setjmp` saves CPU state (registers, SP, PC). `longjmp` restores it non-local jump. Used for error handling in C (before exceptions existed).

**Dangers:** No destructors called, no stack unwinding, local variables may be indeterminate after longjmp.

**Follow-up:** Use in kernels? → Linux kernel's `BUG_ON` and recovery mechanisms. Also in interpreters for exception-like behavior.

---

### Q16: Flexible Array Member Level: SDE 2

**Question:**
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

struct Packet {
    uint16_t length;
    uint8_t  type;
    uint8_t  data[];  // flexible array member (C99)
};

struct Packet* create_packet(uint8_t type, const void *payload, size_t len) {
    struct Packet *p = malloc(sizeof(struct Packet) + len);
    p->length = len;
    p->type = type;
    memcpy(p->data, payload, len);
    return p;
}

int main() {
    printf("sizeof(Packet): %lu\n", sizeof(struct Packet));
    const char *msg = "hello";
    struct Packet *pkt = create_packet(1, msg, 5);
    printf("type=%d data=%.*s\n", pkt->type, pkt->length, pkt->data);
    free(pkt);
    return 0;
}
```

**Answer:**
```
sizeof(Packet): 4
type=1 data=hello
```
`sizeof` doesn't include the flexible array. Memory is allocated contiguously: header + payload in one allocation. Common in network protocols, IPC, and kernel data structures.

**Follow-up:** Why not just use a pointer? → One allocation (cache-friendly), no separate free, data is inline.

---

### Q17: Restrict Keyword Level: SDE 3 (HFT)

**Question:**
```c
#include <stdio.h>
#include <string.h>

// Without restrict: compiler must assume a and b might overlap
void copy_no_restrict(int *a, int *b, int n) {
    for (int i = 0; i < n; i++) a[i] = b[i];
}

// With restrict: compiler can vectorize aggressively
void copy_restrict(int *restrict a, int *restrict b, int n) {
    for (int i = 0; i < n; i++) a[i] = b[i];
}

int main() {
    int src[] = {1, 2, 3, 4, 5};
    int dst[5];
    copy_restrict(dst, src, 5);
    printf("%d %d %d\n", dst[0], dst[2], dst[4]);
    return 0;
}
```

**Answer:**
```
1 3 5
```
`restrict` tells the compiler that pointers don't alias enables SIMD vectorization and instruction reordering. Violating the contract (overlapping pointers) is UB.

**Follow-up:** Why does `memcpy` require non-overlapping but `memmove` doesn't? → `memcpy` uses restrict internally for speed; `memmove` copies via temp buffer for safety.

---

### Q18: Bitfield Portability Level: SDE 2 (Embedded)

**Question:**
```c
#include <stdio.h>

struct Flags {
    unsigned int ready   : 1;
    unsigned int mode    : 3;
    unsigned int error   : 1;
    unsigned int         : 3;  // padding
    unsigned int counter : 8;
};

int main() {
    struct Flags f = {0};
    f.ready = 1;
    f.mode = 5;
    f.counter = 200;
    
    printf("sizeof: %lu\n", sizeof(struct Flags));
    printf("ready=%u mode=%u counter=%u\n", f.ready, f.mode, f.counter);
    
    f.mode = 8;  // overflow: 3 bits max = 7
    printf("mode overflow: %u\n", f.mode);
    return 0;
}
```

**Answer:**
```
sizeof: 4
ready=1 mode=5 counter=200
mode overflow: 0
```
Bitfields: `mode` is 3 bits (0-7). Assigning 8 (binary 1000) stores only low 3 bits = 0.

**Portability issues:** Bit ordering (MSB-first vs LSB-first), padding, alignment all implementation-defined.

**Follow-up:** Why avoid bitfields for hardware registers? → Non-portable layout. Use explicit masks: `reg |= (mode & 0x7) << 1;`

---

### Q19: typedef vs #define for Types Level: SDE 1

**Question:**
```c
#include <stdio.h>

#define PTR_INT_DEF int*
typedef int* ptr_int_t;

int main() {
    PTR_INT_DEF a, b;     // What types are a and b?
    ptr_int_t c, d;        // What types are c and d?
    
    int x = 10, y = 20;
    a = &x;
    // b = &y;  // compile error if b isn't a pointer!
    c = &x;
    d = &y;
    
    printf("a=%d c=%d d=%d\n", *a, *c, *d);
    return 0;
}
```

**Answer:**
- `PTR_INT_DEF a, b` → expands to `int* a, b` → `a` is `int*`, `b` is `int`.
- `ptr_int_t c, d` → both are `int*`.

This is why `typedef` is preferred over `#define` for type definitions.

**Follow-up:** What about `const PTR_INT_DEF p`? → Expands to `const int *p` (pointer to const int, not const pointer to int).

---

### Q20: Static Functions and Linkage Level: SDE 2

**Question:**
```c
// file1.c
#include <stdio.h>
static int counter = 0;  // internal linkage
static void increment() { counter++; }

void file1_work() {
    increment();
    increment();
    printf("file1 counter: %d\n", counter);
}

// file2.c
#include <stdio.h>
static int counter = 0;  // different variable!
static void increment() { counter++; }

void file2_work() {
    increment();
    printf("file2 counter: %d\n", counter);
}

// main.c
void file1_work(void);
void file2_work(void);
int main() {
    file1_work();
    file2_work();
    return 0;
}
```

**Answer:**
```
file1 counter: 2
file2 counter: 1
```
`static` at file scope = internal linkage. Each translation unit has its own `counter` and `increment`. No name collision at link time.

**Follow-up:** What's the difference between `static` at file scope vs function scope? → File scope = internal linkage. Function scope = persistent across calls.

---

### Q21: Compound Literal Level: SDE 2

**Question:**
```c
#include <stdio.h>

struct Point { int x; int y; };

void print_point(struct Point p) {
    printf("(%d, %d)\n", p.x, p.y);
}

int main() {
    print_point((struct Point){3, 4});  // compound literal
    
    int *p = (int[]){10, 20, 30};      // anonymous array
    printf("%d\n", p[1]);
    
    // Lifetime: block scope (unlike string literals)
    struct Point *pp = &(struct Point){5, 6};
    printf("(%d, %d)\n", pp->x, pp->y);
    return 0;
}
```

**Answer:**
```
(3, 4)
20
(5, 6)
```
Compound literals (C99) create unnamed objects. In block scope, they have automatic storage duration (valid until end of block).

**Follow-up:** Are compound literals lvalues? → Yes! You can take their address and modify them (unlike C++ temporary objects).

---

### Q22: _Generic Selection (C11) Level: SDE 3

**Question:**
```c
#include <stdio.h>

#define type_name(x) _Generic((x), \
    int: "int",                     \
    float: "float",                 \
    double: "double",               \
    char*: "string",                \
    default: "unknown")

#define print_val(x) _Generic((x),  \
    int: printf("%d\n", x),         \
    float: printf("%f\n", x),       \
    double: printf("%lf\n", x),     \
    char*: printf("%s\n", x))

int main() {
    printf("%s\n", type_name(42));
    printf("%s\n", type_name(3.14f));
    printf("%s\n", type_name("hello"));
    
    print_val(42);
    print_val("world");
    return 0;
}
```

**Answer:**
```
int
float
string
42
world
```
`_Generic` is C11's type-safe dispatch (compile-time, not runtime). Closest thing to C++ function overloading in C.

**Follow-up:** Can _Generic dispatch on `const` qualified types? → Yes, `const int` and `int` are different association types.

---

### Q23: Designated Initializers Level: SDE 1

**Question:**
```c
#include <stdio.h>

struct Config {
    int timeout;
    int retries;
    int verbose;
    int port;
};

int main() {
    struct Config cfg = {
        .port = 8080,
        .timeout = 30,
        .retries = 3,
        // .verbose not mentioned → ?
    };
    
    printf("timeout=%d retries=%d verbose=%d port=%d\n",
           cfg.timeout, cfg.retries, cfg.verbose, cfg.port);
    
    int arr[10] = {[0] = 1, [5] = 5, [9] = 9};
    printf("arr[3]=%d arr[5]=%d\n", arr[3], arr[5]);
    return 0;
}
```

**Answer:**
```
timeout=30 retries=3 verbose=0 port=8080
arr[3]=0 arr[5]=5
```
Unmentioned fields/elements are zero-initialized. Order of designated initializers doesn't matter in C (it does in C++20).

**Follow-up:** Can you mix designated and positional initializers? → In C99 yes (implementation-defined behavior); in C++ no.

---

### Q24: Variadic Functions Level: SDE 2

**Question:**
```c
#include <stdio.h>
#include <stdarg.h>

int sum(int count, ...) {
    va_list ap;
    va_start(ap, count);
    int total = 0;
    for (int i = 0; i < count; i++) {
        total += va_arg(ap, int);
    }
    va_end(ap);
    return total;
}

// Type-unsafe! What if caller passes wrong types?
int main() {
    printf("%d\n", sum(4, 10, 20, 30, 40));
    // printf("%d\n", sum(2, 3.14, 2.71));  // UB!
    return 0;
}
```

**Answer:**
```
100
```
`va_arg(ap, int)` interprets bytes as int. Passing `double` when `int` is expected → reads wrong bytes (UB). No type checking for variadic arguments.

**Follow-up:** How does `printf` know types? → Format string acts as a type manifest. Mismatch = UB (caught by `-Wformat`).

---

### Q25: Atomic Operations (C11) Level: SDE 3

**Question:**
```c
#include <stdio.h>
#include <stdatomic.h>
#include <threads.h>

atomic_int counter = 0;
atomic_flag lock = ATOMIC_FLAG_INIT;

void spinlock_acquire() {
    while (atomic_flag_test_and_set_explicit(&lock, memory_order_acquire)) {
        // spin
    }
}
void spinlock_release() {
    atomic_flag_clear_explicit(&lock, memory_order_release);
}

int thread_func(void *arg) {
    for (int i = 0; i < 100000; i++) {
        atomic_fetch_add_explicit(&counter, 1, memory_order_relaxed);
    }
    return 0;
}

int main() {
    thrd_t t1, t2;
    thrd_create(&t1, thread_func, NULL);
    thrd_create(&t2, thread_func, NULL);
    thrd_join(t1, NULL);
    thrd_join(t2, NULL);
    printf("counter: %d\n", counter);
    return 0;
}
```

**Answer:**
```
counter: 200000
```
Always correct `atomic_fetch_add` is an atomic RMW operation. `memory_order_relaxed` is sufficient here because we only care about the counter's final value, not ordering with other data.

**Follow-up:** When do you need `memory_order_seq_cst` vs `relaxed`? → Sequential consistency when you need total ordering visible to all threads; relaxed when you only need atomicity of a single variable.

---

### Q26: Container_of Macro (Linux Kernel) Level: SDE 3

**Question:**
```c
#include <stdio.h>
#include <stddef.h>

#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))

struct list_node {
    struct list_node *next;
};

struct task {
    int pid;
    char name[32];
    struct list_node node;  // embedded at some offset
    int priority;
};

int main() {
    struct task t = {.pid = 42, .name = "init", .priority = 5};
    struct list_node *np = &t.node;
    
    // Recover task pointer from list_node pointer
    struct task *tp = container_of(np, struct task, node);
    printf("pid=%d name=%s priority=%d\n", tp->pid, tp->name, tp->priority);
    printf("offset of node: %lu\n", offsetof(struct task, node));
    return 0;
}
```

**Answer:**
```
pid=42 name=init priority=5
offset of node: 36
```
`container_of` subtracts the member's offset from the member pointer to find the enclosing struct. Foundation of Linux kernel linked lists allows intrusive data structures without separate allocation.

**Follow-up:** Why is this preferred over storing `void*` in list nodes? → No extra allocation, better cache locality, type-safe with macro.
