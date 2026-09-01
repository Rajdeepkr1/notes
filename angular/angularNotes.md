# Angular — Deep Dive Roadmap

We'll go from fundamentals → internals → architecture → performance → testing → production → interview problems.

---

## 1. Angular Fundamentals

**Definition:** Angular is a TypeScript-based, component-oriented front-end framework, maintained by Google, for building single-page web applications — it provides built-in solutions for templating, data binding, dependency injection, routing, and HTTP communication so teams don't have to assemble these from separate libraries.

**Architecture.** **Definition:** An Angular application's architecture is a tree of components, each pairing a TypeScript class with an HTML template, supported by directives, pipes, services, and a router. Components talk to the DOM only through Angular's abstractions (bindings, directives) — you rarely touch `document` directly. Supporting pieces: **directives** (behavior on elements), **pipes** (template value transforms), **services** (shared logic/state, injected via DI), and a **router** (URL → component mapping).

```ts
// app.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  standalone: true,
  template: `<h1>Hello, {{ name }}</h1>`,
})
export class AppComponent {
  name = 'Angular';
}
```

**Data binding. Definition:** Data binding is the mechanism that keeps a value in a component's TypeScript class synchronized with a value in its HTML template, in one or both directions.

```html
<!-- interpolation -->
<p>{{ user.name }}</p>

<!-- property binding -->
<img [src]="user.avatarUrl" />

<!-- event binding -->
<button (click)="save()">Save</button>

<!-- two-way binding (sugar for [value] + (valueChange)) -->
<input [(ngModel)]="user.name" />
```

**Directives. Definition:** A directive is a class decorated with `@Directive()` (or `@Component()`, which is a directive with a template) that attaches additional behavior or DOM structure to an element. Three kinds:
- **Component** — a directive with a template (the vast majority of what you write).
- **Structural** — reshape the DOM: `*ngIf`, `*ngFor`, or the modern `@if`/`@for` block syntax.
- **Attribute** — change appearance/behavior of an existing element: `[ngClass]`, `[ngStyle]`, or custom ones like `appHighlight`.

**Services & DI. Definition:** A service is a class dedicated to a single, focused responsibility (data fetching, logging, shared state) that Angular's dependency injection (DI) system constructs and hands to whatever component or service asks for it, instead of each consumer instantiating it directly:

```ts
@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  getUser(id: number) { return this.http.get(`/api/users/${id}`); }
}
```

**Modules vs standalone. Definition:** An `NgModule` is a class decorated with `@NgModule()` that groups a set of components, directives, pipes, and providers into one compilable/loadable unit; a **standalone** component/directive/pipe instead declares its own dependencies directly via an `imports` array, requiring no enclosing module at all. Since Angular 14–19, standalone is the default: apps can bootstrap without a root module (`bootstrapApplication`). New code should be standalone-only; `NgModule` still exists for legacy codebases.

**Angular CLI. Definition:** The Angular CLI (`ng`) is the official command-line tool for scaffolding, building, serving, testing, and updating Angular projects.

```bash
ng new my-app --standalone
ng generate component features/user-profile
ng generate service core/services/auth
ng serve
ng build --configuration production
ng test
ng add @angular/material
```

**Project structure (default CLI output):**

```
src/
 ├── app/
 │    ├── app.component.ts
 │    ├── app.config.ts     # providers, router, http client
 │    └── app.routes.ts
 ├── assets/
 ├── environments/
 └── main.ts                # bootstrapApplication(AppComponent, appConfig)
```

**Compilation. Definition:** Compilation is the process of transforming a component's template into executable JavaScript rendering instructions. Two modes:
- **JIT** (Just-in-Time) — compiles templates in the browser at runtime. Used historically in dev; mostly gone from modern workflows.
- **AOT** (Ahead-of-Time) — compiles templates to JS during `ng build`. Smaller bundles, faster bootstrap, template errors caught at build time. Default for both `ng serve` and `ng build` since Angular 9 (the Ivy renderer).

---

## 2. Components — Deep Dive

**Definition:** A component is the fundamental building block of an Angular UI — a TypeScript class decorated with `@Component()` that controls a patch of screen (a "view") through an associated HTML template and CSS.

### Component lifecycle

**Definition:** The component lifecycle is the fixed, ordered sequence of hook methods Angular invokes as it creates, updates, and eventually destroys a component instance.

Hooks fire in this order for a given component instance:

```
constructor
  → ngOnChanges (only if there are @Input()s, before first ngOnInit and on every input change)
  → ngOnInit (once)
  → ngDoCheck (every change detection run)
  → ngAfterContentInit (once, after projected content is initialized)
  → ngAfterContentChecked (every change detection run)
  → ngAfterViewInit (once, after component's own view + children are initialized)
  → ngAfterViewChecked (every change detection run)
  → ... (ngOnChanges / ngDoCheck / ngAfterContentChecked / ngAfterViewChecked repeat each CD cycle) ...
  → ngOnDestroy (once, when component is removed)
```

```ts
import {
  Component, Input, OnChanges, OnInit, DoCheck,
  AfterViewInit, AfterContentInit, OnDestroy, SimpleChanges,
} from '@angular/core';

@Component({ selector: 'app-widget', template: `...` })
export class WidgetComponent implements
  OnChanges, OnInit, DoCheck, AfterContentInit, AfterViewInit, OnDestroy {

  @Input() userId!: number;

  ngOnChanges(changes: SimpleChanges) {
    // fires before ngOnInit, and again whenever a bound @Input() changes
    if (changes['userId'] && !changes['userId'].firstChange) {
      this.reload();
    }
  }

  ngOnInit() {
    // one-time setup: fetch initial data, subscribe to services
  }

  ngDoCheck() {
    // runs on EVERY change detection pass — expensive, use sparingly
    // (custom dirty-checking for things Angular can't detect, e.g. mutated object)
  }

  ngAfterContentInit() {
    // projected <ng-content> children are now available
  }

  ngAfterViewInit() {
    // @ViewChild refs are now populated
  }

  ngOnDestroy() {
    // unsubscribe, clear timers, detach observers — prevents memory leaks
  }

  private reload() { /* ... */ }
}
```

