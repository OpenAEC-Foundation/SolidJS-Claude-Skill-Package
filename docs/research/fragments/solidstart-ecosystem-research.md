# SolidStart & Ecosystem Research

> Research date: 2026-03-19
> Sources: docs.solidjs.com, github.com/solidjs/solid, github.com/solidjs-community/solid-primitives

---

## 1. SolidStart Overview

SolidStart is the official meta-framework for SolidJS. It provides full-stack capabilities on top of SolidJS's reactive primitives.

**Architecture stack:**
- **SolidJS** — fine-grained reactivity, component rendering
- **Vite** — bundling, dev server
- **Nitro** — server runtime, deployment adapters
- **Vinxi** — CLI and configuration layer combining Vite + Nitro

**Key features:**
- File-based routing (via `@solidjs/start/router` `<FileRoutes />`)
- Server functions (`"use server"` directive)
- Multiple rendering modes: CSR, SSR (sync, async, streaming), SSG
- Isomorphic code — write once, runs on client and server
- Deployment presets: Netlify, Vercel, AWS, Cloudflare, etc.

**Version history:**
- SolidStart 0.x — early beta, different APIs (e.g., `createServerData$`, `createServerAction$`)
- SolidStart 1.0 — stable release, uses `"use server"` directive, integrates with solid-router `query`/`action`/`createAsync`

**Source:** https://docs.solidjs.com/solid-start

### 1.1 Getting Started

```bash
npm init solid@latest      # or pnpm/yarn/bun/deno equivalents
```

Available templates: basic, bare, with-solidbase, with-auth, with-authjs, with-drizzle, with-mdx, with-prisma, with-solid-styled, with-tailwindcss.

**Default project structure:**
```
public/
src/
├── routes/
│   └── index.tsx
├── entry-client.tsx    # Client-side hydration entry
├── entry-server.tsx    # Server request handler entry
└── app.tsx             # HTML root component (shared client/server)
```

- `src/` is aliased as `~/` for imports
- `entry-client.tsx` and `entry-server.tsx` rarely need modification
- `app.tsx` is the root component wrapping `<Router>` and `<FileRoutes />`

**Minimum app.tsx:**
```tsx
import { Suspense } from "solid-js";
import { Router } from "@solidjs/router";
import { FileRoutes } from "@solidjs/start/router";

export default function App() {
  return (
    <Router root={(props) => <Suspense>{props.children}</Suspense>}>
      <FileRoutes />
    </Router>
  );
}
```

**Source:** https://docs.solidjs.com/solid-start/getting-started

---

## 2. File-Based Routing

SolidStart uses the `routes/` directory for automatic route generation. The `<FileRoutes />` component from `@solidjs/start/router` traverses this directory and generates route definitions.

### 2.1 Basic Route Mapping

| File | URL |
|------|-----|
| `routes/index.tsx` | `/` |
| `routes/blog.tsx` | `/blog` |
| `routes/contact.tsx` | `/contact` |

Each route file MUST default-export a component.

### 2.2 Nested Routes

Folders create nested URL segments:
- `routes/blog/article-1.tsx` → `/blog/article-1`
- `routes/work/job-1.tsx` → `/work/job-1`

### 2.3 Nested Layouts

A file with the same name as a folder becomes the layout wrapper for that folder's routes:

```
routes/
  blog.tsx              # Layout for /blog/*
  blog/
    article-1.tsx       # /blog/article-1
    article-2.tsx       # /blog/article-2
```

Layout component receives `props.children`:
```tsx
import { RouteSectionProps } from "@solidjs/router";

export default function BlogLayout(props: RouteSectionProps) {
  return (
    <div>
      <h1>Blog</h1>
      {props.children}
    </div>
  );
}
```

### 2.4 Index File Alternatives

Rename `index.tsx` to `(foldername).tsx` for better file searchability:
```
routes/socials/(socials).tsx    # renders at /socials
```

### 2.5 Dynamic Routes

Square brackets for dynamic segments:

| File | URL Pattern |
|------|-------------|
| `routes/users/[id].tsx` | `/users/:id` |
| `routes/users/[id]/[name].tsx` | `/users/:id/:name` |

```tsx
import { useParams } from "@solidjs/router";

export default function UserPage() {
  const params = useParams();
  return <div>User {params.id}</div>;
}
```

### 2.6 Optional Parameters

Double brackets for optional segments:
- `routes/users/[[id]].tsx` → matches `/users`, `/users/1`, `/users/abc`

### 2.7 Catch-All Routes

Ellipsis syntax for wildcard matching:
- `routes/blog/[...post].tsx` → captures all remaining segments

