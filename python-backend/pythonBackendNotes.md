# Python Backend Development — Deep Dive Roadmap

We'll go from language fundamentals → language internals → async Python → web frameworks → data access → testing → production → interview problems.

---

## 1. Python Language Fundamentals

**Definition:** Python is a high-level, interpreted, dynamically-typed, multi-paradigm programming language (supports procedural, object-oriented, and functional styles), known for readable syntax and a "batteries included" standard library — the dominant reference implementation is **CPython**, which is what "Python" means unless otherwise specified.

**The CPython interpreter & `.pyc` bytecode (brief) — Definition:** CPython compiles Python source into an intermediate **bytecode** representation (cached as `.pyc` files, so re-running unchanged code skips recompilation), which the CPython **virtual machine** then interprets — conceptually parallel to the Java notes' bytecode/JVM model, though CPython's interpreter (unlike the JVM's JIT, Java notes' section 5) does not compile hot code to native machine code by default, which is one structural reason Python is generally slower than Java/compiled languages for CPU-bound work (mitigated by PyPy, an alternative JIT-compiling Python implementation, or by delegating hot paths to C extensions/native libraries like NumPy).

**Variables & dynamic typing — Definition:** Python variables are not declared with a fixed type — a name is simply bound to an object, and can be rebound to an object of a completely different type later (`x = 5; x = "hello"` is valid) — type checking happens at runtime, not compile time (contrast with Java's/TypeScript's static typing, though see section 9 for Python's optional type-hint system).

**Data types — Definition:** `int` (arbitrary precision — Python integers don't overflow the way fixed-width integers in other languages do), `float`, `str` (immutable, like Java's `String`), `bool`, `None` (Python's null/absence value, a singleton — always compared with `is None`, not `== None`, by convention).

**Mutable vs immutable types — Definition:** immutable types (`int`, `float`, `str`, `tuple`, `frozenset`) cannot be changed after creation — any "modification" creates a new object; mutable types (`list`, `dict`, `set`) can be changed in place — this distinction has real consequences for default function arguments (section 3) and for using a value as a dictionary key (only hashable, generally immutable, types can be dict keys/set members).

**String formatting — Definition:** `f-strings` (Python 3.6+, `f"Hello, {name}!"`) are the modern, preferred approach — evaluated at runtime, support arbitrary expressions inline, and are generally the fastest of the three; `.format()` (`"Hello, {}!".format(name)`) is the older, still-common method-based approach; `%`-formatting (`"Hello, %s!" % name`) is the oldest, printf-style approach, now largely legacy.

**Truthy & falsy values — Definition:** every value has an implicit boolean interpretation; the falsy values are `False`, `None`, `0`, `0.0`, `''` (empty string), and any empty collection (`[]`, `{}`, `()`, `set()`) — conceptually the same falsy-value list philosophy as JavaScript (JS/TS notes' section 1), though the specific values differ slightly.

**The `if __name__ == '__main__':` idiom — Definition:** every Python module has a `__name__` attribute, automatically set to `'__main__'` only when that file is run **directly** (not when it's imported by another module) — this idiom guards code that should only execute when the file is run as a script, letting the same file be both an importable module (for its functions/classes) and a standalone runnable script.

---

## 2. Core Data Structures

**Lists — Definition:** an ordered, **mutable** sequence, allowing duplicates and mixed types — Python's most commonly-used general-purpose collection, roughly analogous to a JS array or a Java `ArrayList`.

```python
nums = [1, 2, 3]
nums.append(4)      # O(1) amortized, same dynamic-array resizing behavior as the DSA/JS notes
nums[0]              # O(1) index access
```

**Tuples — Definition:** an ordered, **immutable** sequence — used for fixed-size groupings of values that shouldn't change (e.g. coordinates `(x, y)`), and (being hashable, since immutable) usable as dictionary keys, unlike lists.

**Dictionaries — Definition:** a key-value mapping, implemented as a hash table (O(1) average lookup/insert/delete — same hash table concept as the DSA notes' section 5) — since Python 3.7, dictionaries maintain **insertion order** as a language guarantee (not just an implementation detail).

```python
user = {'name': 'Ada', 'age': 30}
user.get('email', 'unknown')  # returns a default instead of raising KeyError if the key is absent
```

**Sets — Definition:** an unordered collection of unique, hashable elements — O(1) average membership testing, and supports mathematical set operations (`|` union, `&` intersection, `-` difference) directly.

**Comprehensions — Definition:** a concise, declarative syntax for building a list/dict/set from an iterable, optionally filtered — Python's equivalent of the JS/TS notes' `.map()`/`.filter()` chains, expressed as a single inline expression rather than chained method calls.

```python
squares = [n ** 2 for n in range(10)]                    # list comprehension
evens_only = [n for n in range(10) if n % 2 == 0]          # with a filter condition
squares_dict = {n: n ** 2 for n in range(5)}                # dict comprehension
unique_lengths = {len(word) for word in words}               # set comprehension
```

**Slicing — Definition:** `sequence[start:stop:step]` extracts a sub-portion of any sequence type (list, tuple, string) — omitted indices default to the start/end, and a negative `step` reverses direction (`s[::-1]` reverses a sequence, a common idiom).

**Mutability implications — the default mutable argument pitfall — Definition:** a mutable default argument value (`def f(items=[])`) is evaluated **once**, at function-definition time, and that **same object is reused across every call** that doesn't explicitly supply the argument — a notorious, easy-to-hit Python gotcha, since it looks like each call should get a fresh empty list.

```python
# ❌ BUG — the SAME list object is shared across every call using the default
def add_item(item, items=[]):
    items.append(item)
    return items
add_item('a')  # ['a']
add_item('b')  # ['a', 'b'] — NOT ['b'], because it's the same list from before!

# ✅ correct pattern
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

**`collections` module — Definition:** the standard library module providing specialized container types beyond the built-ins: `defaultdict` (a dict that auto-creates a default value for a missing key, avoiding manual `if key not in dict` checks); `Counter` (a dict subclass for counting hashable items, with a convenient `.most_common()` method); `OrderedDict` (largely redundant since plain dicts guarantee insertion order, but still useful for its explicit `.move_to_end()` method, e.g. implementing an LRU cache, DSA notes' section 4); `namedtuple` (a lightweight, immutable class-like tuple with named fields, largely superseded by `dataclasses`, section 4, for new code); `deque` (a double-ended queue with O(1) append/pop from both ends, unlike a plain list's O(n) `insert(0, ...)`/`pop(0)` — the correct choice for stack/queue use cases, same rationale as the DSA notes' section 4).

```python
from collections import defaultdict, Counter

word_count = defaultdict(int)
for word in text.split(): word_count[word] += 1

Counter(text.split()).most_common(3) # top 3 most frequent words
```

---

## 3. Functions — Deep Dive

**Default arguments (recap)** — see section 2's mutability pitfall; otherwise straightforward: `def greet(name, greeting='Hello'):`.

**`*args` and `**kwargs` — Definition:** `*args` collects any number of extra **positional** arguments into a tuple; `**kwargs` collects any number of extra **keyword** arguments into a dict — Python's equivalent of the JS/TS notes' rest parameters, but split into positional vs keyword collection explicitly.

```python
def log(*args, **kwargs):
    print(args)    # tuple of positional args
    print(kwargs)  # dict of keyword args

log(1, 2, name='Ada') # args=(1, 2), kwargs={'name': 'Ada'}
```

**First-class functions — Definition:** functions in Python are objects like any other value — assignable to variables, passable as arguments, returnable from other functions — the same first-class-function concept already covered in the JS/TS notes' section 6, underlying decorators (below) and Python's own functional-programming capabilities.

**Lambda functions — Definition:** anonymous, single-expression functions (`lambda x, y: x + y`) — more restrictive than a JS arrow function (limited to a single expression, no statements/multiple lines) — typically used inline for short callback-style arguments (e.g. a `sort()` key function) rather than as a general-purpose function-definition tool.

**Closures — Definition:** the same fundamental concept as the JS/TS notes' section 3 — a nested function retains access to variables from its enclosing scope even after that enclosing function has returned. Python closures are **read-only** by default for enclosing-scope variables unless explicitly declared `nonlocal`.

```python
def make_counter():
    count = 0
    def increment():
        nonlocal count  # without this, `count += 1` would raise UnboundLocalError
        count += 1
        return count
    return increment
```

**Decorators — Definition:** a function that takes another function (or class) as input and returns a modified/wrapped version of it — Python's native, first-class syntax (`@decorator_name`) for the same Decorator pattern already covered in the JS/TS and Java notes, and structurally similar to Express/Node.js middleware or Spring's AOP-based annotations (`@Transactional`).

```python
import functools

def log_calls(func):
    @functools.wraps(func)  # preserves the original function's __name__/docstring — omitting this is a common bug
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@log_calls
def add(a, b): return a + b
```

**Decorators with arguments — Definition:** to make a decorator itself configurable (`@retry(times=3)`), an extra outer function layer is needed — the decorator factory returns the actual decorator, which returns the wrapper.

```python
def retry(times):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(times):
                try: return func(*args, **kwargs)
                except Exception:
                    if attempt == times - 1: raise
            return None
        return wrapper
    return decorator

@retry(times=3)
def fetch_data(): ...
```

**`functools` utilities — Definition:** `lru_cache` (a built-in memoization decorator — same memoization concept as the DSA notes' section 13, caching a function's results by its arguments automatically); `partial` (fixes some of a function's arguments upfront, returning a new callable — Python's equivalent of the JS/TS notes' partial application); `reduce` (folds an iterable down to a single accumulated value, the same operation as JS's `.reduce()`, but as a standalone function rather than a method).

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)  # naive recursive fib made fast via memoization
```

---

## 4. Object-Oriented Programming in Python

**Classes & instances — Definition:** a class is a blueprint; `__init__` is the constructor-equivalent method, called automatically when a new instance is created via `ClassName(...)`, used to set up initial instance state.

```python
class User:
    def __init__(self, name, email):
        self.name = name    # instance attribute
        self.email = email
```

**`self` — Definition:** the explicit first parameter of every instance method, referring to the instance the method was called on — unlike Java/JS's implicit `this`, Python requires `self` to be an explicit parameter in every method definition (though it's passed automatically by Python when calling `instance.method()`).

**Class attributes vs instance attributes — Definition:** a **class attribute** (defined directly in the class body, not inside `__init__`) is shared across **all** instances of the class; an **instance attribute** (typically set via `self.x = ...` in `__init__`) is unique per instance — mutating a mutable class attribute (a class-level list, for example) affects every instance, a common source of the same kind of shared-mutable-state bug as section 2's default-argument pitfall.

**Method types — Definition:**
- **Instance method** (default) — takes `self`, operates on a specific instance.
- **`@classmethod`** — takes `cls` (the class itself) instead of `self`, commonly used for alternative constructors (`User.from_dict(data)`).
- **`@staticmethod`** — takes neither `self` nor `cls` — a plain function namespaced within the class purely for organizational grouping, with no access to instance/class state at all.

```python
class User:
    def __init__(self, name): self.name = name

    @classmethod
    def from_dict(cls, data): return cls(data['name'])  # alternative constructor

    @staticmethod
    def is_valid_email(email): return '@' in email       # doesn't need self or cls
```

**Inheritance & MRO — Definition:** `class Dog(Animal):` — a subclass inherits its parent's attributes/methods; when a class has multiple ancestors (via multiple inheritance, below), Python resolves which method actually gets called via the **Method Resolution Order (MRO)** — a deterministic linearization (using the **C3 linearization algorithm**) of the entire inheritance graph, inspectable via `ClassName.__mro__` or `ClassName.mro()`.

**Multiple inheritance — Definition:** unlike Java (single inheritance of classes only, section 2 of the Java notes), Python classes can inherit directly from multiple parent classes simultaneously (`class C(A, B):`) — powerful but can create genuine ambiguity/complexity (the "diamond problem," where two parents share a common ancestor) resolved deterministically by the MRO, but still worth using cautiously — **mixins** (small, focused classes designed specifically to be combined via multiple inheritance, each contributing one specific capability) are the idiomatic, disciplined way to use this feature well.

**Magic/dunder methods — Definition:** methods with double-underscore names (`__str__`, `__repr__`, `__eq__`, `__len__`, `__add__`) that Python's built-in syntax/functions call implicitly — implementing them lets a custom class integrate with native language operators/functions.

```python
class Point:
    def __init__(self, x, y): self.x, self.y = x, y
    def __repr__(self): return f"Point({self.x}, {self.y})"      # used by repr(), and in the REPL/debugger
    def __eq__(self, other): return self.x == other.x and self.y == other.y  # enables `==`
    def __add__(self, other): return Point(self.x + other.x, self.y + other.y)  # enables `+`
```

**Properties (`@property`) — Definition:** turns a method into an attribute-like accessor, called without parentheses (`obj.value` instead of `obj.get_value()`) — the same getter/setter concept as the JS/TS/Java notes, letting you add computed-value or validation logic behind what looks like plain attribute access from the caller's side, without changing the public API if a plain attribute is later upgraded to computed logic.

```python
class Circle:
    def __init__(self, radius): self._radius = radius
    @property
    def area(self): return 3.14159 * self._radius ** 2       # computed, read-only "attribute"
    @property
    def radius(self): return self._radius
    @radius.setter
    def radius(self, value):
        if value < 0: raise ValueError("radius cannot be negative")
        self._radius = value
```

**`dataclasses` — Definition:** the `@dataclass` decorator (Python 3.7+) automatically generates `__init__`, `__repr__`, and `__eq__` from declared class-level type-annotated fields — eliminating the exact boilerplate for a simple data-carrying class that Java's records (Java notes' section 7) also eliminate.

```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int
    email: str = ""  # default value
```

**Abstract base classes (`abc` module) — Definition:** `ABC` and `@abstractmethod` let you define a class that cannot be instantiated directly and requires subclasses to implement specific methods — Python's equivalent of Java's abstract classes (Java notes' section 2), used to enforce a contract across a family of related subclasses.

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self): ...

class Circle(Shape):
    def area(self): return 3.14159 * self.radius ** 2  # MUST implement, or Circle also can't be instantiated
```

**Duck typing — Definition:** Python's dynamically-typed nature means code often doesn't check an object's actual class at all — it just calls the method/attribute it needs and trusts it will work ("if it walks like a duck and quacks like a duck") — the same structural-typing philosophy already covered in the TypeScript notes' section 16, but enforced only at **runtime** in plain Python (no compile-time structural check exists, unless you're using type hints + a type checker, section 9).

---

## 5. Iterators, Generators & Context Managers

**The iterator protocol — Definition:** an object is **iterable** if it implements `__iter__` (returning an iterator); an object is an **iterator** if it implements `__next__` (returning the next value, raising `StopIteration` when exhausted) — the same two-part protocol already covered in the JS/TS notes' section 9, expressed via dunder methods instead of `Symbol.iterator`/`.next()`.

**Generator functions (`yield`) — Definition:** a function containing `yield` becomes a generator — calling it doesn't execute the body immediately, it returns a generator object; each call to `next()` (or each iteration in a `for` loop) resumes execution from where it last `yield`ed, until the function returns or is exhausted — the same pause/resume, lazy-sequence concept as the JS/TS notes' section 9.

```python
def id_generator():
    n = 1
    while True:
        yield n  # pauses here, resumes on the next next() call
        n += 1

gen = id_generator()
next(gen)  # 1
next(gen)  # 2
```

**Generator expressions — Definition:** a lazily-evaluated version of a list comprehension, using parentheses instead of brackets — computes each value only as it's requested, rather than building the entire list upfront in memory — the standard, idiomatic choice when a comprehension's full result doesn't actually need to exist as a complete list all at once (e.g. feeding directly into `sum()` or a `for` loop).

```python
total = sum(n ** 2 for n in range(1_000_000))  # never builds a million-element list in memory
```

**`yield from` — Definition:** delegates iteration to a sub-generator/sub-iterable, yielding each of its values in turn, without an explicit inner loop — commonly used to flatten/compose generators, or (its original motivating use case) to delegate to a sub-coroutine in older, pre-`async`/`await` asyncio code (section 8).

```python
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)  # recursively delegate
        else:
            yield item
```

**Context managers (`with` statement) — Definition:** a construct guaranteeing a setup action runs before a block and a corresponding cleanup action runs after it — **even if an exception occurs inside the block** — Python's equivalent of the Node.js/Java notes' try-with-resources/`AutoCloseable`, most commonly used for reliably closing files/connections.

```python
with open('data.txt') as f:  # file automatically closed when the block exits, even on exception
    content = f.read()
```

**`__enter__`/`__exit__` — Definition:** implementing these two dunder methods makes a class usable in a `with` statement — `__enter__` runs at block entry (its return value is what `as` binds to); `__exit__` runs at block exit, receiving any exception info that occurred (and can suppress it by returning `True`).

```python
class DatabaseConnection:
    def __enter__(self):
        self.conn = connect()
        return self.conn
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.conn.close()
        return False  # False = don't suppress any exception that occurred
```

**`contextlib` (`@contextmanager`) — Definition:** a decorator that lets you write a context manager as a single generator function instead of a full class with `__enter__`/`__exit__` — the code before `yield` is the setup, the code after is the teardown, with `yield` itself marking where the `with` block's body executes.

```python
from contextlib import contextmanager

@contextmanager
def db_connection():
    conn = connect()
    try:
        yield conn
    finally:
        conn.close()
```

---

## 6. Error Handling

**`try`/`except`/`else`/`finally` — Definition:** `try` wraps risky code; `except` handles a specific (or general) exception type; `else` runs only if **no** exception occurred in the `try` block (a Python-specific addition beyond the standard try/catch/finally most languages have); `finally` always runs regardless of outcome.

```python
try:
    result = risky_operation()
except ValueError as e:
    print(f"Invalid value: {e}")
except (TypeError, KeyError) as e:  # catch multiple exception types with one handler
    print(f"Error: {e}")
else:
    print(f"Success: {result}")  # only runs if the try block succeeded
finally:
    cleanup()
```

**Exception hierarchy — Definition:** `BaseException` is the root of everything, including `SystemExit`/`KeyboardInterrupt` (deliberately **not** under `Exception`, so a broad `except Exception:` doesn't accidentally swallow a user's Ctrl+C or a deliberate `sys.exit()`); `Exception` is the base for essentially all "normal" catchable errors an application should handle — a bare `except:` (catching literally everything, including `BaseException`) is considered a serious anti-pattern for exactly this reason.

**Custom exceptions — Definition:** subclassing `Exception` (Python has no separate checked/unchecked exception distinction like Java — every exception is effectively "unchecked," with no compiler enforcement to catch/declare it) to create domain-specific error types.

```python
class InsufficientFundsError(Exception):
    def __init__(self, balance, amount):
        super().__init__(f"Cannot withdraw {amount}, balance is {balance}")
        self.balance = balance
        self.amount = amount
```

**`raise ... from` — Definition:** explicitly chains a new exception to the original one that caused it, preserving both tracebacks — Python's equivalent of the Node.js/JS notes' `Error.cause` and the Java notes' exception chaining, giving full causal context when wrapping a lower-level error in a more meaningful higher-level one.

```python
try:
    parse_config()
except json.JSONDecodeError as e:
    raise ConfigError("Invalid configuration file") from e
```

**EAFP vs LBYL — Definition:** two contrasting philosophies for handling conditions that might fail. **LBYL** ("Look Before You Leap") checks preconditions explicitly before acting (`if key in dict: value = dict[key]`); **EAFP** ("Easier to Ask Forgiveness than Permission") just attempts the operation and catches the exception if it fails (`try: value = dict[key] except KeyError: ...`) — Python idiom strongly favors **EAFP**, partly because a separate check-then-act sequence can itself have a race condition in concurrent code (the state could change between the check and the act), and partly because it's considered more readable/Pythonic for the common case.

---

## 7. Python Internals

A major deep-dive topic.

**The GIL (Global Interpreter Lock) — Definition:** a mutex in CPython ensuring only **one thread executes Python bytecode at a time**, even on a multi-core machine — exists because CPython's memory management (reference counting, below) is not thread-safe by default, and the GIL was the original, simpler solution to that problem rather than making every object's internals individually thread-safe.

**Implications for CPU-bound vs I/O-bound code — Definition:** the GIL means Python **threads do not achieve true parallelism for CPU-bound work** — multiple threads computing simultaneously on separate cores gets no real speedup, since only one can hold the GIL and execute Python bytecode at any instant. However, the GIL is **released** during I/O operations (network calls, file reads, `time.sleep()`) and by certain C-extension code (e.g. NumPy's core numeric operations) — so threading **does** provide real concurrency benefit for I/O-bound work (many threads can be simultaneously *waiting* on I/O, even though only one executes actual Python bytecode at a time) — this single distinction drives the threading-vs-multiprocessing-vs-asyncio decision in section 8.

*(Note: Python 3.13 introduced an experimental, opt-in "free-threaded" build without a GIL — a significant, actively-evolving development, but not yet the default/universal CPython behavior as of this writing.)*

**Memory management — Definition:**
- **Reference counting — Definition:** every Python object tracks how many references currently point to it; when that count drops to zero, the object is **immediately** deallocated — unlike a generational garbage collector (Node.js/Java notes) that runs periodically, most Python memory is reclaimed deterministically and immediately the instant it becomes unreachable.
- **Cyclic garbage collection — Definition:** reference counting alone cannot detect **reference cycles** (object A references B, B references A, but nothing external references either — their count never reaches zero) — Python's separate, periodically-run cyclic garbage collector specifically detects and reclaims these otherwise-unreachable cycles, complementing reference counting rather than replacing it.

**CPython object model — Definition:** in CPython, **everything is an object** — every `int`, `function`, `class`, and `module` is itself an instance of some type, uniformly, with no separate "primitive" category the way Java has (Java notes' section 1) — this uniformity is part of why introspection (`type()`, `dir()`, `isinstance()`) works so consistently across every kind of value in Python.

**Namespaces & scope (LEGB rule) — Definition:** when a name is referenced, Python resolves it by searching, in order: **L**ocal (the current function's scope) → **E**nclosing (any enclosing function's scope, for closures) → **G**lobal (the current module's top level) → **B**uilt-in (Python's built-in names like `len`, `print`) — the same lexical-scope-chain-walk concept as the JS/TS notes' section 2, with Python's specific four-tier naming.

**`is` vs `==` — Definition:** `is` compares **identity** (are these two names bound to the literal same object in memory?); `==` compares **value equality** (calling `__eq__`, potentially custom-defined, section 4) — the same distinction as Java's `==` vs `.equals()` (Java notes' section 1), and directly analogous to (though textually opposite naming from) JS's `===` reference-vs-value distinction on objects. `is` should be used specifically for singleton comparisons (`x is None`, `x is True`), never as a general substitute for `==`.

---

## 8. Concurrency & Parallelism

**Threading (`threading` module) — Definition:** creates OS-level threads within a single process, sharing memory — subject to the GIL (section 7), so genuinely useful primarily for **I/O-bound** concurrency (many threads simultaneously waiting on network/disk), not CPU-bound parallelism.

```python
import threading

def worker(n): print(f"Processing {n}")
threads = [threading.Thread(target=worker, args=(i,)) for i in range(5)]
for t in threads: t.start()
for t in threads: t.join()
```

**Multiprocessing (`multiprocessing` module) — Definition:** spawns entirely separate **OS processes**, each with its own Python interpreter and its own GIL — genuinely achieves parallel execution across multiple CPU cores for **CPU-bound** work, at the cost of higher memory overhead (each process has its own full memory space) and the need for explicit inter-process communication (pickling data across process boundaries, rather than freely sharing memory the way threads do).

```python
from multiprocessing import Pool

def cpu_heavy_task(n): return sum(i * i for i in range(n))

with Pool(processes=4) as pool:
    results = pool.map(cpu_heavy_task, [10_000_000] * 4) # genuinely parallel across 4 cores
```

**Threading vs multiprocessing (GIL implications, recap)** — the single most important Python concurrency decision: **I/O-bound → threading (or asyncio, below) — CPU-bound → multiprocessing** (or delegate the CPU-heavy portion to a C-extension library like NumPy, which releases the GIL internally).

**`concurrent.futures` — Definition:** a higher-level, unified abstraction over both threading and multiprocessing — `ThreadPoolExecutor` and `ProcessPoolExecutor` share the same simple `.submit()`/`.map()` API, letting code switch between thread-based and process-based concurrency by changing just the executor class, without restructuring the calling code.

```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=5) as executor:
    results = list(executor.map(fetch_url, urls))
```

**Async I/O fundamentals — Definition:** `asyncio` provides **single-threaded concurrency** via **cooperative multitasking** — an event loop (conceptually the same event-loop model as the Node.js notes' section 2) runs many **coroutines**, each voluntarily yielding control back to the loop at `await` points (typically I/O waits), letting the loop run other coroutines in the meantime — achieving high I/O concurrency without threads at all, and therefore without any GIL-contention concern between them (only one coroutine's Python code ever runs at a literal instant, but none of them ever sit idle blocking on I/O).

**Coroutines (`async`/`await`) — Definition:** an `async def` function is a coroutine — calling it doesn't execute the body immediately, it returns a coroutine object that must be `await`ed (or scheduled as a Task, below) to actually run — structurally the same `async`/`await` syntax and mental model as the JS/TS notes' section 7, though Python's underlying event-loop/coroutine machinery is distinct from JS's.

```python
import asyncio

async def fetch_user(user_id):
    await asyncio.sleep(1)  # simulates an async I/O wait, yields control to the event loop
    return {"id": user_id}

async def main():
    user = await fetch_user(1)
    print(user)

asyncio.run(main())  # entry point — creates and runs the event loop
```

**`asyncio.gather` — Definition:** runs multiple coroutines **concurrently** (not sequentially), waiting for all of them to complete and returning their results in order — the asyncio equivalent of the JS/TS notes' `Promise.all` and the Java notes' `CompletableFuture` combinators.

```python
async def main():
    user, orders = await asyncio.gather(fetch_user(1), fetch_orders(1)) # both run concurrently, not sequentially
```

**Tasks — Definition:** `asyncio.create_task(coro())` schedules a coroutine to start running **immediately**, concurrently with the rest of the current code, returning a `Task` handle that can be `await`ed later for its result — distinct from just calling `await coro()` directly, which runs and waits for that one coroutine before continuing (the same "did I accidentally serialize independent async work" pitfall already covered in the JS/TS notes' section 7, applies identically here).

**Async context managers/iterators — Definition:** `async with`/`async for` are the asynchronous counterparts of `with`/`for`, used with objects implementing `__aenter__`/`__aexit__` or `__aiter__`/`__anext__` — needed when the setup/teardown or iteration step itself must perform an `await`able operation (e.g. an async database connection pool's connection acquisition).

**When to use threading vs multiprocessing vs asyncio — Definition:**

| Workload | Best choice |
|---|---|
| CPU-bound (heavy computation) | `multiprocessing` — the only option that achieves true parallelism given the GIL |
| I/O-bound, moderate concurrency, existing sync/blocking libraries | `threading` (or `concurrent.futures.ThreadPoolExecutor`) |
| I/O-bound, very high concurrency (thousands of connections), async-native libraries available | `asyncio` — lowest overhead per concurrent operation, but requires the whole call chain to be async-aware |

---

## 9. Type Hints & Static Typing

**Definition:** type hints (PEP 484, Python 3.5+) let you optionally annotate variables, function parameters, and return values with expected types — Python remains **dynamically typed at runtime** regardless of hints (they are **not enforced** by the interpreter itself) — hints exist purely for tooling: editor autocomplete, and static type checkers (below) that analyze code *before* running it, the same core value proposition as TypeScript (JS/TS notes' section 15), but as an optional annotation layer on an otherwise-untyped language rather than a full compiled superset.

```python
def greet(name: str, times: int = 1) -> str:
    return (f"Hello, {name}! ") * times
```

**`Optional`, `Union`, `Any` — Definition:** `Optional[str]` (equivalent to `Union[str, None]`) signals a value might be `None`; `Union[int, str]` signals a value could be either type — both directly parallel the TypeScript notes' union types (section 16); `Any` opts a value out of type checking entirely — the direct Python equivalent of TypeScript's `any` (JS/TS notes' section 15), with the same associated disadvantage of silently disabling type safety for that value.

```python
from typing import Optional, Union

def find_user(user_id: int) -> Optional[dict]: ...
value: Union[int, str] = 5

# Python 3.10+ shorthand syntax:
def find_user(user_id: int) -> dict | None: ...
```

**Generics (`TypeVar`, `Generic`) — Definition:** the same generics concept as the TypeScript/Java notes (JS/TS notes' section 17, Java notes' section 3) — parameterizing a function/class over a type, preserving type-safety while remaining reusable across many concrete types.

```python
from typing import TypeVar, Generic

T = TypeVar('T')

class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []
    def push(self, item: T) -> None: self._items.append(item)
    def pop(self) -> T: return self._items.pop()
```

**`Protocol` (structural typing) — Definition:** defines a structural (duck-typed, section 4) interface that any class satisfying its shape automatically conforms to, **without needing to explicitly inherit from it** — Python's typing-system equivalent of TypeScript's structural typing (JS/TS notes' section 16), formalizing Python's existing duck-typing philosophy into something a static type checker can actually verify.

```python
from typing import Protocol

class Comparable(Protocol):
    def __lt__(self, other) -> bool: ...

def sort_items(items: list[Comparable]) -> list[Comparable]:
    return sorted(items) # accepts ANY class implementing __lt__, no explicit inheritance needed
```

**`TypedDict` — Definition:** describes the expected shape of a plain `dict` — the exact keys it should have and each value's type — used when working with dict-shaped data (e.g. parsed JSON) that you want type-checked without wrapping it in a full class/dataclass.

```python
from typing import TypedDict

class UserDict(TypedDict):
    name: str
    age: int
```

**Type checking tools (mypy, pyright) — Definition:** static analyzers that read type hints and flag inconsistencies **before** the code ever runs — `mypy` is the original, most widely-adopted checker; `pyright` (from Microsoft, also powering VS Code's Python type checking) is generally faster and increasingly popular — neither is built into the Python interpreter itself; both are separate tools run in CI/editor tooling, since (as noted above) Python's runtime completely ignores type hints on its own.

**Pydantic — Definition:** a library that goes a critical step further than plain type hints — it performs **runtime validation and parsing** based on type-annotated model classes, raising a clear validation error if actual data doesn't match the declared shape/types — since plain type hints provide zero runtime safety on their own (identical caveat to the TypeScript notes' section 15's point about `any`/external data), Pydantic is the standard way to get real runtime guarantees at a system boundary (an incoming API request body, a config file), and is the foundation FastAPI (section 12) builds its entire request-validation story on.

```python
from pydantic import BaseModel, EmailStr

class CreateUserRequest(BaseModel):
    name: str
    email: EmailStr
    age: int

user = CreateUserRequest(**request_json)  # raises ValidationError if the data doesn't match
```

---

## 10. Modules, Packages & Environments

**Modules vs packages — Definition:** a **module** is a single `.py` file; a **package** is a directory of modules, historically requiring an `__init__.py` file to be recognized as a package (Python 3.3+ also supports "namespace packages" without one, in specific scenarios) — organizing related modules under one importable namespace.

**Import system — Definition:** `import module` imports the whole module (accessed as `module.thing`); `from module import thing` imports a specific name directly into the current namespace — the same default-vs-named-export distinction conceptually as the JS/TS notes' section 10, though Python has no separate "default export" concept — every name is just a named attribute of the module.

**Absolute vs relative imports — Definition:** an **absolute** import specifies the full path from the project's top-level package (`from myapp.services.user import UserService`); a **relative** import specifies a path relative to the current module's location (`from .user import UserService`, `from ..config import settings`) — absolute imports are generally preferred for clarity in application code, with relative imports more common within a package's own internal cross-references.

**`__init__.py` — Definition:** the file marking a directory as a Python package, run automatically when that package is first imported — commonly used to control what's exposed at the package's top level (re-exporting specific names from submodules) or left empty when no such control is needed.

**Virtual environments (`venv`) — Definition:** an isolated Python installation (its own `site-packages` directory) created per-project, so each project's dependencies (and their specific versions) don't conflict with another project's, or with the system-wide Python installation — the standard practice for essentially every real Python project, analogous in purpose to a `node_modules` directory (Node.js notes' section 1), though Python's isolation happens at the interpreter/environment level rather than per-project dependency folders.

```bash
python -m venv .venv
source .venv/bin/activate   # macOS/Linux
.venv\Scripts\activate      # Windows
```

**Dependency management — Definition:**
- **`pip` & `requirements.txt`** — the traditional approach: `pip install package` installs a package into the active virtual environment; `pip freeze > requirements.txt` snapshots exact installed versions for reproducibility, `pip install -r requirements.txt` reinstalls them elsewhere — simple, but `requirements.txt` doesn't natively distinguish direct dependencies from transitive ones, or separate dev-only dependencies, without extra convention/tooling.
- **Poetry — Definition:** a modern, all-in-one dependency management and packaging tool, using a `pyproject.toml` file (below) and a lock file (`poetry.lock`, guaranteeing fully reproducible installs across machines — the same lock-file concept as npm's `package-lock.json`) — handles virtual environment creation, dependency resolution, and package publishing in one coherent tool, addressing several of plain pip's gaps.

**`pyproject.toml` — Definition:** the modern, standardized (PEP 518/621) configuration file for a Python project — project metadata, dependencies, and build-system configuration in one TOML file, increasingly the standard replacing the older combination of `setup.py`/`requirements.txt`/tool-specific config files scattered across a project.

**Publishing a package (brief)** — a Python package is published to **PyPI** (the Python Package Index, the ecosystem's equivalent of npm's registry) via a build step (`python -m build`, or Poetry's built-in `poetry publish`) producing a distributable wheel/sdist file, uploaded with `twine` or the tool's own publish command.

---

## 11. Web Frameworks — Flask

A microframework deep-dive.

**Definition:** Flask is a **microframework** — deliberately minimal and unopinionated, providing routing and request/response handling as its core, with everything else (database access, authentication, validation) left to be added via separate extensions/libraries as needed, rather than bundled in — the philosophical opposite of Django's "batteries included" approach (section 13).

**Routes & view functions:**

```python
from flask import Flask, jsonify, request

app = Flask(__name__)

@app.route('/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    user = find_user(user_id)
    return jsonify(user)

@app.route('/users', methods=['POST'])
def create_user():
    data = request.get_json()
    user = save_user(data)
    return jsonify(user), 201
```

**Request/response objects — Definition:** Flask's global `request` object (thread-local, automatically scoped to the current request being handled) exposes `.json`/`.args`(query params)/`.form`/`.headers`; view functions return either a plain value (auto-wrapped into a response), a `(body, status_code)` tuple, or an explicit `Response` object for full control.

**Blueprints — Definition:** Flask's mechanism for organizing routes into reusable, mountable modules — analogous to Express's `Router` (Node.js notes' section 4) — letting a large application's routes be split across multiple files/logical groupings, each registered onto the main app.

```python
from flask import Blueprint
users_bp = Blueprint('users', __name__, url_prefix='/api/users')

@users_bp.route('/<int:user_id>')
def get_user(user_id): ...

app.register_blueprint(users_bp)
```

**Flask extensions** — the ecosystem answer to Flask's deliberate minimalism: `Flask-SQLAlchemy` (integrates the SQLAlchemy ORM, section 14), `Flask-Migrate` (database migrations, via Alembic), `Flask-Login` (session-based authentication) — each an independently-maintained package following a loose common convention, rather than a single unified framework module.

**Application factories — Definition:** a pattern (`def create_app(): app = Flask(__name__); ...; return app`) that constructs the Flask app instance inside a function rather than at module import time — enables creating multiple app instances with different configurations (crucial for testing, where a fresh, isolated app instance per test is often desired) and avoids import-time side effects/circular-import issues that a module-level global `app` object can create as a project grows.

**When to choose Flask** — small-to-medium APIs/services, projects wanting fine-grained control over exactly which libraries/architecture are used rather than a framework's prescribed structure, or teams already comfortable assembling their own stack from independent pieces.

---

## 12. Web Frameworks — FastAPI

A modern, async-first framework deep-dive.

**Definition:** FastAPI is a modern Python web framework built around **type hints** (section 9) and **async-first** design — it derives request validation, serialization, and automatic API documentation directly from Python type annotations and Pydantic models (section 9), and treats `async def` route handlers as first-class citizens (Flask, by contrast, is fundamentally synchronous by default).

**Path operations & path/query parameters:**

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/users/{user_id}")
async def get_user(user_id: int, verbose: bool = False):  # path param + query param, both TYPE-VALIDATED automatically
    return {"id": user_id, "verbose": verbose}
```

FastAPI automatically validates that `user_id` is actually an integer (returning a `422` if not, with no manual parsing/validation code written) purely from the type annotation `user_id: int` — the direct payoff of building the framework around type hints from the ground up.

**Request body validation with Pydantic:**

```python
from pydantic import BaseModel

class CreateUserRequest(BaseModel):
    name: str
    email: str
    age: int

@app.post("/users")
async def create_user(user: CreateUserRequest):  # body automatically parsed + validated against the Pydantic model
    return {"created": user.name}
```

**Dependency injection (`Depends`) — Definition:** FastAPI's built-in DI system — a route handler declares a dependency as a parameter using `Depends(some_function)`, and FastAPI automatically calls that function (resolving any of *its* own nested dependencies first) and injects the result — used for shared logic like "get the current authenticated user" or "get a database session," conceptually similar to Spring's constructor injection (Java notes' section 8) or Angular's `inject()` (Angular notes' section 5), but resolved per-request via plain function calls rather than a class-based container.

```python
from fastapi import Depends

def get_db():
    db = SessionLocal()
    try: yield db
    finally: db.close()

@app.get("/users/{user_id}")
async def get_user(user_id: int, db=Depends(get_db)):
    return db.query(User).filter(User.id == user_id).first()
```

**Async endpoints** — an `async def` route handler runs on FastAPI's underlying asyncio event loop (section 8), able to `await` async database/HTTP calls without blocking other concurrent requests — a plain `def` (synchronous) route handler is automatically run in a separate thread pool by FastAPI instead, so both styles work, but `async def` unlocks the full I/O-concurrency benefit for I/O-bound endpoints.

**Automatic OpenAPI/Swagger docs — Definition:** FastAPI automatically generates a full, interactive OpenAPI specification and Swagger UI (`/docs`) directly from your route definitions and Pydantic models — no separate documentation-writing step, and the docs are guaranteed to never drift out of sync with the actual code, since they're generated *from* it.

**Background tasks — Definition:** `BackgroundTasks` lets a route handler schedule a lightweight function to run **after** the response has already been sent to the client — useful for quick fire-and-forget work (sending a confirmation email) that shouldn't delay the response, though for anything more substantial/reliable, a real task queue (Celery, section 17) is the more robust choice.

**Middleware (recap)** — same interception concept as the Node.js/Express notes' section 4 and Spring's filter chain (Java notes' section 12), applied via FastAPI's `@app.middleware("http")` decorator or ASGI middleware classes.

---

## 13. Web Frameworks — Django

A batteries-included framework deep-dive.

**Definition:** Django is a "batteries included" full-stack web framework — bundling an ORM, an admin interface, authentication, forms, templating, and more as integrated, first-party parts of the framework itself, rather than requiring separate extensions the way Flask does — trading some flexibility for a strongly opinionated, comprehensive, out-of-the-box structure.

**Project vs app structure — Definition:** a Django **project** is the overall configuration/settings container for a deployment; an **app** is a self-contained, reusable module of related functionality (e.g. a `users` app, an `orders` app) — a single project typically contains multiple apps, and Django apps are designed to be reasonably reusable across different projects.

**The Django ORM — Definition:** models defined as Python classes (subclassing `django.db.models.Model`) map directly to database tables, with Django auto-generating migrations from model changes (`python manage.py makemigrations`) — conceptually the same ORM role as SQLAlchemy/JPA-Hibernate (Java notes' section 11), tightly and specifically integrated into Django itself rather than a swappable third-party layer.

```python
from django.db import models

class User(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField(unique=True)
    created_at = models.DateTimeField(auto_now_add=True)
```

**Views (function-based vs class-based) — Definition:** a **function-based view** is a plain function taking a request and returning a response — simple, explicit, easy to follow. A **class-based view** (`ListView`, `DetailView`, etc.) provides reusable, inheritable behavior for common patterns (list a queryset, show a single object, handle a form) via class inheritance and mixins — less code for standard CRUD-style views, at the cost of more "magic"/indirection to trace through when customizing behavior.

**URL routing — Definition:** a project-level `urls.py` maps URL patterns to views, typically delegating sub-patterns to each app's own `urls.py` — the same path-based routing concept as every other framework covered in this workspace, expressed via Django's specific `path()`/`include()` functions.

**Django templates (brief) — Definition:** Django's built-in server-side templating language for rendering HTML with embedded Python-like expressions/logic — relevant for traditional server-rendered Django applications, though many modern Django deployments instead pair Django REST Framework (below) as a pure JSON API backend with a separate React/Angular frontend (this workspace's respective notes) rather than using Django templates at all.

**The admin interface — Definition:** Django's signature feature — a fully-functional, auto-generated web-based admin UI for viewing/creating/editing/deleting model data, derived automatically from your model definitions with minimal configuration (`admin.site.register(User)`) — a major practical advantage over Flask/FastAPI, which have no equivalent bundled in, for internal tooling/data-management needs.

**Django REST Framework (DRF) — Definition:** the de facto standard toolkit (a separate, but extremely widely-adopted, package) for building REST APIs on top of Django — serializers (converting model instances to/from JSON, with built-in validation, conceptually parallel to Pydantic models), viewsets, and browsable API documentation.

**When to choose Django vs Flask vs FastAPI — Definition:**

| Priority | Choice |
|---|---|
| Full-featured, batteries-included, admin UI, rapid traditional web-app development | Django |
| Minimal, maximum flexibility, small services, full control over architecture | Flask |
| Modern async-first API, automatic type-driven validation/docs, high I/O concurrency | FastAPI |

---

## 14. Data Access & ORMs

**SQLAlchemy Core vs ORM — Definition:** SQLAlchemy provides two layers — **Core** (a lower-level, expression-language-based SQL toolkit, giving fine-grained control while still being Pythonic and database-agnostic, roughly analogous in spirit to Knex from the Node.js notes' section 7) and the **ORM** (built on top of Core, mapping Python classes to database tables the way Django's ORM or Java's Hibernate does) — most applications use the ORM layer, dropping to Core (or raw SQL) for specific performance-critical or complex queries the ORM expresses awkwardly.

**SQLAlchemy models & sessions:**

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase): pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True)

# usage
session.add(User(email="ada@example.com"))
session.commit()
user = session.query(User).filter_by(email="ada@example.com").first()
```

A **Session** (the same conceptual role as JPA's `EntityManager`, Java notes' section 11) tracks loaded/pending objects and manages the transaction boundary — changes made to tracked objects are flushed/committed as a unit.

**Relationships (Python-specific syntax) — Definition:** the same one-to-many/many-to-many relational concepts already covered in the SQL/Java notes, expressed via SQLAlchemy's `relationship()` construct or Django's `ForeignKey`/`ManyToManyField` model fields.

```python
class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    user: Mapped["User"] = relationship(back_populates="orders")
```

**Django ORM (querysets, managers) — Definition:** a **QuerySet** is a **lazily-evaluated** representation of a database query — chaining `.filter()`/`.exclude()`/`.order_by()` builds up the query without hitting the database, only actually executing when the QuerySet is evaluated (iterated, sliced, or converted to a list) — the same lazy-evaluation philosophy as the DSA/JS notes' lazy generators/sequences, applied to query-building.

```python
User.objects.filter(active=True).order_by('-created_at')[:10]  # SQL only runs when this is actually evaluated
```

A **Manager** (`User.objects`) is the interface through which QuerySets for a model are initiated — customizable to add reusable, model-specific query logic.

**Alembic (migrations for SQLAlchemy) — Definition:** the standard migration tool for SQLAlchemy-based applications (Flask/FastAPI, which have no built-in migration system the way Django does) — generates and applies versioned schema migration scripts, the same migration concept as Django's own `makemigrations`/`migrate` and the Java/SQL notes' Flyway/Liquibase.

**Raw SQL vs ORM tradeoffs (recap)** — the same tradeoff already covered in the Node.js/MongoDB notes: an ORM speeds up development and adds a layer of safety/portability, at the cost of occasionally generating less-optimal SQL for complex queries, where dropping to raw SQL (or SQLAlchemy Core) becomes the pragmatic choice.

**Connection pooling in Python — Definition:** SQLAlchemy maintains its own connection pool by default (configurable pool size); in an async context (FastAPI + an async database driver like `asyncpg`), an async-compatible pool/engine is used instead — the same connection-reuse rationale as every other backend notes file in this workspace.

---

## 15. Authentication & Security

**Session-based auth in Flask/Django — Definition:** the same session-cookie-based authentication model already covered in the Node.js/Java notes — Django ships this behavior built-in (`django.contrib.auth`); Flask achieves it via the `Flask-Login` extension, both storing a session identifier in a cookie and looking up server-side session state on each request.

**JWT-based auth in FastAPI — Definition:** since FastAPI is commonly used as a pure API backend for a separate SPA frontend (no server-rendered session cookie relationship), token-based (JWT) authentication is the more typical pattern — implemented via a `Depends`-based dependency (section 12) that validates the `Authorization` header's bearer token and injects the resolved current user into any route that declares it, the same JWT-verification-middleware pattern already covered in the Node.js/Java notes.

```python
from fastapi import Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        return payload["sub"]
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/me")
async def read_me(user=Depends(get_current_user)):
    return {"user": user}
```

**Password hashing (`passlib`, `bcrypt`) — Definition:** the same deliberately-slow, salted password-hashing principle already covered in the Node.js/Java notes — `passlib` provides a unified interface over multiple hashing algorithms (bcrypt/argon2), commonly used across Flask/FastAPI/Django applications; Django ships its own built-in, pluggable password-hashing system out of the box.

**OAuth2 (recap, Python-specific libraries)** — `Authlib` and framework-specific integrations (`django-allauth`, FastAPI's `OAuth2PasswordBearer`/related security utilities) implement the same OAuth2/social-login flows already covered in the Node.js/Java/AWS notes.

**CORS handling — Definition:** the same Cross-Origin Resource Sharing concept already covered throughout this workspace — Flask's `flask-cors` extension, FastAPI's built-in `CORSMiddleware`, and Django's `django-cors-headers` package each provide the equivalent configuration to explicitly allow a separate frontend origin to call the API.

**SQL injection prevention (recap)** — the same principle covered in the SQL/Node.js notes: SQLAlchemy/Django ORM queries are parameterized automatically, protecting against injection as long as raw SQL string concatenation with untrusted input is avoided (`session.execute(text("... :param"), {"param": value})` with bound parameters, never f-string-interpolated raw SQL).

**Common Python-specific security pitfalls** — using `eval()`/`exec()` on any untrusted input (arbitrary code execution — a Python-specific, especially severe injection vector with no real equivalent in most other languages covered in this workspace); unsafe deserialization via `pickle.loads()` on untrusted data (pickle can execute arbitrary code during deserialization, unlike JSON parsing); leaving Django's `DEBUG = True` enabled in production (exposes detailed stack traces/environment info to any error response).

---

## 16. Testing in Python

**`pytest` fundamentals — Definition:** the de facto standard Python testing framework — simpler, more concise syntax than the standard-library `unittest` module (plain `assert` statements instead of `self.assertEqual()`-style methods), with a rich plugin ecosystem.

```python
def test_addition():
    assert add(2, 3) == 5

def test_raises_on_invalid_input():
    with pytest.raises(ValueError):
        parse_age("not a number")
```

**Fixtures — Definition:** reusable setup/teardown functions, declared with `@pytest.fixture`, injected into test functions simply by naming them as a parameter — pytest's dependency-injection-flavored alternative to the class-based `setUp`/`tearDown` methods of `unittest`/JUnit (Java notes' section 13).

```python
@pytest.fixture
def db_session():
    session = create_test_session()
    yield session  # provided to the test
    session.close()  # teardown, runs after the test regardless of pass/fail

def test_create_user(db_session):
    user = create_user(db_session, name="Ada")
    assert user.name == "Ada"
```

**Parametrized tests — Definition:** `@pytest.mark.parametrize` runs the same test function multiple times with different input/expected-output pairs, avoiding copy-pasted near-identical test functions.

```python
@pytest.mark.parametrize("input_val,expected", [(1, 1), (2, 4), (3, 9)])
def test_square(input_val, expected):
    assert square(input_val) == expected
```

**Mocking (`unittest.mock`) — Definition:** Python's standard-library mocking toolkit — `Mock`/`MagicMock` for creating test doubles, and `patch` (as a decorator or context manager) for temporarily replacing a real object/function with a mock for the duration of a test — the same test-double concept already covered in the Node.js/React/Java notes.

```python
from unittest.mock import patch

@patch('myapp.services.email_service.send_email')
def test_signup_sends_email(mock_send_email):
    signup(email="a@b.com")
    mock_send_email.assert_called_once_with("a@b.com")
```

**Testing async code — Definition:** `pytest-asyncio` (a plugin) enables writing `async def test_...` functions directly, letting tests `await` async application code naturally — plain pytest, without this plugin, cannot run async test functions correctly on its own.

**Testing Flask/FastAPI/Django apps — Definition:** each framework provides a **test client** — Flask's `app.test_client()`, FastAPI's `TestClient` (built on `httpx`), Django's `Client` — that issues requests against the application in-process without a real running server, the same in-process-HTTP-testing approach as the Node.js notes' `supertest` and the Java notes' `TestRestTemplate`/`WebTestClient`.

**Test coverage (`coverage.py`) — Definition:** measures what percentage/which specific lines of application code were actually executed during a test run — a useful signal for finding genuinely untested code paths, though (as with any codebase) a high coverage percentage on its own doesn't guarantee the tests actually assert meaningful, correct behavior.

---

## 17. Background Tasks & Messaging

**Why background tasks are needed — Definition:** work that's too slow to perform synchronously within a request/response cycle (sending an email, generating a report, processing an uploaded video) needs to be handed off to run **outside** that cycle, so the user gets an immediate response rather than waiting — the same rationale already covered in the Node.js/AWS notes' queue-based-load-leveling and background-processing sections.

**Celery — Definition:** the standard, most widely-used distributed task queue for Python — a **worker** process consumes tasks from a **broker** (typically Redis or RabbitMQ, section 12 of the AWS notes covers the SQS/SNS equivalent) and executes them asynchronously, decoupled entirely from the web application process that originally enqueued the task.

```python
from celery import Celery

celery_app = Celery('tasks', broker='redis://localhost:6379/0')

@celery_app.task
def send_welcome_email(user_id):
    user = get_user(user_id)
    email_service.send(user.email, "Welcome!")

# from a Flask/FastAPI/Django view:
send_welcome_email.delay(user.id)  # enqueues the task, returns IMMEDIATELY — doesn't block the request
```

**Periodic tasks (Celery Beat) — Definition:** a companion scheduler process that periodically enqueues tasks on a fixed schedule (cron-like syntax) — the Python/Celery ecosystem's equivalent of Node's `node-cron` (Node.js notes' section 17) or Spring's `@Scheduled` (Java notes' section 17), but running as a dedicated process rather than embedded in the application itself, which naturally avoids the "duplicate scheduled run across multiple horizontally-scaled instances" problem those other approaches need explicit coordination to solve.

**FastAPI `BackgroundTasks` (lightweight alternative, recap)** — see section 12; suitable only for very lightweight, non-critical fire-and-forget work executed within the same process, since it offers none of Celery's retry/persistence/distributed-worker guarantees — anything where failure/reliability genuinely matters belongs in Celery (or an equivalent real task queue), not `BackgroundTasks`.

**Async task patterns** — combining `asyncio` (section 8) with background task execution (e.g. Celery tasks that themselves run async code internally, or async-native task queue alternatives like `arq`) for I/O-heavy background work that benefits from asyncio's concurrency model rather than Celery's traditional synchronous-worker-per-task model.

---

## 18. Performance & Profiling

**Profiling tools — Definition:** `cProfile` (the standard-library deterministic profiler, tracking exact call counts/cumulative time per function — comprehensive but adds real overhead, best for targeted, non-production profiling runs); `py-spy` (a sampling profiler that can attach to a **running production process** with near-zero overhead, without needing to restart or specially instrument the target process) — the same "measure before optimizing" discipline emphasized throughout this workspace's other notes, with Python-specific tooling.

```bash
python -m cProfile -s cumulative myscript.py
py-spy top --pid 12345   # live, low-overhead profiling of an already-running process
```

**Common performance pitfalls** — string concatenation in a loop (same O(n²) issue as the Java notes' section 1, mitigated in Python with `''.join(parts)` instead of repeated `+=`); using a `list` where a `set`/`dict` would give O(1) membership testing instead of O(n) (DSA notes' section 1); accidentally serializing independent I/O-bound work instead of using `asyncio.gather`/threading (section 8); the GIL limiting CPU-bound threaded code to no real parallelism speedup (section 7).

**Caching strategies (recap)** — `functools.lru_cache` (section 3) for simple, single-process, in-memory memoization; Redis for a shared, distributed cache across multiple application instances — the same cache-aside pattern already covered in the Node.js notes' section 11.

**Async vs sync performance tradeoffs** — async I/O (section 8) provides the highest raw *throughput* for I/O-bound workloads under high concurrency (many more concurrent connections handled per unit of memory/overhead than a thread-per-request model), but every library in the call chain needs to genuinely support async to realize that benefit — calling a blocking, synchronous library function from inside an `async def` route handler blocks the **entire event loop**, stalling every other concurrent request, not just the one that made the blocking call — a serious, easy-to-introduce performance bug specific to async Python code.

**WSGI vs ASGI (brief, expanded in section 19) — Definition:** **WSGI** (Web Server Gateway Interface) is the traditional, synchronous Python web server interface (Flask/Django have historically run on it); **ASGI** (Asynchronous Server Gateway Interface) is its modern, async-capable successor (required for FastAPI, and supported by newer Django versions) — the interface layer determines whether the web server itself can actually deliver on async code's concurrency benefits end-to-end.

---

## 19. Production Engineering

**WSGI servers (Gunicorn) vs ASGI servers (Uvicorn) — Definition:** a Python web framework itself (Flask, FastAPI) is not a production-ready server on its own (its built-in dev server is explicitly not meant for production traffic) — it needs a dedicated **application server** in front of it. **Gunicorn** is the standard WSGI application server, typically running multiple synchronous **worker processes** (achieving parallelism via multiprocessing, section 8, since each worker is a separate process with its own GIL) for Flask/Django. **Uvicorn** is the standard ASGI server for FastAPI (and ASGI-capable Django), running an async event loop per worker — commonly deployed as `gunicorn` managing multiple `uvicorn` worker processes together, combining Gunicorn's mature process-management with Uvicorn's async event-loop execution.

```bash
gunicorn -k uvicorn.workers.UvicornWorker -w 4 myapp:app  # 4 worker processes, each running an async event loop
```

**Environment configuration (recap)** — the same `.env`/environment-variable-driven configuration principle covered throughout this workspace, commonly loaded via `python-dotenv` in development, with Pydantic's `BaseSettings` (section 9) providing typed, validated configuration loading in FastAPI applications specifically.

**Logging — Definition:** the standard-library `logging` module provides structured, leveled logging (the same `info`/`warn`/`error` leveling concept as the Node.js notes' section 16); production Python applications commonly configure it to output structured JSON logs (for compatibility with a log-aggregation tool) rather than plain-text `print()` statements.

**Dockerizing a Python app:**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["gunicorn", "-k", "uvicorn.workers.UvicornWorker", "-w", "4", "-b", "0.0.0.0:8000", "myapp:app"]
```

**Health checks & graceful shutdown (recap)** — the same principles already covered throughout the Node.js/AWS/Docker-Kubernetes notes: a lightweight `/health` endpoint for load balancer/orchestrator probes, and handling `SIGTERM` to stop accepting new requests while letting in-flight requests (and, for Celery, in-flight tasks) complete before actually exiting.

**Monitoring & error tracking — Definition:** the same Sentry-style error-tracking (capturing unhandled exceptions with full context) and Prometheus/Grafana-style metrics monitoring already covered in the Node.js/AWS/Docker-Kubernetes/Java notes apply identically to Python services, via Python-specific SDK integrations (`sentry-sdk`, `prometheus-client`) for each framework.

---

## 20. Interview Preparation

**Core Python interview questions** — explain the difference between a list and a tuple (section 2); explain the mutable-default-argument pitfall and how to avoid it (section 2); the difference between `is` and `==` (section 7); how Python's reference counting and cyclic garbage collector work together (section 7); the difference between `@staticmethod` and `@classmethod` (section 4); what a decorator is and how to write one (section 3).

**GIL & concurrency questions** — explain what the GIL is and why it exists (section 7); why threading doesn't speed up CPU-bound Python code, and what does instead (section 8); the difference between concurrency and parallelism, and how threading/multiprocessing/asyncio each relate to that distinction (section 8); how `asyncio`'s event loop achieves concurrency without multiple threads (section 8).

**Common Python coding problems** — the same core DSA problems covered in the DSA notes (arrays/strings/trees/graphs/DP), implemented with Python-specific idioms (list/dict comprehensions instead of explicit loops where natural, `collections.Counter`/`defaultdict` for frequency-counting problems instead of hand-rolled hash-map logic, `heapq` module for Python's built-in min-heap implementation directly satisfying the heap/priority-queue patterns from the DSA notes' section 7).

**Framework-specific interview questions** — how FastAPI's `Depends` dependency injection works (section 12); the difference between Django's function-based and class-based views (section 13); how Django QuerySets achieve lazy evaluation (section 14); when to choose Flask vs Django vs FastAPI for a given project (section 13).

**Python quirks & gotchas cheat sheet:**

```python
# mutable default argument — see section 2's full explanation
def f(items=[]): ...  # ❌ shared across calls

# late binding closures in a loop
funcs = [lambda: i for i in range(3)]
[f() for f in funcs]  # [2, 2, 2] — NOT [0, 1, 2] — all three lambdas share the same `i` variable
# fix: capture the current value as a default argument
funcs = [lambda i=i: i for i in range(3)]  # [0, 1, 2]

# integer identity caching (like the Java notes' Integer cache, section 1)
a, b = 256, 256
a is b  # True — small integers (-5 to 256) are cached/interned by CPython
c, d = 257, 257
c is d  # False (usually) — outside the cached range, may be separate objects — NEVER rely on `is` for int comparison

0.1 + 0.2 == 0.3  # False — IEEE 754 floating-point imprecision, same as every other language covered in this workspace
```
