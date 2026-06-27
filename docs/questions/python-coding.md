# Python Coding & Output Questions

> Real Python interview gotchas from Google, Meta, Stripe, and fintech firms.
> Focus: language quirks, memory model, concurrency, and "What does this print?"

---

### Q1: Mutable Default Arguments Level: SDE 1

**Question:**
```python
def append_to(element, target=[]):
    target.append(element)
    return target

print(append_to(1))
print(append_to(2))
print(append_to(3))
```

**Answer:**
```
[1]
[1, 2]
[1, 2, 3]
```
Default mutable arguments are evaluated **once** at function definition, not per call. The same list object is reused.

**Fix:** `def append_to(element, target=None): target = target or []`

**Follow-up:** Where is the default stored? → `append_to.__defaults__` tuple.

---

### Q2: `is` vs `==` with Small Integers Level: SDE 1

**Question:**
```python
a = 256
b = 256
print(a is b)

c = 257
d = 257
print(c is d)
```

**Answer:**
```
True
False  # (in interactive mode; may vary in script)
```
CPython caches integers in [-5, 256]. `a is b` checks identity (same object). Beyond that range, new objects are created. In a compiled script, the compiler may intern literals making both `True`.

**Follow-up:** Should you ever use `is` for value comparison? → Only for `None`, `True`, `False`.

---

### Q3: List Comprehension Scope (Python 3) Level: SDE 2

**Question:**
```python
x = 10
result = [x for x in range(5)]
print(x)
print(result)
```

**Answer:**
```
10
[0, 1, 2, 3, 4]
```
In Python 3, list comprehensions have their own scope. The outer `x` is not affected. (In Python 2, `x` would be 4.)

**Follow-up:** What about generator expressions? → Always had their own scope, even in Python 2.

---

### Q4: Closure Late Binding Level: SDE 2

**Question:**
```python
funcs = [lambda: i for i in range(4)]
print([f() for f in funcs])
```

**Answer:**
```
[3, 3, 3, 3]
```
Closures capture the **variable**, not the value. By the time the lambdas execute, `i` is 3.

**Fix:** `funcs = [lambda i=i: i for i in range(4)]` default argument captures value at definition time.

**Follow-up:** Alternative fix using `functools.partial`.

---

### Q5: Generator vs List Memory Level: SDE 2

**Question:**
```python
import sys
list_comp = [i*i for i in range(1000000)]
gen_expr = (i*i for i in range(1000000))
print(sys.getsizeof(list_comp))  # ~8MB
print(sys.getsizeof(gen_expr))   # ~200 bytes
```

**Answer:**
List: ~8,448,728 bytes (stores all values).
Generator: ~200 bytes (stores only the frame/state).

Generators produce values lazily. Use for pipelines where you don't need all values in memory simultaneously.

**Follow-up:** Can you index into a generator? → No. It's a forward-only iterator. Use `itertools.islice`.

---

### Q6: GIL Implications Level: SDE 3

**Question:**
```python
import threading
counter = 0
def increment():
    global counter
    for _ in range(1000000):
        counter += 1

t1 = threading.Thread(target=increment)
t2 = threading.Thread(target=increment)
t1.start(); t2.start()
t1.join(); t2.join()
print(counter)  # Always 2000000?
```

**Answer:**
**No.** Result will be less than 2000000 (typically ~1.2-1.8M). `counter += 1` is not atomic it's `LOAD, ADD, STORE`. GIL can release between these bytecodes.

**Fix:** Use `threading.Lock()` or `itertools`-based approach, or `multiprocessing` for true parallelism.

**Follow-up:** Does Python 3.13's free-threaded mode fix this? → Still need synchronization; it removes GIL but doesn't make operations atomic.

---

### Q7: Decorator Order Level: SDE 2

**Question:**
```python
def decorator_a(func):
    def wrapper(*args, **kwargs):
        print("A before")
        result = func(*args, **kwargs)
        print("A after")
        return result
    return wrapper

def decorator_b(func):
    def wrapper(*args, **kwargs):
        print("B before")
        result = func(*args, **kwargs)
        print("B after")
        return result
    return wrapper

@decorator_a
@decorator_b
def hello():
    print("hello")

hello()
```

