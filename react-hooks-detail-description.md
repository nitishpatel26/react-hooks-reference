# Complete React Hooks Reference (Detailed)

---

## 1. useState

**What it does:** Adds a piece of local state to a function component. Calling the setter triggers a re-render with the new value.

**Signature:** `const [state, setState] = useState(initialValue)`

- `initialValue` — the starting value, or a function that returns it (lazy initialization, useful if computing the initial value is expensive).
- `state` — current value.
- `setState` — function to update it. Accepts either a new value, or an updater function `(prevState) => newState`.

**Key details:**
- Updates are **asynchronous/batched** — don't expect `state` to reflect the new value immediately after calling `setState`.
- If the new state depends on the previous state, use the updater function form to avoid stale-state bugs (especially inside loops, timeouts, or rapid clicks).
- React bails out of re-rendering if the new value is `Object.is`-equal to the old one.

```jsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  // Safe against stale closures because it uses the updater form
  const incrementTwice = () => {
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
  };

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={incrementTwice}>+2</button>
    </div>
  );
}

// Lazy initialization example
function ExpensiveInit() {
  const [data] = useState(() => computeExpensiveValue()); // runs once, not every render
  return <p>{data}</p>;
}
```

**Common pitfalls:** mutating state directly (e.g. `arr.push()` then calling `setArr(arr)`), which won't trigger a re-render since the reference didn't change. Always create new arrays/objects: `setArr([...arr, newItem])`.

---

## 2. useEffect

**What it does:** Runs "side effects" — code that reaches outside of rendering, like fetching data, subscribing to events, manually touching the DOM, or logging — after the render is committed to the screen.

**Signature:** `useEffect(setupFunction, dependencyArray?)`

- `setupFunction` — runs after render; can optionally return a cleanup function.
- `dependencyArray`:
  - Omitted → runs after **every** render.
  - `[]` → runs **once**, after the first render (mount).
  - `[a, b]` → runs whenever `a` or `b` change.
- The cleanup function runs before the effect re-runs, and when the component unmounts.

```jsx
import { useState, useEffect } from 'react';

function ChatRoom({ roomId }) {
  const [messages, setMessages] = useState([]);

  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    connection.on('message', (msg) => {
      setMessages(prev => [...prev, msg]);
    });

    // Cleanup: disconnect when roomId changes or component unmounts
    return () => connection.disconnect();
  }, [roomId]); // re-run only when roomId changes

  return <ul>{messages.map((m, i) => <li key={i}>{m}</li>)}</ul>;
}
```

**Common pitfalls:**
- Forgetting the dependency array causes infinite loops if the effect itself sets state that's a dependency.
- Missing dependencies (values used inside the effect but not listed) lead to stale-closure bugs — ESLint's `react-hooks/exhaustive-deps` rule catches most of these.
- Not returning a cleanup function for subscriptions/timers/listeners causes memory leaks.
- Don't use `useEffect` for things that can be computed during render (derived state) — that's wasted work and causes an extra render.

---

## 3. useContext

**What it does:** Reads the current value of a Context, letting deeply nested components access shared data (theme, auth user, locale, etc.) without manually passing props through every level ("prop drilling").

**Signature:** `const value = useContext(SomeContext)`

- Must be paired with a `<SomeContext.Provider value={...}>` higher up the tree.
- If no Provider exists above it, the hook returns the `defaultValue` passed to `createContext()`.
- Re-renders the component whenever the Provider's `value` changes.

```jsx
import { createContext, useContext, useState } from 'react';

const ThemeContext = createContext('light');

function ThemedButton() {
  const theme = useContext(ThemeContext); // reads nearest Provider's value
  return <button className={`btn-${theme}`}>Click me</button>;
}

function App() {
  const [theme, setTheme] = useState('light');
  return (
    <ThemeContext.Provider value={theme}>
      <ThemedButton />
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
        Toggle theme
      </button>
    </ThemeContext.Provider>
  );
}
```

