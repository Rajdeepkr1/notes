# JavaScript & TypeScript — Deep Dive Roadmap

We'll go from fundamentals → language internals → async patterns → modern JS → the TypeScript type system → tooling → interview problems.

*Each major concept includes a **Definition**, and — where it represents a real choice or trade-off — **Advantages** and **Disadvantages**, so you know not just what something is but when to reach for it.*

---

## 1. JavaScript Fundamentals

**Definition:** JavaScript is a high-level, interpreted (with JIT compilation, section 14), single-threaded, multi-paradigm (supports procedural, object-oriented, and functional styles) programming language, originally designed for the browser and now also running server-side (Node.js) and in countless other environments.

**Variables — `var` vs `let` vs `const`:**

```js
var x = 1;   // function-scoped, hoisted with an "undefined" initial value, re-declarable
let y = 2;   // block-scoped, hoisted but in the Temporal Dead Zone (section 2) until declared
const z = 3; // block-scoped like let, but cannot be reassigned after initialization
```

**`var` — Advantages:** works in every JS environment ever (no compatibility concerns for very old code).
**`var` — Disadvantages:** function-scoped rather than block-scoped means it leaks out of `if`/`for` blocks unexpectedly; re-declarable, easy to accidentally shadow/overwrite; hoisted with `undefined`, hiding bugs that `let`'s Temporal Dead Zone would catch immediately.

**`let`/`const` — Advantages:** block-scoped (matches most programmers' intuition from other languages), `const` prevents accidental reassignment, the Temporal Dead Zone surfaces use-before-declaration bugs as errors instead of silent `undefined`.
**`let`/`const` — Disadvantages:** essentially none in modern JS — `let`/`const` are the recommended default today, with `var` retained only for legacy-code familiarity.

**Data types — Definition:** JavaScript has 7 **primitive** types (`string`, `number`, `bigint`, `boolean`, `undefined`, `null`, `symbol`) — immutable, compared by value — and one composite type, **object** (which includes arrays, functions, dates, etc.) — mutable, compared by reference.

**Type coercion — Definition:** JavaScript's automatic conversion of a value from one type to another when an operation expects a different type (e.g. `'5' + 1` coerces `1` to `'1'`, yielding `'51'`; `'5' - 1` coerces `'5'` to `5`, yielding `4`, since `-` has no string-concatenation meaning).

**Equality — `==` vs `===` — Definition:** `==` (**loose equality**) compares after performing type coercion if the operand types differ; `===` (**strict equality**) compares both value and type with no coercion at all.
**`==` — Disadvantage:** coercion rules are numerous and genuinely surprising in edge cases (`[] == false` is `true`; `null == undefined` is `true` but `null === undefined` is `false`), a frequent source of bugs.
**`===` — Advantage:** predictable, no hidden coercion — the recommended default; use `==` only deliberately (e.g. `x == null` as a concise check for both `null` and `undefined` at once).

**Truthy & falsy values — Definition:** every value coerces to `true` or `false` in a boolean context (an `if` condition, `!!value`); the **falsy** values are exactly `false`, `0`, `-0`, `0n`, `''`, `null`, `undefined`, and `NaN` — every other value (including `'0'`, `[]`, and `{}`) is truthy.

---

## 2. Execution Context & Scope

**Execution context — Definition:** the environment in which JS code is evaluated and executed, tracking variable bindings, the value of `this`, and a reference to the outer (lexically enclosing) environment — created for the global scope once, and freshly for each function call.

**The call stack — Definition:** the stack of currently-executing execution contexts — calling a function pushes a new context onto the stack; returning from it pops that context off — the same call stack referenced in the Node.js and DSA notes' recursion sections, and the mechanism behind stack-trace error output.

**Lexical scope — Definition:** a variable's accessibility is determined by *where it is written* in the source code (its nesting within functions/blocks), not by the call chain at runtime — the foundation that makes closures (section 3) predictable and analyzable just by reading the code's structure.

**Block scope vs function scope — Definition:** `let`/`const` are **block-scoped** (confined to the nearest enclosing `{ }`, including `if`/`for`/bare blocks); `var` is **function-scoped** (confined only to the nearest enclosing function, ignoring blocks entirely) — see section 1's `var` vs `let` comparison for the practical consequences.

**Hoisting — Definition:** JS's behavior of processing variable and function **declarations** during a preliminary pass before executing code line-by-line — `var` declarations and function declarations are hoisted with their binding created upfront (function declarations even get their full body/value hoisted, which is why they're callable before their textual position); `let`/`const` declarations are hoisted too, but left uninitialized.

**The Temporal Dead Zone (TDZ) — Definition:** the span of code between the start of a block and a `let`/`const` variable's actual declaration line, during which referencing that variable throws a `ReferenceError` rather than returning `undefined` — a deliberate safety improvement over `var`'s silent-`undefined` hoisting behavior.

```js
console.log(a); // undefined (var hoisted, initialized to undefined)
var a = 1;

console.log(b); // ReferenceError: Cannot access 'b' before initialization (TDZ)
let b = 2;
```

**The scope chain — Definition:** when a variable is referenced, the JS engine looks it up in the current execution context first, then walks outward through each enclosing lexical scope (following the chain established at the point the code was *written*, not called) until it's found or the global scope is exhausted (throwing a `ReferenceError` if never found) — this walk is exactly what makes closures work (section 3).

---

## 3. Closures

**Definition:** a closure is the combination of a function together with references to its surrounding (enclosing) lexical scope — a function "closes over" the variables it references from an outer scope, and continues to have access to them even after that outer function has finished executing and would otherwise have gone out of scope.

```js
function makeCounter() {
  let count = 0; // captured by the closure below
  return function () { return ++count; };
}
const counter = makeCounter();
counter(); // 1
counter(); // 2 — `count` persisted between calls, private to this counter instance
```

**Advantages:**
- Enables genuine **data privacy** — `count` above is inaccessible from outside except through the returned function, an encapsulation technique that predates (and still coexists with) private class fields (section 5).
- Enables **function factories** (functions that generate specialized functions, like `makeCounter` above) and **partial application/currying** (section 6).
- Enables **memoization** — a closure can hold a private cache object, persisting results across calls without any external/global state.

**Disadvantages:**
- Every variable captured by a closure is kept alive in memory for as long as the closure itself is reachable, even if only one small piece of the captured scope is actually still needed — a common, easy-to-miss source of memory retention (section 14) in long-lived closures (e.g. event listeners never removed).
- Overuse can make code harder to reason about, since a function's behavior may depend on captured state that isn't visible at its call site.

**Closures in loops (the classic `var` pitfall):**

