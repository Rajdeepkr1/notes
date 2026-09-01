# Java Backend Development — Deep Dive Roadmap

We'll go from language fundamentals → JVM internals → concurrency → Spring ecosystem → data access → testing → production → interview problems.

---

## 1. Java Language Fundamentals

**Definition:** Java is a statically-typed, object-oriented, general-purpose programming language that compiles to platform-independent **bytecode**, executed by the **Java Virtual Machine (JVM)** — the "write once, run anywhere" model, since the same compiled `.class` bytecode runs unmodified on any platform with a compatible JVM.

**The compile → bytecode → JVM execution model — Definition:** source code (`.java`) is compiled by `javac` into bytecode (`.class` files) — an intermediate, platform-neutral instruction format, not native machine code — which the JVM then executes, either by interpreting it directly or compiling hot paths to native machine code just-in-time (JIT, section 5) for performance.

**Primitive types vs reference types — Definition:** Java has 8 **primitive** types (`int`, `long`, `double`, `float`, `boolean`, `char`, `byte`, `short`) — stored directly by value, not objects, with no method calls possible on them; everything else (classes, arrays, interfaces) is a **reference type** — a variable holds a reference (pointer) to an object living on the heap (section 5), not the object's data directly.

```java
int age = 30;              // primitive — value stored directly
String name = "Ada";       // reference — `name` holds a reference to a String object on the heap
```

**Arrays — Definition:** a fixed-size, homogeneously-typed, contiguous collection — `int[] nums = new int[5];` — a reference type in Java (even for primitive-typed arrays), with a fixed length set at creation that cannot grow (unlike `ArrayList`, section 3).

**Strings & the String pool — Definition:** `String` objects are **immutable** — any operation that appears to modify a string actually creates a new one. Java maintains a **String pool** (a special area of the heap) — string **literals** (`"hello"`) are automatically interned (deduplicated) into this pool, so two identical literals reference the *same* object; strings created via `new String("hello")` are **not** automatically pooled, creating a separate object even with identical content — a classic source of `==` vs `.equals()` confusion (`==` compares references, `.equals()` compares content).

```java
String a = "hello";
String b = "hello";
String c = new String("hello");
a == b;        // true — both reference the same pooled literal
a == c;        // false — c is a distinct object on the heap
a.equals(c);   // true — content is equal
```