```tsx
export default function BlogPage() {
  const params = useParams();
  // params.post = "foo/bar/baz" (forward-slash delimited)
  return <div>Blog {params.post}</div>;
}
```

### 2.8 Route Groups

Parenthesized folder names organize routes without affecting URLs:
```
routes/
  (static)/
    about-us/
      index.tsx         # /about-us
    contact-us/
      index.tsx         # /contact-us
```

### 2.9 Escaping Nested Routes

Parenthesized folders can also escape nested layout inheritance:
```
routes/
  users/
    index.tsx           # /users (under users layout)
    projects.tsx        # /users/projects (under users layout)
  users(details)/
    [id].tsx            # /users/:id (separate layout, NOT under users/)
```

### 2.10 Route Configuration

Export a `route` object alongside the default component:
```tsx
import { RouteSectionProps, RouteDefinition } from "@solidjs/router";

export const route = {
  preload() {
    // preload data before navigation
  }
} satisfies RouteDefinition;

export default function UsersLayout(props: RouteSectionProps) {
  return <div><h1>Users</h1>{props.children}</div>;
}
```

**Source:** https://docs.solidjs.com/solid-start/building-your-application/routing

---

## 3. Data Loading

### 3.1 query — Server-Side Data Fetching with Caching

```typescript
function query<T extends (...args: any) => any>(
  fn: T,
  name: string
): CachedFunction<T>;
```

The `query` function wraps an async fetcher, caches results, and enables deduplication + revalidation.

```typescript
import { query } from "@solidjs/router";

const getUser = query(async (userId: string) => {
  "use server";
  const response = await fetch(`https://api.example.com/users/${userId}`);
  if (!response.ok) throw new Error("Failed to load user");
  return response.json();
}, "userProfile");
```

**Cache key management:**
- `getUser.key` → `"userProfile"` (base key)
- `getUser.keyFor(userId)` → `"userProfile[\"abc123\"]"` (argument-specific key)

**Caching behavior — data is reused without re-fetching when:**
1. Within 5-second preloading window (hover-triggered)
2. While components actively use the query via `createAsync`
3. On back/forward navigation (up to 5 minutes)
4. Repeated calls within a single SSR request (server-side deduplication)
5. During client hydration (uses server-provided data)

**Key serialization:** Arguments are serialized with `JSON.stringify` with sorted keys — `{ name: "Ryan", awesome: true }` and `{ awesome: true, name: "Ryan" }` produce the same cache key.

**Source:** https://docs.solidjs.com/solid-router/reference/data-apis/query, https://docs.solidjs.com/solid-router/reference/data-apis/cache

### 3.2 createAsync — Reactive Async Data Primitive

```typescript
function createAsync<T>(
  fn: (prev: T | undefined) => Promise<T>,
  options?: {
    name?: string;
    initialValue?: T;
    deferStream?: boolean;
  }
): AccessorWithLatest<T | undefined>;
```

The recommended async primitive. Currently wraps `createResource` internally, but intended to be the standard in Solid 2.0.

```tsx
import { createAsync } from "@solidjs/router";

const user = createAsync(() => getUser(params.id));

return (
  <ErrorBoundary fallback={<p>Error</p>}>
    <Suspense fallback={<p>Loading...</p>}>
      <div>{user()?.name}</div>
    </Suspense>
  </ErrorBoundary>
);
```

**Key behaviors:**
- Returns `undefined` until Promise resolves (unless `initialValue` provided)
- Automatically re-runs when reactive dependencies change
- Integrates with `<Suspense>` for loading states
- Integrates with `<ErrorBoundary>` for error handling
- `deferStream` defers streaming SSR until fetcher completes

**Source:** https://docs.solidjs.com/solid-router/reference/data-apis/create-async

### 3.3 createAsyncStore — Fine-Grained Async Store

```typescript
function createAsyncStore<T>(
  fn: (prev: T | undefined) => Promise<T>,
  options?: {
    name?: string;
    initialValue?: T;
    deferStream?: boolean;
    reconcile?: ReconcileOptions;
  }
): AccessorWithLatest<T | undefined>;
```

Like `createAsync` but uses store reconciliation — when new data arrives, it intelligently merges with existing store, updating only changed fields while preserving unchanged state.

```tsx
const notifications = createAsyncStore(() =>
  getNotificationsQuery(unreadOnly())
);