**Answer:**
```
A before
B before
hello
B after
A after
```
Decorators apply bottom-up: `hello = decorator_a(decorator_b(hello))`. Execution is outer-first (A wraps B wraps hello).

**Follow-up:** How would you preserve `__name__` and `__doc__`? → `@functools.wraps(func)`.

---

### Q8: `__slots__` vs `__dict__` Level: SDE 2

**Question:**
```python
import sys

class WithDict:
    def __init__(self, x, y):
        self.x = x
        self.y = y

class WithSlots:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x = x
        self.y = y

a = WithDict(1, 2)
b = WithSlots(1, 2)
print(sys.getsizeof(a) + sys.getsizeof(a.__dict__))
print(sys.getsizeof(b))
```

**Answer:**
`WithDict`: ~152+ bytes (object + dict overhead).
`WithSlots`: ~56 bytes.

`__slots__` avoids per-instance `__dict__`, saving memory. Useful for millions of small objects.

**Follow-up:** Can you add new attributes to a slots class? → No, raises `AttributeError`. No `__dict__` unless explicitly added to `__slots__`.

---

### Q9: `deepcopy` vs `copy` Level: SDE 2

**Question:**
```python
import copy
original = [[1, 2, 3], [4, 5, 6]]
shallow = copy.copy(original)
deep = copy.deepcopy(original)

original[0][0] = 99
original.append([7, 8, 9])

print(shallow)
print(deep)
```

**Answer:**
```
[[99, 2, 3], [4, 5, 6]]
[[1, 2, 3], [4, 5, 6]]
```
Shallow copy: new outer list, same inner list objects. `original[0][0] = 99` affects shallow but not deep. `append` doesn't affect either (new element in outer list).

**Follow-up:** How does `deepcopy` handle circular references? → Uses a memo dict to track already-copied objects.

---

### Q10: asyncio Event Loop Basics Level: SDE 3

**Question:**
```python
import asyncio

async def say(msg, delay):
    await asyncio.sleep(delay)
    print(msg)

async def main():
    await say("first", 2)
    await say("second", 1)

asyncio.run(main())
```
How long does this take?

**Answer:**
**3 seconds.** Each `await` is sequential. `say("first", 2)` takes 2s, then `say("second", 1)` takes 1s.

**Fix for concurrency:**
```python
async def main():
    await asyncio.gather(say("first", 2), say("second", 1))
```
Now takes 2s (concurrent).

**Follow-up:** What's the difference between `gather`, `TaskGroup`, and `create_task`?

---

### Q11: Dictionary Ordering Level: SDE 1

**Question:**
```python
d = {}
d['b'] = 2
d['a'] = 1
d['c'] = 3
print(list(d.keys()))
del d['a']
d['a'] = 1
print(list(d.keys()))
```

**Answer:**
```
['b', 'a', 'c']
['b', 'c', 'a']
```
Since Python 3.7, dicts maintain **insertion order**. Deleting and re-adding moves the key to the end.

**Follow-up:** Is `dict` a valid replacement for `OrderedDict`? → Mostly, except `OrderedDict` equality considers order; `dict` equality doesn't.

---

### Q12: `*args` Unpacking Edge Cases Level: SDE 2

**Question:**
```python
def f(a, b, *args, key=None, **kwargs):
    print(a, b, args, key, kwargs)

f(1, 2, 3, 4, key="K", extra="E")
```

**Answer:**
```
1 2 (3, 4) K {'extra': 'E'}
```
- `a=1, b=2` positional
- `args=(3,4)` remaining positional
- `key="K"` keyword-only (after `*args`)
- `kwargs={'extra':'E'}` remaining keyword

**Follow-up:** What does `def f(*, key)` mean? → All arguments must be keyword-only; no positional allowed.

---

### Q13: Class vs Instance Variables Level: SDE 2

**Question:**
```python
class Dog:
    tricks = []  # class variable!
    
    def __init__(self, name):
        self.name = name
    
    def add_trick(self, trick):
        self.tricks.append(trick)

d1 = Dog("Rex")
d2 = Dog("Buddy")
d1.add_trick("roll over")
print(d2.tricks)
```

**Answer:**
```
['roll over']
```
`tricks` is a **class variable** shared by all instances. Mutating it via one instance affects all.

