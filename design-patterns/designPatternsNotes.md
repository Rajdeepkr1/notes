# Design Patterns — Deep Dive Roadmap

We'll go from fundamentals & principles → creational patterns → structural patterns → behavioral patterns → modern/architectural patterns → anti-patterns → interview problems.

*Examples are shown mostly in TypeScript/JavaScript for consistency with this workspace, with cross-references to how each pattern already appears in the Java, Python, Node.js, and frontend framework notes.*

---

## 1. Design Pattern Fundamentals

**Definition:** a design pattern is a general, reusable, **named** solution to a problem that recurs repeatedly in software design — not finished code you copy-paste, but a proven *template* for structuring the relationships between classes/objects to solve that recurring problem well.

**The Gang of Four (GoF) origin — Definition:** the term "design pattern" in software was popularized by the 1994 book *Design Patterns: Elements of Reusable Object-Oriented Software*, authored by four people (Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides — hence "Gang of Four"), which catalogued 23 recurring object-oriented design patterns, organized into three categories (below) — the vocabulary and pattern names this roadmap covers originate directly from that catalogue, even though the underlying ideas (and many patterns discovered since) extend well beyond it.

**Why patterns exist — Definition:** patterns give engineers a **shared vocabulary** for structural solutions — saying "let's use an Observer here" communicates an entire, well-understood design shape (and its known tradeoffs) in three words, instead of requiring a lengthy from-scratch explanation every time — the same value a shared technical vocabulary provides in any discipline, letting experienced engineers communicate design intent quickly and precisely.

**Patterns vs algorithms vs architecture — Definition:** an **algorithm** describes a precise computational procedure (see the DSA notes) — a pattern is not that granular. **Architecture** (see the System Design notes) describes the high-level structure of an entire system — a pattern is not that broad, either. A design pattern sits in between: it describes a reusable structural relationship between a handful of classes/objects, solving one recurring, medium-grained design problem, not an entire system's shape nor a single function's logic.

**The three GoF categories — Definition:**
- **Creational** (section 3) — patterns concerned with *how objects are created*, decoupling that creation from the code that uses the object.
- **Structural** (section 7) — patterns concerned with *how objects/classes are composed* into larger structures.
- **Behavioral** (section 12) — patterns concerned with *how objects communicate and distribute responsibility* among each other.

**How to know when a pattern actually applies — Definition:** a pattern earns its place when the *problem it solves* is genuinely present in your code — not because using a "proper" design pattern feels more sophisticated than a simpler, more direct solution. Reaching for a pattern before the corresponding problem actually exists is precisely the **premature abstraction** anti-pattern covered in section 19 — every pattern in this roadmap is presented with its concrete intent specifically so you can check "do I actually have this problem" before reaching for it.

---

## 2. SOLID Principles

**Definition:** SOLID is an acronym for five object-oriented design principles, formulated (and later named) by Robert C. Martin, that describe the properties of a maintainable, extensible class-based design — most of the GoF patterns in this roadmap are, in some sense, concrete techniques for *achieving* one or more of these principles in a specific recurring situation.

**Single Responsibility Principle (SRP) — Definition:** a class should have only one reason to change — i.e., it should be responsible for one, cohesive piece of functionality, not several unrelated ones bundled together — the same single-responsibility discipline already emphasized throughout this workspace's architecture-focused notes (the Node.js notes' layered-architecture section, splitting routes/controllers/services/models specifically so each layer has exactly one reason to change).

```ts
// ❌ violates SRP — this class has multiple reasons to change (validation logic, persistence logic, email logic)
class User {
  validate() { /* ... */ }
  save() { /* database logic */ }
  sendWelcomeEmail() { /* email logic */ }
}

// ✅ each responsibility is its own class
class User { /* just data + validate() */ }
class UserRepository { save(user: User) { /* ... */ } }
class EmailService { sendWelcomeEmail(user: User) { /* ... */ } }
```

**Open/Closed Principle (OCP) — Definition:** a class/module should be **open for extension, but closed for modification** — you should be able to add new behavior without editing existing, already-tested code — typically achieved by depending on an abstraction (an interface) that new behavior can implement, rather than a chain of `if`/`switch` statements that must be edited every time a new case is added. The Strategy pattern (section 13) is a direct, concrete implementation of this principle.

**Liskov Substitution Principle (LSP) — Definition:** if `S` is a subtype of `T`, then objects of type `T` should be replaceable with objects of type `S` **without altering the correctness of the program** — a subclass must honor the behavioral contract (not just the method signatures) of its parent class; a classic violation is a `Square` subclassing `Rectangle` and overriding `setWidth`/`setHeight` to keep both sides equal, which breaks code that correctly assumes a `Rectangle`'s width and height can be set independently.

**Interface Segregation Principle (ISP) — Definition:** clients should not be forced to depend on methods/interfaces they don't actually use — prefer several small, focused interfaces over one large, general-purpose one — the same principle behind good tool design in the Agentic AI notes' section 5 (a tool should do one clear thing) and good API design generally.