**Key gotchas:**
- `ngOnChanges` only fires for `@Input()`-bound properties, and only when the *reference* changes (mutating an object/array in place won't trigger it).
- `ngDoCheck`/`ngAfterContentChecked`/`ngAfterViewChecked` run on **every** CD cycle — put nothing expensive in them.
- Reading `@ViewChild` before `ngAfterViewInit` gives `undefined`.

### Component communication

**Definition:** Component communication refers to the defined channels — inputs, outputs, and template queries — through which a parent and child component exchange data and events, since components cannot otherwise reach into each other directly.

**`@Input()` / `@Output()`** — parent → child data, child → parent events:

```ts
// child.component.ts
@Component({ selector: 'app-child', standalone: true, template: `
  <button (click)="notify()">Send</button>
` })
export class ChildComponent {
  @Input() label = '';
  @Input({ required: true }) itemId!: number;
  @Output() selected = new EventEmitter<number>();

  notify() { this.selected.emit(this.itemId); }
}
```

```html
<!-- parent.component.html -->
<app-child [label]="'Item'" [itemId]="42" (selected)="onSelected($event)" />
```

**View queries** — reach into the component's own template:

```ts
@Component({ template: `<input #box /><app-child #childRef />` })
export class ParentComponent implements AfterViewInit {
  @ViewChild('box') box!: ElementRef<HTMLInputElement>;
  @ViewChild(ChildComponent) child!: ChildComponent;
  @ViewChildren(ChildComponent) children!: QueryList<ChildComponent>;

  ngAfterViewInit() {
    this.box.nativeElement.focus();
  }
}
```

**Content queries** — reach into `<ng-content>`-projected content:

```ts
@Component({ selector: 'app-panel', template: `<ng-content /><ng-content select="[footer]" />` })
export class PanelComponent implements AfterContentInit {
  @ContentChild(TooltipDirective) tooltip?: TooltipDirective;
  ngAfterContentInit() { /* tooltip is now available */ }
}
```

### Dynamic components

**Definition:** A dynamic component is a component instance created imperatively at runtime (e.g. into a modal host or plugin slot), rather than declared statically in a parent's template — created via `ViewContainerRef`:

```ts
@Component({ template: `<ng-container #host />` })
export class HostComponent {
  private vcr = inject(ViewContainerRef);

  open() {
    const ref = this.vcr.createComponent(AlertComponent);
    ref.instance.message = 'Saved!';
    ref.changeDetectorRef.detectChanges();
  }
}
```

### Component inheritance

**Definition:** Component inheritance is the use of ordinary TypeScript class inheritance (`extends`) to share fields, methods, and lifecycle-hook logic between component classes. Angular runs the *subclass's* lifecycle hooks (base hooks only fire if not overridden, or explicitly called with `super`):

```ts
abstract class BaseFormComponent implements OnDestroy {
  protected destroyed$ = new Subject<void>();
  ngOnDestroy() { this.destroyed$.next(); this.destroyed$.complete(); }
}

@Component({ selector: 'app-signup', template: `...` })
export class SignupComponent extends BaseFormComponent {
  // inherits ngOnDestroy cleanup automatically
}
```

### Host bindings / listeners

**Definition:** A host binding/listener is a property binding or event listener attached to a component's or directive's own host element (the element the selector matched), configured from inside the class rather than from a parent's template.

```ts
@Directive({ selector: '[appHighlight]', standalone: true })
export class HighlightDirective {
  @HostBinding('class.highlighted') isHighlighted = false;

  @HostListener('mouseenter') onEnter() { this.isHighlighted = true; }
  @HostListener('mouseleave') onLeave() { this.isHighlighted = false; }
}
```

Modern equivalent using the `host` metadata object (preferred in new code):

```ts
@Component({
  selector: 'app-card',
  host: {
    '[class.selected]': 'isSelected()',
    '(click)': 'select()',
  },
  template: `...`,
})
export class CardComponent { /* ... */ }
```

---

## 3. Templates & Change Detection

### Binding syntax recap

**Definition:** Template syntax is the set of special markup (beyond plain HTML) that Angular templates use to express bindings, directives, and control flow.

```html
{{ expr }}              <!-- interpolation, text only -->
[prop]="expr"           <!-- property binding -->
(event)="handler($event)"  <!-- event binding -->
[(ngModel)]="value"     <!-- two-way binding -->
[attr.aria-label]="label"  <!-- attribute binding (for non-DOM-property attrs) -->
[class.active]="isActive"  <!-- class binding -->
[style.color]="color"      <!-- style binding -->
```

### Modern control flow (`@if` / `@for` / `@switch`)

**Definition:** Built-in control flow is template syntax — introduced in Angular 17 — for expressing conditionals and loops directly in a template using `@if`/`@for`/`@switch` blocks, without importing `NgIf`/`NgFor`/`NgSwitch` directives.

```html
@if (user(); as u) {
  <p>Welcome, {{ u.name }}</p>
} @else if (loading()) {
  <p>Loading…</p>
} @else {
  <p>Please log in.</p>
}

@for (item of items(); track item.id) {
  <li>{{ item.name }}</li>
} @empty {
  <li>No items.</li>
}

@switch (status()) {
  @case ('active') { <span>Active</span> }
  @case ('pending') { <span>Pending</span> }
  @default { <span>Unknown</span> }
}
```

`track item.id` is **mandatory reasoning** for `@for` — without a stable identity, Angular re-renders the whole list on every change instead of patching individual DOM nodes (the old `*ngFor` equivalent, `trackBy`, was easy to forget; `@for` forces you to think about it).

### Structural / attribute directives (legacy syntax, still valid)

**Definition:** A structural directive changes DOM *layout* by adding, removing, or repeating elements (e.g. `*ngIf`, `*ngFor`); an attribute directive changes the *appearance or behavior* of an existing element without altering DOM structure (e.g. `[ngClass]`).

```html
<div *ngIf="isVisible">Shown conditionally</div>
<li *ngFor="let item of items; trackBy: trackById">{{ item.name }}</li>
<div [ngSwitch]="status">
  <span *ngSwitchCase="'active'">Active</span>
  <span *ngSwitchDefault>Unknown</span>
</div>
```

### Change detection

**Definition:** Change detection (CD) is Angular's process of checking a component tree's bound expressions for changes and synchronizing the DOM to match, run automatically after events, timers, and HTTP responses.

Angular walks the component tree top-to-bottom each CD cycle, checking bindings and patching the DOM. Two strategies:

```ts
@Component({ changeDetection: ChangeDetectionStrategy.Default })  // checks every component, every cycle
@Component({ changeDetection: ChangeDetectionStrategy.OnPush })   // only checks when:
```

`OnPush` skips a subtree unless one of:
1. An `@Input()` **reference** changed.
2. An event originated from within the component (click, etc.).
3. An `Observable` bound with `| async` emitted.
4. `ChangeDetectorRef.markForCheck()` is called explicitly.
5. A **signal** read in the template changed (signals interop with `OnPush` automatically — this is a big reason Signals + `OnPush` is the recommended modern default).

```ts
@Component({
  selector: 'app-list',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<li *ngFor="let i of items">{{ i }}</li>`,
})
export class ListComponent {
  @Input() items: string[] = [];
}
```

```html
<!-- parent must pass a NEW array reference for OnPush to detect the change -->
<app-list [items]="items()" />
```

### Zone.js vs zoneless

**Definition:** Zone.js is a library that monkey-patches asynchronous browser APIs (`setTimeout`, `Promise`, DOM events, XHR) so Angular is automatically notified whenever application state may have changed, and can schedule a change-detection pass in response. **Zoneless** Angular is an execution mode (stable as of Angular 18+) that removes this dependency entirely, relying instead on explicit reactivity — primarily signals — to know when to re-render.

```ts
// app.config.ts
import { provideExperimentalZonelessChangeDetection } from '@angular/core';

export const appConfig: ApplicationConfig = {
  providers: [provideExperimentalZonelessChangeDetection(), /* ... */],
};
```

Zoneless apps must be signal-driven (or manually call `markForCheck()`/`ApplicationRef.tick()`), since there's no Zone to auto-trigger CD after a raw `setTimeout` or unpatched async call.

### `ChangeDetectorRef`

**Definition:** `ChangeDetectorRef` is an injectable handle on a component's node in the change-detection tree, used to imperatively mark it dirty or force an immediate check.

```ts
@Component({ changeDetection: ChangeDetectionStrategy.OnPush, template: `{{ value }}` })
export class LiveComponent {
  private cdr = inject(ChangeDetectorRef);
  value = 0;

  ngOnInit() {
    someExternalEmitter.on('update', (v) => {
      this.value = v;
      this.cdr.markForCheck();   // schedules CD for this component & ancestors on next tick
      // this.cdr.detectChanges(); // alternative: runs CD synchronously, right now, for this subtree
    });
  }
}
```

- **`markForCheck()`** — flags the component (and ancestors, so the path isn't skipped) as dirty; Angular checks it on the *next* CD cycle.
- **`detectChanges()`** — runs CD immediately, synchronously, just for this component and its children. Useful in tests or when you need the DOM updated before the next tick (e.g. before measuring an element).
- **`ApplicationRef.tick()`** — manually triggers a full CD pass across the whole app; used by the framework itself and rarely called directly.

### Preventing unnecessary rendering

**Definition:** This is the set of practices that minimize how much change-detection work Angular performs per cycle, keeping large component trees responsive.

- Prefer `OnPush` + immutable data + signals.
- Avoid calling methods in templates (`{{ getTotal() }}`) — they re-run every CD cycle. Use a `computed()` signal or a pure pipe instead.
- Use `track` in `@for` / `trackBy` in `*ngFor`.
- Use `| async` (or `toSignal`) instead of manual `subscribe()` + field assignment, so Angular manages the CD trigger correctly.

---

## 4. Signals

A major modern Angular topic.

**Definition:** A signal is a wrapper around a value that keeps track of who reads it, and notifies those readers whenever the value changes — the primitive underlying Angular's fine-grained reactivity system.

### Core API

**`signal()` — Definition:** a function that creates a writable, reactive value container.

```ts
import { signal } from '@angular/core';

const count = signal(0);

count();                  // read → 0
count.set(5);              // overwrite
count.update(v => v + 1);  // derive from current value → 6
```

Unlike RxJS Observables, a signal always has a current value you can read synchronously by calling it — no subscription needed.

**`computed()` — Definition:** a function that creates a read-only signal whose value is derived from other signals, recomputed lazily — only when read, and only if a dependency actually changed.

```ts
import { signal, computed } from '@angular/core';

const price = signal(100);
const qty = signal(2);
const total = computed(() => price() * qty());

total(); // 200
qty.set(3);
total(); // 300 — recalculated because qty changed
```

`computed()` values are memoized — reading `total()` twice without a dependency change does not re-run the function.

**`effect()` — Definition:** a function that registers a side-effect callback to be re-run automatically whenever any signal it reads changes. Runs once immediately on creation, then re-runs after each change, scheduled asynchronously (batched via the microtask queue, not synchronously per `set()` call).

```ts
import { signal, effect } from '@angular/core';

const count = signal(0);

effect(() => {
  console.log(`count is now ${count()}`);
});

count.set(1); // logs "count is now 1"
```

Rules of thumb for `effect()`:
- Use it for side effects only (logging, syncing to localStorage, DOM manipulation outside templates) — never to derive state (use `computed()` for that).
- Must be created in an injection context (constructor, field initializer, or with an explicit `Injector` via `{ injector }`).
- By default it's tied to the component/directive lifecycle and is cleaned up on destroy.

```ts
@Component({...})
export class CartComponent {
  private cartService = inject(CartService);

  constructor() {
    effect(() => {
      localStorage.setItem('cart', JSON.stringify(this.cartService.items()));
    });
  }
}
```

**`set()` vs `update()` vs `mutate()`**

```ts
const items = signal<string[]>([]);

items.set(['a', 'b']);                 // replace entirely
items.update(list => [...list, 'c']);  // derive new value from old (immutable pattern)

// mutate() was removed in Angular 17+ — always create a new
// reference with set()/update() so change detection can compare by identity.
```

Angular's Signals implementation compares values by reference (like `OnPush`), so always produce a *new* array/object rather than mutating in place:

```ts
// ❌ won't trigger dependents reliably
items.update(list => { list.push('c'); return list; });

// ✅ correct
items.update(list => [...list, 'c']);
```

### Signal dependencies

**Definition:** A signal dependency is a signal read during the execution of a `computed()` or `effect()` callback; Angular tracks these reads automatically, so the consumer re-runs whenever any of them changes — no manual dependency arrays like React's `useEffect`.

```ts
const showDetails = signal(false);
const detail = signal('secret');

const label = computed(() => {
  // detail() is only read conditionally, so it's only
  // a dependency when showDetails() is true
  return showDetails() ? detail() : 'hidden';
});
```

This is dynamic dependency tracking — the dependency graph can change between runs.

### Signal-based inputs

**Definition:** A signal input, declared with `input()`, is a component `@Input()` exposed to the class as a read-only signal instead of a plain mutable property. Since Angular 17.1+, `@Input()` can be replaced with `input()`.

```ts
import { Component, input } from '@angular/core';

@Component({...})
export class UserCardComponent {
  // optional, defaults to undefined
  name = input<string>();

  // required — must be bound by the parent
  userId = input.required<number>();

  // with default value
  role = input('guest');

  // with a transform function
  disabled = input(false, { transform: (v: string | boolean) => !!v });
}
```

Template usage stays reactive automatically:

```html
<p>{{ name() }}</p>
```

Benefits over decorator `@Input`: read-only outside the component, composable with `computed()`, and no need for `ngOnChanges` to react to changes.

### Signal-based queries

**Definition:** Signal queries — `viewChild()`, `viewChildren()`, `contentChild()`, `contentChildren()` — are the signal-based equivalents of `@ViewChild`/`@ContentChild`, exposing a queried template element or component as a signal instead of a plain property.

```ts
import { Component, viewChild, ElementRef } from '@angular/core';

@Component({...})
export class SearchBoxComponent {
  inputRef = viewChild<ElementRef<HTMLInputElement>>('searchInput');

  focus() {
    this.inputRef()?.nativeElement.focus();
  }
}
```

```html
<input #searchInput type="text" />
```

Advantages: available as a signal (so it composes with `computed`/`effect`), and resolved before `ngAfterViewInit` in some cases, removing timing footguns.

### Signal ↔ RxJS interop

**Definition:** Signal–RxJS interop is the pair of conversion utilities, `toSignal()` and `toObservable()` (from `@angular/core/rxjs-interop`), that bridge Angular's signal graph and RxJS's observable streams.

```ts
import { toSignal, toObservable } from '@angular/core/rxjs-interop';

// Observable -> Signal (e.g., for use in templates)
const user$ = this.userService.getUser();
const user = toSignal(user$, { initialValue: null });

// Signal -> Observable (e.g., to feed into RxJS operators)
const searchTerm = signal('');
const searchTerm$ = toObservable(searchTerm);

searchTerm$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.api.search(term))
).subscribe(results => this.results.set(results));
```

### When to use Signals vs RxJS vs NgRx

| Use case | Prefer |
|---|---|
| Simple synchronous UI state (toggle, counter, form field) | Signals |
| Derived/computed UI state | `computed()` |
| Async streams, cancellation, complex operator chains (debounce, retry, combine) | RxJS |
| Global, cross-feature state with time-travel debugging / devtools | NgRx (or a signal store like NgRx SignalStore) |
| Component-local state that used to live in a `BehaviorSubject` | Signals — simpler, no subscription management, no memory leaks |

Rule of thumb: **Signals for state, RxJS for events/streams.** Signals don't replace RxJS — they replace a lot of the *state-holding* patterns that people used to build on top of `BehaviorSubject` + `async` pipe.

---

## 5. Dependency Injection — Internals

**Definition:** Dependency Injection (DI) is a design pattern in which a class declares the dependencies it needs and receives them from an external source (an injector), rather than constructing them itself. Angular's DI system is a hierarchical tree of injectors that resolve and supply those dependencies.

### Providers

**Definition:** A provider is a configuration entry that tells an injector *how to construct* the value associated with a given DI token.

```ts
providers: [
  UserService,                                       // shorthand for { provide: UserService, useClass: UserService }
  { provide: Logger, useClass: ConsoleLogger },        // swap implementation
  { provide: API_BASE_URL, useValue: 'https://api.example.com' }, // static value
  { provide: WINDOW, useFactory: () => window },       // computed value, can inject deps
  { provide: LegacyLogger, useExisting: Logger },      // alias — same instance as Logger
]
```

- **`useClass`** — instantiate a (possibly different) class for the token.
- **`useValue`** — provide a plain value (config objects, constants, mocks in tests).
- **`useFactory`** — provide a value computed by a function, optionally with its own deps:
  ```ts
  { provide: API_CLIENT, useFactory: (http: HttpClient) => new ApiClient(http), deps: [HttpClient] }
  ```
- **`useExisting`** — alias one token to an already-registered provider, so both resolve to the *same instance*.

### `providedIn`

**Definition:** `providedIn` is metadata on an `@Injectable()` service that registers it directly with a specific injector scope (`'root'`, `'platform'`, or a module), without needing to be listed in that scope's `providers` array.

```ts
@Injectable({ providedIn: 'root' })   // singleton for the whole app (tree-shakable if unused)
export class AuthService {}

@Injectable({ providedIn: 'platform' })  // shared across multiple Angular apps on one page
export class PlatformLogger {}

@Injectable({ providedIn: SomeFeatureModule })  // scoped to one NgModule (legacy pattern)
export class FeatureService {}
```

`providedIn: 'root'` is preferred: it's tree-shakable (unused services are dropped from the bundle) unlike listing a service in a module's `providers` array.

### Injection tokens

**Definition:** An `InjectionToken` is a unique, type-carrying object used as a DI token for values that have no class to serve as a token — interfaces, primitives, and plain configuration objects don't exist at runtime, so they can't be used as tokens directly.

```ts
import { InjectionToken } from '@angular/core';

export interface AppConfig { apiUrl: string; retries: number; }
export const APP_CONFIG = new InjectionToken<AppConfig>('APP_CONFIG');

// registration
providers: [{ provide: APP_CONFIG, useValue: { apiUrl: '/api', retries: 3 } }]

// consumption
export class ApiService {
  private config = inject(APP_CONFIG);
}
```

### Hierarchical injectors

**Definition:** Hierarchical injection is the property that Angular's injectors form a tree mirroring the component tree; resolving a token walks **up** this tree from the requesting component until a matching provider is found.

```
Element injector (this component's own `providers: []`)
      ↓ (not found)
Parent element injectors (up the component tree)
      ↓ (not found)
Environment injector (root / module-level)
      ↓ (not found)
Platform injector
      ↓ (not found)
NullInjector — throws "NoProviderError"
```

A provider declared on a component (`providers: [FeatureService]` in `@Component`) creates a **new instance per component instance** — useful for per-instance state (e.g. each `<app-tab>` getting its own `TabStateService`).

```ts
@Component({
  selector: 'app-tab',
  providers: [TabStateService],   // one instance PER <app-tab> in the DOM
  template: `...`,
})
export class TabComponent {}
```

Lookup modifiers change traversal:

```ts
@Optional() private logger?: Logger;         // don't throw if not found — value is null
@Self() private local: LocalService;          // only look in THIS element's injector
@SkipSelf() private parent: SharedService;    // skip this element, start at the parent
@Host() private hostSvc: HostService;         // stop at the host component boundary
```

### Environment injectors

**Definition:** An environment injector is an injector scoped to the application as a whole (or to a lazily-loaded route), as opposed to an **element injector**, which is scoped to a specific component/directive instance in the template.

Since Ivy, DI has two injector hierarchies:
- **Environment injectors** — root, platform, and (for lazy-loaded routes) per-route injectors. Configured via `providers` in `ApplicationConfig` / `bootstrapApplication` / route config.
- **Element injectors** — per-component/directive, configured via `providers`/`viewProviders` in `@Component`/`@Directive`.

Environment injectors are what `providedIn: 'root'` registers into; they back lazy-loading (`provideRouter` with lazy routes can attach route-scoped providers).

### `inject()`

**Definition:** `inject()` is a function that retrieves a dependency from the currently active injector, callable anywhere within an injection context (constructors, field initializers, factory functions, functional guards/interceptors) as an alternative to constructor-parameter injection.

```ts
@Component({...})
export class ProfileComponent {
  private userService = inject(UserService);
  private route = inject(ActivatedRoute);
  private config = inject(APP_CONFIG, { optional: true });
}
```

Advantages over constructor injection: works in functions (functional guards/interceptors/resolvers), composes well in base classes without `super()` boilerplate, and reads top-to-bottom instead of being buried in a constructor signature.

### Provider scopes summary

| Scope | How | Lifetime |
|---|---|---|
| App singleton | `providedIn: 'root'` | One instance, whole app |
| Route-lazy singleton | `providedIn` on a lazy-loaded feature, or route `providers` | One instance per loaded route injector |
| Component-scoped | `providers: []` in `@Component` | New instance per component instance |
| View-only | `viewProviders: []` | Like `providers`, but invisible to content-projected children |

### Dependency resolution & custom DI patterns

**Definition:** A multi-provider is a provider configuration (`multi: true`) that lets several providers contribute values to the *same* token, resolved as an array rather than a single instance.

```ts
// Multi-provider: collect multiple values under one token (e.g., plugin lists)
export const VALIDATORS = new InjectionToken<Validator[]>('VALIDATORS');
providers: [
  { provide: VALIDATORS, useClass: RequiredValidator, multi: true },
  { provide: VALIDATORS, useClass: EmailValidator, multi: true },
]
// inject(VALIDATORS) -> Validator[] (both instances)
```

```ts
// Factory pattern for environment-dependent services
export function loggerFactory(config: AppConfig) {
  return config.production ? new RemoteLogger() : new ConsoleLogger();
}
providers: [{ provide: Logger, useFactory: loggerFactory, deps: [APP_CONFIG] }]
```

---

## 6. RxJS — Angular Deep Dive

**Definition:** RxJS (Reactive Extensions for JavaScript) is a library for composing asynchronous and event-based programs using **Observables** — lazy, push-based sequences of values over time.

### Core building blocks

**Observable — Definition:** a lazy, push-based stream of values that does nothing until something subscribes to it, at which point it may emit zero or more values (`next`), optionally followed by an error or a completion signal.

```ts
import { Observable, Subject, BehaviorSubject, ReplaySubject, AsyncSubject } from 'rxjs';

// Observable: lazy, unicast by default — code inside only runs per subscriber
const nums$ = new Observable<number>(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.complete();
});

const sub = nums$.subscribe({
  next: v => console.log(v),
  error: err => console.error(err),
  complete: () => console.log('done'),
});
sub.unsubscribe(); // Subscription — always cleanable
```

**Subject — Definition:** a value that is simultaneously an `Observable` (can be subscribed to) and an `Observer` (has `.next()`/`.error()`/`.complete()`), used to multicast values to many subscribers imperatively.

| Subject type | Behavior |
|---|---|
| `Subject` | No initial value; late subscribers miss past emissions. Pure multicast. |
| `BehaviorSubject` | Requires initial value; new subscribers get the **current** value immediately. |
| `ReplaySubject(n)` | New subscribers get the last `n` emitted values replayed. |
| `AsyncSubject` | Emits only the **final** value, and only on `complete()`. |

```ts
const state$ = new BehaviorSubject({ loggedIn: false });
state$.subscribe(s => console.log(s));   // fires immediately with current value
state$.next({ loggedIn: true });
```

### Operators (with the ones that come up constantly)

**Definition:** An operator is a pure function that takes an Observable as input and returns a new Observable, used inside `.pipe()` to transform, filter, or combine streams without mutating the source.

```ts
import { of, from, combineLatest, forkJoin, zip } from 'rxjs';
import {
  map, filter, switchMap, mergeMap, concatMap, exhaustMap,
  catchError, retry, debounceTime, distinctUntilChanged, shareReplay,
} from 'rxjs/operators';

search$.pipe(
  debounceTime(300),          // wait for typing pause
  distinctUntilChanged(),      // skip if same as last value
  switchMap(term =>            // cancel previous in-flight request when a new one starts
    this.api.search(term).pipe(
      catchError(() => of([])) // swallow error, fall back to empty results
    )
  ),
).subscribe(results => this.results = results);
```

**Flattening operator cheat sheet** (all take an Observable-returning function):

| Operator | Behavior | Typical use |
|---|---|---|
| `switchMap` | Cancels the previous inner observable when a new outer value arrives | Typeahead search, "latest wins" |
| `mergeMap` | Runs all inner observables concurrently | Parallel independent requests |
| `concatMap` | Queues inner observables, runs one at a time in order | Sequential writes that must not interleave |
| `exhaustMap` | Ignores new outer values while an inner observable is still running | Prevent double-submit on a button |

```ts
// combineLatest — emits whenever ANY source emits, using latest from all
combineLatest([user$, settings$]).subscribe(([user, settings]) => { /* ... */ });

// forkJoin — waits for ALL to complete, emits once with the final values (like Promise.all)
forkJoin({ user: this.api.getUser(), orders: this.api.getOrders() })
  .subscribe(({ user, orders }) => { /* ... */ });

// zip — pairs emissions by index across sources
zip(a$, b$).subscribe(([a, b]) => { /* ... */ });
```

```ts
// retry with backoff-ish pattern
this.api.getData().pipe(
  retry({ count: 3, delay: 1000 }),
  catchError(err => { this.toast.error('Failed'); return throwError(() => err); }),
).subscribe();
```

`shareReplay(1)` turns a cold observable (e.g. an HTTP call) into a hot, cached one shared by multiple subscribers — common for "load once, reuse everywhere" config/data:

```ts
readonly config$ = this.http.get<AppConfig>('/api/config').pipe(shareReplay(1));
```

### Memory leaks & `takeUntilDestroyed()`

**Definition:** In RxJS, a memory leak occurs when a subscription is created but never unsubscribed, keeping its callback closure — and everything that closure references — alive indefinitely, even after the component that created it is destroyed.

Every manual `.subscribe()` needs a matching unsubscribe, or the subscription leaks for the app's lifetime. Modern Angular provides `takeUntilDestroyed()` to tie a subscription to the current injection context automatically:

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({...})
export class LiveFeedComponent {
  constructor() {
    interval(1000).pipe(
      takeUntilDestroyed(), // auto-unsubscribes on ngOnDestroy
    ).subscribe(tick => this.tick = tick);
  }
}
```

Outside a constructor/field initializer, pass the `DestroyRef` explicitly:

```ts
export class LiveFeedComponent {
  private destroyRef = inject(DestroyRef);

  start() {
    interval(1000).pipe(takeUntilDestroyed(this.destroyRef)).subscribe(/* ... */);
  }
}
```

Prefer the `| async` pipe in templates over manual `subscribe()` + field assignment — Angular unsubscribes automatically when the template is destroyed.

### Signals + RxJS

See [Signal ↔ RxJS interop](#4-signals) above — `toSignal()`/`toObservable()` are the standard bridge.

---

## 7. Forms

**Definition:** An Angular form is a mechanism for capturing, validating, and submitting user input, built either declaratively in the template (**template-driven**) or programmatically as an explicit object model in the component class (**reactive**).

### Template-driven vs Reactive

| | Template-driven | Reactive |
|---|---|---|
| Source of truth | The DOM (via `ngModel`) | The component class (`FormGroup`) |
| Setup | Declarative in template | Programmatic in TS |
| Best for | Simple forms | Complex, dynamic, or heavily-validated forms |
| Testability | Harder (needs DOM) | Easier (pure TS) |

```html
<!-- template-driven -->
<form #f="ngForm" (ngSubmit)="save(f.value)">
  <input name="email" [(ngModel)]="model.email" required email />
</form>
```

### Reactive forms

**Definition:** Reactive forms model form state explicitly, as a tree of `FormControl`, `FormGroup`, and `FormArray` objects constructed in the component class, with the template merely binding to that pre-built model.

```ts
import { FormBuilder, FormGroup, FormArray, Validators } from '@angular/forms';

@Component({...})
export class SignupComponent {
  private fb = inject(FormBuilder);

  form: FormGroup = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(8)]],
    addresses: this.fb.array([this.addressGroup()]),
  });

  addressGroup() {
    return this.fb.group({ street: [''], city: [''] });
  }

  get addresses() { return this.form.get('addresses') as FormArray; }
  addAddress() { this.addresses.push(this.addressGroup()); }

  save() {
    if (this.form.invalid) { this.form.markAllAsTouched(); return; }
    console.log(this.form.value);
  }
}
```

```html
<form [formGroup]="form" (ngSubmit)="save()">
  <input formControlName="email" />
  <div *ngIf="form.get('email')?.hasError('required')">Required</div>

  <div formArrayName="addresses">
    <div *ngFor="let addr of addresses.controls; let i = index" [formGroupName]="i">
      <input formControlName="street" />
    </div>
  </div>