return (
  <For each={notifications()}>
    {(notification) => <li>{notification.message}</li>}
  </For>
);
```

**Source:** https://docs.solidjs.com/solid-router/reference/data-apis/create-async-store

### 3.4 createResource — Lower-Level Async Primitive

```typescript
const [data, { mutate, refetch }] = createResource(source, fetcher, options?);
```

**Return value properties:**
- `data()` — resolved value accessor
- `data.state` — `"unresolved"` | `"pending"` | `"ready"` | `"refreshing"` | `"errored"`
- `data.loading` — boolean
- `data.error` — error value
- `data.latest` — most recent resolved value (survives refetch)
- `mutate(value)` — optimistic update
- `refetch()` — manual re-fetch

```tsx
const [userId, setUserId] = createSignal<string>();
const [user] = createResource(userId, fetchUser);

return (
  <Suspense fallback={<p>Loading...</p>}>
    <Show when={user()}>{(u) => <div>{u().name}</div>}</Show>
  </Suspense>
);
```

**IMPORTANT:** Do NOT use `cache`/`query` directly inside `createResource` — the fetcher is not reactive and will not invalidate properly. Use `createAsync` with `query` instead.

**Source:** https://docs.solidjs.com/guides/fetching-data

### 3.5 Route Preloading

```typescript
type RoutePreloadFunc<T = unknown> = (args: RoutePreloadFuncArgs) => T;

interface RoutePreloadFuncArgs {
  params: Params;
  location: Location;
  intent: "initial" | "native" | "navigate" | "preload";
}
```

Preload functions run before navigation to eliminate data waterfalls:

```tsx
import { Route, query, createAsync } from "@solidjs/router";

const getProduct = query(async (id: string) => {
  "use server";
  return db.getProduct(id);
}, "product");

function preloadProduct({ params }: RoutePreloadFuncArgs) {
  getProduct(params.id);  // fire-and-forget, warms cache
}

// In route definition:
<Route path="/products/:id" component={ProductPage} preload={preloadProduct} />
```

The `intent` field distinguishes between:
- `"preload"` — hover-triggered, return value ignored
- `"navigate"` / `"initial"` — actual navigation, return value becomes `data` prop

**Source:** https://docs.solidjs.com/solid-router/reference/preload-functions/preload

---

## 4. Server Functions

The `"use server"` directive marks functions to execute exclusively on the server. The compiler transforms them into RPC calls.

### 4.1 Function-Level Usage

```typescript
const getUser = async (id: string) => {
  "use server";
  return db.getUser(id);
};
```

### 4.2 File-Level Usage

```typescript
"use server";
// ALL exports in this file run server-only

export async function getUser(id: string) {
  return db.getUser(id);
}

export async function getProducts() {
  return db.getProducts();
}
```

### 4.3 Requirements

- Functions MUST be `async` or return a `Promise`
- Arguments and return values MUST be serializable (uses Seroval for serialization)
- Functions run server-side regardless of rendering mode (CSR, SSR, etc.)

### 4.4 Integration with query/action

```typescript
const getUser = query((id: string) => {
  "use server";
  return db.getUser(id);
}, "users");

const updateUser = action(async (id: string, data: UserData) => {
  "use server";
  await db.setUser(id, data);
  throw redirect("/", { revalidate: getUser.keyFor(id) });
});
```

### 4.5 Single-Flight Mutations

Server-thrown `redirect()` is handled server-side — the redirected page's data streams in the same HTTP response. This avoids a client-side round-trip.

### 4.6 Meta Information

```typescript
import { getServerFunctionMeta } from "@solidjs/start/server";
```

Provides stable function identifiers for multi-worker/multi-core environments.

**Source:** https://docs.solidjs.com/solid-start/reference/server/use-server

---

## 5. Actions and Mutations

### 5.1 action() — Mutation Wrapper

```typescript
function action<T extends Array<any>, U = void>(
  fn: (...args: T) => Promise<U>,
  name?: string
): Action<T, U>;

// Alternative with options:
function action<T extends Array<any>, U = void>(
  fn: (...args: T) => Promise<U>,
  options?: { name?: string; onComplete?: (s: Submission<T, U>) => void }
): Action<T, U>;
```

Actions create submission objects that track execution status.

### 5.2 Form Integration

```tsx
import { action } from "@solidjs/router";

const addTodo = action(async (formData: FormData) => {
  "use server";
  const title = formData.get("title") as string;
  await db.addTodo(title);
}, "addTodo");

function TodoForm() {
  return (
    <form action={addTodo} method="post">
      <input name="title" />
      <button>Add</button>
    </form>
  );
}
```

### 5.3 Passing Additional Arguments

Use `.with()` to prepend arguments before `FormData`:

```tsx
const addTodo = action(async (userId: number, formData: FormData) => {
  "use server";
  // userId comes from .with(), formData from form submission
}, "addTodo");

<form action={addTodo.with(currentUserId)} method="post">
  <input name="title" />
  <button>Add</button>
