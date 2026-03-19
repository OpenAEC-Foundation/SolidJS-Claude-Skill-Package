# SolidJS Reactivity Research

> Researched: 2026-03-19
> Sources: Official SolidJS documentation (docs.solidjs.com), SolidJS GitHub releases

---

## 1. Fine-Grained Reactivity Model

SolidJS implements **fine-grained reactivity** — there is NO virtual DOM. Instead, SolidJS maintains a **reactive dependency graph** where changes propagate automatically through the system. When a signal changes, only the specific DOM nodes or computations that depend on that signal update. There is no diffing, no reconciliation pass, no component re-rendering.

### How Tracking Works

SolidJS uses automatic dependency tracking within **tracking scopes**. When a reactive primitive (signal, memo) is read inside a tracking scope (effect, memo, JSX expression), it registers as a dependency. When the signal updates, only subscribed computations re-execute.

**Tracking scopes include:**
- `createEffect` callbacks
- `createMemo` callbacks
- `createRenderEffect` callbacks
- `createComputed` callbacks
- JSX expressions (e.g., `{count()}` in templates)

**NOT tracking scopes:**
- Top-level component function body (runs once, never re-runs)
- Event handlers (run on demand, not reactively tracked)
- `onMount` callbacks (explicitly non-tracking)
- Code inside `untrack()`

```typescript
const [count, setCount] = createSignal(0);

// WRONG: Not in a tracking scope — runs once, never updates
console.log(count()); // prints 0, never re-runs

// CORRECT: Inside a tracking scope — re-runs when count changes
createEffect(() => {
  console.log(count()); // re-runs every time count changes
});
```

### Synchronous Reactivity (1.x)

SolidJS 1.x defaults to **synchronous reactivity**: when a signal changes, dependent subscribers update immediately in order. This ensures derived values always reflect current state without scheduling delays.

### Key Difference from React

In React, the **component function** re-runs on every state change (re-render). In SolidJS, the component function runs **exactly once** to set up the reactive graph. Only the specific reactive expressions (effects, memos, JSX bindings) re-execute when their dependencies change.

> Source: https://docs.solidjs.com/concepts/intro-to-reactivity

---

## 2. Signals (createSignal)

Signals are the foundational reactive primitive in SolidJS. They store a value and notify subscribers when it changes.

### API Signature

```typescript
import { createSignal } from "solid-js";

// Overloads
function createSignal<T>(): Signal<T | undefined>;
function createSignal<T>(value: T, options?: SignalOptions<T>): Signal<T>;

// Return type
type Signal<T> = [get: Accessor<T>, set: Setter<T>];
type Accessor<T> = () => T;

type Setter<T> = {
  <U extends T>(value: Exclude<U, Function> | ((prev: T) => U)): U;
  <U extends T>(value: (prev: T) => U): U;
  <U extends T>(value: Exclude<U, Function>): U;
};

// Options
interface SignalOptions<T> {
  name?: string;                                    // Debug label (removed in production)
  equals?: false | ((prev: T, next: T) => boolean); // Custom equality check, default: ===
  internal?: boolean;                                // Hide from devtools
}
```

### Usage

```typescript
const [count, setCount] = createSignal(0);

// Read (getter is a function call!)
console.log(count()); // 0

// Write — direct value
setCount(5);

// Write — functional update (receives previous value)
setCount((prev) => prev + 1);
```

### Equality Checking

By default, signals use reference equality (`===`). If the new value equals the old value, no updates propagate.

```typescript
// Custom equality — useful for objects
const [date, setDate] = createSignal(new Date(), {
  equals: (prev, next) => prev.getTime() === next.getTime()
});

// Always update, even if value is the same
const [value, setValue] = createSignal(0, { equals: false });
```

### WRONG (React) vs CORRECT (SolidJS)

```typescript
// WRONG — React pattern: destructuring kills reactivity
const { count } = props;        // ❌ Reads once, loses tracking
return <div>{count}</div>;      // ❌ Never updates

// CORRECT — SolidJS: access props reactively
return <div>{props.count}</div>; // ✅ Tracked, updates when props.count changes

// WRONG — React pattern: useState returns value, not getter
const [count, setCount] = useState(0);
return <div>{count}</div>;      // React re-renders entire component

// CORRECT — SolidJS: createSignal returns getter function
const [count, setCount] = createSignal(0);
return <div>{count()}</div>;    // ✅ Only this text node updates
```

> Source: https://docs.solidjs.com/concepts/signals, https://docs.solidjs.com/reference/basic-reactivity/create-signal

---

## 3. Effects (createEffect, createRenderEffect, createComputed)

Effects are reactive computations that run side effects when their dependencies change.

### createEffect

```typescript
import { createEffect } from "solid-js";

function createEffect<Next>(
  fn: EffectFunction<undefined | NoInfer<Next>, Next>
): void;

function createEffect<Next, Init = Next>(
  fn: EffectFunction<Init | Next, Next>,
  value: Init,
  options?: { name?: string }
): void;
```

