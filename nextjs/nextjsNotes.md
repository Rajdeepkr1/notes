# Next.js — Deep Dive Roadmap

We'll go from fundamentals → routing & rendering → data fetching & mutations → caching → optimization → auth → testing → production.

*Builds directly on the React notes — this file assumes React fundamentals (components, hooks, state) are already covered there and focuses specifically on what Next.js adds on top: routing, rendering strategies, Server Components, and the full-stack framework layer. Examples target the modern **App Router** throughout.*

---

## 1. Next.js Fundamentals

**Definition:** Next.js is a **React framework** — a layer built on top of React (React notes) that adds the pieces a real production application needs but React itself deliberately doesn't provide: file-based routing, server-side rendering, data-fetching conventions, bundling/build tooling, and a deployment story — the difference between "a UI library" (React) and "a full-stack framework for building an application" (Next.js).

**Why Next.js exists — Definition:** plain React (via Vite, React notes' section 1) gives you components and rendering, but leaves routing, server rendering, code splitting, and data-fetching conventions entirely up to you to assemble from separate libraries (React Router, a manual SSR setup, etc.) — Next.js exists specifically to provide official, integrated, opinionated answers to all of these at once, the same "batteries included" tradeoff already discussed for Django vs Flask in the Python Backend notes' section 13, applied here to the React ecosystem.

**The App Router vs the Pages Router (brief history) — Definition:** Next.js originally shipped with the **Pages Router** (`pages/` directory, one file per route, entirely Client-Component-by-default) — still supported, but now considered legacy for new projects. The **App Router** (`app/` directory, introduced in Next.js 13, the default and recommended choice since) is built around **React Server Components** (section 4) from the ground up, supporting nested layouts, streaming, and Server Actions natively — this roadmap covers the App Router throughout, as the modern default; Pages Router knowledge remains relevant mainly for maintaining older, existing codebases.

**Next.js project setup:**

```bash
npx create-next-app@latest my-app --typescript --app --tailwind
cd my-app
npm run dev   # http://localhost:3000
```

**The React Server Components model (orientation) — Definition:** the single biggest conceptual shift Next.js's App Router introduces over plain React — components render, **by default, on the server**, and their JavaScript is never shipped to the browser at all — only components explicitly opted into client-side interactivity (via `"use client"`, section 4) are bundled and hydrated in the browser the way *every* React component was in a plain client-rendered app — this fundamentally changes the default mental model from "everything runs in the browser" to "everything runs on the server unless you specifically say otherwise," covered in full depth in section 4.

---

## 2. Project Structure & App Router Basics

**Definition:** the App Router uses the filesystem itself as the routing configuration — the folder structure under `app/` directly determines the application's URL structure, with specially-named files within each folder defining that route segment's UI and behavior.

```
app/
 ├── layout.tsx        # root layout — wraps every page, required
 ├── page.tsx           # the UI for the "/" route
 ├── loading.tsx         # loading UI for "/" (and its children)
 ├── error.tsx            # error UI for "/" (and its children)
 ├── not-found.tsx          # 404 UI
 └── users/
      ├── page.tsx           # UI for "/users"
      └── [id]/
           └── page.tsx        # UI for "/users/:id" (dynamic route, section 3)
```

**Special files — Definition:** each of these filenames is reserved and given specific meaning by the framework within any route segment (folder):
- **`page.tsx`** — makes a route segment publicly accessible, rendering the actual UI for that URL; a folder *without* a `page.tsx` is not itself a navigable route (useful for organizing shared layouts/components without adding a URL).
- **`layout.tsx`** — shared UI that wraps a segment and all its nested children (section 7).
- **`loading.tsx`** — an automatic Suspense fallback shown while that segment's content is loading (section 8).
- **`error.tsx`** — an automatic error boundary for that segment (section 8).
- **`not-found.tsx`** — the UI shown when `notFound()` is called or a route doesn't match (section 8).

**Colocation of components, styles, and tests — Definition:** the App Router deliberately allows placing any **non-special-named** file (a component, a CSS module, a test file) directly alongside the route files that use it, within the same route folder — only the reserved filenames above are treated specially and made routable; everything else colocated in a route folder is simply organizational and entirely invisible to routing, encouraging feature-colocated project structure (the same principle as this workspace's Node.js/Angular architecture notes' feature-based folder structure) rather than grouping strictly by file type.

**Route segments & the file-to-URL mapping — Definition:** each folder nested under `app/` corresponds to one URL path **segment** — `app/users/settings/page.tsx` maps to `/users/settings` — the folder nesting depth directly mirrors the URL's path depth, and only folders containing (directly or via a route group) a `page.tsx` actually become reachable routes.

**Private folders & route groups (brief intro) — Definition:** a folder prefixed with an underscore (`_components/`) is a **private folder** — explicitly excluded from routing, guaranteed never to become a URL segment, used for colocating implementation details you want excluded from the route structure with certainty; **route groups** (`(marketing)/`, parentheses instead of an underscore) organize routes into logical groups **without** adding a segment to the URL at all — covered in full in section 3.

---

## 3. Routing

**File-based routing fundamentals (recap)** — see section 2; every `page.tsx` file's location under `app/` directly determines its URL.

**Dynamic routes — Definition:** a folder named `[paramName]` matches any single URL segment at that position, making its value available to the page as a route parameter — the App Router's direct equivalent of the React Router / Angular Router `:param` syntax already covered in the React and Angular notes.

```tsx
// app/users/[id]/page.tsx
export default function UserPage({ params }: { params: { id: string } }) {
  return <p>User ID: {params.id}</p>; // /users/42 → params.id === '42'
}
```

**Catch-all & optional catch-all routes — Definition:** `[...slug]` matches **any number** of remaining path segments, collecting them into an array (`/docs/a/b/c` → `slug: ['a', 'b', 'c']`) — used for deeply nested, variable-depth content structures (documentation sites, CMS-driven pages); `[[...slug]]` (double brackets) is the **optional** variant, additionally matching the base route itself with no further segments at all (`/docs` alone also matches, with `slug` undefined), where the plain single-bracket version would not match the bare parent route.

**Route groups — Definition:** a folder wrapped in parentheses, `(marketing)/`, groups related routes together for **organizational purposes only** — it does not add a segment to the resulting URL at all — used to apply a distinct layout (section 7) to a subset of routes, or simply to organize a large route tree logically, without that organization leaking into the actual URL structure.

```
app/
 ├── (marketing)/
 │    ├── layout.tsx      # a marketing-specific layout
 │    ├── about/page.tsx    # → /about (NOT /marketing/about)
 │    └── pricing/page.tsx   # → /pricing
 └── (app)/
      ├── layout.tsx      # a different, app-specific layout
      └── dashboard/page.tsx  # → /dashboard
```

**Parallel routes — Definition:** a folder named `@slotName` defines a **named slot** that a layout can render independently alongside its other content — enables rendering multiple, independently-loading/erroring sections of a page simultaneously (e.g. a dashboard's main content and a sidebar analytics panel, each with its own `loading.tsx`/`error.tsx`) without one slot's loading state blocking the others.

**Intercepting routes — Definition:** lets a route "intercept" another route's URL and render different content **in the current layout context**, while the browser's URL updates to reflect the intercepted route — the classic use case is a photo grid where clicking a photo opens it in a modal *on top of* the grid (intercepting the route) while a full page refresh or direct navigation to that same URL renders the photo's full, standalone page instead — denoted with `(.)`, `(..)`, or `(...)` folder-naming conventions indicating how many segment levels up to intercept from.

**`Link`, `useRouter`, `usePathname`, `useSearchParams` — Definition:** `<Link href="/about">` is Next.js's client-side navigation component (analogous to React Router's `<Link>`/Angular's `routerLink`), automatically prefetching the linked route's code in the background for instant subsequent navigation; `useRouter()` provides imperative navigation (`router.push('/dashboard')`) for use inside event handlers; `usePathname()`/`useSearchParams()` read the current URL's path/query parameters reactively — all four are **Client Component-only** hooks/APIs (section 4), unavailable directly inside a Server Component.

```tsx
'use client';
import { useRouter } from 'next/navigation';

function LogoutButton() {
  const router = useRouter();
  return <button onClick={async () => { await logout(); router.push('/login'); }}>Log out</button>;
}
```

---

## 4. Server Components vs Client Components

**Definition:** the App Router's foundational architectural distinction — every component is a **Server Component by default**, rendered entirely on the server with zero of its own JavaScript shipped to the browser; a component only becomes a **Client Component** — hydrated and interactive in the browser, the way all of plain React's components are — when explicitly marked with the `"use client"` directive.

**The mental model shift from plain React — Definition:** in a plain client-rendered React app (React notes' sections 1–6), *every* component's code ships to and runs in the browser, by default and without exception; in the Next.js App Router, the **default is now the opposite** — a component runs only on the server, produces static HTML/RSC payload output, and never becomes JS the browser needs to download or execute at all, unless it specifically opts into client rendering — this default flip is the single most important thing to internalize before writing App Router code, since code written with plain-React habits (assuming everything runs in the browser) will frequently reach for browser-only APIs or React hooks that simply don't work in a Server Component.

**What runs where, and why it matters — Definition:** Server Components can directly access server-only resources — a database (SQL/MongoDB notes), the filesystem, environment secrets — with **zero risk of ever exposing that access to the client**, since the code implementing it never ships to the browser at all; this is a meaningfully stronger security/architecture boundary than a plain SPA calling a separate API, where the actual database credentials at least have to live somewhere the API server can reach, but the frontend code calling that API is still fully client-shipped. Server Components also mean **less JavaScript sent to the browser** overall (a direct, built-in performance win, section 20), since only the components that genuinely need interactivity are ever bundled for the client.

**The `"use client"` directive — Definition:** a string literal placed at the very top of a file, marking that component (and everything it directly imports, transitively) as a Client Component — required for any component using React state/effects/browser APIs/event handlers, or any of the navigation hooks from section 3.

```tsx
'use client'; // must be the first line

import { useState } from 'react';

export function Counter() {
  const [count, setCount] = useState(0); // useState requires a Client Component
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

**Composing Server and Client Components together — Definition:** a Server Component can render a Client Component as a child (the common, expected pattern — a mostly-static Server Component page embedding one small interactive island); a Client Component, however, **cannot** directly import and render a Server Component as its own child the usual way (since once you're in client-rendered territory, everything beneath it in that render tree is client-rendered too) — the standard workaround is passing a Server Component down **as `children`** (or another prop) from a parent Server Component, letting the Client Component simply render whatever pre-rendered content it was handed, without needing to import/render a Server Component's source directly itself.

```tsx
// Server Component
import { ClientWrapper } from './ClientWrapper';
import { ServerContent } from './ServerContent'; // stays server-rendered

export default function Page() {
  return (
    <ClientWrapper>
      <ServerContent /> {/* passed as children — composition, not a Client Component importing a Server one */}
    </ClientWrapper>
  );
}
```

**Passing data from Server to Client Components (the serialization boundary) — Definition:** props passed from a Server Component to a Client Component must be **serializable** (plain objects, arrays, strings, numbers — the same constraint as `JSON.stringify`, plus a few React-specific exceptions like Server Actions, section 9) — you cannot pass a function, a class instance, or a database connection object as a prop across this boundary, since the Client Component's code runs in an entirely separate JavaScript environment (the browser) that never had access to the server-side value in the first place; this is the direct, concrete manifestation of the "different code, different environment" split described above.

**When you're forced into a Client Component** — using any React hook that manages client-side state/lifecycle (`useState`, `useEffect`, `useReducer`); handling browser events (`onClick`, `onChange`); using browser-only APIs (`window`, `localStorage`); using React Context (`useContext`, since Context requires a client-side Provider); using the navigation hooks from section 3 — the practical rule of thumb is to mark the **smallest, most specific** component that actually needs one of these as the Client Component ("push client components down the tree"), keeping as much of the page as possible in the zero-JS-shipped Server Component default.

---

## 5. Data Fetching in Server Components

**Definition:** Server Components can be declared `async` and use `await` **directly in the component body** — no `useEffect`, no separate loading-state management, no client-side data-fetching library needed for data a Server Component itself needs, since the fetch simply happens on the server before that component's HTML is ever produced.

```tsx
// Server Component — no useEffect, no loading state needed for this fetch
async function UserProfile({ userId }: { userId: string }) {
  const res = await fetch(`https://api.example.com/users/${userId}`);
  const user = await res.json();
  return <h1>{user.name}</h1>;
}
```

**Fetching in parallel vs sequentially — Definition:** multiple independent `await fetch(...)` calls written one after another, each awaited individually, run **sequentially** (each waits for the previous to finish before starting) — exactly the same "accidentally serialized independent async work" pitfall already covered in the JS/TS notes' section 7 and Python Backend notes' section 8; fetching genuinely independent data in **parallel** requires explicitly kicking off all the requests before awaiting any of them (`Promise.all`, or simply calling multiple `fetch()`s without awaiting each individually before starting the next), the same fix pattern as those other notes' sections.

```tsx
// ❌ sequential — the second fetch doesn't start until the first fully finishes
const user = await fetch(`/api/users/${id}`).then(r => r.json());
const orders = await fetch(`/api/orders?userId=${id}`).then(r => r.json());

// ✅ parallel — both requests are in flight simultaneously
const [user, orders] = await Promise.all([
  fetch(`/api/users/${id}`).then(r => r.json()),
  fetch(`/api/orders?userId=${id}`).then(r => r.json()),
]);
```

**Request deduplication — Definition:** if multiple components within the same render pass call `fetch()` with the exact same URL and options, Next.js automatically **deduplicates** them into a single actual network request, sharing the result across every component that requested it — this "Request Memoization" caching layer (detailed further in section 12) means you don't need to manually thread fetched data down through props purely to avoid duplicate requests; each component can independently fetch exactly the data it needs, and Next.js collapses genuinely identical requests behind the scenes automatically.

**Colocating data fetching with the component that needs it — Definition:** because deduplication (above) removes the traditional performance cost of "just fetch it again in each component that needs it," the App Router's idiomatic pattern is to let **each Server Component fetch its own data directly**, rather than the older pattern (common in Pages Router and plain SPA code) of fetching everything once at a page's top level and threading it down through many layers of props — keeps each component more self-contained and independently reasoned-about, similar in spirit to the composability benefits of custom hooks in plain React (React notes' section 3), but for server-side data specifically.

**Passing fetched data down vs fetching in multiple components** — colocated fetching (above) is the default, idiomatic choice; explicitly fetching once and passing data down via props remains the right choice when a specific value genuinely needs to be shared/kept consistent across sibling components in a way that re-fetching independently in each wouldn't guarantee (e.g. if the underlying data could change between two nominally-identical fetches due to real-world timing), or when a fetch is expensive enough that even deduplicated, you want tight, explicit control over exactly when and how often it runs.

---

## 6. Rendering Strategies

**Definition:** Next.js supports multiple strategies for *when* a route's HTML is actually generated — at build time, at request time, or periodically in the background — each with different performance and freshness tradeoffs, and the App Router automatically infers which strategy applies to a given route based on what that route's code actually does, rather than requiring it to always be manually specified.

**Static Rendering (SSG) — Definition:** HTML is generated **once, at build time**, and reused for every subsequent request — the fastest possible strategy (serving a pre-built, cacheable file, potentially straight from a CDN, AWS notes' section 11) — the App Router's **default** for any route that doesn't use any dynamic, request-specific APIs (below).

**Dynamic Rendering (SSR) — Definition:** HTML is generated **fresh, on every request**, at request time on the server — necessary whenever a route's content genuinely depends on request-specific information (the current user's session, request headers/cookies, search params that affect content) that can't be known ahead of time at build — a route is automatically opted into dynamic rendering the moment its code uses a dynamic API (reading cookies/headers, or an uncached `fetch` call, section 12).

**Incremental Static Regeneration (ISR) — Definition:** combines Static Rendering's speed with the ability to **periodically refresh** the cached static output in the background, without requiring a full site rebuild — configured via a `revalidate` time on a `fetch` call (or an exported `revalidate` constant) — after that time window passes, the *next* request still receives the (now-stale) cached version instantly, while Next.js regenerates a fresh version in the background for subsequent requests — giving most of static rendering's speed with data that doesn't need to be perfectly real-time.

```tsx
// ISR — this page's data is revalidated at most every 60 seconds
async function ProductsPage() {
  const res = await fetch('https://api.example.com/products', { next: { revalidate: 60 } });
  const products = await res.json();
  return <ProductList products={products} />;
}
```

**Client-Side Rendering (CSR) — when it still applies — Definition:** for data that's genuinely user-specific, frequently changing, and doesn't need to be present in the initial server-rendered HTML at all (e.g. real-time data polled after the page has already loaded), fetching directly in a Client Component (via `useEffect`, or a data-fetching library like TanStack Query, React notes' section 10) remains the right tool — the App Router doesn't eliminate CSR as an option, it simply makes Server Component data fetching (section 5) the new *default* starting point rather than CSR being the only choice, the way it was in a plain SPA.

**What forces a route into dynamic rendering — Definition:** using `cookies()` or `headers()` (reading request-specific data); using `searchParams` in a page component; an uncached `fetch` call (`{ cache: 'no-store' }`, section 12); using a Route Handler that reads the incoming `Request` object directly (section 10) — any of these signal to Next.js that this route's output genuinely depends on the specific incoming request and cannot safely be reused as static output across different users/requests.

**Choosing a rendering strategy per route** — Static/ISR for content that's the same (or near-enough) for every visitor and doesn't need to reflect the current request specifically (marketing pages, blog posts, product listings that update periodically); Dynamic (SSR) for genuinely per-request/per-user content (a personalized dashboard, anything reading the current session); CSR for post-load, client-only interactivity/data that doesn't need to be part of the initial HTML at all — the App Router's automatic inference (above) means this is often decided implicitly by what your code does, rather than requiring an explicit, separate configuration choice per route.

---

## 7. Layouts & Templates

**Definition:** a `layout.tsx` defines UI that's **shared** across a route segment and all of its nested child routes — rendered once and persisted across navigations between its child routes, rather than being torn down and rebuilt on every navigation the way a full page re-render would.

**Nested layouts & shared UI — Definition:** layouts nest according to the same folder structure as routes (section 2/3) — a root layout wraps the entire application, and each nested folder can define its own additional layout wrapping just that subtree — building up a composed layout automatically, without any single layout file needing to know about or duplicate the layouts above it.

```tsx
// app/layout.tsx — root layout, required, wraps EVERYTHING
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Header />
        {children}
      </body>
    </html>
  );
}

// app/dashboard/layout.tsx — an additional layout, nested inside the root layout
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="dashboard-shell">
      <Sidebar />
      <main>{children}</main>
    </div>
  );
}
```

**Root layout requirements — Definition:** every App Router application must have exactly one root `app/layout.tsx`, which must render the `<html>` and `<body>` tags itself (unlike every other layout, which renders only its own wrapping content plus `children`) — the one mandatory structural anchor the entire route tree hangs off of.

**Layouts vs templates (state preservation differences) — Definition:** a `layout.tsx` **persists** across navigations between its child routes — its component instance is not remounted, so any local state it holds is preserved as a user navigates between sibling routes beneath it; a `template.tsx` looks nearly identical in usage but creates a **new instance on every navigation** — state is reset, effects re-run — used specifically on the rare occasion you *want* that reset behavior (e.g. resetting a UI animation on every page transition within a section) rather than a layout's default state-preserving behavior.

**Layout groups for multiple root layouts — Definition:** combining route groups (section 3) with layouts lets an application define **entirely separate root-level layouts** for different sections (e.g. a completely different `<html>`/shell structure for a marketing site section vs an authenticated app section) — each route group under `app/` can have its own top-level `layout.tsx`, since route groups don't add a URL segment but do participate fully in the layout-nesting structure.

---

## 8. Loading, Error & Not Found UI

**`loading.tsx` & React Suspense integration — Definition:** a `loading.tsx` file in any route segment automatically wraps that segment's `page.tsx` (and its data fetching, section 5) in a **React Suspense boundary**, using the `loading.tsx` component as the `fallback` — shown automatically while that segment's async Server Component work is still in progress, with zero manual `<Suspense>` JSX needed for this common case.

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return <Spinner />; // shown automatically while dashboard/page.tsx's data fetching is in flight
}
```