</form>
```

### Validators

**Definition:** A validator is a pure function that receives a form control (or group) and returns an error object describing what's wrong, or `null` if the current value is valid.

```ts
// built-in
Validators.required
Validators.minLength(3)
Validators.pattern(/^[a-z]+$/)

// custom synchronous validator
function forbiddenNameValidator(forbidden: RegExp): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null =>
    forbidden.test(control.value) ? { forbiddenName: { value: control.value } } : null;
}

// custom async validator (e.g. uniqueness check against a backend)
function uniqueEmailValidator(api: ApiService): AsyncValidatorFn {
  return (control: AbstractControl) =>
    api.checkEmail(control.value).pipe(
      map(taken => (taken ? { emailTaken: true } : null)),
    );
}

// cross-field validation, applied at the FormGroup level
function passwordsMatch(group: AbstractControl): ValidationErrors | null {
  const pass = group.get('password')?.value;
  const confirm = group.get('confirmPassword')?.value;
  return pass === confirm ? null : { mismatch: true };
}

this.form = this.fb.group(
  { password: [''], confirmPassword: [''] },
  { validators: passwordsMatch },
);
```

### Typed forms (Angular 14+)

**Definition:** Typed forms are reactive forms whose controls carry an explicit TypeScript type for their value, so the compiler — not just runtime behavior — enforces the correct shape wherever the form's value is read or written.

```ts
interface SignupForm {
  email: FormControl<string>;
  password: FormControl<string>;
}

