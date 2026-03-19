# SolidJS Vooronderzoek (Preliminary Research)

> Compiled from research fragments. Last updated: 2026-03-19
> Sources: docs.solidjs.com, github.com/solidjs/solid, github.com/solidjs-community/solid-primitives

---

## 1. SolidJS Architecture & Reactivity Model
<!-- Lines: ~15-50 -->

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
<!-- Lines: ~60-135 -->

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
const { count } = props;        // Reads once, loses tracking
return <div>{count}</div>;      // Never updates

// CORRECT — SolidJS: access props reactively
return <div>{props.count}</div>; // Tracked, updates when props.count changes

// WRONG — React pattern: useState returns value, not getter
const [count, setCount] = useState(0);
return <div>{count}</div>;      // React re-renders entire component

// CORRECT — SolidJS: createSignal returns getter function
const [count, setCount] = createSignal(0);
return <div>{count()}</div>;    // Only this text node updates
```

> Source: https://docs.solidjs.com/concepts/signals, https://docs.solidjs.com/reference/basic-reactivity/create-signal

---

## 3. Derived Values (createMemo)
<!-- Lines: ~140-220 -->

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
  setDouble(count() * 2); // Unnecessary effect + signal
});

// CORRECT — Use createMemo for derived values
const double = createMemo(() => count() * 2); // Cached, reactive

// WRONG — React: useMemo is not a reactive source
const value = useMemo(() => expensive(), [dep]); // Can't be tracked

// CORRECT — SolidJS: createMemo IS a reactive source
const value = createMemo(() => expensive());
createEffect(() => console.log(value())); // Tracks value changes
```

> Source: https://docs.solidjs.com/reference/basic-reactivity/create-memo

---

## 4. Side Effects (createEffect, createRenderEffect, createComputed)
<!-- Lines: ~225-325 -->

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

## 5. Resources (createResource)
<!-- Lines: ~330-465 -->

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
<!-- Lines: ~470-590 -->

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
setState({ ...state, users: [...state.users, newUser] }); // Replaces everything

// CORRECT — SolidJS: path syntax for surgical updates
setStore("users", store.users.length, newUser); // Only adds to array

// WRONG — Destructuring store (kills reactivity)
const { username } = store.users[0]; // Static snapshot, never updates

// CORRECT — Access in tracking scope
<span>{store.users[0].username}</span> // Fine-grained tracking
```

> Source: https://docs.solidjs.com/concepts/stores, https://docs.solidjs.com/reference/store-utilities/create-store

---

## 7. Store Utilities (produce, reconcile, unwrap, createMutable)
<!-- Lines: ~595-720 -->

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
<!-- Lines: ~725-880 -->

### batch

Defers downstream computations until the callback completes. All signal updates accumulate and propagate once.

```typescript
import { batch } from "solid-js";

function batch<T>(fn: () => T): T;
```

**Performance impact:**
- Without batching: `m x n` updates (m downstream x n signal changes)
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
<!-- Lines: ~885-980 -->

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

## 10. JSX Compilation Model
<!-- Lines: ~985-1060 -->

**Source**: https://docs.solidjs.com/concepts/understanding-jsx

SolidJS compiles JSX to **direct DOM creation calls**, NOT a virtual DOM. This is the fundamental architectural difference from React.

### How It Works

- JSX compiles to `document.createElement` and direct DOM manipulation calls
- Components execute **once** during initial rendering — they are NOT re-called when state changes
- Dynamic expressions in `{}` are automatically wrapped in reactive effects
- The compiler distinguishes between static and dynamic content, optimizing static parts

### Compilation Output

```typescript
// JSX input:
const Component = () => {
  const [count, setCount] = createSignal(0);
  return <div>{count()}</div>;
};

