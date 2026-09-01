# React — Deep Dive Roadmap

We'll go from fundamentals → internals → architecture → performance → testing → production → interview problems.

---

## 1. React Fundamentals

**Definition:** React is a JavaScript library (not a full framework) for building user interfaces out of composable, reusable **components**, using a **declarative** programming model — you describe what the UI should look like for a given state, and React figures out how to update the real DOM to match.

**JSX. Definition:** JSX is a syntax extension to JavaScript that lets you write HTML-like markup inside JS/TS files; it is compiled (by Babel/TypeScript) into nested `React.createElement()` calls, not interpreted directly by the browser.

```jsx
// JSX
const el = <h1 className="title">Hello, {name}</h1>;

// compiles to (roughly)
const el = React.createElement('h1', { className: 'title' }, 'Hello, ', name);
```

**Elements vs Components. Definition:** A **React element** is a plain, immutable JavaScript object describing a piece of UI (`{ type, props }`) — cheap to create, not yet rendered. A **component** is a function (or, historically, a class) that *returns* elements; React calls it to produce the elements to render.

```jsx
// element
const element = <h1>Hi</h1>; // { type: 'h1', props: { children: 'Hi' } }

// component
function Greeting({ name }) {
  return <h1>Hi, {name}</h1>; // returns an element
}
```

**Props. Definition:** Props ("properties") are the read-only input arguments passed from a parent component into a child, analogous to function parameters — a component must never modify its own props.

**State. Definition:** State is data owned and managed by a component that can change over time and, when it changes, causes the component (and its descendants) to re-render — introduced via the `useState` hook in function components.

```jsx
function Counter() {
  const [count, setCount] = useState(0); // state: local, owned by this component
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

**Rendering & re-rendering. Definition:** Rendering is React calling a component function to produce its element tree; a re-render happens whenever that component's state, props, or context changes, and React re-runs the function to compute the new tree.

**The Virtual DOM (high level). Definition:** The Virtual DOM is an in-memory tree of React elements that React diffs against the previous tree to compute the minimal set of real DOM mutations needed, rather than re-creating the whole page on every update. (Full mechanics in section 5 & 7.)

**Composition vs inheritance. Definition:** Composition is building complex UI by nesting and combining simpler components (via props, especially `children`) rather than through class inheritance hierarchies — React's official, idiomatic way to share and reuse UI logic.

```jsx
function Card({ children }) {
  return <div className="card">{children}</div>;
}
// usage: <Card><UserProfile /></Card>  — composition, not `class UserCard extends Card`
```

**Project setup:**

```bash
npm create vite@latest my-app -- --template react-ts   # modern, fast dev server (recommended)
npx create-react-app my-app                             # legacy, webpack-based, largely superseded
```

**Project structure (Vite default):**

```
src/
 ├── main.tsx        # createRoot(...).render(<App />)
 ├── App.tsx
 ├── components/
 └── assets/
```

---

## 2. Components — Deep Dive

**Definition:** In modern React, a component is (almost always) a **function component** — a plain JavaScript/TypeScript function that accepts a `props` object and returns JSX describing the UI for the current render.

```jsx
function UserCard({ name, avatarUrl }: { name: string; avatarUrl: string }) {
  return (
    <div className="card">
      <img src={avatarUrl} alt={name} />
      <p>{name}</p>
    </div>
  );
}
```

### Props

**Default props. Definition:** default values used for a prop when the caller doesn't supply one — in function components, expressed via a default parameter, not a special API.

```jsx
function Button({ label, variant = 'primary' }) {
  return <button className={variant}>{label}</button>;
}
```

**`props.children`. Definition:** the special prop holding whatever JSX was nested between a component's opening and closing tags — the mechanism behind composition/slots.

```jsx
function Panel({ children }) { return <div className="panel">{children}</div>; }
// <Panel><p>Content</p></Panel> → children is <p>Content</p>
```

**Prop drilling. Definition:** passing a prop through several intermediate components that don't use it themselves, purely so a deeply nested descendant can receive it — a code smell that Context or composition usually resolves.

### Conditional rendering

```jsx
{isLoggedIn ? <Dashboard /> : <LoginPrompt />}
{items.length > 0 && <ItemList items={items} />}
{status === 'loading' ? <Spinner /> : status === 'error' ? <ErrorMsg /> : <Content />}
```

### Lists & keys

**Key. Definition:** a `key` is a string prop given to each element in a list that gives it a stable identity across renders, so React's reconciler can tell which items were added, removed, or reordered instead of re-creating the whole list.

```jsx
{users.map(user => <UserRow key={user.id} user={user} />)}
// ❌ avoid `key={index}` for lists that can reorder/insert/delete — breaks identity tracking
```

### Event handling

```jsx
function Form() {
  function handleSubmit(e: React.FormEvent) {
    e.preventDefault(); // React wraps native events in a SyntheticEvent (see section 7)
    console.log('submitted');
  }
  return <form onSubmit={handleSubmit}><button type="submit">Go</button></form>;
}
```

### Controlled vs uncontrolled components

**Definition:** A **controlled** component's value is driven entirely by React state (`value` + `onChange`) — React is the single source of truth. An **uncontrolled** component keeps its value in the DOM itself, read on demand via a `ref`.

```jsx
// controlled
function ControlledInput() {
  const [value, setValue] = useState('');
  return <input value={value} onChange={e => setValue(e.target.value)} />;
}