const form = this.fb.group<SignupForm>({
  email: this.fb.control('', { nonNullable: true, validators: [Validators.required] }),
  password: this.fb.control('', { nonNullable: true }),
});

form.value.email; // string | undefined, fully typed — no more `any`
```

### `ControlValueAccessor` — custom form components

**Definition:** `ControlValueAccessor` is an interface that bridges a custom component to Angular's forms API, allowing it to be used with `formControlName`/`ngModel` exactly like a native `<input>`.

```ts
@Component({
  selector: 'app-rating',
  template: `<button *ngFor="let s of stars" (click)="select(s)">★</button>`,
  providers: [{
    provide: NG_VALUE_ACCESSOR,
    useExisting: forwardRef(() => RatingComponent),
    multi: true,
  }],
})
export class RatingComponent implements ControlValueAccessor {
  value = 0;
  stars = [1, 2, 3, 4, 5];
  private onChange: (v: number) => void = () => {};
  private onTouched: () => void = () => {};

  writeValue(v: number): void { this.value = v; }
  registerOnChange(fn: (v: number) => void): void { this.onChange = fn; }
  registerOnTouched(fn: () => void): void { this.onTouched = fn; }

  select(s: number) {
    this.value = s;
    this.onChange(s);
    this.onTouched();
  }
}
```

```html
<app-rating formControlName="rating" />
```

---

## 8. Routing

**Definition:** The Angular Router is a built-in module that maps URL paths to components, enabling navigation between "pages" within a single-page application without a full browser reload.

### Basic setup (standalone)

```ts
// app.routes.ts
export const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'users/:id', component: UserDetailComponent },
  { path: 'admin', loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES) },
  { path: '**', component: NotFoundComponent },
];

// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [provideRouter(routes)],
};
```

```html
<!-- app.component.html -->
<nav><a routerLink="/users/1" routerLinkActive="active">User 1</a></nav>
<router-outlet />
```

### Route/query params

**Definition:** A route parameter is a dynamic segment of a URL path (e.g. `:id` in `/users/:id`); a query parameter is a key-value pair appended after `?` (e.g. `?page=2`). Both are exposed through `ActivatedRoute`.

```ts
export class UserDetailComponent {
  private route = inject(ActivatedRoute);

  // snapshot — value at the moment of navigation, doesn't update on param change
  id = this.route.snapshot.paramMap.get('id');

  // observable — updates if the SAME component instance is reused for a new param
  id$ = this.route.paramMap.pipe(map(params => params.get('id')));

  page$ = this.route.queryParamMap.pipe(map(q => q.get('page') ?? '1'));
}
```

### Lazy loading

**Definition:** Lazy loading is the deferral of a route's JavaScript bundle until the user actually navigates to that route, rather than including it in the initial page load.

```ts
{ path: 'admin', loadComponent: () => import('./admin/admin.component').then(m => m.AdminComponent) }
{ path: 'admin', loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES) }
```

Each lazy chunk becomes a separate JS bundle, fetched only when the route is visited — smaller initial load.

### Guards (functional, modern style)

**Definition:** A route guard is a function the Router invokes before activating, deactivating, or loading a route, returning `true`/`false`/a `UrlTree` to allow, block, or redirect the navigation.

```ts
export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  const router = inject(Router);
  return auth.isLoggedIn() ? true : router.parseUrl('/login');
};

export const unsavedChangesGuard: CanDeactivateFn<EditorComponent> = (component) =>
  component.isDirty() ? confirm('Discard unsaved changes?') : true;

{ path: 'admin', canActivate: [authGuard], component: AdminComponent }
```

### Resolvers

**Definition:** A resolver is a function that pre-fetches data before a route is activated, so the destination component renders with data already present instead of showing a loading state internally.

```ts
export const userResolver: ResolveFn<User> = (route) => {
  const api = inject(UserApi);
  return api.getUser(route.paramMap.get('id')!);
};

{ path: 'users/:id', component: UserDetailComponent, resolve: { user: userResolver } }
```

```ts
export class UserDetailComponent {
  private route = inject(ActivatedRoute);
  user = this.route.snapshot.data['user'];
}
```

### Preloading strategies

**Definition:** A preloading strategy determines which lazy-loaded route bundles the Router fetches in the background after the initial app load, before the user has navigated to them.

```ts
provideRouter(routes, withPreloading(PreloadAllModules))
// or a custom strategy that preloads only routes flagged with data: { preload: true }
```

### Navigation lifecycle (high level)

```
NavigationStart → route matching → guards (CanDeactivate old, CanActivate new)
  → resolvers run → ResolveEnd → component instantiated → NavigationEnd
```

Errors or guard rejections short-circuit to `NavigationCancel`/`NavigationError`.

---

## 9. HTTP & APIs

**Definition:** `HttpClient` is Angular's built-in service for issuing HTTP requests to a backend, returning each request's response as an Observable.

### `HttpClient` basics

```ts
export class UserApi {
  private http = inject(HttpClient);

  getUsers() { return this.http.get<User[]>('/api/users'); }
  getUser(id: number) { return this.http.get<User>(`/api/users/${id}`); }
  create(user: Partial<User>) { return this.http.post<User>('/api/users', user); }
  update(id: number, patch: Partial<User>) { return this.http.patch<User>(`/api/users/${id}`, patch); }
  delete(id: number) { return this.http.delete<void>(`/api/users/${id}`); }

  search(term: string) {
    return this.http.get<User[]>('/api/users', {
      params: new HttpParams().set('q', term),
      headers: new HttpHeaders({ 'X-Client': 'web' }),
    });
  }
}
```

### Interceptors (functional, modern style)

**Definition:** An HTTP interceptor is a function that sits in the request/response pipeline, able to inspect or modify every outgoing `HttpRequest` and incoming `HttpEvent` that passes through `HttpClient`.

```ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);
  const token = auth.token();
  const authedReq = token
    ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
    : req;
  return next(authedReq);
};

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const toast = inject(ToastService);
  return next(req).pipe(
    catchError(err => {
      toast.error(err.status === 401 ? 'Session expired' : 'Request failed');
      return throwError(() => err);
    }),
  );
};

// app.config.ts
provideHttpClient(withInterceptors([authInterceptor, errorInterceptor]))
```

### Refresh-token pattern (sketch)

```ts
export const refreshInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthService);
  return next(req).pipe(
    catchError(err => {
      if (err.status === 401) {
        return auth.refreshToken().pipe(
          switchMap(() => next(req.clone({ setHeaders: { Authorization: `Bearer ${auth.token()}` } }))),
        );
      }
      return throwError(() => err);
    }),
  );
};
```

### Request cancellation

**Definition:** Request cancellation is the aborting of an in-flight HTTP request before it completes — in Angular, achieved simply by unsubscribing from the request's Observable, which aborts the underlying XHR/fetch.

```ts
// switchMap auto-cancels the previous request when a new one starts
this.searchTerm$.pipe(switchMap(term => this.api.search(term))).subscribe();

// manual cancellation: just unsubscribe — Angular's HttpClient aborts the underlying XHR/fetch
const sub = this.http.get('/api/slow').subscribe();
sub.unsubscribe(); // cancels the in-flight request
```

### API caching

```ts
@Injectable({ providedIn: 'root' })
export class ConfigService {
  private http = inject(HttpClient);
  private config$ = this.http.get<AppConfig>('/api/config').pipe(shareReplay(1));
  getConfig() { return this.config$; } // subsequent calls reuse the cached response
}
```

---

## 10. State Management

**Definition:** State management is the set of patterns and tools used to store, share, and update application data across components in a predictable way.

Understand the progression:

```
Component State
      ↓
Services
      ↓
RxJS
      ↓
Signals
      ↓
NgRx
```

- **Component state** — a plain field or `signal()` local to one component; fine for anything not shared.
- **Service-based state** — an injectable service holding a `signal()` (or historically a `BehaviorSubject`), shared by any component that injects it:

```ts
@Injectable({ providedIn: 'root' })
export class CartService {
  private _items = signal<CartItem[]>([]);
  readonly items = this._items.asReadonly();
  readonly total = computed(() => this._items().reduce((s, i) => s + i.price, 0));

  add(item: CartItem) { this._items.update(list => [...list, item]); }
}
```

This "signal store in a service" pattern covers the majority of apps that used to reach for NgRx just to share state across unrelated components.

### NgRx Store (when you need it)

**Definition:** NgRx is a state-management library for Angular that implements the Redux pattern — a single immutable store, pure reducer functions, and a strictly unidirectional data flow driven by dispatched **actions**.

Reach for NgRx when you have: complex state transitions that benefit from a strict unidirectional data flow, time-travel debugging needs, or many features reading/writing overlapping state where explicit actions make the data flow auditable.

```ts
// actions
export const loadUsers = createAction('[Users] Load');
export const loadUsersSuccess = createAction('[Users] Load Success', props<{ users: User[] }>());

// reducer
export const usersReducer = createReducer(
  initialState,
  on(loadUsers, state => ({ ...state, loading: true })),
  on(loadUsersSuccess, (state, { users }) => ({ ...state, loading: false, users })),
);

