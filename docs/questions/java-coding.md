# Java Coding & Output Questions

> Real Java interview questions from Amazon, Google, Goldman Sachs, and backend roles.
> Focus: JVM behavior, concurrency, generics, collections, and "What does this print?"

---

### Q1: String Pool and == vs .equals() Level: SDE 1

**Question:**
```java
public class Main {
    public static void main(String[] args) {
        String a = "hello";
        String b = "hello";
        String c = new String("hello");
        String d = c.intern();

        System.out.println(a == b);
        System.out.println(a == c);
        System.out.println(a == d);
        System.out.println(a.equals(c));
    }
}
```

**Answer:**
```
true
false
true
true
```
- `a == b`: Same interned string literal → same reference.
- `a == c`: `new String()` creates a heap object → different reference.
- `a == d`: `intern()` returns the pool reference → same as `a`.
- `equals()` compares content, always `true`.

**Follow-up:** Where is the String pool stored? → In the heap (moved from PermGen in Java 7+).

---

### Q2: HashMap Collision and equals/hashCode Level: SDE 2

**Question:**
```java
import java.util.*;

class Key {
    int id;
    Key(int id) { this.id = id; }
    
    @Override
    public int hashCode() { return 1; } // all same bucket!
    
    @Override
    public boolean equals(Object o) {
        return o instanceof Key && ((Key) o).id == this.id;
    }
}

public class Main {
    public static void main(String[] args) {
        Map<Key, String> map = new HashMap<>();
        map.put(new Key(1), "one");
        map.put(new Key(2), "two");
        map.put(new Key(1), "ONE");
        
        System.out.println(map.size());
        System.out.println(map.get(new Key(1)));
        System.out.println(map.get(new Key(2)));
    }
}
```

**Answer:**
```
2
ONE
two
```
All keys hash to bucket 1. Within the bucket, `equals()` differentiates them. `Key(1)` replaces previous value. Java 8+ converts long chains to red-black trees (threshold: 8).

**Follow-up:** What's the time complexity if all elements collide? → O(log n) with treeification (Java 8+), O(n) before.

---

### Q3: ConcurrentModificationException Level: SDE 2

**Question:**
```java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        
        for (Integer num : list) {
            if (num % 2 == 0) {
                list.remove(num);  // Bug!
            }
        }
        System.out.println(list);
    }
}
```

**Answer:**
**Throws `ConcurrentModificationException`** at the second iteration after removal. The enhanced for-loop uses an iterator; modifying the list during iteration breaks the fail-fast mechanism.

**Fix:** Use `list.removeIf(n -> n % 2 == 0)` or explicit `Iterator` with `iter.remove()`.

**Follow-up:** Does `CopyOnWriteArrayList` throw this? → No, iterators work on a snapshot.

---

### Q4: Generics Type Erasure Level: SDE 2

**Question:**
```java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        List<Integer> ints = new ArrayList<>();
        List<String> strs = new ArrayList<>();
        
        System.out.println(ints.getClass() == strs.getClass());
        System.out.println(ints instanceof List);
        // System.out.println(ints instanceof List<Integer>); // compile error?
    }
}
```

**Answer:**
```
true
true
```
Type erasure removes generic type info at runtime. Both are just `ArrayList`. `instanceof List<Integer>` is a **compile error** can't check parameterized types at runtime.

**Follow-up:** How to get around erasure? → Pass `Class<T>` token, or use super type tokens (TypeReference pattern).

---

### Q5: Autoboxing/Unboxing Null Level: SDE 2

**Question:**
```java
public class Main {
    public static void main(String[] args) {
        Integer a = 127;
        Integer b = 127;
        Integer c = 128;
        Integer d = 128;
        
        System.out.println(a == b);
        System.out.println(c == d);
        
        Integer x = null;
        int y = x;  // what happens?
    }
}
```

**Answer:**
```
true
false
```
Then **`NullPointerException`** on unboxing `null` to `int`.

