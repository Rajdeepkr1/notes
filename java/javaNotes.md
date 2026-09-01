# Java (Core Language) — Deep Dive Roadmap

We'll go from syntax fundamentals → OOP deep dive → generics & collections internals → functional programming → concurrency → JVM internals → modern language features → tooling.

*This file goes deep on **core Java the language and the JVM itself** — the Java Backend notes cover Java fundamentals briefly (sections 1–7 there) before pivoting to Spring/backend engineering; this file is the detailed expansion of exactly that language layer, and is meant to be read as its prerequisite foundation. Cross-references the Java Backend notes (where Spring builds on these concepts), the DSA notes (Collections Framework internals mirror data structures covered there), and the Design Patterns notes (idiomatic Java implementations of each pattern).*

---

## 1. Java Fundamentals & the JVM Ecosystem

**Definition:** Java is a statically-typed, class-based, object-oriented language compiled not directly to machine code but to platform-independent **bytecode** — executed by the **JVM (Java Virtual Machine)**, the runtime that actually interprets/JIT-compiles bytecode into native instructions for the host machine — the mechanism behind Java's "write once, run anywhere" promise, since the same compiled `.class` bytecode runs unmodified on any platform with a compatible JVM.

**JVM/JRE/JDK relationship — Definition:** the **JVM** is the runtime engine that executes bytecode; the **JRE (Java Runtime Environment)** bundles the JVM plus the core standard library classes needed to *run* a Java program; the **JDK (Java Development Kit)** bundles the JRE plus the tools needed to *develop* Java (the `javac` compiler, debugger, `jar` tool) — a production server typically only strictly needs a JRE to run compiled applications, while development machines need the full JDK — though modern distributions increasingly just ship a full JDK for simplicity.