// selector
export const selectUsers = createSelector(selectUsersState, s => s.users);

// effect
@Injectable()
export class UsersEffects {
  private actions$ = inject(Actions);
  private api = inject(UserApi);

  loadUsers$ = createEffect(() => this.actions$.pipe(
    ofType(loadUsers),
    switchMap(() => this.api.getUsers().pipe(
      map(users => loadUsersSuccess({ users })),
    )),
  ));
}
```

```ts
// component
export class UsersComponent {
  private store = inject(Store);
  users$ = this.store.select(selectUsers);
  ngOnInit() { this.store.dispatch(loadUsers()); }
}
```

**NgRx SignalStore** (newer, lighter alternative to classic Store) gives NgRx-style structure using signals instead of Observables/reducers — worth learning once classic NgRx concepts are solid.

### NgRx vs Signals — decision guide

| Situation | Choice |
|---|---|
| 2–3 components share simple state | Signal-based service |
| Deeply nested state, many features read/write it, need devtools time-travel | NgRx Store |
| Team already knows Redux patterns, wants enforced discipline | NgRx |
| Small-to-medium app, want minimal boilerplate | Signals (service or SignalStore) |

Avoid reaching for global state (NgRx or otherwise) for state that's genuinely local to one feature or component — it adds indirection without payoff.

---

## 11. Angular Architecture

**Definition:** Application architecture, in this context, is how a codebase's folders, modules, and components are organized so a large Angular application stays maintainable, testable, and safe to change as it grows.

Learn how to build large applications:

```
src/
 ├── app/
 │    ├── core/           # singletons: auth, logging, app-wide interceptors/guards
 │    ├── shared/          # dumb/reusable components, pipes, directives — no business logic
 │    ├── features/        # one folder per feature/domain, lazy-loaded
 │    │    └── orders/
 │    │         ├── components/
 │    │         ├── services/
 │    │         └── orders.routes.ts
 │    ├── layout/          # shell: header, sidebar, footer
 │    ├── guards/
 │    ├── interceptors/
 │    └── app.routes.ts
 └── assets/
```

**Feature-based structure** beats grouping by file type (`components/`, `services/`, `pipes/` at the top level) once an app grows past a handful of screens — it keeps everything for one feature co-located and makes lazy-loading boundaries obvious.

**Smart vs presentational components. Definition:** a **smart** (container) component injects services, fetches data, and holds state; a **presentational** (dumb) component receives data purely through `@Input()`/`@Output()` and contains no business logic — separating the two maximizes reusability and testability.

```ts
// smart/container — knows about services, fetches data, holds state
@Component({ selector: 'app-order-list-page', template: `
  <app-order-list [orders]="orders()" (select)="openOrder($event)" />
` })
export class OrderListPageComponent {
  private orderApi = inject(OrderApi);
  orders = toSignal(this.orderApi.getOrders(), { initialValue: [] });
  openOrder(id: string) { /* navigate */ }
}

// presentational/dumb — pure inputs/outputs, no DI of business services, easy to test & reuse
@Component({ selector: 'app-order-list', changeDetection: ChangeDetectionStrategy.OnPush, template: `
  <li *ngFor="let o of orders" (click)="select.emit(o.id)">{{ o.name }}</li>
` })
export class OrderListComponent {
  @Input() orders: Order[] = [];
  @Output() select = new EventEmitter<string>();
}
```

- **`core/`** — services meant to exist exactly once (`AuthService`, global interceptors) — import-guard it so it's only ever provided at the root.
- **`shared/`** — genuinely reusable, presentation-only pieces (buttons, date pipes) with zero feature-specific logic — prevents circular dependencies between features.
- **Dependency boundary rule of thumb:** features can depend on `core`/`shared`, never on each other directly; cross-feature communication goes through a service in `core` or through routing/query params.

**Monorepos / Nx** — for multiple apps or a large app split into publishable libraries, Nx enforces module boundaries (`@nx/enforce-module-boundaries`), gives dependency graphs, and only rebuilds/retests what actually changed (affected-based CI). Worth adopting once you have >1 deployable or a codebase big enough that a full rebuild/test run is slow.

**Micro-frontends** — splitting a large app into independently deployable pieces (e.g. via Module Federation). High complexity cost; only justified at real organizational scale (multiple independent teams shipping on their own cadence into one shell app).

---

## 12. Performance Optimization

**Definition:** Performance optimization in Angular is the practice of minimizing unnecessary change-detection work, bundle size, and rendering cost so the application stays fast as data volume and feature count grow.

Very important for senior interviews.

- **`OnPush` everywhere** — pairs with immutable data/signals to skip whole subtrees during CD.
- **Signals** — fine-grained reactivity means only the DOM nodes that actually depend on a changed signal re-render, not the whole component.
- **`track` in `@for`** — always provide a stable key (`track item.id`), never `track $index` for reorderable/mutable lists, so Angular patches instead of recreating DOM nodes.
- **Lazy loading** (`loadComponent`/`loadChildren`) — ships only the code the current route needs.
- **Deferrable views (`@defer`)** — defer rendering (and its JS) of below-the-fold or non-critical UI:

```html
@defer (on viewport) {
  <app-heavy-chart [data]="data()" />
} @placeholder {
  <div class="chart-skeleton"></div>
} @loading (minimum 200ms) {
  <app-spinner />
} @error {
  <p>Failed to load chart.</p>
}
```

Triggers: `on viewport`, `on idle`, `on interaction`, `on timer(2s)`, `on hover`, or `when someCondition()`.

- **Code splitting / bundle optimization / tree shaking** — mostly automatic via `ng build` + ESM + standalone APIs (unused providers/components are dropped); avoid barrel-importing an entire library when you need one function.
- **Pure pipes — Definition:** a pipe with `pure: true` (the default) only re-runs its `transform()` when its input *reference* changes, unlike a template method call, which re-runs every CD cycle:

```ts
@Pipe({ name: 'formatCurrency', pure: true, standalone: true })
export class FormatCurrencyPipe implements PipeTransform {
  transform(value: number) { return `$${value.toFixed(2)}`; }
}
```

- **Memoization** — `computed()` is memoization built into the framework; for non-signal code, cache expensive derived values manually rather than recomputing in the template.
- **Virtual scrolling — Definition:** a rendering technique that keeps only the DOM nodes for the currently visible slice of a long list, recycling them as the user scrolls, instead of rendering every item:

```html
<cdk-virtual-scroll-viewport itemSize="48" class="viewport">
  <div *cdkVirtualFor="let item of items">{{ item.name }}</div>
</cdk-virtual-scroll-viewport>
```

- **Avoid expensive template functions** — `{{ compute() }}` runs on every CD cycle; hoist to a `computed()` signal or a pure pipe.
- **Avoid unnecessary subscriptions** — prefer `| async`/`toSignal` over manual `subscribe()`, and always clean up with `takeUntilDestroyed()` when manual subscription is unavoidable.
- **Web workers — Definition:** background threads that run JavaScript off the main UI thread, used to offload CPU-heavy work (parsing, image processing) so it doesn't block rendering: `ng generate web-worker`.
- **SSR / Hydration** — see section 15; improves perceived load time and Core Web Vitals (especially LCP).
- **Profiling** — Angular DevTools' Profiler tab records a CD cycle and shows per-component time, so you can find the exact component blowing the render budget instead of guessing.

---

## 13. Angular Internals

**Definition:** Angular internals refers to the framework's own runtime implementation — the compiler and renderer (Ivy/Render3) — that turns a component's template into a live, self-updating DOM tree.

This is where we go beyond normal tutorials.

Understand:

```
Angular Application
        ↓
Bootstrap        (bootstrapApplication / platformBrowser)
        ↓
Dependency Injection   (root EnvironmentInjector built from providers)
        ↓
Component Tree     (root component instantiated, DI resolved per node)
        ↓
Template Compilation  (AOT, done at build time — produces ɵɵdefineComponent + instructions)
        ↓
Rendering        (Ivy "Render3" walks the compiled instructions, builds LView/TView)
        ↓
Change Detection    (dirty-checks bindings top-down, patches only what changed)
        ↓