**Fix:** Initialize in `__init__`: `self.tricks = []`.

**Follow-up:** What if you do `d1.tricks = ['sit']`? → Creates an instance variable, shadowing the class variable only for `d1`.

---

### Q14: `__new__` vs `__init__` Level: SDE 3

**Question:**
```python
class Singleton:
    _instance = None
    
    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            print("created")
        return cls._instance
    
    def __init__(self, value):
        self.value = value
        print(f"init {value}")

a = Singleton(1)
b = Singleton(2)
print(a is b)
print(a.value)
```

**Answer:**
```
created
init 1
init 2
True
2
```
`__new__` creates the instance (called once due to singleton check). `__init__` is called every time overwriting `value`.

**Follow-up:** How to prevent `__init__` re-runs? → Use a flag: `if not hasattr(self, '_initialized')`.

---

### Q15: Exception Chaining Level: SDE 2

**Question:**
```python
def load_config():
    try:
        open("nonexistent.yaml")
    except FileNotFoundError as e:
        raise RuntimeError("Config failed") from e

try:
    load_config()
except RuntimeError as e:
    print(e)
    print(type(e.__cause__))
```

**Answer:**
```
Config failed
<class 'FileNotFoundError'>
```
`from e` explicitly chains exceptions. `__cause__` stores the original. Without `from e`, you get `__context__` (implicit chaining).

**Follow-up:** How to suppress chaining? → `raise RuntimeError("msg") from None`.

---

### Q16: f-string Evaluation Order Level: SDE 2

**Question:**
```python
x = 0
def inc():
    global x
    x += 1
    return x

print(f"{inc()} {inc()} {inc()}")
print(f"{x=}")
```

**Answer:**
```
1 2 3
x=3
```
f-string expressions evaluate left-to-right. Each `inc()` call executes in sequence.

**Follow-up:** Can f-strings contain `=` for debugging? → Yes, `f"{expr=}"` prints both the expression and its value (Python 3.8+).

---

### Q17: Walrus Operator Patterns Level: SDE 2

**Question:**
```python
data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
result = [y for x in data if (y := x**2) > 20]
print(result)
print(y)  # what's y here?
```

