# React Context, Compiler, `use()`, Suspense & Cache

> These notes are based on the second Rootline session material and expanded with official React and MDN documentation.
>
> The original session connects several concepts together:
>
> **Provider → Context → Consumer → `useContext()` / `use()` → Promise → Suspense → Error Boundary → Cache**
>
> It also connects **stable references → memoization → React Compiler**.

---

# 1. Context API

## What problem does Context solve?

Normally, data moves through a React tree using props:

```text
App
 ↓
Layout
 ↓
Page
 ↓
Section
 ↓
Button
```

If `Button` needs some data from `App`, we may have to pass that data through components that don't actually need it:

```jsx
<App theme={theme}>
  <Layout theme={theme}>
    <Page theme={theme}>
      <Section theme={theme}>
        <Button theme={theme} />
      </Section>
    </Page>
  </Layout>
</App>
```

This is commonly called **prop drilling**.

Context provides another way:

```text
Provider
   │
   ├── Layout
   │    └── Page
   │         └── Section
   │              └── Button
   │
   └── all descendants can read the context
```

The intermediate components don't need to receive and forward the value.

React describes Context as a way for a parent to make information available to any component in the tree below it without explicitly passing it through props.

---

# 2. First Theme Provider

The first example in the session is based around a theme provider.

The important idea is:

```text
First Theme Provider
        │
        │ provides
        ▼
      Context
        │
        │ consumed by
        ▼
    Components
```

The Excalidraw material explicitly labels the first section as:

```text
First Theme Provider
```

and uses `FirstThemeProvider` as part of the diagram.

A simple implementation could look like:

```jsx
import { createContext, useState } from "react";

const ThemeContext = createContext("light");

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");

  return (
    <ThemeContext value={theme}>
      {children}
    </ThemeContext>
  );
}
```

Then:

```jsx
function App() {
  return (
    <ThemeProvider>
      <Page />
    </ThemeProvider>
  );
}
```

And somewhere deep inside:

```jsx
function Button() {
  const theme = useContext(ThemeContext);

  return (
    <button className={`button-${theme}`}>
      Click me
    </button>
  );
}
```

The important thing is that `Button` doesn't need:

```jsx
<Button theme={theme} />
```

The value comes through the Context.

---

# 3. What exactly is a Context?

A common misunderstanding is:

> "Context stores my state."

Not exactly.

`createContext()` creates a **context object**.

```jsx
const ThemeContext = createContext("light");
```

The context object represents:

> "This is the kind of information that components can provide or read."

React's documentation explicitly notes that the context object itself does not hold the application information; it represents the context being provided/read.

Think of it as a channel:

```text
             ThemeContext
                  │
          ┌───────┴───────┐
          │               │
       Provider        Consumer
          │               │
       provides          reads
          │               │
          └────── value ──┘
```

---

# 4. Provider Pattern

The session specifically contains a section called:

```text
Provider Pattern
```

The Provider Pattern means:

> Put some value/service at a higher level of the component tree and allow descendants to consume it without manually passing it through every intermediate component.

Conceptually:

```text
Provider
   │
   ├── Component A
   │      │
   │      └── Component B
   │              │
   │              └── Component C
   │
   └── Component D
```

All components underneath the provider can access the provided value.

---

## React 19 Provider Syntax

Older React code commonly used:

```jsx
<ThemeContext.Provider value={theme}>
  <App />
</ThemeContext.Provider>
```

Starting with React 19, the context object itself can be rendered as the provider:

```jsx
<ThemeContext value={theme}>
  <App />
</ThemeContext>
```

React's current documentation identifies `<SomeContext>` as the provider syntax in React 19 and `<SomeContext.Provider>` as the older syntax.

---

# 5. Closest Provider Wins

Context follows the component tree.

If we have:

```jsx
<ThemeContext value="dark">

  <Page />

  <ThemeContext value="light">
    <Footer />
  </ThemeContext>

</ThemeContext>
```

Then:

```text
Page   → dark
Footer → light
```