**Primitive types vs reference types, autoboxing/unboxing — Definition:** Java has eight **primitive types** (`int`, `long`, `double`, `boolean`, `char`, etc.) — stored directly by value, not objects, with no method calls possible on them and no `null` possibility; every other type is a **reference type** (objects, arrays, `String`) — variables holding a reference to heap-allocated memory, nullable, and interacting through method calls. **Autoboxing** automatically converts a primitive to its wrapper object (`int` → `Integer`) when an object is required (e.g. adding to a `List<Integer>`, since generics can't work with primitives directly, section 6); **unboxing** is the reverse — convenient, but a hidden performance cost (extra object allocation) and a subtle `NullPointerException` risk (unboxing a `null` `Integer` throws) worth being deliberately aware of.

```java
int x = 5;                    // primitive, stack-stored, value semantics
Integer boxed = x;              // autoboxing — allocates an Integer object
List<Integer> list = new ArrayList<>();
list.add(x);                     // autoboxed automatically to add to the generic collection
```

**Arrays — fixed-size, type-safety, multidimensional — Definition:** a Java array is a fixed-length, homogeneously-typed, index-accessed reference type (`int[] arr = new int[10]`) — its length is set permanently at creation and cannot grow (unlike an `ArrayList`, section 7); arrays are **covariant** (`Object[] objs = new String[3]` compiles), a design choice that trades compile-time type safety for runtime `ArrayStoreException` risk if a mismatched type is later stored into it — one of a few places generics (section 6), designed after arrays, deliberately made a stricter, invariant choice instead.

---

## 2. Classes, Objects & Object-Oriented Fundamentals

**Classes, objects, constructors, `this` — Definition:** a **class** is a blueprint defining a type's fields (state) and methods (behavior); an **object** is a runtime instance of a class, allocated on the heap; a **constructor** initializes a newly-created object's state, invoked automatically via `new` — `this` refers to the current object instance, most commonly used to disambiguate a constructor/method parameter from a same-named field.

```java
public class Account {
    private final String owner;
    private double balance;

    public Account(String owner, double balance) {
        this.owner = owner;      // `this.owner` (field) vs `owner` (parameter)
        this.balance = balance;
    }
}
```

**The four OOP pillars in Java — Definition:**
- **Encapsulation** — bundling data (fields) with the methods that operate on it, and restricting direct external access via access modifiers (below) — the `Account` example above encapsulates `balance`, exposing it only through controlled methods rather than a public field any external code could mutate arbitrarily.
- **Inheritance** — a class (`extends`) acquiring the fields/methods of a parent class, covered fully in section 3.
- **Polymorphism** — a single interface/method call behaving differently depending on the actual runtime type of the object it's invoked on, covered fully in section 3.
- **Abstraction** — exposing only essential behavior through a simplified interface while hiding implementation complexity — realized concretely through abstract classes and interfaces (section 4).

**`static` vs instance members — Definition:** an **instance member** (field or method) belongs to, and its value/behavior varies per, each individual object instance; a **`static`** member belongs to the **class itself**, shared identically across every instance (and accessible without any instance at all, e.g. `Math.sqrt(x)`) — the practical rule of thumb: state or behavior that's conceptually the same regardless of which specific object you're looking at (a utility function, a shared counter of total instances created) belongs on the class as `static`; state that genuinely varies per object belongs as an instance member.

**Access modifiers — Definition:** `public` (accessible from anywhere), `protected` (accessible within the same package, plus subclasses in other packages), package-private/default (no modifier — accessible only within the same package), `private` (accessible only within the declaring class itself) — this four-level visibility system is Java's concrete mechanism for enforcing encapsulation above, and the same modifier vocabulary NestJS/TypeScript (JS/TS notes) deliberately borrowed its own `public`/`private`/`protected` keywords from.

---

## 3. Inheritance & Polymorphism Deep Dive

**`extends`, overriding vs overloading — Definition:** a subclass declared with `extends` inherits its superclass's non-private members and can **override** an inherited method (`@Override`, same signature, providing new behavior for that specific subclass) — distinct from **overloading** (multiple methods in the *same* class sharing a name but differing in parameter types/count, resolved at *compile time* based on the static argument types) — overriding is resolved at *runtime* based on the object's actual type (dynamic dispatch, below), the fundamental distinction between these two commonly-confused, similarly-named concepts.

```java
class Shape {
    double area() { return 0; }
}
class Circle extends Shape {
    private final double radius;
    Circle(double radius) { this.radius = radius; }
    @Override
    double area() { return Math.PI * radius * radius; } // overriding — resolved at runtime
}
```

**`super`, constructor chaining — Definition:** `super.method()` explicitly invokes the parent class's version of an overridden method (used when a subclass wants to *extend*, not fully replace, inherited behavior); `super(...)` as a constructor's first statement invokes the parent's constructor — every constructor implicitly calls `super()` (the no-arg parent constructor) if not written explicitly, which is why a parent class lacking a no-arg constructor *forces* every subclass constructor to explicitly call a specific parent constructor instead.

**Abstract classes vs interfaces (brief, full detail section 4)** — an `abstract class` can provide partial implementation alongside abstract (unimplemented) methods a subclass must fill in, and a class can `extend` only **one** abstract (or any) class; an `interface` traditionally declared only method signatures (though modern Java relaxes this considerably, section 4), and a class can `implement` **multiple** interfaces — this single-inheritance-of-class-vs-multiple-implementation-of-interface asymmetry is one of Java's most consequential and frequently interview-tested design decisions.

**Dynamic dispatch — how polymorphism actually works at runtime — Definition:** when code calls `shape.area()` on a variable declared as type `Shape` but actually referencing a `Circle` object, the JVM determines **at runtime** which actual `area()` implementation to invoke, based on the object's real, concrete type — not the variable's declared, static type — implemented internally via a **virtual method table (vtable)** each class carries, mapping method signatures to the correct implementation for that specific class — this runtime lookup is precisely what makes polymorphism work (`Shape[] shapes = {new Circle(...), new Square(...)}`, calling `.area()` uniformly on each, correctly invoking each one's own specific implementation).

---

## 4. Interfaces & Abstract Classes in Depth

**Interface evolution: default, static, private methods — Definition:** originally, an `interface` could only declare abstract method signatures with no implementation at all; since Java 8, an interface can provide a **default method** (a concrete implementation a class inherits automatically unless it chooses to override it — specifically added to let library interfaces add new methods without breaking every existing implementing class), a **static method** (utility methods callable on the interface itself, `Comparator.comparing(...)`), and (since Java 9) a **private method** (internal helper logic shared between an interface's own default methods, not exposed to implementers) — this evolution significantly blurred the traditional abstract-class/interface distinction.

```java
interface Greetable {
    String name();
    default String greet() { return "Hello, " + name() + "!"; } // default method
    static Greetable of(String n) { return () -> n; }             // static factory method
}
```

**Multiple inheritance of type vs behavior — Definition:** a Java class can `implement` multiple interfaces (multiple inheritance of *type* — a single object can satisfy many different type contracts simultaneously) but can only `extend` one class (no multiple inheritance of *implementation state* — avoiding the classic "diamond problem" ambiguity C++ multiple inheritance is notorious for) — default methods (above) technically permit a limited form of multiple inheritance of *behavior*, and Java requires an implementing class to explicitly resolve any conflict if it inherits the same default method signature from two different interfaces, rather than silently picking one.

**Functional interfaces, sealed interfaces — Definition:** a **functional interface** (`@FunctionalInterface`) declares exactly **one** abstract method — the target type lambda expressions (section 13) implement (`Runnable`, `Comparator<T>`, `Function<T, R>`); a **sealed interface** (Java 17+, section 14) restricts *which specific classes* are permitted to implement it (`sealed interface Shape permits Circle, Square {}`), enabling the compiler to verify a `switch` expression pattern-matching over that interface's permitted types is genuinely exhaustive — a meaningfully stronger compile-time guarantee than an unsealed interface allows.

**When to choose an interface vs an abstract class — Definition:** favor an **interface** when defining a capability/contract unrelated classes might share (`Comparable`, `Serializable`) with no shared implementation state needed, since a class can implement many interfaces but extend only one class; favor an **abstract class** when subclasses genuinely share common state/implementation and form a real "is-a" hierarchy (a `Shape` base class holding shared fields, with concrete subclasses filling in `area()`) — the modern, post-default-methods practical guidance increasingly leans interface-first except where actual shared mutable state or constructor logic is genuinely needed, since interfaces alone often now suffice for what previously required an abstract class.

---

## 5. The `Object` Class & Its Contracts

**`equals()`/`hashCode()` contract — Definition:** every Java class implicitly extends `Object`, inheriting its default `equals()` (reference identity — `a == b`) and `hashCode()` (derived from memory address by default) — overriding `equals()` to compare logical/field equality **requires** also overriding `hashCode()` consistently, per Java's documented contract: **equal objects must have equal hash codes** (though unequal objects may coincidentally share a hash code) — violating this contract (overriding `equals()` without `hashCode()`) doesn't throw an error, but silently breaks any `HashMap`/`HashSet` usage (section 7), since those collections locate objects by hash bucket first, then confirm with `equals()` — an object whose hash code doesn't match its logically-equal counterpart's simply won't be found, a notoriously subtle, hard-to-diagnose bug class.

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Point p)) return false; // pattern matching instanceof, section 14
    return x == p.x && y == p.y;
}
@Override
public int hashCode() { return Objects.hash(x, y); } // consistent with equals — same fields
```

**`toString()`, `clone()`, `getClass()` — Definition:** `toString()`'s default implementation (`ClassName@hashCode`) is rarely useful — overriding it to produce a readable representation is standard practice, particularly valuable for logging/debugging; `clone()` (from `Cloneable`) provides object copying but is widely considered a design misstep in Java's original API (shallow-copies by default, requires awkward exception handling, easy to get wrong for objects containing mutable references) — a copy constructor or static factory method is now generally the preferred alternative; `getClass()` returns an object's actual runtime `Class` object, the entry point into the Reflection API (section 16).

**Immutability — why and how — Definition:** an **immutable** object's state cannot change after construction — `final` fields, no setters, and defensive copying of any mutable fields passed in/out — immutability eliminates entire categories of bugs (no risk of unexpected mutation by a shared reference elsewhere in the code, section 12's concurrency implications: an immutable object is automatically thread-safe with zero synchronization needed) — `String` (section 9) is Java's most consequential built-in immutable type, and Records (section 14) are the modern, concise way to define custom immutable data carriers.

---

## 6. Generics

**Type erasure — what it means and its consequences — Definition:** Java generics exist **only at compile time** — the compiler uses type parameters (`List<String>`) to enforce type safety and insert automatic casts, but **erases** them from the actual compiled bytecode, where `List<String>` and `List<Integer>` are both simply `List` at runtime — this design choice (made to preserve backward compatibility with pre-generics Java code and bytecode) has real practical consequences: you cannot create a generic array (`new T[10]` doesn't compile), cannot use `instanceof` against a parameterized type (`obj instanceof List<String>` doesn't compile — only the erased `List` is checkable), and reflection cannot recover a generic collection's element type at runtime.

```java
List<String> strings = new ArrayList<>();
List<Integer> integers = new ArrayList<>();
System.out.println(strings.getClass() == integers.getClass()); // true — both erased to raw ArrayList
```

**Bounded type parameters — Definition:** `<T extends Comparable<T>>` restricts a generic type parameter to types satisfying a given upper bound, both restricting what types can be substituted *and* enabling the generic code to call methods that bound guarantees exist (e.g. calling `.compareTo()` on a `T` bounded by `Comparable<T>`, which wouldn't be callable on an unbounded `T`).

```java
static <T extends Comparable<T>> T max(List<T> list) {
    T result = list.get(0);
    for (T item : list) if (item.compareTo(result) > 0) result = item;
    return result;
}
```

**Wildcards: `? extends`, `? super`, PECS — Definition:** `List<? extends Number>` accepts a list of `Number` *or any subtype* (used when only **reading** from the collection — "Producer Extends"); `List<? super Integer>` accepts a list of `Integer` *or any supertype* (used when only **writing** into the collection — "Consumer Super") — together forming the **PECS** mnemonic ("Producer Extends, Consumer Super"), the standard guideline for choosing which wildcard direction a generic method parameter needs, directly analogous in spirit to the covariance/contravariance concepts already touched on for TypeScript's structural type system (JS/TS notes).

**Generic methods vs generic classes — Definition:** a **generic class** (`class Box<T> { T value; }`) parameterizes the entire class, with the type fixed once per instance; a **generic method** (the `max` example above) parameterizes just a single method's type independently of any enclosing class's own type parameters — useful for utility/static methods that need genericity without the surrounding class itself needing to be generic at all.

---

## 7. The Collections Framework In Depth

**Collection hierarchy — Definition:** `Collection` is the root interface, branching into `List` (ordered, index-accessible, duplicates allowed — `ArrayList`, `LinkedList`), `Set` (no duplicates — `HashSet`, `TreeSet`, `LinkedHashSet`), and `Queue`/`Deque` (FIFO/double-ended access — `ArrayDeque`, `PriorityQueue`); `Map` (key-value pairs — `HashMap`, `TreeMap`) is notably **not** a `Collection` subtype at all, since its fundamental access pattern (by key) differs enough from the single-element access model the `Collection` interface is built around.

**`ArrayList` vs `LinkedList` internals (recap DSA notes) — Definition:** `ArrayList` is backed by a dynamically-resized array (DSA notes' dynamic array section) — O(1) index-based access, amortized O(1) append, but O(n) insertion/removal at an arbitrary middle position (requiring shifting subsequent elements); `LinkedList` is a doubly-linked list (DSA notes' linked-list section) — O(1) insertion/removal *once positioned* at a node, but O(n) index-based access (requiring traversal from an end) — the practical guidance mirrors the DSA notes' general array-vs-linked-list tradeoff exactly: `ArrayList` is the correct default for nearly all cases (better cache locality, simpler), reaching for `LinkedList` only when genuinely frequent insertion/removal at arbitrary, already-known positions dominates the access pattern.

**`HashMap` internals: hashing, buckets, treeification — Definition:** `HashMap` stores entries in an internal array of **buckets**, with a key's `hashCode()` (section 5) determining which bucket it lands in — multiple keys hashing to the same bucket (a **collision**) are historically chained in a linked list within that bucket; since Java 8, a bucket accumulating enough collisions (default threshold: 8 entries) is automatically **treeified** into a balanced red-black tree instead — improving worst-case lookup within a heavily-collided bucket from O(n) to O(log n) — a JDK-internal optimization directly built on the same tree/hash-table concepts covered generally in the DSA notes, applied specifically to guard against pathological hash-collision performance degradation.

**`TreeMap`/`TreeSet`, `LinkedHashMap` — Definition:** `TreeMap`/`TreeSet` are backed by a **red-black tree** (DSA notes' balanced-BST section), maintaining keys in sorted order at the cost of O(log n) operations (versus `HashMap`'s average O(1)) — the right choice specifically when sorted iteration order or range queries (`headMap`, `tailMap`) are needed; `LinkedHashMap` maintains **insertion order** (or, configurably, access order — useful for implementing an LRU cache) by additionally threading a doubly-linked list through its hash-table entries, trading a small memory/performance overhead over plain `HashMap` for predictable iteration order.

**Choosing the right collection — decision framework:**
- Need index-based access, mostly append/read? → `ArrayList`
- Need no duplicates, don't care about order? → `HashSet`
- Need no duplicates, sorted order? → `TreeSet`
- Need no duplicates, insertion order preserved? → `LinkedHashSet`
- Need key-value lookup, don't care about order? → `HashMap`
- Need key-value lookup, sorted by key? → `TreeMap`
- Need FIFO/LIFO/priority-based processing? → `ArrayDeque`/`PriorityQueue`
- Need thread-safe collection access? → see section 12's concurrent collections, not a synchronized wrapper around the above by default.

---

## 8. Exception Handling Deep Dive

**Checked vs unchecked exceptions — Definition:** a **checked exception** (extending `Exception` but not `RuntimeException`) must be either caught or explicitly declared (`throws`) by any method that might throw it — enforced by the compiler; an **unchecked exception** (extending `RuntimeException`, or `Error`) requires no such declaration — Java's checked-exception design was intended to force explicit handling of recoverable, expected failure conditions (a file not found), while unchecked exceptions represent programming errors or conditions not generally meant to be recovered from inline (`NullPointerException`, `IllegalArgumentException`) — this design remains genuinely controversial: many modern Java codebases and libraries (and other JVM languages like Kotlin) deliberately avoid checked exceptions, arguing they tend to produce excessive, formulaic try-catch/throws-clause boilerplate more often than they meaningfully improve error handling, a design debate worth knowing exists rather than treating checked exceptions as an uncontroversial best practice.

**Exception hierarchy, custom exceptions — Definition:** `Throwable` is the root of Java's exception hierarchy, branching into `Error` (serious, generally unrecoverable JVM-level problems — `OutOfMemoryError` — not meant to be caught by application code) and `Exception` (application-level conditions, further split into checked and unchecked as above); a custom exception typically extends `RuntimeException` (unchecked, avoiding forcing every caller up the stack to handle it explicitly) unless the failure is genuinely something callers should be compiler-forced to consciously handle.

**try-with-resources & `AutoCloseable` — Definition:** any class implementing `AutoCloseable` (defining a `close()` method) can be used in a **try-with-resources** statement, which automatically calls `close()` when the block exits — whether normally or via an exception — eliminating the classic, error-prone manual `finally { resource.close(); }` boilerplate pattern, and correctly handling the subtle case of an exception being thrown both from the try block *and* from `close()` itself (the original exception is preserved as primary, with the close-exception attached as a "suppressed" exception rather than silently discarded).

```java
try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
    return reader.readLine();
} // reader.close() called automatically, even if readLine() throws
```

**Best practices: exception chaining, catch vs propagate — Definition:** when catching an exception specifically to wrap it in a different, more contextually meaningful exception type, always pass the original as the **cause** (`throw new ServiceException("...", originalException)`) — preserving the full original stack trace for debugging rather than silently discarding it; the general guidance on catching vs propagating mirrors the same principle already covered across this workspace's backend notes (Node.js/Python/NestJS): catch an exception only where you can meaningfully *handle* it (retry, fall back, translate to a proper error response) — catching purely to log and rethrow unchanged, or swallowing an exception silently, are both anti-patterns that hide genuine failures.

---

## 9. String Handling & the String Pool

**String immutability — why, and performance implications — Definition:** `String` in Java is **immutable** — every apparent "modification" (`str.concat(...)`, `str.toUpperCase()`) actually returns a **new** `String` object, leaving the original untouched — deliberately designed this way for thread-safety (an immutable object needs no synchronization to share safely across threads, section 12), safe use as a `HashMap` key (a mutable key's changing hash code, section 5, would corrupt the map), and to enable the String Pool optimization below — the performance cost is that naive repeated string concatenation in a loop creates a new object on every iteration, which is precisely what `StringBuilder` (below) exists to avoid.

**The String Pool, `String` vs `new String()` — Definition:** string literals (`"hello"`) are automatically **interned** — stored once in a special JVM memory region (the **String Pool**), with every occurrence of the same literal value reusing the identical pooled object rather than allocating a new one — `new String("hello")` deliberately bypasses this, forcing a genuinely new, separate heap object even for a value already present in the pool — this is exactly why `==` (reference equality) can appear to work for string literals but is still fundamentally the wrong tool for string comparison (`.equals()` is always correct; `==` on strings is a classic, common bug source precisely because pooling makes it *sometimes* accidentally work).

```java
String a = "hello";
String b = "hello";
String c = new String("hello");
System.out.println(a == b);        // true — same pooled literal object
System.out.println(a == c);        // false — c is a distinct heap object
System.out.println(a.equals(c));    // true — correct way to compare string content
```

**`StringBuilder`/`StringBuffer` — Definition:** `StringBuilder` is a **mutable** character sequence, letting repeated append/insert/delete operations modify the same underlying buffer in place rather than allocating a new `String` object on every operation — the correct tool for building up a string through many operations (e.g. within a loop); `StringBuffer` is `StringBuilder`'s older, `synchronized` (thread-safe, section 12) equivalent — largely superseded by `StringBuilder` for single-threaded use, where the synchronization overhead is pure, unnecessary cost.

**Text blocks (Java 15+) — Definition:** `"""..."""` triple-quoted text blocks let multi-line string literals (JSON payloads, SQL queries, HTML fragments) be written directly with natural line breaks and without manual `\n` concatenation/escaping — a direct, welcome quality-of-life improvement for the kind of embedded multi-line string content that's genuinely common in real backend code (the Java Backend notes' SQL/JSON examples benefit directly from this).

---

## 10. Java Memory Model & Garbage Collection

**Stack vs heap, method area/metaspace — Definition:** each thread has its own **stack**, storing method call frames (local variables, primitive values, object references) — automatically reclaimed the instant a method returns, extremely fast; the **heap** (shared across all threads) stores every actual object instance, managed by the garbage collector (below) rather than reclaimed deterministically on scope exit; **metaspace** (replacing the older, fixed-size "PermGen" as of Java 8) stores class metadata itself (method bytecode, static fields) — this three-region split is the JVM's concrete memory architecture underlying every "primitive vs reference type" distinction already discussed in section 1.

**Object lifecycle, reachability, GC roots — Definition:** an object becomes eligible for garbage collection the instant it becomes **unreachable** — no longer reachable via any chain of references starting from a **GC root** (active thread stacks, static fields, JNI references) — Java's garbage collector doesn't use simple reference counting (unlike, e.g., Python's primary mechanism) specifically because reference counting can't reclaim **cycles** (two objects referencing each other but unreachable from any root) — instead using a mark-and-sweep-family algorithm that traces reachability from roots directly, correctly reclaiming cyclic garbage that reference counting alone would miss.

**GC algorithms overview — Definition:** **Serial GC** — single-threaded, simplest, appropriate only for small heaps/single-core environments; **Parallel GC** — multi-threaded collection, optimized for maximum application *throughput*, historically the default, accepting longer individual pause times in exchange; **G1 (Garbage First)** — the modern default since Java 9, divides the heap into regions and prioritizes collecting the most garbage-dense regions first, targeting a configurable, predictable *pause-time goal* rather than pure throughput; **ZGC**/**Shenandoah** — newer, low-latency collectors targeting sub-millisecond pause times even on very large (multi-terabyte) heaps, at some throughput cost — the practical guidance: G1 is the right default for most applications; ZGC/Shenandoah are reached for specifically when an application has hard, strict latency requirements a G1 pause could violate (cross-reference the Java Backend notes' JVM-tuning section for how this choice is actually configured in production).

**Memory leaks in Java despite GC — how they still happen — Definition:** garbage collection prevents *dangling-pointer*-style memory corruption, but does **not** prevent memory leaks — an object remains ineligible for collection as long as *any* reference chain from a GC root still reaches it, so a genuine leak occurs whenever code unintentionally retains such a reference far longer than needed — classic causes: a static collection that's only ever added to and never cleared, listeners/callbacks registered but never unregistered (keeping the registering object reachable through the listener registry indefinitely), or inner classes holding an implicit reference to their outer class instance — this is a direct parallel to the Android notes' section 15 discussion of Activity-reference leaks, the same underlying "reachability" mechanism, just a different specific leak pattern.

---

## 11. Multithreading Fundamentals

**The `Thread` class vs `Runnable` — Definition:** a unit of concurrent work can be defined either by extending `Thread` directly and overriding `run()`, or (the generally preferred approach) by implementing the `Runnable` functional interface (section 13) and passing it to a `Thread` constructor — preferred specifically because Java doesn't support multiple inheritance (section 4): a class extending `Thread` can't extend anything else, while a class implementing `Runnable` remains free to extend a different, more meaningful base class if needed, and separates "what work to do" from "how it's executed" — a separation that also makes `Runnable`s directly reusable with the `ExecutorService` thread-pooling model covered in section 12.

```java
Runnable task = () -> System.out.println("running on: " + Thread.currentThread().getName());
new Thread(task).start();
```

**Thread lifecycle, thread states — Definition:** a thread moves through defined states — `NEW` (created, not yet started) → `RUNNABLE` (executing or eligible to execute) → `BLOCKED`/`WAITING`/`TIMED_WAITING` (paused, waiting on a lock or explicit wait condition) → `TERMINATED` (finished) — understanding this lifecycle is essential for diagnosing concurrency bugs, since a deadlocked thread typically shows as permanently `BLOCKED`, visible directly in a thread-dump analysis.

**`synchronized`, intrinsic locks, `volatile` — Definition:** the `synchronized` keyword (on a method or a block) ensures only one thread at a time can execute that critical section for a given object's **intrinsic lock** (every object implicitly carries one) — preventing race conditions (below) at the cost of serializing access and the throughput/contention overhead that implies; `volatile` on a field guarantees that **visibility** of writes to that field is immediately propagated across threads (preventing a thread from reading a stale, CPU-cache-local copy of the value) — but, critically, `volatile` alone does **not** provide atomicity for compound operations (`count++` is actually read-modify-write, three separate steps — still a race condition even on a `volatile` field) — a commonly-tested, easy-to-misunderstand distinction.

**Race conditions, deadlocks — Definition:** a **race condition** occurs when multiple threads access shared mutable state without proper synchronization, and the final outcome depends unpredictably on the actual timing/interleaving of their execution (classically, two threads both reading, incrementing, and writing back a shared counter, with one increment silently lost); a **deadlock** occurs when two or more threads each hold a lock the other needs, waiting on each other indefinitely, with neither able to proceed — the standard prevention strategy is establishing and consistently following a fixed, global lock-acquisition ordering across the entire codebase, so no two threads can ever end up waiting on each other's already-held locks in reverse order.

---

## 12. Concurrency Utilities (java.util.concurrent)

**`ExecutorService` & thread pools — Definition:** rather than manually creating and managing raw `Thread` objects (expensive to create, no built-in reuse), an `ExecutorService` manages a **pool** of reusable worker threads, accepting submitted tasks (`Runnable`/`Callable`) and executing them across the pool — `Executors.newFixedThreadPool(n)` (a bounded pool), `newCachedThreadPool()` (grows/shrinks dynamically), or (modern, section 14) `Executors.newVirtualThreadPerTaskExecutor()` — the standard, recommended way to run concurrent work rather than manual thread management, directly analogous to the connection-pooling principle already covered for database connections in the SQL/Database notes, applied here to threads themselves.

```java
ExecutorService executor = Executors.newFixedThreadPool(4);
Future<Integer> result = executor.submit(() -> computeExpensiveValue());
executor.shutdown();
```

**`Future`, `CompletableFuture` — Definition:** a `Future<T>` represents the eventual result of an asynchronous computation, with `.get()` blocking until it's available; `CompletableFuture<T>` (Java 8+) is a substantially richer, composable alternative — supporting chaining (`.thenApply()`, `.thenCompose()`), combining multiple async operations (`.thenCombine()`, `allOf()`), and non-blocking callback registration (`.thenAccept()`) — Java's answer to the same async-composition problem JavaScript's Promises (JS/TS notes' section 7) solve, with broadly analogous chaining/combinator semantics.

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> fetchUser(id))
    .thenApply(user -> user.getName())
    .exceptionally(ex -> "unknown");
```