**Answer:**
```
[25, 36, 49, 64, 81, 100]
100
```
The walrus operator `:=` assigns within an expression. `y` leaks into the enclosing scope (unlike the iteration variable `x` in Python 3 list comps wait, `x` also leaks? No, list comp variables don't leak in Python 3, but walrus-assigned variables DO).

**Follow-up:** Why does `:=` leak? → By design: it assigns in the enclosing scope, not the comprehension scope.

---

### Q18: Type Hints and Runtime Behavior Level: SDE 2

**Question:**
```python
def add(a: int, b: int) -> int:
    return a + b

result = add("hello", " world")
print(result)
print(add.__annotations__)
```

**Answer:**
```
hello world
{'a': <class 'int'>, 'b': <class 'int'>, 'return': <class 'int'>}
```
Type hints are **not enforced at runtime**. They're metadata only. Use `mypy` or `pyright` for static checking.

**Follow-up:** How to enforce at runtime? → Use `beartype`, `typeguard`, or manual `isinstance` checks.

---

### Q19: Property and Descriptor Protocol Level: SDE 3

**Question:**
```python
class Temperature:
    def __init__(self, celsius=0):
        self._celsius = celsius
    
    @property
    def fahrenheit(self):
        return self._celsius * 9/5 + 32
    
    @fahrenheit.setter
    def fahrenheit(self, value):
        self._celsius = (value - 32) * 5/9

t = Temperature(100)
print(t.fahrenheit)
t.fahrenheit = 32
print(t._celsius)
```

**Answer:**
```
212.0
0.0
```
`@property` creates a descriptor that intercepts attribute access. The setter converts Fahrenheit back to Celsius.

**Follow-up:** How would you implement `@property` from scratch? → Using `__get__`, `__set__`, `__delete__` descriptor protocol.

---

### Q20: `collections.defaultdict` Trap Level: SDE 1

**Question:**
```python
from collections import defaultdict
d = defaultdict(list)
print(d["missing"])
print(len(d))
print("missing" in d)
```

**Answer:**
```
[]
1
True
```
Accessing a missing key in `defaultdict` **creates it** with the default factory. The key now exists.

**Follow-up:** How to check without creating? → Use `"key" in d` before access, or `d.get("key")`.

---

### Q21: Multiple Inheritance MRO Level: SDE 3

**Question:**
```python
class A:
    def method(self):
        print("A")
        
class B(A):
    def method(self):
        print("B")
        super().method()

class C(A):
    def method(self):
        print("C")
        super().method()

class D(B, C):
    def method(self):
        print("D")
        super().method()

D().method()
print(D.__mro__)
```

**Answer:**
```
D
B
C
A
(<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
```
C3 linearization: D → B → C → A. `super()` follows MRO, not parent class directly.

**Follow-up:** What if B doesn't call `super()`? → C's `method` is never called (cooperative multiple inheritance breaks).

---

### Q22: Context Manager and Exception Handling Level: SDE 2

**Question:**
```python
class MyContext:
    def __enter__(self):
        print("enter")
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        print(f"exit: {exc_type}")
        return True  # suppress exception

with MyContext() as ctx:
    print("inside")
    raise ValueError("oops")

print("after")  # does this execute?
```

**Answer:**
```
enter
inside
exit: <class 'ValueError'>
after
```
`__exit__` returning `True` suppresses the exception. Execution continues after the `with` block.

**Follow-up:** What if `__exit__` returns `False`/`None`? → Exception propagates normally.

---

### Q23: `itertools` and Lazy Evaluation Level: SDE 2

**Question:**
```python
from itertools import islice, count

def fibonacci():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

fib = fibonacci()
first_10 = list(islice(fib, 10))
next_5 = list(islice(fib, 5))
print(first_10)
print(next_5)
```

**Answer:**
```
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
[55, 89, 144, 233, 377]
```
Generator maintains state. After consuming 10, the next `islice` continues from where it left off.

**Follow-up:** How to restart? → Create a new generator instance. Generators are single-use iterators.

---

### Q24: Metaclass Basics Level: SDE 3

**Question:**
```python
class Meta(type):
    def __new__(mcs, name, bases, namespace):
        namespace['created_by'] = 'Meta'
        cls = super().__new__(mcs, name, bases, namespace)
        print(f"Creating class: {name}")
        return cls

class MyClass(metaclass=Meta):
    pass

class SubClass(MyClass):
    pass

print(MyClass.created_by)
print(SubClass.created_by)
```

**Answer:**
```
Creating class: MyClass
Creating class: SubClass
Meta
Meta
```
Metaclass `__new__` is called for every class creation (including subclasses). It injects `created_by` into each class namespace.

**Follow-up:** Real use case? → Django models, SQLAlchemy declarative base, ABCMeta, dataclass-like registration.

---

### Q25: `__del__` and Garbage Collection Level: SDE 3

**Question:**
```python
import gc

class Ref:
    def __init__(self, name):
        self.name = name
    def __del__(self):
        print(f"del {self.name}")

a = Ref("a")
b = Ref("b")
a.other = b
b.other = a  # cycle
del a
del b
print("before gc")
gc.collect()
print("after gc")
```

**Answer:**
```
before gc
del a
del b
after gc
```
(Order of del may vary.) `del a` and `del b` remove name bindings but don't free objects due to circular reference. `gc.collect()` detects the cycle and calls `__del__`.

**Follow-up:** Why are finalizers (`__del__`) considered problematic? → Non-deterministic timing, resurrection issues, can prevent GC of cycles in older Python.

---

### Q26: Dataclass Gotchas Level: SDE 2

**Question:**
```python
from dataclasses import dataclass, field

@dataclass
class Config:
    name: str
    tags: list = field(default_factory=list)
    # tags: list = []  # Would this work?

c1 = Config("a")
c2 = Config("b")
c1.tags.append("x")
print(c2.tags)
```

**Answer:**
```
[]
```
`field(default_factory=list)` creates a new list per instance. Using `tags: list = []` directly would raise `ValueError` dataclasses explicitly prevent mutable defaults.

**Follow-up:** What does `@dataclass(frozen=True)` do? → Makes instances immutable (hashable). Attribute assignment raises `FrozenInstanceError`.