**Execution timing:**
- Initial run: scheduled AFTER the current rendering phase completes, AFTER DOM elements are created, but BEFORE browser paint
- Refs ARE set before first run
- Subsequent runs: whenever tracked dependencies change
- Multiple dependency changes in same batch → effect runs ONCE per batch
- Order among multiple effects is NOT guaranteed
- ALWAYS runs after all pure computations (memos) in same update cycle
- NEVER runs during SSR
- Does NOT run during initial client hydration

```typescript
const [count, setCount] = createSignal(0);

createEffect(() => {
  console.log("Count changed:", count()); // Tracks count automatically
});
```

**With previous value:**

```typescript
createEffect((prev) => {
  const current = count();
  console.log("Changed from", prev, "to", current);
  return current; // Becomes 'prev' on next run
}, 0); // Initial value for prev
```

### createRenderEffect

```typescript
import { createRenderEffect } from "solid-js";

function createRenderEffect<Next>(
  fn: EffectFunction<undefined | NoInfer<Next>, Next>
): void;

function createRenderEffect<Next, Init = Next>(
  fn: EffectFunction<Init | Next, Next>,
  value: Init,
  options?: { name?: string }
): void;
```

**Key differences from createEffect:**

| Aspect | createRenderEffect | createEffect |
|--------|-------------------|--------------|
| Timing | Runs synchronously during render phase | Scheduled after render completes |
| Initial run | Executes before DOM mounting | Executes after initial render |
| Refs | NOT set during initial run | Available when effect runs |
| Re-runs | After all memos complete in update cycle | After render + memos |
| SSR | Runs once during SSR's synchronous phase | Never runs during SSR |

**Execution order demonstration:**

```typescript
function Counter() {
  const [count, setCount] = createSignal(0);

  console.log("Hello from counter");          // 1st

  createEffect(() => console.log("Effect:", count()));         // 4th, 6th
  createRenderEffect(() => console.log("Render effect:", count())); // 2nd, 3rd, 5th

  setCount(1);
  queueMicrotask(() => setCount(2));
}
// Output order:
// "Hello from counter"
// "Render effect: 0"
// "Render effect: 1"
// "Effect: 1"
// "Render effect: 2"
// "Effect: 2"
```

### createComputed

```typescript
import { createComputed } from "solid-js";

function createComputed<Next>(
  fn: EffectFunction<undefined | NoInfer<Next>, Next>
): void;

function createComputed<Next, Init = Next>(
  fn: EffectFunction<Init | Next, Next>,
  value: Init,
  options?: { name?: string }
): void;
```

Runs **before** the rendering phase. Used for synchronizing state to prevent double-render cycles.

| Feature | createComputed | createEffect | createMemo |
|---------|----------------|--------------|------------|
| Timing | Before rendering | After rendering | On access (lazy) |
| Purpose | State synchronization | Side effects | Derived values |
| Returns | void | void | Computed value |

```typescript
function UserEditor(props) {
  const [formData, setFormData] = createStore({ name: "" });

  // Synchronously updates store BEFORE render
  createComputed(() => {
    setFormData("name", props.user.name);
  });

  return <form><h1>Editing: {formData.name}</h1></form>;
}
```

### Nested Effects

Inner effects track independently from outer effects. Signals accessed only within inner effects do NOT become outer effect dependencies.

```typescript
createEffect(() => {
  console.log("Outer effect starts");
  createEffect(() => console.log(count())); // Only this runs when count changes
  console.log("Outer effect ends");
});
```

### Cleanup in Effects

Use `onCleanup` inside effects to clean up resources when the effect re-runs or the component unmounts.

```typescript
createEffect(() => {
  const handler = () => console.log(count());
  window.addEventListener("resize", handler);
  onCleanup(() => window.removeEventListener("resize", handler));
});
```

### WRONG (React) vs CORRECT (SolidJS)

```typescript
// WRONG — React useEffect mental model: dependency array
useEffect(() => {
  console.log(count);
}, [count]); // Manual dependency list

// CORRECT — SolidJS: automatic tracking, no dependency array
createEffect(() => {
  console.log(count()); // Dependencies tracked automatically
});

// WRONG — React pattern: writing state in effects causes re-renders
useEffect(() => {
  setDouble(count * 2); // Triggers re-render
}, [count]);

// CORRECT — SolidJS: use createMemo for derived values instead
const double = createMemo(() => count() * 2); // No effect needed

// WRONG — Expecting effects to run during SSR
createEffect(() => {
  // This NEVER runs on server — use isServer check or different approach
});
```

> Source: https://docs.solidjs.com/concepts/effects, https://docs.solidjs.com/reference/basic-reactivity/create-effect, https://docs.solidjs.com/reference/secondary-primitives/create-render-effect, https://docs.solidjs.com/reference/secondary-primitives/create-computed