DOM Update
```

**Ivy / Render3 — Definition:** Ivy (codename Render3) is Angular's compilation-and-rendering pipeline, the default since Angular 9, which compiles each component (ahead-of-time) into a self-contained **component definition** (`ɵcmp`) made of low-level template **instructions**, rather than relying on a shared runtime interpreter:

```ts
// (simplified, generated code — you never write this by hand)
ɵɵdefineComponent({
  type: AppComponent,
  selectors: [['app-root']],
  template: function AppComponent_Template(rf, ctx) {
    if (rf & 1) { // creation mode — runs once
      ɵɵelementStart(0, 'h1');
      ɵɵtext(1);
      ɵɵelementEnd();
    }
    if (rf & 2) { // update mode — runs every CD cycle
      ɵɵadvance(1);
      ɵɵtextInterpolate1('Hello, ', ctx.name, '');
    }
  },
});
```

Each template function runs twice per node conceptually: **creation mode** (build the DOM once) and **update mode** (only touch bindings that could have changed) — this split is why Ivy is fast and why templates are tree-shakable (unused directives/components are never even referenced in the compiled output).

**`LView` / `TView` — Definition:** the two runtime data structures behind every component instance:
- **`TView`** ("Template View") — the *static*, shared-per-component-type description: binding indices, instruction pointers, directive metadata. Created once per component **class**, reused by every instance.
- **`LView`** ("Logical View") — the *per-instance* array holding actual values: DOM node references, component instance, binding values from the last CD run (used to compare "did this change?"), child view references.

This split (static shape in `TView`, live data in `LView`) is why Ivy CD is fast: comparing "did binding #4 change" is an array index lookup, not a tree walk or string-based diff.

**View hierarchy — Definition:** the runtime tree of `LView` instances — one per component instance, plus one per structural-directive-created embedded view (`@if`/`@for` blocks) — that mirrors, but is not identical to, the logical component tree.

**Zone-based vs zoneless execution** — see section 3; internally, Zone.js patches async APIs so `ApplicationRef.tick()` (which walks every `LView` doing CD) gets triggered automatically after any async callback. Zoneless relies on explicit "this LView is dirty" signals (from a changed `signal()`, `markForCheck()`, etc.) added to a notification queue instead.

**AOT vs JIT recap** — AOT does the "template → instructions" compilation at build time (via the Angular compiler, `@angular/compiler-cli`); JIT did it in-browser at bootstrap. AOT is the default and only supported mode for production builds today — smaller runtime (no compiler shipped to the browser), and template errors surface as build failures instead of runtime exceptions.

---

## 14. Standalone Angular

**Definition:** A standalone component, directive, or pipe is one that does not require registration in an `NgModule`; it declares its own template dependencies directly via an `imports` array, and an application built entirely from standalone pieces can bootstrap without a root module at all.

Modern Angular heavily uses standalone APIs.

```ts
// a fully standalone component
@Component({
  selector: 'app-user-card',
  standalone: true,               // default `true` since Angular 19; explicit here for clarity
  imports: [CommonModule, RouterLink, FormatCurrencyPipe],
  template: `...`,
})
export class UserCardComponent {}
```

Standalone directives/pipes work the same way — declare `standalone: true` (or rely on the default) and list them in a consuming component's `imports`.

### Bootstrapping without a root module

**Definition:** `bootstrapApplication()` is the standalone entry point that launches an Angular application directly from a root component and a set of providers, replacing the classic `platformBrowserDynamic().bootstrapModule(AppModule)` call.

```ts
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, appConfig);
```

```ts
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptors([authInterceptor])),
    provideAnimations(),
    { provide: APP_CONFIG, useValue: environment },
  ],
};
```

`ApplicationConfig` replaces `AppModule`'s `providers` array; `provideRouter`/`provideHttpClient`/etc. are **functional providers** — tree-shakable feature setup functions instead of importing a whole `NgModule`.

### Functional guards / interceptors (recap, see sections 8–9)

Standalone architecture leans on functions over classes throughout: `CanActivateFn`, `HttpInterceptorFn`, `ResolveFn` are all plain functions using `inject()`, replacing the old class-based `implements CanActivate` pattern.

### Migrating from NgModules

```bash
ng generate @angular/core:standalone   # official schematic, run in phases:
# 1. Convert declarations to standalone
# 2. Remove unnecessary NgModules
# 3. Convert bootstrapping to bootstrapApplication
```

Migration is incremental — standalone components can be declared inside an `NgModule` (via `imports:`) and NgModule-based components can still be lazy-loaded alongside standalone ones during a gradual migration.

---

## 15. SSR & Full-Stack Angular

**Definition:** Server-Side Rendering (SSR) is the process of rendering an Angular application to a static HTML string on the server, before it reaches the browser, rather than starting from an empty page and rendering entirely client-side.

### Angular SSR (`@angular/ssr`)

```bash
ng add @angular/ssr
```

This scaffolds a server entry point that renders the app to an HTML string on the server (Node.js), so the browser receives fully-formed HTML on first load instead of an empty `<app-root></app-root>` waiting for JS.

```ts
// server.ts (simplified)
const app = express();
app.get('*', (req, res, next) => {
  commonEngine
    .render({ bootstrap: bootstrapAppServer, url: req.url, documentFilePath: indexHtml })
    .then(html => res.send(html));
});
```

### Hydration

**Definition:** Hydration is the process by which the client-side Angular application "attaches" to already-rendered server HTML — reusing the existing DOM nodes and wiring up event listeners/reactivity — instead of discarding that markup and re-rendering from scratch.

Without hydration, the client used to throw away the server-rendered DOM and rebuild it (causing a visible flicker and wasted work). **Non-destructive hydration** (stable since Angular 17) reuses the existing server-rendered DOM nodes and just attaches event listeners/reactivity to them:

```ts
// app.config.ts
providers: [provideClientHydration()]
```

**Incremental hydration** (newer) goes further — pairs with `@defer` so parts of the page hydrate (become interactive) lazily, on the same triggers as deferred rendering (`on viewport`, `on interaction`, etc.), instead of the whole page hydrating at once on load.

### Browser/server differences

Server-side code has no `window`, `document`, `localStorage`, etc. Guard platform-specific code:

```ts
import { isPlatformBrowser } from '@angular/common';

export class WidgetComponent {
  private platformId = inject(PLATFORM_ID);

  ngOnInit() {
    if (isPlatformBrowser(this.platformId)) {
      localStorage.getItem('theme');
    }
  }
}
```

### SEO & performance payoff

SSR gives crawlers (and social-share link previews) real HTML immediately, and improves Largest Contentful Paint since the browser doesn't wait for JS to download/parse/execute before showing content.

### Angular + backend

- **+ Node.js** — the SSR server itself is typically Express/Node; can share the same process as an API server or sit behind one as a separate service.
- **+ REST API** — standard `HttpClient` calls (section 9); on the server, requests to relative URLs need a base URL configured (no implicit `window.location` origin).
- **+ GraphQL** — typically via Apollo Angular (`apollo-angular`), which exposes queries/mutations as Observables that integrate with the same RxJS/signals patterns used for REST.

---

## 16. Testing

**Definition:** Testing, in this context, is the automated verification of Angular code at the **unit** (an isolated class/component), **integration** (multiple collaborating pieces), and **end-to-end** (the full app in a real browser) levels.

### Unit testing — Jasmine vs Jest

Angular CLI historically defaults to **Karma + Jasmine** (`describe`/`it`/`expect`, runs in a real browser). **Jest** (`ng add @angular-builders/jest` or the newer built-in `@angular/build` Jest support) is a common swap — faster (jsdom, no browser launch), better watch mode, snapshot testing.

### `TestBed` — component testing

**Definition:** `TestBed` is Angular's core testing utility that dynamically constructs a testing module, used to configure and instantiate components, directives, and services together with their dependencies in isolation from the rest of the app.

```ts
describe('CounterComponent', () => {
  let fixture: ComponentFixture<CounterComponent>;
  let component: CounterComponent;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [CounterComponent], // standalone components are imported, not declared
      providers: [{ provide: CounterService, useValue: { increment: jasmine.createSpy() } }],
    }).compileComponents();

    fixture = TestBed.createComponent(CounterComponent);
    component = fixture.componentInstance;
    fixture.detectChanges(); // triggers ngOnInit + initial render
  });

  it('increments on click', () => {
    const button = fixture.nativeElement.querySelector('button');
    button.click();
    fixture.detectChanges();
    expect(fixture.nativeElement.querySelector('span').textContent).toContain('1');
  });
});
```

### Service testing

```ts
describe('UserApi', () => {
  let api: UserApi;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [UserApi, provideHttpClient(), provideHttpClientTesting()],
    });
    api = TestBed.inject(UserApi);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpMock.verify()); // fails the test if requests were left unhandled

  it('fetches a user', () => {
    api.getUser(1).subscribe(user => expect(user.id).toBe(1));
    const req = httpMock.expectOne('/api/users/1');
    expect(req.request.method).toBe('GET');
    req.flush({ id: 1, name: 'Ada' });
  });
});
```

### Directive & pipe testing

```ts
@Component({ standalone: true, imports: [HighlightDirective], template: `<div appHighlight>Text</div>` })
class HostComponent {}

it('applies highlight class', () => {
  const fixture = TestBed.createComponent(HostComponent);
  fixture.detectChanges();
  const div = fixture.nativeElement.querySelector('div');
  div.dispatchEvent(new Event('mouseenter'));
  fixture.detectChanges();
  expect(div.classList).toContain('highlighted');
});

it('formats currency', () => {
  expect(new FormatCurrencyPipe().transform(9.5)).toBe('$9.50'); // pipes are plain classes — no TestBed needed
});
```

### Router testing

```ts
TestBed.configureTestingModule({
  providers: [provideRouter(routes)],
});
const router = TestBed.inject(Router);
const harness = await RouterTestingHarness.create();
await harness.navigateByUrl('/users/1');
expect(harness.routeNativeElement?.textContent).toContain('User 1');
```

### Mocking & spies

**Definition:** A **mock** is a fake implementation substituted for a real dependency in a test; a **spy** wraps a function to record how it was called (arguments, call count) so a test can assert on those interactions without invoking the real logic.

```ts
const authSpy = jasmine.createSpyObj('AuthService', ['login', 'isLoggedIn']);
authSpy.isLoggedIn.and.returnValue(true);