**Concurrent collections — Definition:** `ConcurrentHashMap` provides thread-safe map access with dramatically better concurrent throughput than a naively `synchronized`-wrapped `HashMap`, by internally partitioning locking to only the specific segment/bucket being modified rather than locking the entire map for every operation; `CopyOnWriteArrayList` creates a fresh copy of its entire backing array on every write, making reads completely lock-free and safe during concurrent iteration — a good fit specifically for read-heavy, write-rare scenarios (e.g. a list of event listeners), a poor fit for write-heavy use given the full-copy cost on every mutation.

**Locks, atomics, coordination primitives — Definition:** `ReentrantLock` provides the same mutual-exclusion guarantee as `synchronized`, but with additional capabilities `synchronized` lacks (tryLock with a timeout, interruptible lock acquisition, fairness policies); `AtomicInteger`/`AtomicLong`/etc. provide lock-free, hardware-CAS-based (compare-and-swap) atomic operations for simple counters/flags, correctly solving the exact `count++` compound-operation race condition flagged in section 11 without needing an explicit lock at all; `CountDownLatch` lets one or more threads wait until a set of other threads' work completes; `Semaphore` limits the number of threads concurrently accessing a resource — all part of Java's `java.util.concurrent` toolkit, purpose-built to avoid needing to hand-roll lower-level `synchronized`/`wait`/`notify` coordination directly.

