# Custom React Hooks — Guide & Examples

## What makes a hook "custom"?

A custom hook is just a regular JavaScript function that:
1. Its name starts with `use` (so React's linter and other hooks can track it correctly).
2. It calls one or more built-in hooks (`useState`, `useEffect`, etc.) internally.
3. It extracts and reuses **stateful logic** between components — the state itself is NOT shared between components that use the hook; each call gets its own independent instance.

```jsx
// The pattern:
function useSomething(args) {
  const [state, setState] = useState(...);
  useEffect(() => { /* ... */ }, []);
  return state; // or an object/array of values and functions
}
```

Below are commonly-needed custom hooks, from simple to more advanced.

---

## 1. useToggle
Toggles a boolean value — great for modals, sidebars, checkboxes.

```jsx
import { useState, useCallback } from 'react';

function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);
  const toggle = useCallback(() => setValue(v => !v), []);
  return [value, toggle];
}

// Usage
function Modal() {
  const [isOpen, toggleOpen] = useToggle(false);
  return (
    <>
      <button onClick={toggleOpen}>{isOpen ? 'Close' : 'Open'} modal</button>
      {isOpen && <div className="modal">Modal content</div>}
    </>
  );
}
```

---

## 2. useLocalStorage
Syncs a piece of state with `localStorage`, so it persists across page reloads.

```jsx
import { useState, useEffect } from 'react';

function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    try {
      const stored = window.localStorage.getItem(key);
      return stored !== null ? JSON.parse(stored) : initialValue;
    } catch {
      return initialValue;
    }
  });

  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch (err) {
      console.error('Failed to save to localStorage', err);
    }
  }, [key, value]);

  return [value, setValue];
}

// Usage
function SettingsPanel() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');
  return (
    <select value={theme} onChange={e => setTheme(e.target.value)}>
      <option value="light">Light</option>
      <option value="dark">Dark</option>
    </select>
  );
}
```

---

## 3. useDebounce
Delays updating a value until the user has stopped changing it for a given time — ideal for search inputs to avoid firing a request on every keystroke.

```jsx
import { useState, useEffect } from 'react';

function useDebounce(value, delayMs = 500) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delayMs);
    return () => clearTimeout(timer); // cancel if value changes before delay ends
  }, [value, delayMs]);

  return debouncedValue;
}

// Usage
function SearchBox() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 400);

  useEffect(() => {
    if (debouncedQuery) {
      fetch(`/api/search?q=${debouncedQuery}`);
    }
  }, [debouncedQuery]);

  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

---

## 4. useFetch
Encapsulates data fetching with loading/error/data state, and cancels stale requests.

```jsx
import { useState, useEffect } from 'react';

function useFetch(url, options) {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);
    setError(null);

    fetch(url, { ...options, signal: controller.signal })
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(json => setData(json))
      .catch(err => {
        if (err.name !== 'AbortError') setError(err);
      })
      .finally(() => setLoading(false));

    return () => controller.abort(); // cancel on unmount or url change
  }, [url]);

  return { data, error, loading };
}