---

## 4. Memos (createMemo)

Memos are derived reactive computations that cache their results and only recalculate when dependencies change. Unlike effects, memos ARE reactive sources — they can be tracked by other computations.

### API Signature

```typescript
import { createMemo } from "solid-js";

function createMemo<T>(
  fn: (v: T) => T,
  value?: T,
  options?: {
    equals?: false | ((prev: T, next: T) => boolean);
    name?: string;
  }
): () => T;
```

**Parameters:**
- `fn: (v: T) => T` — Calculation function. Receives previous return value. Should be pure (no side effects).
- `value?: T` — Optional initial value passed to `fn` on first execution.
- `options.equals` — Custom equality check. Default: `===`. Set to `false` to always propagate.
- `options.name` — Debug identifier for devtools.

**Returns:** A read-only accessor function `() => T`.

### When Memos Recalculate

- Only when tracked dependencies change
- Result is cached and reused on subsequent reads
- If new result equals previous (per `equals` logic), downstream dependents are NOT notified

### Examples

```typescript
// Basic derived value
const double = createMemo(() => count() * 2);
console.log(double()); // Cached, only recalculates when count() changes

// Filtering with memoization
const filteredNames = createMemo(() => {
  return NAMES.filter(name =>
    name.toLowerCase().includes(query().toLowerCase())
  );
});

// Custom equality (Date comparison)
const dateObject = createMemo(
  () => new Date(dateString()),
  undefined,
  {
    equals: (prev, next) => prev.getTime() === next.getTime()
  }
);

// Accessing previous value
const trend = createMemo((prev) => {
  const current = count();
  return {
    value: current,
    label: current > prev.value ? "Up" : "Down"
  };
}, { value: 0, label: "Same" });
```

### WRONG (React) vs CORRECT (SolidJS)

```typescript
// WRONG — React useMemo: dependency array, recalculates on re-render
const double = useMemo(() => count * 2, [count]);

// CORRECT — SolidJS createMemo: automatic tracking, cached reactive source
const double = createMemo(() => count() * 2);

// WRONG — Using createEffect where createMemo should be used
createEffect(() => {
  setDouble(count() * 2); // ❌ Unnecessary effect + signal
});

// CORRECT — Use createMemo for derived values
const double = createMemo(() => count() * 2); // ✅ Cached, reactive

// WRONG — React: useMemo is not a reactive source
const value = useMemo(() => expensive(), [dep]); // Can't be tracked

// CORRECT — SolidJS: createMemo IS a reactive source
const value = createMemo(() => expensive());
createEffect(() => console.log(value())); // ✅ Tracks value changes
```

> Source: https://docs.solidjs.com/reference/basic-reactivity/create-memo

---

## 5. Resources (createResource)

Resources handle asynchronous data fetching with built-in loading states, error handling, and Suspense integration.

### API Signature

```typescript
import { createResource } from "solid-js";

// Without source
function createResource<T, R = unknown>(
  fetcher: ResourceFetcher<true, T, R>,
  options?: ResourceOptions<T>
): ResourceReturn<T, R>;

// With source (reactive trigger)
function createResource<T, S, R = unknown>(
  source: ResourceSource<S>,
  fetcher: ResourceFetcher<S, T, R>,
  options?: ResourceOptions<T, S>
): ResourceReturn<T, R>;
```

### Type Definitions

```typescript
type ResourceReturn<T, R = unknown> = [Resource<T>, ResourceActions<T, R>];

type Resource<T> = {
  (): T | undefined;                    // Accessor for current value
  state: "unresolved" | "pending" | "ready" | "refreshing" | "errored";
  loading: boolean;
  error: any;
  latest: T | undefined;               // Most recent successfully resolved value
};

type ResourceActions<T, R = unknown> = {
  mutate: (value: T | undefined) => T | undefined;  // Update without fetching
  refetch: (info?: R) => Promise<T> | T | undefined; // Re-trigger fetcher
};

type ResourceSource<S> =
  | S
  | false
  | null
  | undefined
  | (() => S | false | null | undefined);

type ResourceFetcher<S, T, R = unknown> = (
  source: S,
  info: { value: T | undefined; refetching: R | boolean }
) => T | Promise<T>;

interface ResourceOptions<T, S = unknown> {
  initialValue?: T;
  name?: string;
  deferStream?: boolean;
  ssrLoadFrom?: "initial" | "server";
  storage?: (init: T | undefined) => [Accessor<T | undefined>, Setter<T | undefined>];
  onHydrated?: (k: S | undefined, info: { value: T | undefined }) => void;
}
```

### Resource State Machine