---

## 13. Java 8 Functional Programming

**Lambda expressions — Definition:** a **lambda expression** (`(params) -> expression`) provides a concise, inline implementation of a **functional interface** (section 4 — an interface with exactly one abstract method) without the verbose anonymous-inner-class syntax it replaces — Java's introduction of first-class function-like values, brought considerably later than most other mainstream languages but built directly on the existing interface system rather than adding entirely new function-type syntax.

```java
// before Java 8: verbose anonymous class
Comparator<String> byLength = new Comparator<String>() {
    public int compare(String a, String b) { return a.length() - b.length(); }
};
// Java 8+: lambda
Comparator<String> byLength2 = (a, b) -> a.length() - b.length();
```

**Method references — Definition:** `ClassName::methodName` is shorthand for a lambda that simply delegates to an existing method — `String::toUpperCase` is equivalent to `s -> s.toUpperCase()` — purely a conciseness improvement over an already-simple lambda, used specifically when a lambda would do nothing but call one existing method with its arguments passed straight through unchanged.

**The Stream API: intermediate vs terminal, laziness — Definition:** a `Stream` represents a sequence of elements supporting functional-style, declarative operations — **intermediate operations** (`.filter()`, `.map()`, `.sorted()`) return a new stream and are **lazy** — they don't actually execute until a **terminal operation** (`.collect()`, `.forEach()`, `.reduce()`) is invoked, at which point the entire pipeline runs in a single pass over the data — this laziness is what allows short-circuiting operations (`.findFirst()`, `.limit()`) to avoid processing the entire source unnecessarily, and closely mirrors the same lazy-evaluation principle already covered for JavaScript generators/RxJS (JS/TS notes) and Kotlin Flow (Android notes' section 10).