**Common pitfalls:** Every consumer re-renders whenever the Provider's value changes — even if the consumer only cares about part of it. For large/frequently-changing contexts, split into multiple contexts or memoize the value object (`useMemo`) to avoid unnecessary re-renders.

---

## 4. useReducer

**What it does:** An alternative to `useState` for managing more complex state logic — especially when the next state depends on multiple sub-values or the previous state in non-trivial ways (similar to Redux's reducer pattern, but local to a component).

**Signature:** `const [state, dispatch] = useReducer(reducer, initialState, init?)`

- `reducer(state, action)` — pure function returning the new state.
- `initialState` — starting state.
- `init` (optional) — lazy initializer function, called as `init(initialState)`.
- `dispatch(action)` — sends an action object to the reducer to compute the next state.

```jsx
import { useReducer } from 'react';

const initialState = { count: 0, step: 1 };

function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return { ...state, count: state.count + state.step };
    case 'decrement':
      return { ...state, count: state.count - state.step };
    case 'setStep':
      return { ...state, step: action.payload };
    case 'reset':
      return initialState;
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <div>
      <p>Count: {state.count} (step: {state.step})</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <input
        type="number"
        value={state.step}
        onChange={e => dispatch({ type: 'setStep', payload: Number(e.target.value) })}
      />
      <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
    </div>
  );
}
```

**When to prefer over useState:** state has multiple sub-values that update together, next state depends on previous state in complex ways, or you want to centralize update logic for easier testing.

---

## 5. useCallback

**What it does:** Returns a **memoized version of a function** that only changes if one of its dependencies changes. Prevents unnecessary re-creation of functions on every render — important when passing callbacks to child components wrapped in `React.memo`, or as dependencies of other hooks like `useEffect`.

**Signature:** `const memoizedFn = useCallback(fn, dependencyArray)`

```jsx
import { useState, useCallback, memo } from 'react';

const ExpensiveChild = memo(function ExpensiveChild({ onClick }) {
  console.log('Child rendered');
  return <button onClick={onClick}>Click child</button>;
});

function Parent() {
  const [count, setCount] = useState(0);
  const [other, setOther] = useState(0);

  // Without useCallback, a new function is created every render,
  // causing ExpensiveChild to re-render even though nothing it cares about changed.
  const handleClick = useCallback(() => {
    console.log('Count is', count);
  }, [count]);

  return (
    <>
      <ExpensiveChild onClick={handleClick} />
      <button onClick={() => setOther(o => o + 1)}>Re-render parent ({other})</button>
    </>
  );
}
```

**Common pitfalls:** Overusing `useCallback` everywhere adds memory/comparison overhead without benefit — it only pays off when the function is passed to a memoized child or used as a dependency elsewhere. Don't reach for it by default; profile first.

---

## 6. useMemo

**What it does:** Memoizes the **result of a computation**, recalculating it only when dependencies change — useful for expensive calculations (filtering/sorting large lists, heavy math) or to keep referential equality of objects/arrays passed to memoized children.

**Signature:** `const memoizedValue = useMemo(calculateFn, dependencyArray)`

```jsx
import { useState, useMemo } from 'react';

function ProductList({ products, searchTerm }) {
  const [sortAsc, setSortAsc] = useState(true);

  const filteredAndSorted = useMemo(() => {
    console.log('Recalculating...'); // only logs when deps change
    const filtered = products.filter(p =>
      p.name.toLowerCase().includes(searchTerm.toLowerCase())
    );
    return filtered.sort((a, b) =>
      sortAsc ? a.price - b.price : b.price - a.price
    );
  }, [products, searchTerm, sortAsc]);

  return (
    <>
      <button onClick={() => setSortAsc(s => !s)}>Toggle sort</button>
      <ul>{filteredAndSorted.map(p => <li key={p.id}>{p.name} - ${p.price}</li>)}</ul>
    </>
  );
}
```

**Common pitfalls:** Like `useCallback`, don't wrap every value in `useMemo` — it has its own overhead. Reserve it for genuinely expensive computations or to stabilize references for dependency arrays/memoized children.

---

## 7. useRef

**What it does:** Creates a mutable object (`{ current: value }`) that persists across renders **without** causing a re-render when changed. Two main uses: (1) holding a reference to a DOM element, (2) storing any mutable value that shouldn't trigger re-renders (timers, previous values, instance variables).

**Signature:** `const ref = useRef(initialValue)`

```jsx
import { useRef, useEffect, useState } from 'react';

function StopwatchWithDomRef() {
  const inputRef = useRef(null); // DOM ref use case

  useEffect(() => {
    inputRef.current.focus(); // runs after mount
  }, []);

  return <input ref={inputRef} placeholder="Auto-focused" />;
}

function RenderCounter() {
  const renderCount = useRef(0); // mutable value use case
  renderCount.current += 1; // does NOT cause a re-render

  return <p>This component has rendered {renderCount.current} times</p>;
}
```

**Common pitfalls:** Changing `ref.current` does **not** trigger a re-render — if you need the UI to update, use `useState` instead. Don't read/write `ref.current` during rendering (outside effects/handlers) — it can cause inconsistent behavior in concurrent rendering.

---

## 8. useImperativeHandle

**What it does:** Customizes what a parent sees when it attaches a `ref` to a child component wrapped in `forwardRef`. Lets you expose a controlled, minimal API (like `focus()` or `reset()`) instead of the raw DOM node.

**Signature:** `useImperativeHandle(ref, createHandle, dependencyArray?)`

```jsx
import { forwardRef, useImperativeHandle, useRef, useState } from 'react';

const FancyInput = forwardRef(function FancyInput(props, ref) {
  const inputRef = useRef();
  const [value, setValue] = useState('');

  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    clear: () => setValue(''),
  }));

  return (
    <input
      ref={inputRef}
      value={value}
      onChange={e => setValue(e.target.value)}
    />
  );
});

function Parent() {
  const fancyInputRef = useRef();

  return (
    <>
      <FancyInput ref={fancyInputRef} />
      <button onClick={() => fancyInputRef.current.focus()}>Focus</button>
      <button onClick={() => fancyInputRef.current.clear()}>Clear</button>
    </>
  );
}
```

**Common pitfalls:** This is a relatively rare/advanced hook — most components shouldn't expose imperative APIs. Use it sparingly, mainly for reusable UI library components (inputs, modals, video players).

---

## 9. useLayoutEffect

**What it does:** Identical to `useEffect`, but fires **synchronously** after DOM mutations and **before** the browser paints the screen. Use it when you need to measure the DOM (size, position) or make DOM changes that must be visible immediately, to avoid a visual flicker.

**Signature:** `useLayoutEffect(setupFunction, dependencyArray?)`

```jsx
import { useLayoutEffect, useRef, useState } from 'react';

function Tooltip({ text }) {
  const tooltipRef = useRef();
  const [height, setHeight] = useState(0);

  useLayoutEffect(() => {
    // Measures synchronously before paint, avoiding a flash of wrong position
    const rect = tooltipRef.current.getBoundingClientRect();
    setHeight(rect.height);
  }, [text]);

  return (
    <div ref={tooltipRef} style={{ marginTop: -height }}>
      {text}
    </div>
  );
}
```

**Common pitfalls:** Blocks the browser from painting until it finishes, so overusing it (or doing expensive work inside it) can hurt performance. Default to `useEffect`; only reach for `useLayoutEffect` when you see visual flicker from layout reads/writes. It also doesn't run on the server (no-op during SSR), which can trigger warnings — guard with checks if needed.

---

## 10. useDebugValue

**What it does:** Adds a readable label for a **custom hook's** value in React DevTools. Purely a developer-experience tool — it has no effect on the app's behavior or rendering.

**Signature:** `useDebugValue(value, formatFn?)`

```jsx
import { useState, useEffect, useDebugValue } from 'react';

function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useEffect(() => {
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);
    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);
    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);

  // Shows "Online" or "Offline" next to this hook in React DevTools
  useDebugValue(isOnline ? 'Online' : 'Offline');

  return isOnline;
}
```

**Common pitfalls:** Only use it inside custom hooks meant to be shared/reused (e.g. in a library or across a large codebase) — it's unnecessary in one-off application-specific hooks. If formatting the debug value is expensive, pass a second `formatFn` argument so it only runs when DevTools is actually open.

---

## 11. useDeferredValue

**What it does:** Lets you defer re-rendering a non-urgent part of the UI. React renders with the old value first, then re-renders in the background with the new one once it's ready — keeping typing/interactions responsive even if a downstream component is slow to render.

**Signature:** `const deferredValue = useDeferredValue(value)`

```jsx
import { useState, useDeferredValue, useMemo } from 'react';

function SearchPage({ allItems }) {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);

  // Expensive filter uses the deferred (possibly lagging) value,
  // so typing in the input stays smooth even with thousands of items.
  const results = useMemo(
    () => allItems.filter(item => item.includes(deferredQuery)),
    [allItems, deferredQuery]
  );

  const isStale = query !== deferredQuery;

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <ul style={{ opacity: isStale ? 0.5 : 1 }}>
        {results.map(r => <li key={r}>{r}</li>)}
      </ul>
    </>
  );
}
```

**Common pitfalls:** It doesn't debounce (no fixed delay) — it's tied to render performance, not time. It's a performance escape hatch for concurrent rendering, not a replacement for `useState`; only use it when you have a genuine slow-render problem.

---

## 12. useTransition

**What it does:** Lets you mark a state update as a **low-priority "transition"**, so React can keep the UI responsive (e.g. keep an input working) while a slower update (like switching tabs, filtering a big list) happens in the background. Provides an `isPending` flag to show loading indicators.

**Signature:** `const [isPending, startTransition] = useTransition()`

```jsx
import { useState, useTransition } from 'react';

function TabContainer() {
  const [tab, setTab] = useState('about');
  const [isPending, startTransition] = useTransition();

  function selectTab(nextTab) {
    startTransition(() => {
      setTab(nextTab); // marked as non-urgent; UI stays responsive
    });
  }

  return (
    <>
      <button onClick={() => selectTab('about')}>About</button>
      <button onClick={() => selectTab('posts')}>Posts (slow)</button>
      {isPending && <span> Loading...</span>}
      <div style={{ opacity: isPending ? 0.6 : 1 }}>
        {tab === 'about' ? <AboutTab /> : <PostsTab />}
      </div>
    </>
  );
}
```

**Common pitfalls:** Only state updates wrapped inside `startTransition` are deprioritized — updates outside it (like typing in an input) still run at normal priority, which is exactly the point. Don't use it for updates that must feel instant (e.g. text input value itself).

---

## 13. useId

**What it does:** Generates a unique, stable ID string that's consistent between server and client renders — solving the problem of hydration mismatches you'd get from `Math.random()` or a simple counter. Mainly used for linking form elements via `htmlFor`/`id` for accessibility.

**Signature:** `const id = useId()`

```jsx
import { useId } from 'react';

function PasswordField() {
  const id = useId();
  const hintId = `${id}-hint`;

  return (
    <div>
      <label htmlFor={id}>Password</label>
      <input id={id} type="password" aria-describedby={hintId} />
      <p id={hintId}>Must be at least 8 characters</p>
    </div>
  );
}

// Rendering <PasswordField /> twice gives each instance a distinct, stable id
```

**Common pitfalls:** `useId` is **not** meant for generating keys in a list (use stable data-derived keys for that instead) — it's for accessibility attribute wiring within a single component instance.

---

## 14. useSyncExternalStore

**What it does:** Safely subscribes a component to an external data source that lives outside React's state system (e.g. a browser API, a third-party state library, or a custom store), ensuring the UI stays consistent even under concurrent rendering.

**Signature:** `const state = useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?)`

- `subscribe(callback)` — registers a listener for store changes; must return an unsubscribe function.
- `getSnapshot()` — returns the current value of the store.
- `getServerSnapshot()` (optional) — value to use during server-side rendering.

```jsx
import { useSyncExternalStore } from 'react';

function subscribe(callback) {
  window.addEventListener('online', callback);
  window.addEventListener('offline', callback);
  return () => {
    window.removeEventListener('online', callback);
    window.removeEventListener('offline', callback);
  };
}

function getSnapshot() {
  return navigator.onLine;
}

function useOnlineStatus() {
  return useSyncExternalStore(subscribe, getSnapshot, () => true); // true = server default
}

function StatusBadge() {
  const isOnline = useOnlineStatus();
  return <span>{isOnline ? '🟢 Online' : '🔴 Offline'}</span>;
}
```

**Common pitfalls:** Most app developers won't need this directly — it's primarily used internally by state-management libraries (Redux, Zustand, Jotai, etc.) to integrate safely with React. Reach for `useState`/`useEffect` first unless you're specifically bridging to an external store.

---

## 15. useInsertionEffect

**What it does:** Fires **before** any DOM mutations occur (even before `useLayoutEffect`), designed specifically for CSS-in-JS library authors to inject `<style>` tags before the browser needs to compute layout — avoiding performance issues from injecting styles at the wrong time.

**Signature:** `useInsertionEffect(setupFunction, dependencyArray?)`

```jsx
import { useInsertionEffect } from 'react';

function useCSS(css) {
  useInsertionEffect(() => {
    const styleTag = document.createElement('style');
    styleTag.textContent = css;
    document.head.appendChild(styleTag);
    return () => document.head.removeChild(styleTag);
  }, [css]);
}

function StyledBox() {
  useCSS('.box { padding: 20px; background: papayawhip; }');
  return <div className="box">Styled content</div>;
}
```

**Common pitfalls:** This is a very niche, low-level hook. Application developers almost never need it directly — it exists for library authors (styled-components, Emotion, etc.). You cannot read refs or trigger state updates inside it.

---

## 16. useOptimistic (React 19)

**What it does:** Lets you show an "optimistic" UI state immediately while an async action (e.g. a server request) is still in flight, then automatically reconciles with the real result once it resolves (or rolls back on error).

**Signature:** `const [optimisticState, addOptimistic] = useOptimistic(state, updateFn)`

```jsx
import { useOptimistic, useState } from 'react';

function Thread({ initialMessages, sendMessage }) {
  const [messages, setMessages] = useState(initialMessages);
  const [optimisticMessages, addOptimisticMessage] = useOptimistic(
    messages,
    (currentMessages, newMessage) => [
      ...currentMessages,
      { text: newMessage, sending: true }
    ]
  );

  async function formAction(formData) {
    const text = formData.get('message');
    addOptimisticMessage(text); // shows instantly
    const saved = await sendMessage(text); // real request
    setMessages(prev => [...prev, saved]); // reconcile with server truth
  }

  return (
    <>
      {optimisticMessages.map((m, i) => (
        <p key={i}>{m.text} {m.sending && <em>(sending...)</em>}</p>
      ))}
      <form action={formAction}>
        <input name="message" />
        <button type="submit">Send</button>
      </form>
    </>
  );
}
```

**Common pitfalls:** Requires React 19+. The optimistic update is automatically discarded and replaced once the real state updates — don't try to manually clear it. Best paired with Actions (`<form action={...}>`) and `useActionState`.

---

## 17. useActionState (React 19)

**What it does:** Manages state that's updated by a form action — tracks the pending status and the returned result/error automatically, removing the need for manual `useState` + `useEffect` wiring around form submissions.

**Signature:** `const [state, formAction, isPending] = useActionState(actionFn, initialState, permalink?)`

- `actionFn(previousState, formData)` — async function; its return value becomes the new state.
- `initialState` — value before the first submission.
- `isPending` — `true` while the action is running.

```jsx
import { useActionState } from 'react';

async function updateName(previousState, formData) {
  const name = formData.get('name');
  if (!name || name.trim() === '') {
    return { error: 'Name cannot be empty' };
  }
  await saveNameToServer(name); // pretend API call
  return { success: true, name };
}

function NameForm() {
  const [state, formAction, isPending] = useActionState(updateName, {});

  return (
    <form action={formAction}>
      <input name="name" disabled={isPending} />
      <button disabled={isPending}>
        {isPending ? 'Saving...' : 'Save'}
      </button>
      {state.error && <p style={{ color: 'red' }}>{state.error}</p>}
      {state.success && <p style={{ color: 'green' }}>Saved as {state.name}!</p>}
    </form>
  );
}
```

**Common pitfalls:** Requires React 19+ and is designed to work with the native `<form action={...}>` pattern. Don't call `formAction` manually with arbitrary arguments outside a form submission — it expects `FormData`.

---

## 18. use (React 19)

**What it does:** Reads the value of a resource — a Promise or a Context — and, unlike other hooks, **can be called conditionally** (inside `if` statements, loops, after early returns). When passed a pending Promise, it suspends the component (works with `<Suspense>`) until the Promise resolves.

**Signature:** `const value = use(resource)`

```jsx
import { use, Suspense, createContext } from 'react';

// Reading a Promise
function UserProfile({ userPromise }) {
  const user = use(userPromise); // suspends here until resolved
  return <p>Welcome, {user.name}</p>;
}

function App({ userPromise }) {
  return (
    <Suspense fallback={<p>Loading user...</p>}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}

// Reading Context conditionally (not possible with useContext)
const ThemeContext = createContext('light');

function ConditionalThemedText({ show }) {
  if (!show) return null;
  const theme = use(ThemeContext); // fine to call after the early return
  return <p className={theme}>Themed text</p>;
}
```

**Common pitfalls:** Requires React 19+. The Promise passed to `use` should typically be created outside the component (e.g. in a data-fetching layer) and cached/stable — creating a new Promise on every render causes infinite re-fetching. Errors from a rejected Promise should be caught with an Error Boundary.

---

## Quick reference table

| Hook | Category | Purpose | Min. React version |
|---|---|---|---|
| useState | State | Local component state | 16.8 |
| useEffect | Effect | Side effects after render | 16.8 |
| useContext | Context | Consume context value | 16.8 |
| useReducer | State | Complex state logic | 16.8 |
| useCallback | Performance | Memoize functions | 16.8 |
| useMemo | Performance | Memoize computed values | 16.8 |
| useRef | Ref | Mutable values / DOM refs | 16.8 |
| useImperativeHandle | Ref | Customize exposed ref API | 16.8 |
| useLayoutEffect | Effect | Synchronous DOM effects | 16.8 |
| useDebugValue | Debug | Label custom hooks in DevTools | 16.8 |
| useDeferredValue | Concurrent | Defer non-urgent value updates | 18 |
| useTransition | Concurrent | Mark updates as low priority | 18 |
| useId | Utility | Unique, SSR-safe stable IDs | 18 |
| useSyncExternalStore | Store | Subscribe to external stores | 18 |
| useInsertionEffect | Effect | CSS-in-JS style injection | 18 |
| useOptimistic | React 19 | Optimistic UI updates | 19 |
| useActionState | React 19 | Form action state management | 19 |
| use | React 19 | Read promises/context, conditionally | 19 |
