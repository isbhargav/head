---
title: "React Component Composition"
date: 2026-09-12T00:00:00+00:00
description: "How to build flexible React UIs by composing components instead of relying on inheritance or prop drilling."
tags:
  - react
  - javascript
  - frontend
categories:
  - React

---

React has a simple but powerful idea at its core: components are just
functions that return UI, and you build complex interfaces by **composing**
small components together. React's own docs put it bluntly — they recommend
composition over inheritance, and nothing in React's component model actually
requires class hierarchies.

This post walks through the composition patterns I reach for most often, from
the humble `children` prop to slot-style APIs.

## Why composition?

Deep prop chains and bloated "god components" are the two failure modes of a
growing React codebase. Composition solves both:

- **Small, focused components** are easier to test and reason about.
- **Flexible APIs** let callers decide what goes inside a component, instead of
  the component trying to predict every use case with props.

## Extract components, pass JSX as children

Every component receives a special `children` prop — whatever you nest between
its opening and closing tags. This unlocks the most useful composition habit:
when you find yourself passing *data* through many layers of intermediate
components that don't use that data (and only pass it further down), it often
means you forgot to extract some components along the way.

Suppose a `Layout` receives `posts` only to hand them straight to the `Posts`
inside it:

```jsx
function Layout({ posts }) {
  return (
    <div className="layout">
      <SiteNav />
      <main>
        <Posts posts={posts} />
      </main>
    </div>
  );
}

function App() {
  return <Layout posts={posts} />;
}
```

`Layout` is a purely visual component — it doesn't care about `posts`, yet its
props are now coupled to them. Every new piece of data the `Posts` section
needs becomes another prop threaded through `Layout`. Instead, extract `Posts`
out of `Layout` and let `Layout` take `children`:

```jsx
function Layout({ children }) {
  return (
    <div className="layout">
      <SiteNav />
      <main>{children}</main>
    </div>
  );
}

function App() {
  return (
    <Layout>
      <Posts posts={posts} />
    </Layout>
  );
}
```

Now the component that specifies the data (`App`) renders the component that
needs it (`Posts`) directly — the number of layers between them shrinks to
zero. `Layout` provides the box; the caller provides the content, and `Layout`
never has to change when that content does.

## Slots: passing elements as props

`children` works when a component has one "hole" to fill. When there are
several, pass elements as named props — sometimes called the *slot* pattern:

```jsx
function Layout({ header, sidebar, children }) {
  return (
    <div className="layout">
      <header>{header}</header>
      <aside>{sidebar}</aside>
      <main>{children}</main>
    </div>
  );
}

function App() {
  return (
    <Layout
      header={<SiteNav />}
      sidebar={<TagCloud tags={tags} />}
    >
      <ArticleList posts={posts} />
    </Layout>
  );
}
```

`Layout` controls *where* things render; the caller controls *what* renders.
Props aren't limited to strings and numbers — JSX elements are values too.

## Specialization

Sometimes you want a component that is a more specific version of a generic
one. Instead of inheritance, specialize through composition:

```jsx
function Button({ variant = "default", children, ...rest }) {
  return (
    <button className={`btn btn--${variant}`} {...rest}>
      {children}
    </button>
  );
}

function DangerButton({ children, ...rest }) {
  return (
    <Button variant="danger" {...rest}>
      {children}
    </Button>
  );
}
```

`DangerButton` *is* a `Button` with a preset — no subclassing required. The
`...rest` spread keeps the generic component's full API (event handlers,
`aria-*` attributes, `type`, etc.) available to callers.

## Composition beats prop drilling

A common pain point: a piece of state lives high in the tree, and intermediate
components forward props they don't care about. Before reaching for context or
a state library, try *moving the component down* and passing the UI up instead:

```jsx
// Instead of drilling `user` through Page -> Layout -> Header -> Avatar,
// render the Avatar where the data lives and pass the element down.
function App() {
  const user = useUser();
  return (
    <Page header={<Header avatar={<Avatar user={user} />} />}>
      <Dashboard user={user} />
    </Page>
  );
}
```

