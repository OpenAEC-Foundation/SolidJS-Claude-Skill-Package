# SolidJS JSX, Components & Control Flow Research

> Research date: 2026-03-19
> Sources: docs.solidjs.com (concepts + reference sections)
> Status: Complete — 16/18 URLs fetched successfully (splitProps/mergeProps reference pages returned 404, covered via concepts/props page)

---

## 1. JSX Compilation Model

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

## 2. Component Basics

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

## 3. Props Handling

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

## 4. Children

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

## 5. Control Flow Components

### 5.1 Show

**Source**: https://docs.solidjs.com/reference/components/show

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

### 5.2 For

**Source**: https://docs.solidjs.com/reference/components/for

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

### 5.3 Index

**Source**: https://docs.solidjs.com/reference/components/index-component

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

### 5.4 Switch/Match

**Source**: https://docs.solidjs.com/reference/components/switch-and-match

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

### 5.5 Dynamic

**Source**: https://docs.solidjs.com/reference/components/dynamic

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

### 5.6 Portal

**Source**: https://docs.solidjs.com/reference/components/portal

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

### 5.7 ErrorBoundary

**Source**: https://docs.solidjs.com/reference/components/error-boundary

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

### 5.8 Suspense

**Source**: https://docs.solidjs.com/reference/components/suspense

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

## 6. Event Handling

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

## 7. Refs and Directives

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

## 8. React vs SolidJS Anti-Patterns (JSX/Components)

### 8.1 Array.map vs `<For>`

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

### 8.2 Ternary vs `<Show>`

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

### 8.3 switch/case vs `<Switch>`/`<Match>`

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

### 8.4 Destructuring Props

```typescript
// WRONG (React pattern):
function Greeting({ name, age }: Props) {
  return <p>{name} is {age}</p>;  // Frozen values
}

// CORRECT (SolidJS):
function Greeting(props: Props) {
  return <p>{props.name} is {props.age}</p>;  // Reactive
}

// CORRECT (SolidJS — when you need to separate props):
function Greeting(props: Props) {
  const [local, rest] = splitProps(props, ["name", "age"]);
  return <p>{local.name} is {local.age}</p>;
}
```

### 8.5 Early Returns in Component Body

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

### 8.6 key Prop (React) vs `<For>` Keyed-by-Reference

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

### 8.7 useRef vs let ref / ref Callback

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

### 8.8 React.createElement vs Direct DOM

```typescript
// React: JSX → React.createElement() → Virtual DOM → Reconciliation → DOM
// SolidJS: JSX → Compiled DOM creation → Direct DOM nodes + reactive effects

// This means in SolidJS:
// - No virtual DOM overhead
// - No diffing algorithm
// - Updates are surgical — only the exact text node/attribute changes
```

### 8.9 Re-render Assumption

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

### 8.10 children as ReactNode vs children() Helper

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

## Version Notes (1.x vs 2.x)

All components and APIs documented above are available since **SolidJS 1.0**. The documentation reviewed (docs.solidjs.com, March 2026) does not indicate breaking changes to these APIs between 1.x and 2.x. Key observations:

- `Show`, `For`, `Index`, `Switch`, `Match`, `Dynamic`, `Portal`, `ErrorBoundary`, `Suspense` — stable API since 1.0
- `splitProps`, `mergeProps`, `children` — stable API since 1.0
- `keyed` prop on `<Show>` — available since 1.0
- Directive system (`use:`) — stable since 1.0
- Event delegation list — stable since 1.0

**Note**: SolidJS 2.0 introduces changes primarily around the reactivity system internals and compilation improvements, but the component-level API surface documented here remains compatible.