// Usage
function UserList() {
  const { data, error, loading } = useFetch('/api/users');

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;
  return <ul>{data.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

---

## 5. usePrevious
Keeps track of a value from the previous render — useful for comparisons (e.g. "did this prop just change?").

```jsx
import { useRef, useEffect } from 'react';

function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value; // updates AFTER render, so it lags by one render
  }, [value]);
  return ref.current;
}

// Usage
function Counter({ count }) {
  const prevCount = usePrevious(count);
  return (
    <p>Now: {count}, before: {prevCount ?? 'N/A'}</p>
  );
}
```

---

## 6. useWindowSize
Tracks the browser window's width/height and updates on resize.

```jsx
import { useState, useEffect } from 'react';

function useWindowSize() {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight,
  });

  useEffect(() => {
    function handleResize() {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    }
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return size;
}

// Usage
function ResponsiveComponent() {
  const { width } = useWindowSize();
  return <p>{width < 768 ? 'Mobile view' : 'Desktop view'}</p>;
}
```

---

## 7. useOnClickOutside
Detects clicks outside a referenced element — common for closing dropdowns/modals.

```jsx
import { useEffect } from 'react';

function useOnClickOutside(ref, handler) {
  useEffect(() => {
    function listener(event) {
      if (!ref.current || ref.current.contains(event.target)) return;
      handler(event);
    }
    document.addEventListener('mousedown', listener);
    return () => document.removeEventListener('mousedown', listener);
  }, [ref, handler]);
}

// Usage
function Dropdown() {
  const [open, setOpen] = useState(true);
  const menuRef = useRef(null);

  useOnClickOutside(menuRef, () => setOpen(false));

  return open ? <div ref={menuRef} className="menu">Menu items</div> : null;
}
```

---

## 8. useInterval
A declarative wrapper around `setInterval` that handles stale closures correctly (based on Dan Abramov's well-known pattern).

```jsx
import { useEffect, useRef } from 'react';

function useInterval(callback, delayMs) {
  const savedCallback = useRef(callback);

  useEffect(() => {
    savedCallback.current = callback; // always keep latest callback
  }, [callback]);

  useEffect(() => {
    if (delayMs === null) return; // pass null to pause
    const id = setInterval(() => savedCallback.current(), delayMs);
    return () => clearInterval(id);
  }, [delayMs]);
}

// Usage
function Clock() {
  const [time, setTime] = useState(new Date());
  useInterval(() => setTime(new Date()), 1000);
  return <p>{time.toLocaleTimeString()}</p>;
}
```

---

## 9. useMediaQuery
Reactively tracks whether a CSS media query currently matches.

```jsx
import { useState, useEffect } from 'react';

function useMediaQuery(query) {
  const [matches, setMatches] = useState(() => window.matchMedia(query).matches);

  useEffect(() => {
    const mediaQueryList = window.matchMedia(query);
    const listener = (event) => setMatches(event.matches);
    mediaQueryList.addEventListener('change', listener);
    return () => mediaQueryList.removeEventListener('change', listener);
  }, [query]);

  return matches;
}

// Usage
function Layout() {
  const isDarkMode = useMediaQuery('(prefers-color-scheme: dark)');
  return <div className={isDarkMode ? 'dark' : 'light'}>Content</div>;
}
```

---

## 10. useForm
A lightweight form-state hook handling values, changes, and validation errors — a mini version of what libraries like Formik/React Hook Form do.

```jsx
import { useState, useCallback } from 'react';

function useForm(initialValues, validate) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});

  const handleChange = useCallback((e) => {
    const { name, value } = e.target;
    setValues(prev => ({ ...prev, [name]: value }));
  }, []);

  const handleSubmit = useCallback((onSubmit) => (e) => {
    e.preventDefault();
    const validationErrors = validate ? validate(values) : {};
    setErrors(validationErrors);
    if (Object.keys(validationErrors).length === 0) {
      onSubmit(values);
    }
  }, [values, validate]);

  return { values, errors, handleChange, handleSubmit };
}

// Usage
function LoginForm() {
  const validate = (vals) => {
    const errs = {};
    if (!vals.email) errs.email = 'Email is required';
    return errs;
  };

  const { values, errors, handleChange, handleSubmit } = useForm({ email: '', password: '' }, validate);

  return (
    <form onSubmit={handleSubmit((vals) => console.log('Submitting', vals))}>
      <input name="email" value={values.email} onChange={handleChange} />
      {errors.email && <p>{errors.email}</p>}
      <input name="password" type="password" value={values.password} onChange={handleChange} />
      <button type="submit">Log in</button>
    </form>
  );
}
```

---

## 11. useCounter
Simple numeric counter with increment/decrement/reset — good "hello world" example for teaching the pattern.

```jsx
import { useState, useCallback } from 'react';

function useCounter(initialValue = 0, step = 1) {
  const [count, setCount] = useState(initialValue);

  const increment = useCallback(() => setCount(c => c + step), [step]);
  const decrement = useCallback(() => setCount(c => c - step), [step]);
  const reset = useCallback(() => setCount(initialValue), [initialValue]);

  return { count, increment, decrement, reset };
}

// Usage
function Counter() {
  const { count, increment, decrement, reset } = useCounter(0, 2);
  return (
    <>
      <p>{count}</p>
      <button onClick={increment}>+2</button>
      <button onClick={decrement}>-2</button>
      <button onClick={reset}>Reset</button>
    </>
  );
}
```

---

## 12. useAsync
Runs any async function and tracks its status/value/error — more general-purpose than `useFetch`.

```jsx
import { useState, useCallback, useEffect } from 'react';

function useAsync(asyncFn, immediate = true) {
  const [status, setStatus] = useState('idle'); // idle | pending | success | error
  const [value, setValue] = useState(null);
  const [error, setError] = useState(null);

  const execute = useCallback((...args) => {
    setStatus('pending');
    setValue(null);
    setError(null);
    return asyncFn(...args)
      .then(response => {
        setValue(response);
        setStatus('success');
      })
      .catch(err => {
        setError(err);
        setStatus('error');
      });
  }, [asyncFn]);

  useEffect(() => {
    if (immediate) execute();
  }, [immediate, execute]);

  return { execute, status, value, error };
}

// Usage
function SaveButton({ id }) {
  const { execute, status } = useAsync(() => api.saveItem(id), false); // don't run immediately

  return (
    <button onClick={execute} disabled={status === 'pending'}>
      {status === 'pending' ? 'Saving...' : 'Save'}
    </button>
  );
}
```

---

## Rules & best practices for writing custom hooks

- **Naming**: always prefix with `use` — React's ESLint plugin relies on this to enforce the Rules of Hooks.
- **No shared state**: each component calling the hook gets its own independent state — a custom hook is not a global store (use Context or a state library for that).
- **Follow the Rules of Hooks inside them too**: don't call hooks conditionally or in loops within your custom hook.
- **Return what's useful**: an array (`[value, setValue]`) suggests a small, positional API like `useState`; an object (`{ data, loading, error }`) is better for hooks with several named values, since object destructuring doesn't depend on order.
- **Clean up after yourself**: any subscription, timer, or event listener started in an effect should be torn down in its cleanup function.
- **Memoize returned functions when they'll be used as effect dependencies or passed to memoized children** (as shown with `useCallback` in several examples above).
- **Keep them focused**: one hook, one concern. Compose multiple small hooks together rather than building one hook that does everything.

---

## Quick reference table

| Hook | Solves |
|---|---|
| useToggle | Boolean on/off state |
| useLocalStorage | Persist state across reloads |
| useDebounce | Delay reacting to fast-changing values |
| useFetch | Data fetching with loading/error state |
| usePrevious | Access previous render's value |
| useWindowSize | Track viewport dimensions |
| useOnClickOutside | Detect outside clicks (dropdowns/modals) |
| useInterval | Declarative setInterval |
| useMediaQuery | React to CSS media query changes |
| useForm | Form values, changes, validation |
| useCounter | Numeric increment/decrement state |
| useAsync | Run and track any async function |