**Streaming SSR with Suspense boundaries — Definition:** because each route segment's `loading.tsx` creates its own Suspense boundary, Next.js can **stream** the HTML response — sending the parts of the page that are already ready immediately, while slower-loading segments continue rendering on the server and are streamed in afterward, filling in their `loading.tsx` placeholders progressively — the same streaming/`@defer`-style benefit already covered conceptually in the Angular notes' section 12 and React notes' section 6, here built directly into the framework's default routing/rendering behavior rather than requiring an explicit opt-in per component.

**`error.tsx` & error boundaries — Definition:** an `error.tsx` file automatically wraps its route segment in a React error boundary, catching any error thrown during that segment's rendering (including errors from Server Component data fetching) and rendering the error UI instead of crashing the entire application — `error.tsx` **must** be a Client Component (errors need to be caught and potentially retried client-side), receiving the thrown `error` and a `reset()` function to attempt re-rendering the segment.

```tsx
'use client'; // error.tsx is always a Client Component

export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div>
      <p>Something went wrong: {error.message}</p>
      <button onClick={() => reset()}>Try again</button>
    </div>
  );
}
```

**`not-found.tsx` & `notFound()` — Definition:** `not-found.tsx` defines the UI shown for a genuinely unmatched route, or when a Server Component explicitly calls the `notFound()` function (e.g. after fetching data and discovering the requested resource doesn't actually exist) — immediately halts rendering of that segment and renders the nearest `not-found.tsx` instead, the App Router's built-in equivalent of the classic wildcard `**` 404 route already covered in the Angular/React routing notes.

**Granular loading states per route segment** — because `loading.tsx`/`error.tsx` are defined **per route segment** (not just once, globally), different parts of a deeply-nested route tree can each have their own independent loading/error UI — a slow-loading sidebar widget doesn't need to block or share a loading spinner with the main page content next to it, each segment's boundary is independent.

---

## 9. Server Actions & Mutations

**Definition:** a Server Action is an **asynchronous function that runs exclusively on the server**, callable directly from Client (or Server) Component code — including directly as a `<form>`'s `action` — without manually building a separate API endpoint (section 10) and a client-side `fetch` call to reach it; the framework handles the network call transparently.

**The `"use server"` directive — Definition:** marks a function (or every exported function in a file) as a Server Action — placed either at the top of a dedicated file (marking every export in it) or as the first line inside an individual `async function` — Next.js compiles a Server Action into a secure, callable endpoint automatically, without you needing to hand-write a Route Handler for it.

```tsx
// app/actions.ts
'use server';

export async function createTodo(formData: FormData) {
  const title = formData.get('title') as string;
  await db.todos.insertOne({ title });
  revalidatePath('/todos'); // refresh the todos list after mutating (section 12)
}
```

**Calling a Server Action from a form (progressive enhancement) — Definition:** passing a Server Action directly as a `<form>`'s `action` prop lets the form submit and correctly trigger the server-side mutation **even before any client-side JavaScript has loaded/hydrated** — genuine progressive enhancement, since the underlying mechanism builds on the browser's native form-submission behavior rather than depending entirely on client-side JS to intercept the submit event, a meaningfully more resilient default than a typical SPA form's `onSubmit` + `fetch()` pattern.

```tsx
import { createTodo } from './actions';

export default function NewTodoForm() {
  return (
    <form action={createTodo}>
      <input name="title" />
      <button type="submit">Add</button>
    </form>
  );
}
```

**Calling a Server Action from a Client Component — Definition:** Server Actions can also be invoked directly as regular async function calls from event handlers within a Client Component (not just as a form's `action`), e.g. `onClick={() => deleteTodo(id)}` — letting the same server-side mutation logic be reused across both a plain-HTML-form submission path and a more interactive, client-driven UI interaction.

**Revalidating data after a mutation — Definition:** `revalidatePath('/todos')`/`revalidateTag('todos')` (section 12) explicitly invalidate previously-cached data after a Server Action performs a mutation, so the next request/render reflects the change — without calling one of these, a page that was statically cached (section 6) or whose data was cached (section 12) would continue showing stale data even after the underlying mutation succeeded, since Next.js has no automatic way to know a given mutation should invalidate a given piece of previously-fetched, cached data.

**Error handling & pending states with `useActionState`/`useFormStatus` — Definition:** `useFormStatus()` (called from a Client Component nested inside the `<form>`) exposes the form's current submission `pending` state, letting you show a loading indicator on the submit button without manually wiring up your own pending-state tracking; `useActionState()` wraps a Server Action to additionally capture and expose its return value (e.g. a validation error message) back into the calling component's render, giving a clean, built-in pattern for surfacing Server Action-side validation errors back into the UI.

---

## 10. Route Handlers (API Routes)

**Definition:** a `route.ts` file defines a traditional HTTP endpoint (a REST API route) directly within the `app/` directory — the App Router's direct equivalent of an Express route handler (Node.js notes' section 4), used when you need a genuine HTTP API (called from an external client, a mobile app, or a webhook) rather than a Server Action's tighter, React-specific mutation mechanism.

```ts
// app/api/users/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest, { params }: { params: { id: string } }) {
  const user = await db.users.findById(params.id);
  if (!user) return NextResponse.json({ error: 'Not found' }, { status: 404 });
  return NextResponse.json(user);
}

export async function DELETE(request: NextRequest, { params }: { params: { id: string } }) {
  await db.users.deleteById(params.id);
  return new NextResponse(null, { status: 204 });
}
```

**Supported HTTP methods** — a Route Handler file exports one `async function` per HTTP method it supports (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`), each receiving the incoming `NextRequest` (an extension of the standard Web `Request` object) and returning a `NextResponse` (extending the standard Web `Response`) — built on genuine web-standard APIs rather than a framework-proprietary request/response shape.

**Route Handlers vs Server Actions (when to use which) — Definition:** Server Actions (section 9) are the right choice for mutations triggered *from within your own Next.js application's UI* (a form submission, a button click) — tightly integrated with React, requiring no separate API contract to design; Route Handlers are the right choice when you need a genuine, standalone HTTP endpoint — consumed by an external client (a mobile app, a third-party webhook receiver, a public API), or when you specifically need fine-grained control over HTTP semantics (status codes, headers, streaming responses) that Server Actions' more constrained RPC-style model doesn't expose as directly.

**Building a REST API inside a Next.js app** — Route Handlers can implement a complete REST API (mirroring the same resource-routing, status-code, and validation conventions already covered in the Node.js notes' section 6) directly within a Next.js application, without needing a wholly separate backend service — a common, legitimate architecture for small-to-medium applications where a separate dedicated backend isn't yet justified.

---

## 11. Middleware

**Definition:** a single `middleware.ts` file at the project root defines code that runs **before** a request completes routing to its matched page/Route Handler — able to inspect, redirect, rewrite, or modify the response for matching requests, running on Next.js's lightweight **Edge Runtime** rather than a full Node.js environment.

```ts
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('session')?.value;
  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*'], // only runs middleware for matching paths
};
```

**Common use cases** — authentication redirects (the example above — the same route-guard concept as the Angular/React notes' auth guards, enforced at the very edge of the request rather than client-side); A/B testing (rewriting a request to serve a different variant of a page based on a cookie/random assignment); geolocation-based routing (redirecting users to a locale-specific route based on their detected region); bot detection and request filtering before a request ever reaches your application code.

**`matcher` config — Definition:** the exported `config.matcher` restricts which request paths actually trigger the middleware function — since middleware runs on essentially every request by default, scoping it explicitly to only the paths that genuinely need it (rather than running unnecessary logic on every single static asset request) is both a meaningful performance consideration and simply clearer about the middleware's actual intended scope.

**Middleware limitations (Edge runtime constraints) — Definition:** middleware runs on the **Edge Runtime**, a deliberately restricted JavaScript environment (not full Node.js) optimized for extremely fast cold-starts at edge locations close to users — this means many Node.js-specific APIs and npm packages that depend on native Node.js modules (a full database driver, for instance) are **not available** inside middleware — middleware is best suited to lightweight, fast decisions (read a cookie, redirect, rewrite) rather than heavier logic like a full database query, which belongs in a Server Component, Server Action, or Route Handler instead.

---

## 12. Caching in Next.js

**Definition:** the App Router layers **four distinct caching mechanisms**, each operating at a different scope and lifetime — understanding which layer is responsible for a given piece of unexpectedly-stale (or unexpectedly-fresh) data is the single most common source of Next.js-specific debugging confusion, making this arguably the most important section in this file to genuinely internalize.

**The four caching layers — Definition:**
1. **Request Memoization** — deduplicates identical `fetch()` calls made during the **same render pass** (section 5) — scoped to a single request/render only, automatically cleared afterward, requiring no configuration.
2. **Data Cache** — persists `fetch()` results **across requests and deployments**, on the server — this is what powers ISR (section 6); controlled explicitly via `fetch`'s `cache`/`next.revalidate` options (below).
3. **Full Route Cache** — caches the entire rendered **HTML and RSC payload** for statically-rendered routes at build time, served directly on matching requests without re-running any component code at all.
4. **Router Cache** (Client-side Router Cache) — an **in-browser** cache of already-visited route segments' RSC payloads, enabling instant back/forward navigation and prefetching (via `<Link>`, section 3) without a fresh server round-trip.

**`fetch` caching options — Definition:** Next.js extends the native `fetch()` API with additional, framework-specific options controlling the Data Cache (layer 2 above):

```ts
fetch(url);                                     // default: cached indefinitely (static-like behavior)
fetch(url, { cache: 'no-store' });                // never cached — always a fresh request (forces dynamic rendering)
fetch(url, { next: { revalidate: 60 } });          // cached, revalidated at most every 60s (ISR, section 6)
fetch(url, { next: { tags: ['products'] } });       // cached, taggable for on-demand invalidation (below)
```

**On-demand revalidation — Definition:** `revalidatePath('/products')` invalidates the cache for a specific route path; `revalidateTag('products')` invalidates every cached `fetch` call anywhere in the app tagged with that tag (regardless of which route it was fetched from) — both callable from a Server Action (section 9) or Route Handler (section 10), the mechanism by which a mutation ("a product was updated") tells Next.js "the previously-cached product data is now stale, regenerate it on next request" — tag-based invalidation in particular is powerful for invalidating data consistently across multiple different routes that all happen to display the same underlying resource.

**Opting out of caching** — `{ cache: 'no-store' }` on a specific `fetch` call, or exporting `export const dynamic = 'force-dynamic'` from a route file to opt the *entire route* out of static rendering and the Full Route Cache — necessary for genuinely request-specific content that must never be served from a shared cache.

**Debugging unexpected caching behavior** — the practical troubleshooting approach: identify which of the four layers is actually responsible for the stale data you're seeing (is it a specific `fetch` call's Data Cache, or the whole route's Full Route Cache, or just the client-side Router Cache from a previous visit) — a hard browser refresh clears the Router Cache (layer 4) but not the server-side layers; explicitly setting `cache: 'no-store'` and reloading isolates whether the Data Cache (layer 2) was the culprit — treating "just add `revalidatePath` everywhere defensively" as a first resort rather than actually diagnosing which layer is stale tends to produce a codebase with confusing, inconsistent caching behavior overall.

---

## 13. Metadata & SEO

**The Metadata API — Definition:** rather than manually managing `<head>` tags, a route file exports a `metadata` object (for static values) or a `generateMetadata` async function (for values that depend on fetched data) — Next.js automatically injects the resulting `<title>`, `<meta>`, and related tags into the rendered HTML `<head>`.

```tsx
// static metadata
export const metadata = {
  title: 'My App',
  description: 'The best app for doing things.',
};

// dynamic metadata — depends on this specific route's data
export async function generateMetadata({ params }: { params: { id: string } }) {
  const product = await getProduct(params.id);
  return { title: product.name, description: product.description };
}
```

**Static vs dynamic metadata** — a plain exported `metadata` object suffices for routes whose title/description are fixed and known ahead of time; `generateMetadata` (an `async` function, able to `fetch` just like a Server Component, section 5) is required whenever the metadata genuinely depends on that specific route's dynamic data (a product page's title needs to be the actual product's name, not a generic placeholder).

**Open Graph & Twitter card metadata — Definition:** the `metadata` object also supports `openGraph`/`twitter` fields controlling how a link to the page renders when shared on social platforms (the preview image, title, description shown in a Slack/Twitter/Facebook link unfurl) — configured declaratively alongside the standard title/description metadata, rather than requiring separately hand-managed meta tags.

**`sitemap.ts` & `robots.ts` — Definition:** special files that, when present, automatically generate a `/sitemap.xml` and `/robots.txt` respectively — `sitemap.ts` exports a function returning an array of URLs (with optional priority/change-frequency hints) helping search engines discover and prioritize crawling your site's pages; `robots.ts` controls which crawlers/paths are allowed or disallowed — both directly supporting the SEO benefits of server rendering already covered in the React notes' section 12.

**Structured data (JSON-LD) — Definition:** embedding a `<script type="application/ld+json">` tag containing schema.org-formatted structured data directly in a page's output — helps search engines understand a page's content more precisely (e.g. marking up a product page's price/availability/reviews explicitly) than plain HTML/metadata alone conveys, often enabling richer search-result presentations (star ratings, price ranges shown directly in search results).

---

## 14. Images, Fonts & Static Assets

**`next/image` — Definition:** Next.js's built-in `<Image>` component automatically optimizes images — resizing to the actual rendered dimensions, serving modern formats (WebP/AVIF) where the browser supports them, lazy-loading images below the fold by default, and preventing layout shift by reserving the correct space before the image loads — directly addressing several of the image-related Core Web Vitals concerns already covered in the HTML/CSS notes' section 14, with essentially zero manual configuration required beyond specifying dimensions.

```tsx
import Image from 'next/image';

<Image src="/hero.jpg" alt="Hero banner" width={1200} height={600} priority />
// `priority` opts this specific image out of lazy-loading — for above-the-fold, immediately-visible images
```

**`next/font` — Definition:** self-hosts web fonts (including Google Fonts) automatically at build time — the font files are downloaded once during the build and served from your own domain, rather than the browser making a separate request to an external font provider's servers at runtime — eliminates the extra network round-trip and the layout-shift/FOUT concerns covered in the HTML/CSS notes' section 14, with the correct `font-display` and fallback-metric behavior configured automatically.

```tsx
import { Inter } from 'next/font/google';
const inter = Inter({ subsets: ['latin'] });

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return <html lang="en" className={inter.className}>{children}</html>;
}
```

**The `public/` directory — Definition:** files placed in `public/` at the project root are served as-is, directly at the root URL path (`public/favicon.ico` → `/favicon.ico`) — the location for static assets that don't need any build-time processing (raw downloadable files, `robots.txt` if not using `robots.ts`, favicon files).

**Static asset caching (recap)** — Next.js automatically applies long-lived, immutable cache headers to its own build output (hashed JS/CSS bundles, optimized images served through `next/image`), the same hashed-filename-plus-far-future-cache-header strategy already covered in the Angular/React/Node.js production-engineering sections, handled automatically rather than requiring manual server/CDN configuration.

---

## 15. Styling in Next.js

**CSS Modules (built-in support) — Definition:** any file named `*.module.css` is automatically treated as a CSS Module — scoped, locally-unique class names, generated automatically at build time — with zero extra configuration required, the same CSS Modules concept already covered in the React notes' section 8, natively built into Next.js's default tooling rather than requiring separate setup.

```tsx
import styles from './Button.module.css';
export function Button() { return <button className={styles.primary}>Click</button>; }
```

**Global CSS — Definition:** a plain (non-module) `.css` file imported into the **root layout** applies globally across the entire application — Next.js specifically restricts global CSS imports to the root layout (or `_app` in the legacy Pages Router) to avoid the unpredictable, load-order-dependent global style conflicts that importing global CSS from arbitrary component files could otherwise cause.

**Tailwind CSS integration — Definition:** first-class, officially-supported integration (`create-next-app --tailwind`, section 1) — Tailwind's utility classes (HTML/CSS notes' section 10) work identically whether applied within a Server or Client Component, since Tailwind's output is plain, static CSS with no runtime JavaScript dependency at all, requiring no special handling across the Server/Client Component boundary (section 4) the way some CSS-in-JS approaches do (below).

**CSS-in-JS considerations under Server Components — Definition:** traditional runtime CSS-in-JS libraries (styled-components, classic Emotion usage, HTML/CSS notes' section 8) generate their styles by executing JavaScript **in the browser** at render time — which is fundamentally incompatible with Server Components' entire premise of not shipping component JavaScript to the client at all; using a runtime CSS-in-JS library forces the components using it into Client Components, losing Server Components' benefits for that part of the tree — CSS Modules or Tailwind (both producing plain, static CSS with no runtime JS dependency) are the better-aligned default choices under the App Router specifically for this reason; some CSS-in-JS libraries have introduced Server-Component-compatible, build-time/zero-runtime modes specifically to address this tension, worth checking for explicitly if a team has an existing investment in a particular CSS-in-JS library.

**Sass support** — built-in, first-class support for `.scss`/`.sass` files (Sass notes, HTML/CSS notes' section 9) requires only installing the `sass` package, with no additional Next.js configuration needed beyond that.

---

## 16. Authentication & Authorization Patterns

**Session vs token-based auth in an App Router context (recap)** — see the Node.js/Java/Python Backend notes' respective auth sections for the underlying session-vs-JWT tradeoffs; in a Next.js App Router application, session state is most commonly stored in an **HttpOnly cookie** (readable server-side in Server Components/Middleware/Route Handlers via `cookies()`, but deliberately inaccessible to client-side JavaScript) — a stronger security default than a JS-readable token in `localStorage`, and naturally well-suited to Next.js's server-first rendering model.

**Protecting routes with Middleware (recap)** — see section 11's example; checking for a valid session cookie in `middleware.ts` and redirecting unauthenticated requests before they ever reach a protected route's actual page code — the App Router's most common pattern for route-level access control, running at the very edge of the request rather than requiring each individual page to implement its own redirect-if-unauthenticated check.

**Reading auth state in Server Components — Definition:** `cookies()` (from `next/headers`) reads the incoming request's cookies directly within a Server Component — letting a Server Component fetch the current user's session/identity server-side and use it to conditionally fetch/render user-specific data, without needing a separate client-side "who am I" API call the way a plain SPA typically requires.

```tsx
import { cookies } from 'next/headers';