```js
// ❌ with var — all three callbacks share ONE function-scoped `i`, which is 3 by the time any timeout fires
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // logs 3, 3, 3
}

// ✅ with let — each loop iteration gets its OWN block-scoped `i`, captured independently
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // logs 0, 1, 2
}
```

---

## 4. `this`, Function Binding & Execution

**Definition:** `this` is a special, implicit binding available within every (non-arrow) function, whose value is determined **by how the function is called** (the "call-site"), not by where the function is defined — a frequent source of confusion coming from languages where `this`/`self` is bound more predictably to the enclosing class instance.

**Call-site binding rules (in order of precedence):**

```js
// 1. `new` binding — `this` is the newly created object
function Person(name) { this.name = name; }
const p = new Person('Ada'); // this === p

// 2. explicit binding — call/apply/bind set `this` directly
function greet() { console.log(this.name); }
greet.call({ name: 'Bob' }); // 'Bob'

// 3. implicit binding — `this` is whatever object the method was called ON
const obj = { name: 'Cid', greet() { console.log(this.name); } };
obj.greet(); // 'Cid'

// 4. default binding — `this` is undefined in strict mode (or the global object, non-strict)
function bare() { console.log(this); }
bare(); // undefined (strict mode / ES modules)
```

**Arrow functions — Definition:** arrow functions do **not** have their own `this` binding at all — they inherit `this` lexically from their enclosing (non-arrow) scope at the point they're *defined*, exactly like a regular variable reference, rather than being determined by the call-site rules above.

**Arrow functions — Advantages:** eliminates the classic "`this` is wrong inside a callback" bug (below), since there's no separate `this` to lose; more concise syntax for short functions.
**Arrow functions — Disadvantages:** cannot be used as a constructor (`new` on an arrow function throws); cannot be used as an object method that needs to reference the object via `this` (since it would instead inherit the *enclosing* scope's `this`, not the object it's a property of); no `arguments` object of its own (inherits the enclosing one, if any).

**`call`, `apply`, `bind` — Definition:** three methods every function has for explicitly controlling its `this` value. `call(thisArg, arg1, arg2, ...)` invokes the function immediately with `this` set and arguments passed individually; `apply(thisArg, [argsArray])` is identical but takes arguments as an array; `bind(thisArg, ...)` does **not** invoke the function — it returns a **new function** permanently bound to that `this` value, callable later.

**`this` pitfalls (losing context in callbacks):**

```js
class Timer {
  constructor() { this.seconds = 0; }
  start() {
    // ❌ regular function — `this` inside setInterval's callback is NOT the Timer instance
    setInterval(function () { this.seconds++; }, 1000); // this.seconds is undefined-territory

    // ✅ arrow function — inherits `this` from `start`'s enclosing scope (the Timer instance)
    setInterval(() => { this.seconds++; }, 1000);
  }
}
```

---

## 5. Objects & Prototypes