</form>
```

### 5.4 Programmatic Invocation with useAction

```tsx
import { useAction } from "@solidjs/router";

const markDone = action(async (id: string) => {
  "use server";
  await db.markTodoDone(id);
});

function TodoItem(props: { id: string }) {
  const doMarkDone = useAction(markDone);
  return <button onClick={() => doMarkDone(props.id)}>Done</button>;
}
```

### 5.5 Submission Tracking with useSubmission

```tsx
import { useSubmission } from "@solidjs/router";

const submission = useSubmission(addTodo);
// submission.pending — boolean
// submission.result — resolved value
// submission.error — error if failed
// submission.input — original arguments
```

`useSubmissions` (plural) tracks ALL active submissions for an action.

### 5.6 Automatic Revalidation

After an action completes successfully, ALL active queries on the same page are automatically revalidated — unless revalidation is explicitly configured via response helpers.

### 5.7 Response Helpers

- **`redirect(url, options?)`** — server-side redirect after action
- **`json(data, options?)`** — return JSON response
- **`reload(options?)`** — force page reload after action
- **`revalidate(key?)`** — manually revalidate specific or all queries

```typescript
import { redirect, revalidate } from "@solidjs/router";

const updateUser = action(async (id: string, data: UserData) => {
  "use server";
  await db.updateUser(id, data);
  throw redirect("/users", { revalidate: getUser.keyFor(id) });
});
```

**IMPORTANT:** `redirect()` is thrown, not returned, from server functions.

**Source:** https://docs.solidjs.com/solid-router/reference/data-apis/action

---

## 6. SSR and Hydration

### 6.1 renderToString — Synchronous SSR

```typescript
function renderToString<T>(
  fn: () => T,
  options?: { nonce?: string; renderId?: string }
): string;
```

Renders synchronously, generates hydration script inline. Good for static pages without async data.

```typescript
const html = renderToString(() => <App />);
```

**Source:** https://docs.solidjs.com/reference/rendering/render-to-string

### 6.2 renderToStream — Streaming SSR

```typescript
function renderToStream<T>(
  fn: () => T,
  options?: {
    nonce?: string;
    renderId?: string;
    onCompleteShell?: () => void;
    onCompleteAll?: () => void;
  }
): {
  pipe: (writable: { write: (v: string) => void }) => void;
  pipeTo: (writable: WritableStream) => void;
};
```

Renders synchronous shell with Suspense fallbacks, then streams async content as it resolves.

```typescript
const stream = renderToStream(() => <App />);
stream.pipe(res);       // Node.js writable
stream.pipeTo(writable); // Web Streams API
```

**Callbacks:**
- `onCompleteShell` — fires when sync rendering completes (before first flush)
- `onCompleteAll` — fires when all Suspense boundaries settle

**Note:** `renderToStream` replaces the deprecated `pipeToWritable` and `pipeToNodeWritable` APIs.

**Source:** https://docs.solidjs.com/reference/rendering/render-to-stream

### 6.3 hydrate — Client-Side Rehydration

```typescript
function hydrate(
  fn: () => JSX.Element,
  node: MountableElement,
  options?: { renderId?: string; owner?: unknown }
): () => void;
```

Attaches client-side reactivity to server-rendered HTML without re-rendering the DOM. Returns a dispose function.

```typescript
import { hydrate } from "solid-js/web";
const dispose = hydrate(() => <App />, document.getElementById("app")!);
```

**Source:** https://docs.solidjs.com/reference/rendering/hydrate

### 6.4 HydrationScript / generateHydrationScript

```typescript
function generateHydrationScript(options: {
  nonce?: string;
  eventNames?: string[];  // default: ["click", "input"]
}): string;

function HydrationScript(props: {
  nonce?: string;
  eventNames?: string[];
}): JSX.Element;
```

Bootstrap script that captures user events before Solid's runtime loads, then replays them after hydration. MUST be placed once on the page.

Default captured events: `click` and `input` (delegated, composed, bubbling events only).

**Source:** https://docs.solidjs.com/reference/rendering/hydration-script

### 6.5 isServer

Boolean constant — `true` on server, `false` on client. Tree-shaken at build time.

```typescript
import { isServer } from "solid-js/web";

if (isServer) {
  // Server-only code — removed from client bundle
}
```

---

## 7. API Routes

SolidStart supports server-side API endpoints alongside UI routes.

### 7.1 File Conventions

API routes use the same file naming as UI routes but export HTTP method handlers instead of a default component. Commonly placed in `routes/api/`.

```
routes/
  api/
    users/
      [id].ts          # /api/users/:id
    products.ts         # /api/products
