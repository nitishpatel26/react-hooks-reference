# Complete React Hooks Reference

Here's a complete reference covering all 18 built-in React hooks (including the React 19 additions — useOptimistic, useActionState, and use), each with a working code example and a quick-reference table at the end.

A few highlights on how they group:

State: `useState`, `useReducer`
Effects: `useEffect`, `useLayoutEffect`, `useInsertionEffect`
Performance: `useCallback`, `useMemo`
Refs/DOM: `useRef`, `useImperativeHandle`
Concurrent UI: `useTransition`, `useDeferredValue`
React 19: `useOptimistic`, `useActionState`, use`

## 1. useState
Manages local state in a function component.

```jsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}
```

## 2. useEffect
Runs side effects (data fetching, subscriptions, DOM updates) after render.

```jsx
import { useState, useEffect } from 'react';

function Timer() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => setSeconds(s => s + 1), 1000);
    return () => clearInterval(interval); // cleanup
  }, []); // empty deps = run once on mount

  return <p>Seconds: {seconds}</p>;
}
```

## 3. useContext
Reads a value from React Context without wrapping in a Consumer.

```jsx
import { createContext, useContext } from 'react';

const ThemeContext = createContext('light');

function ThemedButton() {
  const theme = useContext(ThemeContext);
  return <button className={theme}>Click me</button>;
}

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <ThemedButton />
    </ThemeContext.Provider>
  );
}
```

## 4. useReducer
Manages complex state logic using a reducer function (like Redux, but local).

```jsx
import { useReducer } from 'react';

function reducer(state, action) {
  switch (action.type) {
    case 'increment': return { count: state.count + 1 };
    case 'decrement': return { count: state.count - 1 };
    default: return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });

  return (
    <>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <span>{state.count}</span>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
    </>
  );
}
```

## 5. useCallback
Memoizes a function so it isn't recreated on every render (useful for passing callbacks to optimized child components).

```jsx
import { useState, useCallback } from 'react';

function Parent() {
  const [count, setCount] = useState(0);

  const handleClick = useCallback(() => {
    console.log('Clicked, count is', count);
  }, [count]);

  return <Child onClick={handleClick} />;
}
```

## 6. useMemo
Memoizes an expensive computed value so it's only recalculated when dependencies change.

```jsx
import { useMemo } from 'react';

function ExpensiveList({ items, filter }) {
  const filteredItems = useMemo(() => {
    return items.filter(item => item.includes(filter));
  }, [items, filter]);

  return <ul>{filteredItems.map(i => <li key={i}>{i}</li>)}</ul>;
}
```

## 7. useRef
Persists a mutable value across renders without causing re-renders; commonly used to access DOM nodes.

```jsx
import { useRef } from 'react';

function TextInput() {
  const inputRef = useRef(null);

  const focusInput = () => inputRef.current.focus();

  return (
    <>
      <input ref={inputRef} type="text" />
      <button onClick={focusInput}>Focus input</button>
    </>
  );
}
```

## 8. useImperativeHandle
Customizes the instance value exposed to parent components when using `ref` with `forwardRef`.

```jsx
import { forwardRef, useImperativeHandle, useRef } from 'react';

const FancyInput = forwardRef((props, ref) => {
  const inputRef = useRef();

  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    clear: () => (inputRef.current.value = '')
  }));

  return <input ref={inputRef} />;
});

// Parent: ref.current.focus() or ref.current.clear()
```

## 9. useLayoutEffect
Like `useEffect`, but fires synchronously after DOM mutations, before the browser paints. Use for reading/writing layout (e.g. measuring elements).

```jsx
import { useLayoutEffect, useRef, useState } from 'react';

function Box() {
  const boxRef = useRef();
  const [width, setWidth] = useState(0);

  useLayoutEffect(() => {
    setWidth(boxRef.current.getBoundingClientRect().width);
  }, []);

  return <div ref={boxRef}>Width: {width}px</div>;
}
```

## 10. useDebugValue
Displays a label for custom hooks in React DevTools. Only useful inside custom hooks.

```jsx
import { useState, useDebugValue } from 'react';

function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(true);
  useDebugValue(isOnline ? 'Online' : 'Offline');
  return isOnline;
}
```

## 11. useDeferredValue
Defers updating a value until more urgent updates have finished rendering — helps keep the UI responsive.

```jsx
import { useState, useDeferredValue } from 'react';