**Object literals & property access — Definition:** the `{ key: value }` syntax for creating a plain object; properties are accessed via dot notation (`obj.key`, when the key is a valid identifier) or bracket notation (`obj['key']`, required for dynamic/computed keys or keys that aren't valid identifiers).

**Property descriptors — Definition:** every object property has, beneath its value, metadata controlling its behavior — `writable` (can it be reassigned), `enumerable` (does it show up in `for...in`/`Object.keys`), `configurable` (can it be deleted/redefined) — normally set implicitly by ordinary assignment, but controllable explicitly via `Object.defineProperty`.

**`Object.freeze` / `Object.seal` — Definition:** `Object.freeze` prevents adding, removing, *or modifying* any property (a shallow, one-level-deep immutability guarantee); `Object.seal` prevents adding/removing properties but still allows modifying existing ones' values.
**Advantage:** cheap, built-in way to enforce (shallow) immutability without a library.
**Disadvantage:** both are shallow only — freezing an object doesn't freeze nested objects within it — and neither throws in non-strict mode on a violation (the mutation attempt just silently fails), which can mask bugs.

**The prototype chain — Definition:** every JS object has an internal link (`[[Prototype]]`) to another object (or `null`); when a property is looked up and not found directly on the object, the engine automatically continues the search up this chain — the mechanism underlying JS's inheritance model, fundamentally different from classical class-based inheritance in languages like Java.

**`__proto__` vs `prototype` — Definition:** `obj.__proto__` (legacy accessor; `Object.getPrototypeOf(obj)` is the modern equivalent) is the *actual link* to an object's prototype, walked during property lookup; `Function.prototype` is a property that exists **only on functions**, specifying the object that will become the `__proto__` of any instance created by calling that function with `new` — these are two related but distinct concepts, a very common source of confusion.

**Prototypal inheritance — Definition:** objects inherit directly from other objects via the prototype chain, rather than from a class blueprint — `Object.create(protoObj)` creates a new object with `protoObj` explicitly set as its prototype, the most direct expression of this model.

**Classes as syntactic sugar — Definition:** ES6 `class` syntax is (mostly) **syntactic sugar** over the same underlying prototype-based mechanism — a `class`'s methods are installed onto its `prototype` object exactly as if written manually, and `extends` sets up the prototype chain between parent and child — classes don't introduce a genuinely different inheritance model, just a more familiar, class-oriented syntax over the same engine.

```js
class Animal {
  #name; // private field (# prefix) — truly inaccessible from outside the class, unlike a `_name` convention
  constructor(name) { this.#name = name; }
  speak() { return `${this.#name} makes a sound.`; }
  get name() { return this.#name; } // getter
}
class Dog extends Animal {
  speak() { return `${super.speak()} Woof!`; } // super calls the parent method
}
```

**Classes — Advantages:** more familiar/readable syntax for OOP-style code (especially to developers from class-based languages), private fields (`#`) give real (not just conventional) encapsulation, built-in `super`/inheritance ergonomics.
**Classes — Disadvantages:** can encourage a more rigid, deep inheritance hierarchy style that's often harder to refactor than JS's more flexible native composition/prototype patterns; `this` binding pitfalls (section 4) still apply to class methods passed as callbacks, just as with plain functions.

**Getters & setters — Definition:** `get`/`set` accessor methods that look like plain property access from the outside (`obj.name`, not `obj.name()`) but run custom logic underneath — used for computed properties or validating a value on assignment.

---

## 6. Functions — Deep Dive

**Function declarations vs expressions — Definition:** a **declaration** (`function foo() {}`) is fully hoisted (callable before its textual position, section 2); a **function expression** (`const foo = function() {}`) is only hoisted as a variable binding — the function value itself isn't available until that assignment line actually executes.

**Default parameters, rest & spread:**

```js
function greet(name = 'stranger') { return `Hi, ${name}`; } // default parameter

function sum(...nums) { return nums.reduce((a, b) => a + b, 0); } // rest — collects args into an array
sum(1, 2, 3); // 6

const arr1 = [1, 2], arr2 = [...arr1, 3, 4]; // spread — expands an iterable in place
```

**IIFE (Immediately Invoked Function Expression) — Definition:** a function defined and called in one expression, `(function () { ... })()`, historically used to create a private, isolated scope in pre-ES6 code (before block scope/modules existed) to avoid polluting the global scope — largely superseded today by ES modules (each module already has its own scope) and `let`/`const` block scoping, but still occasionally seen for scripts that need immediate, self-contained execution.

**Higher-order functions — Definition:** a function that either accepts another function as an argument, returns a function, or both — the foundation of JS's functional-programming capabilities (`.map`, `.filter`, `.reduce`, section 8, are all higher-order functions).

**Pure functions — Definition:** a function whose output depends *only* on its input arguments, with no observable side effects (no mutating external state, no I/O) — given the same input, a pure function always returns the same output.
**Advantages:** trivially testable in isolation, safely memoizable/cacheable, safely parallelizable/reorderable, and easier to reason about since there's no hidden external dependency to track.
**Disadvantages:** a strictly pure codebase eventually needs *some* boundary that performs actual side effects (writing to a database, rendering to a screen) — purity is a discipline applied to the *logic* layer, not a constraint that can (or should) apply to literally the entire program.

**Currying & partial application — Definition:** **currying** transforms a function taking multiple arguments into a sequence of functions each taking a single argument (`f(a, b, c)` → `f(a)(b)(c)`); **partial application** fixes some (not necessarily reducing to exactly one at a time) of a function's arguments upfront, returning a new function expecting the rest.

```js
const curriedAdd = a => b => c => a + b + c;
curriedAdd(1)(2)(3); // 6

const add = (a, b) => a + b;
const add5 = add.bind(null, 5); // partial application via bind
add5(3); // 8
```

**Function composition — Definition:** building a new function by chaining several smaller functions together, where each function's output feeds directly into the next function's input (`compose(f, g, h)(x) = f(g(h(x)))`) — a core functional-programming technique for building complex behavior from small, single-purpose, independently-testable pieces (section 12).

---

## 7. Asynchronous JavaScript

**Callbacks — Definition:** a function passed as an argument to be invoked later, once an asynchronous operation completes — JS's original mechanism for async code, still used directly for simple cases but largely superseded by Promises for anything more complex.

**Callback hell — Definition:** deeply nested callbacks (each async step nested inside the previous one's callback) resulting from chaining several sequential async operations with plain callbacks — hard to read, hard to handle errors consistently across (each nested callback needs its own error check), and hard to modify — the direct motivation for Promises being introduced into the language.

```js
// callback hell
getUser(id, (user) => {
  getOrders(user.id, (orders) => {
    getOrderDetails(orders[0].id, (details) => {
      console.log(details); // three levels deep, and this can keep growing
    });
  });
});
```

**Promises — Definition:** an object representing the eventual completion (or failure) of an asynchronous operation, existing in one of three **states**: `pending` (not yet resolved), `fulfilled` (completed successfully, holding a value), or `rejected` (failed, holding a reason/error) — once settled (fulfilled or rejected), a promise's state and value are permanent and cannot change again.

**Advantages over plain callbacks:** chainable via `.then()` (flattening nested callback hell into a linear chain), a single, consistent error-handling path via `.catch()` (an error anywhere in the chain propagates down to the nearest `.catch`, unlike needing a separate error check at every callback level), and composable via the combinators below.
**Disadvantages:** still noticeably more verbose than synchronous-looking code (mitigated by `async`/`await`, below); an *unhandled* rejected promise (no `.catch` anywhere in the chain) can silently swallow an error or crash the process (Node.js — see the Node.js notes' section 10), depending on runtime configuration.

```js
fetchUser(id)
  .then(user => fetchOrders(user.id))
  .then(orders => console.log(orders))
  .catch(err => console.error(err))
  .finally(() => console.log('done')); // runs regardless of success/failure
```

**Promise combinators (recap from the Node.js notes' section 10)** — `Promise.all` (fails fast on any rejection), `Promise.allSettled` (never rejects, reports every outcome), `Promise.race` (settles on whichever settles first), `Promise.any` (resolves on the first fulfillment, rejects only if all reject).

**`async`/`await` — Definition:** syntax sugar over Promises that lets asynchronous code be written and read like synchronous code — an `async` function always implicitly returns a Promise, and `await` pauses execution *within that function* (without blocking the rest of the JS runtime/event loop) until the awaited Promise settles.

```js
async function getOrderDetails(id) {
  try {
    const user = await getUser(id);
    const orders = await getOrders(user.id);
    return await getOrderDetails(orders[0].id);
  } catch (err) {
    console.error(err); // one try/catch covers the whole sequential chain
  }
}
```

**Advantages:** reads linearly, top-to-bottom, like synchronous code — dramatically easier to follow than an equivalent `.then()` chain for sequential logic; standard `try`/`catch` handles errors, no separate `.catch()` mechanism to learn.
**Disadvantages:** easy to accidentally serialize independent async operations that could have run in parallel (`await a(); await b();` runs sequentially even if `a()` and `b()` don't depend on each other — use `Promise.all([a(), b()])` instead); a function marked `async` always returns a Promise even if the caller forgets to `await`/handle it, which can hide bugs.

---

## 8. ES6+ Modern JavaScript Features

**Destructuring — Definition:** extracting values from arrays/objects into distinct variables via pattern-matching syntax, rather than accessing each property/index individually.

```js
const [first, second, ...rest] = [1, 2, 3, 4]; // array destructuring
const { name, age = 18 } = user;                // object destructuring, with a default
function greet({ name, age }) { /* ... */ }      // destructuring directly in a parameter
```
**Advantage:** concise, self-documenting extraction of exactly the fields needed. **Disadvantage:** deeply nested destructuring patterns can become harder to read than the equivalent explicit access, especially for reviewers unfamiliar with the shape being destructured.

**Optional chaining (`?.`) — Definition:** safely accesses a nested property, short-circuiting to `undefined` instead of throwing if any link in the chain is `null`/`undefined`.

```js
const city = user?.address?.city; // undefined if user or address is null/undefined, no error thrown
user?.greet?.(); // also works for optionally calling a method that might not exist
```
**Advantage:** eliminates verbose manual `&&`-chained null checks. **Disadvantage:** can silently mask a genuine bug (a property that *should* always exist but doesn't due to an upstream error) by quietly resolving to `undefined` rather than surfacing the problem loudly.

**Nullish coalescing (`??`) — Definition:** returns the right-hand operand only if the left-hand one is `null` or `undefined` specifically — unlike `||`, which falls through for *any* falsy value (`0`, `''`, `false` included).

```js
const count = 0;
count || 10;  // 10 — WRONG if 0 is a valid value, `||` treats it as falsy
count ?? 10;  // 0  — correct, only null/undefined trigger the fallback
```

**Array methods — Definition:** `.map()` (transform each element, returns a new array), `.filter()` (keep elements matching a predicate), `.reduce()` (fold the array down to a single accumulated value), `.find()`/`.findIndex()` (first matching element/index), `.some()`/`.every()` (does any/all elements match a predicate), `.flat()`/`.flatMap()` (flatten nested arrays, optionally combined with a map step) — the core, idiomatic toolkit for declarative array transformation, generally preferred over manual `for` loops for readability (section 12).

**`Map`/`Set`/`WeakMap`/`WeakSet` — Definition:** `Map` is a key-value collection like a plain object, but allows **any** value (not just strings/symbols) as a key, preserves insertion order, and has an O(1) `.size`; `Set` stores unique values only (no duplicates); `WeakMap`/`WeakSet` are similar but hold their keys/values **weakly** — an entry doesn't prevent its key from being garbage-collected (section 14) if nothing else references it, making them suitable for attaching metadata to an object without causing a memory leak if that object is later discarded elsewhere.

**Symbols — Definition:** a primitive type producing a guaranteed-unique value, commonly used as an object property key to avoid naming collisions with other code's properties (e.g. `Symbol.iterator`, section 9's iterable protocol).

---

## 9. Iterators & Generators

**The iterable protocol — Definition:** an object is **iterable** if it implements a method at the well-known key `Symbol.iterator` that returns an iterator — this is what makes `for...of`, spread syntax, and destructuring work uniformly across arrays, strings, `Map`s, `Set`s, and any custom object that implements the protocol.

**The iterator protocol — Definition:** an object is an **iterator** if it has a `.next()` method returning `{ value, done }` — called repeatedly to step through a sequence, until `done: true`.

**`for...of` vs `for...in` — Definition:** `for...of` iterates over an iterable's **values** (array elements, string characters, Map entries — requires the iterable protocol above); `for...in` iterates over an object's **enumerable property keys** (including, problematically, any inherited enumerable keys from the prototype chain) — `for...in` is generally discouraged for arrays specifically (use `for...of` or `.forEach`/`.map` instead), since it iterates indices as strings and can pick up unexpected inherited properties.

**Generator functions (`function*`) — Definition:** a special function that can be **paused and resumed**, using `yield` to emit a value and suspend execution until `.next()` is called again — unlike a normal function, which runs to completion once invoked, a generator's body executes incrementally, on demand.

```js
function* idGenerator() {
  let id = 1;
  while (true) yield id++; // infinite sequence — safe, because it's LAZY, only computed on demand
}
const gen = idGenerator();
gen.next().value; // 1
gen.next().value; // 2
```

**Advantages:** naturally express **lazy sequences** (including infinite ones, as above, which would be impossible to build eagerly), and provide a clean way to implement custom iterables (a generator function automatically satisfies the iterator protocol).
**Disadvantages:** less familiar/intuitive syntax than a plain function for most developers, and the "pause/resume" execution model can complicate reasoning about control flow compared to straightforward top-to-bottom code.

**Async generators & `for await...of` — Definition:** a generator function marked both `async` and `*` (`async function*`) that can `yield` values arriving asynchronously over time — consumed with `for await (const item of asyncGen())`, combining generators' lazy-pull model with Promise-based async values — used for processing paginated API results or streaming data chunk by chunk.

---

## 10. Modules

**CommonJS vs ES Modules (recap from the Node.js notes)** — CJS (`require`/`module.exports`) is Node's original, synchronous module system; ESM (`import`/`export`) is the JS-standard, asynchronous-by-design module system, now supported natively in both Node and every modern browser.

**CJS — Advantages:** universally supported in older Node code and packages, dynamic `require()` calls (conditionally requiring a module by a computed path) are natural.
**CJS — Disadvantages:** synchronous loading (blocks while resolving each `require`), no static structure (imports/exports can't be statically analyzed before running the code, which limits tree-shaking/dead-code elimination by bundlers).

**ESM — Advantages:** statically analyzable imports/exports enable **tree-shaking** (bundlers can determine and strip unused exports at build time, unlike CJS); the language-standard system, working identically (mostly) across browser and server; supports top-level `await`.
**ESM — Disadvantages:** some interop friction with the vast existing ecosystem of CJS-only packages (see the Node.js notes' ESM/CJS interop note); slightly more ceremony for certain dynamic/conditional-loading patterns (though `import()` — dynamic import — covers most of these).

**Default vs named exports — Definition:** a module can have **one** default export (`export default foo`, imported without braces and any local name — `import anything from './foo'`) and/or any number of **named** exports (`export { foo, bar }`, imported by their exact name with braces, optionally aliased — `import { foo as f } from './foo'`).

**Dynamic `import()` — Definition:** an expression (not a statement, unlike static `import`) that asynchronously loads a module and returns a Promise resolving to its exports — enables code-splitting/lazy-loading (see the React/Angular notes' lazy-loading sections), and can be called conditionally/at any point in code, unlike static imports which must appear at the top level.

**Module scope & singletons — Definition:** each module has its own isolated top-level scope (nothing leaks into the global scope by default, unlike classic `<script>` tags); a module's top-level code runs exactly **once**, the first time it's imported anywhere, and every subsequent import of the same module receives the **same** already-evaluated module object — which is why a service/store exported from a module behaves as a natural singleton across an application.

**Circular dependencies — Definition:** module A imports from module B, which imports from module A — both ESM and CJS *can* handle this without crashing, but whichever module's code runs first may receive an incomplete (partially-initialized) version of the other's exports, since the cycle is resolved by returning whatever has been evaluated so far rather than waiting — a design smell worth refactoring around rather than relying on the runtime's specific (and easy-to-get-wrong) resolution behavior.

---

## 11. Error Handling

**`try`/`catch`/`finally` — Definition:** `try` wraps code that might throw; `catch` handles a thrown error; `finally` runs regardless of whether an error occurred — the standard synchronous (and, with `async`/`await`, asynchronous) error-handling construct.

```js
try {
  riskyOperation();
} catch (err) {
  console.error(err);
} finally {
  cleanup(); // always runs
}
```

**Custom error classes — Definition:** extending the built-in `Error` class to create domain-specific error types carrying extra context (e.g. a status code), letting calling code branch on error type via `instanceof` rather than parsing an error message string.

```js
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = 'ValidationError';
    this.field = field;
  }
}
```

**Error propagation — Definition:** an error thrown inside a function, if not caught locally, automatically propagates up the call stack to the nearest enclosing `try`/`catch` (or, in async code, to the nearest `.catch()`/enclosing `try` around an `await`) — letting error handling be centralized at an appropriate layer rather than requiring every single function to handle every possible error itself.

**`Error.cause` — Definition:** a standard (ES2022+) way to attach an underlying/original error as context when wrapping and re-throwing a higher-level error, preserving the full causal chain for debugging (`new Error('Failed to save user', { cause: originalDbError })`) rather than losing the original error's information when wrapping it in a more descriptive one.

**Defensive vs assertive error handling — Definition:** **defensive** handling anticipates and gracefully recovers from an error condition (e.g. falling back to a default value); **assertive** handling deliberately throws/fails loudly and immediately when an invariant is violated, rather than silently continuing in a possibly-corrupted state — the right choice depends on whether continuing execution with a fallback is actually safe for that specific failure, or whether failing fast is the safer/more correct behavior.

---

## 12. Functional Programming in JavaScript

**Definition:** functional programming (FP) is a programming paradigm that treats computation as the evaluation of pure functions (section 6), favoring immutability and function composition over mutable state and imperative step-by-step instructions — JavaScript is multi-paradigm and doesn't enforce FP, but supports it well (first-class functions, closures, the array methods in section 8).

**Immutability — Definition:** the practice of never mutating existing data structures, instead producing new ones with the desired changes (e.g. `[...arr, newItem]` instead of `arr.push(newItem)`) — the same principle already covered in the React/Angular notes (why `OnPush`/`React.memo` require new references), rooted in FP's broader philosophy.
**Advantage:** eliminates an entire class of bugs caused by unexpected shared-reference mutation, and makes change detection/memoization trivially correct (reference equality is sufficient to detect a change).
**Disadvantage:** creating a new copy on every change has real performance/memory overhead compared to in-place mutation, particularly for large data structures — mitigated in practice by structural sharing (libraries like Immer/Immutable.js) or by simply not over-applying immutability to genuinely hot, performance-critical paths.

**Declarative vs imperative style — Definition:** **imperative** code describes *how* to achieve a result step by step (a `for` loop manually building up a result); **declarative** code describes *what* result is wanted, delegating the "how" to a higher-level construct (`.map()`/`.filter()`/`.reduce()`).

```js
// imperative
const doubled = [];
for (let i = 0; i < nums.length; i++) doubled.push(nums[i] * 2);

// declarative
const doubled = nums.map(n => n * 2);
```
**Declarative — Advantage:** typically more concise and communicates *intent* directly, reducing the chance of an off-by-one or similar loop-mechanics bug. **Declarative — Disadvantage:** chaining many array methods (`.map().filter().reduce()`) can create intermediate arrays and multiple full passes over the data, which may be less performance-efficient than a single hand-written imperative loop for very large datasets or hot paths — usually an acceptable trade for readability, but worth knowing when it isn't.

**Avoiding side effects (recap)** — see pure functions, section 6; FP style pushes side effects (I/O, mutation) to the edges of a program, keeping the core logic pure and easily testable.

**When FP patterns help (and when they don't)** — FP shines for data transformation pipelines and business logic that's naturally expressible as "take this data, produce that data"; it's a poorer fit (or needs to be applied more loosely) for code that's inherently about managing mutable state or sequencing side effects over time (a stateful UI component, a database transaction) — most real JS/TS codebases pragmatically mix FP-style data transformations with OOP/imperative code for stateful concerns, rather than committing to pure FP throughout.

---

## 13. Design Patterns in JavaScript

**Module pattern — Definition:** using a closure (historically an IIFE, section 6) to create a private scope, exposing only a deliberate public API — the pre-ES-modules way to achieve the encapsulation ES modules now provide natively.
**Advantage:** true privacy without needing a module system. **Disadvantage:** largely superseded by native ES modules (section 10) and class private fields (section 5), which achieve the same goal with clearer, more standard syntax.

**Singleton pattern — Definition:** ensures a class/module has only one instance, providing a single global point of access to it — in JS, this is achieved almost for free by a module's natural "runs once, cached on subsequent imports" behavior (section 10), rather than needing the more elaborate instance-tracking boilerplate the pattern requires in classical OOP languages.
**Advantage:** guarantees shared, consistent state/resources (e.g. one shared database connection). **Disadvantage:** introduces global mutable state, which can make testing harder (tests can leak state into each other via the shared singleton) and hides a dependency that would be more explicit/testable if passed in directly (dependency injection, see the Angular/Node.js notes).

**Factory pattern — Definition:** a function/method that creates and returns objects, encapsulating the creation logic (which concrete type to instantiate, based on input) so callers don't need to know the details.
**Advantage:** decouples object creation from usage, easy to extend with new types without changing calling code. **Disadvantage:** can add an unnecessary layer of indirection for genuinely simple object creation that doesn't actually vary.

**Observer pattern — Definition:** an object (the "subject") maintains a list of dependents ("observers") and automatically notifies them of state changes — the conceptual foundation behind DOM events, RxJS Observables (Angular notes), and React/Angular's own reactivity systems.
**Advantage:** decouples the subject from its observers — the subject doesn't need to know anything about who's listening. **Disadvantage:** can make control flow harder to trace (an event fires, but *where* it ultimately gets handled isn't obvious from reading the emitting code alone), and forgetting to unsubscribe an observer is a classic memory-leak source (section 14).

**Pub/Sub pattern — Definition:** similar to Observer, but decoupled further via a central event bus/broker that publishers and subscribers both go through, so they never reference each other directly at all (unlike Observer, where the subject directly holds its observer list) — the pattern behind Node's `EventEmitter` and systems like the AWS notes' SNS/EventBridge.

**Decorator pattern — Definition:** dynamically adding behavior to an object/function by wrapping it, without modifying its original code — Express/Node.js middleware (Node.js notes, section 4) and higher-order functions/components (React notes, section 15) are both practical applications of this pattern.

**Proxy pattern (`Proxy`/`Reflect`) — Definition:** JS's native `Proxy` object wraps a target object and lets you intercept and customize fundamental operations on it (property get/set, deletion, function calls) via **traps** — used for validation, logging, or reactivity systems (e.g. Vue 3's reactivity is built on `Proxy`).

```js
const validated = new Proxy({}, {
  set(target, prop, value) {
    if (prop === 'age' && typeof value !== 'number') throw new TypeError('age must be a number');
    target[prop] = value;
    return true;
  },
});
```
**Advantage:** transparent interception without changing the target object's own code. **Disadvantage:** proxied operations have real performance overhead versus direct property access, and behavior that happens "invisibly" via a trap can be harder to debug/discover than explicit code.

**Strategy pattern — Definition:** encapsulating a family of interchangeable algorithms behind a common interface, letting the algorithm used be selected/swapped at runtime — in JS, often as simple as passing a different function as an argument (a comparator function to `.sort()` is a lightweight, everyday instance of this pattern).

---

## 14. Memory Management & Performance

**The JS memory lifecycle — Definition:** allocate memory (when a value is created) → use it (read/write) → release it (once no longer reachable) — unlike lower-level languages, JS handles the release step automatically via **garbage collection**, not manual `free()` calls.

**Garbage collection (V8-specific, recap from the Node.js notes)** — see the Node.js notes' section 12 for the generational, mark-and-sweep mechanics; the same V8 engine (and thus the same GC behavior) underlies both Node.js and Chrome/Chromium-based browsers.

**Memory leaks in JavaScript — Definition:** memory that's no longer needed but remains reachable (and therefore never collected) because something still references it, unintentionally, causing memory usage to climb over time.
- **Detached DOM nodes** — a DOM element removed from the document but still referenced by a JS variable (or, commonly, by an event listener/closure) remains in memory, "detached" from the visible page but not garbage-collected.
- **Forgotten timers/listeners** — a `setInterval` or event listener that's never cleared/removed keeps its callback (and everything that callback's closure captures) alive indefinitely.
- **Closures holding references** — see section 3's closure disadvantage; a long-lived closure capturing a large object it no longer actually needs keeps that entire object alive.

**Performance profiling basics** — browser DevTools' Performance/Memory tabs (and Node's equivalents, covered in the Node.js notes' section 11) let you record a heap snapshot or CPU profile to find *actual* bottlenecks/leaks rather than guessing — the same "measure, don't assume" discipline emphasized throughout the other notes in this workspace.

**Debouncing & throttling — Definition:** both limit how often a rapidly-firing function (a scroll/resize/keystroke handler) actually executes. **Debouncing** delays execution until a pause of a specified duration has occurred since the last call (only the *last* call in a rapid burst actually runs) — ideal for "wait until the user stops typing" search-input scenarios. **Throttling** guarantees the function runs at most once per specified interval, regardless of how many times it's called within that window — ideal for scroll/resize handlers that should react periodically, not just once at the very end.

```js
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

function throttle(fn, interval) {
  let lastCall = 0;
  return (...args) => {
    const now = Date.now();
    if (now - lastCall >= interval) { lastCall = now; fn(...args); }
  };
}
```

---

## 15. TypeScript Fundamentals

**Definition:** TypeScript is a superset of JavaScript that adds an optional, static, **structural** type system, compiling (via `tsc`, or transpilers like esbuild/SWC) down to plain JavaScript — every valid JS program is (nearly) valid TypeScript, but TS adds compile-time type checking on top.

**Advantages:** catches an entire class of bugs (wrong argument type, typo'd property name, forgetting to handle `null`) at **compile time** instead of at runtime in production; dramatically better editor tooling (autocomplete, inline documentation, safe rename-refactoring) since the editor understands the actual shape of your data; serves as always-up-to-date, enforced documentation of function signatures and data shapes, unlike comments which can silently drift out of sync with the code.
**Disadvantages:** adds a compilation/build step (though modern tools make this fast and often invisible in dev); a learning curve for the type system itself (especially advanced types, section 18); can occasionally fight the developer with overly strict or hard-to-satisfy types for genuinely dynamic JS patterns, requiring type assertions/workarounds; the type system is fully erased at runtime (see below), so it provides zero *runtime* safety on its own — a value coming from outside the type system's view (an API response, `JSON.parse`) can still violate its declared type unless separately validated (e.g. with Zod, already covered in the Node.js/React forms notes).

**Basic types & type inference:**

```ts
let age: number = 30;      // explicit annotation
let name = 'Ada';           // inferred as `string` — TS infers types from initializers whenever possible
function double(n: number): number { return n * 2; } // parameter types must be explicit; return type can be inferred
```

**Type annotations vs inference — Advantage of relying on inference:** less visual noise, since TS is very good at inferring types from context (initializers, return statements). **Advantage of explicit annotations:** documents intent clearly at function boundaries (parameters/return types) — the common convention is to let TS infer *local* variable types, but explicitly annotate function signatures, since inference can't "see" what a function *should* accept/return from outside its own body.

**`any` vs `unknown` vs `never` — Definition:**
- **`any`** — opts a value **out** of type checking entirely; any operation on it is allowed, and it's assignable to/from anything — effectively turns off TypeScript for that value.
- **`unknown`** — the type-safe counterpart to `any`: a value of unknown type can be assigned *from* anything, but cannot be **used** (called, have properties accessed) until its type is first narrowed (section 16) — forces you to actually check what it is before operating on it.
- **`never`** — represents a value that can **never occur** (a function that always throws, or an unreachable code branch after exhaustive checking) — useful for exhaustiveness checking (section 16's discriminated unions).

**`any` — Disadvantage:** silently reintroduces every category of bug TypeScript exists to prevent, for that value and anything derived from it — should be used sparingly and deliberately, not as a default escape hatch.
**`unknown` — Advantage:** gets the same "I don't know this value's shape yet" flexibility as `any`, without giving up type safety — the recommended choice over `any` for genuinely-unknown-until-runtime values (e.g. a caught exception, or raw JSON before validation).

**Type aliases vs interfaces — Definition:** both name a type for reuse. `interface` is specifically for describing the shape of an object (or a class's contract) and supports **declaration merging** (multiple `interface` declarations with the same name automatically combine); `type` can name *any* type — object shapes, unions, tuples, primitives — but cannot be re-opened/merged after declaration.
**`interface` — Advantage:** merging is useful for extending third-party library types; slightly more familiar to developers from other typed OOP languages for describing object/class shapes.
**`type` — Advantage:** strictly more flexible — can express unions, intersections, and mapped/conditional types (section 18) that `interface` alone cannot. In practice: prefer `interface` for public object/class shapes meant to be extended, `type` for everything else (unions, utility compositions) — many teams simply standardize on one for consistency, since the practical difference rarely matters for typical object shapes.

**Enums — Definition:** a TS-specific construct for defining a named set of constant values.

```ts
enum Status { Active, Inactive, Pending }
```
**Advantage:** groups related constants under one namespace with reverse-lookup support. **Disadvantage:** numeric enums compile to actual runtime JS objects (unlike most type-only TS constructs, which are fully erased), adding to bundle size, and enums don't interoperate as cleanly with plain JS/JSON as a simpler alternative; many teams prefer a plain union of string literals (`type Status = 'active' | 'inactive' | 'pending'`, section 16) instead, since it's zero-runtime-cost and works naturally with JSON data.

---

## 16. TypeScript — The Type System

A major deep-dive topic.

**Structural typing (duck typing) — Definition:** TypeScript's type system is **structural**, not nominal — two types are considered compatible if they have the same *shape* (the same properties with compatible types), regardless of their declared name or explicit inheritance relationship — "if it walks like a duck and quacks like a duck." (Contrast with a **nominal** type system, like Java's or C#'s, where two classes with identical shapes are still *incompatible* unless one explicitly extends/implements the other.)

```ts
interface Point { x: number; y: number; }
function logPoint(p: Point) { console.log(p.x, p.y); }
logPoint({ x: 1, y: 2, z: 3 }); // ✅ valid — has AT LEAST x and y, extra properties are fine here
```
**Advantage:** very flexible — works naturally with plain object literals and doesn't force an explicit class hierarchy just to satisfy a type. **Disadvantage:** can occasionally accept a value that's structurally compatible but semantically wrong (e.g. a `Point` and a `Vector2D` with identical `{x, y}` shape are freely interchangeable, even if that's not conceptually intended) — mitigated with "nominal-like" techniques (a unique branding property) when that distinction genuinely matters.

**Union & intersection types — Definition:** a **union** (`A | B`) means a value could be *either* type A or type B; an **intersection** (`A & B`) means a value must satisfy *both* types simultaneously (combining all their properties).

```ts
type ID = string | number;                          // union
type Timestamped = { createdAt: Date };
type User = { name: string } & Timestamped;          // intersection — has both `name` and `createdAt`
```

**Literal types — Definition:** a type that represents one specific, exact value rather than a whole category (`'active'` as a type, not just `string`) — the building block for discriminated unions and precise, JSON-friendly enums (see the enum comparison above).

**Type narrowing — Definition:** the process by which TypeScript refines a broader type (e.g. a union) down to a more specific one within a particular code branch, based on runtime checks — the mechanism that lets you safely operate on an `unknown`/union value after first verifying what it actually is.

```ts
// typeof guard
function format(value: string | number) {
  if (typeof value === 'string') return value.toUpperCase(); // narrowed to string here
  return value.toFixed(2);                                    // narrowed to number here
}

// instanceof guard
function handle(err: Error | string) {
  if (err instanceof Error) console.log(err.message); // narrowed to Error
}
```

**Discriminated unions — Definition:** a union of object types that all share a common, literal-typed property (the "discriminant" — often named `type` or `kind`), letting TypeScript automatically narrow to the exact variant based on a simple check of that one property.

```ts
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'rectangle'; width: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case 'circle': return Math.PI * shape.radius ** 2;      // narrowed — radius is known to exist
    case 'rectangle': return shape.width * shape.height;    // narrowed — width/height known to exist
  }
}
```
**Advantage:** combined with a `switch` and the `never` type (section 15), TS can enforce **exhaustiveness checking** at compile time — adding a new shape variant without handling it in every relevant `switch` produces a compile error, rather than a silent runtime gap.

**Custom type guards (`is`) — Definition:** a function whose return type is `paramName is SomeType`, telling TypeScript that if the function returns `true`, the argument can be narrowed to `SomeType` at every call site — used to encapsulate a reusable narrowing check that's more complex than a simple `typeof`/`instanceof`.

```ts
function isString(val: unknown): val is string {
  return typeof val === 'string';
}
```

**Type assertions — Definition:** `value as SomeType` (or the older `<SomeType>value` syntax) tells the compiler "trust me, treat this as this type," **without** any actual runtime check — unlike narrowing, which TS verifies is logically sound from the code's control flow, an assertion is a pure compile-time override with zero runtime safety.
**Disadvantage:** an incorrect assertion compiles fine but can cause a genuine runtime type error later — assertions should be a last resort (e.g. when you have external knowledge TS genuinely cannot infer, such as the known shape of a DOM query result) rather than a routine way to silence type errors.

**`readonly` — Definition:** marks a property as unassignable after initial creation, enforced at **compile time only** (readonly is erased at runtime, unlike `Object.freeze`'s actual runtime enforcement, section 5) — a lighter-weight, TS-only immutability signal for catching accidental mutation during development.

**Optional & nullable properties:**

```ts
interface User {
  name: string;
  nickname?: string;       // optional — may be `undefined` or simply absent
  deletedAt: Date | null;  // nullable — must be present, but its value can be null
}
```

**Index signatures — Definition:** describes the type of values for keys not individually known ahead of time (`{ [key: string]: number }`), used for dictionary/map-like object shapes with dynamic keys.

---

## 17. Generics

**Definition:** generics let a function, interface, or class be written **once** but work correctly and type-safely across many different types, by parameterizing the type itself (conventionally written `<T>`) rather than hardcoding a specific type or falling back to `any`.

```ts
function identity<T>(value: T): T {
  return value;
}
identity('hello'); // T inferred as string, return type is string
identity(42);       // T inferred as number, return type is number
```

**Advantages:** provides the flexibility of `any` (works across many types) while **preserving full type safety** — unlike `any`, the actual type flowing in is tracked and enforced on the way out, so callers still get accurate autocomplete/type-checking on the result.
**Disadvantages:** more complex to read/write than a concretely-typed or `any`-typed equivalent, especially once generic constraints and multiple type parameters are involved — a genuine learning curve, though one that pays off significantly for reusable library-style code.

**Generic interfaces & classes:**

```ts
interface Box<T> { value: T; }
const stringBox: Box<string> = { value: 'hi' };

class Stack<T> {
  private items: T[] = [];
  push(item: T) { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
}
```

**Generic constraints (`extends`) — Definition:** restricts what types a generic parameter may be, so the function/class can safely rely on some minimum shape being present.

```ts
function getLength<T extends { length: number }>(item: T): number {
  return item.length; // safe — T is guaranteed to have a `length` property
}
getLength('hello'); // ✅ strings have .length
getLength([1, 2]);  // ✅ arrays have .length
getLength(42);       // ❌ compile error — number has no .length
```

**Default generic parameters — Definition:** `<T = string>` supplies a fallback type if the caller doesn't explicitly specify one and it can't be inferred from usage — reduces verbosity for the common case while still allowing full flexibility when needed.

**Common generic utility patterns** — a generic `ApiResponse<T>` wrapper (`{ data: T; error: string | null }`) reused across every API endpoint's specific response shape; a generic `Repository<T>` interface (`findById(id): T`, `save(item: T): void`) reused across different entity types — the general pattern of "write the reusable shape/behavior once, parameterize the entity-specific type."

---

## 18. Advanced Types

**Mapped types — Definition:** builds a new object type by transforming every property of an existing type, according to a rule, rather than listing each property manually.

```ts
type Readonly<T> = { readonly [K in keyof T]: T[K] };
type Partial<T> = { [K in keyof T]?: T[K] };
```

**Conditional types — Definition:** a type-level `if/else`, choosing between two types based on whether one type is assignable to another (`T extends U ? X : Y`).

```ts
type IsString<T> = T extends string ? true : false;
type A = IsString<'hi'>;  // true
type B = IsString<42>;    // false
```

**`infer` — Definition:** used within a conditional type to extract and name a type from within a larger, more complex type — e.g. pulling the return type out of a function type.

```ts
type ReturnTypeOf<T> = T extends (...args: any[]) => infer R ? R : never;
type Result = ReturnTypeOf<() => string>; // string
```

**Template literal types — Definition:** builds new string literal types by combining literal types with template-literal-like syntax, enabling precisely-typed string patterns.

```ts
type EventName = `on${Capitalize<'click' | 'hover'>}`; // 'onClick' | 'onHover'
```

**Utility types — Definition:** a set of built-in generic type helpers shipped with TypeScript itself, covering extremely common type transformations so they don't need to be hand-written repeatedly:

```ts
interface User { id: number; name: string; email: string; }

type PartialUser = Partial<User>;         // all properties optional
type UserPreview = Pick<User, 'id' | 'name'>;  // only the listed properties
type UserWithoutEmail = Omit<User, 'email'>;   // every property EXCEPT the listed ones
type UserMap = Record<number, User>;      // { [key: number]: User }
type RequiredUser = Required<User>;       // all properties required (opposite of Partial)
type ReadonlyUser = Readonly<User>;       // all properties readonly
```

**Recursive types — Definition:** a type that references itself in its own definition, used for genuinely recursive data shapes (a JSON value, a tree, a nested comment thread).

```ts
type JsonValue = string | number | boolean | null | JsonValue[] | { [key: string]: JsonValue };
```

**Type-level programming (brief) — Definition:** the broader practice of using conditional types, `infer`, mapped types, and template literal types together to compute genuinely complex types from other types entirely at compile time — extremely powerful for building precisely-typed library APIs, but a real complexity cost: heavily type-level code can become as hard to read/debug as any other complex code, and should generally be reserved for shared library/framework-level type definitions rather than routine application code.

---

## 19. TypeScript Tooling & Configuration

**`tsconfig.json` — Definition:** the configuration file controlling how the TypeScript compiler behaves — target JS version, module system, strictness flags, included/excluded files, and more.

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "./dist"
  }
}
```

**`strict` mode — Definition:** a single flag that enables a whole bundle of stricter type-checking sub-flags together — most notably `strictNullChecks` (without it, `null`/`undefined` are silently assignable to *any* type, defeating a huge portion of TS's safety value) and `noImplicitAny` (requires an explicit type when TS can't infer one, rather than silently falling back to `any`).
**Advantage:** the single highest-leverage setting for actually getting TypeScript's safety benefits — running without `strict` gives up most of what makes TS valuable in the first place. **Disadvantage:** enabling `strict` on an existing, previously-loose codebase can surface a large number of pre-existing type errors that need to be addressed — a real, one-time migration cost, though generally still worth paying.

**Module resolution — Definition:** the strategy TS uses to resolve `import` specifiers to actual files/type-declaration sources (`node` resolution mirrors Node.js's own algorithm from the Node.js notes' section 5; `bundler` resolution, newer, matches how modern bundlers like Vite/esbuild actually resolve imports, which differs subtly from Node's classic algorithm).

**Declaration files (`.d.ts`) — Definition:** files containing only type information (no actual runtime code), used to describe the shape of existing JavaScript code (e.g. a JS-only npm package) so TypeScript consumers get full type checking/autocomplete against it without that package needing to be rewritten in TS itself.

**Ambient declarations — Definition:** `declare` statements describing a value/module that exists at runtime but isn't defined in the current file (e.g. a global variable injected by a `<script>` tag, or a module TS can't otherwise resolve) — tells the compiler "trust that this exists with this shape," without providing an actual implementation.

**Type-only imports/exports — Definition:** `import type { Foo } from './foo'` explicitly marks an import as type-only, guaranteeing it's fully erased from the compiled JS output (some build tools require this explicit marking to safely eliminate type-only imports, since a plain `import { Foo }` is ambiguous about whether `Foo` is a type or a runtime value without deeper analysis).

---

## 20. Testing, Performance & Interview Prep

**Testing JS/TS code (recap)** — see the framework-specific notes (React's Testing Library section, Node's supertest/Jest section) for concrete testing setups; the core principles (unit vs integration vs E2E, mocking, TestBed-equivalents) apply consistently across vanilla JS/TS code too.

**Type-safe testing patterns** — TypeScript's type checking extends into test files too — a test asserting on a mocked object's shape gets the same compile-time safety as application code, catching a test that's silently testing against an outdated/wrong shape.

**Common JavaScript interview problems:**

```js
// implement debounce (see section 14 for the canonical implementation)

// implement Promise.all
function myPromiseAll(promises) {
  return new Promise((resolve, reject) => {
    const results = new Array(promises.length);
    let completed = 0;
    if (promises.length === 0) return resolve(results);
    promises.forEach((p, i) => {
      Promise.resolve(p).then(val => {
        results[i] = val;
        if (++completed === promises.length) resolve(results);
      }, reject); // any single rejection rejects the whole thing immediately
    });
  });
}

// implement a deep clone
function deepClone(obj, seen = new WeakMap()) {
  if (obj === null || typeof obj !== 'object') return obj;
  if (seen.has(obj)) return seen.get(obj); // handles circular references
  const clone = Array.isArray(obj) ? [] : {};
  seen.set(obj, clone);
  for (const key in obj) clone[key] = deepClone(obj[key], seen);
  return clone;
}

// polyfill Array.prototype.map
Array.prototype.myMap = function (callback) {
  const result = [];
  for (let i = 0; i < this.length; i++) result.push(callback(this[i], i, this));
  return result;
};
```

**Common TypeScript interview questions** — explain structural vs nominal typing (section 16); `interface` vs `type` (section 15); how discriminated unions enable exhaustiveness checking (section 16); what `unknown` provides over `any` (section 15); write a mapped/conditional type for a given transformation (section 18).

**Language quirks & gotchas cheat sheet:**

```js
typeof null;             // 'object' — a famous, long-standing JS bug, kept for backwards compatibility
0.1 + 0.2;                // 0.30000000000000004 — IEEE 754 floating-point imprecision, not JS-specific
NaN === NaN;              // false — NaN is never equal to itself; use Number.isNaN() to check
[] + [];                  // '' — both arrays coerce to empty strings, concatenated
[] + {};                  // '[object Object]'
typeof NaN;                // 'number' — NaN is technically a number value
Array.isArray([]);         // true — the correct way to check for an array (typeof [] is 'object')
```