```

### 7.2 HTTP Method Handlers

```typescript
export function GET() { }
export function POST() { }
export function PATCH() { }
export function DELETE() { }
```

Multiple methods can share a handler:
```typescript
async function handler(event: APIEvent) { /* ... */ }
export const GET = handler;
export const POST = handler;
```

### 7.3 APIEvent Type

```typescript
interface APIEvent {
  request: Request;          // Standard Web Request object
  params: Record<string, string>;  // Dynamic route params
  fetch: (input: RequestInfo, init?: RequestInit) => Promise<Response>;  // Internal fetch
}
```

### 7.4 Example

```typescript
import type { APIEvent } from "@solidjs/start/server";

export async function GET({ params }: APIEvent) {
  const products = await store.getProducts(params.category);
  return products;  // Automatically serialized to JSON
}
```

### 7.5 Session/Cookie Access

```typescript
import { getCookie } from "vinxi/http";

export async function GET(event: APIEvent) {
  const userId = getCookie("userId");
  if (!userId) return new Response("Unauthorized", { status: 401 });
  return await store.getUser(event.params.userId);
}
```

**IMPORTANT:** API routes are prioritized over UI routes at the same path. Use `Accept` headers to differentiate if they overlap.

**Source:** https://docs.solidjs.com/solid-start/building-your-application/api-routes

---

## 8. Solid Router (Standalone)

`@solidjs/router` — universal routing for SolidJS applications, client and server.

**Requires:** Solid v1.8.4 or later.

```bash
npm install @solidjs/router
```

### 8.1 Router Setup

```tsx
import { Router, Route } from "@solidjs/router";

function App() {
  return (
    <Router>
      <Route path="/" component={Home} />
      <Route path="/about" component={About} />
      <Route path="/users/:id" component={User} />
    </Router>
  );
}
```

Alternative: config-based routing with route definition objects.

### 8.2 Core Components

#### `<A>` — Navigation Link
Enhanced anchor tag with client-side navigation, active state, and base path support.

```tsx
<A href="/about">About</A>
<A href="/users" activeClass="font-bold" end>Users</A>
```

| Prop | Type | Description |
|------|------|-------------|
| `href` | `string` | Target path (relative or absolute) |
| `noScroll` | `boolean` | Disable scroll-to-top |
| `replace` | `boolean` | Replace history entry |
| `state` | `unknown` | Push state to history |
| `activeClass` | `string` | CSS class when link matches current URL |
| `inactiveClass` | `string` | CSS class when link does NOT match |
| `end` | `boolean` | Exact match only (prevents `/` matching everything) |

**Source:** https://docs.solidjs.com/solid-router/reference/components/a

#### `<Route>` — Route Definition

```tsx
<Route
  path="/users/:id"
  component={UserPage}
  preload={preloadUser}
  matchFilters={{ id: /^\d+$/ }}
/>
```

| Prop | Type | Description |
|------|------|-------------|
| `path` | `string \| string[]` | URL pattern(s) |
| `component` | `Component` | Rendered component |
| `preload` | `RoutePreloadFunc` | Data preloading function |
| `matchFilters` | `MatchFilters` | Additional match constraints |
| `children` | `JSX.Element` | Nested routes |

Multiple paths keep component mounted: `<Route path={["/login", "/register"]} component={AuthPage} />`

**Source:** https://docs.solidjs.com/solid-router/reference/components/route

#### `<Navigate>` — Programmatic Redirect Component
```tsx
<Navigate href="/dashboard" />
```

#### `<HashRouter>` — Hash-Based Navigation
```tsx
<HashRouter>{/* routes */}</HashRouter>
```
Uses `/#/path` URLs.

#### `<MemoryRouter>` — Testing Router
In-memory navigation without browser history.

### 8.3 Navigation Hooks

#### useNavigate

```typescript
function useNavigate(): (
  to: string | number,
  options?: Partial<NavigateOptions>
) => void;

interface NavigateOptions<S = unknown> {
  resolve: boolean;   // default: true (resolve relative to current route)
  replace: boolean;   // default: false
  scroll: boolean;    // default: true
  state: S;           // stored in history.state
}
```

```tsx
const navigate = useNavigate();
navigate("/dashboard");                    // basic
navigate("/dashboard", { replace: true }); // replace history
navigate(-1);                              // go back
navigate("/checkout", { state: { from: "cart" } }); // with state
```

**Source:** https://docs.solidjs.com/solid-router/reference/primitives/use-navigate

#### useParams

```tsx
const params = useParams();
// From /users/:id → params.id
```

#### useSearchParams

```tsx
const [search, setSearch] = useSearchParams();
// From ?page=1&sort=name → search.page, search.sort
setSearch({ page: "2" });
```