| State | loading | error | latest | Description |
|-------|---------|-------|--------|-------------|
| `unresolved` | false | undefined | undefined | Initial state before first fetch |
| `pending` | true | undefined | undefined | First fetch in progress |
| `ready` | false | undefined | T | Successfully resolved |
| `refreshing` | true | undefined | T | Re-fetching with prior value retained |
| `errored` | false | any | undefined | Fetch failed |

### Examples

```typescript
// Basic: no source
const [data] = createResource(async () => {
  const response = await fetch("/api/data");
  return response.json();
});

// With reactive source — automatically refetches when source changes
const [userId, setUserId] = createSignal(1);
const [user] = createResource(userId, async (id) => {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
});
setUserId(2); // Triggers automatic refetch

// With actions
const [posts, { refetch, mutate }] = createResource(fetchPosts);
await refetch();                        // Manual refetch
mutate(prev => [...prev, newPost]);     // Optimistic update

// With initial value (resource is never undefined)
const [user] = createResource(() => fetchUser(), {
  initialValue: { name: "Loading...", id: 0 }
});

// False/null/undefined source prevents fetching
const [id, setId] = createSignal<number | false>(false);
const [data] = createResource(id, fetchById); // Does NOT fetch until id is truthy
setId(1); // NOW it fetches
```

### Suspense Integration

```tsx
<Suspense fallback={<div>Loading...</div>}>
  <UserProfile /> {/* Component using createResource */}
</Suspense>

<ErrorBoundary fallback={<div>Error loading data</div>}>
  <div>{data()?.title}</div>
</ErrorBoundary>
```

### WRONG (React) vs CORRECT (SolidJS)

```typescript
// WRONG — React pattern: manual loading/error state
const [data, setData] = useState(null);
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);
useEffect(() => {
  fetch("/api/data")
    .then(res => res.json())
    .then(setData)
    .catch(setError)
    .finally(() => setLoading(false));
}, []);

// CORRECT — SolidJS: createResource handles all of this
const [data] = createResource(async () => {
  const res = await fetch("/api/data");
  return res.json();
});
// data(), data.loading, data.error, data.state — all built-in
```

> Source: https://docs.solidjs.com/reference/basic-reactivity/create-resource

---

## 6. Stores (createStore)

Stores provide proxy-based reactive state management for complex nested data structures with fine-grained tracking at the property level.

### API Signature

```typescript
import { createStore } from "solid-js/store";

function createStore<T extends StoreNode>(
  state: T | Store<T>
): [get: Store<T>, set: SetStoreFunction<T>];

type Store<T> = T; // Reactive proxy — conceptually readonly
```

### Proxy-Based Reactivity

Stores use JavaScript Proxies to extend reactivity to nested objects and arrays. Signals are created **lazily** — only when a property is accessed within a tracking scope. This creates a "dynamic tree of reactive data."

### Accessing Store Values

Store properties are accessed directly (no function call needed, unlike signals):

```typescript
const [store, setStore] = createStore({
  userCount: 3,
  users: [
    { id: 0, username: "felix909", location: "England", loggedIn: false },
    { id: 1, username: "tracy634", location: "Canada", loggedIn: true },
  ]
});

console.log(store.userCount);       // 3 (direct property access)
console.log(store.users[0].username); // "felix909" (nested access)
```

### setStore with Path Syntax

The setter supports a powerful path-based syntax for surgical updates:

```typescript
// Basic property update
setStore("userCount", 5);

// Nested property update
setStore("users", 0, "loggedIn", true);

// Functional update
setStore("users", 0, "loggedIn", (prev) => !prev);

// Append to array via index
setStore("users", store.users.length, { id: 3, username: "michael584", location: "Nigeria", loggedIn: false });

// Update multiple array indices
setStore("users", [2, 7, 10], "loggedIn", false);

// Range updates
setStore("users", { from: 1, to: store.users.length - 1 }, "loggedIn", false);
setStore("users", { from: 0, to: store.users.length - 1, by: 2 }, "loggedIn", false);

// Filter-based updates (function predicate)
setStore("users", (user) => user.username.startsWith("t"), "loggedIn", false);
setStore("users", (user) => user.location === "Canada", "loggedIn", false);
```

### Object Merge Behavior

Object updates perform **shallow merge** — new properties combine with existing ones:

```typescript
setStore("users", 0, { id: 109 });
// Only id changes, username/location/loggedIn preserved

// Function-based merge
setState((state) => ({
  preferredName: state.firstName,
  lastName: "Milner"
}));
```

### Computed Getters in Stores

```typescript
const [state, setState] = createStore({
  user: {
    firstName: "John",
    lastName: "Smith",
    get fullName() {
      return `${this.firstName} ${this.lastName}`;
    }
  }
});
```

### Property Deletion

Remove properties by setting to `undefined`. In TypeScript, use non-null assertion: `undefined!`

### Stores vs Signals