function SearchResults({ query }) {
  const deferredQuery = useDeferredValue(query);
  // Expensive list rendering uses the deferred (lagging) value
  return <List query={deferredQuery} />;
}
```

## 12. useTransition
Marks a state update as a non-urgent "transition," keeping the UI responsive during expensive updates.

```jsx
import { useState, useTransition } from 'react';

function TabContainer() {
  const [isPending, startTransition] = useTransition();
  const [tab, setTab] = useState('home');

  function selectTab(nextTab) {
    startTransition(() => {
      setTab(nextTab);
    });
  }

  return (
    <>
      {isPending && <span>Loading...</span>}
      <button onClick={() => selectTab('profile')}>Profile</button>
    </>
  );
}
```

## 13. useId
Generates a unique, stable ID — useful for accessibility attributes (e.g. linking labels and inputs).

```jsx
import { useId } from 'react';

function LabeledInput() {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>Name</label>
      <input id={id} type="text" />
    </>
  );
}
```

## 14. useSyncExternalStore
Subscribes to an external data store (outside React), useful for integrating with non-React state libraries.

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

function useOnlineStatus() {
  return useSyncExternalStore(subscribe, () => navigator.onLine);
}
```

## 15. useInsertionEffect
Fires before any DOM mutations, mainly designed for CSS-in-JS library authors to inject styles before layout reads happen. Rarely used directly by app developers.

```jsx
import { useInsertionEffect } from 'react';

function useDynamicStyle(css) {
  useInsertionEffect(() => {
    const style = document.createElement('style');
    style.textContent = css;
    document.head.appendChild(style);
    return () => document.head.removeChild(style);
  }, [css]);
}
```

## 16. useOptimistic (React 19)
Shows an optimistic UI state while an async action is in progress, then reconciles with the real result.

```jsx
import { useOptimistic } from 'react';

function Thread({ messages, sendMessage }) {
  const [optimisticMessages, addOptimisticMessage] = useOptimistic(
    messages,
    (state, newMessage) => [...state, { text: newMessage, sending: true }]
  );

  async function formAction(formData) {
    const text = formData.get('message');
    addOptimisticMessage(text);
    await sendMessage(text);
  }

  return (
    <>
      {optimisticMessages.map((m, i) => (
        <p key={i}>{m.text} {m.sending && '(sending...)'}</p>
      ))}
      <form action={formAction}>
        <input name="message" />
      </form>
    </>
  );
}
```

## 17. useActionState (React 19)
Manages state driven by a form action, including pending status and returned result.

```jsx
import { useActionState } from 'react';

async function updateName(prevState, formData) {
  const name = formData.get('name');
  if (!name) return { error: 'Name is required' };
  return { success: true, name };
}

function NameForm() {
  const [state, formAction, isPending] = useActionState(updateName, {});

  return (
    <form action={formAction}>
      <input name="name" />
      <button disabled={isPending}>Save</button>
      {state.error && <p>{state.error}</p>}
    </form>
  );
}
```

## 18. use (React 19)
Reads the value of a resource like a Promise or Context, and can be called conditionally (unlike other hooks).

```jsx
import { use, Suspense } from 'react';

function UserProfile({ userPromise }) {
  const user = use(userPromise); // suspends until resolved
  return <p>{user.name}</p>;
}

function App({ userPromise }) {
  return (
    <Suspense fallback={<p>Loading...</p>}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}
```

---

### Quick reference table

| Hook | Category | Purpose |
|---|---|---|
| useState | State | Local component state |
| useEffect | Effect | Side effects after render |
| useContext | Context | Consume context value |
| useReducer | State | Complex state logic |
| useCallback | Performance | Memoize functions |
| useMemo | Performance | Memoize computed values |
| useRef | Ref | Mutable values / DOM refs |
| useImperativeHandle | Ref | Customize exposed ref |
| useLayoutEffect | Effect | Synchronous DOM effects |
| useDebugValue | Debug | Label custom hooks in DevTools |
| useDeferredValue | Concurrent | Defer non-urgent value updates |
| useTransition | Concurrent | Mark updates as low priority |
| useId | Utility | Unique stable IDs |
| useSyncExternalStore | Store | Subscribe to external stores |
| useInsertionEffect | Effect | CSS-in-JS style injection |
| useOptimistic | React 19 | Optimistic UI updates |
| useActionState | React 19 | Form action state management |
| use | React 19 | Read promises/context, conditionally |

*Note: `useState` through `useSyncExternalStore` are stable in all modern React versions (16.8+/18). `useInsertionEffect` requires React 18+. `useOptimistic`, `useActionState`, and `use` require React 19+.*