#### useLocation

```tsx
const location = useLocation();
// location.pathname, location.search, location.hash, location.state
```

#### useMatch

```tsx
const match = useMatch(() => "/users/:id");
// Returns match object or undefined
```

#### useIsRouting

```tsx
const isRouting = useIsRouting();
// Boolean — true during navigation transitions
```

#### useBeforeLeave

```tsx
useBeforeLeave((e) => {
  if (hasUnsavedChanges() && !confirm("Discard changes?")) {
    e.preventDefault();
  }
});
```

#### useCurrentMatches

```tsx
const matches = useCurrentMatches();
// Array of all matched route segments
```

#### usePreloadRoute

```tsx
const preload = usePreloadRoute();
// Trigger preloading on hover:
<div onMouseEnter={() => preload("/dashboard")}>Dashboard</div>
```

### 8.4 Lazy Loading

```tsx
import { lazy } from "solid-js";
const Dashboard = lazy(() => import("./pages/Dashboard"));

<Route path="/dashboard" component={Dashboard} />
```

**Source:** https://docs.solidjs.com/solid-router

---

## 9. Context API

### 9.1 createContext

```typescript
import { createContext } from "solid-js";

const MyContext = createContext<T>(defaultValue?);
```

Returns a context object with a `Provider` property.

### 9.2 Provider Pattern

```tsx
<MyContext.Provider value={someValue}>
  {props.children}
</MyContext.Provider>
```

### 9.3 useContext

```typescript
import { useContext } from "solid-js";

const value = useContext(MyContext);
```

### 9.4 Reactive Context with Signals

```tsx
const CounterContext = createContext<[Accessor<number>, { increment: () => void }]>();

function CounterProvider(props: ParentProps) {
  const [count, setCount] = createSignal(0);
  const counter = [
    count,
    { increment: () => setCount((prev) => prev + 1) }
  ] as const;

  return (
    <CounterContext.Provider value={counter}>
      {props.children}
    </CounterContext.Provider>
  );
}
```

### 9.5 Custom Hook Pattern (Recommended)

```tsx
function useCounter() {
  const context = useContext(CounterContext);
  if (!context) throw new Error("useCounter must be used within CounterProvider");
  return context;
}
```

### 9.6 Nested Contexts

Multiple contexts can coexist. Inner providers override outer ones for the same context.

**Source:** https://docs.solidjs.com/concepts/context

---

## 10. Ecosystem

### 10.1 Solid Primitives

`@solid-primitives/*` — community-maintained reactive utilities for SolidJS.

**Philosophy:** Wrap client/server functionality to provide fully reactive API layers. All packages are independently published, tree-shakeable, and well-tested.

**Key packages by category:**

**Input Handling:**
| Package | Key Exports | Purpose |
|---------|-------------|---------|
| `@solid-primitives/active-element` | `createActiveElement`, `createFocusSignal` | Track focused DOM elements |
| `@solid-primitives/keyboard` | `useKeyDownList`, `createShortcut`, `createKeyHold` | Keyboard interactions |
| `@solid-primitives/mouse` | `createMousePosition`, `createPositionToElement` | Cursor position tracking |
| `@solid-primitives/pointer` | `createPointerListeners`, `createPointerPosition` | Pointer events |
| `@solid-primitives/scroll` | `createScrollPosition`, `useWindowScrollPosition` | Scroll behavior |
| `@solid-primitives/input-mask` | `createInputMask`, `createMaskPattern` | Input formatting |

**Display & Media:**
| Package | Key Exports | Purpose |
|---------|-------------|---------|
| `@solid-primitives/media` | `createMediaQuery`, `createBreakpoints`, `usePrefersDark` | Media queries |
| `@solid-primitives/resize-observer` | `createResizeObserver`, `createElementSize` | Element size tracking |
| `@solid-primitives/intersection-observer` | `createIntersectionObserver`, `createViewportObserver` | Element visibility |
| `@solid-primitives/bounds` | `createElementBounds` | Element dimensions |
| `@solid-primitives/page-visibility` | `createPageVisibility` | Page focus state |
| `@solid-primitives/audio` | `makeAudio`, `makeAudioPlayer` | Audio playback |

**Browser APIs:**
| Package | Key Exports | Purpose |
|---------|-------------|---------|
| `@solid-primitives/event-listener` | `createEventListener`, `createEventSignal` | Unified event handling |
| `@solid-primitives/clipboard` | `copyClipboard`, `writeClipboard` | Copy/paste |
| `@solid-primitives/broadcast-channel` | `makeBroadcastChannel` | Cross-tab communication |
| `@solid-primitives/filesystem` | `createFileSystem` | File system access |
| `@solid-primitives/fullscreen` | `createFullscreen` | Fullscreen API |
| `@solid-primitives/idle` | `createIdleTimer` | User inactivity detection |
| `@solid-primitives/devices` | `createDevices`, `createMicrophones`, `createCameras` | Hardware access |