- `127 == 127`: Integer cache range [-128, 127] → same object.
- `128 == 128`: Outside cache → different objects.
- Unboxing `null` calls `x.intValue()` → NPE.

**Follow-up:** Can you extend the Integer cache range? → Yes: `-XX:AutoBoxCacheMax=N`.

---

### Q6: synchronized vs volatile Level: SDE 3

**Question:**
```java
public class Main {
    private static volatile boolean running = true;
    private static int counter = 0;

    public static void main(String[] args) throws Exception {
        Thread t = new Thread(() -> {
            while (running) {
                counter++;  // not atomic!
            }
            System.out.println("counter: " + counter);
        });
        t.start();
        Thread.sleep(100);
        running = false;
        t.join();
    }
}
```

**Answer:**
Prints some value of counter. `volatile` on `running` ensures visibility of the stop signal. But `counter++` is **not** thread-safe (read-modify-write). With a single writer thread, the final value is correct here. With multiple writers, use `AtomicInteger`.

**Follow-up:** Does `volatile` prevent reordering? → Yes, it establishes a happens-before relationship. But doesn't provide atomicity for compound actions.

---

### Q7: Serialization Gotcha Level: SDE 2

**Question:**
```java
import java.io.*;

class Parent {
    int x;
    Parent() { x = 10; System.out.println("Parent()"); }
}

class Child extends Parent implements Serializable {
    int y;
    Child() { y = 20; System.out.println("Child()"); }
}

public class Main {
    public static void main(String[] args) throws Exception {
        Child c = new Child();
        c.x = 99;
        
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        new ObjectOutputStream(bos).writeObject(c);
        
        Child c2 = (Child) new ObjectInputStream(
            new ByteArrayInputStream(bos.toByteArray())).readObject();
        
        System.out.println(c2.x + " " + c2.y);
    }
}
```

**Answer:**
```
Parent()
Child()
Parent()
10 20
```
During deserialization, the **non-serializable parent's constructor is called** (resets `x` to 10). The serializable child's state (`y=20`) is restored from the stream. `x=99` is lost.

**Follow-up:** How to preserve parent state? → Implement `writeObject`/`readObject` to manually serialize parent fields.

---

### Q8: Default Methods Diamond Problem Level: SDE 2

**Question:**
```java
interface A {
    default String greet() { return "A"; }
}
interface B extends A {
    default String greet() { return "B"; }
}
interface C extends A {
    default String greet() { return "C"; }
}
// class D implements B, C {}  // Compile error?

class D implements B, C {
    @Override
    public String greet() { return B.super.greet(); }
}

public class Main {
    public static void main(String[] args) {
        System.out.println(new D().greet());
    }
}
```

**Answer:**
Without override: **compile error** ambiguous default methods.
With explicit override delegating to `B.super.greet()`:
```
B
```

**Follow-up:** What's the resolution rule? → Class wins > sub-interface wins > must override.

---

### Q9: Stream Lazy Evaluation Level: SDE 2

**Question:**
```java
import java.util.stream.*;
import java.util.*;

public class Main {
    public static void main(String[] args) {
        List<String> result = Stream.of("a", "b", "c", "d", "e")
            .filter(s -> {
                System.out.println("filter: " + s);
                return s.compareTo("c") < 0;
            })
            .map(s -> {
                System.out.println("map: " + s);
                return s.toUpperCase();
            })
            .limit(1)
            .collect(Collectors.toList());
        
        System.out.println(result);
    }
}
```

**Answer:**
```
filter: a
map: a
[A]
```
Streams are lazy + short-circuit. `limit(1)` stops after the first element passes the pipeline. "b", "c", "d", "e" are never processed.

**Follow-up:** What if `limit` came before `filter`? → Only first element would enter `filter`.

---

### Q10: CompletableFuture Composition Level: SDE 3