Components in the middle (`Page`, `Header`) now receive ready-made elements
and never touch `user` at all. Fewer props, fewer re-renders — and context
stays reserved for state that genuinely can't be passed down directly (more on
using it well in the last section).

## Compound components

For tightly-coupled UI like tabs, accordions, or menus, the *compound
components* pattern lets a parent share implicit state with its children via
context:

```jsx
const TabsContext = createContext(null);

function Tabs({ children }) {
  const [active, setActive] = useState(0);
  return (
    <TabsContext.Provider value={{ active, setActive }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function Tab({ index, children }) {
  const { active, setActive } = useContext(TabsContext);
  return (
    <button
      className={active === index ? "tab is-active" : "tab"}
      onClick={() => setActive(index)}
    >
      {children}
    </button>
  );
}
```

The usage reads declaratively, and the caller still controls the markup:

```jsx
<Tabs>
  <Tab index={0}>Overview</Tab>
  <Tab index={1}>Specs</Tab>
  <TabPanel index={0}>...</TabPanel>
  <TabPanel index={1}>...</TabPanel>
</Tabs>
```

It's more machinery than `children` alone, so reserve it for components that
genuinely share internal state.

## Context, done effectively

Composition gets you surprisingly far, but some state genuinely has to cross
the tree — a logged-in user, feature flags, the current theme. When you *do*
reach for context, Kent C. Dodds'
[How to Use React Context Effectively](https://kentcdodds.com/blog/how-to-use-react-context-effectively)
lays out a pattern worth copying: a custom provider component plus a custom
consumer hook.

```jsx
// count-context.jsx
import { createContext, useContext, useReducer } from "react";

// No default value, on purpose: consumers must render inside a CountProvider.
const CountContext = createContext(undefined);

function countReducer(state, action) {
  switch (action.type) {
    case "increment":
      return { count: state.count + 1 };
    case "decrement":
      return { count: state.count - 1 };
    default:
      throw new Error(`Unhandled action type: ${action.type}`);
  }
}

function CountProvider({ children }) {
  const [state, dispatch] = useReducer(countReducer, { count: 0 });
  // NOTE: you *might* need to memoize this value — see kcd.im/optimize-context
  const value = { state, dispatch };
  return (
    <CountContext.Provider value={value}>{children}</CountContext.Provider>
  );
}

function useCount() {
  const context = useContext(CountContext);
  if (context === undefined) {
    throw new Error("useCount must be used within a CountProvider");
  }
  return context;
}

// Only the provider and the hook are exported — never the context itself.
export { CountProvider, useCount };
```

Usage stays clean for consumers:

```jsx
function App() {
  return (
    <CountProvider>
      <CountDisplay />
      <Counter />
    </CountProvider>
  );
}
```

Three details make this pattern effective:

- **No default value.** A default would silently mask the real mistake — a
  consumer rendered outside its provider. The custom hook instead *fails fast*
  with a clear error, instead of a confusing "cannot destructure `undefined`".
- **Don't export the context object.** Exporting only `CountProvider` and
  `useCount` gives consumers exactly one way to provide and one way to consume.
  That freedom lets you add helpers later — for example, an async action like
  `updateUser(dispatch, user, updates)` that dispatches start/finish/fail
  around a request — without changing any call sites.
- **Context doesn't have to be global.** Scope a provider to the subtree that
  actually needs it, and prefer several logically separated contexts over one
  app-wide mega-context.

## Advanced: namespaced compound components

The `Tabs` example above used context to wire up implicit state. Component
libraries like shadcn/ui push the same idea much further — see Vercel's
[Compound Components and Advanced Composition](https://vercel.com/academy/shadcn-ui/compound-components-and-advanced-composition) —
turning a whole component family into a namespace of cooperating parts, much
like HTML's native `<select>` and `<option>`.

Instead of one monolithic component drowning in configuration:

```jsx
<DataTable
  data={data}
  columns={columns}
  pagination
  sorting
  filtering
  actions={["edit", "delete"]}
  rowSelection
  // ...20 more props
/>
```

...responsibility is distributed across parts the caller composes:

```jsx
<Card.Root collapsible onDismiss={handleDismiss}>
  <Card.Header actions={<Button>Refresh</Button>}>
    <Card.Title>Real-time analytics</Card.Title>
    <Card.Description>Live performance metrics</Card.Description>
  </Card.Header>
  <Card.Content>
    <MetricsGrid data={metrics} />
  </Card.Content>
  <Card.Footer>
    <Button>View details</Button>
  </Card.Footer>
</Card.Root>
```

The mechanics build directly on the previous section: `Card.Root` is a
provider component holding shared state (`isCollapsed`, `isExpanded`) and
configuration (`variant`, `size`), and each part reads it through a fail-fast
hook. Parts can then *react* to that shared state — here `Card.Content` hides
itself when the card is collapsed, unless told otherwise:

```jsx
const CardContext = createContext(null);

function useCardContext() {
  const context = useContext(CardContext);
  if (!context) {
    throw new Error("Card compound components must be used within Card.Root");
  }
  return context;
}

function CardContent({ children, forceVisible = false, ...props }) {
  const { isCollapsed } = useCardContext();
  if (isCollapsed && !forceVisible) return null;
  return <div {...props}>{children}</div>;
}

// Export the parts as a single namespace
export const Card = {
  Root: CardRoot,
  Header: CardHeader,
  Title: CardTitle,
  Description: CardDescription,
  Content: CardContent,
  Footer: CardFooter,
};
```

Because everything shares context, configuration set once on `Root` (say
`size="sm"`) cascades through the whole tree, and the header can render
collapse/expand/dismiss controls automatically whenever `Root` enables them.
Two further techniques make this pattern scale:

**Smart wrappers.** Compound parts are building blocks, not necessarily the
final API. You can compose them into higher-level components that encode
opinions — for example, deriving behavior from the content itself:

```jsx
function SmartCard({ title, data, children }) {
  // Lots of data? Make the card collapsible without asking the caller.
  const collapsible = data && Object.keys(data).length > 5;
  return (
    <Card.Root collapsible={collapsible}>
      <Card.Header>
        <Card.Title>{title}</Card.Title>
      </Card.Header>
      <Card.Content>{children}</Card.Content>
    </Card.Root>
  );
}
```

**Split contexts for performance.** With a single context, every state change
re-renders every consumer — including parts that only care about static
config. Separate the frequently-changing *state* from the rarely-changing
*configuration*, and memoize both values:

```jsx
const CardStateContext = createContext(null); // isCollapsed, isExpanded, ...
const CardConfigContext = createContext(null); // variant, size, flags, ...

function CardRoot({ children, variant = "default", size = "default" }) {
  const [isCollapsed, setIsCollapsed] = useState(false);

  const stateValue = useMemo(
    () => ({ isCollapsed, setIsCollapsed }),
    [isCollapsed]
  );
  const configValue = useMemo(() => ({ variant, size }), [variant, size]);

  return (
    <CardConfigContext.Provider value={configValue}>
      <CardStateContext.Provider value={stateValue}>
        {children}
      </CardStateContext.Provider>
    </CardConfigContext.Provider>
  );
}
```

Now a part that only reads `variant` from config no longer re-renders when the
card is collapsed or expanded.

## Takeaways

- Start with `children`; reach for named slots when a component has multiple
  content areas.
- Specialize by wrapping generic components, not by inheriting from them.
- Before adding context, try passing elements down as props to cut prop
  drilling.
- Use compound components when children need the parent's internal state.
- When you do need context, wrap it in a custom provider and a fail-fast
  consumer hook — and never export the raw context object.
- For complex component families, namespace compound parts (`Card.Root`,
  `Card.Header`), build smart wrappers on top, and split contexts by change
  frequency when re-renders hurt.

Composition keeps each component small and each API honest: a component owns
its structure and styling, while callers own the content. That separation is
what makes React codebases scale.