40+ total primitives across all categories, ranging from Stage 0 (experimental) to Stage 3 (stable).

**Source:** https://github.com/solidjs-community/solid-primitives

### 10.2 Other Notable Ecosystem Packages

- **Kobalte** (`@kobalte/core`) — accessible, unstyled UI component library (similar to Radix UI)
- **solid-transition-group** — CSS transition utilities for enter/exit animations
- **babel-preset-solid** — JSX compilation preset (required build dependency)
- **solid-testing-library** — testing utilities following Testing Library conventions
- **solid-devtools** — browser devtools extension for inspecting signals and components

**Source:** https://github.com/solidjs/solid

---

## 11. SolidJS Version Matrix

### SolidJS Core

| Version | Status | Key Features |
|---------|--------|-------------|
| 1.x (1.8+) | Stable, current | Fine-grained reactivity, signals, stores, effects, Suspense, ErrorBoundary, SSR, streaming, portals, web components |
| 2.0 | Planned/In development | `createAsync` as standard primitive (replacing `createResource`), potential API refinements |

**Browser support:** Last 2 years of Firefox, Safari, Chrome, Edge.
**Server support:** Node LTS, Deno, Cloudflare Workers.

### SolidStart

| Version | Status | Key Differences |
|---------|--------|----------------|
| 0.x | Legacy/deprecated | Used `createServerData$`, `createServerAction$`, different routing APIs |
| 1.0 | Stable, current | Uses `"use server"` directive, integrates with solid-router `query`/`action`/`createAsync`, built on Vinxi/Vite/Nitro |

### Solid Router

| Version | Key Changes |
|---------|-------------|
| < 0.15 | Used `cache` function (now deprecated) |
| 0.15+ | Introduced `query` (replaces `cache`), current stable API |

**Requires:** SolidJS v1.8.4 or later.

### Key Version Dependencies

- SolidStart 1.0 requires `@solidjs/router` 0.15+
- `@solidjs/router` requires `solid-js` 1.8.4+
- `createAsync` is the bridge API between 1.x and 2.0

**Sources:** https://github.com/solidjs/solid, https://docs.solidjs.com/solid-router

---

## 12. React vs SolidJS Anti-Patterns (Routing/SSR)

### 12.1 React Router vs Solid Router

| Concept | React (React Router) | SolidJS (Solid Router) |
|---------|---------------------|----------------------|
| Link component | `<Link to="/path">` | `<A href="/path">` |
| Navigate hook | `useNavigate()` returns `navigate(path)` | `useNavigate()` returns `navigate(path, options?)` — same concept |
| Route params | `useParams()` | `useParams()` — same API name |
| Search params | `useSearchParams()` returns `[searchParams, setSearchParams]` | `useSearchParams()` returns `[params, setParams]` — same pattern |
| Location | `useLocation()` | `useLocation()` — same API name |
| Route definition | `<Route path="/" element={<Home />}>` | `<Route path="/" component={Home}>` — uses `component` prop, NOT `element` |
| Lazy loading | `React.lazy(() => import(...))` | `lazy(() => import(...))` from `solid-js` |
| Loader data | `useLoaderData()` | `createAsync(() => query(...))` — no loader concept |
| Route loader | `loader` function in route config | `preload` function in route config |

**CRITICAL ANTI-PATTERN:** Using `element={<Component />}` (React pattern) instead of `component={Component}` (Solid pattern). In SolidJS, passing a JSX element via `element` creates it immediately; `component` defers creation to the router.

### 12.2 Next.js vs SolidStart

| Concept | Next.js | SolidStart |
|---------|---------|------------|
| Meta-framework | Next.js (Webpack/Turbopack) | SolidStart (Vite/Vinxi/Nitro) |
| Server data | `getServerSideProps` / Server Components | `"use server"` directive + `query()` |
| API routes | `pages/api/*.ts` or `app/api/*/route.ts` | `routes/api/*.ts` with exported GET/POST/etc. |
| File routing | `pages/` or `app/` directory | `routes/` directory |
| Dynamic routes | `[param].tsx` | `[param].tsx` — same convention |
| Catch-all | `[...slug].tsx` | `[...slug].tsx` — same convention |
| Layout | `layout.tsx` (app dir) | File matching folder name (e.g., `blog.tsx` + `blog/`) |
| Server actions | `"use server"` in Server Components | `"use server"` inside `action()` functions |
| Data mutation | Server Actions with `revalidatePath`/`revalidateTag` | `action()` with automatic query revalidation |
| Streaming | React Server Components + Suspense | `renderToStream` + Suspense |