// uncontrolled
function UncontrolledInput() {
  const ref = useRef<HTMLInputElement>(null);
  return <input ref={ref} defaultValue="" />; // read ref.current.value when needed
}
```

### Fragments

**Definition:** a `<Fragment>` (shorthand `<>...</>`) groups multiple children without adding an extra wrapper DOM node — used because a component can only return a single root element.

```jsx
return (
  <>
    <Header />
    <Main />
  </>
);
```

### Portals

**Definition:** `createPortal()` renders a child element into a different DOM node than the component's normal parent tree — while still participating in React's tree for event bubbling and context — used for modals, tooltips, and dropdowns that must escape an `overflow: hidden` ancestor.

```jsx
import { createPortal } from 'react-dom';

function Modal({ children }) {
  return createPortal(<div className="modal">{children}</div>, document.getElementById('modal-root')!);
}
```

### Error boundaries

**Definition:** an error boundary is a component (must currently be a class component — no hook equivalent exists) that catches JavaScript errors thrown anywhere in its child tree during rendering, and renders a fallback UI instead of crashing the whole app.

```jsx
class ErrorBoundary extends React.Component<{ children: React.ReactNode }, { hasError: boolean }> {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  componentDidCatch(error: Error, info: React.ErrorInfo) { logError(error, info); }
  render() {
    return this.state.hasError ? <p>Something went wrong.</p> : this.props.children;
  }
}
```

---

## 3. Hooks — Deep Dive

The core of modern React.

**Definition:** A hook is a special function (always starting with `use`) that lets a function component "hook into" React features — state, lifecycle-equivalent effects, context, refs — that were previously only available in class components.

**`useState` — Definition:** returns a stateful value and a setter function that, when called, schedules a re-render with the new value.

```jsx
const [count, setCount] = useState(0);
setCount(count + 1);          // direct value
setCount(prev => prev + 1);   // updater function — safe with stale closures / batching
```

**`useEffect` — Definition:** registers a side-effect callback (data fetching, subscriptions, manual DOM work) to run after React commits the render to the DOM, re-running whenever any value in its dependency array has changed.

```jsx
useEffect(() => {
  const id = setInterval(() => setTick(t => t + 1), 1000);
  return () => clearInterval(id); // cleanup — runs before the next effect and on unmount
}, []); // empty array = run once, on mount
```

- **Dependency arrays** — the list of reactive values the effect reads; omitting a used value is a common bug (caught by the `exhaustive-deps` ESLint rule).
- **Cleanup functions** — the function an effect optionally returns, run before the effect re-runs and when the component unmounts; essential for subscriptions/timers/listeners.
- **Effect timing** — `useEffect` runs *asynchronously after paint* (passive effect); `useLayoutEffect` runs *synchronously before paint* (see next).

**`useLayoutEffect` — Definition:** identical to `useEffect` but fires synchronously after DOM mutations, *before* the browser paints — used when you need to measure or mutate the DOM without a visible flicker (e.g. positioning a tooltip based on measured size).

**`useRef` — Definition:** returns a mutable `{ current: value }` object that persists for the component's entire lifetime without causing a re-render when it changes — used for DOM node references and for "instance variables" in function components.

```jsx
const inputRef = useRef<HTMLInputElement>(null);
useEffect(() => { inputRef.current?.focus(); }, []);
<input ref={inputRef} />
```

**`useMemo` — Definition:** memoizes the *result* of an expensive computation, recomputing it only when one of its dependencies changes.

```jsx
const sorted = useMemo(() => [...items].sort(compareFn), [items]);
```

**`useCallback` — Definition:** memoizes a *function reference* itself (it's `useMemo(() => fn, deps)` under the hood), so a child receiving it as a prop doesn't see a "new" function on every render — important for `React.memo` children.

```jsx
const handleClick = useCallback(() => doSomething(id), [id]);
```

**`useReducer` — Definition:** an alternative to `useState` for state whose transitions are better expressed as a `(state, action) => newState` reducer function — useful for complex state with multiple sub-values or well-defined transitions.

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'increment': return { count: state.count + 1 };
    default: return state;
  }
}
const [state, dispatch] = useReducer(reducer, { count: 0 });
dispatch({ type: 'increment' });
```

**`useContext` — Definition:** reads the current value of a React Context from the nearest matching `<Context.Provider>` above the calling component, re-rendering the component whenever that value changes. (Full treatment in section 8.)

**`useImperativeHandle` — Definition:** customizes the instance value exposed to a parent when that parent attaches a `ref` to this component (used together with `forwardRef`), letting a component expose an imperative API (e.g. `focus()`) instead of its raw DOM node.

```jsx
const FancyInput = forwardRef((props, ref) => {
  const inputRef = useRef<HTMLInputElement>(null);
  useImperativeHandle(ref, () => ({ focus: () => inputRef.current?.focus() }));
  return <input ref={inputRef} />;
});
```

**`useId` — Definition:** generates a unique, stable ID string per component instance, stable across server and client renders — used for `htmlFor`/`id` pairs in accessible forms, safe under SSR (unlike `Math.random()` or a module-level counter).

**`useTransition` — Definition:** marks a state update as a low-priority **transition**, letting React keep the UI responsive to more urgent updates (like typing) while the transition's re-render happens in the background; returns an `isPending` flag.