```java
List<String> names = users.stream()
    .filter(u -> u.getAge() > 18)
    .map(User::getName)
    .sorted()
    .collect(Collectors.toList());
```

**`Optional` — proper usage vs anti-patterns — Definition:** `Optional<T>` explicitly represents "a value that might be absent," intended specifically as a **return type** signaling to callers that they must consciously handle the absent case (`.map()`, `.orElse()`, `.ifPresent()`) rather than risking an unchecked `NullPointerException` — the well-documented anti-patterns worth flagging: using `Optional` as a field type or method *parameter* type (adds indirection/allocation overhead without the same return-type benefit, and `Optional` itself isn't serializable, complicating its use as persisted state), and calling `.get()` without first checking `.isPresent()` (defeats the entire purpose — reintroduces exactly the same unchecked-null risk `Optional` exists to prevent).

---

## 14. Modern Java Features (9 through 21+)

**Records (Java 16+) — Definition:** a `record` declaration is a concise syntax for an immutable data-carrier class — automatically generating a canonical constructor, `equals()`/`hashCode()` (correctly implemented per the contract in section 5), `toString()`, and accessor methods, from a single declaration — eliminating the substantial, error-prone boilerplate a hand-written immutable class (section 5) previously required, and the direct Java equivalent of TypeScript interfaces/Kotlin data classes (Android notes) for representing plain, structured data.

```java
record Point(int x, int y) {} // constructor, equals, hashCode, toString, accessors — all generated

Point p1 = new Point(1, 2);
Point p2 = new Point(1, 2);
System.out.println(p1.equals(p2)); // true — value-based equality, generated correctly
System.out.println(p1.x());          // 1 — generated accessor
```

**Sealed classes/interfaces (Java 17+) — Definition:** `sealed class Shape permits Circle, Square {}` restricts inheritance to only the explicitly permitted subtypes — combined with pattern-matching `switch` (below), the compiler can verify a `switch` over a sealed type's subtypes is **exhaustive** at compile time (no `default` branch needed, and adding a new permitted subtype later causes every non-exhaustive `switch` elsewhere in the codebase to fail compilation until updated) — directly parallel to TypeScript's discriminated unions (JS/TS notes) providing the same exhaustiveness-checking benefit, here as a first-class Java language feature.

**Pattern matching for `instanceof` and `switch` — Definition:** `if (obj instanceof String s)` both checks the type **and** binds a correctly-typed variable (`s`) in one expression, eliminating the previously-required separate explicit cast; pattern-matching `switch` (Java 21) extends this to `switch` expressions, supporting type patterns and (combined with records above) **record deconstruction patterns** — destructuring a record's components directly within a `case` branch.

```java
sealed interface Shape permits Circle, Square {}
record Circle(double radius) implements Shape {}
record Square(double side) implements Shape {}

double area(Shape shape) {
    return switch (shape) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Square s -> s.side() * s.side();
        // no default needed — compiler verifies exhaustiveness over the sealed hierarchy
    };
}
```

**Virtual threads (Project Loom, Java 21) — Definition:** a **virtual thread** is a lightweight thread managed by the JVM itself rather than mapped one-to-one to a scarce, expensive OS thread (the traditional `Thread` model, section 11) — millions of virtual threads can exist simultaneously, since the JVM efficiently multiplexes them onto a small pool of actual OS "carrier" threads, automatically unmounting a virtual thread from its carrier whenever it blocks (e.g. on I/O) and mounting a different, ready virtual thread in its place — this is a genuine paradigm shift: **simple, blocking-style code** (`result = blockingCall()`) now scales to massive concurrency without needing the complex, harder-to-reason-about async/reactive composition (`CompletableFuture` chains, section 12) that was previously required specifically to avoid exhausting a limited pool of expensive OS threads — directly relevant to the Java Backend notes' Spring Boot section, since Spring MVC's traditional thread-per-request model becomes dramatically more scalable simply by running on virtual threads, without rewriting application code into a reactive style at all.

---

## 15. I/O & NIO

**Classic I/O streams — Definition:** `InputStream`/`OutputStream` handle raw byte-oriented I/O; `Reader`/`Writer` handle character-oriented (text) I/O with charset decoding/encoding — the classic `java.io` package models I/O as **blocking streams** — a read call blocks the calling thread until data is actually available, the traditional, simplest-to-reason-about I/O model, and still entirely adequate for the vast majority of straightforward file/network I/O needs.

**NIO fundamentals: `Path`, `Files`, channels & buffers — Definition:** `java.nio.file.Path`/`Files` (NIO.2, Java 7+) provide a modern, more capable file-system API superseding the older `java.io.File` (richer metadata access, symbolic link support, more ergonomic static utility methods like `Files.readAllLines(path)`); the lower-level NIO **channel/buffer** model represents I/O as reading/writing into `ByteBuffer`s through `Channel`s, supporting non-blocking mode (below) that the classic stream model fundamentally cannot express.

```java
List<String> lines = Files.readAllLines(Path.of("data.txt"));
Files.writeString(Path.of("output.txt"), "hello");
```

**Why NIO exists — blocking vs non-blocking I/O — Definition:** classic blocking I/O ties up one thread per in-flight I/O operation — fine at small scale, but a genuine scalability ceiling for a server handling many thousands of concurrent connections (each blocked thread consumes real OS resources, section 11) — NIO's non-blocking mode lets a single thread monitor many channels simultaneously via a `Selector`, only actually processing a channel once it's genuinely ready for I/O, without dedicating a full blocked thread to each connection — the same underlying motivation behind Node.js's entire non-blocking event-loop architecture (Node.js notes' section 1), here as an optional, lower-level model Java provides alongside (not replacing) its simpler classic blocking streams — worth noting that virtual threads (section 14) now let simple *blocking-style* code achieve much of this same scalability benefit without needing NIO's more complex non-blocking API directly, narrowing when reaching for raw NIO channels/selectors is actually still necessary.