Why?

Because `Footer` finds the closest matching provider above it.

This makes nested providers useful when a subtree needs to override a value.

This connects directly to the session's idea of having:

```text
FirstThemeProvider
SecondThemeProvider
```

and different parts of the tree receiving different context values.

---

# 6. Context API vs Redux

Context and Redux are often compared because both can help solve problems related to passing data through a component tree.

But they are not the same abstraction.

## Context

Context primarily solves:

> How can a component access data from a distant parent without prop drilling?

Example:

```jsx
const ThemeContext = createContext("light");
```

Then:

```jsx
<ThemeContext value={theme}>
  <App />
</ThemeContext>
```

and:

```jsx
const theme = useContext(ThemeContext);
```

Context itself is **not a complete state-management architecture**.

You normally combine it with:

```jsx
useState()
```

or:

```jsx
useReducer()
```

to manage changing state. React explicitly recommends combining context with state/reducers when context-backed data needs to change.

---

## Redux

Redux provides a more structured state-management model.

The conceptual model is:

```text
UI
 ↓
dispatch(action)
 ↓
Reducer
 ↓
New State
 ↓
Store
 ↓
UI
```

Redux also provides tooling and concepts around actions, reducers, middleware, and centralized state.

The Redux documentation notes that Redux and Context can both avoid prop drilling, but Redux provides additional state-management and debugging capabilities such as Redux DevTools and middleware.

---

## Important distinction

Don't memorize:

> Context = small Redux.

A better mental model is:

```text
Context
= mechanism for making a value available through a tree

Redux
= state-management architecture + store + update model + tooling
```

Redux itself can use React Context internally to make its store available to the component tree.

---

# 7. Stable References + Context Re-renders

This is an important part of the session because the diagram contains:

```text
Stable Reference
```

and:

```text
SAME Reference (SKIP)
```

It also contains the idea:

```text
WE'RE NOT GOING
TO MEMOIZE EVERYTHING
```

The important React concept is **reference identity**.

Consider:

```jsx
const value = {
  theme,
  toggleTheme
};
```

If this object is recreated every render:

```jsx
const value = {
  theme,
  toggleTheme
};
```

then:

```js
previousValue !== nextValue
```

even if:

```js
previousValue.theme === nextValue.theme
```

because they are two different objects.

React compares context values using `Object.is`. If the provider receives a different value, components reading that context can re-render.

---

## Example

```jsx
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");

  const value = {
    theme,
    setTheme
  };

  return (
    <ThemeContext value={value}>
      {children}
    </ThemeContext>
  );
}
```

Every render creates a new object:

```text
Render #1 → { theme, setTheme } ← object A

Render #2 → { theme, setTheme } ← object B
```

Even though the contents may look identical:

```js
objectA !== objectB
```

This can cause consumers of the context to update.

React's documentation demonstrates using `useCallback` and `useMemo` when a context value contains objects/functions and unnecessary context updates need to be avoided.

---

## But don't memoize everything

The session explicitly highlights:

```text
WE'RE NOT GOING
TO MEMOIZE EVERYTHING
```

This is important.

Memoization is an optimization.

Don't automatically write:

```jsx
useMemo(...)
useCallback(...)
memo(...)
```

everywhere.

First understand:

```text
What changes?
Why does it change?
Is the extra render actually expensive?
Would memoization solve the real problem?
```

This idea becomes especially relevant when we talk about **React Compiler**.

---

# 8. React Compiler

React Compiler is a **build-time optimization tool**.

Its purpose is to automatically optimize React code, especially around memoization and unnecessary work during updates.

React describes it as a compiler that understands the Rules of React and can automatically apply optimizations that previously often required manual memoization.

---

## Before React Compiler

Developers might manually optimize:

```jsx
const ExpensiveComponent = memo(function ExpensiveComponent({
  data
}) {
  const result = useMemo(() => {
    return expensiveCalculation(data);
  }, [data]);

  return <div>{result}</div>;
});
```