```jsx
const [isPending, startTransition] = useTransition();
function handleChange(e) {
  setInput(e.target.value);           // urgent
  startTransition(() => setResults(filterBigList(e.target.value))); // low priority
}
```

**`useDeferredValue` — Definition:** returns a "lagging" copy of a value that updates at lower priority than the source, letting an expensive render based on it fall behind urgent updates without blocking them — a value-based alternative to `useTransition`.

**`useSyncExternalStore` — Definition:** the official hook for subscribing a component to an external (non-React) mutable data source — e.g. a browser API or a third-party store — in a way that's safe under concurrent rendering; the low-level primitive most state-management libraries build on.

### Rules of Hooks

1. Only call hooks at the **top level** — never inside loops, conditions, or nested functions (React relies on call *order* to associate hook state, see section 7).
2. Only call hooks from **React function components** or from **custom hooks**.

### Custom hooks

**Definition:** a custom hook is a plain function, named `useSomething`, that calls other hooks internally — the standard mechanism for extracting and reusing stateful logic between components.

```jsx
function useLocalStorage<T>(key: string, initial: T) {
  const [value, setValue] = useState<T>(() => {
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) : initial;
  });
  useEffect(() => { localStorage.setItem(key, JSON.stringify(value)); }, [key, value]);
  return [value, setValue] as const;
}
```

---

## 4. State Management

**Definition:** State management is the set of patterns and libraries used to store, share, and update application data across components in a predictable way.

Understand the progression:

```
Component State (useState)
      ↓
Lifted State (shared via props)
      ↓
Context API
      ↓
useReducer + Context
      ↓
External libraries (Redux, Zustand, Jotai, Recoil)
```

**Lifting state up. Definition:** moving state from a child component to their closest common ancestor, so multiple children can share and stay in sync with it via props.

**Context API — Definition:** a built-in mechanism for passing a value down a component tree without threading it through props at every level (avoiding prop drilling), consumed via `useContext`. See section 8 for the deep dive.

```jsx
const ThemeContext = createContext<'light' | 'dark'>('light');

function App() {
  return <ThemeContext.Provider value="dark"><Toolbar /></ThemeContext.Provider>;
}
function Toolbar() {
  const theme = useContext(ThemeContext);
  return <div className={theme}>...</div>;
}
```

**`useReducer` + Context** — a common lightweight pattern for feature-scoped global-ish state without adopting a full library: a reducer holds the logic, a Context distributes `state`/`dispatch` to the subtree.

### Redux

**Definition:** Redux is a predictable state container library implementing a single, immutable global store updated only by pure reducer functions in response to dispatched **actions** — a strict unidirectional data flow.

```ts
// Redux Toolkit (the modern, recommended way to use Redux)
import { createSlice, configureStore } from '@reduxjs/toolkit';

const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    incremented: state => { state.value += 1; }, // Immer lets you "mutate" safely under the hood
  },
});

export const store = configureStore({ reducer: { counter: counterSlice.reducer } });
```

- **Middleware** — functions that sit between `dispatch` and the reducer, used for side effects: `redux-thunk` (dispatch functions, common default) and `redux-saga` (generator-based effect orchestration, more powerful/complex).

### Zustand

**Definition:** Zustand is a minimal state-management library that stores state in a plain hook-based store, without requiring providers, actions, or reducers — much less boilerplate than Redux.

```ts
import { create } from 'zustand';

const useCartStore = create<{ items: Item[]; add: (i: Item) => void }>((set) => ({
  items: [],
  add: (item) => set(state => ({ items: [...state.items, item] })),
}));

function Cart() {
  const items = useCartStore(state => state.items); // subscribes only to `items`
}
```

### Jotai / Recoil — atomic state

**Definition:** atomic state libraries model global state as many small, independent units ("atoms") rather than one big store, so a component subscribing to one atom re-renders only when *that* atom changes.

```ts
import { atom, useAtom } from 'jotai';

const countAtom = atom(0);
function Counter() {
  const [count, setCount] = useAtom(countAtom);
}
```

### React Query / TanStack Query for server state

**Definition:** "server state" is data that lives on a remote server and is merely cached on the client (fetched, paginated, goes stale) — fundamentally different from client UI state, which is why dedicated libraries like TanStack Query exist rather than stuffing server data into Redux/Context. See section 10 for details.

### Decision guide

| Situation | Choice |
|---|---|
| State used by one component | `useState`/`useReducer` |
| Shared by a few nearby components | Lift state up |
| Shared across a subtree, low update frequency (theme, auth) | Context |
| Complex client state, many features, need devtools/middleware | Redux (Toolkit) |
| Want global state with minimal boilerplate | Zustand |
| Many independent, fine-grained pieces of state | Jotai/Recoil |
| Data fetched from a server (lists, details, pagination) | React Query / SWR — **not** Redux |

---

## 5. Rendering & Reconciliation

**Definition:** Rendering is React computing a component's element tree; reconciliation is the algorithm React uses to compare ("diff") the new element tree against the previous one and determine the minimal set of real DOM mutations required.

**The Virtual DOM — Definition:** a lightweight, in-memory tree of plain JS objects (React elements) that mirrors the intended UI; React never mutates the real DOM directly from your component code — it always goes through this intermediate representation first.