---

## 16. Reflection & Annotations

**The Reflection API — Definition:** `java.lang.reflect` lets code inspect and manipulate classes, methods, fields, and constructors **at runtime**, even ones not known at compile time — `Class<?> clazz = obj.getClass()`, then `clazz.getMethods()`/`.getDeclaredFields()` to enumerate a class's structure, or `method.invoke(obj, args)` to call a method dynamically by name — the mechanism underlying frameworks that need to work generically with arbitrary user-defined classes without those classes needing to implement any specific interface.

**Performance and safety costs of reflection — Definition:** reflective method calls carry real overhead compared to a direct, statically-resolved call (bypassing JIT-compiler optimizations a normal call benefits from, and involving additional security/access checks) — and reflection can bypass normal access-modifier enforcement entirely (`field.setAccessible(true)` lets code read/write even `private` fields) — a powerful capability frameworks rely on, but one application code should reach for sparingly and deliberately, given both the performance cost and the way it can silently violate a class's intended encapsulation.

**Built-in vs custom annotations, retention policies — Definition:** an annotation (`@Override`, `@Deprecated`, or a custom `@interface`) attaches metadata to code without directly affecting its runtime logic — a `@Retention` policy controls how long that metadata persists: `SOURCE` (discarded by the compiler, e.g. `@Override` — purely a compile-time check), `CLASS` (kept in the compiled bytecode but not loaded at runtime by default), or `RUNTIME` (kept and queryable via reflection at runtime — the retention policy required for any annotation a framework needs to inspect dynamically, below).

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Benchmark {}
```

**How frameworks rely on reflection — Definition:** Spring's dependency injection (Java Backend notes' section 8) inspects constructors/fields for `@Autowired`/`@Component` `RUNTIME`-retained annotations via reflection to wire dependencies automatically; JUnit (Java Backend notes' section 13) uses reflection to discover and invoke `@Test`-annotated methods without those methods needing to implement any common interface — nearly every "magic," configuration-light framework behavior covered elsewhere in this workspace's Java-adjacent notes is, underneath, built directly on the reflection and annotation mechanisms covered in this section.

---

## 17. The Java Module System (JPMS)

**Why modules were introduced (Java 9) — Definition:** before modules, Java's unit of encapsulation stopped at the **class** level (via `public`/`private` etc., section 2) — anything `public` in any JAR on the classpath was accessible from anywhere else on the classpath, with no way to say "this package is public API, but that internal package genuinely isn't meant to be used externally" — the **Java Platform Module System** (Project Jigsaw) introduces the **module** as a higher-level unit of encapsulation specifically to close this gap, and was also motivated by breaking the JDK's own previously-monolithic runtime into separately versioned, more maintainable pieces.

**`module-info.java`, `requires`/`exports` — Definition:** a `module-info.java` file at a module's root declares its name, which other modules it `requires` (depends on), and which of its own packages it `exports` (makes accessible to other modules) — a package *not* exported remains genuinely, compiler-enforced inaccessible to other modules even if its classes are `public`, closing the classpath-era gap described above.

```java
module com.example.myapp {
    requires java.sql;
    exports com.example.myapp.api;      // public API, accessible to other modules
    // com.example.myapp.internal is NOT exported — genuinely encapsulated
}
```

**Modules vs the classic classpath — encapsulation benefits — Definition:** the classic classpath remains fully supported and is still what most application code (including much Spring Boot application code, Java Backend notes) actually runs on day to day — full adoption of the module system throughout an application's own code remains relatively uncommon in practice, given the migration effort for existing large codebases and libraries not yet modularized themselves — but understanding the module system remains valuable both for the stronger-encapsulation concept itself and because the JDK's own standard library is now internally modularized, which occasionally surfaces directly (e.g. needing an explicit `requires` for a JDK module that used to be implicitly available on the classpath).

---

## 18. Design Patterns in Java

**Idiomatic Java implementations of each GoF category — Definition:** the Design Patterns notes cover each pattern's intent and structure generally and language-agnostically — this section is specifically about how each pattern is *idiomatically expressed in Java's own type system and standard library conventions*:
- **Creational** — `Builder` (fluent method-chaining constructors, common for classes with many optional parameters, avoiding "telescoping constructor" anti-patterns); static factory methods (`List.of(...)`, preferred in many modern Java APIs over public constructors, since a factory method can return a cached/specialized implementation and has a more descriptive name than an overloaded constructor); `Singleton` (classically an `enum` with a single instance, since Java enums are inherently serialization-safe and reflection-attack-resistant in a way a manually-written singleton class isn't).
- **Structural** — `Adapter`/`Decorator` are directly visible throughout the I/O streams API (section 15) itself — a `BufferedReader` *decorates* a `Reader`, adding buffering behavior without changing the underlying `Reader` interface, a textbook Decorator-pattern usage baked directly into the JDK's own standard library design.
- **Behavioral** — `Iterator` is a first-class Java language feature (the `Iterable`/`Iterator` interfaces powering every for-each loop); `Strategy` is directly realized through functional interfaces (section 13) — passing a `Comparator` or any other functional interface as a parameter *is* the Strategy pattern, made lightweight and idiomatic by lambda syntax rather than requiring a full separate class hierarchy the way pre-Java-8 Strategy implementations needed.

**Where the JDK itself uses each pattern internally** — recognizing that `Collections.unmodifiableList(...)` is a Decorator, that `Stream`'s pipeline construction is a form of Builder, and that `Optional` (section 13) functions similarly to a minimal Null Object pattern implementation, builds genuine pattern-recognition fluency that transfers directly to reading and reasoning about unfamiliar Java library/framework code, not just to writing new code from scratch.

---

## 19. Build Tools, Testing & Tooling

**Maven vs Gradle (recap, brief)** — full detail lives in the Java Backend notes' section 14; the short version: Maven uses declarative XML (`pom.xml`) with a fixed, convention-driven build lifecycle; Gradle uses a Groovy or Kotlin DSL build script, offering more flexibility and generally faster incremental builds (closer in philosophy to the multi-module Gradle setup already covered in the Android notes' section 16) — both are dependency-management-plus-build-orchestration tools serving the same fundamental role.

**JUnit 5 fundamentals (recap)** — full detail also lives in the Java Backend notes' section 13; JUnit remains the standard JVM testing framework, with `@Test`, assertions (`assertEquals`, `assertThrows`), and lifecycle annotations (`@BeforeEach`/`@AfterEach`) forming the same core testing vocabulary already covered there.

**JVM tuning flags, profiling tools overview — Definition:** common JVM flags control heap sizing (`-Xms`/`-Xmx`), GC algorithm selection (`-XX:+UseG1GC`, section 10), and diagnostics (`-Xlog:gc` for GC logging); profiling tools — **JVisualVM**, **JConsole** (both bundled with the JDK), and more advanced commercial/open-source profilers (async-profiler, JProfiler) — attach to a running JVM to observe heap usage, thread activity, and CPU hotspots directly, the JVM-specific application of the general profiling discipline already covered across this workspace's various performance-optimization sections (Node.js, Python Backend, Game Development notes).

---

## 20. Java Interview Prep

**Common core-Java interview questions** — explain the `equals()`/`hashCode()` contract and what breaks it (section 5); what is type erasure and what does it actually prevent you from doing (section 6); walk through how `HashMap` resolves collisions, including treeification (section 7); explain checked vs unchecked exceptions and the design debate around them (section 8); why is `String` immutable, and what is the String Pool (section 9); explain the difference between `synchronized` and `volatile`, and why `volatile` alone doesn't make `count++` thread-safe (section 11); what are virtual threads and what specific problem do they solve (section 14); explain PECS and give an example of when you'd use `? extends` vs `? super` (section 6).

**Java vs Kotlin (recap Android notes) — Definition:** as already covered from Kotlin's side in the Android notes' section 1, Kotlin (fully interoperable with Java, both compiling to JVM bytecode) adds compile-time-enforced null safety, more concise syntax (data classes are Kotlin's pre-Records answer to section 14's Records, arriving years earlier), and first-class coroutines (a different, more deeply language-integrated concurrency model than Java's virtual threads, section 14, though both target the same "don't force developers into complex async composition just to get scalable concurrency" goal) — Java's own recent evolution (Records, pattern matching, virtual threads, sections 14) has directly narrowed much of the conciseness/safety gap that originally motivated Kotlin's adoption, without Java sacrificing its own backward compatibility guarantees in the process.

**Where this file hands off to the Java Backend notes — Definition:** everything covered here (sections 1–19) is the language/JVM foundation; the Java Backend notes pick up directly from here into Spring Framework fundamentals, building REST APIs, Spring Data/JPA, Spring Security, and production/microservices engineering — read this file first when core-Java fluency itself is the gap, and the Java Backend notes when the goal is specifically building production backend systems on top of that foundation.
