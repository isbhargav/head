---
title: "Handling Errors in React Query"
date: 2026-09-20T00:00:00+00:00
description: "Don't swallow errors in your query functions — let React Query own the error lifecycle, then layer local states, error boundaries, and global toasts on top."
tags:
  - react
  - react-query
  - javascript
  - frontend
categories:
  - React

---

Error handling is an integral part of working with asynchronous data — not all
requests succeed, and not all Promises fulfil. Yet it's usually an
afterthought: we build for the sunshine case first and bolt error handling on
later, which is exactly how silent failures and broken retries sneak into a
codebase.

This post distils the patterns from two excellent sources — TkDodo's
[React Query Error Handling](https://tkdodo.eu/blog/react-query-error-handling)
and Tiger Abrodi's
[Proper Error Handling in React Query](https://tigerabrodi.blog/proper-error-handling-in-react-query) —
into one practical setup.

## Prerequisite: give React Query a rejected Promise

React Query can only handle errors it can *see*, and what it sees is a rejected
Promise. Libraries like axios reject automatically on 4xx/5xx responses, but
the `fetch` API does **not** — it resolves happily for a 500. So with `fetch`,
you must throw yourself:

```jsx
useQuery({
  queryKey: ["repos"],
  queryFn: async () => {
    const response = await fetch("/api/repos");
    if (!response.ok) {
      throw new Error(`Request failed: ${response.status}`);
    }
    return response.json();
  },
});
```

## The cardinal sin: swallowing errors

The most common mistake is catching an error inside the `queryFn` and *not*
re-throwing it:

```jsx
function useRepos() {
  return useQuery({
    queryKey: ["repos"],
    queryFn: async () => {
      try {
        const response = await fetch("/api/repos");
        if (!response.ok) throw new Error("Failed");
        return response.json();
      } catch (error) {
        console.log("Error:", error); // ❌ swallows the error
      }
    },
  });
}
```

From React Query's point of view, the Promise resolved (with `undefined`). The
consequences cascade:

- **No retries** — React Query doesn't know anything failed.
- **`status` stays `'success'`** — your component thinks everything is fine.
- **Error boundaries never fire** — there's nothing to propagate.
- **Silent failures** — users never learn something went wrong.

The fix is simple: let the error propagate. React Query *wants* the rejected
Promise so it can own the error lifecycle — retries, status, error boundaries —
and let you hook into it where you need to.

```jsx
function useRepos() {
  return useQuery({
    queryKey: ["repos"],
    queryFn: async () => {
      const response = await fetch("/api/repos");
      if (!response.ok) {
        throw new Error(`Request failed: ${response.status}`); // ✅ throw to RQ
      }
      return response.json();
    },
    retry: 3, // works because the error propagates
    retryDelay: 1000,
  });
}
```

## Level 1: the `isError` flag

The standard approach is checking the `isError` boolean (derived from the
`status` enum) in the component:

```jsx
function TodoList() {
  const todos = useQuery({ queryKey: ["todos"], queryFn: fetchTodos });

  if (todos.isPending) {
    return "Loading...";
  }

  if (todos.isError) {
    return "An error occurred";
  }

  return (
    <div>
      {todos.data.map((todo) => (
        <Todo key={todo.id} {...todo} />
      ))}
    </div>
  );
}
```

This is fine for some scenarios, but has two drawbacks:

1. **It handles background errors poorly.** If a background refetch fails —
   the API is briefly down, or you hit a rate limit — this unmounts your
   entire list even though you have perfectly good stale data on screen.
2. **It's boilerplate.** Every component that uses a query repeats the same
   dance.

## Level 2: error boundaries with `throwOnError`

Error boundaries are React's built-in way to catch render-time errors and show
fallback UI, scoped to whatever granularity you like so the rest of the page
keeps working. They can't catch *asynchronous* errors — so React Query does
something clever: it catches the fetch error internally and re-throws it during
the next render, where the boundary can pick it up.

All you do is opt in with `throwOnError` (called `useErrorBoundary` before v5):

```jsx
function TodoList() {
  // ✅ fetching errors propagate to the nearest error boundary
  const todos = useQuery({
    queryKey: ["todos"],
    queryFn: fetchTodos,
    throwOnError: true,
  });

  if (todos.data) {
    return (
      <div>
        {todos.data.map((todo) => (
          <Todo key={todo.id} {...todo} />
        ))}
      </div>
    );
  }
  return "Loading...";
}
```

You can also decide *which* errors reach the boundary by passing a function —
a great fit for mutations, where 4xx validation errors belong next to the form
but 5xx server errors should blow up to the boundary:

```jsx
useQuery({
  queryKey: ["todos"],
  queryFn: fetchTodos,
  // 🚀 only server errors go to the error boundary
  throwOnError: (error) => error.response?.status >= 500,
});
```

## Level 3: global notifications, fired exactly once

Sometimes you want an imperative toast instead of inline UI. The tempting
approach is the `onError` callback on the query (removed from `useQuery` in
v5, but still worth understanding):

```jsx
// ⚠️ looks right, but fires once per *observer*
useQuery({
  queryKey: ["todos"],
  queryFn: fetchTodos,
  onError: (error) => toast.error(`Something went wrong: ${error.message}`),
});
```

The catch: callbacks on `useQuery` run for every *observer* of the query —
conceptually like a `useEffect`. Use the same custom hook in two components
and one failed request pops two toasts.

If you want to notify the user *once per failed request*, put the callback on
the `QueryCache` instead:

```jsx
const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error) =>
      toast.error(`Something went wrong: ${error.message}`),
  }),
});
```

This is also the ideal home for error tracking and monitoring: it's guaranteed
to run once per request, and unlike `defaultOptions`, it can't be overridden
by individual queries.

## Putting it all together

React Query gives you three tools — the `error` property, the callbacks (per
query or global on the cache), and error boundaries — and you can mix them
freely. The setup both sources converge on:

- **Error boundaries** for initial-load failures, where there's nothing to
  show but a fallback.
- **Toast notifications** for *background* failures, so the stale UI stays
  intact and the user just learns that a refresh didn't land.
- **Automatic retries** for transient issues, which only work if errors
  propagate.

You can express "background vs. initial" by inspecting the cache: if the query
already has data, the failure happened during a background refetch.

```jsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Only throw to the error boundary when there's no cached data —
      // i.e. the initial load failed and there's nothing to show.
      throwOnError: (error, query) =>
        typeof query.state.data === "undefined",
    },
  },
  queryCache: new QueryCache({
    onError: (error, query) => {
      // We have cached data, so this was a background refetch —
      // keep the stale UI and just notify.
      if (typeof query.state.data !== "undefined") {
        toast.error(`Background sync failed: ${error.message}`);
      }
    },
  }),
});
```

One global config, and every query in the app gets sensible defaults:
first-load failures hit the nearest boundary, background hiccups surface as a
single toast, and nothing ever fails silently.

## Takeaways

- React Query needs a rejected Promise — with `fetch`, throw on non-OK
  responses yourself.
- Never catch-and-swallow inside a `queryFn`; it kills retries, fakes a
  `success` status, and hides failures from users.
- Use `throwOnError` (optionally as a function) to route errors to error
  boundaries — 4xx handled locally, 5xx propagated.
- Per-query callbacks fire once per observer; for once-per-request
  notifications and monitoring, use the global `QueryCache` / `MutationCache`
  callbacks.
- Combine them: error boundaries for initial loads, toasts for background
  refetches, retries for transient failures — and let React Query own the
  error lifecycle throughout.

## References

- [React Query Error Handling](https://tkdodo.eu/blog/react-query-error-handling) — TkDodo (Dominik Dorfmeister)
- [Proper Error Handling in React Query](https://tigerabrodi.blog/proper-error-handling-in-react-query) — Tiger Abrodi