**Question:**
```java
import java.util.concurrent.*;

public class Main {
    public static void main(String[] args) throws Exception {
        CompletableFuture<String> future = CompletableFuture
            .supplyAsync(() -> {
                System.out.println("step 1: " + Thread.currentThread().getName());
                return "hello";
            })
            .thenApply(s -> {
                System.out.println("step 2: " + Thread.currentThread().getName());
                return s.toUpperCase();
            })
            .thenApply(s -> s + " WORLD");
        
        System.out.println(future.get());
    }
}
```

**Answer:**
```
step 1: ForkJoinPool.commonPool-worker-1
step 2: main (or worker thread)
HELLO WORLD
```
`thenApply` may execute on the completing thread OR the calling thread (if already complete). Use `thenApplyAsync` to guarantee a different thread.

**Follow-up:** What happens if `supplyAsync` throws? → The future completes exceptionally. `get()` throws `ExecutionException`.

---

### Q11: Memory Leak with Inner Classes Level: SDE 2

**Question:**
```java
public class Outer {
    private byte[] data = new byte[10_000_000]; // 10MB
    
    class Inner {
        void doWork() { /* doesn't use data */ }
    }
    
    public static Inner createInner() {
        Outer outer = new Outer();
        return outer.new Inner(); // leaks Outer!
    }
    
    public static void main(String[] args) {
        List<Inner> list = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            list.add(createInner());
        }
        // 1GB retained!
    }
}
```

**Answer:**
Non-static inner class holds implicit reference to outer class. 100 `Inner` instances → 100 `Outer` instances (each 10MB) → 1GB retained.

**Fix:** Make `Inner` a `static` nested class (no implicit outer reference).

**Follow-up:** How to detect? → Heap dump analysis. Look for `this$0` field in inner class instances.

---

### Q12: ClassLoader Basics Level: SDE 3

**Question:**
```java
public class Main {
    public static void main(String[] args) {
        System.out.println(String.class.getClassLoader());
        System.out.println(Main.class.getClassLoader());
        System.out.println(Main.class.getClassLoader().getParent());
    }
}
```

**Answer:**
```
null
jdk.internal.loader.ClassLoaders$AppClassLoader@...
jdk.internal.loader.ClassLoaders$PlatformClassLoader@...
```
- `String` → Bootstrap classloader (native, returns `null`)
- `Main` → Application classloader
- Parent → Platform classloader (was Extension in Java 8)

**Follow-up:** What's the delegation model? → Parent-first: App → Platform → Bootstrap. Prevents class shadowing.

---

### Q13: ThreadLocal Usage Level: SDE 3

**Question:**
```java
import java.util.concurrent.*;

public class Main {
    private static ThreadLocal<List<String>> context = 
        ThreadLocal.withInitial(ArrayList::new);
    
    public static void main(String[] args) throws Exception {
        ExecutorService pool = Executors.newFixedThreadPool(1);
        
        pool.submit(() -> {
            context.get().add("task1");
            System.out.println(context.get());
        });
        
        pool.submit(() -> {
            System.out.println(context.get()); // clean?
        });
        
        pool.shutdown();
        pool.awaitTermination(1, TimeUnit.SECONDS);
    }
}
```

**Answer:**
```
[task1]
[task1]
```
Thread pool reuses threads. ThreadLocal state persists across tasks on the same thread. **Memory leak** if not cleaned.

**Fix:** Always call `context.remove()` in a `finally` block.

**Follow-up:** What's `InheritableThreadLocal`? → Child threads inherit parent's value (copy at thread creation).

---

### Q14: Optional Misuse Patterns Level: SDE 2

**Question:**
```java
import java.util.*;

public class Main {
    // Anti-pattern: Optional as method parameter
    static String greet(Optional<String> name) {
        return "Hello, " + name.orElse("World");
    }
    
    public static void main(String[] args) {
        // Anti-pattern: Optional.get() without check
        Optional<String> opt = Optional.empty();
        
        try {
            String val = opt.get();
        } catch (NoSuchElementException e) {
            System.out.println("caught: " + e.getMessage());
        }
        
        // Correct pattern
        String result = opt.map(String::toUpperCase).orElse("default");
        System.out.println(result);
    }
}
```