async function Dashboard() {
  const sessionToken = cookies().get('session')?.value;
  const user = await getUserFromSession(sessionToken);
  return <p>Welcome, {user.name}</p>;
}
```

**Popular auth libraries (brief)** — **Auth.js** (formerly NextAuth.js) — the most widely-used, open-source, self-hosted auth solution purpose-built for Next.js, handling OAuth providers, session management, and the Middleware/Server Component integration patterns above; **Clerk** — a managed, hosted auth-as-a-service alternative, trading self-hosting control for significantly less auth infrastructure to build/maintain yourself — the same self-hosted-vs-managed tradeoff already discussed for various other infrastructure choices throughout this workspace (e.g. the AWS notes' RDS discussion).

**Server Actions & auth checks — Definition:** because a Server Action (section 9) is a real server-side function directly callable from client code, it must **independently re-verify** the caller's authentication/authorization on every invocation — the same principle as the Angular/AWS notes' repeated point that client-side route guards are a UX nicety, never the actual security boundary; a Server Action performing a sensitive mutation should check the current session server-side, inside the action itself, rather than trusting that only an already-authenticated UI would ever call it.

---

## 17. Environment Variables & Configuration

**`.env` file conventions (recap)** — see the Node.js/Python/Java notes' respective environment-configuration sections; Next.js automatically loads `.env`, `.env.local`, `.env.production`, etc., following a defined precedence order, without needing a separate library like `dotenv` (already built in).

**`NEXT_PUBLIC_` prefix (client-exposed vars) — Definition:** by default, environment variables are only available **server-side** (inside Server Components, Route Handlers, Server Actions) — a variable prefixed with `NEXT_PUBLIC_` is additionally **inlined directly into the client-side JavaScript bundle at build time**, making it accessible in Client Components too — this prefix is a deliberate, explicit opt-in specifically because anything exposed this way is visible to anyone inspecting the shipped bundle, so genuinely sensitive values (API secrets, database credentials) must **never** carry this prefix.

```bash
# .env.local
DATABASE_URL=postgres://...              # server-only — safe for secrets
NEXT_PUBLIC_ANALYTICS_ID=UA-12345         # bundled into client JS — must not be sensitive
```

**`next.config.js` key options — Definition:** the project-wide configuration file — common options include `images` (configuring allowed external image domains for `next/image`, section 14), `redirects`/`rewrites` (URL-level redirect/rewrite rules configured declaratively, an alternative to handling these in Middleware, section 11, for simple, static cases), and `output` (controlling the build output mode, e.g. `'standalone'` for self-hosted Docker deployment or `'export'` for a fully static build, both covered in section 19).

**Runtime config vs build-time config — Definition:** most Next.js configuration (including environment variables read server-side) is resolved at **build time** — baked into the built output; genuinely runtime-configurable values (needed to change without a full rebuild, e.g. across different deployment environments sharing one build artifact, section 4 of the Deployment notes) require either reading them dynamically inside Server Components/Route Handlers (which do run fresh per-request, not baked in) or Next.js's more specialized runtime-config mechanisms — a distinction worth being deliberate about, since assuming build-time-resolved config is always freely runtime-changeable is a common source of confusing, environment-specific bugs.

---

## 18. Testing Next.js Applications

**Unit testing components (recap)** — see the React notes' section 14 in full; standard React Testing Library patterns apply directly to Client Components in a Next.js app without modification.

**Testing Server Components — Definition:** genuinely testing a Server Component in true isolation is harder than testing a plain Client Component, since Server Components are `async` functions producing React elements rather than something React Testing Library's `render()` (built around the client-rendering model) directly supports out of the box — the more common, pragmatic pattern is testing the underlying data-fetching/business logic (section 5) as plain, standalone async functions independent of the component (unit tests, straightforward), and relying on integration/E2E tests (below) to validate the Server Component's actual rendered output end-to-end.

**Testing Server Actions — Definition:** since a Server Action (section 9) is fundamentally just an async function, it can be unit-tested directly by importing and calling it with test input (a mock `FormData`), asserting on its side effects (did it call the database correctly) and return value — the same straightforward function-testing approach as testing any other backend service function (Node.js/Python/Java notes' respective testing sections), since a Server Action's *mechanism* of being callable from the client doesn't change how it's tested when called directly in a test.

**E2E testing (Playwright) against a running Next.js app (recap)** — see the React notes' section 14; Playwright exercises a genuinely running Next.js application (`next dev` or a production build) in a real browser, the most reliable way to validate the full, real Server Component + Client Component + routing + Server Action integration actually works correctly end-to-end, complementing the narrower unit tests above.

**Mocking `fetch`/data sources — Definition:** for tests that do render a Server Component (via a testing setup that supports it, or via integration/E2E tests), mocking the underlying data source (a database client, or `fetch` itself via a library like MSW, already covered in the React notes' section 14) keeps tests fast and independent of a real backend/network — the same mocking discipline already established throughout this workspace's testing sections, applied here specifically to server-side data fetching rather than client-side.

---

## 19. Deployment

**Deploying to Vercel (the zero-config path) — Definition:** Vercel (the company behind Next.js) provides first-party, zero-configuration hosting specifically optimized for Next.js — connecting a Git repository automatically builds and deploys on every push, with the Full Route Cache, ISR, Edge Middleware, and Image Optimization (sections 12, 11, 14) all working out of the box with no additional infrastructure setup — the path of least friction, and the reference implementation every one of this framework's caching/rendering features was originally designed against.

**Self-hosting (Node.js server, standalone output) — Definition:** Next.js can also be self-hosted on any Node.js-capable infrastructure — `next build` followed by `next start` runs a Node.js server implementing the same rendering/caching behavior; the `output: 'standalone'` config option (section 17) produces a minimal, self-contained build (only the exact dependencies actually needed at runtime, not the full `node_modules`) specifically optimized for container-based deployment.

**Docker deployment (recap)** — see the Docker/Kubernetes notes' section 2; a typical Next.js Dockerfile uses a multi-stage build (install deps → `next build` with `output: 'standalone'` → copy only the standalone output into a slim final image) — the same multi-stage-build pattern already covered for Node.js/Java applications, applied to Next.js's specific standalone build output.

**Static export (`output: 'export'`) — when it applies and its limitations — Definition:** produces a fully static HTML/CSS/JS build with **no Node.js server required at all** — deployable to any static file host/CDN (the same deployment model as a plain React SPA build, section 13 of the Deployment notes) — but this mode is incompatible with any feature requiring a live server: Server Actions, Route Handlers reading dynamic request data, ISR's on-demand revalidation, and Middleware are all unavailable in a static export — appropriate only for applications that are genuinely, entirely static, with no dynamic server-side behavior needed at all.

**Edge vs Node.js runtime per route — Definition:** individual routes (Route Handlers, and increasingly pages themselves) can opt into running on the lightweight Edge Runtime (section 11's same restricted environment, but available beyond just Middleware) instead of the default full Node.js runtime — trading away full Node.js API/package compatibility for faster cold starts and execution physically closer to users at edge locations — a per-route choice (`export const runtime = 'edge'`) made based on whether a specific route's logic is simple/fast enough to fit the Edge Runtime's constraints and whether its latency profile actually benefits meaningfully from edge execution.

---

## 20. Performance Optimization & Interview Prep

**Bundle analysis (recap)** — see the React notes' section 17; `@next/bundle-analyzer` visualizes exactly what's contributing to the client-side JavaScript bundle size — particularly valuable in a Next.js context for verifying that Server Components are actually keeping non-interactive parts of a page out of the client bundle as intended (section 4), rather than assuming it's working correctly without checking.

**Code splitting (automatic + `next/dynamic`) — Definition:** Next.js automatically code-splits by route (each route's JS is its own chunk, loaded only when that route is actually visited, the same route-based splitting already covered in the React notes' section 6); `next/dynamic` additionally allows explicitly lazy-loading a specific **component** (not just an entire route) — commonly used for a large, rarely-needed Client Component (a rich chart library, a complex modal) that shouldn't be included in the initial bundle for a page that only sometimes needs it.

```tsx
import dynamic from 'next/dynamic';
const HeavyChart = dynamic(() => import('./HeavyChart'), { loading: () => <Spinner /> });
```

**Streaming & Suspense for perceived performance (recap)** — see section 8; the App Router's default streaming behavior means a slow-loading section of a page no longer needs to block the entire page's initial paint, a direct, built-in perceived-performance win over the older model where an entire page waited for its single slowest data dependency before rendering anything at all.

**Core Web Vitals in a Next.js context (recap)** — see the HTML/CSS notes' section 14 for the underlying metrics (LCP, CLS, etc.); Next.js's built-in `next/image` (preventing layout shift, section 14), `next/font` (eliminating font-swap layout shift), and Server Components (reducing client-side JS, improving interactivity metrics) each directly and specifically target one or more of these metrics as a core design goal of the framework itself, not an afterthought bolted on separately.

**Common Next.js interview questions** — explain the difference between Server and Client Components, and how you decide which a given component should be (section 4); walk through the four caching layers and how they interact (section 12); explain the difference between SSG, SSR, and ISR, and when you'd choose each (section 6); how does streaming SSR with Suspense improve perceived performance over a traditional, blocking SSR page (section 8); when would you reach for a Server Action versus a Route Handler (sections 9–10).

**Common pitfalls & anti-patterns:**
- Marking a component `"use client"` far higher in the tree than necessary, unintentionally pulling large portions of the app out of the Server Component default and into the client bundle (section 4's "push client components down" guidance).
- Fetching sequentially (accidentally) instead of in parallel across independent Server Component data needs (section 5).
- Forgetting `revalidatePath`/`revalidateTag` after a mutation, leaving cached pages showing stale data indefinitely (section 12).
- Putting heavy, Node.js-dependent logic in Middleware, hitting Edge Runtime compatibility limits (section 11).
- Using a runtime CSS-in-JS library without checking Server Component compatibility, unintentionally forcing large parts of the UI into Client Components purely for styling (section 15).
- Choosing static export (`output: 'export'`) for an application that actually needs Server Actions or dynamic Route Handlers, then discovering that incompatibility late (section 19).