This can work, but it introduces more code and more dependency management.

---

## With React Compiler

The goal is to let you write normal React:

```jsx
function ExpensiveComponent({ data }) {
  const result = expensiveCalculation(data);

  return <div>{result}</div>;
}
```

and let the compiler determine where memoization can help.

React's documentation explains that the compiler can automatically memoize components/values and expensive calculations used during rendering.

---

# 9. React Compiler Playground

The **React Compiler Playground** is useful because it lets you see what the compiler does to React code.

You can take code like:

```jsx
function Component({ items }) {
  const result = expensiveCalculation(items);

  return (
    <div>
      {result.map(item => (
        <Item key={item.id} item={item} />
      ))}
    </div>
  );
}
```

and inspect how the compiler transforms/optimizes it.

The official React Compiler documentation links directly to the playground as a way to inspect compiler behavior.

[React Compiler Playground](https://playground.react.dev/?utm_source=chatgpt.com)

### Important mental model

React Compiler does **not** mean:

> "React no longer renders."

It means:

> "The compiler can automatically optimize some of the work React would otherwise perform."

So keep the previous mental model:

```text
State changes
      ↓
React update
      ↓
Compiler-optimized component code
      ↓
Reconciliation
      ↓
Commit
```

---

# 10. `use()` — React Resource API

The session then moves to:

```text
use()
```

and the diagram connects it to:

```text
Promise
Context
Consume it
```

React's `use` API lets a component read a **resource** during rendering.

Currently important resources include:

```text
Promise
Context
```

React's documentation explicitly describes `use` as an API for reading a resource such as a Promise or Context.

Basic syntax:

```jsx
const value = use(resource);
```

---

# 11. `use(context)`

You can pass a Context to `use()`:

```jsx
const theme = use(ThemeContext);
```

This gives you the context value from the closest provider.

Conceptually:

```text
ThemeProvider
      │
      ▼
   ThemeContext
      │
      ▼
    use()
      │
      ▼
   "dark"
```

---

# 12. `use()` vs `useContext()`

This is one of the most important distinctions.

## `useContext()`

```jsx
const theme = useContext(ThemeContext);
```

`useContext` is a React Hook specifically designed for reading and subscribing to Context.

It normally follows the Rules of Hooks:

```jsx
function Button() {
  const theme = useContext(ThemeContext);

  return <button>{theme}</button>;
}
```

React documents `useContext()` as a Hook that reads and subscribes to context.

---

## `use()`

```jsx
const theme = use(ThemeContext);
```

`use()` is a broader resource-reading API.

The major difference is that unlike normal Hooks, `use()` can be called inside conditions and loops.

For example:

```jsx
function Button({ shouldReadTheme }) {
  if (shouldReadTheme) {
    const theme = use(ThemeContext);

    return <button>{theme}</button>;
  }

  return <button>No theme</button>;
}
```

This is specifically allowed for `use()`.

React's documentation explicitly contrasts this with `useContext()`.

---

## Very important

Don't think:

```text
use() = new version of useContext()
```

Instead:

```text
useContext()
    ↓
Context-specific Hook

use()
    ↓
Resource-reading API
    ├── Context
    └── Promise
```

---

# 13. Why `use()` Can Read a Promise

This is where the concept becomes much more interesting.

Normally:

```js
const promise = fetch("/api/user");
```

doesn't give you the final data immediately.

It gives you:

```text
Promise
   │
   ├── pending
   ├── fulfilled
   └── rejected
```

The session diagram explicitly shows this idea with:

```text
Pending Promise#1
Rejected Promise#2
Resolved Promise#3
Promise#4
Promise#5
Promise#6
```

With `use()`:

```jsx
const user = use(userPromise);
```

React can treat the Promise as a resource.

---

# 14. `use(promise)`

Example:

```jsx
function User({ userPromise }) {
  const user = use(userPromise);

  return <h1>{user.name}</h1>;
}
```

If the Promise is already resolved:

```text
Promise
   ↓
resolved value
   ↓
use()
   ↓
user
```

If it is still pending:

```text
Promise
   ↓
pending
   ↓
use()
   ↓
suspend
```

The component doesn't simply continue rendering as if it had a value.

React suspends that part of the render.

React's official documentation describes this exact behavior: when `use()` receives a pending Promise, the component suspends.

---

# 15. What Does "Suspend" Mean?

This does **not** mean:

> JavaScript thread magically stops.

Instead, React says conceptually:

> "I cannot finish rendering this component yet because the resource isn't ready."

Then React can use a nearby Suspense boundary.

Conceptually:

```text
Component
   │
   │ use(promise)
   │
   ├── resolved → continue rendering
   │
   └── pending → suspend
                    │
                    ▼
                 Suspense
                    │
                    ▼
                 fallback
```

This connects directly to the session's:

```text
Throw Promise
```

and:

```text
<Suspense>
</Suspense>
```

---

# 16. Suspense

Suspense provides a boundary around content that might suspend.

Example:

```jsx
<Suspense fallback={<Loading />}>
  <User userPromise={userPromise} />
</Suspense>
```

If:

```jsx
use(userPromise)
```

suspends, React displays:

```jsx
<Loading />
```

When the Promise resolves:

```text
Promise resolves
      ↓
React retries rendering
      ↓
use(promise)
      ↓
returns data
      ↓
actual UI
```

React describes Suspense as a mechanism for displaying a fallback until suspended children are ready.

---

# 17. Suspense Is Not Just "Loading UI"

A common misunderstanding is:

> Suspense = loading spinner.

Not exactly.

The deeper idea is:

```text
Suspension
    ↓
Boundary
    ↓
Fallback
    ↓
Retry when resource becomes available
```

The fallback happens because something inside the boundary **suspended**.

For example:

```jsx
<Suspense fallback={<Loading />}>
  <Albums />
</Suspense>
```

where:

```jsx
function Albums() {
  const albums = use(albumsPromise);

  return (
    <ul>
      {albums.map(album => (
        <li key={album.id}>
          {album.title}
        </li>
      ))}
    </ul>
  );
}
```

---

# 18. Promise Rejection and Error Boundary

There are two different situations:

```text
Promise
   │
   ├── Pending
   │      ↓
   │   Suspense
   │
   ├── Resolved
   │      ↓
   │   Render data
   │
   └── Rejected
          ↓
     Error Boundary
```

The session diagram explicitly connects:

```text
<ErrorBoundary>
   ...
</ErrorBoundary>
```

with Promise states and `use()`.

React's documentation also states that a rejected Promise passed to `use()` propagates to the nearest Error Boundary, while a pending Promise activates the nearest Suspense fallback.

---

# 19. The Complete Promise Flow

The whole concept can be visualized as:

```text
                  Promise
                     │
             ┌───────┴────────┐
             │                │
          Pending           Settled
             │             ┌──┴───┐
             │             │      │
             ▼             ▼      ▼
         Suspense       Resolved  Rejected
             │             │      │
             ▼             ▼      ▼
          fallback        data   ErrorBoundary
```

This is one of the most important mental models from this session.

---

# 20. Promise Caching

There is an extremely important detail:

```jsx
use(fetch("/api/data"))
```

can be problematic in a Client Component.

Why?

Because:

```jsx
fetch("/api/data")
```

creates a new Promise every time the component renders.

Imagine:

```text
Render #1
fetch()
   ↓
Promise A

Render #2
fetch()
   ↓
Promise B

Render #3
fetch()
   ↓
Promise C
```

React is no longer seeing the same resource.

React's documentation warns that Promises passed to `use()` must be cached so the same Promise instance can be reused across renders.

---

# 21. Simple Promise Cache

A basic cache can look like:

```js
const cache = new Map();

function fetchData(url) {
  if (!cache.has(url)) {
    cache.set(url, getData(url));
  }

  return cache.get(url);
}
```

Now:

```jsx
const data = use(fetchData("/api/data"));
```

produces:

```text
First render
     ↓
fetchData("/api/data")
     ↓
Map miss
     ↓
create Promise A
     ↓
store Promise A

Second render
     ↓
fetchData("/api/data")
     ↓
Map hit
     ↓
return Promise A
```

So React receives the same Promise instance.

---

# 22. Why Stable References Matter Here

This connects directly to the session's earlier idea:

```text
SAME Reference (SKIP)
```

and:

```text
CHANGE REFERENCE
```

The same general principle appears again:

```text
Stable identity
     ↓
React can recognize the same resource/value
```

This is not exactly the same mechanism as list `key`s, but it is the same broader engineering idea:

> Identity matters.

For example:

```js
promiseA === promiseA
```

but:

```js
promiseA !== promiseB
```

even if both Promises represent the same URL.

---

# 23. React `cache()`

React also provides a `cache()` API.

Example:

```jsx
import { cache } from "react";

const getUser = cache(async (userId) => {
  return await db.user.query(userId);
});
```

Then:

```jsx
async function Profile({ userId }) {
  const user = await getUser(userId);

  return <h1>{user.name}</h1>;
}
```

React can reuse the cached result when the same cached function is called with the same arguments during the relevant React render/cache lifetime.

---

# 24. Important Difference: `cache()` vs `useMemo()`

These are easy to confuse.

## `useMemo`

```jsx
const result = useMemo(() => {
  return expensiveCalculation(data);
}, [data]);
```

This is a component Hook used to cache a calculation between renders when dependencies are unchanged.

---

## `cache()`

```jsx
const getUser = cache(async (id) => {
  return getUserFromDatabase(id);
});
```

This creates a cached version of a function.

The current React documentation describes `cache()` as intended for React Server Components.

So don't think:

```text
cache() = useMemo()
```

A better mental model:

```text
useMemo()
    ↓
cache a calculation/value for a component

cache()
    ↓
cache a function's result within React's cache model
```

---

# 25. Cache and `use()`

These concepts work together.

A simplified mental model is:

```text
        cache
          │
          ▼
       Promise
          │
          ▼
        use()
          │
     ┌────┴────┐
     │         │
 pending     resolved
     │         │
     ▼         ▼
 Suspense     UI
```

This is why the session placed:

```text
Cache
```

near:

```text
Promise
Suspense
use()
```

---

# 26. Preloading + Cache

Caching can also allow preloading.

For example:

```jsx
<button
  onMouseEnter={() => fetchData(`/artists/${id}/albums`)}
  onClick={() => setArtistId(id)}
>
  Open artist
</button>
```

When the user hovers:

```text
Mouse enter
    ↓
fetchData()
    ↓
Promise enters cache
```

Then when the user clicks:

```text
use(cachedPromise)
```

the data may already be available.

React's `use()` documentation describes this pattern as a way to start loading data before it is actually needed.

---

# 27. `Image()` Constructor — MDN

The final topic is not a React API.

It is a **browser API**.

MDN documents:

```js
new Image()
```

as the `Image()` constructor.

It creates a new `HTMLImageElement`.

It is functionally equivalent to:

```js
document.createElement("img");
```

---

## Example

```js
const img = new Image();

img.src = "image.jpg";
img.alt = "Example";

document.body.appendChild(img);
```

Equivalent conceptually to:

```js
const img = document.createElement("img");

img.src = "image.jpg";
img.alt = "Example";

document.body.appendChild(img);
```

---

# 28. `new Image()` Does Not Automatically Put It in the DOM

This is important.

```js
const img = new Image();
```

creates:

```text
HTMLImageElement
```

but it isn't automatically attached to the document.

You can configure it:

```js
img.src = "image.jpg";
```

and then attach it:

```js
document.body.appendChild(img);
```

MDN explicitly describes the object created by `Image()` as an `<img>` element that is not attached to the DOM tree until you insert it.

---

# 29. Why `new Image()` Can Be Useful

One useful browser-side pattern is preparing an image before inserting it into the document:

```js
const img = new Image();

img.onload = () => {
  console.log("Image loaded");
};

img.onerror = () => {
  console.log("Image failed");
};

img.src = "/images/avatar.png";
```

The browser starts loading the resource after assigning the `src`.

You can also use:

```js
img.decode()
```

which returns a Promise that resolves when the image has been decoded and is ready to be safely appended. MDN documents `HTMLImageElement.decode()` for this purpose.

---

# 30. Connection Between `Image()` and React

This is a useful connection to understand.

`new Image()` is a **browser API**:

```text
Browser
 └── HTMLImageElement
       └── Image()
```

React's:

```jsx
<img src="/image.png" />
```

is different.

React creates/manages the DOM representation through its rendering system.

So:

```js
new Image()
```

is imperative browser-side DOM work.

While:

```jsx
<img src="/image.png" />
```

is declarative React UI.

The two can still interact when browser APIs are needed.

---

# 31. Complete Mental Model

At this point, the concepts from this session can be connected together:

```text
                         React Tree
                             │
                             ▼
                      Provider Pattern
                             │
                             ▼
                        Context API
                             │
                    ┌────────┴────────┐
                    │                 │
              useContext()          use()
                    │                 │
                    │           ┌─────┴─────┐
                    │           │           │
                    │        Context     Promise
                    │                       │
                    │                  ┌────┴────┐
                    │                  │         │
                    │               Pending   Settled
                    │                  │       ┌──┴──┐
                    │                  │       │     │
                    │                  ▼       ▼     ▼
                    │              Suspense  Data  Error
                    │                         │
                    │                         │
                    │                       Cache
                    │
                    ▼
                 UI update
```

And around all of this:

```text
Stable references
       │
       ▼
Memoization
       │
       ▼
React Compiler
```

---

# 32. The Most Important Connections

## Connection 1 — Context and Provider

```text
createContext()
      ↓
Provider
      ↓
useContext()
      ↓
consumer
```

---

## Connection 2 — `use()` and Context

```text
useContext(ThemeContext)
```

versus:

```text
use(ThemeContext)
```

The second is part of the broader Resource API and has different calling rules.

---

## Connection 3 — `use()` and Promise

```text
use(promise)
      ↓
Promise pending?
      │
      ├── Yes → Suspend
      │
      └── No → return value
```

---

## Connection 4 — Suspense

```text
Promise pending
      ↓
component suspends
      ↓
nearest Suspense
      ↓
fallback
      ↓
Promise resolves
      ↓
React retries
      ↓
real UI
```

---

## Connection 5 — Rejected Promise

```text
Promise rejected
      ↓
Error
      ↓
nearest Error Boundary
```

---

## Connection 6 — Cache

```text
same resource
      ↓
same Promise/reference
      ↓
React can reuse it
```

---

## Connection 7 — Stable Reference

```text
same reference
      ↓
Object.is(...)
      ↓
React can recognize no value change
```

This concept appears repeatedly across React optimization and resource management.

---

# 33. Common Mistakes

## Mistake 1 — "Context is state management"

Not by itself.

Context provides a way to make values available through a tree.

State can live elsewhere:

```jsx
useState()
```

or:

```jsx
useReducer()
```

and then be provided through Context.

---

## Mistake 2 — "Context replaces Redux"

Not necessarily.

They solve overlapping problems but provide different abstractions.

Context:

```text
data distribution
```

Redux:

```text
structured state management
```

---

## Mistake 3 — "use() is just useContext()"

No.

```text
useContext()
→ context Hook

use()
→ resource-reading API
```

`use()` can read Context **and** Promise resources.

---

## Mistake 4 — "Suspense fetches data"

Suspense itself isn't a data-fetching library.

It provides the boundary/fallback behavior when something suspends.

---

## Mistake 5 — Creating a new Promise during every render

Avoid:

```jsx
const data = use(fetch("/api/data"));
```

if that creates a new Promise on every render.

Use a cached Promise/resource instead. React explicitly warns about this pattern.

---

## Mistake 6 — Memoizing everything

Don't automatically use:

```jsx
memo()
useMemo()
useCallback()
```

everywhere.

Memoization is an optimization.

The session itself emphasizes:

```text
WE'RE NOT GOING
TO MEMOIZE EVERYTHING
```

And React Compiler is designed to reduce the amount of manual memoization needed.

---

## Mistake 7 — Confusing `new Image()` with React `<img>`

```js
new Image()
```

is a browser API.

```jsx
<img />
```

is React JSX describing UI.

---

# 34. Interview Questions

### Q1 — What problem does Context solve?

It allows a parent to make data available to descendants without explicitly passing that data through every intermediate component.

---

### Q2 — Does Context store state?

Not by itself.

Context provides a value. State can be managed using `useState`, `useReducer`, or another state-management mechanism.

---

### Q3 — What is the Provider Pattern?

A pattern where a higher-level component provides a value/service to descendants, commonly through Context.

---

### Q4 — Context vs Redux?

Context primarily provides data through the React tree.

Redux is a broader state-management architecture with a centralized store, reducers/actions, middleware, and tooling.

---

### Q5 — What is React Compiler?

A build-time tool that automatically performs React optimizations such as memoization where appropriate.

---

### Q6 — What can `use()` read?

Important cases include:

```text
Context
Promise
```

React also documents other resource integrations such as browser-only resources.

---

### Q7 — What is the difference between `use()` and `useContext()`?

`useContext()` is specifically for reading Context and follows normal Hook calling rules.

`use()` can read Context or a Promise and can be called inside conditions and loops.

---

### Q8 — What happens when `use()` receives a pending Promise?

The component suspends.

A surrounding `<Suspense>` boundary can then display its fallback.

---

### Q9 — What happens when the Promise rejects?

The rejection propagates to the nearest Error Boundary.

---

### Q10 — Why must Promises passed to `use()` be cached?

Because creating a new Promise during every render creates a different resource identity every time and can cause repeated suspension.

---

### Q11 — What is `cache()`?

A React API for caching the result of a function within React's cache model. The current React documentation describes it for Server Components.

---

### Q12 — What does `new Image()` return?

A new `HTMLImageElement`, equivalent in purpose to:

```js
document.createElement("img");
```

---

# 35. Mini Exercises

## Exercise 1 — Theme Context

Create:

```jsx
ThemeContext
```

with:

```text
light
dark
```

Then create:

```jsx
ThemeProvider
```

and a deeply nested:

```jsx
Button
```

that reads the theme without props.

---

## Exercise 2 — Nested Providers

Create:

```jsx
<ThemeContext value="dark">
  <Page />

  <ThemeContext value="light">
    <Footer />
  </ThemeContext>
</ThemeContext>
```

Predict:

```text
Page   → ?
Footer → ?
```

---

## Exercise 3 — `use()` Context

Rewrite:

```jsx
const theme = useContext(ThemeContext);
```

using:

```jsx
use()
```

Then test the difference in where the resource can be read.

---

## Exercise 4 — Promise + Suspense

Create a cached Promise:

```js
const promise = fetchData();
```

Then:

```jsx
<Suspense fallback={<p>Loading...</p>}>
  <Component promise={promise} />
</Suspense>
```

Inside:

```jsx
const data = use(promise);
```

Observe:

```text
Pending
 ↓
Loading

Resolved
 ↓
Data
```

---

## Exercise 5 — Rejected Promise

Make the Promise reject.

Wrap the Suspense tree with an Error Boundary.

Observe the difference between:

```text
Pending
```

and:

```text
Rejected
```

---

## Exercise 6 — Reference Identity

Compare:

```js
const a = {};
const b = {};

console.log(a === b);
```

with:

```js
const a = {};
const b = a;

console.log(a === b);
```

Connect the result to:

```text
Stable Reference
```

and Context value updates.

---

## Exercise 7 — Image Constructor

Create an image without putting it in the DOM:

```js
const image = new Image();

image.src = "/test.png";
```

Then:

```js
image.onload = () => {
  console.log("loaded");
};
```

Finally append it:

```js
document.body.appendChild(image);
```

---

# 36. Quick Revision

```text
Context
→ Allows distant descendants to receive data without prop drilling.

Provider
→ Provides a value to a subtree.

useContext()
→ Reads and subscribes to Context.

Provider Pattern
→ Higher-level component provides data/service to descendants.

Redux
→ Broader state-management architecture; not simply another Context.

Stable Reference
→ Same object/resource identity.

React Compiler
→ Build-time automatic React optimization.

use()
→ Reads resources such as Context or Promise.

use(context)
→ Similar purpose to useContext(), but different calling rules.

use(promise)
→ Reads Promise result during rendering.

Pending Promise
→ Component suspends.

Suspense
→ Displays fallback while suspended content isn't ready.

Resolved Promise
→ React can continue rendering with the value.

Rejected Promise
→ Error Boundary handles the error.

Promise Cache
→ Reuses the same Promise/resource between renders.

cache()
→ React caching API, currently documented for Server Components.

Image()
→ Browser constructor that creates an HTMLImageElement.

new Image()
≈
document.createElement("img")
```

---

# 37. Final Mental Model

The entire session can be remembered as one chain:

```text
                    CONTEXT
                       │
                       ▼
                   PROVIDER
                       │
                       ▼
                 Component Tree
                       │
              ┌────────┴────────┐
              │                 │
        useContext()          use()
                                  │
                          ┌───────┴───────┐
                          │               │
                       Context         Promise
                                          │
                                ┌─────────┴─────────┐
                                │                   │
                             Pending             Settled
                                │                ┌──┴──┐
                                ▼                │     │
                             Suspend          Resolve Reject
                                │                │     │
                                ▼                │     ▼
                             Suspense             │ ErrorBoundary
                                │                │
                                ▼                ▼
                            fallback            Data
                                                   
                           Cache
                             │
                             ▼
                     Stable Resource
```

And optimization sits around the system:

```text
Stable References
       │
       ▼
Memoization
       │
       ▼
React Compiler
       │
       ▼
Less unnecessary work
```

This is the key conceptual connection between the topics in this session.

---

# Sources

## React

* [React — Passing Data Deeply with Context](https://react.dev/learn/passing-data-deeply-with-context)
* [React — createContext](https://react.dev/reference/react/createContext)
* [React — useContext](https://react.dev/reference/react/useContext)
* [React — use](https://react.dev/reference/react/use)
* [React — Suspense](https://react.dev/reference/react/Suspense)
* [React — cache](https://react.dev/reference/react/cache)
* [React — React Compiler Introduction](https://react.dev/learn/react-compiler/introduction)
* [React Compiler Playground](https://playground.react.dev/)

## Redux

* [Redux — React Redux FAQ: Context API vs Redux](https://redux.js.org/faq/react-redux)

## MDN

* [MDN — HTMLImageElement: Image() constructor](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement/Image)
* [MDN — HTMLImageElement](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement)

## Rootline

* [Rootline — use API article](https://www.root-line.tech/blog/use-api)

> Note: the Rootline article URL was included in the session material, but the page itself could not be fetched during research. The explanations above therefore use the Excalidraw session material plus the current official React/MDN documentation rather than pretending to quote or summarize unavailable text from that article.

---

# Session Connection

The second session's visual material specifically connects:

```text
First Theme Provider
        ↓
Provider Pattern
        ↓
Context API
        ↓
useContext / use()
        ↓
Promise
        ↓
Throw Promise / Suspend
        ↓
Suspense
        ↓
Error Boundary
        ↓
Cache
```

It also connects this to:

```text
Stable Reference
SAME Reference (SKIP)
        ↓
Memoization
        ↓
React Compiler
```

The final browser-level topic:

```text
Image()
```

connects the React concepts back to the underlying browser APIs and DOM.