| Aspect | Stores | Signals |
|--------|--------|---------|
| Use case | Complex state, nested objects, arrays | Single values, primitives |
| Reactivity | Fine-grained per property | Entire value |
| Access syntax | Direct property access (`store.x`) | Function call (`signal()`) |
| Updates | Path syntax setter | Direct value or function |
| Proxy overhead | Yes (Proxy wrapper) | No |

### WRONG (React) vs CORRECT (SolidJS)

```typescript
// WRONG — React pattern: spread/replace entire state
setState({ ...state, users: [...state.users, newUser] }); // ❌ Replaces everything

// CORRECT — SolidJS: path syntax for surgical updates
setStore("users", store.users.length, newUser); // ✅ Only adds to array

// WRONG — Destructuring store (kills reactivity)
const { username } = store.users[0]; // ❌ Static snapshot, never updates

// CORRECT — Access in tracking scope
<span>{store.users[0].username}</span> // ✅ Fine-grained tracking
```

> Source: https://docs.solidjs.com/concepts/stores, https://docs.solidjs.com/reference/store-utilities/create-store

---

## 7. Store Utilities (produce, reconcile, unwrap, createMutable)

### produce

Immer-inspired utility for writing mutation-style code that gets applied immutably.

```typescript
import { produce } from "solid-js/store";

function produce<T>(
  fn: (state: T) => void
): (
  state: T extends NotWrappable ? T : Store<T>
) => T extends NotWrappable ? T : Store<T>;
```

**Usage:**

```typescript
const [state, setState] = createStore({
  user: { name: "John", age: 30 },
  list: ["book", "pen"]
});

setState(
  produce((state) => {
    state.user.name = "Jane";
    state.list.push("pencil");
  })
);

// Also works with path syntax
setStore("users", 0, produce((user) => {
  user.username = "newUsername";
  user.location = "newLocation";
}));
```

**Limitations:** Works with arrays and objects only. Incompatible with Sets and Maps.

### reconcile

Diffs new data against existing store, updating only what changed. Useful for API responses and external data synchronization.

```typescript
import { reconcile } from "solid-js/store";

function reconcile<T>(
  value: T | Store<T>,
  options?: {
    key?: string | null;  // Default: "id"
    merge?: boolean;       // Default: false
  }
): (state: T extends NotWrappable ? T : Store<T>) => T extends NotWrappable ? T : Store<T>;
```

**Options:**
- `key` (default: `"id"`) — Property used to match items during reconciliation
- `merge`:
  - `false` — Referential checks determine equality; non-equal items are replaced
  - `true` — Morphs previous data to new value, pushing diffing to leaf-level properties

```typescript
const [data, setData] = createStore({ animals: ["cat", "dog"] });
const newData = ["cat", "dog", "koala"];
setData("animals", reconcile(newData)); // Only "koala" triggers update

// Syncing with subscriptions
const unsubscribe = externalStore.subscribe(({ todos }) =>
  setState("todos", reconcile(todos))
);
onCleanup(() => unsubscribe());
```

### unwrap

Strips the reactive proxy from a store, returning a plain JavaScript object.

```typescript
import { unwrap } from "solid-js/store";

function unwrap<T>(store: Store<T>): T;
```

**Use cases:**
- Passing data to third-party libraries that expect plain objects
- Serialization (JSON.stringify)
- Debugging (inspect raw data without proxy)
- Snapshots outside tracking scopes

```typescript
const [data, setData] = createStore({ animals: ["cat", "dog"] });
const rawData = unwrap(data); // Plain JS object, no proxy
```

### createMutable

Reactive proxy store that allows direct mutations (like MobX/Vue).

```typescript
import { createMutable } from "solid-js/store";

function createMutable<T extends StoreNode>(
  state: T | Store<T>
): Store<T>;
```

```typescript
const state = createMutable({
  firstName: "John",
  lastName: "Smith",
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  },
  set fullName(value) {
    [this.firstName, this.lastName] = value.split(" ");
  },
});

// Direct mutation (unlike createStore which needs setStore)
state.firstName = "Jane";
state.list.push(item);
```

**Warning:** The official docs caution that mutable state "may complicate the code structure and increase the risk of breaking unidirectional flow." Prefer `createStore` for most cases; use `createMutable` only for MobX/Vue migration or wrapping mutable third-party libraries.

> Source: https://docs.solidjs.com/reference/store-utilities/produce, https://docs.solidjs.com/reference/store-utilities/reconcile, https://docs.solidjs.com/reference/store-utilities/unwrap, https://docs.solidjs.com/reference/store-utilities/create-mutable

---

## 8. Reactive Utilities (batch, untrack, on, observable, from)

### batch

Defers downstream computations until the callback completes. All signal updates accumulate and propagate once.

```typescript
import { batch } from "solid-js";

function batch<T>(fn: () => T): T;
```

**Performance impact:**
- Without batching: `m × n` updates (m downstream × n signal changes)
- With batching: `m` updates (once per downstream computation)

