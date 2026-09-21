---
title: "useState vs useReducer: When to Reach for a Reducer"
date: 2026-09-20T00:00:00+00:00
description: "React gives you two state primitives. Here's how to choose — with the stale-closure bugs, event-driven reducers, and decision rules from Kent C. Dodds, TkDodo, and Kyle Shevlin."
tags:
  - react
  - javascript
  - hooks
  - frontend
categories:
  - React

---

React gives you two primitives for managing local state: `useState` and
`useReducer`. Neither is the "old way" — they come with different trade-offs,
so the real question is which trade-offs suit your situation. This post
distils three excellent takes on the question — Kent C. Dodds'
[Should I useState or useReducer?](https://kentcdodds.com/blog/should-i-usestate-or-usereducer),
TkDodo's
[useState vs useReducer](https://tkdodo.eu/blog/use-state-vs-use-reducer), and
Kyle Shevlin's
[Why Use useReducer?](https://kyleshevlin.com/why-use-use-reducer/) — into a
set of practical decision rules.

## Two primitives, two shapes

`useState` hands you a `[read, write]` tuple:

```jsx
const [count, setCount] = React.useState(0);

console.log(count); // "read" the value
setCount(count + 1); // "write" the next value
```

`useReducer` hands you a `[read, emitEvent]` tuple. Shevlin deliberately names
them `emitEvent`/`event` instead of the usual `dispatch`/`action` to make the
mental model explicit: you're *emitting events* and *responding* to them:

```jsx
const reducer = (state, event) => {
  switch (event) {
    case "INCREMENT":
      return state + 1;
    default:
      return state;
  }
};

const [count, dispatch] = React.useReducer(reducer, 0);

console.log(count); // "read" the value
dispatch("INCREMENT"); // emit the event
```

`useState` is the sensible default. The interesting question is what makes
`useReducer` worth its extra ceremony.

## When useState wins: independent state

Kent's `useDarkMode` example shows a case where `useState` is clearly right —
a single independent value, initialized from `localStorage`/`matchMedia` and
kept in sync:

```jsx
function useDarkMode() {
  const preferDarkQuery = "(prefers-color-scheme: dark)";
  const [mode, setMode] = React.useState(
    () =>
      window.localStorage.getItem("colorMode") ||
      (window.matchMedia(preferDarkQuery).matches ? "dark" : "light"),
  );

  React.useEffect(() => {
    const mediaQuery = window.matchMedia(preferDarkQuery);
    const handleChange = () =>
      setMode(mediaQuery.matches ? "dark" : "light");
    mediaQuery.addListener(handleChange);
    return () => mediaQuery.removeListener(handleChange);
  }, []);

  React.useEffect(() => {
    window.localStorage.setItem("colorMode", mode);
  }, [mode]);

  return [mode, setMode];
}
```

The redux-style reducer version of the same hook needs a `switch`, action
constants, an `init` function, and a memoized `setMode` wrapper to preserve
the API — dramatically more code for zero benefit. And if you strip the
reducer down to `(prevMode, nextMode) => nextMode`, you've just reimplemented
`useState` with extra steps. Kent's rule:

> When it's just an independent element of state you're managing: `useState`.

A related guideline from TkDodo: **state that updates together should live
together**. Mouse coordinates `(x, y)` always change at once, so splitting
them into two `useState` calls is weirder than a single object. A simple
generic form hook is similar — one state object, one field updated at a time.

## When useReducer wins: state that depends on other state

Kent's counter-example is a `useUndo` hook tracking `past`, `present`, and
`future`. The naive implementation uses three separate `useState` calls:

```jsx
const undo = React.useCallback(() => {
  if (!canUndo) return;

  const previous = past[past.length - 1];
  const newPast = past.slice(0, past.length - 1);

  setPast(newPast);
  setPresent(previous);
  setFuture([present, ...future]);
}, [canUndo, future, past, present]);
```

This *looks* fine, but hides three stale-closure bugs (in `undo`, `redo`, and
`set`). Every updater closes over `past`, `present`, and `future`, so each one
must be re-created whenever any of those change — which makes the memoized
callbacks unstable, which makes effects that depend on them re-run, which (in
Kent's contrived-but-instructive demo) produces an infinite undo history.

You *can* fix it with `useState` — merge everything into one state object and
use the updater-callback form so each function receives `currentState` instead
of closing over it — but with `useReducer` the problem never arises:

```jsx
const UNDO = "UNDO";
const REDO = "REDO";
const SET = "SET";
const RESET = "RESET";

function undoReducer(state, action) {
  const { past, present, future } = state;

  switch (action.type) {
    case UNDO: {
      if (past.length === 0) return state;
      const previous = past[past.length - 1];
      const newPast = past.slice(0, past.length - 1);
      return {
        past: newPast,
        present: previous,
        future: [present, ...future],
      };
    }

    case REDO: {
      if (future.length === 0) return state;
      const next = future[0];
      const newFuture = future.slice(1);
      return {
        past: [...past, present],
        present: next,
        future: newFuture,
      };
    }

    case SET: {
      if (action.newPresent === present) return state;
      return {
        past: [...past, present],
        present: action.newPresent,
        future: [],
      };
    }

    case RESET: {
      return { past: [], present: action.newPresent, future: [] };
    }
  }
}

function useUndo(initialPresent) {
  const [state, dispatch] = React.useReducer(undoReducer, {
    past: [],
    present: initialPresent,
    future: [],
  });

  const canUndo = state.past.length !== 0;
  const canRedo = state.future.length !== 0;
  // dispatch is stable, so these callbacks never change — no dependency bugs
  const undo = React.useCallback(() => dispatch({ type: UNDO }), []);
  const redo = React.useCallback(() => dispatch({ type: REDO }), []);
  const set = React.useCallback(
    (newPresent) => dispatch({ type: SET, newPresent }),
    [],
  );
  const reset = React.useCallback(
    (newPresent) => dispatch({ type: RESET, newPresent }),
    [],
  );

  return [state, { set, reset, undo, redo, canUndo, canRedo }];
}
```

Because `dispatch` is stable and the reducer always receives the *current*
state, there's nothing to close over and no dependency array to get wrong.
Kent's second rule:

> When one element of your state relies on the value of another element of
> your state in order to update: `useReducer`.

## Model actions as events, not setters

TkDodo's key piece of advice — borrowed from the
[redux style guide](https://redux.js.org/style-guide/style-guide#model-actions-as-events-not-setters) —
is to keep the logic *inside* the reducer by treating actions as events:

```jsx
const reducer = (state, action) => {
  // ✅ UI only dispatches events; logic lives in the reducer
  switch (action) {
    case "increment":
      return state + 1;
    case "decrement":
      return state - 1;
  }
};

function App() {
  const [count, dispatch] = React.useReducer(reducer, 0);

  return (
    <div>
      Count: {count}
      <button onClick={() => dispatch("increment")}>Increment</button>
      <button onClick={() => dispatch("decrement")}>Decrement</button>
    </div>
  );
}
```

Compare that to a "dumb" reducer that merely accepts a new value:

```jsx
// 🚨 dumb reducer — the logic leaked into the UI
const reducer = (state, action) => {
  switch (action.type) {
    case "set":
      return action.value;
  }
};

<button onClick={() => dispatch({ type: "set", value: count + 1 })}>
  Increment
</button>
```

Both work today, but only the event-driven version can grow: adding an upper
bound, clamping, or changing the step amount touches the reducer alone. Pure
functions are also the easiest thing in the world to test. As a smell
check — **avoid actions with "set" in their name.**

While you're at it, follow the rest of the redux style guide too: don't
mutate state, and don't put side effects in reducers.

## Reducers surface hidden events

Shevlin's jumping-ball demo shows the organizational payoff. A ball jumps when
you click a button, arcs under "gravity", and ignores clicks while airborne.
The `useState` + `useRef` version smears the state logic across two functions
— `handleClick` mutates `delta` and `ballState`, while `tick` reads them and
updates `position`. To understand the ball, you have to hold both functions in
your head.

Refactored to a reducer, the component's handlers become trivial:

```jsx
function JumpingBall() {
  const [state, dispatch] = React.useReducer(reducer, initialState);

  const tick = React.useCallback(() => {
    dispatch("TICK");
  }, []);

  const handleClick = React.useCallback(() => {
    dispatch("CLICK");
  }, []);

  React.useEffect(() => {
    const id = setInterval(tick, 1000 / 60);
    return () => clearInterval(id);
  }, [tick]);
}
```

Two things fall out of this. First, the refactor *discovers* an event that
wasn't obvious before — `TICK` — and names it. Second, features get cheap.
Adding a double jump means adding `jumpsRemaining` to the state and one
condition to the reducer; the component and its handlers don't change at all.
And landing resets everything by simply returning `initialState` — one event,
many state updates, in a single fell swoop.

## Bonus: closing over props and server state

TkDodo highlights an underrated trick: because the reducer is just a function
called during render, you can inline it and *close over* props — or data from
a query — instead of copying values into initial state:

```jsx
const reducer = (amount) => (state, action) => {
  switch (action) {
    case "increment":
      return state + amount;
    case "decrement":
      return state - amount;
  }
};

const useCounterState = () => {
  const { data } = useQuery({
    queryKey: ["amount"],
    queryFn: fetchAmount,
  });
  // ✅ the reducer always sees the latest server state
  return React.useReducer(reducer(data ?? 1), 0);
};
```

Passing `data` via the initializer would capture `undefined` on the first
render (the fetch hasn't finished) and force you into sync-via-effects hacks.
The closure keeps server state and client state properly separated, and the UI
doesn't change at all.

## Takeaways

- `useState` for independent values; reach for `useReducer` when elements of
  state depend on each other to update.
- State that updates together should live together — even if that's one
  `useState` object rather than a reducer.
- Multiple related `useState` setters that close over each other breed
  stale-closure bugs; a reducer with a stable `dispatch` sidesteps the whole
  class.
- Model actions as events, not setters — keep the logic in the reducer where
  it's testable, and be suspicious of any action named `set*`.
- Reducers concentrate scattered state logic in one place, name your hidden
  events, and make follow-up features (double jump!) surprisingly cheap.
- It's not "more than N `useState`s → switch." Mix both primitives in one
  component, separated by domain; start with `useState` and graduate to
  `useReducer` when you notice state changing together.