**Answer:**
```
caught: No value present
default
```

Anti-patterns: (1) `Optional` as parameter use overloading or `@Nullable`. (2) `opt.get()` without `isPresent()`. (3) `Optional` for class fields.

**Follow-up:** When to use Optional? → Only as return type to signal "may be absent". Never in fields, parameters, or collections.

---

### Q15: try-with-resources Edge Cases Level: SDE 2

**Question:**
```java
public class Main {
    static class MyResource implements AutoCloseable {
        String name;
        MyResource(String name) {
            this.name = name;
            System.out.println("open: " + name);
        }
        @Override
        public void close() {
            System.out.println("close: " + name);
            throw new RuntimeException(name + " close failed");
        }
    }
    
    public static void main(String[] args) {
        try (MyResource a = new MyResource("A");
             MyResource b = new MyResource("B")) {
            throw new RuntimeException("body");
        } catch (RuntimeException e) {
            System.out.println("caught: " + e.getMessage());
            System.out.println("suppressed: " + e.getSuppressed().length);
        }
    }
}
```

**Answer:**
```
open: A
open: B
close: B
close: A
caught: body
suppressed: 2
```
Resources close in reverse order. Exceptions from `close()` are **suppressed** (attached to the primary exception from the body).

**Follow-up:** How to access suppressed exceptions? → `e.getSuppressed()` returns an array.

---

### Q16: Enum Singleton Pattern Level: SDE 2

**Question:**
```java
enum DatabaseConnection {
    INSTANCE;
    
    private String url = "jdbc:mysql://localhost:3306/db";
    
    public String getUrl() { return url; }
}

public class Main {
    public static void main(String[] args) {
        System.out.println(DatabaseConnection.INSTANCE.getUrl());
        System.out.println(DatabaseConnection.INSTANCE == DatabaseConnection.INSTANCE);
    }
}
```