```typescript
// Without batching — 3 separate update cycles
setUp1(4);  // triggers downstream
setUp2(5);  // triggers downstream
setUp3(6);  // triggers downstream

// With batching — single update cycle
batch(() => {
  setUp1(10);
  setUp2(10);
  setUp3(10);
}); // All downstream updates fire once here
```

**Edge cases:**
- Accessing a signal/memo inside batch triggers immediate recalculation if needed (since Solid 1.4)
- Async breaks batching: only updates before the first `await` are batched
- Nesting works: multiple `batch()` calls combine into one logical batch
- **Automatic batching:** Solid internally batches updates in `createEffect`, `onMount`, and store setters

```typescript
batch(() => {
  setUp1(11);
  console.log(down()); // Forces immediate recalculation
  setUp2(11);          // Still batched afterward
});
```

### untrack

Prevents dependency tracking for code executed within its scope.

```typescript
import { untrack } from "solid-js";

function untrack<T>(fn: () => T): T;
```

```typescript
createEffect(() => {
  // count() IS tracked, name() is NOT
  console.log(count(), untrack(() => name()));
});

// Use for static prop values
export function Component(props) {
  const value = untrack(() => props.value); // Read once, never track
  return <div>{value}</div>;
}
```

**Naming convention shortcut:** Props named with `initial` or `default` prefixes are automatically untracked when used as signal initial values:

```typescript
const [name, setName] = createSignal(props.initialName);  // No untrack needed
const [name, setName] = createSignal(props.defaultName);   // No untrack needed
```

### on

Helper for explicit dependency specification in effects. Wraps an effect function to only track specified dependencies.

```typescript
import { on } from "solid-js";

function on<S, Next>(
  deps: Accessor<S> | AccessorArray<S>,
  fn: (input: S, prevInput: S, prev: Next) => Next,
  options?: { defer?: boolean }
): EffectFunction<undefined | NoInfer<Next>, Next>;
```

**Note:** The `on` utility page returned 404 at multiple URL patterns on docs.solidjs.com as of the research date. The above signature is from the SolidJS source types. The `defer` option when `true` skips the initial run and only fires on subsequent changes.

```typescript
// Explicit dependency — only tracks `a`, ignores `b` inside fn
createEffect(on(a, (aVal) => {
  console.log(aVal, b()); // b() is read but NOT tracked
}));

// Multiple dependencies
createEffect(on([a, b], ([aVal, bVal]) => {
  console.log(aVal, bVal);
}));

// Deferred — skip initial run
createEffect(on(a, (aVal, prevA) => {
  console.log("Changed from", prevA, "to", aVal);
}, { defer: true }));
```

### observable

Converts a SolidJS signal accessor to a standard Observable for interop with RxJS and similar libraries.

```typescript
import { observable } from "solid-js";

function observable<T>(input: () => T): Observable<T>;
```

```typescript
import { observable } from "solid-js";
import { from } from "rxjs";

const [s, set] = createSignal(0);
const obsv$ = from(observable(s));
obsv$.subscribe((v) => console.log(v));
```

### from

Bridges external reactive systems (RxJS, Svelte stores, custom producers) into SolidJS signals with automatic lifecycle management.

```typescript
import { from } from "solid-js";

function from<T>(
  producer:
    | ((setter: (v: T) => T) => () => void)
    | {
        subscribe: (
          fn: (v: T) => void
        ) => (() => void) | { unsubscribe: () => void };
      }
): () => T | undefined;
```

```typescript
// From an RxJS observable
const signal = from(obsv$);

// Custom producer with cleanup
const clock = from((set) => {
  const interval = setInterval(() => {
    set((v) => v + 1);
  }, 1000);
  return () => clearInterval(interval); // Cleanup function
});
```

> Source: https://docs.solidjs.com/reference/reactive-utilities/batch, https://docs.solidjs.com/reference/reactive-utilities/untrack, https://docs.solidjs.com/reference/reactive-utilities/observable, https://docs.solidjs.com/reference/reactive-utilities/from

---

## 9. Lifecycle (onMount, onCleanup)

### onMount

Runs once after initial rendering completes and DOM elements are mounted. Non-tracking — does NOT create reactive dependencies.

```typescript
import { onMount } from "solid-js";

function onMount(fn: () => void): void;
```

Equivalent to `createEffect` with no reactive dependencies (runs once, never re-runs).

```typescript
function MyComponent() {
  let ref: HTMLButtonElement;

  onMount(() => {
    ref.disabled = true; // DOM is available, refs are set
  });

  return <button ref={ref}>Click me</button>;
}

// Async data fetching on mount
function Component() {
  const [data, setData] = createSignal(null);
  onMount(async () => {
    const fetchedData = await fetch("https://example.com/data");
    setData(fetchedData);
  });
  return <div>...</div>;
}
```

### onCleanup

Registers a cleanup function that runs when the current tracking scope is disposed or recalculated.