**Dependency Inversion Principle (DIP) — Definition:** high-level modules should not depend directly on low-level modules — both should depend on **abstractions** (interfaces); and abstractions should not depend on details — details should depend on abstractions. This is the formal principle underlying Dependency Injection (this workspace's Angular/Java/Node.js DI sections, and section 18 here) — a service class should depend on an interface it needs (e.g. a `Logger` interface), not on one specific concrete logging implementation, so the concrete implementation can be swapped without changing the high-level class at all.

**How SOLID motivates the patterns that follow** — most of the GoF patterns are, at their core, disciplined techniques for satisfying one or more SOLID principles in a specific, recurring shape of problem: Factory patterns (section 5) and Dependency Injection (section 18) both implement DIP; Strategy (section 13) and Decorator (section 9) both implement OCP directly; keeping this connection in mind is what separates *understanding why a pattern helps* from merely memorizing its structure.

---

## 3. Creational Patterns Overview

**Definition:** creational patterns address **how objects get created**, deliberately decoupling the code that *uses* an object from the code that *constructs* it — valuable whenever object creation is non-trivial (depends on configuration, involves choosing among several related classes, or should be controlled/limited in some way) rather than a simple, direct `new SomeClass()` call.

**The list at a glance:**

| Pattern | Solves |
|---|---|
| Singleton (§4) | Ensuring exactly one instance of a class exists |
| Factory Method (§5) | Letting subclasses decide which concrete class to instantiate |
| Abstract Factory (§5) | Creating families of related objects without specifying their concrete classes |
| Builder (§6) | Constructing a complex object step by step |
| Prototype (§6) | Creating new objects by cloning an existing instance |

---

## 4. Singleton Pattern

**Definition:** the Singleton pattern ensures a class has **exactly one instance** throughout an application's lifetime, and provides a single, well-known global point of access to it.

```ts
class Configuration {
  private static instance: Configuration;
  private constructor(private settings: Record<string, string>) {}

  static getInstance(): Configuration {
    if (!Configuration.instance) {
      Configuration.instance = new Configuration(loadSettingsFromDisk());
    }
    return Configuration.instance;
  }
}

const config = Configuration.getInstance(); // always the same instance
```

**Implementation approaches — Definition:** the classic GoF implementation uses a **private constructor** (preventing `new` from outside the class) plus a static `getInstance()` method that lazily creates the instance on first call and returns the cached instance on every subsequent call — the shape shown above, common across class-based languages including Java (Java Backend notes' section 15).

**Singleton vs a plain module-level instance — Definition:** in languages/environments with real module systems (JS/TS ES modules, Python modules — see the JS/TS notes' section 10 and Python notes' section 10), a module's top-level code runs exactly once, and every import of that module receives the same already-evaluated exports — which means simply exporting an instance directly from a module (`export const config = new Configuration();`) achieves the *exact same* singleton guarantee as the classic private-constructor pattern above, with far less boilerplate — the classic Singleton *pattern* is largely a workaround for languages that lack this natural module-level singleton behavior.

**Why Singleton is controversial — Definition:** Singleton introduces **global mutable state** into a program, which makes unit testing significantly harder (tests can leak state into each other through the shared singleton instance, and it's hard to substitute a mock/fake singleton for a test in isolation) and hides a real dependency that would be more explicit, and more testable, if passed in directly via constructor/dependency injection (section 18) — it's the single most consistently criticized pattern in the entire GoF catalogue for exactly this reason, and is frequently listed as its own kind of anti-pattern (section 19) when reached for by default rather than deliberately.

**Modern replacements — Definition:** in practice, most modern codebases achieve Singleton's actual goal (one shared instance) through mechanisms with fewer of its downsides — a **DI container** managing a service's lifetime as "singleton scope" (Java's Spring `providedIn`-equivalent, the Angular notes' `providedIn: 'root'`, section 8 of the Angular notes) gives you a shared instance *and* keeps the dependency explicit and swappable for tests; plain **module scope** (above) achieves the same for simpler cases with zero extra ceremony — the classic private-constructor Singleton pattern itself is comparatively rare in modern, framework-based code, precisely because these alternatives solve the same problem with fewer of its well-known drawbacks.

---

## 5. Factory Method & Abstract Factory

**Factory Method — Definition:** defines an interface for creating an object, but lets **subclasses decide which concrete class to instantiate** — the calling code depends only on the abstract creator/product interface, never on a specific concrete class directly, satisfying the Dependency Inversion Principle (section 2) for object creation specifically.

```ts
abstract class DialogCreator {
  abstract createButton(): Button; // factory method — subclasses decide the concrete type

  render() {
    const button = this.createButton();
    button.onClick(() => console.log('clicked'));
    return button;
  }
}

class WindowsDialogCreator extends DialogCreator {
  createButton(): Button { return new WindowsButton(); }
}
class WebDialogCreator extends DialogCreator {
  createButton(): Button { return new WebButton(); }
}
```

**Abstract Factory — Definition:** provides an interface for creating **families of related objects** without specifying their concrete classes — where Factory Method produces *one* product via subclassing, Abstract Factory produces a whole *set* of related products (that are meant to be used together) via composition, typically choosing the concrete factory implementation once (e.g. at startup, based on configuration) rather than per-call.

```ts
interface UIFactory {
  createButton(): Button;
  createCheckbox(): Checkbox;
}

class DarkThemeFactory implements UIFactory {
  createButton(): Button { return new DarkButton(); }
  createCheckbox(): Checkbox { return new DarkCheckbox(); }
}
class LightThemeFactory implements UIFactory {
  createButton(): Button { return new LightButton(); }
  createCheckbox(): Checkbox { return new LightCheckbox(); }
}

function renderUI(factory: UIFactory) {
  const button = factory.createButton();
  const checkbox = factory.createCheckbox(); // guaranteed to match the button's theme
}
```

**Factory Method vs Abstract Factory — Definition:** Factory Method is about **one product**, varied via subclass override; Abstract Factory is about a **family of related products** that must stay consistent with each other (all dark-themed, or all belonging to one platform), varied via swapping the whole factory implementation — Abstract Factory is frequently implemented internally using several Factory Methods, one per product type, making the two patterns naturally compose rather than being mutually exclusive alternatives.

**When a plain constructor is enough** — most object creation in real code needs neither pattern — reach for Factory Method/Abstract Factory specifically when *which concrete class to instantiate* is a genuine decision that varies (by configuration, platform, or subclass) and that decision needs to be made in one centralized place rather than scattered as `new ConcreteClassX()` calls throughout the codebase — for a class with exactly one concrete implementation and no anticipated need to vary it, a plain constructor call is simpler and entirely sufficient; wrapping it in a factory "just in case" is premature abstraction (section 19).

---

## 6. Builder & Prototype Patterns

**Builder — Definition:** separates the construction of a complex object from its representation, so the same step-by-step construction process can produce different configurations — via a fluent, chainable API rather than one large constructor.

```ts
class RequestBuilder {
  private url = '';
  private method = 'GET';
  private headers: Record<string, string> = {};
  private body?: string;

  setUrl(url: string) { this.url = url; return this; }
  setMethod(method: string) { this.method = method; return this; }
  addHeader(key: string, value: string) { this.headers[key] = value; return this; }
  setBody(body: string) { this.body = body; return this; }
  build(): Request { return new Request(this.url, this.method, this.headers, this.body); }
}

const req = new RequestBuilder()
  .setUrl('/api/users')
  .setMethod('POST')
  .addHeader('Content-Type', 'application/json')
  .setBody(JSON.stringify({ name: 'Ada' }))
  .build();
```

**Builder vs a constructor with many optional parameters — Definition:** a constructor with a dozen optional parameters (`new User(name, email, undefined, undefined, true, undefined, 'admin', ...)`) is hard to read and error-prone at every call site (easy to mix up positional argument order) — Builder trades that for a self-documenting, chainable sequence of clearly-named method calls, each setting exactly one property — the same problem this workspace's Java notes' section 15 covers, noting that records/named-parameter-style constructs (or Lombok's `@Builder`) have automated away much of Builder's manual boilerplate in modern code, without eliminating the pattern's underlying value for genuinely complex, multi-step construction.

**Prototype — Definition:** creates new objects by **cloning an existing instance** (a "prototype") rather than instantiating from a class — useful when creating an object from scratch is expensive (e.g. it required an expensive computation or a network call to initialize) and a very similar object is already available to copy instead, or when the exact concrete class to instantiate isn't known at compile time but an existing instance of the right type is available to clone.

```ts
class ShapeConfig {
  constructor(public color: string, public size: number, public metadata: Record<string, unknown>) {}
  clone(): ShapeConfig {
    return new ShapeConfig(this.color, this.size, { ...this.metadata }); // deep-copy the nested object
  }
}

const base = new ShapeConfig('red', 10, { tag: 'default' });
const variant = base.clone();
variant.color = 'blue'; // base is unaffected
```

**Cloning semantics (shallow vs deep copy) — Definition:** a **shallow copy** duplicates an object's top-level properties, but any property that's itself an object/array still points to the *same* underlying nested object as the original (mutating it through the clone would also affect the original); a **deep copy** recursively duplicates every nested object/array too, producing a fully independent copy — Prototype implementations must be deliberate about which one they need (the example above deep-copies `metadata` specifically to avoid the shallow-copy pitfall), the same distinction covered from a pure-language-mechanics angle in the JS/TS notes' section 20 interview-problems section ("implement a deep clone").

---

## 7. Structural Patterns Overview

**Definition:** structural patterns address **how classes and objects are composed** into larger structures, while keeping those structures flexible and efficient — concerned with the *relationships and composition* between already-created objects, distinct from creational patterns' concern with *how* those objects come into existence in the first place.

**The list at a glance:**

| Pattern | Solves |
|---|---|
| Adapter (§8) | Making an incompatible interface usable by translating it |
| Facade (§8) | Providing a simple interface to a complex subsystem |
| Decorator (§9) | Adding behavior to an object dynamically, without subclassing |
| Proxy (§10) | Controlling access to an object via a stand-in with the same interface |
| Composite (§11) | Treating individual objects and compositions of them uniformly |
| Bridge (§11) | Decoupling an abstraction from its implementation so both vary independently |
| Flyweight (§11) | Sharing common state across many objects to reduce memory use |

---

## 8. Adapter & Facade Patterns

**Adapter — Definition:** converts the interface of one class into another interface clients expect — letting classes with genuinely incompatible interfaces work together without modifying either one's existing source code, by introducing a translating wrapper between them.

```ts
// third-party library with an interface you don't control
class LegacyPaymentProcessor {
  makePayment(amountInCents: number, cardNumber: string) { /* ... */ }
}

// your application's expected interface
interface PaymentGateway {
  pay(amount: number, currency: string, card: string): void;
}

// Adapter — translates your interface to the legacy one
class LegacyPaymentAdapter implements PaymentGateway {
  constructor(private legacy: LegacyPaymentProcessor) {}
  pay(amount: number, currency: string, card: string): void {
    this.legacy.makePayment(Math.round(amount * 100), card); // adapts units + signature
  }
}
```

**Object adapter vs class adapter — Definition:** the **object adapter** (shown above) holds a reference to the adapted (`legacy`) instance and delegates to it via composition — works in any language, and is the far more common approach today; the **class adapter** achieves the same translation via multiple inheritance (inheriting from both the target interface and the adaptee class simultaneously) — only possible in languages supporting multiple inheritance, making it largely unavailable in single-inheritance languages like Java/TypeScript, which is why the object-adapter form dominates in practice.

**Facade — Definition:** provides a single, simplified, higher-level interface to a complex subsystem made up of many interacting classes — the calling code interacts only with the Facade, insulated from the subsystem's internal complexity and the coordination required to use it correctly.

```ts
// a Facade hiding the coordination needed across several subsystems
class VideoConverterFacade {
  convert(filePath: string, format: string): File {
    const codec = CodecFactory.extract(filePath);
    const buffer = VideoReader.read(filePath);
    const result = new BitrateReader().convert(buffer, codec);
    return new AudioMixer().mix(result);
  }
}

// calling code: one simple call, none of the internal coordination exposed
new VideoConverterFacade().convert('movie.avi', 'mp4');
```

**Adapter vs Facade (often confused)** — an **Adapter** makes one existing interface *look like* a different, specific interface the calling code already expects (a 1:1 translation, changing the shape of an interface without simplifying what it does); a **Facade** doesn't translate an existing interface at all — it creates an *entirely new, simpler* interface in front of a complex multi-class subsystem, hiding coordination complexity rather than reshaping a single interface's signature — the core distinction: Adapter answers "how do I make this fit the interface I need," Facade answers "how do I make this whole subsystem simpler to use."

---

## 9. Decorator Pattern

**Definition:** attaches additional responsibilities/behavior to an object **dynamically**, by wrapping it in one or more decorator objects sharing the same interface as the original — an alternative to subclassing for extending an object's behavior, one that can be composed flexibly at runtime rather than fixed at compile time by a class hierarchy.

```ts
interface Coffee {
  cost(): number;
  description(): string;
}

class SimpleCoffee implements Coffee {
  cost() { return 2; }
  description() { return 'Coffee'; }
}

class MilkDecorator implements Coffee {
  constructor(private coffee: Coffee) {}
  cost() { return this.coffee.cost() + 0.5; }
  description() { return `${this.coffee.description()} + Milk`; }
}

class SugarDecorator implements Coffee {
  constructor(private coffee: Coffee) {}
  cost() { return this.coffee.cost() + 0.25; }
  description() { return `${this.coffee.description()} + Sugar`; }
}

const order = new SugarDecorator(new MilkDecorator(new SimpleCoffee()));
order.cost();         // 2.75
order.description();  // "Coffee + Milk + Sugar"
```

**Decorator vs inheritance — Definition:** subclassing to add every possible combination of behavior (a `MilkAndSugarCoffee`, a `MilkOnlyCoffee`, a `SugarOnlyCoffee`) causes a **combinatorial explosion** of subclasses as the number of optional behaviors grows; Decorator instead lets any combination be composed at runtime by nesting a variable number of decorators, avoiding that explosion entirely — directly implementing the Open/Closed Principle (section 2): new decorators can be added without modifying `SimpleCoffee` or any existing decorator.

**Stacking decorators** — since each decorator implements the same interface as what it wraps, decorators can be nested arbitrarily deep, each one adding its own behavior on top of whatever it wraps, as shown in the `order` example above — the order of wrapping directly determines the order behaviors are applied in.

**Real-world appearances — Definition:** this pattern is genuinely everywhere in this workspace's own notes, even where not named "Decorator" explicitly — Express/Node.js **middleware** (Node.js notes' section 4, each middleware wraps the next handler in the chain, adding behavior around it); React **Higher-Order Components** (React notes' section 15); HTTP client **interceptors** (Node.js/Angular notes' auth interceptor sections, wrapping a request/response with added behavior); Java's `BufferedReader(new FileReader(...))`-style stream wrapping is the pattern's own textbook canonical example.

---

## 10. Proxy Pattern

**Definition:** provides a **surrogate or placeholder object** — implementing the same interface as a real "subject" object — that controls access to it, adding logic (lazy loading, access control, logging, remote communication) around every interaction, transparently from the calling code's perspective.

```ts
interface Image { display(): void; }

class RealImage implements Image {
  constructor(private filename: string) { this.loadFromDisk(); } // expensive
  private loadFromDisk() { console.log(`Loading ${this.filename}`); }
  display() { console.log(`Displaying ${this.filename}`); }
}

class ImageProxy implements Image {
  private realImage?: RealImage;
  constructor(private filename: string) {}
  display() {
    if (!this.realImage) this.realImage = new RealImage(this.filename); // lazy — only loads when actually needed
    this.realImage.display();
  }
}
```

**Common Proxy variants — Definition:**
- **Virtual proxy** — defers creation of an expensive object until it's actually needed (the `ImageProxy` example above — lazy loading).
- **Protection proxy** — checks access permissions before forwarding a request to the real object, adding authorization without modifying the real object itself.
- **Remote proxy** — represents an object living in a different address space/process/machine, handling the network communication transparently so calling code interacts with it as if it were local.
- **Logging/caching proxy** — transparently logs every call, or caches results of expensive calls, without the real object needing any awareness of it.

**Proxy vs Decorator (a subtle but real distinction) — Definition:** both wrap an object behind the same interface, which makes them structurally near-identical — the distinction is one of **intent**: Decorator's purpose is to *add new behavior/responsibilities* to an object; Proxy's purpose is to *control access* to an object (deferring its creation, checking permissions, adding a network hop) without fundamentally changing what it conceptually does — in practice this distinction sometimes matters more for communicating design intent clearly than for any concrete code difference.

**Native language support — Definition:** JavaScript's built-in `Proxy` object (JS/TS notes' section 13) is a direct, first-class language implementation of exactly this pattern, intercepting fundamental operations (get/set/delete) on a target object via traps; Java's dynamic proxies (`java.lang.reflect.Proxy`, Java Backend notes' section 15) serve the same role at the JVM level, and are literally the mechanism Spring uses internally to implement `@Transactional`/`@Cacheable` — wrapping your actual bean in a runtime-generated Proxy object.

---

## 11. Composite, Bridge & Flyweight Patterns

**Composite — Definition:** composes objects into **tree structures** to represent part-whole hierarchies, letting client code treat individual objects ("leaves") and compositions of objects ("composites"/branches) **uniformly**, through one shared interface.

```ts
interface FileSystemNode { getSize(): number; }

class File implements FileSystemNode {
  constructor(private size: number) {}
  getSize() { return this.size; }
}

class Directory implements FileSystemNode {
  private children: FileSystemNode[] = [];
  add(node: FileSystemNode) { this.children.push(node); }
  getSize(): number {
    return this.children.reduce((sum, child) => sum + child.getSize(), 0); // recurses uniformly
  }
}

const root = new Directory();
root.add(new File(100));
const subdir = new Directory();
subdir.add(new File(50));
root.add(subdir);
root.getSize(); // 150 — calling code never needed to know which nodes were files vs directories
```

**Bridge — Definition:** decouples an **abstraction** from its **implementation** so the two can vary independently — instead of a class hierarchy that grows combinatorially when both "what it is" and "how it's implemented" need to vary separately (the same combinatorial-explosion problem Decorator, section 9, solves for optional behaviors), Bridge splits them into two separate, independently-extensible hierarchies connected by composition.

```ts
// implementation hierarchy — "how"
interface Renderer { renderCircle(radius: number): void; }
class VectorRenderer implements Renderer { renderCircle(r: number) { /* draw vector */ } }
class RasterRenderer implements Renderer { renderCircle(r: number) { /* draw raster pixels */ } }

// abstraction hierarchy — "what" — bridges to a Renderer via composition, not inheritance
abstract class Shape {
  constructor(protected renderer: Renderer) {}
  abstract draw(): void;
}
class Circle extends Shape {
  constructor(renderer: Renderer, private radius: number) { super(renderer); }
  draw() { this.renderer.renderCircle(this.radius); }
}

new Circle(new VectorRenderer(), 5).draw();  // any Shape + any Renderer combination, without new subclasses
new Circle(new RasterRenderer(), 5).draw();
```

**Flyweight — Definition:** minimizes memory usage by **sharing** as much data as possible among many similar objects — splitting an object's state into **intrinsic** state (shared, context-independent, stored once in a shared flyweight object — e.g. a character glyph's shape in a text editor) and **extrinsic** state (unique per usage, passed in from outside rather than stored in the flyweight — e.g. that character's position on the page) — valuable specifically when an application needs to instantiate a very large number of fine-grained, mostly-similar objects (rendering thousands of trees in a game, characters in a text document) where naively storing full state per instance would consume excessive memory.

---

## 12. Behavioral Patterns Overview

**Definition:** behavioral patterns address **how objects communicate and how responsibility is distributed** among them — concerned with algorithms and the assignment of responsibilities *between* collaborating objects, distinct from creational patterns (how objects are made) and structural patterns (how objects are composed).

**The list at a glance:**

| Pattern | Solves |
|---|---|
| Strategy (§13) | Selecting an algorithm's behavior at runtime, interchangeably |
| Observer (§14) | Notifying multiple dependents automatically when state changes |
| Command (§15) | Encapsulating a request as an object (enables undo, queuing, logging) |
| Chain of Responsibility (§16) | Passing a request along a chain of potential handlers |
| Mediator (§16) | Centralizing complex communication between objects |
| State (§16) | Changing an object's behavior when its internal state changes |
| Iterator (§17) | Accessing elements of a collection without exposing its structure |
| Template Method (§17) | Defining an algorithm's skeleton, letting subclasses fill in steps |
| Visitor (§17) | Adding new operations to a class hierarchy without modifying it |
| Memento (§17) | Capturing and restoring an object's internal state |

---

## 13. Strategy Pattern

**Definition:** defines a family of interchangeable algorithms, encapsulates each one behind a common interface, and makes them swappable at runtime — the calling code depends only on the shared interface, never on any one specific algorithm's concrete implementation, directly implementing the Open/Closed Principle (section 2): a new strategy can be added without modifying any existing code.

```ts
interface ShippingStrategy { calculate(weight: number): number; }

class StandardShipping implements ShippingStrategy {
  calculate(weight: number) { return weight * 0.5; }
}
class ExpressShipping implements ShippingStrategy {
  calculate(weight: number) { return weight * 1.5 + 10; }
}

class Order {
  constructor(private shippingStrategy: ShippingStrategy) {}
  getShippingCost(weight: number) { return this.shippingStrategy.calculate(weight); }
}

new Order(new ExpressShipping()).getShippingCost(5); // strategy chosen/injected at construction time
```

**Strategy vs a plain function parameter — Definition:** in languages with first-class functions (JS/TS, Python, Java with lambdas), a "strategy" is very often just a function passed as a parameter (`array.sort((a, b) => a - b)` — the comparator *is* the Strategy pattern, expressed as a plain function rather than a full class hierarchy) — the full class-based Strategy implementation above is genuinely valuable when a strategy needs to bundle multiple related methods or hold its own internal state, but a single-function strategy is very often better expressed as simply passing a function directly, without the extra interface/class ceremony.

**Strategy vs State (they look structurally similar) — Definition:** Strategy and State (section 16) have nearly identical *structure* (an object holding a reference to an interchangeable interface implementation) but different *intent*: with **Strategy**, the calling code (or a client) explicitly *chooses* which algorithm to use, and that choice is typically independent of the object's own internal state; with **State** (section 16), the object *itself* switches its own internal implementation automatically, in response to its own state transitions, without external code choosing it explicitly — the distinction is about *who decides* and *why*, not the code shape.

**Real-world appearances** — sort comparators (as above), form validation rule sets (different validation strategies per field type), payment method selection (credit card vs PayPal vs bank transfer, each a distinct strategy behind one `processPayment()` interface) — anywhere "the same overall operation needs one of several genuinely interchangeable algorithms, chosen by the caller."

---

## 14. Observer Pattern

**Definition:** defines a one-to-many dependency between objects, so that when one object (the **subject**) changes state, all of its registered dependents (**observers**) are automatically notified and updated — without the subject needing to know anything concrete about its observers beyond that they implement the observer interface.

```ts
interface Observer { update(temperature: number): void; }

class WeatherStation {
  private observers: Observer[] = [];
  subscribe(observer: Observer) { this.observers.push(observer); }
  unsubscribe(observer: Observer) { this.observers = this.observers.filter(o => o !== observer); }

  setTemperature(temp: number) {
    for (const observer of this.observers) observer.update(temp); // notify everyone, subject stays decoupled
  }
}

class Display implements Observer {
  update(temperature: number) { console.log(`Display: ${temperature}°`); }
}

const station = new WeatherStation();
station.subscribe(new Display());
station.setTemperature(72); // "Display: 72°"
```

**Push vs pull observer models — Definition:** in the **push** model (shown above), the subject sends the actual changed data directly to observers in the notification call; in the **pull** model, the subject only notifies observers that *something* changed, and each observer separately queries the subject for whatever specific data it actually needs — push is simpler for a small, fixed set of data; pull scales better when different observers need different subsets of a subject's state, avoiding sending data no particular observer actually needed.

**Observer vs Pub/Sub (a real distinction) — Definition:** in the classic **Observer** pattern, subjects and observers know about each other directly (the observer calls `subject.subscribe(this)`) — a direct, if decoupled-by-interface, relationship; **Pub/Sub** (section 18) introduces a third party — a message broker/event bus — so publishers and subscribers never reference each other at all, communicating only through the intermediary — Pub/Sub is effectively Observer's relationship *further* decoupled by removing even the direct subscribe-to-subject link.

**Real-world appearances — Definition:** DOM events (`element.addEventListener`, the browser's own built-in Observer implementation); RxJS Observables (Angular notes' section 6 — the pattern's name is literally embedded in the library's name); Angular/React reactivity via **signals** (Angular notes' section 4, React notes via similar reactive-state libraries) — a signal's dependents (computed values, effects) are, conceptually, its observers, automatically notified and re-run when the signal's value changes.

---

## 15. Command Pattern

**Definition:** encapsulates a **request as a standalone object**, containing everything needed to perform an action (or undo it) — decoupling the object that *invokes* an operation from the object that actually *knows how to perform* it, and letting requests be queued, logged, parameterized, and reversed as first-class objects rather than immediate, one-shot function calls.

```ts
interface Command { execute(): void; undo(): void; }

class AddTextCommand implements Command {
  constructor(private document: TextDocument, private text: string) {}
  execute() { this.document.append(this.text); }
  undo() { this.document.removeLast(this.text.length); }
}

class CommandHistory {
  private history: Command[] = [];
  execute(command: Command) { command.execute(); this.history.push(command); }
  undoLast() { this.history.pop()?.undo(); }
}

const history = new CommandHistory();
history.execute(new AddTextCommand(doc, 'Hello'));
history.undoLast(); // reverses exactly that action, generically, via the Command interface
```

**Encapsulating a request as an object** — rather than the invoker (a UI button, a menu item) calling `document.append(text)` directly, it holds and invokes a `Command` object — the invoker doesn't need to know *what* the command actually does, only that it can be `execute()`d, the same decoupling principle behind the Strategy pattern (section 13) applied specifically to *actions/requests* rather than algorithms.

**Undo/redo with Command — Definition:** because each Command object encapsulates both `execute()` **and** the information needed to reverse it (`undo()`), maintaining a history stack of executed commands (as `CommandHistory` does above) gives undo/redo functionality "for free," generically, without the history-tracking code needing any awareness of what any specific command actually did — one of Command's most common and compelling real-world use cases (text editors, drawing applications, any UI needing undo support).

**Command vs Strategy — Definition:** structurally similar (both wrap behavior behind a common interface, both are objects passed around and invoked generically) — the distinguishing factor is intent and typical lifecycle: a **Strategy** is chosen once and represents an *interchangeable algorithm* for an ongoing operation; a **Command** represents a *specific, one-shot request/action* (often queued, logged, or made reversible) — Command objects are also far more likely to carry undo logic and be stored in a history, which Strategy objects typically never need.

---

## 16. Chain of Responsibility, Mediator & State Patterns

**Chain of Responsibility — Definition:** passes a request along a **chain** of potential handler objects, each deciding either to handle the request itself or pass it along to the next handler in the chain — the sender doesn't need to know which handler (if any) will actually end up processing the request, decoupling request senders from specific receivers.

```ts
abstract class Handler {
  private next?: Handler;
  setNext(handler: Handler): Handler { this.next = handler; return handler; }
  handle(request: Request): void {
    if (this.canHandle(request)) this.process(request);
    else this.next?.handle(request);
  }
  protected abstract canHandle(request: Request): boolean;
  protected abstract process(request: Request): void;
}

// Express/Node.js middleware (Node.js notes' section 4) is a direct, real-world Chain of Responsibility —
// each middleware either handles the request or calls next() to pass it further down the chain.
```

**Mediator — Definition:** defines an object that centralizes and encapsulates how a set of other objects interact, so those objects communicate **only through the mediator** rather than referencing each other directly — reduces a tangled web of many-to-many direct object references down to a simpler hub-and-spoke structure, at the cost of concentrating coordination logic/complexity into the mediator itself.

```ts
class ChatRoomMediator {
  private users: User[] = [];
  register(user: User) { this.users.push(user); user.setMediator(this); }
  broadcast(message: string, sender: User) {
    for (const user of this.users) {
      if (user !== sender) user.receive(message); // users never reference each other directly
    }
  }
}
```

**State — Definition:** lets an object alter its behavior when its internal state changes, appearing to the outside as if the object changed its class — implemented by delegating state-specific behavior to separate state objects, and having the context object switch which state object it currently delegates to as transitions occur.

```ts
interface OrderState {
  next(order: Order): void;
  getStatus(): string;
}

class PendingState implements OrderState {
  next(order: Order) { order.setState(new ShippedState()); }
  getStatus() { return 'pending'; }
}
class ShippedState implements OrderState {
  next(order: Order) { order.setState(new DeliveredState()); }
  getStatus() { return 'shipped'; }
}
class DeliveredState implements OrderState {
  next(order: Order) { /* terminal state — no further transition */ }
  getStatus() { return 'delivered'; }
}

class Order {
  private state: OrderState = new PendingState();
  setState(state: OrderState) { this.state = state; }
  advance() { this.state.next(this); }
  getStatus() { return this.state.getStatus(); }
}
```

State avoids a large, error-prone `switch (this.status) { case 'pending': ... case 'shipped': ... }` conditional scattered across every method that needs to behave differently per status — instead, each state's specific behavior lives together in its own dedicated class, and adding a new state means adding a new class rather than editing an existing switch statement everywhere it appears (Open/Closed Principle again, section 2).

---

## 17. Iterator, Template Method & Visitor Patterns

**Iterator — Definition:** provides a way to access the elements of a collection sequentially, without exposing the collection's underlying internal representation — the calling code interacts only with a standard iterator interface (`hasNext()`/`next()`, or the language's native iteration protocol), never needing to know whether the underlying collection is an array, a linked list, or a tree.

```ts
// TypeScript/JavaScript's native iterable protocol (JS/TS notes' section 9) IS the Iterator pattern,
// built directly into the language rather than needing to be hand-implemented per collection:
class Range implements Iterable<number> {
  constructor(private start: number, private end: number) {}
  [Symbol.iterator]() {
    let current = this.start;
    const end = this.end;
    return { next: () => current < end ? { value: current++, done: false } : { value: undefined, done: true } };
  }
}
for (const n of new Range(1, 4)) console.log(n); // 1, 2, 3 — works with for...of, spread, destructuring, etc.
```

**Template Method — Definition:** defines the **skeleton** of an algorithm in a base class method, deferring specific steps to subclasses via overridable methods — the overall sequence/structure of the algorithm is fixed in the base class (and cannot be changed by subclasses), while specific steps within it vary per subclass.

```ts
abstract class DataProcessor {
  // the template method — the fixed algorithm skeleton
  process(): void {
    this.readData();
    this.transform();  // varies per subclass
    this.writeData();
  }
  private readData() { console.log('reading...'); }
  protected abstract transform(): void; // subclasses fill in this one step
  private writeData() { console.log('writing...'); }
}

class UppercaseProcessor extends DataProcessor {
  protected transform() { console.log('uppercasing...'); }
}
```

Template Method inverts the usual calling relationship compared to Strategy (section 13) — with Strategy, your code calls out to an injected algorithm; with Template Method, the base class calls out to subclass-overridden steps ("don't call us, we'll call you" — sometimes called the **Hollywood Principle**), while the base class retains control of the overall sequence.

**Visitor — Definition:** lets you define a new operation over a class hierarchy without modifying the classes in that hierarchy — each element accepts a visitor object and calls back into it (`element.accept(visitor)` → `visitor.visitConcreteElement(this)`), letting new operations be added by writing a new Visitor class rather than editing every existing element class — a comparatively advanced, less commonly reached-for pattern, valuable specifically when a class hierarchy is stable (rarely gains new element types) but the *operations* performed over it need to grow frequently — notably the *inverse* tradeoff from Strategy/State: Visitor makes adding new **operations** easy at the cost of making adding new **element types** hard (every Visitor implementation would need updating), while a simpler polymorphic-method approach has the opposite tradeoff.

**Memento (brief) — Definition:** captures and externalizes an object's internal state (without violating its encapsulation) so it can be restored to that state later — the standard mechanism for implementing undo functionality when the state to restore is too broad/complex to reverse via Command's (section 15) explicit `undo()` logic alone, instead simply snapshotting and later restoring the object's full internal state directly.

---

## 18. Modern & Architectural Patterns

**Dependency Injection (recap, as a pattern in its own right) — Definition:** a technique implementing the Dependency Inversion Principle (section 2) — a class declares the dependencies (abstractions/interfaces) it needs, and an external system (a DI container, or simply a constructor argument passed in manually) supplies concrete implementations at runtime, rather than the class constructing its own dependencies internally — covered in full in this workspace's Angular notes' section 5, Java Backend notes' section 8, and Node.js notes' architecture section; DI is arguably the single most consistently *used* pattern across every modern framework covered in this workspace, more so than most of the classic GoF catalogue.

**Repository pattern — Definition:** mediates between the domain/business-logic layer and the data-access layer (a database), presenting a collection-like interface (`findById`, `save`, `delete`) for retrieving/persisting domain objects, while hiding the actual underlying data-access/query mechanics behind that interface — lets business logic depend on an abstraction rather than directly on ORM/query-building details, matching the same layered-architecture principle already covered in the Node.js notes' section 14 and Java Backend notes' section 14 (Spring Data JPA's repository interfaces are a direct, framework-native implementation of exactly this pattern).

```ts
interface UserRepository {
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<void>;
}

class MongoUserRepository implements UserRepository {
  async findById(id: string) { /* actual MongoDB query logic lives ONLY here */ }
  async save(user: User) { /* ... */ }
}

// business logic depends on the interface, not on MongoDB specifics — swappable, and mockable in tests
class UserService {
  constructor(private userRepo: UserRepository) {}
}
```

**MVC / MVVM / MVP — Definition:** architectural patterns separating an application into distinct layers by responsibility. **MVC** (Model-View-Controller) — Model holds data/business logic, View renders it, Controller mediates user input and updates the Model — the classic web-framework structure (Django, Spring MVC, Java Backend notes' section 10). **MVVM** (Model-View-ViewModel) — the View binds declaratively to a ViewModel that exposes state/commands, with the binding framework (Angular's/React's reactivity, this workspace's respective frontend notes) automatically syncing View and ViewModel, reducing the manual synchronization code MVC's Controller typically has to write by hand. **MVP** (Model-View-Presenter) — similar to MVC, but the Presenter (rather than the View itself) contains all presentation logic, with the View reduced to a passive, mostly-logic-free interface the Presenter drives directly — favored in some contexts specifically because it makes the presentation logic easier to unit-test in isolation from any real UI framework.

**CQRS (recap) — Definition:** Command Query Responsibility Segregation — already covered in the System Design notes' section 7 — separating the model used to **write** data (commands) from the model used to **read** data (queries), letting each be independently optimized (the write side for correctness/validation, the read side for the specific query shapes the application actually needs) rather than one shared model serving both purposes adequately but optimally for neither.

**Pub/Sub & Event-Driven patterns (recap)** — see section 14's Observer-vs-Pub/Sub distinction, and the System Design notes' section 7/15, AWS notes' section 12, and Node.js notes' section 6 (`EventEmitter`) for the concrete, infrastructure-level implementations of this pattern across different contexts — a message broker/event bus fully decouples publishers from subscribers, who never reference each other directly at all.

**Middleware pattern (recap)** — see section 16; the Node.js/Express request pipeline, Spring Security's filter chain (Java Backend notes' section 12), and similar request/response pipelines across virtually every web framework are a direct, practical application of Chain of Responsibility, specifically adapted to processing an HTTP request through a sequence of composable, independently-addable handling steps.

---

## 19. Anti-Patterns & Pitfalls

**Definition:** an anti-pattern is a **commonly-occurring solution to a recurring problem that is actually ineffective or counterproductive** — the mirror image of a design pattern: patterns are proven-good recurring solutions, anti-patterns are proven-bad recurring solutions, both worth recognizing by name specifically because they recur often enough to deserve a shared vocabulary.

**Over-engineering / pattern overuse — Definition:** applying a design pattern where the problem it solves doesn't actually exist yet (or may never materialize) — adding a Factory, a Strategy interface with only one real implementation, or a full Observer setup for a single, simple, one-off notification — patterns add real indirection and complexity, which is only worth paying for when the corresponding flexibility is actually needed; using a pattern to seem more "sophisticated" rather than because the underlying problem genuinely calls for it is a direct violation of this workspace's general engineering guidance against unnecessary abstraction.

**God Object — Definition:** a single class that has accumulated far too many responsibilities — knowing about and controlling large swaths of an entire system — a direct, severe violation of the Single Responsibility Principle (section 2), typically emerging gradually as new functionality keeps getting bolted onto an already-large, "convenient" existing class rather than being given its own properly-scoped home.

**Anemic Domain Model — Definition:** domain/model classes that contain **only data** (fields, getters, setters) with essentially *all* actual business logic living elsewhere, in separate "service" classes that operate on that data from the outside — while superficially resembling good separation of concerns, this pattern is widely considered an anti-pattern in genuinely object-oriented design specifically because it abandons **encapsulation** (the data and the logic that should govern its valid state/behavior are pulled apart, making it easy for the data to end up in states the "real" business rules would never have allowed) — a real, ongoing tension in modern layered architectures (including several of this workspace's own layered-architecture recommendations), worth being aware of as a deliberate tradeoff rather than an unconsidered default.

**Spaghetti code vs lasagna code (over-layering) — Definition:** **spaghetti code** — tangled, poorly-structured code with unclear control flow and responsibility boundaries, the classic, widely-recognized anti-pattern; **lasagna code** — the less commonly discussed *opposite* failure mode, where a codebase has so many thin, over-abstracted layers (a pattern wrapping a pattern wrapping a pattern) that simply tracing what a single operation actually does requires jumping through an excessive number of indirection layers — both are real failure modes, on opposite ends of the same spectrum, and "more layers/more patterns" is not unconditionally an improvement over "not enough."

**Premature abstraction (recap)** — see section 1's closing point; introducing a generalized, flexible abstraction (an interface with only one real implementation, a plugin system for a feature that's never actually varied) before there's concrete, demonstrated need for that flexibility — the design-pattern-specific instance of the broader "don't design for hypothetical future requirements" principle emphasized throughout this workspace's general engineering guidance — the antidote is typically to write the simple, direct version first, and introduce the abstraction later, specifically when a second real, concrete use case actually demonstrates the need for it.

---

## 20. Interview Preparation & Cross-Language Notes

**Common design pattern interview questions** — explain the difference between Factory Method and Abstract Factory, and give a concrete example of each (section 5); why is Singleton considered a controversial pattern, and what would you use instead in a modern codebase (section 4); walk through the difference between Strategy and State, given how structurally similar they are (sections 13 and 16); explain how Decorator avoids the subclass-explosion problem that plain inheritance would cause (section 9); what's the difference between Adapter and Facade (section 8) — most of these questions test whether you understand a pattern's actual *intent and tradeoffs*, not just whether you've memorized its class diagram.

**Identifying which pattern fits a described problem** — the practical interview (and real-world) skill: given a described problem, recognize *which category* (creational/structural/behavioral, sections 3/7/12) it belongs to first, then narrow to the specific pattern by matching the problem's shape against each pattern's stated intent (the **Definition** line at the top of each pattern's section throughout this file is deliberately written to make that matching straightforward) — "I need to add optional behavior to individual objects at runtime without a subclass explosion" maps directly and unambiguously to Decorator (section 9); "I need one shared point of coordination between many interacting objects" maps directly to Mediator (section 16).

**How these patterns show up across this workspace** — a genuinely useful cross-reference for connecting this abstract catalogue to code you've already seen concretely in this workspace's other notes: **Dependency Injection** everywhere (Angular §5, Java Backend §8, Node.js architecture section); **Decorator** as Express/Node.js middleware and React HOCs (Node.js §4, React §15); **Observer** as RxJS/signals (Angular §4/§6) and DOM events; **Proxy** as JS's native `Proxy` (JS/TS §13) and Spring's `@Transactional` internals (Java Backend §15); **Chain of Responsibility** as the Express/Spring Security middleware/filter chain (Node.js §4, Java Backend §12); **Repository** as Spring Data JPA repositories and this workspace's own Node.js/Python data-access-layer conventions (Node.js §14, Python Backend §14); **Builder** as Lombok's `@Builder` and fluent request-building APIs; **Facade** as nearly every SDK/client library wrapping a more complex underlying API (the AWS SDK, database client libraries) — recognizing a pattern you already use daily, now with its formal name and category, is often the fastest way to make this catalogue stick.

**Design pattern cheat sheet:**

| Need to... | Pattern |
|---|---|
| Ensure exactly one instance exists | Singleton (§4) |
| Let subclasses choose which class to instantiate | Factory Method (§5) |
| Create families of related objects consistently | Abstract Factory (§5) |
| Construct a complex object step by step | Builder (§6) |
| Create new objects by cloning | Prototype (§6) |
| Make an incompatible interface work with existing code | Adapter (§8) |
| Simplify access to a complex subsystem | Facade (§8) |
| Add behavior to an object dynamically, without subclassing | Decorator (§9) |
| Control/defer access to an object | Proxy (§10) |
| Treat individual objects and groups of them uniformly | Composite (§11) |
| Let an abstraction and its implementation vary independently | Bridge (§11) |
| Share state across many objects to save memory | Flyweight (§11) |
| Swap an algorithm at runtime | Strategy (§13) |
| Notify many dependents automatically on a change | Observer (§14) |
| Encapsulate a request as an undoable object | Command (§15) |
| Pass a request along a chain of handlers | Chain of Responsibility (§16) |
| Centralize communication between many objects | Mediator (§16) |
| Change behavior based on internal state | State (§16) |
| Traverse a collection without exposing its structure | Iterator (§17) |
| Fix an algorithm's skeleton, vary its steps | Template Method (§17) |
| Add new operations to a stable class hierarchy | Visitor (§17) |
| Snapshot and restore an object's state | Memento (§17) |