**`String` vs `StringBuilder` vs `StringBuffer` — Definition:** because `String` is immutable, repeatedly concatenating strings in a loop (`result += item`) creates a new object on every iteration — O(n²) overall for n concatenations. `StringBuilder` is a **mutable** character sequence designed for efficient repeated modification (O(1) amortized append, same amortized-array-resize logic as the DSA notes' dynamic array section) — the standard tool for building up a string incrementally. `StringBuffer` is functionally identical to `StringBuilder` but with all its methods `synchronized` for thread safety — meaningfully slower in the (overwhelmingly common) single-threaded case, so `StringBuilder` is preferred unless the same builder instance is genuinely shared across threads.

```java
StringBuilder sb = new StringBuilder();
for (String item : items) sb.append(item).append(", ");
String result = sb.toString();
```

**Autoboxing & unboxing — Definition:** Java automatically converts between a primitive (`int`) and its corresponding **wrapper class** (`Integer`) where needed — autoboxing wraps a primitive into its object form (needed, e.g., to store `int`s in a `List<Integer>`, since generics, section 3, cannot use primitives directly); unboxing extracts the primitive back out. This convenience has real costs: autoboxing in a tight loop adds object-allocation overhead absent from pure primitive code, and comparing boxed values with `==` compares references (not content) outside the small cached range (`-128` to `127`, which the JVM pre-caches) — another classic gotcha.

```java
Integer a = 100, b = 100;
a == b; // true — within the cached Integer range
Integer c = 200, d = 200;
c == d; // false — outside the cache, two distinct objects
```

**Wrapper classes — Definition:** `Integer`, `Long`, `Double`, `Boolean`, etc. — object wrappers around each primitive type, providing utility methods (`Integer.parseInt`, `Integer.MAX_VALUE`) and enabling primitives to be used wherever an `Object` is required (generics, collections, nullability).

---

## 2. Object-Oriented Programming in Java

**Classes & objects — Definition:** a **class** is a blueprint defining the fields (state) and methods (behavior) an object of that type will have; an **object** is a specific instance of a class, created via `new`, living on the heap.

**Constructors & `this` — Definition:** a constructor initializes a newly-created object's state, sharing the class's name and having no return type; `this` refers to the current instance, most commonly used to disambiguate a constructor parameter from a field of the same name.

```java
public class User {
  private String name;
  public User(String name) { this.name = name; } // `this.name` (field) vs `name` (parameter)
}
```

**Encapsulation — Definition:** bundling an object's data together with the methods that operate on it, and restricting direct external access to that data via **access modifiers**: `private` (only within the same class), `protected` (same class, subclasses, and same package), (no modifier / package-private) (same package only), `public` (accessible everywhere) — the mechanism behind information hiding, letting a class's internal representation change without breaking external code that only depends on its public interface.

**Inheritance (`extends`) — Definition:** a class can inherit fields/methods from a single parent class (Java, unlike C++, supports only **single inheritance** of classes — a class can `extend` only one other class, though it can `implement` multiple interfaces).

```java
public class Animal { public void eat() { System.out.println("eating"); } }
public class Dog extends Animal { public void bark() { System.out.println("woof"); } }
```

**Polymorphism — Definition:** the ability for objects of different types to be treated through a common interface/parent-type reference, with the *actual* runtime type determining which specific behavior executes.

- **Method overloading** — multiple methods in the same class sharing a name but differing in parameter types/count — resolved at **compile time**, based on the declared argument types.
- **Method overriding** — a subclass provides its own implementation of a method inherited from its parent, marked with `@Override` (not required by the compiler, but a strongly-recommended annotation catching typos/signature mismatches at compile time) — resolved at **runtime**, via **dynamic dispatch**: calling a method on a parent-typed reference actually invokes the subclass's overridden version if the underlying object is a subclass instance.

```java
Animal a = new Dog(); // reference type Animal, actual object type Dog
a.eat();               // dynamic dispatch — calls Dog's eat() if overridden, else inherited Animal's eat()
```

**Abstraction — Definition:** exposing only essential behavior/interface while hiding implementation details.

- **Abstract classes** — a class that cannot be instantiated directly, may contain both fully-implemented methods and `abstract` methods (declared but not implemented, forcing subclasses to provide the implementation) — used when related classes share significant common implementation.
- **Interfaces** — a purely-behavioral contract (traditionally only method signatures, no implementation or state); since Java 8, interfaces may also include `default` methods (a default implementation subclasses can optionally override) and `static` methods — used when unrelated classes need to satisfy a common contract, and Java's single-inheritance limitation on classes doesn't apply to interfaces (a class can `implement` many).

```java
public interface Shape {
  double area();
  default String describe() { return "A shape with area " + area(); } // default method
}
```

**`super`** — refers to the immediate parent class, used to call an overridden parent method explicitly (`super.eat()`) or to invoke a parent constructor (`super(args)`, must be the first statement in a subclass constructor).

**Composition over inheritance — Definition:** a widely-held design principle favoring building complex behavior by **composing** objects together (a class holding a reference to another class as a field, delegating to it) rather than through deep inheritance hierarchies — inheritance creates tight coupling to a parent's implementation details and can produce fragile, hard-to-modify hierarchies as requirements evolve; composition keeps components independently swappable/testable.

**`equals()`, `hashCode()`, `toString()` — Definition:** every class implicitly inherits these from `Object`. The default `equals()` compares references (identity, like `==`); overriding it to compare actual field values is essential for objects used as, e.g., keys in a `HashMap`/`HashSet` (section 3) or compared meaningfully in business logic — and **`hashCode()` must be overridden consistently alongside `equals()`** (equal objects must produce equal hash codes, or hash-based collections will behave incorrectly, failing to find an object that's genuinely `.equals()` to a stored key). `toString()`'s default (`ClassName@hashcode`) is rarely useful; overriding it provides a meaningful string representation for logging/debugging.

**Immutability & `final` — Definition:** `final` on a variable prevents reassignment after initialization (like `const` conceptually, but only for the reference itself — a `final` object's internal fields can still be mutated unless the class itself is designed to be immutable); `final` on a method prevents it from being overridden; `final` on a class prevents it from being subclassed at all (`String` itself is `final`, guaranteeing its immutability contract can never be broken by a subclass).

---

## 3. Collections Framework

**Definition:** the Collections Framework is Java's unified set of interfaces and implementations for storing and manipulating groups of objects — `Collection` (the root interface for `List`/`Set`/`Queue`) and `Map` (a separate hierarchy, since it stores key-value pairs rather than a plain group of elements).

**`List` — Definition:** an ordered collection permitting duplicates, indexed access.
- **`ArrayList`** — backed by a dynamic array (same amortized-O(1)-append behavior as the DSA/JS notes' dynamic array sections) — O(1) index access, O(n) insert/delete at arbitrary positions.
- **`LinkedList`** — backed by a doubly-linked list — O(1) insert/delete given a position/iterator, O(n) index access — also implements `Deque`, usable as a stack/queue.

**`Set` — Definition:** a collection with **no duplicates**.
- **`HashSet`** — backed by a `HashMap` internally, O(1) average add/contains/remove, **no ordering guarantee**.
- **`LinkedHashSet`** — like `HashSet`, but maintains **insertion order** during iteration, at slightly more memory overhead.
- **`TreeSet`** — backed by a red-black tree (same self-balancing-BST concept as the DSA notes' section 18), keeps elements in **sorted order**, O(log n) operations.

**`Map` — Definition:** a key-value store.
- **`HashMap`** — O(1) average get/put, **no ordering guarantee** — the general-purpose default.
- **`LinkedHashMap`** — like `HashMap` but maintains insertion order (or, configurable, access order — usable to implement an LRU cache, DSA notes' section 4).
- **`TreeMap`** — sorted by key, O(log n) operations, backed by a red-black tree.

```java
Map<String, Integer> ages = new HashMap<>();
ages.put("Ada", 30);
ages.getOrDefault("Bob", 0); // 0 — avoids a manual null check
```

**`Queue`/`Deque` — Definition:** `Queue` provides FIFO semantics (`offer`/`poll`); `Deque` (Double-Ended Queue) supports insertion/removal at both ends, usable as either a stack or a queue — `ArrayDeque` is the generally-preferred modern implementation for both use cases, typically outperforming the legacy `Stack` class (which is synchronized, and considered a legacy holdover) and `LinkedList` (worse cache locality than the array-backed `ArrayDeque`).

**Choosing the right collection — Definition:** `ArrayList` for most general-purpose ordered storage (default choice); `LinkedList` specifically when frequent insertion/deletion at arbitrary positions dominates over random access; `HashSet`/`HashMap` when order doesn't matter and O(1) lookup is the priority; `LinkedHashSet`/`LinkedHashMap` when insertion order must be preserved; `TreeSet`/`TreeMap` when sorted order must be maintained continuously.

**Iterators & `Iterable` — Definition:** a class implementing `Iterable` can be used in a `for-each` loop (`for (String s : list)`), which internally uses an `Iterator` (`.hasNext()`/`.next()`) — the same iterator protocol concept covered in the JS/TS notes' section 9. `Iterator` also provides `.remove()`, the **only** safe way to remove elements while iterating (mutating a collection directly during a for-each loop throws `ConcurrentModificationException`).

**`Comparable` vs `Comparator` — Definition:** a class implements `Comparable<T>` (a single `compareTo()` method) to define its own **natural ordering** (used automatically by `TreeSet`/`TreeMap`/`Collections.sort()`); a `Comparator<T>` is a separate, external object defining an *alternative* ordering, passed in where needed — used when a class either has no single natural order, or you need to sort the same objects differently in different contexts.

```java
List<User> users = new ArrayList<>();
users.sort(Comparator.comparing(User::getAge).thenComparing(User::getName));
```

**Immutable collections — Definition:** `List.of(...)`/`Set.of(...)`/`Map.of(...)` (Java 9+) create genuinely **unmodifiable** collections (any mutation attempt throws `UnsupportedOperationException`) directly; `Collections.unmodifiableList(list)` wraps an *existing* mutable list in an unmodifiable view (though the underlying original list can still be mutated directly, which would then be visible through the wrapper — a subtlety worth knowing).

**Generics (Java-specific notes) — Definition:** Java generics (conceptually the same as TypeScript's, JS/TS notes' section 17) use **type erasure** — generic type information exists only at compile time for type-checking purposes and is completely erased from the compiled bytecode (a `List<String>` and `List<Integer>` are, at runtime, both just `List`) — the reason primitives can't be used directly as generic type parameters (they must be boxed, above), and why certain generic operations (`new T()`, `instanceof T`) aren't possible. **Bounded wildcards** (`List<? extends Number>` — accepts a list of Number or any subtype; `List<? super Integer>` — accepts a list of Integer or any supertype) express flexible generic method signatures without type erasure limitations getting in the way.

---

## 4. Exception Handling

**Checked vs unchecked exceptions — Definition:** a **checked** exception (any subclass of `Exception` other than `RuntimeException`) **must** be either caught or explicitly declared in a method's `throws` clause — enforced by the compiler, forcing callers to consciously handle a known failure mode (e.g. `IOException`). An **unchecked** exception (subclasses of `RuntimeException`) requires no such declaration — typically used for programming errors (`NullPointerException`, `IllegalArgumentException`) that shouldn't need to be anticipated/declared at every call site.

```java
public void readFile(String path) throws IOException { // checked — must be declared or caught
  Files.readAllBytes(Paths.get(path));
}
```

**`try`/`catch`/`finally`:**

```java
try {
  riskyOperation();
} catch (IOException e) {
  log.error("Failed", e);
} finally {
  cleanup(); // always runs, even if a return/exception occurs above
}
```

**Try-with-resources & `AutoCloseable` — Definition:** a `try` variant that automatically calls `.close()` on any resource(s) declared in its parentheses once the block exits (normally or via exception) — eliminates the boilerplate/bug-risk of manually closing resources in a `finally` block, and works with any class implementing `AutoCloseable`.

```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement stmt = conn.prepareStatement(sql)) {
  stmt.executeQuery();
} // conn and stmt are automatically closed here, in reverse declaration order
```

**Custom exceptions — Definition:** extending `Exception` (checked) or `RuntimeException` (unchecked) to create domain-specific error types carrying additional context — the same principle as the Node.js/JS notes' custom error classes.

```java
public class UserNotFoundException extends RuntimeException {
  public UserNotFoundException(Long id) { super("User not found: " + id); }
}
```

**Exception hierarchy — Definition:** all exceptions/errors descend from `Throwable`, splitting into `Error` (serious problems an application generally shouldn't try to catch/recover from — `OutOfMemoryError`, `StackOverflowError`) and `Exception` (conditions an application might reasonably want to catch and handle, further split into checked and unchecked as above).

**Best practices** — never catch an exception and silently swallow it (an empty `catch` block hides failures rather than fixing them); use exception chaining (`throw new ServiceException("...", originalException)`, the Java equivalent of the JS/TS notes' `Error.cause`) when wrapping a lower-level exception in a more meaningful higher-level one, preserving the original stack trace/cause for debugging; catch the most specific exception type reasonable, not a blanket `catch (Exception e)`, so unrelated failures aren't accidentally handled by logic meant for a specific, anticipated one.

---

## 5. Java Memory Model & JVM Internals

A major deep-dive topic.

**JVM architecture overview — Definition:** the JVM consists of a **class loader subsystem** (loads, links, and initializes `.class` files), **runtime data areas** (memory regions used during execution — heap, stack, method area, below), and an **execution engine** (the interpreter and JIT compilers that actually run bytecode).

**Heap vs Stack — Definition:**
- **Heap** — where all objects (and arrays) live, shared across all threads — managed by the garbage collector (below).
- **Stack** — each thread has its own private stack, holding stack frames (one per active method call, tracking local variables and partial results) — primitives and object *references* live here (the objects themselves are still on the heap); a stack frame is popped when its method returns, and exceeding stack depth (e.g. unbounded recursion, DSA notes' section 11) throws `StackOverflowError`.

**Method Area / Metaspace — Definition:** stores per-class structural information — the class's bytecode, constant pool, static variables, method data — shared across all threads. Historically part of the heap as "PermGen" (Permanent Generation, notoriously prone to `OutOfMemoryError: PermGen space`); since Java 8, this was replaced by **Metaspace**, allocated from native (off-heap) memory instead, which can grow dynamically and largely eliminated the classic PermGen-exhaustion problem.

**Garbage collection — Definition:** automatic reclamation of heap memory occupied by objects no longer reachable from any live reference — conceptually the same generational/mark-and-sweep principles already covered in the Node.js notes' V8 GC section, applied to the JVM.

**Generational GC — Definition:** the heap is divided into a **Young Generation** (further split into Eden and two Survivor spaces, where new objects are allocated, collected frequently and cheaply via "Minor GC" — most objects die young, the same generational hypothesis from the Node.js notes) and an **Old Generation** (holding objects that survived enough Minor GC cycles, collected less often via a more expensive "Major/Full GC").

**GC algorithms (brief) — Definition:** **Serial** (single-threaded, simplest, suited to small heaps/single-core environments); **Parallel** (multi-threaded collection, optimizes for throughput, historically Java's default); **G1 (Garbage-First)** (divides the heap into many regions, prioritizes collecting regions with the most garbage first, balances throughput and pause times — the default since Java 9); **ZGC** (a low-latency collector targeting sub-millisecond pause times even on very large heaps, at some throughput cost — for latency-critical applications where G1's pause times are still too long).

**JIT compilation (interpreter vs C1/C2) — Definition:** the JVM initially **interprets** bytecode (slower per-instruction, but zero compilation delay); code paths executed frequently enough ("hot" code) are compiled to native machine code by the JIT compiler for much faster subsequent execution — HotSpot (the standard JVM) uses a **tiered compilation** strategy: **C1** (the "client" compiler — compiles quickly, with lighter optimization, good for fast startup) handles moderately-hot code first, and **C2** (the "server" compiler — slower to compile, but applies much more aggressive optimization) kicks in for code that's proven to be genuinely hot, giving the best of both fast startup and eventual peak throughput.

**Class loading process — Definition:** **Loading** (reading the `.class` file's bytecode into memory), **Linking** (**Verification** — bytecode is valid/safe; **Preparation** — static fields allocated with default values; **Resolution** — symbolic references resolved to actual references), **Initialization** (static initializers and static field initial values actually run) — occurring lazily, the first time a class is actually used, not necessarily at JVM startup.

**The Java Memory Model (JMM) — Definition:** the specification defining how threads interact through memory — specifically, when a write by one thread is guaranteed to be **visible** to another thread. The **happens-before** relationship is the JMM's core concept: if action A happens-before action B, then A's effects (including its writes to memory) are guaranteed visible to B — established by specific constructs like `synchronized` blocks (unlocking happens-before a subsequent lock by another thread), `volatile` writes (happens-before a subsequent read of that same variable), and thread start/join — without an established happens-before relationship, one thread's writes may never become visible to another thread at all (each CPU core can cache values locally), which is the root cause of many multithreading bugs (section 6).

---

## 6. Multithreading & Concurrency

**Threads — Definition:** a `Thread` represents an independent path of execution within a process, sharing the same heap memory but with its own private stack — created either by extending `Thread` and overriding `run()`, or (preferred) by implementing `Runnable`/`Callable` and passing it to a `Thread` or, better, an `ExecutorService` (below).

**`Runnable` vs `Callable` — Definition:** `Runnable`'s `run()` method takes no arguments and returns nothing, cannot throw a checked exception; `Callable<V>`'s `call()` method returns a value of type `V` and can throw a checked exception — `Callable` is the modern choice whenever a task needs to produce a result or report a checked failure, used together with `ExecutorService.submit()` and `Future`.

**The `synchronized` keyword — Definition:** ensures only one thread at a time can execute a given block/method on a given object (or class, for `static synchronized`) — implemented via an intrinsic lock ("monitor") associated with the object — the most basic Java mechanism for enforcing mutual exclusion and establishing a happens-before relationship (section 5) between threads.

```java
public synchronized void increment() { count++; } // only one thread can run this method at a time (per instance)
```

**`ReentrantLock` — Definition:** an explicit lock class (from `java.util.concurrent.locks`) providing capabilities `synchronized` lacks — the ability to attempt a lock with a **timeout**, to be **interruptible** while waiting, and to support fairness policies — at the cost of requiring manual lock/unlock (always in a `try`/`finally`) rather than `synchronized`'s automatic scope-based release.

```java
Lock lock = new ReentrantLock();
lock.lock();
try { /* critical section */ }
finally { lock.unlock(); } // MUST be in finally, or a mid-block exception leaves the lock held forever
```

**`volatile` — Definition:** guarantees that reads/writes to a variable are always visible across threads immediately (bypassing per-thread/per-core caching), establishing a happens-before relationship for that specific variable — but, critically, does **not** provide atomicity for compound operations (`count++` on a `volatile int` is still a read-modify-write race, since it's three separate operations, not one) — `volatile` is correct for simple flags/status variables read by many threads and written by one, not for counters or anything requiring atomic read-modify-write.

**Thread safety & race conditions — Definition:** code is **thread-safe** if it behaves correctly when accessed concurrently by multiple threads without external synchronization; a **race condition** occurs when the correctness of a result depends on the unpredictable timing/interleaving of concurrent operations — the underlying problem `synchronized`/locks/`volatile`/concurrent collections all exist to prevent.

**Deadlocks, livelocks, starvation — Definition:** a **deadlock** occurs when two or more threads each hold a lock the other needs, waiting forever (the same concept as the SQL notes' section 11, applied to application-level locks instead of database row locks); a **livelock** occurs when threads keep actively responding to each other in a way that prevents any of them from making actual progress (unlike deadlock's total standstill); **starvation** occurs when a thread is perpetually denied the resources/CPU time it needs to proceed, often due to other threads being unfairly prioritized.

**The Executor framework — Definition:** the standard, high-level abstraction for managing thread execution, replacing manual `Thread` creation/management. An `ExecutorService` manages a **thread pool** — a fixed or dynamically-sized set of reusable worker threads that execute submitted tasks, avoiding the overhead of creating a brand-new OS thread per task.

```java
ExecutorService executor = Executors.newFixedThreadPool(10);
executor.submit(() -> processTask());
executor.shutdown(); // initiates orderly shutdown, letting submitted tasks complete
```

**`Future` & `CompletableFuture` — Definition:** a `Future<V>` represents the eventual result of an asynchronous computation submitted to an `ExecutorService` — `.get()` blocks until the result is available (or throws if the task failed). `CompletableFuture<V>` (Java 8+) is a much richer, composable async abstraction — conceptually parallel to a JavaScript Promise (JS/TS notes' section 7) — supporting chaining (`.thenApply`), combining multiple futures (`.thenCombine`), and non-blocking callback-style composition rather than `Future`'s blocking `.get()`.

```java
CompletableFuture.supplyAsync(() -> fetchUser(id))
  .thenApply(user -> user.getName())
  .thenAccept(System.out::println)
  .exceptionally(err -> { log.error("Failed", err); return null; });
```

**Concurrent collections — Definition:** thread-safe collection implementations designed for concurrent access without requiring external synchronization. `ConcurrentHashMap` — a highly-concurrent map using fine-grained internal locking (or lock-free techniques), dramatically outperforming a plain `HashMap` wrapped in `Collections.synchronizedMap()` (which locks the *entire* map for every operation) under contention. `CopyOnWriteArrayList` — every mutation creates a new underlying copy of the array, making **reads** entirely lock-free/wait-free — ideal for read-heavy, rarely-mutated collections (e.g. a list of event listeners), poor for write-heavy use cases given the full-copy cost per write.

**`java.util.concurrent` utilities — Definition:** `CountDownLatch` (lets one or more threads wait until a set of operations being performed by other threads completes — a one-time-use gate); `Semaphore` (limits the number of threads that can access a resource concurrently, via a permit count); `CyclicBarrier` (lets a fixed number of threads wait for each other to reach a common barrier point, then proceed together — reusable, unlike `CountDownLatch`).

**Virtual threads (Project Loom, brief) — Definition:** introduced as a stable feature in Java 21 — lightweight threads managed by the JVM itself (not directly mapped 1:1 to OS threads, the way traditional Java threads are), allowing an application to spawn potentially **millions** of concurrent virtual threads cheaply — dramatically simplifies writing high-concurrency, blocking-style code (e.g. a thread-per-request server model) that would previously have required either exhausting OS thread limits or adopting more complex, harder-to-read async/reactive programming models to achieve the same scale.

---

## 7. Modern Java (8+) Features

**Lambda expressions — Definition:** a concise, inline syntax for representing an implementation of a **functional interface** (an interface with exactly one abstract method) — Java's equivalent of a JS/TS arrow function, though strictly tied to satisfying a specific single-method interface type rather than being a fully first-class, freely-typed value the way a JS function is.

```java
Runnable r = () -> System.out.println("running");
Comparator<String> byLength = (a, b) -> a.length() - b.length();
```

**Functional interfaces — Definition:** `java.util.function` provides a standard set: `Function<T, R>` (takes T, returns R), `Predicate<T>` (takes T, returns boolean), `Supplier<T>` (takes nothing, returns T), `Consumer<T>` (takes T, returns nothing) — the common building blocks used throughout the Streams API (below) and functional-style Java code generally.

**Method references — Definition:** a shorthand syntax (`ClassName::methodName`) for a lambda that does nothing but call an existing method — `String::toUpperCase` is equivalent to `s -> s.toUpperCase()`.

**The Streams API — Definition:** a functional-style API for processing sequences of elements through a pipeline of **intermediate operations** (lazy, return a new stream, don't execute until a terminal operation is invoked — `map`, `filter`, `sorted`) followed by exactly one **terminal operation** (triggers actual execution and produces a result — `collect`, `reduce`, `forEach`, `count`) — conceptually the same declarative-transformation philosophy as the JS/TS notes' array methods (section 12), but with lazy evaluation and built-in easy parallelization.

```java
List<String> names = users.stream()
  .filter(u -> u.getAge() > 18)
  .map(User::getName)
  .sorted()
  .collect(Collectors.toList());
```

**Collectors — Definition:** the `Collectors` utility class provides standard terminal-operation strategies for a stream — `toList()`/`toSet()`/`toMap()`, `groupingBy()` (the Java equivalent of SQL's `GROUP BY`, DSA/SQL notes), `joining()` (concatenate strings with a delimiter), `counting()`, `summingInt()`.

```java
Map<String, List<User>> byDepartment = users.stream()
  .collect(Collectors.groupingBy(User::getDepartment));
```

**Parallel streams — Definition:** `.parallelStream()` (or `.stream().parallel()`) automatically splits the stream's work across multiple threads (backed by the common `ForkJoinPool`) — can provide real speedups for large, CPU-bound, easily-partitionable workloads, but has real overhead (splitting/merging work, thread coordination) that can make it *slower* than a sequential stream for small datasets or I/O-bound operations — not a default choice; benchmark before adopting.

**`Optional` — Definition:** a container object that may or may not hold a non-null value, used as a return type to make the **possibility of absence explicit** in a method's signature — replacing the historical Java convention of returning `null` (which callers could easily forget to check, causing a `NullPointerException`).

```java
Optional<User> findUser(Long id) { /* ... */ }

findUser(1L)
  .map(User::getName)
  .orElse("Unknown"); // no explicit null check needed anywhere in this chain
```
`Optional` is intended specifically as a **return type** (a signal to callers), not as a field type or method parameter type, which are considered misuses of the pattern.

**Records (Java 14+) — Definition:** a concise syntax for declaring an immutable data-carrier class — automatically generates a constructor, accessor methods, `equals()`, `hashCode()`, and `toString()` from the declared components, eliminating the boilerplate a traditional immutable POJO required by hand (Lombok's `@Value`/`@Data` annotations covered similar ground before records existed natively in the language).

```java
public record Point(int x, int y) {} // constructor, x(), y(), equals(), hashCode(), toString() all generated
```

**Sealed classes (Java 17+) — Definition:** restricts which classes/interfaces may extend/implement a given type, explicitly listed via `permits` — combined with pattern matching (below), enables the compiler to verify **exhaustiveness** when handling every possible subtype — the same fundamental idea as the JS/TS notes' discriminated-union exhaustiveness checking (section 16), applied to a real class hierarchy instead of a literal-tagged union.

```java
public sealed interface Shape permits Circle, Rectangle {}
public record Circle(double radius) implements Shape {}
public record Rectangle(double width, double height) implements Shape {}
```

**Pattern matching — Definition:** `instanceof` pattern matching (Java 16+) combines a type check and cast into one expression, eliminating a separate manual cast; **switch expressions** with pattern matching (Java 21+) let a `switch` branch directly on a sealed type's variants, exhaustiveness-checked by the compiler.

```java
if (shape instanceof Circle c) { // combined check + cast into variable `c`
  System.out.println(c.radius());
}

double area = switch (shape) {
  case Circle c -> Math.PI * c.radius() * c.radius();
  case Rectangle r -> r.width() * r.height();
  // compiler enforces every sealed subtype is handled — no default needed, and adding a new
  // permitted subtype without updating this switch is a compile error
};
```

**Text blocks — Definition:** `"""..."""` multi-line string literals (Java 15+), avoiding manual `\n` concatenation for embedded multi-line content like SQL queries or JSON — the Java equivalent of JS/TS template literals for multi-line text.

---

## 8. Spring Framework Fundamentals

**Definition:** Spring is a comprehensive framework for building Java applications, built around **Inversion of Control (IoC)** — rather than application code creating and wiring together its own dependencies, Spring's **IoC container** creates, configures, and injects them, based on configuration/annotations.

**Inversion of Control & Dependency Injection — Definition:** the same DI concept already covered in the Angular/Node.js notes (sections 5), applied to Java — a class declares what it needs (via constructor parameters or annotated fields), and Spring's container supplies (**injects**) those dependencies at runtime, rather than the class instantiating them itself with `new`.

**The `ApplicationContext` — Definition:** Spring's IoC container implementation — holds and manages all application **beans** (Spring-managed objects), resolving and injecting dependencies between them, and providing additional framework services (event publishing, resource loading, internationalization) on top.

**Bean lifecycle — Definition:** a bean's life within the container follows a defined sequence: instantiation → dependency injection → (any `@PostConstruct`-annotated initialization method runs) → the bean is ready for use → (on container shutdown) any `@PreDestroy`-annotated cleanup method runs — Spring manages this entire lifecycle automatically once a class is registered as a bean.

**Bean scopes — Definition:** `singleton` (default — exactly one shared instance for the entire `ApplicationContext`, the Spring equivalent of the Node.js notes' `providedIn: 'root'`); `prototype` (a new instance created every time the bean is requested/injected); `request`/`session` (web-specific — one instance per HTTP request/session, only meaningful in a web application context).

**Configuration styles — Definition:** **XML configuration** (the original, verbose approach, largely legacy today); **Java config** (`@Configuration` classes with `@Bean`-annotated factory methods, giving full programmatic control over bean creation); **annotation-based/component scanning** (`@Component` and its stereotypes, below, on the class itself, discovered automatically by Spring scanning specified packages) — modern Spring Boot applications predominantly use annotation-based configuration with Java config for anything needing explicit setup logic.

```java
@Configuration
public class AppConfig {
  @Bean
  public RestTemplate restTemplate() { return new RestTemplate(); }
}
```

**`@Component` vs `@Service` vs `@Repository` vs `@Controller` — Definition:** all four are **stereotype annotations**, functionally identical to plain `@Component` in terms of registering a class as a Spring-managed bean — the more specific ones exist purely for **semantic clarity** (and, for `@Repository` specifically, one real behavioral addition: automatic translation of database-specific exceptions into Spring's unified `DataAccessException` hierarchy). `@Service` marks business-logic classes, `@Repository` marks data-access classes, `@Controller`/`@RestController` mark web-layer request handlers (section 10) — the same layered architecture already covered in the Node.js notes' section 14 (routes/controllers → services → repositories/models).

**Constructor injection vs field injection — Definition:**

```java
// ✅ constructor injection — RECOMMENDED
@Service
public class OrderService {
  private final UserRepository userRepository;
  public OrderService(UserRepository userRepository) { this.userRepository = userRepository; }
}

// ❌ field injection — discouraged
@Service
public class OrderService {
  @Autowired
  private UserRepository userRepository;
}
```
**Constructor injection — Advantages:** dependencies can be declared `final` (immutable once set), the class is impossible to construct in an invalid/half-initialized state, and it makes the class trivially testable without Spring at all (just call the constructor directly with mock dependencies in a plain unit test) — the modern, Spring-team-recommended default. **Field injection — Disadvantages:** requires Spring's container (or reflection) to set the field at all, since there's no way to set a `private` field via a constructor call from outside the framework — makes plain (non-Spring-context) unit testing more awkward, and allows a bean to exist in a partially-uninitialized state before injection completes.

---

## 9. Spring Boot

**Definition:** Spring Boot is an opinionated, convention-over-configuration layer on top of the core Spring Framework, designed to minimize the manual configuration traditionally required to get a Spring application running — providing auto-configuration, embedded servers, and curated dependency "starters."

**What Spring Boot adds over plain Spring** — plain Spring requires explicit configuration of nearly every component (a `DataSource` bean, a `DispatcherServlet`, a web server); Spring Boot infers sensible defaults automatically based on what's present on the classpath, letting a working application start from a minimal amount of code/configuration.

**Auto-configuration — Definition:** Spring Boot inspects the classpath and existing bean definitions at startup, and automatically configures beans it infers you likely need — e.g. if Spring Boot detects the `spring-boot-starter-data-jpa` dependency and a database driver on the classpath, it automatically configures a `DataSource`, an `EntityManagerFactory`, and transaction management, without any of that being manually declared.

**Starters — Definition:** curated dependency bundles (`spring-boot-starter-web`, `spring-boot-starter-data-jpa`, `spring-boot-starter-security`) that pull in a coherent, version-compatible set of libraries for a given concern, so you declare one dependency instead of manually assembling and version-matching a dozen individual libraries.

**`application.properties`/`application.yml` — Definition:** the default externalized configuration file(s) for a Spring Boot application, letting settings (database URL, server port, logging levels) be configured declaratively rather than hardcoded.

```yaml
# application.yml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
```

**Profiles — Definition:** named configuration variants (`application-dev.yml`, `application-prod.yml`) activated via a `spring.profiles.active` setting — the same environment-configuration principle covered in every other notes file in this workspace (Angular, Node.js, AWS), applied to Spring's own config system.

**The embedded server model — Definition:** unlike traditional Java web applications (deployed as a WAR file to an external servlet container like Tomcat), Spring Boot **embeds** a servlet container (Tomcat by default, or Jetty/Undertow) directly inside the application's own executable JAR — the application *is* the server, started with a plain `java -jar app.jar`, without needing a separately-installed/managed application server.

**Spring Boot Actuator — Definition:** a starter providing production-ready operational endpoints out of the box — `/actuator/health` (the same health-check concept as the Node.js/AWS/Kubernetes notes), `/actuator/metrics`, `/actuator/info` — exposing operational visibility into a running application with minimal setup.

**Building an executable JAR** — the Spring Boot Maven/Gradle plugin packages the application, all its dependencies, and the embedded server into one single, self-contained "fat JAR," runnable anywhere a JVM is available with no separate dependency installation step — the same self-containment philosophy as a multi-stage Docker build (Docker/K8s notes' section 2), just at the JAR-packaging level instead.

---

## 10. Building REST APIs with Spring MVC

**`@RestController` & `@RequestMapping` — Definition:** `@RestController` (a combination of `@Controller` + `@ResponseBody`) marks a class whose methods' return values are automatically serialized (typically to JSON, via Jackson) directly into the HTTP response body, rather than being resolved as a view template name — the standard annotation for a JSON REST API controller, as opposed to a traditional server-rendered-HTML `@Controller`.

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
  private final UserService userService;
  public UserController(UserService userService) { this.userService = userService; }

  @GetMapping("/{id}")
  public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
    return ResponseEntity.ok(userService.findById(id));
  }

  @PostMapping
  public ResponseEntity<UserDto> createUser(@Valid @RequestBody CreateUserRequest request) {
    UserDto created = userService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(created);
  }
}
```

**Request mapping annotations — Definition:** `@GetMapping`/`@PostMapping`/`@PutMapping`/`@PatchMapping`/`@DeleteMapping` are shorthand specializations of the more general `@RequestMapping(method = ...)`, mapping a method to a specific HTTP verb + path — the same REST resource-routing concept as the Node.js/Express notes' section 4.

**`@PathVariable`/`@RequestParam`/`@RequestBody` — Definition:** `@PathVariable` extracts a value from the URL path (`/users/{id}`); `@RequestParam` extracts a query-string parameter (`?verbose=true`); `@RequestBody` deserializes the entire request body (typically JSON) into a Java object.

**`ResponseEntity` — Definition:** a wrapper letting a controller method control the full HTTP response — status code, headers, and body — explicitly, rather than only returning the body and letting Spring infer a default `200 OK` status.

**Request validation (`@Valid`, Bean Validation) — Definition:** annotating a `@RequestBody` parameter with `@Valid` tells Spring to automatically validate it against constraint annotations declared on the target class's fields (`@NotNull`, `@Email`, `@Min`/`@Max`, from the Jakarta Bean Validation spec) — invalid requests are automatically rejected with a `400 Bad Request` before the controller method body even runs, the same declarative-validation principle as the Node.js notes' Zod validation section, but framework-integrated rather than a manual check.

```java
public record CreateUserRequest(
  @NotBlank String name,
  @Email String email,
  @Min(18) int age
) {}
```

**Exception handling (`@ExceptionHandler`, `@ControllerAdvice`) — Definition:** `@ExceptionHandler` on a method (within a controller, or a globally-applied `@ControllerAdvice`-annotated class) intercepts a specific exception type thrown anywhere in the request-handling flow and converts it into an appropriate HTTP response — the Spring equivalent of the Node.js notes' centralized Express error-handling middleware.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
  @ExceptionHandler(UserNotFoundException.class)
  public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException ex) {
    return ResponseEntity.status(HttpStatus.NOT_FOUND).body(new ErrorResponse(ex.getMessage()));
  }
}
```

**Content negotiation** — Spring MVC can serve different representations (JSON, XML) of the same resource based on the request's `Accept` header, automatically selecting the appropriate `HttpMessageConverter`.

**API versioning strategies (recap)** — see the AWS/Node.js notes; the same URL-path, header, or query-parameter versioning approaches apply identically in a Spring-based API.

---

## 11. Data Access — JDBC, JPA & Hibernate

**JDBC fundamentals — Definition:** JDBC (Java Database Connectivity) is the low-level, standard Java API for connecting to and executing SQL against a relational database — `Connection`, `PreparedStatement` (parameterized queries, preventing SQL injection exactly as covered in the Node.js/SQL notes), `ResultSet` — the foundational layer that every higher-level Java data-access tool (JPA/Hibernate, Spring's `JdbcTemplate`) is ultimately built on top of.

**The ORM problem JPA/Hibernate solves — Definition:** an **Object-Relational Mapper (ORM)** bridges the mismatch between Java's object-oriented model (objects, inheritance, references) and a relational database's tabular model (rows, foreign keys) — **JPA (Jakarta Persistence API)** is the standard *specification* for this mapping in Java; **Hibernate** is the most widely-used *implementation* of that specification (conceptually parallel to the Node.js notes' Prisma/Mongoose, but for the Java ecosystem specifically, and JPA-standardized rather than proprietary).

**Entities & `@Entity` — Definition:** a plain Java class annotated `@Entity` becomes a persistent, database-mapped object — its fields map to table columns, and JPA/Hibernate handles translating between Java object state and database rows automatically.

```java
@Entity
@Table(name = "users")
public class User {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, unique = true)
  private String email;
}
```

**Relationships — Definition:** `@OneToMany`/`@ManyToOne` (a user has many orders; an order belongs to one user), `@ManyToMany` (e.g. posts and tags) — annotations expressing the same relational-modeling concepts already covered in the SQL notes' section 7, mapped onto Java object references/collections.

```java
@Entity
public class Order {
  @ManyToOne
  @JoinColumn(name = "user_id")
  private User user;
}
```

**Spring Data JPA — Definition:** a Spring project that dramatically reduces JPA/Hibernate boilerplate by letting you declare a **repository interface**, with Spring automatically generating the implementation at runtime — no hand-written DAO/implementation class needed for standard CRUD.

```java
public interface UserRepository extends JpaRepository<User, Long> {
  Optional<User> findByEmail(String email);              // derived query method
  List<User> findByAgeGreaterThanAndActiveTrue(int age);  // derived from the method name itself

  @Query("SELECT u FROM User u WHERE u.email LIKE %:domain%") // explicit JPQL when derivation isn't enough
  List<User> findByEmailDomain(@Param("domain") String domain);
}
```

**Derived query methods — Definition:** Spring Data JPA parses a repository method's **name** itself (`findByAgeGreaterThanAndActiveTrue`) to automatically generate the corresponding query — eliminates writing SQL/JPQL for straightforward queries, at the cost of method names becoming unwieldy for genuinely complex conditions (where an explicit `@Query` becomes clearer).

**Lazy vs eager loading — Definition:** **lazy** loading (the default for `@OneToMany`/`@ManyToMany`) defers fetching a related entity/collection until it's actually accessed in code, rather than loading it immediately alongside its parent; **eager** loading (the default for `@ManyToOne`/`@OneToOne`) fetches the related entity immediately, in the same query (or an immediately-following one). Lazy loading accessed *outside* an active transaction/session throws `LazyInitializationException` — a very common Hibernate pitfall.

**The N+1 query problem (recap)** — see the Node.js/MongoDB/SQL notes; lazy-loading a collection inside a loop over a list of parent entities triggers exactly this pattern in JPA/Hibernate — solved with `JOIN FETCH` in a JPQL query, or Spring Data JPA's `@EntityGraph`, to fetch the related data in one batched query upfront.

**Transactions (`@Transactional`) — Definition:** annotating a service method `@Transactional` wraps its execution in a database transaction (the same ACID transaction concept as the SQL notes' section 10) — Spring automatically begins the transaction on method entry and commits on successful completion, or rolls back on an (unchecked, by default) exception — eliminating manual `begin`/`commit`/`rollback` boilerplate.

**Database migrations (Flyway/Liquibase) — Definition:** version-controlled, incremental scripts that evolve the database schema over time, applied automatically on application startup — the same concept as the SQL/AWS notes' migration sections, integrated into the Spring Boot startup lifecycle so schema and application code version together.

---

## 12. Spring Security

**Authentication vs authorization (recap)** — same concepts as the Node.js/AWS/SQL notes; who you are vs what you're allowed to do.

**The security filter chain — Definition:** Spring Security intercepts every incoming HTTP request through a chain of servlet **filters**, each handling one security concern (authentication, CSRF checking, authorization) before the request ever reaches a controller — configured declaratively via a `SecurityFilterChain` bean.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
  http
    .authorizeHttpRequests(auth -> auth
      .requestMatchers("/api/public/**").permitAll()
      .anyRequest().authenticated())
    .csrf(csrf -> csrf.disable()) // commonly disabled for stateless, token-based APIs
    .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
  return http.build();
}
```

**Form-based auth vs JWT-based auth — Definition:** **form-based** authentication (Spring Security's traditional default) issues a server-side session and a session cookie after login, requiring `sessionCreationPolicy` to allow session state; **JWT-based** authentication (far more common for modern REST APIs, especially ones consumed by a separate SPA frontend) validates a signed token on every request (via a custom filter added to the chain) with `SessionCreationPolicy.STATELESS` — the same session-vs-token tradeoff already covered in the Node.js notes' section 8.

**`UserDetailsService` — Definition:** the interface Spring Security calls to load a user's credentials/authorities during authentication — implemented to fetch user data from wherever it actually lives (typically via a `UserRepository`, section 11), decoupling Spring Security's authentication mechanics from the specific data source.

**Password encoding (`BCryptPasswordEncoder`) — Definition:** Spring Security's standard password-hashing implementation, using the bcrypt algorithm — the same deliberately-slow, salted hashing concept already covered in the Node.js notes' section 8.

**Method-level security (`@PreAuthorize`) — Definition:** annotating a service/controller method with `@PreAuthorize("hasRole('ADMIN')")` enforces an authorization check *before* the method executes, expressed via Spring Expression Language (SpEL) — enables fine-grained, business-logic-level authorization rather than only coarse, URL-pattern-based rules in the filter chain.

**CORS & CSRF configuration (recap)** — same concepts as the Node.js/AWS notes; configured via the `SecurityFilterChain` above, with CSRF protection typically disabled for stateless, token-authenticated APIs (which aren't vulnerable to the classic cookie-based CSRF attack in the same way session-cookie-authenticated apps are) but left enabled for traditional session-based web applications.

**OAuth2 / OpenID Connect with Spring Security — Definition:** Spring Security provides first-class support for both acting as an OAuth2 **client** (delegating login to Google/GitHub/etc., the same social-login concept as the Node.js notes' section 8) and as an OAuth2 **resource server** (validating incoming bearer tokens issued by an external identity provider), via dedicated starters (`spring-boot-starter-oauth2-client`/`-resource-server`) that handle the protocol's considerable complexity declaratively.

---

## 13. Testing in Java

**JUnit 5 fundamentals — Definition:** the standard Java unit-testing framework — `@Test` marks a test method, lifecycle annotations (`@BeforeEach`/`@AfterEach`/`@BeforeAll`/`@AfterAll`) control setup/teardown timing, and assertions (`assertEquals`, `assertTrue`, `assertThrows`) verify expected behavior.

```java
class CalculatorTest {
  private Calculator calculator;

  @BeforeEach
  void setUp() { calculator = new Calculator(); }

  @Test
  void addsNumbersCorrectly() {
    assertEquals(5, calculator.add(2, 3));
  }
}
```

**Mockito — Definition:** a mocking framework for creating test doubles of a class's dependencies (the same concept as the Node.js/React notes' mocking sections), letting a unit test isolate the class under test from its real collaborators.

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
  @Mock private UserRepository userRepository;
  @InjectMocks private OrderService orderService; // constructor-injects the mocks above

  @Test
  void throwsWhenUserNotFound() {
    when(userRepository.findById(1L)).thenReturn(Optional.empty());
    assertThrows(UserNotFoundException.class, () -> orderService.createOrder(1L));
  }
}
```

**`@SpringBootTest` vs slice tests — Definition:** `@SpringBootTest` boots the **entire** Spring `ApplicationContext` for a test — most realistic, but slowest, since every bean in the application gets wired up. **Slice tests** boot only the specific layer relevant to what's being tested: `@WebMvcTest` (only the web layer — controllers, with the service layer mocked) tests controller behavior/routing/serialization in isolation; `@DataJpaTest` (only the JPA/repository layer, against an in-memory or configured test database) tests repository queries in isolation — slice tests are significantly faster than a full `@SpringBootTest` and should be preferred whenever a test genuinely only needs to exercise one layer.

**Testcontainers — Definition:** a library for spinning up real, temporary infrastructure (a real PostgreSQL/MySQL/Kafka instance) in a Docker container for the duration of a test suite — same concept as the MongoDB notes' `mongodb-memory-server` (section 13) and the SQL notes' testing section, giving higher-fidelity integration tests than mocking the database entirely, without needing a permanently-running shared test database.

```java
@Testcontainers
class UserRepositoryIntegrationTest {
  @Container
  static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
  // ...
}
```

**Integration testing REST APIs** — `@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)` combined with `TestRestTemplate`/`WebTestClient` lets a test issue real HTTP requests against a running (test) instance of the application, exercising the full request pipeline end-to-end — the Spring equivalent of the Node.js notes' `supertest`.

**Test-Driven Development (brief) — Definition:** a development practice of writing a (initially failing) test *before* writing the implementation code that makes it pass, then refactoring — a discipline/workflow choice applicable across any of the languages/frameworks in this workspace, not Java-specific, but frequently discussed alongside Java's traditionally strong unit-testing culture and tooling.

---

## 14. Build Tools & Dependency Management

**Maven — Definition:** the long-standing, XML-configuration-based build and dependency management tool for Java, built around a `pom.xml` (Project Object Model) file declaring dependencies, plugins, and build configuration.

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
</dependencies>
```

**Dependency scopes — Definition:** `compile` (default — available everywhere, bundled into the final artifact), `provided` (available at compile time, but expected to be supplied by the runtime environment, not bundled — e.g. the servlet API when deploying to an external container), `test` (only available during test compilation/execution, e.g. JUnit/Mockito), `runtime` (needed at runtime but not compile time, e.g. a JDBC driver).

**The build lifecycle — Definition:** Maven defines a fixed sequence of phases (`validate` → `compile` → `test` → `package` → `verify` → `install` → `deploy`) — running a given phase automatically runs every phase before it in sequence (`mvn package` implies `compile` and `test` have already run).

**Gradle — Definition:** a newer build tool using a Groovy or Kotlin DSL (`build.gradle`/`build.gradle.kts`) instead of Maven's XML, built around a more flexible, programmable task-graph model rather than Maven's fixed lifecycle phases.

```kotlin
// build.gradle.kts
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**Gradle vs Maven tradeoffs — Definition:** **Gradle — Advantages:** generally faster builds (incremental compilation, build caching, parallel task execution are more sophisticated/aggressive than Maven's); a genuine programming language (Groovy/Kotlin) for build logic, more flexible than XML for complex/custom build requirements. **Maven — Advantages:** simpler, more rigid, more universally standardized structure — often easier to reason about/onboard onto for straightforward projects specifically *because* it's less flexible/configurable than Gradle; the more common default in many enterprise Java shops historically, meaning wider tooling/documentation familiarity.

**Dependency management & version conflicts — Definition:** when multiple dependencies transitively pull in different versions of the same underlying library, both tools apply a conflict-resolution strategy (Maven: "nearest wins" by declaration depth; Gradle: highest version wins, by default) — dependency version conflicts causing subtle runtime `NoSuchMethodError`s are a genuinely common Java-ecosystem pain point, addressed by explicitly pinning a version (Maven's `<dependencyManagement>`, Gradle's constraints) when the automatic resolution picks the wrong one.

**Multi-module projects — Definition:** both tools support splitting a large application into multiple, independently-buildable sub-modules (e.g. `api`, `service`, `common`) sharing one overall build configuration and version — the same monorepo/multi-package rationale already covered in the Angular/React notes' architecture sections (Nx), applied to Java's own build tooling.

---

## 15. Design Patterns in Java

**Singleton — Definition:** ensures a class has exactly one instance — in modern Spring applications, this is achieved almost for free by a bean's default `singleton` scope (section 8) rather than needing the classic hand-rolled private-constructor-plus-static-instance boilerplate the pattern traditionally requires.

**Factory & Abstract Factory — Definition:** a **Factory** encapsulates object creation logic behind a method, so callers don't need to know which concrete class to instantiate; an **Abstract Factory** provides an interface for creating *families* of related objects without specifying their concrete classes — the same conceptual pattern already covered in the JS/TS notes' section 13, expressed with Java's stronger interface/abstract-class system.

**Builder — Definition:** constructs a complex object step by step via a fluent, chainable API, rather than a single large constructor with many parameters (especially useful when many parameters are optional) — Lombok's `@Builder` annotation, or records' compact syntax (section 7) for simpler immutable cases, largely automate this pattern's boilerplate today.

```java
User user = User.builder()
  .name("Ada")
  .email("ada@example.com")
  .build();
```

**Strategy — Definition:** encapsulates interchangeable algorithms behind a common interface, selected/injected at runtime — a `Comparator` (section 3) is a lightweight, everyday instance of this pattern; more elaborately, a `PaymentStrategy` interface with `CreditCardPayment`/`PayPalPayment` implementations, injected via Spring DI (section 8), is a common real-world Spring application of it.

**Observer — Definition:** an object maintains and notifies a list of dependents on state change — Spring's own `ApplicationEventPublisher`/`@EventListener` mechanism is a built-in framework implementation of this pattern, letting decoupled parts of a Spring application react to application events without direct references to each other.

**Decorator — Definition:** dynamically adds behavior to an object by wrapping it while preserving the same interface — Spring's AOP (Aspect-Oriented Programming, e.g. `@Transactional`, `@Cacheable`) is effectively a framework-managed, declarative application of this pattern: Spring wraps your bean in a dynamic proxy adding the annotated cross-cutting behavior around your actual method calls.

**Proxy (dynamic proxies, brief) — Definition:** Java supports creating **dynamic proxies** at runtime (via `java.lang.reflect.Proxy`, for interface-based proxying, or CGLIB-based subclass proxying for concrete classes) that intercept method calls to a target object — the actual underlying mechanism Spring uses to implement `@Transactional`, `@Cacheable`, `@Async`, and Spring Security's method-level security (section 12) — understanding that these annotations work via a wrapping proxy explains some of their common gotchas (e.g. `@Transactional` has no effect when a method calls another `@Transactional` method on `this` directly, since that internal call bypasses the proxy entirely).

**Dependency Injection as a pattern (recap)** — see section 8; DI itself is a design pattern (a specific application of the broader Inversion of Control principle), which Spring implements as its central, framework-level organizing mechanism rather than something applied ad hoc.

---

## 16. Microservices with Spring

**Spring Cloud overview — Definition:** a collection of Spring projects providing the common infrastructure patterns needed for building microservices (already covered conceptually in the System Design notes' section 9) with Spring-native tooling — service discovery, config management, resilience, API gateways.

**Service discovery (Eureka) — Definition:** Netflix Eureka (integrated via Spring Cloud) provides the same service-registry function described in the System Design notes' section 9 — services register themselves on startup, and other services look up a target's current network location by name rather than a hardcoded address, accommodating dynamically-scaled/rescheduled instances.

**API Gateway (Spring Cloud Gateway) — Definition:** a Spring-native API Gateway implementation providing the same single-entry-point role already covered in the AWS/System Design notes — routing, cross-cutting concerns (auth, rate limiting) — implemented as just another Spring Boot application, rather than a separate managed cloud service.

**Config management (Spring Cloud Config) — Definition:** a centralized configuration server that multiple microservices pull their `application.yml`-equivalent configuration from at startup, backed by a Git repository as the source of truth — lets configuration for many services be managed/versioned in one place, and (with Spring Cloud Bus) can push configuration changes to running services without a redeploy.

**Resilience (Resilience4j) — Definition:** a lightweight library providing the fault-tolerance patterns already covered conceptually in the Node.js and System Design notes — **circuit breakers** (stop calling a failing downstream service temporarily), **retries** (with configurable backoff), **rate limiters**, **bulkheads** — as composable annotations/decorators wrappable around any Spring bean method.

```java
@CircuitBreaker(name = "userService", fallbackMethod = "fallbackUser")
public User getUser(Long id) { return userClient.getUser(id); }

public User fallbackUser(Long id, Throwable t) { return User.unknown(); }
```

**Inter-service communication — Definition:** **Feign clients** provide a declarative, interface-based way to define an HTTP client for calling another service (annotate an interface, Spring generates the implementation — conceptually similar to Spring Data JPA's repository-interface generation, section 11); `RestTemplate` (legacy, synchronous, blocking) and `WebClient` (modern, reactive, non-blocking, built on Project Reactor) are the two lower-level HTTP client options for calling another service directly without Feign's declarative layer.

**Distributed tracing (recap)** — see the AWS/Docker-Kubernetes/System Design notes; Spring Cloud Sleuth (or, in its modern form, direct Micrometer Tracing integration) automatically instruments requests flowing across Spring-based microservices with correlation IDs, exportable to a tracing backend like Zipkin/Jaeger.

---

## 17. Messaging & Async Processing

**JMS fundamentals — Definition:** the Jakarta Messaging (formerly Java Message Service) API is Java's standard, vendor-neutral abstraction for message-oriented middleware (queues and topics) — the traditional Java-ecosystem messaging API, though largely superseded in modern architectures by Kafka/RabbitMQ (below), each with their own richer, protocol-specific Spring integration.

**Kafka with Spring (`spring-kafka`) — Definition:** provides `KafkaTemplate` (for producing messages) and `@KafkaListener` (for declaratively consuming from a topic) — the same log-based messaging model already covered in the System Design notes' section 15, with Spring-idiomatic configuration/annotation-based integration.

```java
@Service
public class OrderEventProducer {
  private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
  public void publish(OrderEvent event) { kafkaTemplate.send("orders", event); }
}

@KafkaListener(topics = "orders")
public void consume(OrderEvent event) { /* process */ }
```

**RabbitMQ with Spring (`spring-amqp`) — Definition:** provides `RabbitTemplate` and `@RabbitListener`, the equivalent integration for RabbitMQ's traditional queue/exchange (AMQP) model — the same SQS/SNS-style messaging concepts already covered in the AWS/System Design notes, applied to a self-hosted or managed RabbitMQ broker instead.

**`@Async` and Spring's task execution — Definition:** annotating a method `@Async` (with async support enabled via `@EnableAsync`) causes Spring to execute it on a separate thread from a configured task executor, immediately returning control to the caller (optionally returning a `CompletableFuture` for the caller to later obtain the result) — the Spring-declarative equivalent of manually submitting a task to an `ExecutorService` (section 6).

**Scheduled tasks (`@Scheduled`) — Definition:** annotating a method `@Scheduled(cron = "0 0 2 * * *")` (or a fixed-rate/fixed-delay variant) runs it automatically on that recurring schedule — the same cron-job concept already covered in the Node.js notes' `node-cron` section, built directly into Spring rather than requiring a separate library, with the same caveat that a single-instance application's scheduled task can be silently skipped/duplicated across multiple horizontally-scaled instances unless explicitly coordinated (e.g. via ShedLock, or a dedicated scheduler in a multi-instance deployment).

---

## 18. Performance & JVM Tuning

**Heap sizing (`-Xms`/`-Xmx`) — Definition:** `-Xms` sets the JVM's *initial* heap size, `-Xmx` sets the *maximum* heap size the JVM may grow to — setting `-Xms` equal to `-Xmx` avoids the overhead of the heap dynamically resizing during warmup, a common production tuning practice for latency-sensitive services.

**GC tuning basics** — see section 5; choosing/tuning a garbage collector (`-XX:+UseG1GC`, or explicitly configuring pause-time targets) is a genuine performance lever, but should be driven by actual observed GC behavior (via GC logs or profiling, below) rather than applied speculatively — the same "measure, don't guess" discipline emphasized throughout this workspace's other notes.

**Profiling tools — Definition:** **JFR (Java Flight Recorder)** — a low-overhead, built-in profiler shipped with the JDK itself, suitable for continuous production profiling without meaningfully impacting performance; **VisualVM** — a GUI tool for inspecting a running JVM's heap, threads, and CPU usage interactively, well-suited to local development investigation; **async-profiler** — a low-overhead sampling profiler particularly good at accurately profiling native/JIT-compiled code paths that other profilers can misattribute or distort.

**Connection pooling (HikariCP) — Definition:** HikariCP is the default, high-performance JDBC connection pool bundled with Spring Boot — the same connection-pooling concept already covered in the Node.js/SQL notes, reusing a fixed set of database connections across requests rather than opening a new one per query, configured via `spring.datasource.hikari.*` properties (pool size, timeout).

**Caching (`@Cacheable`) — Definition:** Spring's Cache abstraction lets a method's return value be transparently cached (keyed by its arguments) via a single annotation, backed by a pluggable cache provider (an in-memory `ConcurrentMapCache` for simple cases, or Redis for a distributed, production-grade cache — the same Redis caching concept as the Node.js notes' section 11).

```java
@Cacheable("users")
public User findById(Long id) { return userRepository.findById(id).orElseThrow(); }
```

**Common performance pitfalls** — the N+1 query problem (section 11); forgetting connection pool sizing under load; unnecessarily broad `@Transactional` boundaries holding database locks/connections longer than needed; blocking I/O calls inside a thread pool sized for CPU-bound work, exhausting available threads under concurrent load (a problem virtual threads, section 6, substantially mitigate).

---

## 19. Production Engineering

**Packaging & deployment (JAR/WAR, Docker) — Definition:** a Spring Boot application is typically packaged as an executable "fat JAR" (section 9) and containerized with a multi-stage Dockerfile (the same pattern already covered in the Docker/Kubernetes notes' section 2, with a JDK/JRE base image instead of Node's).

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY . .
RUN mvn package -DskipTests

FROM eclipse-temurin:21-jre-alpine
COPY --from=build /app/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Externalized configuration (recap)** — see section 9; production configuration (database URLs, secrets) supplied via environment variables/a config server, never hardcoded — the same principle covered throughout every other notes file's production-engineering sections in this workspace.

**Health checks & readiness/liveness probes (recap)** — Spring Boot Actuator's `/actuator/health` endpoint directly plugs into Kubernetes' liveness/readiness probe mechanism (Docker/Kubernetes notes' section 18) or an AWS load balancer's health check (AWS notes' section 3).

**Logging (SLF4J, Logback) — Definition:** SLF4J is a logging **facade** (a common API that decouples application code from a specific logging implementation); Logback is the default, most common concrete implementation Spring Boot wires up automatically — the same structured-logging principle already covered in the Node.js notes' section 16 (Winston/Pino), applied to the Java ecosystem's standard logging stack.

**Observability (Micrometer, distributed tracing) — Definition:** Micrometer is a vendor-neutral application metrics facade (conceptually the Java equivalent of a Prometheus client library, System Design notes' section 14), automatically wired into Spring Boot Actuator, exportable to Prometheus/Datadog/other backends — combined with distributed tracing (section 16), giving Spring applications the same "three pillars of observability" (metrics/logs/traces) covered in the System Design notes.

**Graceful shutdown (recap)** — see the Node.js/AWS/Kubernetes notes; Spring Boot supports graceful shutdown natively (`server.shutdown=graceful`), stopping new request acceptance while letting in-flight requests complete before the JVM process actually exits.

**Zero-downtime deployment strategies (recap)** — the same blue/green, canary, and rolling deployment strategies already covered in the AWS/Docker-Kubernetes notes apply identically to a Spring Boot application deployed on Kubernetes/ECS/any other orchestrated platform — nothing Spring-specific changes about the deployment strategy itself, only the application being deployed.

---

## 20. Interview Preparation

**Core Java interview questions** — the difference between `==` and `.equals()` (section 1); why `hashCode()` and `equals()` must be overridden together (section 2); checked vs unchecked exceptions (section 4); how `HashMap` handles collisions internally (chaining, same concept as the DSA notes' section 5); the difference between an abstract class and an interface (section 2).

**Spring/Spring Boot interview questions** — what Inversion of Control actually means and why it matters (section 8); the Spring bean lifecycle (section 8); how Spring Boot's auto-configuration works under the hood (section 9); the difference between `@Component`/`@Service`/`@Repository` (section 8); why constructor injection is generally preferred over field injection (section 8).

**Concurrency interview questions** — the difference between `synchronized` and `ReentrantLock` (section 6); what `volatile` does and doesn't guarantee (section 6); how to prevent a deadlock (consistent lock-acquisition ordering across all threads is the classic answer); the difference between `Runnable` and `Callable` (section 6); how `ConcurrentHashMap` achieves better concurrent performance than a synchronized `HashMap` (section 6).

**JVM internals interview questions** — Young vs Old generation garbage collection (section 5); the difference between the heap and the stack (section 5); what happens during class loading (section 5); how JIT compilation improves performance over pure interpretation (section 5).

**Common coding problems in Java** — the same core DSA problems covered in the DSA notes (arrays/strings/trees/graphs/DP), implemented with Java-specific idioms — using `StringBuilder` for string-building problems (section 1), the Collections Framework's specific classes (`ArrayDeque` for a stack/queue, `PriorityQueue` for a heap — Java's built-in `PriorityQueue` class directly implements the min-heap behavior described in the DSA notes' section 7) rather than hand-rolling the underlying data structure from scratch, as is more common in the JS-based DSA notes' examples.