```typescript
import { onCleanup } from "solid-js";

function onCleanup(fn: () => void): void;
```

**Runs when:**
- Component unmounts
- Effect recalculates (before the new run)
- Memo refreshes
- `createRoot` scope is disposed

**Multiple onCleanup calls:** Execute in reverse order (LIFO — Last In, First Out).

```typescript
function App() {
  const [count, setCount] = createSignal(0);
  const timer = setInterval(() => setCount((prev) => prev + 1), 1000);

  onCleanup(() => clearInterval(timer)); // Runs on unmount

  return <div>Count: {count()}</div>;
}

// Inside effects — cleanup runs before each re-execution
createEffect(() => {
  const handler = () => console.log(count());
  document.addEventListener("click", handler);
  onCleanup(() => document.removeEventListener("click", handler));
});
```

### WRONG (React) vs CORRECT (SolidJS)

```typescript
// WRONG — React useEffect cleanup pattern
useEffect(() => {
  const timer = setInterval(tick, 1000);
  return () => clearInterval(timer); // Return cleanup function
}, []);

// CORRECT — SolidJS: onCleanup is a separate call, not a return value
onMount(() => {
  const timer = setInterval(tick, 1000);
  onCleanup(() => clearInterval(timer)); // Explicit cleanup registration
});

// WRONG — React: useEffect with empty deps for mount
useEffect(() => { /* mount logic */ }, []);

// CORRECT — SolidJS: dedicated onMount
onMount(() => { /* mount logic */ }); // Clear intent, non-tracking
```

> Source: https://docs.solidjs.com/reference/lifecycle/on-mount, https://docs.solidjs.com/reference/lifecycle/on-cleanup

---

## 10. React vs SolidJS Anti-Patterns (Reactivity)

This section catalogs every way React mental models break SolidJS reactivity. These patterns are the PRIMARY reason Claude generates broken SolidJS code.

### Anti-Pattern 1: Destructuring Props (Kills Tracking)

```typescript
// ❌ WRONG — Destructuring reads values once, loses reactive tracking
function Greeting({ name }) {        // Destructured at call time
  return <h1>Hello {name}</h1>;       // Static value, never updates
}

// ✅ CORRECT — Access props object reactively
function Greeting(props) {
  return <h1>Hello {props.name}</h1>; // Tracked, updates when name changes
}

// ✅ ALSO CORRECT — Use splitProps or mergeProps for selective access
import { splitProps } from "solid-js";
function Greeting(props) {
  const [local, others] = splitProps(props, ["name"]);
  return <h1 {...others}>Hello {local.name}</h1>;
}
```

### Anti-Pattern 2: Destructuring Signals (Kills Tracking)

```typescript
// ❌ WRONG — Destructuring reads the signal value once
const [count, setCount] = createSignal(0);
const value = count(); // Snapshot, not reactive
return <div>{value}</div>; // Never updates

// ✅ CORRECT — Call signal getter in JSX
return <div>{count()}</div>; // Tracked, updates reactively
```

### Anti-Pattern 3: useState vs createSignal

```typescript
// React: useState returns a value
const [count, setCount] = useState(0);
return <div>{count}</div>; // Value, triggers re-render

// SolidJS: createSignal returns a GETTER FUNCTION
const [count, setCount] = createSignal(0);
return <div>{count()}</div>; // ← Must call it! Getter function, no re-render
```

### Anti-Pattern 4: useEffect vs createEffect (Different Semantics)

```typescript
// React: dependency array, runs on re-render
useEffect(() => {
  document.title = `Count: ${count}`;
}, [count]); // Manual deps

// SolidJS: automatic tracking, no dependency array
createEffect(() => {
  document.title = `Count: ${count()}`; // Auto-tracked
});

// React: cleanup via return
useEffect(() => {
  const id = setInterval(fn, 1000);
  return () => clearInterval(id);
}, []);

// SolidJS: cleanup via onCleanup (separate call)
createEffect(() => {
  const id = setInterval(fn, 1000);
  onCleanup(() => clearInterval(id));
});
```

### Anti-Pattern 5: useMemo vs createMemo

```typescript
// React: useMemo is NOT a reactive source, recalculates on re-render
const double = useMemo(() => count * 2, [count]); // Manual deps

// SolidJS: createMemo IS a reactive source, auto-tracked
const double = createMemo(() => count() * 2); // No deps needed
```

### Anti-Pattern 6: Component Re-Render Mental Model

```typescript
// React mental model: component re-runs on every state change
function Counter() {
  const [count, setCount] = useState(0);
  console.log("render"); // Logs on EVERY state change
  return <div>{count}</div>;
}

// SolidJS: component function runs ONCE
function Counter() {
  const [count, setCount] = createSignal(0);
  console.log("setup"); // Logs ONCE, never again
  return <div>{count()}</div>; // Only this expression updates
}
```