**Reconciliation algorithm — Definition:** React's heuristic (not truly optimal, but O(n) instead of the O(n³) of a generic tree-diff) diffing algorithm, built on two assumptions: (1) elements of different types produce different trees (React tears down and rebuilds rather than diffing children), and (2) elements with a stable `key` prop are matched across renders by that key.

**Diffing rules, briefly:**
- Different element **type** at the same position → old subtree is torn down, new one built from scratch (state is lost).
- Same type → React keeps the DOM node, updates only changed attributes, and recurses into children.
- Lists → matched by `key`, not position, so reordering doesn't require destroying/recreating every item.

**Keys and reconciliation** — see section 2; a wrong/missing key is one of the most common sources of subtle bugs (state "sticking" to the wrong row after a reorder).

**Batching — Definition:** grouping multiple state updates that occur within the same event/task into a single re-render, instead of re-rendering once per `setState` call. React 18 introduced **automatic batching** — even updates inside promises, timeouts, and native event handlers are now batched (previously only React event handlers were).

```jsx
function handleClick() {
  setCount(c => c + 1);
  setFlag(f => !f);
  // React 18: one re-render for both, even though this is two setState calls
}
```

**Synchronous vs concurrent rendering — Definition:** in the legacy (synchronous) model, once React starts rendering an update it cannot be interrupted. **Concurrent rendering** (opt-in via `createRoot` in React 18+) lets React start rendering an update, pause it, abandon it, or prioritize a more urgent update ahead of it — enabling features like `useTransition`.

**`StrictMode` — Definition:** a development-only wrapper component that helps surface potential bugs by intentionally double-invoking certain functions (component bodies, some effects) to verify they're pure/idempotent — it renders nothing itself and has zero effect in production builds.

**Re-render triggers** — a component re-renders when: its own state changes, its parent re-renders (by default — see `memo`), a consumed context value changes, or (rarely) `forceUpdate`-equivalents are used.

**Avoiding unnecessary re-renders** — see section 6 (`memo`/`useMemo`/`useCallback`), and structurally: keep state as local as possible, split large components so unrelated state doesn't force unrelated UI to re-render together.

**Derived state anti-pattern** — copying a prop into local state with `useState(prop)` and trying to keep them in sync (usually with an effect) is almost always wrong; compute the derived value directly during render instead (`const derived = compute(prop)`), which is simpler and can't drift out of sync.

---

## 6. Performance Optimization

**Definition:** Performance optimization in React is the practice of minimizing unnecessary re-renders, expensive re-computation, and JavaScript bundle size so the app stays responsive as it scales.

Very important for senior interviews.

**`React.memo` — Definition:** a higher-order component that wraps a component so React skips re-rendering it when its props are shallow-equal to the previous render's props (it still re-renders if the *parent's* re-render changes props, or if the component's own state/context changes).

```jsx
const UserRow = React.memo(function UserRow({ user }: { user: User }) {
  return <li>{user.name}</li>;
});
```

`memo` only helps if the props passed in are actually stable — an inline object/array/function literal is a *new reference* every render, defeating the shallow comparison; pair with `useMemo`/`useCallback` in the parent.

**Code splitting — Definition:** breaking the JS bundle into smaller chunks loaded on demand rather than all at once, typically via `React.lazy` + dynamic `import()`.

```jsx
const AdminPanel = React.lazy(() => import('./AdminPanel'));

<Suspense fallback={<Spinner />}>
  <AdminPanel />
</Suspense>
```

**`Suspense` — Definition:** a component that lets children "suspend" rendering while waiting for something asynchronous (a lazy-loaded component, or — with supporting libraries — a data fetch), showing a `fallback` UI in the meantime instead of an incomplete tree.

**Virtualization — Definition:** rendering only the DOM nodes for the currently visible slice of a long list (via `react-window`/`react-virtualized`), recycling nodes as the user scrolls, instead of mounting every item up front.

```jsx
import { FixedSizeList } from 'react-window';

<FixedSizeList height={400} itemCount={items.length} itemSize={35} width="100%">
  {({ index, style }) => <div style={style}>{items[index].name}</div>}
</FixedSizeList>
```

**Avoiding inline function/object props** — `<Child onClick={() => doThing(id)} style={{ color: 'red' }} />` creates a new function/object every render, breaking `memo` on `Child`; hoist with `useCallback`/`useMemo` or move the literal outside the render if it never needs props.

**Debouncing/throttling expensive handlers** — for handlers that fire rapidly (scroll, resize, keystroke-driven search), delay or rate-limit the expensive work rather than running it on every event.

**Profiling** — the React DevTools **Profiler** tab records a render pass and shows per-component render time and *why* each component re-rendered, replacing guesswork with data.

**`useTransition` / `useDeferredValue`** — see section 3; the modern, React-native way to keep urgent interactions (typing, clicking) smooth while a large, related re-render happens at lower priority.

---

## 7. React Internals

**Definition:** React internals refers to the framework's own implementation — how JSX becomes elements, how elements become a Fiber tree, and how that tree is rendered and committed to the real DOM.

This is where we go beyond normal tutorials.

Understand:

```
JSX
  ↓
React.createElement (Babel/TS transform)
  ↓
React Element tree (plain JS objects)
  ↓
Fiber tree construction
  ↓
Render phase (reconciliation, work-in-progress tree)
  ↓
Commit phase (DOM mutations, effects)
  ↓
Browser paint
```