// Conceptual compilation output:
// 1. Template is created once (static parts)
// 2. Dynamic expression count() is wrapped in an effect
// 3. Only the text node containing count() updates when the signal changes
```

### Attribute Handling

SolidJS evaluates attributes in **declaration order** — later definitions override earlier ones:

```typescript
<input type="text" {...defaultProps} value={currentValue} />
// value={currentValue} overrides any value in defaultProps
```

### Special Attribute Prefixes (Namespaced Attributes)

| Prefix | Purpose | Example |
|--------|---------|---------|
| `attr:` | Force HTML attribute (vs property) | `<div attr:data-id={id} />` |
| `prop:` | Set DOM property directly | `<input prop:value={val} />` |
| `class:` | Toggle individual class | `<div class:active={isActive()} />` |
| `on:` | Native event listener (non-delegated) | `<div on:scroll={handler} />` |
| `use:` | Custom directive | `<div use:myDirective={value} />` |
| `bool:` | Boolean attribute | `<div bool:disabled={isDisabled()} />` |
| `@once` | One-time binding (no reactivity) | `<div textContent={@once val} />` |

### Style Object Syntax

```typescript
// Double curly braces — outer for JSX expression, inner for object:
<button style={{ color: "red", "font-size": "2rem" }}>Click</button>
```

### innerHTML / textContent

Can be set directly as props on elements. Use with caution — `innerHTML` creates XSS risk.

---

## 11. Component Model
<!-- Lines: ~1065-1145 -->

**Source**: https://docs.solidjs.com/concepts/components/basics

### Core Principle: Components Run ONCE

```typescript
function MyComponent() {
  // This entire function body executes exactly ONCE
  const [count, setCount] = createSignal(0);
  console.log("I run once, not on every update"); // Logs once

  return (
    <div>
      <p>Count: {count()}</p> {/* This expression is reactive */}
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}
```

### Two Phases

1. **Initialization** (runs once): Sets up signals, stores, effects, computations
2. **Reactive updates** (ongoing): Fine-grained DOM updates via the reactive graph — NO component re-execution

### Component Function Return Values

Components return `JSX.Element` — which in SolidJS is actual DOM nodes, not virtual DOM descriptors.

### TypeScript Component Types

```typescript
import type { Component, ParentComponent, VoidComponent, FlowComponent } from "solid-js";

// Basic component (may or may not have children)
const MyComp: Component<{ name: string }> = (props) => {
  return <div>{props.name}</div>;
};

// Component that accepts children
const Layout: ParentComponent<{ title: string }> = (props) => {
  return <div><h1>{props.title}</h1>{props.children}</div>;
};

// Component that NEVER has children
const Icon: VoidComponent<{ name: string }> = (props) => {
  return <i class={`icon-${props.name}`} />;
};

// Component with specific children type (control flow)
const MyShow: FlowComponent<{ when: boolean }, string> = (props) => {
  // children type is narrowed
  return <div>{props.children}</div>;
};
```

### Import Patterns

```typescript
// Core reactivity
import { createSignal, createEffect, createMemo } from "solid-js";

// Stores
import { createStore, produce } from "solid-js/store";

// Web-specific (Portal, Dynamic, etc.)
import { Portal, Dynamic } from "solid-js/web";

// Control flow components
import { Show, For, Index, Switch, Match, Suspense, ErrorBoundary } from "solid-js";
```

---

## 12. Props (splitProps, mergeProps, default props)
<!-- Lines: ~1150-1275 -->

**Source**: https://docs.solidjs.com/concepts/components/props

### CRITICAL: Why Destructuring Kills Reactivity

Props in SolidJS are **reactive getters on a proxy object**. Destructuring extracts the current value and severs the reactive connection:

```typescript
// WRONG — breaks reactivity:
function Greeting({ name }: { name: string }) {
  return <h1>Hello {name}</h1>; // name is frozen at initial value
}

// WRONG — also breaks reactivity:
function Greeting(props: { name: string }) {
  const { name } = props; // Extracted once, never updates
  return <h1>Hello {name}</h1>;
}

// WRONG — same problem:
function Greeting(props: { name: string }) {
  const name = props.name; // Captured once
  return <h1>Hello {name}</h1>;
}

// CORRECT — access props directly:
function Greeting(props: { name: string }) {
  return <h1>Hello {props.name}</h1>; // Reactive — reads on every access
}

// CORRECT — wrap in a function if needed outside JSX:
function Greeting(props: { name: string }) {
  const name = () => props.name; // Creates a derived accessor
  return <h1>Hello {name()}</h1>;
}
```

### splitProps — Reactive Destructuring Alternative

**Import**: `import { splitProps } from "solid-js";`

```typescript
function splitProps<T extends object, K extends (keyof T)[]>(
  props: T,
  ...keys: K[]
): [...SplitProps<T, K>, Omit<T, K[number]>];
```

Usage:

```typescript
import { splitProps } from "solid-js";

interface ButtonProps {
  variant: string;
  disabled: boolean;
  onClick: () => void;
  class: string;
  id: string;
}

function Button(props: ButtonProps) {
  // Split into groups while PRESERVING reactivity
  const [buttonProps, styleProps, rest] = splitProps(
    props,
    ["onClick", "disabled"],  // Group 1: button behavior
    ["variant", "class"]       // Group 2: styling
    // rest: everything else (id, etc.)
  );

  return (
    <button
      onClick={buttonProps.onClick}
      disabled={buttonProps.disabled}
      class={`btn-${styleProps.variant} ${styleProps.class}`}
      {...rest}
    >
      {props.children}
    </button>
  );
}
```

### mergeProps — Default Props Pattern

**Import**: `import { mergeProps } from "solid-js";`

```typescript
function mergeProps<T extends object[]>(...sources: T): MergedProps<T>;
```

Usage — setting default prop values:

```typescript
import { mergeProps } from "solid-js";

interface ButtonProps {
  variant?: string;
  size?: string;
  disabled?: boolean;
}

function Button(props: ButtonProps) {
  // Merge defaults with incoming props (later values override)
  const merged = mergeProps(
    { variant: "primary", size: "md", disabled: false }, // defaults
    props // actual props override defaults
  );

  return (
    <button
      class={`btn-${merged.variant} btn-${merged.size}`}
      disabled={merged.disabled}
    >
      {props.children}
    </button>
  );
}
```

**Key behavior**: `mergeProps` maintains reactivity. Properties from later objects override earlier ones. This is the SolidJS equivalent of React's `defaultProps`.

### Spreading Props

```typescript
function Parent(props: ParentProps) {
  const [local, rest] = splitProps(props, ["onSpecialEvent"]);

  // Spread remaining props to child — reactivity preserved
  return <Child {...rest} />;
}
```

---

## 13. Children
<!-- Lines: ~1280-1360 -->

**Source**: https://docs.solidjs.com/reference/component-apis/children

### Why props.children Is a Getter

In SolidJS, `props.children` is NOT a value — it's an accessor (getter function) that returns unresolved JSX. Accessing it multiple times can cause children to be **re-created** each time.

### The children() Helper

```typescript
import { children } from "solid-js";

function children(fn: Accessor<JSX.Element>): ChildrenReturn;

type ChildrenReturn = Accessor<ResolvedChildren> & {
  toArray: () => ResolvedChildren[];
};
```

### Why It Exists

1. **Prevents re-creation**: Caches resolved children
2. **Resolves nested structures**: Flattens fragments, arrays, functions
3. **Provides stable accessor**: Safe to call multiple times

### Usage Patterns

```typescript
import { children } from "solid-js";

// Basic — resolve children once:
function Wrapper(props: ParentProps) {
  const resolved = children(() => props.children);
  return <div class="wrapper">{resolved()}</div>;
}

// With toArray — iterate over resolved children:
function List(props: ParentProps) {
  const resolved = children(() => props.children);

  return (
    <ul>
      <For each={resolved.toArray()}>
        {(child) => <li>{child}</li>}
      </For>
    </ul>
  );
}

// Function-as-children (render props):
function Slot(props: { children: (data: string) => JSX.Element }) {
  const resolved = children(() => props.children("hello"));
  return <div>{resolved()}</div>;
}
```

### WRONG vs CORRECT

```typescript
// WRONG — accessing props.children multiple times can re-create elements:
function Bad(props: ParentProps) {
  createEffect(() => {
    console.log(props.children); // Re-creates children!
  });
  return <div>{props.children}</div>; // Creates again!
}

// CORRECT — use children() helper:
function Good(props: ParentProps) {
  const resolved = children(() => props.children);
  createEffect(() => {
    console.log(resolved()); // Stable reference
  });
  return <div>{resolved()}</div>;
}
```

---

## 14. Control Flow Components (Show, For, Index, Switch/Match, Dynamic, Portal)
<!-- Lines: ~1365-1650 -->

### 14.1 Show

**Import**: `import { Show } from "solid-js";`

```typescript
interface ShowProps<T> {
  when: T | undefined | null | false;
  keyed?: boolean;
  fallback?: JSX.Element;
  children: JSX.Element | ((accessor: Accessor<T>) => JSX.Element);
}

function Show<T>(props: ShowProps<T>): JSX.Element;
```

#### Props

| Prop | Type | Description |
|------|------|-------------|
| `when` | `T \| undefined \| null \| false` | Condition — renders children when truthy |
| `keyed` | `boolean` | When true, re-renders on reference change (not just truthiness) |
| `fallback` | `JSX.Element` | Rendered when `when` is falsy |
| `children` | `JSX.Element \| (accessor: Accessor<T>) => JSX.Element` | Content or render function |

#### Examples

```typescript
// Basic conditional:
<Show when={isLoggedIn()} fallback={<LoginForm />}>
  <Dashboard />
</Show>

// Keyed — re-renders when user object changes reference:
<Show when={user()} keyed>
  <UserProfile user={user()} />
</Show>

// Render function — receives narrowed non-null accessor:
<Show when={user()}>
  {(u) => <p>Welcome, {u().name}</p>}
</Show>
```

#### Behavior Notes

- Without `keyed`: only truthiness changes trigger re-render
- With `keyed`: reference changes trigger re-render (useful for object identity)
- Render function children are wrapped with `untrack` — signals accessed directly inside the callback function won't create reactive dependencies (access them via the accessor argument)
- **Version**: Available since Solid 1.0

### 14.2 For

**Import**: `import { For } from "solid-js";`

```typescript
function For<T, U extends JSX.Element>(props: {
  each: readonly T[];
  fallback?: JSX.Element;
  children: (item: T, index: () => number) => U;
}): () => U[];
```

#### Props

| Prop | Type | Description |
|------|------|-------------|
| `each` | `readonly T[]` | Array to iterate |
| `fallback` | `JSX.Element` | Shown when array is empty |
| `children` | `(item: T, index: () => number) => U` | Render callback |

#### Callback Signature

- **`item`**: Direct value (T) — NOT a signal
- **`index`**: Signal (function) — MUST call as `index()` to read

#### Keying: By Reference

`<For>` tracks items by **object reference**. When an item moves in the array, the DOM node moves with it (no re-creation). This is why it works best with arrays of objects.

#### Examples

```typescript
// Basic list:
<For each={todos()} fallback={<p>No todos yet</p>}>
  {(todo, index) => (
    <div>
      <span>#{index() + 1}</span>
      <span>{todo.text}</span>
      <input type="checkbox" checked={todo.completed} />
    </div>
  )}
</For>

// With reactive source:
const [items, setItems] = createSignal<Item[]>([]);
<For each={items()}>
  {(item) => <ItemCard item={item} />}
</For>
```

#### Performance Notes

- When array changes, only affected items re-render
- Moving items moves DOM nodes (no recreation)
- Adding/removing items only affects those specific nodes
- **Version**: Available since Solid 1.0

### 14.3 Index

**Import**: `import { Index } from "solid-js";`

```typescript
function Index<T, U extends JSX.Element>(props: {
  each: readonly T[];
  fallback?: JSX.Element;
  children: (item: () => T, index: number) => U;
}): () => U[];
```

#### Callback Signature — OPPOSITE of For

- **`item`**: Signal (function) — MUST call as `item()` to read
- **`index`**: Direct number — NOT a signal

#### When to Use Index vs For

| Scenario | Use |
|----------|-----|
| Array of objects, items reorder | `<For>` (keyed by reference) |
| Array of primitives (strings, numbers) | `<Index>` (keyed by index) |
| Form inputs where position matters | `<Index>` |
| List where items are added/removed/moved | `<For>` |

#### Example

```typescript
// Array of primitive values — Index is ideal:
const [names, setNames] = createSignal(["Alice", "Bob", "Charlie"]);

<Index each={names()}>
  {(name, i) => (
    <input
      value={name()}     // item is a signal — call it
      onInput={(e) => {
        setNames(prev => {
          const next = [...prev];
          next[i] = e.target.value;  // index is a plain number
          return next;
        });
      }}
    />
  )}
</Index>
```

#### Key Differences Summary

| Aspect | `<For>` | `<Index>` |
|--------|---------|-----------|
| Keyed by | Object reference | Array index position |
| `item` | Direct value (T) | Signal (() => T) |
| `index` | Signal (() => number) | Plain number |
| Best for | Objects, reorderable lists | Primitives, form inputs |
| **Version** | 1.0+ | 1.0+ |

### 14.4 Switch/Match

**Import**: `import { Switch, Match } from "solid-js";`

```typescript
function Switch(props: {
  fallback?: JSX.Element;
  children: JSX.Element;
}): () => JSX.Element;

type MatchProps<T> = {
  when: T | undefined | null | false;
  children: JSX.Element | ((item: T) => JSX.Element);
};

function Match<T>(props: MatchProps<T>): JSX.Element;
```

#### Behavior

- Evaluates `<Match>` children **sequentially** — first truthy `when` wins
- Only ONE Match renders at a time (mutual exclusivity)
- Fallback renders when NO Match conditions are truthy

#### Examples

```typescript
// Route-like switching:
<Switch fallback={<NotFound />}>
  <Match when={route() === "home"}>
    <HomePage />
  </Match>
  <Match when={route() === "about"}>
    <AboutPage />
  </Match>
  <Match when={route() === "settings"}>
    <SettingsPage />
  </Match>
</Switch>

// With render function — receives narrowed value:
<Switch>
  <Match when={user()}>
    {(u) => <UserProfile user={u()} />}
  </Match>
  <Match when={error()}>
    {(e) => <ErrorDisplay error={e()} />}
  </Match>
</Switch>
```

**Version**: Available since Solid 1.0

### 14.5 Dynamic

**Import**: `import { Dynamic } from "solid-js/web";`

```typescript
function Dynamic<T>(props: T & {
  children?: any;
  component?: Component<T> | string | keyof JSX.IntrinsicElements;
}): () => JSX.Element;
```

#### The `component` Prop Accepts

- Custom SolidJS components: `component={MyComponent}`
- HTML tag strings: `component={"div"}`, `component={"button"}`
- JSX intrinsic element keys

All additional props are forwarded to the rendered component.

#### Examples

```typescript
// Dynamic component selection:
const [currentView, setCurrentView] = createSignal<Component>(HomeView);

<Dynamic component={currentView()} user={currentUser()} />

// Dynamic HTML element:
const [tag, setTag] = createSignal("h1");

<Dynamic component={tag()}>Hello World</Dynamic>

// Polymorphic "as" pattern:
interface BoxProps {
  as?: string | Component;
  children: JSX.Element;
}

function Box(props: BoxProps) {
  const merged = mergeProps({ as: "div" }, props);
  return (
    <Dynamic component={merged.as}>
      {props.children}
    </Dynamic>
  );
}

// Usage:
<Box as="section">Content</Box>
<Box as={CustomContainer}>Content</Box>
```

**Version**: Available since Solid 1.0

### 14.6 Portal

**Import**: `import { Portal } from "solid-js/web";`

```typescript
function Portal(props: {
  mount?: Node;
  useShadow?: boolean;
  isSVG?: boolean;
  children: JSX.Element;
}): Text;
```

#### Props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `mount` | `Node` | `document.body` | Target DOM node for portal content |
| `useShadow` | `boolean` | `false` | Enable Shadow DOM for style isolation |
| `isSVG` | `boolean` | `false` | Required when mounting into SVG elements |
| `children` | `JSX.Element` | — | Content to render in portal |

#### Behavior

- Inserts content into a `<div>` within the mount target (unless `isSVG` is true)
- Events propagate through the **component tree**, not the DOM tree
- **Client-side only** — hydration is disabled for portals
- Shadow DOM (`useShadow`) encapsulates styles

#### Examples

```typescript
// Modal portal:
<Portal mount={document.getElementById("modal-root")!}>
  <div class="modal-overlay">
    <div class="modal-content">
      <h2>Modal Title</h2>
      <p>Modal content here</p>
    </div>
  </div>
</Portal>

// With Shadow DOM:
<Portal useShadow>
  <style>{`.tooltip { background: black; color: white; }`}</style>
  <div class="tooltip">{tooltipText()}</div>
</Portal>
```

**Version**: Available since Solid 1.0

---

## 15. Error Handling (ErrorBoundary, Suspense)
<!-- Lines: ~1655-1790 -->

### ErrorBoundary

**Import**: `import { ErrorBoundary } from "solid-js";`

```typescript
interface ErrorBoundaryProps {
  fallback: JSX.Element | ((err: any, reset: () => void) => JSX.Element);
  children: JSX.Element;
}

function ErrorBoundary(props: ErrorBoundaryProps): JSX.Element;
```

#### What It Catches

- Errors during JSX rendering
- Errors in `createEffect`, `createMemo`, `createResource`
- Errors in reactive computations

#### What It Does NOT Catch

- Errors in event handlers
- Errors in `setTimeout` / `setInterval` callbacks
- Errors outside the reactive scope

#### Fallback Function Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `err` | `any` | The caught error object |
| `reset` | `() => void` | Re-renders children, clears error state |

#### Examples

```typescript
// With reset capability:
<ErrorBoundary
  fallback={(error, reset) => (
    <div class="error-panel">
      <h2>Something went wrong</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try Again</button>
    </div>
  )}
>
  <RiskyComponent />
</ErrorBoundary>

// Static fallback:
<ErrorBoundary fallback={<p>An error occurred</p>}>
  <App />
</ErrorBoundary>

// Nested boundaries:
<ErrorBoundary fallback={<p>App-level error</p>}>
  <Layout>
    <ErrorBoundary fallback={(e, reset) => <RetryPanel error={e} onRetry={reset} />}>
      <DataView />
    </ErrorBoundary>
  </Layout>
</ErrorBoundary>
```

#### Important

- Errors in the `fallback` itself are NOT caught by the same ErrorBoundary — they propagate to the parent boundary
- **Version**: Available since Solid 1.0

### Suspense

**Import**: `import { Suspense } from "solid-js";`

```typescript
function Suspense(props: {
  fallback?: JSX.Element;
  children: JSX.Element;
}): JSX.Element;
```

#### Props

| Prop | Type | Description |
|------|------|-------------|
| `fallback` | `JSX.Element` | Loading UI shown while resources resolve |
| `children` | `JSX.Element` | Content shown after resources resolve |

#### Behavior

- Tracks ALL `createResource` calls within its boundary
- DOM nodes are created immediately but NOT attached to the document — fallback shows instead
- When all resources resolve, children are attached and fallback is removed
- `onMount` and `createEffect` inside Suspense only run AFTER resources resolve
- Both branches (fallback + children) exist simultaneously in memory

#### Nested Suspense

```typescript
// Independent loading states:
<Suspense fallback={<HeaderSkeleton />}>
  <Header user={userResource()} />

  <Suspense fallback={<ContentSkeleton />}>
    <Content data={dataResource()} />

    <Suspense fallback={<CommentsSkeleton />}>
      <Comments comments={commentsResource()} />
    </Suspense>
  </Suspense>
</Suspense>
```

Each Suspense boundary resolves independently — the header can show before content, content before comments.

#### Key Advantage Over Show

`<Show when={resource()}>` destroys and recreates DOM. `<Suspense>` creates DOM nodes immediately but delays attachment — preserving state and avoiding flicker.

#### Related

- `createResource` — primary trigger for Suspense
- `SuspenseList` — coordinates multiple Suspense boundaries
- **Version**: Available since Solid 1.0

---

## 16. Event Handling
<!-- Lines: ~1795-1880 -->

**Source**: https://docs.solidjs.com/concepts/components/event-handlers

### Two Event Binding Systems

#### 1. Delegated Events (`onClick`, `onInput`, etc.)

- Adds listener to `document`, dispatches to element
- Case-insensitive (`onClick` and `onclick` both work)
- More efficient for frequently-fired events shared across many elements
- Uses event delegation (single document-level listener)

#### 2. Native Events (`on:click`, `on:scroll`, etc.)

- Adds listener directly to the element
- Case-sensitive
- Required for custom events and events not in the delegation list
- Use when you need `event.stopPropagation()` to work correctly

### Delegated Event List (23 Events)

`beforeinput`, `click`, `dblclick`, `contextmenu`, `focusin`, `focusout`, `input`, `keydown`, `keyup`, `mousedown`, `mousemove`, `mouseout`, `mouseover`, `mouseup`, `pointerdown`, `pointermove`, `pointerout`, `pointerover`, `pointerup`, `touchend`, `touchmove`, `touchstart`

For ALL other events, use the `on:` prefix.

### Event Handler Binding (Data Passing)

```typescript
// Array syntax — first element is handler, second is data:
const handler = (data: string, event: MouseEvent) => {
  console.log("Data:", data, "Target:", event.target);
};

<button onClick={[handler, "hello"]}>Click</button>
// Equivalent to: onClick={(e) => handler("hello", e)}
// But avoids creating a new closure on every render
```

### Important: Handlers Are NOT Reactive

```typescript
// WRONG — handler won't update when signal changes:
<div onClick={handler()} />  // Reads signal once

// CORRECT — wrap in arrow function:
<div onClick={() => props.handleClick?.()} />
```

### Event Delegation Caveats

1. **stopPropagation**: Delegated events attach to `document` — `event.stopPropagation()` won't prevent other delegated handlers. Use `on:` prefix when you need propagation control.
2. **Portals**: Events propagate through the component tree, not DOM tree.
3. **Rare events**: For infrequent events, prefer `on:` to avoid keeping document-level listeners alive.

### TypeScript Event Types

```typescript
// Standard event handler:
function handleClick(event: MouseEvent): void { ... }

// Input event:
function handleInput(event: InputEvent): void {
  const target = event.target as HTMLInputElement;
  console.log(target.value);
}

// Custom event with on: prefix:
<div on:myCustomEvent={(e: CustomEvent<{ detail: string }>) => {
  console.log(e.detail);
}} />
```

---

## 17. Refs and Directives
<!-- Lines: ~1885-2000 -->

**Source**: https://docs.solidjs.com/concepts/refs

### Ref Assignment Patterns

#### Variable Assignment

```typescript
function MyComponent() {
  let canvasRef!: HTMLCanvasElement; // ! asserts definite assignment

  onMount(() => {
    // Safe to use — element is in DOM
    const ctx = canvasRef.getContext("2d");
  });

  return <canvas ref={canvasRef} />;
}
```

#### Callback Pattern

```typescript
function MyComponent() {
  return (
    <canvas
      ref={(el) => {
        // el is created but NOT YET in the DOM
        // Use onMount or createEffect for DOM-dependent operations
        console.log("Canvas element created:", el);
      }}
    />
  );
}
```

### TypeScript: Definite Assignment Assertion

SolidJS assigns refs during render, but TypeScript doesn't know this. Use `!:` to suppress "used before assigned" errors:

```typescript
let myDiv!: HTMLDivElement;   // Correct
let myDiv: HTMLDivElement;    // TypeScript error: possibly undefined
```

### Signals as Refs (Conditional Elements)

```typescript
function App() {
  const [show, setShow] = createSignal(false);
  let elementRef!: HTMLParagraphElement;

  return (
    <div>
      <button onClick={() => setShow(s => !s)}>Toggle</button>
      <Show when={show()}>
        <p ref={elementRef}>Now visible</p>
      </Show>
    </div>
  );
}
```

### Ref Forwarding (Parent-to-Child)

```typescript
// Parent:
function Parent() {
  let childCanvas!: HTMLCanvasElement;

  onMount(() => {
    childCanvas.getContext("2d"); // Access child's DOM element
  });

  return <ChildCanvas ref={childCanvas} />;
}

// Child — ref is ALWAYS passed as a callback function:
function ChildCanvas(props: { ref: HTMLCanvasElement | ((el: HTMLCanvasElement) => void) }) {
  return <canvas ref={props.ref} />;
}
```

**Important**: When a parent passes `ref={variable}`, the child receives it as a callback function, regardless of how the parent declared it.

### Directives (`use:`)

Directives attach reusable DOM behaviors:

```typescript
// Directive definition:
function clickOutside(element: Element, accessor: () => () => void): void {
  const onClick = (e: Event) => {
    if (!element.contains(e.target as Node)) {
      accessor()(); // Call the provided callback
    }
  };
  document.addEventListener("click", onClick);
  onCleanup(() => document.removeEventListener("click", onClick));
}

// Usage in JSX:
<div use:clickOutside={() => setOpen(false)}>
  Dropdown content
</div>
```

#### Directive Signature

```typescript
function directive(element: Element, accessor: () => any): void;
```

- `element`: The DOM node the directive is attached to
- `accessor`: Function returning the directive's reactive value

#### Directive Capabilities

- Create signals and effects
- Attach event listeners
- Manipulate DOM attributes/styles
- Run cleanup via `onCleanup`

#### TypeScript: Declaring Directives

To prevent TypeScript errors, declare directives in module scope:

```typescript
// Must declare to avoid "unused variable" and type errors:
declare module "solid-js" {
  namespace JSX {
    interface Directives {
      clickOutside: () => void;
    }
  }
}
```

### Key Differences from React useRef

| Aspect | React `useRef` | SolidJS `ref` |
|--------|---------------|---------------|
| Declaration | `const ref = useRef(null)` | `let ref!: HTMLElement` |
| Access | `ref.current` | `ref` (direct) |
| Timing | After mount (useEffect) | Assigned during render, use in onMount |
| Hook required | Yes (useRef) | No — plain variable |
| Directives | No built-in system | `use:` prefix for reusable behaviors |
| Forwarding | `forwardRef()` HOC | Pass `ref` as regular prop |
| Multiple behaviors | Requires manual composition | Multiple `use:` directives per element |

---

## 18. Context API
<!-- Lines: ~2005-2090 -->

**Source**: https://docs.solidjs.com/concepts/context

### createContext

```typescript
import { createContext } from "solid-js";

const MyContext = createContext<T>(defaultValue?);
```

Returns a context object with a `Provider` property.

### Provider Pattern

```tsx
<MyContext.Provider value={someValue}>
  {props.children}
</MyContext.Provider>
```

### useContext

```typescript
import { useContext } from "solid-js";

const value = useContext(MyContext);
```

### Reactive Context with Signals

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

### Custom Hook Pattern (Recommended)

```tsx
function useCounter() {
  const context = useContext(CounterContext);
  if (!context) throw new Error("useCounter must be used within CounterProvider");
  return context;
}
```

### Nested Contexts

Multiple contexts can coexist. Inner providers override outer ones for the same context.

---

## 19. SolidStart Meta-Framework
<!-- Lines: ~2095-2480 -->

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

> Source: https://docs.solidjs.com/solid-start

### 19.1 Getting Started

```bash
npm init solid@latest      # or pnpm/yarn/bun/deno equivalents
```

Available templates: basic, bare, with-solidbase, with-auth, with-authjs, with-drizzle, with-mdx, with-prisma, with-solid-styled, with-tailwindcss.

**Default project structure:**
```
public/
src/
  routes/
    index.tsx
  entry-client.tsx    # Client-side hydration entry
  entry-server.tsx    # Server request handler entry
  app.tsx             # HTML root component (shared client/server)
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

> Source: https://docs.solidjs.com/solid-start/getting-started

### 19.2 File-Based Routing

SolidStart uses the `routes/` directory for automatic route generation. The `<FileRoutes />` component from `@solidjs/start/router` traverses this directory and generates route definitions.

#### Basic Route Mapping

| File | URL |
|------|-----|
| `routes/index.tsx` | `/` |
| `routes/blog.tsx` | `/blog` |
| `routes/contact.tsx` | `/contact` |

Each route file MUST default-export a component.

#### Nested Routes

Folders create nested URL segments:
- `routes/blog/article-1.tsx` -> `/blog/article-1`
- `routes/work/job-1.tsx` -> `/work/job-1`

#### Nested Layouts

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

#### Index File Alternatives

Rename `index.tsx` to `(foldername).tsx` for better file searchability:
```
routes/socials/(socials).tsx    # renders at /socials
```

#### Dynamic Routes

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

#### Optional Parameters

Double brackets for optional segments:
- `routes/users/[[id]].tsx` -> matches `/users`, `/users/1`, `/users/abc`

#### Catch-All Routes

Ellipsis syntax for wildcard matching:
- `routes/blog/[...post].tsx` -> captures all remaining segments

```tsx
export default function BlogPage() {
  const params = useParams();
  // params.post = "foo/bar/baz" (forward-slash delimited)
  return <div>Blog {params.post}</div>;
}
```

#### Route Groups

Parenthesized folder names organize routes without affecting URLs:
```
routes/
  (static)/
    about-us/
      index.tsx         # /about-us
    contact-us/
      index.tsx         # /contact-us
```

#### Escaping Nested Routes

Parenthesized folders can also escape nested layout inheritance:
```
routes/
  users/
    index.tsx           # /users (under users layout)
    projects.tsx        # /users/projects (under users layout)
  users(details)/
    [id].tsx            # /users/:id (separate layout, NOT under users/)
```

#### Route Configuration

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

> Source: https://docs.solidjs.com/solid-start/building-your-application/routing

### 19.3 Data Loading

#### query — Server-Side Data Fetching with Caching

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
- `getUser.key` -> `"userProfile"` (base key)
- `getUser.keyFor(userId)` -> `"userProfile[\"abc123\"]"` (argument-specific key)

**Caching behavior — data is reused without re-fetching when:**
1. Within 5-second preloading window (hover-triggered)
2. While components actively use the query via `createAsync`
3. On back/forward navigation (up to 5 minutes)
4. Repeated calls within a single SSR request (server-side deduplication)
5. During client hydration (uses server-provided data)

**Key serialization:** Arguments are serialized with `JSON.stringify` with sorted keys — `{ name: "Ryan", awesome: true }` and `{ awesome: true, name: "Ryan" }` produce the same cache key.

> Source: https://docs.solidjs.com/solid-router/reference/data-apis/query, https://docs.solidjs.com/solid-router/reference/data-apis/cache

#### createAsync — Reactive Async Data Primitive

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

> Source: https://docs.solidjs.com/solid-router/reference/data-apis/create-async

#### createAsyncStore — Fine-Grained Async Store

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

> Source: https://docs.solidjs.com/solid-router/reference/data-apis/create-async-store

#### Route Preloading

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

> Source: https://docs.solidjs.com/solid-router/reference/preload-functions/preload

### 19.4 Server Functions

The `"use server"` directive marks functions to execute exclusively on the server. The compiler transforms them into RPC calls.

#### Function-Level Usage

```typescript
const getUser = async (id: string) => {
  "use server";
  return db.getUser(id);
};
```

#### File-Level Usage

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

#### Requirements

- Functions MUST be `async` or return a `Promise`
- Arguments and return values MUST be serializable (uses Seroval for serialization)
- Functions run server-side regardless of rendering mode (CSR, SSR, etc.)

#### Integration with query/action

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

#### Single-Flight Mutations

Server-thrown `redirect()` is handled server-side — the redirected page's data streams in the same HTTP response. This avoids a client-side round-trip.

#### Meta Information

```typescript
import { getServerFunctionMeta } from "@solidjs/start/server";
```

Provides stable function identifiers for multi-worker/multi-core environments.

> Source: https://docs.solidjs.com/solid-start/reference/server/use-server

### 19.5 Actions and Mutations

#### action() — Mutation Wrapper

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

#### Form Integration

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

#### Passing Additional Arguments

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

#### Programmatic Invocation with useAction

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

#### Submission Tracking with useSubmission

```tsx
import { useSubmission } from "@solidjs/router";

const submission = useSubmission(addTodo);
// submission.pending — boolean
// submission.result — resolved value
// submission.error — error if failed
// submission.input — original arguments
```

`useSubmissions` (plural) tracks ALL active submissions for an action.

#### Automatic Revalidation

After an action completes successfully, ALL active queries on the same page are automatically revalidated — unless revalidation is explicitly configured via response helpers.

#### Response Helpers

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

> Source: https://docs.solidjs.com/solid-router/reference/data-apis/action

### 19.6 API Routes

SolidStart supports server-side API endpoints alongside UI routes.

#### File Conventions

API routes use the same file naming as UI routes but export HTTP method handlers instead of a default component. Commonly placed in `routes/api/`.

```
routes/
  api/
    users/
      [id].ts          # /api/users/:id
    products.ts         # /api/products
```

#### HTTP Method Handlers

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

#### APIEvent Type

```typescript
interface APIEvent {
  request: Request;          // Standard Web Request object
  params: Record<string, string>;  // Dynamic route params
  fetch: (input: RequestInfo, init?: RequestInit) => Promise<Response>;  // Internal fetch
}
```

#### Example

```typescript
import type { APIEvent } from "@solidjs/start/server";

export async function GET({ params }: APIEvent) {
  const products = await store.getProducts(params.category);
  return products;  // Automatically serialized to JSON
}
```

#### Session/Cookie Access

```typescript
import { getCookie } from "vinxi/http";

export async function GET(event: APIEvent) {
  const userId = getCookie("userId");
  if (!userId) return new Response("Unauthorized", { status: 401 });
  return await store.getUser(event.params.userId);
}
```

**IMPORTANT:** API routes are prioritized over UI routes at the same path. Use `Accept` headers to differentiate if they overlap.

> Source: https://docs.solidjs.com/solid-start/building-your-application/api-routes

---

## 20. Solid Router
<!-- Lines: ~2485-2660 -->

`@solidjs/router` — universal routing for SolidJS applications, client and server.

**Requires:** Solid v1.8.4 or later.

```bash
npm install @solidjs/router
```

### Router Setup

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

### Core Components

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

> Source: https://docs.solidjs.com/solid-router/reference/components/a

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

> Source: https://docs.solidjs.com/solid-router/reference/components/route

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

### Navigation Hooks

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

> Source: https://docs.solidjs.com/solid-router/reference/primitives/use-navigate

#### useParams

```tsx
const params = useParams();
// From /users/:id -> params.id
```

#### useSearchParams

```tsx
const [search, setSearch] = useSearchParams();
// From ?page=1&sort=name -> search.page, search.sort
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

### Lazy Loading

```tsx
import { lazy } from "solid-js";
const Dashboard = lazy(() => import("./pages/Dashboard"));

<Route path="/dashboard" component={Dashboard} />
```

> Source: https://docs.solidjs.com/solid-router

---

## 21. SSR and Hydration
<!-- Lines: ~2665-2780 -->

### renderToString — Synchronous SSR

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

> Source: https://docs.solidjs.com/reference/rendering/render-to-string

### renderToStream — Streaming SSR

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

> Source: https://docs.solidjs.com/reference/rendering/render-to-stream

### hydrate — Client-Side Rehydration

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

> Source: https://docs.solidjs.com/reference/rendering/hydrate

### HydrationScript / generateHydrationScript

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

> Source: https://docs.solidjs.com/reference/rendering/hydration-script

### isServer

Boolean constant — `true` on server, `false` on client. Tree-shaken at build time.

```typescript
import { isServer } from "solid-js/web";

if (isServer) {
  // Server-only code — removed from client bundle
}
```

---

## 22. Ecosystem
<!-- Lines: ~2785-2850 -->

### Solid Primitives

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

> Source: https://github.com/solidjs-community/solid-primitives

### Other Notable Ecosystem Packages

- **Kobalte** (`@kobalte/core`) — accessible, unstyled UI component library (similar to Radix UI)
- **solid-transition-group** — CSS transition utilities for enter/exit animations
- **babel-preset-solid** — JSX compilation preset (required build dependency)
- **solid-testing-library** — testing utilities following Testing Library conventions
- **solid-devtools** — browser devtools extension for inspecting signals and components

> Source: https://github.com/solidjs/solid

---

## 23. Version Matrix (SolidJS 1.x/2.x, SolidStart 0.x/1.x)
<!-- Lines: ~2855-2960 -->

### SolidJS Core

| Version | Status | Key Features |
|---------|--------|-------------|
| 1.x (1.8+) | Stable, current | Fine-grained reactivity, signals, stores, effects, Suspense, ErrorBoundary, SSR, streaming, portals, web components |
| 2.0 | Planned/In development | `createAsync` as standard primitive (replacing `createResource`), potential API refinements |

**Browser support:** Last 2 years of Firefox, Safari, Chrome, Edge.
**Server support:** Node LTS, Deno, Cloudflare Workers.

### SolidJS 2.x Major Reactivity Changes

| Feature | SolidJS 1.x | SolidJS 2.x |
|---------|-------------|-------------|
| **Reactivity model** | Synchronous by default | Microtask-batched — reads don't update until batch flushes |
| **Async** | Manual with createResource | First-class — computations can return Promises/async iterables |
| **Effects** | Single `createEffect` | Split compute -> apply pattern |
| **Lifecycle** | `onMount` | `onSettled` (can return cleanup) |
| **Store setters** | Path-style ergonomics | Draft-first by default |
| **List rendering** | `<Index>` component | `<For keyed={false}>` with accessors |
| **DOM directives** | `use:` directives | `ref` directive factories + arrays |
| **Derived state** | createMemo only | `createSignal(fn)` and `createStore(fn)` for derived-but-writable patterns |

### Key 2.x Breaking Changes

1. **Microtask batching**: All signal updates are batched by default. Reads don't reflect updates until the batch flushes. Use `flush()` when immediate propagation is needed.

2. **Async as first-class**: Computations can return Promises or async iterables. The reactivity graph handles suspension and resumption automatically.

3. **`onMount` -> `onSettled`**: The lifecycle hook is renamed and enhanced — `onSettled` can return a cleanup function.

4. **Dev guardrails**: SolidJS 2.x adds warnings for accidental top-level reads and reactive scope writes, catching common mistakes at development time.

### New 2.x Primitives

- **`<Loading>`** — Shows fallback during initial render, maintains UI stability during background work
- **`isPending()`** — Tracks "refreshing" work without tearing down UI
- **`action()`** — Dedicated mutation primitive with optimistic patterns
- **`createOptimistic` / `createOptimisticStore`** — Explicit optimistic UI handling

### Key 1.x Evolution (1.4-1.9)

- Better TypeScript support
- Top-level arrays in stores
- Streaming HTML support
- Improved hydration mechanisms for SSR
- Batch behavior change: accessing signals/memos inside `batch()` triggers immediate recalculation if needed (since 1.4)

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

### Component-Level API Stability

All components and APIs documented in sections 10-17 are available since **SolidJS 1.0**. The documentation reviewed (docs.solidjs.com, March 2026) does not indicate breaking changes to these APIs between 1.x and 2.x:

- `Show`, `For`, `Index`, `Switch`, `Match`, `Dynamic`, `Portal`, `ErrorBoundary`, `Suspense` — stable API since 1.0
- `splitProps`, `mergeProps`, `children` — stable API since 1.0
- `keyed` prop on `<Show>` — available since 1.0
- Directive system (`use:`) — stable since 1.0
- Event delegation list — stable since 1.0

SolidJS 2.0 introduces changes primarily around the reactivity system internals and compilation improvements, but the component-level API surface remains compatible.

> Sources: https://github.com/solidjs/solid/releases, https://docs.solidjs.com/solid-router

---

## 24. React vs SolidJS Anti-Pattern Catalog
<!-- Lines: ~2965-3350 -->

This section consolidates ALL anti-patterns discovered across all research fragments into a single numbered catalog. These patterns are the PRIMARY reason Claude generates broken SolidJS code.

---

### AP-001: Destructuring Props (Kills Tracking)

**Category:** Reactivity
**Severity:** Critical
**React pattern:** Destructure props in function signature or body
**SolidJS consequence:** Values freeze at initial render, never update

```typescript
// WRONG — Destructuring reads values once, loses reactive tracking
function Greeting({ name }) {        // Destructured at call time
  return <h1>Hello {name}</h1>;       // Static value, never updates
}

// WRONG — also breaks reactivity:
function Greeting(props: { name: string }) {
  const { name } = props; // Extracted once, never updates
  return <h1>Hello {name}</h1>;
}

// WRONG — same problem:
function Greeting(props: { name: string }) {
  const name = props.name; // Captured once
  return <h1>Hello {name}</h1>;
}

// CORRECT — Access props object reactively
function Greeting(props) {
  return <h1>Hello {props.name}</h1>; // Tracked, updates when name changes
}

// CORRECT — Use splitProps or mergeProps for selective access
import { splitProps } from "solid-js";
function Greeting(props) {
  const [local, others] = splitProps(props, ["name"]);
  return <h1 {...others}>Hello {local.name}</h1>;
}

// CORRECT — wrap in a function if needed outside JSX:
function Greeting(props: { name: string }) {
  const name = () => props.name; // Creates a derived accessor
  return <h1>Hello {name()}</h1>;
}
```

---

### AP-002: Destructuring Signals (Kills Tracking)

**Category:** Reactivity
**Severity:** Critical
**React pattern:** Store signal value in a variable
**SolidJS consequence:** Snapshot value, never updates

```typescript
// WRONG — Destructuring reads the signal value once
const [count, setCount] = createSignal(0);
const value = count(); // Snapshot, not reactive
return <div>{value}</div>; // Never updates

// CORRECT — Call signal getter in JSX
return <div>{count()}</div>; // Tracked, updates reactively
```

---

### AP-003: useState vs createSignal

**Category:** Reactivity
**Severity:** Critical
**React pattern:** `useState` returns a value
**SolidJS difference:** `createSignal` returns a GETTER FUNCTION

```typescript
// React: useState returns a value
const [count, setCount] = useState(0);
return <div>{count}</div>; // Value, triggers re-render

// SolidJS: createSignal returns a GETTER FUNCTION
const [count, setCount] = createSignal(0);
return <div>{count()}</div>; // Must call it! Getter function, no re-render
```

---

### AP-004: useEffect vs createEffect (Different Semantics)

**Category:** Reactivity
**Severity:** High
**React pattern:** Manual dependency array, cleanup via return
**SolidJS difference:** Automatic tracking, cleanup via onCleanup

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

---

### AP-005: useMemo vs createMemo

**Category:** Reactivity
**Severity:** High
**React pattern:** `useMemo` with dependency array, NOT a reactive source
**SolidJS difference:** `createMemo` auto-tracked, IS a reactive source

```typescript
// React: useMemo is NOT a reactive source, recalculates on re-render
const double = useMemo(() => count * 2, [count]); // Manual deps

// SolidJS: createMemo IS a reactive source, auto-tracked
const double = createMemo(() => count() * 2); // No deps needed

// WRONG — Using createEffect where createMemo should be used
createEffect(() => {
  setDouble(count() * 2); // Unnecessary effect + signal
});

// CORRECT — Use createMemo for derived values
const double = createMemo(() => count() * 2); // Cached, reactive
```

---

### AP-006: Component Re-Render Mental Model

**Category:** Architecture
**Severity:** Critical
**React pattern:** Component function re-runs on every state change
**SolidJS difference:** Component function runs ONCE

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

---

### AP-007: Conditional Signal Access (Breaks Tracking)

**Category:** Reactivity
**Severity:** High
**React pattern:** N/A (React doesn't have this issue due to re-renders)
**SolidJS consequence:** Dependency not always tracked

```typescript
// WRONG — Conditional access means dependency is not always tracked
createEffect(() => {
  if (someCondition) {
    console.log(count()); // Only tracked when condition is true!
  }
});

// CORRECT — Access signal unconditionally, use value conditionally
createEffect(() => {
  const c = count(); // Always tracked
  if (someCondition) {
    console.log(c);
  }
});
```

---

### AP-008: Early Return Before Signal Access

**Category:** Reactivity
**Severity:** High
**React pattern:** Early returns are safe (component re-runs)
**SolidJS consequence:** Signals after early return are never tracked

```typescript
// WRONG — Early return prevents tracking of signals after it
createEffect(() => {
  if (loading()) return; // If true, name() is never tracked
  console.log(name());
});

// CORRECT — Access all signals first
createEffect(() => {
  const isLoading = loading();
  const currentName = name(); // Always tracked
  if (isLoading) return;
  console.log(currentName);
});
```

---

### AP-009: Storing Signals in Variables (Snapshot Problem)

**Category:** Reactivity
**Severity:** Medium
**React pattern:** Values are snapshots (safe due to re-renders)
**SolidJS consequence:** Stale values in closures

```typescript
// WRONG — Storing getter result breaks reactivity
const currentCount = count(); // Snapshot at this moment
setTimeout(() => {
  console.log(currentCount); // Stale value!
}, 1000);

// CORRECT — Call getter when you need the value
setTimeout(() => {
  console.log(count()); // Current value at time of execution
}, 1000);
```

---

### AP-010: Spreading Props into Child Components

**Category:** Reactivity
**Severity:** Medium
**React pattern:** Spread props freely
**SolidJS consequence:** May lose reactivity for dynamic props

```typescript
// WRONG — Spreading reactive props breaks tracking for some patterns
function Parent(props) {
  return <Child {...props} />; // May lose reactivity for dynamic props
}

// CORRECT — Use splitProps and mergeProps
import { splitProps, mergeProps } from "solid-js";
function Parent(props) {
  const [local, others] = splitProps(props, ["class"]);
  const merged = mergeProps({ class: "default" }, others);
  return <Child {...merged} />;
}
```

---

### AP-011: Array.map vs `<For>`

**Category:** JSX/Components
**Severity:** High
**React pattern:** `Array.map` with `key` prop
**SolidJS consequence:** Re-creates all DOM nodes on every change

```typescript
// WRONG (React pattern):
function TodoList(props: { todos: Todo[] }) {
  return (
    <ul>
      {props.todos.map((todo, i) => (
        <li key={todo.id}>{todo.text}</li>  // key prop doesn't exist in Solid
      ))}
    </ul>
  );
}

// CORRECT (SolidJS):
function TodoList(props: { todos: Todo[] }) {
  return (
    <ul>
      <For each={props.todos}>
        {(todo, index) => <li>{todo.text}</li>}
      </For>
    </ul>
  );
}
```

**Why**: `Array.map` re-creates all DOM nodes on every change. `<For>` tracks items by reference and only updates what changed.

---

### AP-012: Ternary vs `<Show>`

**Category:** JSX/Components
**Severity:** Medium
**React pattern:** Ternary expressions for conditional rendering
**SolidJS consequence:** Unnecessary DOM recreation

```typescript
// WRONG (React pattern):
{isLoggedIn() ? <Dashboard /> : <LoginForm />}
// Works but can cause unnecessary DOM recreation

// CORRECT (SolidJS):
<Show when={isLoggedIn()} fallback={<LoginForm />}>
  <Dashboard />
</Show>
```

**Why**: `<Show>` leverages fine-grained reactivity to minimize DOM mutations. Ternaries can work in simple cases but `<Show>` is the idiomatic approach.

---

### AP-013: switch/case vs `<Switch>`/`<Match>`

**Category:** JSX/Components
**Severity:** High
**React pattern:** JavaScript switch statement in component body
**SolidJS consequence:** Runs once, never updates

```typescript
// WRONG (React pattern):
function Router(props: { route: string }) {
  switch (props.route) {
    case "home": return <Home />;
    case "about": return <About />;
    default: return <NotFound />;
  }
}

// CORRECT (SolidJS):
function Router(props: { route: string }) {
  return (
    <Switch fallback={<NotFound />}>
      <Match when={props.route === "home"}><Home /></Match>
      <Match when={props.route === "about"}><About /></Match>
    </Switch>
  );
}
```

**Why**: `switch` in the component body runs once (components don't re-execute). `<Switch>`/`<Match>` are reactive and update when conditions change.

---

### AP-014: Early Returns in Component Body

**Category:** JSX/Components
**Severity:** Critical
**React pattern:** Early return for loading/null states
**SolidJS consequence:** Output frozen at first render

```typescript
// WRONG (React pattern):
function Profile(props: { user: User | null }) {
  if (!props.user) return <p>Loading...</p>;  // Runs once, never updates
  return <div>{props.user.name}</div>;
}

// CORRECT (SolidJS):
function Profile(props: { user: User | null }) {
  return (
    <Show when={props.user} fallback={<p>Loading...</p>}>
      {(user) => <div>{user().name}</div>}
    </Show>
  );
}
```

**Why**: Component body runs ONCE. An early return freezes the output. `<Show>` is reactive and switches when the condition changes.

---

### AP-015: key Prop (React) vs `<For>` Keyed-by-Reference

**Category:** JSX/Components
**Severity:** Low
**React pattern:** `key` prop on list items
**SolidJS consequence:** `key` prop is ignored

```typescript
// WRONG (React pattern — key prop is meaningless in SolidJS):
<For each={items()}>
  {(item) => <div key={item.id}>{item.name}</div>}
</For>

// CORRECT (SolidJS — For keys by object reference automatically):
<For each={items()}>
  {(item) => <div>{item.name}</div>}
</For>
```

**Why**: SolidJS `<For>` tracks items by reference identity, not by a `key` prop. The `key` prop is ignored.

---

### AP-016: useRef vs let ref / ref Callback

**Category:** JSX/Components
**Severity:** Medium
**React pattern:** `useRef` hook with `.current` access
**SolidJS difference:** Plain variable with `!:` assertion

```typescript
// WRONG (React pattern):
import { useRef, useEffect } from "react";
function Canvas() {
  const ref = useRef<HTMLCanvasElement>(null);
  useEffect(() => { ref.current?.getContext("2d"); }, []);
  return <canvas ref={ref} />;
}

// CORRECT (SolidJS):
import { onMount } from "solid-js";
function Canvas() {
  let ref!: HTMLCanvasElement;
  onMount(() => { ref.getContext("2d"); });
  return <canvas ref={ref} />;
}
```

---

### AP-017: React.createElement vs Direct DOM

**Category:** Architecture
**Severity:** Informational
**React pattern:** JSX -> React.createElement() -> Virtual DOM -> Reconciliation -> DOM
**SolidJS difference:** JSX -> Compiled DOM creation -> Direct DOM nodes + reactive effects

```typescript
// React: JSX -> React.createElement() -> Virtual DOM -> Reconciliation -> DOM
// SolidJS: JSX -> Compiled DOM creation -> Direct DOM nodes + reactive effects

// This means in SolidJS:
// - No virtual DOM overhead
// - No diffing algorithm
// - Updates are surgical — only the exact text node/attribute changes
```

---

### AP-018: Re-render Assumption (Derived Values in Component Body)

**Category:** Architecture
**Severity:** Critical
**React pattern:** Compute derived values as plain expressions
**SolidJS consequence:** Computed once, never updates

```typescript
// WRONG (React mental model — expecting component to re-run):
function Counter() {
  const [count, setCount] = createSignal(0);
  const doubled = count() * 2;  // Computed ONCE, never updates
  return <p>{doubled}</p>;
}

// CORRECT (SolidJS — derive reactively):
function Counter() {
  const [count, setCount] = createSignal(0);
  const doubled = () => count() * 2;  // Function — re-evaluates on access
  // OR: const doubled = createMemo(() => count() * 2);  // Cached computation
  return <p>{doubled()}</p>;
}
```

**Why**: In React, `doubled` recalculates on every render. In SolidJS, the component runs once, so `doubled` must be a function or memo to stay reactive.

---

### AP-019: children as ReactNode vs children() Helper

**Category:** JSX/Components
**Severity:** Medium
**React pattern:** Access `children` as a stable value
**SolidJS consequence:** Multiple access re-creates elements

```typescript
// WRONG (React pattern — treating children as a value):
function Wrapper(props: { children: JSX.Element }) {
  const kids = props.children;  // May re-create on each access!
  return <div>{kids}</div>;
}

// CORRECT (SolidJS):
import { children } from "solid-js";
function Wrapper(props: { children: JSX.Element }) {
  const resolved = children(() => props.children);
  return <div>{resolved()}</div>;
}
```

**Why**: In React, `children` is a stable value. In SolidJS, `props.children` is a getter that may re-evaluate. The `children()` helper resolves and caches it.

---

### AP-020: useEffect Cleanup Pattern

**Category:** Reactivity
**Severity:** Medium
**React pattern:** Return cleanup function from useEffect
**SolidJS difference:** Use separate onCleanup call

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
```

---

### AP-021: useEffect with Empty Deps for Mount

**Category:** Reactivity
**Severity:** Low
**React pattern:** `useEffect(() => {}, [])` for mount logic
**SolidJS difference:** Dedicated `onMount` function

```typescript
// WRONG — React: useEffect with empty deps for mount
useEffect(() => { /* mount logic */ }, []);

// CORRECT — SolidJS: dedicated onMount
onMount(() => { /* mount logic */ }); // Clear intent, non-tracking
```

---

### AP-022: State Replacement vs Surgical Updates (Stores)

**Category:** State Management
**Severity:** High
**React pattern:** Spread/replace entire state object
**SolidJS consequence:** Replaces everything, loses fine-grained tracking

```typescript
// WRONG — React pattern: spread/replace entire state
setState({ ...state, users: [...state.users, newUser] }); // Replaces everything

// CORRECT — SolidJS: path syntax for surgical updates
setStore("users", store.users.length, newUser); // Only adds to array
```

---

### AP-023: Destructuring Store (Kills Reactivity)

**Category:** State Management
**Severity:** Critical
**React pattern:** Destructure state freely
**SolidJS consequence:** Static snapshot, never updates

```typescript
// WRONG — Destructuring store (kills reactivity)
const { username } = store.users[0]; // Static snapshot, never updates

// CORRECT — Access in tracking scope
<span>{store.users[0].username}</span> // Fine-grained tracking
```

---

### AP-024: React Router `element` vs Solid Router `component`

**Category:** Routing
**Severity:** High
**React pattern:** `<Route element={<Component />}>`
**SolidJS difference:** `<Route component={Component}>`

```typescript
// WRONG — React pattern
<Route path="/" element={<Home />} />  // Creates element immediately

// CORRECT — SolidJS pattern
<Route path="/" component={Home} />    // Defers creation to router
```

**CRITICAL:** Using `element={<Component />}` (React pattern) instead of `component={Component}` (Solid pattern). In SolidJS, passing a JSX element via `element` creates it immediately; `component` defers creation to the router.

---

### AP-025: React Router `<Link>` vs Solid Router `<A>`

**Category:** Routing
**Severity:** Medium
**React pattern:** `<Link to="/path">`
**SolidJS difference:** `<A href="/path">`

```typescript
// WRONG — React Router pattern
import { Link } from "react-router-dom";
<Link to="/about">About</Link>

// CORRECT — Solid Router
import { A } from "@solidjs/router";
<A href="/about">About</A>
```

---

### AP-026: useRouter (Next.js) vs useNavigate

**Category:** Routing
**Severity:** Medium
**React pattern:** `useRouter()` from Next.js
**SolidJS difference:** `useNavigate()` from @solidjs/router

```tsx
// WRONG — React pattern
import { useRouter } from "next/router";
const router = useRouter();
router.push("/dashboard");

// CORRECT — SolidJS pattern
import { useNavigate } from "@solidjs/router";
const navigate = useNavigate();
navigate("/dashboard");
```

---

### AP-027: Data Fetching in useEffect

**Category:** Data Loading
**Severity:** High
**React pattern:** Fetch data in useEffect with manual state management
**SolidJS difference:** Use query + createAsync (or createResource)

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

---

### AP-028: Form Handling with preventDefault

**Category:** Data Loading
**Severity:** Medium
**React pattern:** Manual form submission with preventDefault
**SolidJS difference:** Form actions with progressive enhancement

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

### AP-029: getServerSideProps Pattern

**Category:** Data Loading
**Severity:** High
**React pattern:** `getServerSideProps` from Next.js
**SolidJS difference:** No equivalent — use `"use server"` inside `query()` functions

```tsx
// WRONG — Next.js pattern (no SolidJS equivalent)
export async function getServerSideProps() {
  const data = await fetchData();
  return { props: { data } };
}

// CORRECT — SolidJS pattern
const getData = query(async () => {
  "use server";
  return fetchData();
}, "data");
```

---

### AP-030: Manual Loading/Error State for Async Data

**Category:** Data Loading
**Severity:** Medium
**React pattern:** Separate useState for loading, error, and data
**SolidJS difference:** createResource provides built-in state machine

```typescript
// WRONG — React pattern: manual loading/error state
const [data, setData] = useState(null);
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);

// CORRECT — SolidJS: createResource handles all of this
const [data] = createResource(fetcher);
// data(), data.loading, data.error, data.state, data.latest — all built-in
```

---

### AP-031: Expecting Effects to Run During SSR

**Category:** SSR
**Severity:** Medium
**React pattern:** N/A
**SolidJS constraint:** createEffect NEVER runs on the server

```typescript
// WRONG — Expecting effects to run during SSR
createEffect(() => {
  // This NEVER runs on server
  localStorage.setItem("count", String(count()));
});

// CORRECT — Guard with isServer or use different approach
import { isServer } from "solid-js/web";
createEffect(() => {
  if (!isServer) {
    localStorage.setItem("count", String(count()));
  }
});
```

---

## Appendix: Complete Import Map
<!-- Lines: ~3355-3400 -->

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

// Component utilities
import {
  splitProps,
  mergeProps,
  children,
} from "solid-js";

// Component types
import type {
  Component,
  ParentComponent,
  VoidComponent,
  FlowComponent,
} from "solid-js";

// Control flow
import {
  Show,
  For,
  Index,
  Switch,
  Match,
  Suspense,
  ErrorBoundary,
} from "solid-js";

// Context
import {
  createContext,
  useContext,
} from "solid-js";

// Store utilities (separate entry point!)
import {
  createStore,
  createMutable,
  produce,
  reconcile,
  unwrap,
} from "solid-js/store";

// Web-specific (separate entry point!)
import {
  Portal,
  Dynamic,
  isServer,
  hydrate,
  render,
  renderToString,
  renderToStream,
  HydrationScript,
  generateHydrationScript,
} from "solid-js/web";

// Solid Router
import {
  Router,
  Route,
  A,
  Navigate,
  HashRouter,
  MemoryRouter,
  useNavigate,
  useParams,
  useSearchParams,
  useLocation,
  useMatch,
  useIsRouting,
  useBeforeLeave,
  useCurrentMatches,
  usePreloadRoute,
  query,
  createAsync,
  createAsyncStore,
  action,
  useAction,
  useSubmission,
  useSubmissions,
  redirect,
  json,
  reload,
  revalidate,
} from "@solidjs/router";

// SolidStart
import { FileRoutes } from "@solidjs/start/router";
import { getServerFunctionMeta } from "@solidjs/start/server";

// Lazy loading
import { lazy } from "solid-js";
```

---

## Appendix: URL Verification Log

| URL | Status | Notes |
|-----|--------|-------|
| https://docs.solidjs.com/concepts/intro-to-reactivity | OK | Reactivity model |
| https://docs.solidjs.com/concepts/signals | OK | Signals concept |
| https://docs.solidjs.com/reference/basic-reactivity/create-signal | OK | createSignal API |
| https://docs.solidjs.com/concepts/effects | OK | Effects concept |
| https://docs.solidjs.com/reference/basic-reactivity/create-effect | OK | createEffect API |
| https://docs.solidjs.com/reference/secondary-primitives/create-render-effect | OK | createRenderEffect API |
| https://docs.solidjs.com/reference/secondary-primitives/create-computed | OK | createComputed API |
| https://docs.solidjs.com/reference/basic-reactivity/create-memo | OK | createMemo API |
| https://docs.solidjs.com/reference/basic-reactivity/create-resource | OK | createResource API |
| https://docs.solidjs.com/concepts/stores | OK | Stores concept |
| https://docs.solidjs.com/reference/store-utilities/create-store | OK | createStore API |
| https://docs.solidjs.com/reference/store-utilities/produce | OK | produce API |
| https://docs.solidjs.com/reference/store-utilities/reconcile | OK | reconcile API |
| https://docs.solidjs.com/reference/store-utilities/unwrap | OK | unwrap API |
| https://docs.solidjs.com/reference/store-utilities/create-mutable | OK | createMutable API |
| https://docs.solidjs.com/reference/reactive-utilities/batch | OK | batch API |
| https://docs.solidjs.com/reference/reactive-utilities/untrack | OK | untrack API |
| https://docs.solidjs.com/reference/reactive-utilities/observable | OK | observable API |
| https://docs.solidjs.com/reference/reactive-utilities/from | OK | from API |
| https://docs.solidjs.com/reference/lifecycle/on-mount | OK | onMount API |
| https://docs.solidjs.com/reference/lifecycle/on-cleanup | OK | onCleanup API |
| https://docs.solidjs.com/concepts/understanding-jsx | OK | JSX compilation |
| https://docs.solidjs.com/concepts/components/basics | OK | Component basics |
| https://docs.solidjs.com/concepts/components/props | OK | Props handling |
| https://docs.solidjs.com/reference/component-apis/children | OK | children helper |
| https://docs.solidjs.com/reference/components/show | OK | Show component |
| https://docs.solidjs.com/reference/components/for | OK | For component |
| https://docs.solidjs.com/reference/components/index-component | OK | Index component |
| https://docs.solidjs.com/reference/components/switch-and-match | OK | Switch/Match |
| https://docs.solidjs.com/reference/components/dynamic | OK | Dynamic component |
| https://docs.solidjs.com/reference/components/portal | OK | Portal component |
| https://docs.solidjs.com/reference/components/error-boundary | OK | ErrorBoundary |
| https://docs.solidjs.com/reference/components/suspense | OK | Suspense |
| https://docs.solidjs.com/concepts/components/event-handlers | OK | Event handling |
| https://docs.solidjs.com/concepts/refs | OK | Refs and directives |
| https://docs.solidjs.com/concepts/context | OK | Context API |
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
| https://docs.solidjs.com/guides/fetching-data | OK | createResource |
| https://docs.solidjs.com/reference/rendering/render-to-string | OK | renderToString |
| https://docs.solidjs.com/reference/rendering/render-to-stream | OK | renderToStream |
| https://docs.solidjs.com/reference/rendering/hydrate | OK | hydrate |
| https://docs.solidjs.com/reference/rendering/hydration-script | OK | HydrationScript |
| https://github.com/solidjs-community/solid-primitives | OK | Community primitives |
| https://github.com/solidjs/solid | OK | Main repo |
| https://github.com/solidjs/solid/releases | OK | Release notes |
| https://docs.solidjs.com/solid-start/building-your-application/api-routes | OK | API routes |
| https://docs.solidjs.com/concepts/ssr | 404 | URL may have changed |