**Answer:**
```
jdbc:mysql://localhost:3306/db
true
```
Enum singletons are: thread-safe (JVM guarantees), serialization-safe (no duplicate on deserialization), reflection-safe (can't create enum via reflection).

**Follow-up:** Why prefer enum over double-checked locking singleton? → Simpler, handles edge cases automatically.

---

### Q17: Comparator and Natural Ordering Level: SDE 2

**Question:**
```java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        List<String> list = Arrays.asList("banana", "Apple", "cherry", "date");
        
        Collections.sort(list);
        System.out.println(list);
        
        list.sort(String.CASE_INSENSITIVE_ORDER);
        System.out.println(list);
        
        list.sort(Comparator.comparingInt(String::length).thenComparing(Comparator.naturalOrder()));
        System.out.println(list);
    }
}
```

**Answer:**
```
[Apple, banana, cherry, date]
[Apple, banana, cherry, date]
[date, Apple, banana, cherry]
```
- Natural order: uppercase letters before lowercase (Unicode).
- Case-insensitive: alphabetical regardless of case.
- By length then natural: 4, 5, 6, 6 characters.

**Follow-up:** What's the difference between `Comparable` and `Comparator`? → `Comparable` defines natural order on the class; `Comparator` is external/alternate ordering.

---

### Q18: WeakReference and GC Level: SDE 3

**Question:**
```java
import java.lang.ref.*;
import java.util.*;

public class Main {
    public static void main(String[] args) {
        Object strong = new Object();
        WeakReference<Object> weak = new WeakReference<>(strong);
        
        System.out.println(weak.get() != null);
        
        strong = null;  // remove strong reference
        System.gc();    // suggest GC
        
        System.out.println(weak.get() != null);
    }
}
```

**Answer:**
```
true
false (most likely)
```
After `strong = null`, the object is only weakly reachable. GC can collect it. `System.gc()` is a hint; in practice it usually triggers collection.

**Follow-up:** Use case? → WeakHashMap for caches, avoiding memory leaks in listener registries.

---

### Q19: Virtual Threads (Java 21) Level: SDE 3

**Question:**
```java
import java.util.concurrent.*;
import java.time.*;

public class Main {
    public static void main(String[] args) throws Exception {
        Instant start = Instant.now();
        
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 10_000; i++) {
                executor.submit(() -> {
                    Thread.sleep(Duration.ofSeconds(1));
                    return null;
                });
            }
        }
        
        System.out.println("Elapsed: " + Duration.between(start, Instant.now()).toMillis() + "ms");
    }
}
```

**Answer:**
~1000ms (not 10000 * 1000ms). Virtual threads are lightweight (no OS thread per task). 10,000 sleeping virtual threads share a small pool of carrier threads.

**Follow-up:** When NOT to use virtual threads? → CPU-bound work, pinning issues with `synchronized` blocks, and native code.

---

### Q20: Record Classes (Java 16+) Level: SDE 2

**Question:**
```java
record Point(int x, int y) {
    Point {
        if (x < 0 || y < 0) throw new IllegalArgumentException("negative");
    }
}

public class Main {
    public static void main(String[] args) {
        Point p1 = new Point(1, 2);
        Point p2 = new Point(1, 2);
        
        System.out.println(p1.equals(p2));
        System.out.println(p1 == p2);
        System.out.println(p1.hashCode() == p2.hashCode());
        System.out.println(p1);
    }
}
```

**Answer:**
```
true
false
true
Point[x=1, y=2]
```
Records auto-generate: `equals` (field-wise), `hashCode`, `toString`, accessors. Compact constructor validates without explicit field assignment.

**Follow-up:** Can records be mutable? → No. Fields are `final`. Records are shallowly immutable (contained objects can mutate).

---

### Q21: Pattern Matching instanceof (Java 16+) Level: SDE 2

**Question:**
```java
public class Main {
    sealed interface Shape permits Circle, Rectangle {}
    record Circle(double radius) implements Shape {}
    record Rectangle(double w, double h) implements Shape {}
    
    static double area(Shape shape) {
        return switch (shape) {
            case Circle c -> Math.PI * c.radius() * c.radius();
            case Rectangle r -> r.w() * r.h();
        };
    }
    
    public static void main(String[] args) {
        System.out.printf("%.2f%n", area(new Circle(5)));
        System.out.printf("%.2f%n", area(new Rectangle(3, 4)));
    }
}
```

**Answer:**
```
78.54
12.00
```
Sealed interfaces + pattern matching = exhaustive switch (no default needed). Compiler checks all cases are covered.

**Follow-up:** What's the advantage over visitor pattern? → Less boilerplate, compiler-enforced exhaustiveness.

---

### Q22: Deadlock with synchronized Level: SDE 3

**Question:**
```java
public class Main {
    static final Object LOCK_A = new Object();
    static final Object LOCK_B = new Object();
    
    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            synchronized (LOCK_A) {
                System.out.println("T1: got A");
                try { Thread.sleep(50); } catch (Exception e) {}
                synchronized (LOCK_B) {
                    System.out.println("T1: got B");
                }
            }
        });
        
        Thread t2 = new Thread(() -> {
            synchronized (LOCK_B) {
                System.out.println("T2: got B");
                try { Thread.sleep(50); } catch (Exception e) {}
                synchronized (LOCK_A) {
                    System.out.println("T2: got A");
                }
            }
        });
        
        t1.start();
        t2.start();
    }
}
```

**Answer:**
```
T1: got A
T2: got B
(deadlock hangs forever)
```
Classic lock ordering deadlock. T1 holds A, wants B. T2 holds B, wants A.

**Fix:** Always acquire locks in the same order, or use `tryLock` with timeout via `ReentrantLock`.

**Follow-up:** How to detect? → `jstack <pid>` shows deadlock analysis. Or `ThreadMXBean.findDeadlockedThreads()`.

---

### Q23: HashMap vs ConcurrentHashMap Null Handling Level: SDE 2

**Question:**
```java
import java.util.*;
import java.util.concurrent.*;

public class Main {
    public static void main(String[] args) {
        Map<String, String> hm = new HashMap<>();
        hm.put(null, "null-key");
        hm.put("key", null);
        System.out.println(hm.get(null));
        System.out.println(hm.get("key"));
        
        Map<String, String> chm = new ConcurrentHashMap<>();
        try {
            chm.put("key", null); // throws?
        } catch (NullPointerException e) {
            System.out.println("NPE: " + e.getMessage());
        }
    }
}
```

**Answer:**
```
null-key
null
NPE: ...
```
`HashMap` allows one null key and null values. `ConcurrentHashMap` prohibits null keys AND null values ambiguity: does `get()` return null because key is absent or value is null?

**Follow-up:** Why the design difference? → In concurrent context, `containsKey` + `get` isn't atomic. Null values make it impossible to distinguish missing vs present-with-null.

---

### Q24: Lambda and Effectively Final Level: SDE 2

**Question:**
```java
import java.util.*;
import java.util.stream.*;

public class Main {
    public static void main(String[] args) {
        int[] counter = {0}; // array trick
        // int counter = 0;  // would this work in lambda?
        
        List<String> words = Arrays.asList("hello", "world", "java");
        words.forEach(w -> counter[0]++);
        
        System.out.println(counter[0]);
    }
}
```

**Answer:**
```
3
```
A plain `int counter` inside a lambda would fail must be effectively final. The array trick works because the **reference** is effectively final (array contents can change).

**Follow-up:** Better alternative? → `AtomicInteger`, or use `stream().count()` / `reduce()` for functional style.

---

### Q25: Checked vs Unchecked Exceptions Level: SDE 2

**Question:**
```java
import java.util.function.*;

public class Main {
    // This won't compile:
    // Function<String, Integer> parse = s -> Integer.parseInt(s); // throws NumberFormatException (unchecked - OK)
    
    // This won't compile either:
    // Function<String, byte[]> read = s -> Files.readAllBytes(Path.of(s)); // throws IOException (checked - ERROR)
    
    @FunctionalInterface
    interface ThrowingFunction<T, R> {
        R apply(T t) throws Exception;
    }
    
    static <T, R> Function<T, R> unchecked(ThrowingFunction<T, R> f) {
        return t -> {
            try { return f.apply(t); }
            catch (Exception e) { throw new RuntimeException(e); }
        };
    }
    
    public static void main(String[] args) {
        Function<String, Integer> safeParseInt = unchecked(Integer::parseInt);
        System.out.println(safeParseInt.apply("42"));
        
        try {
            safeParseInt.apply("abc");
        } catch (RuntimeException e) {
            System.out.println("caught: " + e.getCause().getClass().getSimpleName());
        }
    }
}
```

**Answer:**
```
42
caught: NumberFormatException
```
Java's functional interfaces don't declare checked exceptions. The `unchecked` wrapper converts checked to runtime exceptions.

**Follow-up:** How does Kotlin/Scala handle this? → No checked exceptions at all.

---

### Q26: Garbage Collection and finalize Level: SDE 3

**Question:**
```java
public class Main {
    @Override
    protected void finalize() throws Throwable {
        System.out.println("finalized");
        // Anti-pattern: object resurrection
        Holder.ref = this;
    }
    
    static Main ref;
    
    static class Holder {
        static Main ref;
    }
    
    public static void main(String[] args) throws Exception {
        ref = new Main();
        ref = null;
        System.gc();
        Thread.sleep(100);
        System.out.println("after gc: " + (Holder.ref != null));
    }
}
```

**Answer:**
```
finalized
after gc: true
```
`finalize()` can resurrect an object by storing `this` in a reachable location. The object survives GC. **Deprecated since Java 9, removed in Java 18.**

**Follow-up:** What replaces `finalize`? → `Cleaner` API or explicit `close()` with try-with-resources.