**CRITICAL ANTI-PATTERN:** Using `getServerSideProps`-style patterns. SolidStart has NO equivalent — use `"use server"` inside `query()` functions instead.

### 12.3 Specific Anti-Pattern: useRouter

```tsx
// WRONG — React pattern
import { useRouter } from "next/router";  // Does not exist in SolidJS
const router = useRouter();
router.push("/dashboard");

// CORRECT — SolidJS pattern
import { useNavigate } from "@solidjs/router";
const navigate = useNavigate();
navigate("/dashboard");
```

### 12.4 Specific Anti-Pattern: Data Fetching in useEffect

```tsx
// WRONG — React pattern
import { useEffect, useState } from "react";
function UserPage() {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetch(`/api/users/${id}`).then(r => r.json()).then(setUser);
  }, [id]);
  return <div>{user?.name}</div>;
}

// CORRECT — SolidJS pattern
import { createAsync } from "@solidjs/router";
import { query } from "@solidjs/router";

const getUser = query(async (id: string) => {
  "use server";
  return fetch(`/api/users/${id}`).then(r => r.json());
}, "user");

function UserPage() {
  const params = useParams();
  const user = createAsync(() => getUser(params.id));
  return <div>{user()?.name}</div>;
}
```

### 12.5 Specific Anti-Pattern: Form Handling

```tsx
// WRONG — React pattern
function TodoForm() {
  const [value, setValue] = useState("");
  const handleSubmit = async (e) => {
    e.preventDefault();
    await fetch("/api/todos", { method: "POST", body: JSON.stringify({ title: value }) });
  };
  return <form onSubmit={handleSubmit}>...</form>;
}

// CORRECT — SolidJS pattern
const addTodo = action(async (formData: FormData) => {
  "use server";
  await db.addTodo(formData.get("title") as string);
}, "addTodo");

function TodoForm() {
  return (
    <form action={addTodo} method="post">
      <input name="title" />
      <button>Add</button>
    </form>
  );
}
```

The SolidJS form pattern works with AND without JavaScript (progressive enhancement).

---

## Appendix: URL Verification Log

| URL | Status | Notes |
|-----|--------|-------|
| https://docs.solidjs.com/solid-start | OK | SolidStart overview |
| https://docs.solidjs.com/solid-start/getting-started | OK | Project setup |
| https://docs.solidjs.com/solid-start/building-your-application/routing | OK | File-based routing |
| https://docs.solidjs.com/solid-start/building-your-application/data-loading | 404 | URL may have changed |
| https://docs.solidjs.com/solid-start/building-your-application/mutations | 404 | URL may have changed |
| https://docs.solidjs.com/solid-start/reference/server/use-server | OK | "use server" directive |
| https://docs.solidjs.com/solid-router | OK | Router overview |
| https://docs.solidjs.com/solid-router/reference/components/a | OK | A component |
| https://docs.solidjs.com/solid-router/reference/data-apis/cache | OK | cache (deprecated) |
| https://docs.solidjs.com/solid-router/reference/data-apis/query | OK | query API |
| https://docs.solidjs.com/solid-router/reference/data-apis/action | OK | action API |
| https://docs.solidjs.com/solid-router/reference/data-apis/create-async | OK | createAsync |
| https://docs.solidjs.com/solid-router/reference/data-apis/create-async-store | OK | createAsyncStore |
| https://docs.solidjs.com/solid-router/reference/primitives/use-navigate | OK | useNavigate |
| https://docs.solidjs.com/solid-router/reference/preload-functions/preload | OK | Route preloading |
| https://docs.solidjs.com/concepts/context | OK | Context API |
| https://docs.solidjs.com/guides/fetching-data | OK | createResource |
| https://docs.solidjs.com/reference/rendering/render-to-string | OK | renderToString |
| https://docs.solidjs.com/reference/rendering/render-to-stream | OK | renderToStream |
| https://docs.solidjs.com/reference/rendering/hydrate | OK | hydrate |
| https://docs.solidjs.com/reference/rendering/hydration-script | OK | HydrationScript |
| https://github.com/solidjs-community/solid-primitives | OK | Community primitives |
| https://github.com/solidjs/solid | OK | Main repo |
| https://docs.solidjs.com/solid-start/building-your-application/api-routes | OK | API routes |
| https://www.solidjs.com/blog | Empty | Blog page returned no content |
| https://docs.solidjs.com/concepts/ssr | 404 | URL may have changed |