**React elements vs Fiber nodes — Definition:** a React **element** is a lightweight, throwaway description of "what to render" recreated on every render; a **Fiber** is a persistent, mutable unit-of-work object — one per component instance/DOM node — that tracks that component's type, props, state, and its place in the tree across renders, forming the actual data structure React operates on internally.

**The Fiber architecture — Definition:** Fiber is React's internal reconciliation engine (since React 16), representing the component tree as a linked-list-like structure of Fiber nodes (each with `child`/`sibling`/`return` pointers) that can be traversed **incrementally** — one unit of work at a time — rather than recursively all at once, which is what makes interruptible/concurrent rendering possible.

**Fiber tree vs current tree (double buffering) — Definition:** React keeps two Fiber trees: the **current** tree (what's on screen) and a **work-in-progress** tree (being built for the pending update). Once the work-in-progress tree is complete, React swaps a pointer so it becomes the current tree — analogous to double-buffering in graphics, avoiding a half-updated UI ever being visible.

**Render phase vs commit phase — Definition:**
- **Render phase** — React builds the work-in-progress tree, running component functions and diffing against the current tree. Pure, has no visible side effects, and (in concurrent mode) can be paused, aborted, or restarted.
- **Commit phase** — React applies the computed DOM mutations synchronously, then runs layout effects (`useLayoutEffect`) synchronously, then paints, then runs passive effects (`useEffect`) asynchronously. Cannot be interrupted.

**The Scheduler (priority lanes) — Definition:** an internal package that assigns each unit of work a priority ("lane") — e.g. discrete user input > continuous input > transitions > idle — and decides in what order pending work should be processed, yielding to the browser between chunks of low-priority work so the main thread isn't blocked.

**Concurrent rendering / time slicing — Definition:** the ability to break rendering work into small chunks and interleave it with other browser tasks (like responding to a keystroke) — "time slicing" — so a large re-render doesn't freeze the UI; opt-in via `createRoot` (React 18+) and exercised through APIs like `useTransition`.

**Work loop — Definition:** the internal loop (`workLoop`/`performUnitOfWork`) that walks the Fiber tree depth-first, processing one Fiber at a time, checking after each one whether it should yield control back to the browser before continuing (in concurrent mode).

**Hooks internals — Definition:** each function component's hooks are stored as a **linked list** attached to its Fiber node, in the exact call order they were invoked — which is precisely why the Rules of Hooks (section 3) forbid conditional hook calls: React matches hook state to hook calls purely by call order, not by name.

**Synthetic events — Definition:** React wraps native DOM events in a cross-browser-normalized `SyntheticEvent` object and, since React 17, attaches a single delegated listener at the root DOM container (rather than one per element) — event delegation for performance and cleaner mounting/unmounting.

---

## 8. Context API — Deep Dive

**Definition:** React Context is a built-in mechanism that lets a value be made available to an entire subtree of components without threading it through props at every intermediate level.

```jsx
const AuthContext = createContext<AuthState | null>(null);

function AuthProvider({ children }) {
  const [user, setUser] = useState<User | null>(null);
  return <AuthContext.Provider value={{ user, setUser }}>{children}</AuthContext.Provider>;
}

function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within AuthProvider');
  return ctx;
}
```

**Multiple contexts** — nest providers (or combine into one component) when a subtree needs several independent values (theme, auth, locale) — keep contexts narrow and single-purpose rather than one giant "app context."

**Context + `useReducer`** — a common pattern for feature-scoped state management without an external library: the reducer defines transitions, Context exposes `{ state, dispatch }` to the subtree.

**Performance pitfall — Definition:** *every* component calling `useContext(SomeContext)` re-renders whenever that context's `value` changes, regardless of which part of the value it actually uses — an object literal passed as `value={{ a, b }}` is a new reference every render, so this happens even when nothing meaningful changed.

```jsx
// ❌ new object every render → every consumer re-renders every time
<Context.Provider value={{ user, theme }}>

// ✅ memoize the value
const value = useMemo(() => ({ user, theme }), [user, theme]);
<Context.Provider value={value}>
```

**Splitting contexts** — separate frequently-changing values (e.g. `theme`) from rarely-changing ones (e.g. `currentUser`) into different contexts, so a component only re-renders for the value it actually consumes.

**Selector-based context patterns** — libraries like `use-context-selector` (or moving frequently-changing state to a store library instead) let a consumer subscribe to a *slice* of context value, avoiding the "every consumer re-renders" limitation that plain Context has no built-in answer for.

---

## 9. Routing

**Definition:** React Router is the de facto standard routing library for React, mapping URL paths to components and managing navigation without full page reloads (React itself has no built-in router).

```jsx
import { BrowserRouter, Routes, Route, useParams, useNavigate, Link } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/users/:id" element={<UserDetail />} />
        <Route path="/admin/*" element={<AdminLayout />}>
          <Route path="settings" element={<Settings />} /> {/* nested route */}
        </Route>
        <Route path="*" element={<NotFound />} />
      </Routes>
    </BrowserRouter>
  );
}

function UserDetail() {
  const { id } = useParams();       // dynamic segment
  const navigate = useNavigate();   // imperative navigation
  return <button onClick={() => navigate('/')}>Home</button>;
}
```

**Loaders & actions (data router) — Definition:** in React Router's newer "data" APIs (`createBrowserRouter`), a **loader** fetches data *before* a route renders (avoiding a loading-spinner-inside-component flash) and an **action** handles form submissions/mutations tied to a route, both integrated directly into the router config.

```jsx
const router = createBrowserRouter([
  { path: '/users/:id', element: <UserDetail />, loader: ({ params }) => fetchUser(params.id) },
]);
function UserDetail() {
  const user = useLoaderData(); // data resolved by the loader above
}
```

**Lazy-loaded / protected routes:**

```jsx
const Admin = React.lazy(() => import('./Admin'));
<Route path="/admin" element={<Suspense fallback={<Spinner />}><Admin /></Suspense>} />

function ProtectedRoute({ children }) {
  const { user } = useAuth();
  return user ? children : <Navigate to="/login" />;
}
```

---

## 10. Data Fetching

**Definition:** Data fetching in React covers retrieving data from a server and reflecting its loading/success/error state in the UI — React has no built-in data-fetching API; it's typically done with `fetch`/`axios` inside an effect, or via a dedicated server-state library.

**Fetching in `useEffect` (and its pitfalls):**

```jsx
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [status, setStatus] = useState<'idle' | 'loading' | 'error'>('idle');

  useEffect(() => {
    const controller = new AbortController();
    setStatus('loading');
    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then(r => r.json())
      .then(data => { setUser(data); setStatus('idle'); })
      .catch(err => { if (err.name !== 'AbortError') setStatus('error'); });
    return () => controller.abort(); // cancel if userId changes / component unmounts
  }, [userId]);
}
```

**Race conditions — Definition:** a bug where a slower, earlier request resolves *after* a faster, later one (e.g. rapid `userId` changes), overwriting the UI with stale data — prevented above via `AbortController` (or an "is this still the latest request" flag).

### React Query / TanStack Query

**Definition:** TanStack Query is a library purpose-built for **server state** — data fetched from a remote source — handling caching, background refetching, deduplication, and invalidation, which is qualitatively different from client UI state.

```jsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

function UserProfile({ userId }) {
  const { data: user, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then(r => r.json()),
  });
}

function EditUser() {
  const queryClient = useQueryClient();
  const mutation = useMutation({
    mutationFn: (patch: Partial<User>) => fetch('/api/users/1', { method: 'PATCH', body: JSON.stringify(patch) }),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['user', '1'] }), // refetch after mutation
  });
}
```

- **Caching & invalidation** — results are cached by `queryKey`; `invalidateQueries` marks matching cached data stale, triggering a refetch.
- **Background refetching** — stale data is shown immediately while a fresh copy is fetched silently (stale-while-revalidate).
- **Optimistic updates** — update the cache immediately (before the server confirms), roll back on error, for a snappier UX.

**SWR** — a lighter alternative from Vercel with a similar stale-while-revalidate philosophy and a smaller API surface.

**Suspense for data fetching** — newer patterns (React Query's `useSuspenseQuery`, Next.js Server Components) let a component "suspend" while data loads, handled by a `<Suspense>` boundary instead of manual `isLoading` checks.

---

## 11. Forms

**Definition:** Form handling in React covers capturing, validating, and submitting user input — via controlled/uncontrolled inputs directly, or through a dedicated form library that manages this boilerplate.

**Controlled inputs** — see section 2; every keystroke updates state and re-renders.

**Uncontrolled inputs** — value lives in the DOM, read via `ref` on submit — less re-render overhead for large forms, less "React-y."

### React Hook Form

**Definition:** a form library that keeps form values largely **uncontrolled** internally (subscribing to changes via refs rather than re-rendering on every keystroke) for significantly better performance on large forms, while still exposing a simple hook-based API.

```jsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({ email: z.string().email(), password: z.string().min(8) });

function SignupForm() {
  const { register, handleSubmit, formState: { errors } } = useForm({ resolver: zodResolver(schema) });
  const onSubmit = (data: z.infer<typeof schema>) => console.log(data);

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email')} />
      {errors.email && <span>{errors.email.message}</span>}
    </form>
  );
}
```

**Formik** — an older, still-common alternative with a more explicit, fully-controlled model (`<Formik>` render props / `useFormik`).

**Schema validation — Definition:** defining a form's shape and validation rules once, as a schema object (Zod or Yup), instead of scattering imperative `if` checks — the schema can also drive TypeScript types (as `z.infer<typeof schema>` does above).

**Dynamic form fields / multi-step forms** — typically modeled with `useFieldArray` (React Hook Form) for repeatable groups, and a `step` piece of state gating which fields render, with validation run per-step or on final submit.

---

## 12. Server Rendering & Full-Stack React

**Definition:** Server-Side Rendering (SSR) is rendering React components to an HTML string on the server before sending the page to the browser, rather than shipping an empty HTML shell that renders entirely client-side.

**Hydration — Definition:** the process by which client-side React "attaches" event handlers and internal state to server-rendered HTML that's already in the DOM, making it interactive, instead of discarding and re-rendering it from scratch.

### Next.js

**Definition:** Next.js is a full-stack React framework (built on React, by Vercel) providing file-based routing, SSR/SSG out of the box, API routes, and — in its App Router — React Server Components.

- **Pages Router vs App Router** — the Pages Router (`pages/`) is the original, fully client-rendered-by-default model; the App Router (`app/`, current default) is built around Server Components and nested layouts.
- **Server Components vs Client Components — Definition:** a **Server Component** renders only on the server, never ships its JS to the browser, and can access backend resources (databases, filesystem) directly; a **Client Component** (marked `'use client'`) renders/hydrates in the browser as usual and can use state, effects, and browser APIs. Server Components are the default in the App Router.

```tsx
// app/users/[id]/page.tsx — a Server Component by default
export default async function UserPage({ params }: { params: { id: string } }) {
  const user = await db.user.findUnique({ where: { id: params.id } }); // direct DB access — server-only
  return <UserProfile user={user} />;
}
```

```tsx
'use client'; // opts this component INTO the client bundle
export function LikeButton() {
  const [liked, setLiked] = useState(false);
  return <button onClick={() => setLiked(!liked)}>{liked ? '♥' : '♡'}</button>;
}
```

- **Streaming SSR** — the server sends HTML in chunks as it becomes ready (paired with `<Suspense>`), so the browser can start rendering the page before every piece of data has resolved.
- **SSG (Static Site Generation)** — pages are rendered to HTML entirely at *build* time, served as static files — fastest possible delivery, for content that doesn't change per-request.
- **ISR (Incremental Static Regeneration)** — statically generated pages are re-generated in the background after a configured interval, combining SSG's speed with data that can go stale.

**Remix** — another full-stack React framework, leaning further into web-standard APIs (`FormData`, HTTP caching) and nested-route data loading; conceptually similar goals to Next.js via a different API surface.

**SEO** — SSR/SSG give crawlers fully-rendered HTML immediately, unlike a client-only SPA where crawlers must execute JS to see content.

---

## 13. TypeScript with React

**Definition:** using TypeScript with React means giving components, hooks, and events explicit types so prop mismatches, missing handlers, and incorrect state shapes are caught at compile time instead of at runtime.

```tsx
// typing props
interface UserCardProps {
  name: string;
  age?: number;                  // optional
  onSelect: (id: string) => void;
}
function UserCard({ name, age, onSelect }: UserCardProps) { /* ... */ }

// typing state
const [user, setUser] = useState<User | null>(null);

// typing events
function handleChange(e: React.ChangeEvent<HTMLInputElement>) { console.log(e.target.value); }

// typing refs
const inputRef = useRef<HTMLInputElement>(null);

// generic components
function List<T>({ items, renderItem }: { items: T[]; renderItem: (item: T) => React.ReactNode }) {
  return <ul>{items.map((item, i) => <li key={i}>{renderItem(item)}</li>)}</ul>;
}

// typing custom hooks
function useToggle(initial = false): [boolean, () => void] {
  const [value, setValue] = useState(initial);
  return [value, () => setValue(v => !v)];
}

// discriminated unions for component variants
type ButtonProps =
  | { variant: 'link'; href: string }
  | { variant: 'button'; onClick: () => void };
function Button(props: ButtonProps) {
  return props.variant === 'link' ? <a href={props.href} /> : <button onClick={props.onClick} />;
}
```

---

## 14. Testing

**Definition:** Testing React code covers **unit** tests (an isolated component/hook/function), **integration** tests (multiple components interacting), and **end-to-end** tests (the full app in a real browser).

### Jest / Vitest

**Definition:** test runners/frameworks (`describe`/`it`/`expect`) that execute test files and report pass/fail — Vitest is a newer, Vite-native alternative to Jest with a near-identical API and faster startup.

### React Testing Library

**Definition:** a testing utility that renders components into a jsdom environment and queries the result the way a real user would — by role, label, or visible text — rather than through component internals, encouraging tests that verify behavior and survive refactors.

```jsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

test('increments on click', async () => {
  render(<Counter />);
  const user = userEvent.setup();
  await user.click(screen.getByRole('button', { name: /increment/i }));
  expect(screen.getByText('1')).toBeInTheDocument();
});
```

**Mocking API calls (MSW) — Definition:** Mock Service Worker intercepts network requests at the network layer (not by stubbing `fetch` itself), so components under test make "real" requests that are transparently answered by mock handlers — closer to production behavior than manual `fetch` mocks.

```ts
import { http, HttpResponse } from 'msw';
export const handlers = [
  http.get('/api/users/:id', () => HttpResponse.json({ id: '1', name: 'Ada' })),
];
```

**Snapshot testing** — captures a component's rendered output and fails the test if it changes unexpectedly; useful for catching accidental UI changes, but easy to overuse into meaningless noise if applied to everything.

**Testing custom hooks (`renderHook`)** — renders a hook in a minimal test harness component so its return value/behavior can be asserted without needing a full component:

```jsx
import { renderHook, act } from '@testing-library/react';

test('useToggle flips value', () => {
  const { result } = renderHook(() => useToggle());
  act(() => result.current[1]());
  expect(result.current[0]).toBe(true);
});
```

**E2E — Playwright / Cypress** — exercise the full running app in a real browser; same role as described in the Angular notes' testing section — simulate genuine user flows across pages.

---

## 15. Architecture

**Definition:** Application architecture, in this context, is how a React codebase's folders and components are organized so a large application stays maintainable as it grows.

```
src/
 ├── app/               # app shell, routing, providers
 ├── components/        # shared, reusable UI
 ├── features/          # one folder per feature/domain
 │    └── orders/
 │         ├── components/
 │         ├── hooks/
 │         ├── api/
 │         └── types.ts
 ├── hooks/             # shared custom hooks
 ├── lib/               # utilities, API clients
 └── styles/
```

**Container/presentational pattern — Definition:** separating components that fetch data and hold state (**containers**) from components that only render UI based on props (**presentational**) — the same "smart vs dumb" split covered in the Angular notes' architecture section; largely superseded in React by custom hooks, but still a useful mental model.

**Compound components — Definition:** a pattern where several components are designed to work together, sharing implicit state via Context, so consumers compose them declaratively (`<Tabs><Tabs.List>...</Tabs.List><Tabs.Panel>...</Tabs.Panel></Tabs>`) instead of passing a large config object to one monolithic component.

**Render props (legacy) — Definition:** a pattern where a component takes a function as a prop (often named `children` or `render`) and calls it with internal state/data, letting the caller decide what to render — mostly replaced by custom hooks today, but still seen in some libraries.

```jsx
<Mouse render={({ x, y }) => <p>{x}, {y}</p>} />
```

**Higher-order components (legacy) — Definition:** a function that takes a component and returns a new, enhanced component (`withAuth(MyComponent)`) — the pre-hooks way to share cross-cutting logic; mostly replaced by custom hooks, which avoid the "wrapper hell" and prop-name collisions HOCs were prone to.

**Custom hooks as the modern replacement** — most logic that used to require a render prop or HOC (auth checks, subscriptions, form state) is now extracted into a `useSomething()` custom hook instead, called directly inside the components that need it.

**Monorepos (Nx, Turborepo) / Micro-frontends** — same rationale as covered in the Angular architecture notes: worth adopting once a codebase has multiple deployables or is large enough that a full rebuild/test cycle is slow.

---

## 16. Security

**Definition:** Web application security, in the React context, is the set of practices that prevent an application from being exploited via injected scripts, forged requests, or leaked credentials.

**XSS in React — Definition:** React escapes all values interpolated into JSX by default (`{userInput}` is always rendered as text, never parsed as HTML), which is why React apps are comparatively XSS-resistant out of the box. The escape hatch, `dangerouslySetInnerHTML`, bypasses this and must only be used with content you trust or have explicitly sanitized:

```jsx
// ❌ dangerous with untrusted input
<div dangerouslySetInnerHTML={{ __html: rawUserComment }} />

// ✅ sanitize first (e.g. with DOMPurify)
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(rawUserComment) }} />
```

**CSRF** — same concept as in the Angular notes: an attack that tricks an authenticated user's browser into submitting an unwanted request; defended against via SameSite cookies and/or a CSRF token echoed between cookie and header, implemented on the backend/API client rather than by React itself.

**Authentication patterns / JWT storage** — attach tokens via a fetch/axios interceptor rather than manually per call; prefer an HttpOnly cookie set by the server over `localStorage` for the same reason covered in the Angular security notes — client-readable storage is fully exposed to any XSS that does slip through.

**Protected routes** — see section 9; a client-side redirect-if-unauthenticated check, which is a UX nicety, not a real security boundary — the server/API must independently authorize every request.

**Dependency vulnerabilities** — run `npm audit` (or equivalent) regularly; a React app's attack surface includes every npm package it bundles into client-shipped JS, not just first-party code.

---

## 17. Production Engineering

**Definition:** Production engineering covers the build, deployment, and operational practices that get a React application safely and observably into users' hands.

**Build tools — Definition:** the tooling that bundles, transpiles, and optimizes source code into deployable static assets. **Vite** (dev server using native ES modules + esbuild, production bundling via Rollup) is the modern default; **webpack** is the long-standing, highly configurable bundler behind Create React App and many custom setups; **esbuild** is a Go-based bundler/transpiler valued for raw speed, often used as a building block inside Vite and other tools rather than configured directly.

**Environment configuration** — `.env` files (`VITE_API_URL=...` / `REACT_APP_API_URL=...` depending on tooling) injected at build time; never put secrets here — anything referenced in client code ships to the browser in plain text.

**Bundle analysis:**

```bash
npx vite-bundle-visualizer   # or rollup-plugin-visualizer / webpack-bundle-analyzer
```

**CI/CD, Docker, CDN & caching** — conceptually identical to the Angular production notes: an automated pipeline builds/tests/deploys on every change; a multi-stage Docker build produces a slim image serving the static build output (via Nginx or a Node server for SSR); a CDN serves hashed, immutable static assets close to users.

**Error tracking (Sentry) / logging:**

```jsx
import * as Sentry from '@sentry/react';

Sentry.init({ dsn: 'https://...' });

<Sentry.ErrorBoundary fallback={<p>Something went wrong.</p>}>
  <App />
</Sentry.ErrorBoundary>
```

**Source maps** — generate and upload them to your error tracker (not to end users) so minified production stack traces resolve back to real file/line.

**Feature flags** — gate new/risky code paths behind a flag (via a service like LaunchDarkly, or a simple config-driven boolean) so a bad change can be disabled without a redeploy, reducing the blast radius of production issues.

**Production debugging** — React DevTools works against production builds (component tree, Profiler) as long as the build isn't stripped of all debug info; pair with the error tracker's breadcrumbs/session replay to reconstruct what a user did before a crash.