TestBed.configureTestingModule({ providers: [{ provide: AuthService, useValue: authSpy }] });
```

### Angular Testing Library (alternative style)

**Definition:** Angular Testing Library is a testing utility that queries the rendered DOM the way a real user would — by role, label, or visible text — rather than through component internals, encouraging tests that verify behavior and survive refactors.

```ts
import { render, screen, fireEvent } from '@testing-library/angular';

it('increments on click', async () => {
  await render(CounterComponent);
  fireEvent.click(screen.getByRole('button', { name: /increment/i }));
  expect(screen.getByText('1')).toBeTruthy();
});
```

### E2E — Playwright

**Definition:** End-to-end (E2E) testing exercises the full, running application in a real browser, simulating actual user interactions (clicks, typing, navigation) across pages, rather than testing units in isolation.

```ts
import { test, expect } from '@playwright/test';

test('user can log in', async ({ page }) => {
  await page.goto('/login');
  await page.fill('[name=email]', 'a@b.com');
  await page.fill('[name=password]', 'secret');
  await page.click('button[type=submit]');
  await expect(page).toHaveURL('/dashboard');
});
```

Playwright (or Cypress) has largely replaced Protractor (deprecated/removed from the CLI) for Angular E2E.

---

## 17. Security

**Definition:** Web application security, in the Angular context, is the set of practices and framework-provided protections that prevent an application from being exploited via injected scripts, forged requests, or leaked credentials.

### XSS & Angular sanitization

**Definition:** Cross-Site Scripting (XSS) is an attack in which untrusted input is rendered as executable script or HTML in a victim's browser. Angular's built-in **sanitizer** strips dangerous content (`<script>` tags, inline event handlers) from values bound into the DOM, on by default:

```ts
// ❌ dangerous if userHtml comes from untrusted input
@Component({ template: `<div [innerHTML]="userHtml"></div>` })
export class CommentComponent {
  userHtml = this.sanitizer.bypassSecurityTrustHtml(rawUserComment); // only do this for TRUSTED content
}
```

```ts
// ✅ let Angular sanitize it — strips <script>, event handlers, etc. automatically
@Component({ template: `<div [innerHTML]="userHtml"></div>` })
export class CommentComponent {
  userHtml = rawUserComment; // Angular's built-in sanitizer runs automatically on [innerHTML]
}
```

`DomSanitizer.bypassSecurityTrust*` (`Html`, `Style`, `Script`, `Url`, `ResourceUrl`) exists for content **you** control and know is safe (e.g. a trusted CMS) — never call it on raw user input.

### CSRF

**Definition:** Cross-Site Request Forgery (CSRF) is an attack that tricks an authenticated user's browser into unknowingly submitting a request to another site on their behalf. Angular's `HttpClient` has built-in support for the **double-submit-cookie** defense: it reads a cookie (default name `XSRF-TOKEN`) and echoes it back as a header (`X-XSRF-TOKEN`) on same-origin requests.

```ts
provideHttpClient(withXsrfConfiguration({ cookieName: 'XSRF-TOKEN', headerName: 'X-XSRF-TOKEN' }))
```

### Authentication / Authorization / JWT

**Definition:** **Authentication** verifies *who* a user is; **authorization** determines *what* an authenticated user is allowed to do. A **JSON Web Token (JWT)** is a signed, self-contained token commonly used to carry authentication/authorization claims between client and server.

```ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  private _token = signal<string | null>(sessionStorage.getItem('token'));
  readonly isLoggedIn = computed(() => !!this._token());

  login(token: string) {
    this._token.set(token);
    sessionStorage.setItem('token', token); // see "secure storage" note below
  }
  logout() { this._token.set(null); sessionStorage.removeItem('token'); }
  token() { return this._token(); }
}
```

- Attach the JWT via an interceptor (section 9), not by manually adding headers per-call.
- **Route protection** — `canActivate: [authGuard]` (section 8) plus a matching server-side check; a client-side guard is a UX nicety, never the actual security boundary.
- **Authorization** (roles/permissions) — check on the server for every request; client-side role checks only hide UI, they don't prevent a determined user from calling the API directly.

### Secure storage

`localStorage`/`sessionStorage` are readable by any JS on the page — a single XSS hole anywhere leaks every token stored there. An **HttpOnly, Secure, SameSite** cookie set by the server is the stronger option since client-side JS (and therefore an XSS payload) can't read it at all. If you must use storage, prefer `sessionStorage` (cleared on tab close) over `localStorage`, and keep token lifetime short.

### HTTP security headers (server-side, but Angular-relevant)

**Definition:** HTTP security headers are server-set response headers that instruct the browser to enforce additional restrictions on how the page's content can behave.

- `Content-Security-Policy` — restricts script sources; Angular's Ivy output is CSP-compatible without `unsafe-eval` in AOT builds.
- `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `X-Frame-Options` — standard hardening, configured at the server/CDN, not in Angular itself.

### Avoiding unsafe DOM manipulation

Avoid `ElementRef.nativeElement` writes (`this.el.nativeElement.innerHTML = ...`) — they bypass Angular's sanitizer entirely. Prefer `Renderer2` or template bindings, which stay within Angular's security model:

```ts
constructor(private renderer: Renderer2, private el: ElementRef) {}
setText(text: string) {
  this.renderer.setProperty(this.el.nativeElement, 'textContent', text); // safe — textContent, not innerHTML
}
```

---

## 18. Production Engineering

**Definition:** Production engineering covers the build, deployment, and operational practices that get an Angular application safely and observably into users' hands, and keep it that way after release.

### Environment configuration

**Definition:** Environment configuration is the practice of maintaining separate sets of settings (API URLs, feature flags, keys) for different deployment targets — development, staging, production — swapped in automatically at build time.

```ts
// environments/environment.ts (dev)
export const environment = { production: false, apiUrl: 'http://localhost:3000/api' };

// environments/environment.prod.ts
export const environment = { production: true, apiUrl: 'https://api.example.com' };
```

```json
// angular.json — fileReplacements swap the file per build configuration
"configurations": {
  "production": {
    "fileReplacements": [{ "replace": "src/environments/environment.ts", "with": "src/environments/environment.prod.ts" }],
    "budgets": [{ "type": "initial", "maximumWarning": "500kb", "maximumError": "1mb" }]
  }
}
```

### Build configurations

```bash
ng build --configuration production   # AOT, minified, tree-shaken, hashed filenames
ng build --configuration staging
```

Budgets (above) fail the build if bundle size regresses past a threshold — cheap guardrail against silent bloat.

### CI/CD (sketch)

**Definition:** Continuous Integration/Continuous Deployment (CI/CD) is an automated pipeline that builds, tests, and deploys an application on every code change, replacing manual release steps.

```yaml
# .github/workflows/ci.yml
jobs:
  build-and-test:
    steps:
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --watch=false --browsers=ChromeHeadless
      - run: npm run build -- --configuration production
      - run: npx playwright test
```

### Docker

**Definition:** Docker is a containerization platform that packages an application together with its runtime environment into a single, portable, isolated image that runs identically anywhere.

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build -- --configuration production

FROM nginx:alpine
COPY --from=build /app/dist/my-app/browser /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

### Nginx (SPA fallback + caching)

```nginx
server {
  root /usr/share/nginx/html;
  location / {
    try_files $uri $uri/ /index.html;   # SPA deep-link support
  }
  location ~* \.(js|css|png|jpg|woff2)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";  # hashed filenames make this safe
  }
}
```

### CDN & caching

**Definition:** A Content Delivery Network (CDN) is a geographically distributed set of servers that caches and serves static assets from a location close to the end user, reducing latency compared to a single origin server.

Ship hashed filenames (CLI does this by default — `main.a1b2c3.js`) so you can set far-future cache headers on assets and only `index.html` needs `no-cache`, guaranteeing users always get the latest app shell pointing at the latest hashed bundles.

### Monitoring, error tracking, logging

**Definition:** Monitoring and error tracking are the practices of capturing runtime errors, performance metrics, and logs from a live production application so issues can be detected and diagnosed after deployment, when `console.error` alone is invisible to the team.

```ts
// global error handler
@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  private tracker = inject(ErrorTrackingService);
  handleError(error: unknown) {
    console.error(error);
    this.tracker.report(error); // e.g. Sentry.captureException(error)
  }
}
// app.config.ts
providers: [{ provide: ErrorHandler, useClass: GlobalErrorHandler }]
```

Pair with a real-user-monitoring/error tool (Sentry, Datadog RUM, etc.) to capture stack traces, breadcrumbs, and release version in production.

### Source maps & bundle analysis

```bash
ng build --configuration production --source-map   # upload to error tracker to de-minify stack traces
npx webpack-bundle-analyzer dist/my-app/stats.json  # or `ng build --stats-json` then analyze
```

### Production debugging

- Ship source maps to your error tracker (not to end users — keep them out of the public `dist/` if the app is sensitive) so stack traces resolve to real file/line.
- Angular DevTools works against a production build too (component tree inspection, profiler) as long as Ivy debug info isn't stripped.
- Feature flags / gradual rollouts reduce blast radius of a bad deploy — pair with the monitoring above to catch regressions fast.