### Anti-Pattern 7: Conditional Signal Access (Breaks Tracking)

```typescript
// ❌ WRONG — Conditional access means dependency is not always tracked
createEffect(() => {
  if (someCondition) {
    console.log(count()); // Only tracked when condition is true!
  }
});

// ✅ CORRECT — Access signal unconditionally, use value conditionally
createEffect(() => {
  const c = count(); // Always tracked
  if (someCondition) {
    console.log(c);
  }
});
```

### Anti-Pattern 8: Early Return Before Signal Access

```typescript
// ❌ WRONG — Early return prevents tracking of signals after it
createEffect(() => {
  if (loading()) return; // If true, name() is never tracked
  console.log(name());
});

// ✅ CORRECT — Access all signals first
createEffect(() => {
  const isLoading = loading();
  const currentName = name(); // Always tracked
  if (isLoading) return;
  console.log(currentName);
});
```

### Anti-Pattern 9: Storing Signals in Variables (Snapshot Problem)

```typescript
// ❌ WRONG — Storing getter result breaks reactivity
const currentCount = count(); // Snapshot at this moment
setTimeout(() => {
  console.log(currentCount); // Stale value!
}, 1000);

// ✅ CORRECT — Call getter when you need the value
setTimeout(() => {
  console.log(count()); // Current value at time of execution
}, 1000);
```

### Anti-Pattern 10: Spreading Props into Child Components

```typescript
// ❌ WRONG — Spreading reactive props breaks tracking for some patterns
function Parent(props) {
  return <Child {...props} />; // May lose reactivity for dynamic props
}

// ✅ CORRECT — Use splitProps and mergeProps
import { splitProps, mergeProps } from "solid-js";
function Parent(props) {
  const [local, others] = splitProps(props, ["class"]);
  const merged = mergeProps({ class: "default" }, others);
  return <Child {...merged} />;
}
```

---

## 11. SolidJS 2.x Changes

Based on the SolidJS GitHub releases, SolidJS 2.0 is in **beta** (v2.0.0-beta.0, released March 2023). The latest stable 1.x release is **v1.9.0** (September 2023).

### Major Reactivity Changes in 2.x

| Feature | SolidJS 1.x | SolidJS 2.x |
|---------|-------------|-------------|
| **Reactivity model** | Synchronous by default | Microtask-batched — reads don't update until batch flushes |
| **Async** | Manual with createResource | First-class — computations can return Promises/async iterables |
| **Effects** | Single `createEffect` | Split compute → apply pattern |
| **Lifecycle** | `onMount` | `onSettled` (can return cleanup) |
| **Store setters** | Path-style ergonomics | Draft-first by default |
| **List rendering** | `<Index>` component | `<For keyed={false}>` with accessors |
| **DOM directives** | `use:` directives | `ref` directive factories + arrays |
| **Derived state** | createMemo only | `createSignal(fn)` and `createStore(fn)` for derived-but-writable patterns |

### Key 2.x Breaking Changes

1. **Microtask batching**: All signal updates are batched by default. Reads don't reflect updates until the batch flushes. Use `flush()` when immediate propagation is needed.

2. **Async as first-class**: Computations can return Promises or async iterables. The reactivity graph handles suspension and resumption automatically.

3. **`onMount` → `onSettled`**: The lifecycle hook is renamed and enhanced — `onSettled` can return a cleanup function.

4. **Dev guardrails**: SolidJS 2.x adds warnings for accidental top-level reads and reactive scope writes, catching common mistakes at development time.

### New 2.x Primitives

- **`<Loading>`** — Shows fallback during initial render, maintains UI stability during background work
- **`isPending()`** — Tracks "refreshing" work without tearing down UI
- **`action()`** — Dedicated mutation primitive with optimistic patterns
- **`createOptimistic` / `createOptimisticStore`** — Explicit optimistic UI handling

### Key 1.x Evolution (1.4–1.9)

- Better TypeScript support
- Top-level arrays in stores
- Streaming HTML support
- Improved hydration mechanisms for SSR
- Batch behavior change: accessing signals/memos inside `batch()` triggers immediate recalculation if needed (since 1.4)

> Source: https://github.com/solidjs/solid/releases

---

## Appendix: Complete Import Map

```typescript
// Core reactivity
import {
  createSignal,
  createEffect,
  createMemo,
  createResource,
  createComputed,
  createRenderEffect,
} from "solid-js";

// Reactive utilities
import {
  batch,
  untrack,
  on,
  observable,
  from,
} from "solid-js";

// Lifecycle
import {
  onMount,
  onCleanup,
} from "solid-js";

// Store utilities (separate entry point!)
import {
  createStore,
  createMutable,
  produce,
  reconcile,
  unwrap,
} from "solid-js/store";

// Component utilities
import {
  splitProps,
  mergeProps,
} from "solid-js";
```
