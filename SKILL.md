---
name: state-management-sharqiewicz
description: "Expert guidance for React state management using Zustand, React Query, React Context, and XState. Backed by TK-Dodo's articles and official docs. Triggers on: Zustand, React Query, TanStack Query, XState, server state, client state, state management, useQuery, useMutation, context, store, staleTime, queryKey, invalidateQueries, mutations, optimistic updates, derived state, useEffect sync."
---

# React State Management

Expert guidance for Zustand, React Query, React Context, and XState.

---

## Quick Reference

| Tool | When to Use | Reference |
|------|-------------|-----------|
| **React Query** | Server/async state: data fetched from an API | [react-query.md](react-query.md) |
| **Zustand** | Global client state: filters, cart, UI toggles shared across components | [zustand.md](zustand.md) |
| **React Context** | Dependency injection: pass guaranteed data to a subtree | [react-context.md](react-context.md) |
| **XState Store** | Event-driven client state; upgrade path to machines | [xstate.md](xstate.md) |
| **XState Machine** | Complex flows with mutually exclusive states, guards, actors | [xstate.md](xstate.md) |
| **useState** | Local component state that isn't shared | — |
| **Integration patterns** | Zustand+RQ, Context+RQ, Router loaders, global error handling | [patterns.md](patterns.md) |

---

## Core Principles

### 1. Categorize state by type, not location

Server state (async, remote, can become stale) → **React Query**
Client state (sync, owned locally) → **Zustand** or **useState**
Mandatory shared data for a subtree → **React Context** (as DI)
Complex flows with distinct modes → **XState**

### 2. Query keys are your dependency array

Put everything that affects a query into its key. Let the key drive refetching declaratively. Never pass arguments to `refetch()` — update state instead and let the key change.

```typescript
// ❌ Static key + refetch with args
const { refetch } = useQuery({ queryKey: ['todos'], queryFn: fetchTodos })
refetch({ filters }) // doesn't work

// ✅ Dynamic key driven by state
const filters = useFilters()
useQuery({ queryKey: ['todos', filters], queryFn: () => fetchTodos(filters) })
```

### 3. Derive, don't sync

Compute derived values from server state at render time. Never `useEffect` to copy server state into local state.

```typescript
// ❌ Sync — two sources of truth
useEffect(() => { if (data) setLocal(data) }, [data])

// ✅ Derive — single source of truth
const selectedUser = users?.find((u) => u.id === selectedId) ?? null
```

### 4. Context is dependency injection

Use Context to make data dependencies explicit and type-safe. It is not a state manager. Pass query data directly to Context value — never copy it into `useState`.

### 5. Actions describe events, not setters

Keep business logic inside stores. Actions emit what happened (`increasePopulation`), not what to do (`setBears`). Components describe intent, stores decide implementation.

---

## Decision Flowcharts

### What kind of state is this?

```
Does it come from a server/API?
├── Yes → React Query (useQuery / useMutation)
│         └── Need to pass guaranteed data to subtree?
│             ├── Entire subtree needs it → React Context as DI wrapper
│             └── Few components need it → useSuspenseQuery + Suspense
└── No
    ├── Is it UI state tied to one component? → useState / useReducer
    ├── Is it shared global UI state? → Zustand
    │   └── Need isolation per instance / props-based init?
    │       └── Zustand + React Context (store instance in context)
    └── Is it a complex flow with distinct states? → XState
        ├── Simple event-driven, no explicit states? → XState Store
        └── Multi-step / guards / actors needed? → XState Machine
```

### How complex is my state update logic?

```
Single value toggle / increment?
└── useState

Multiple related values, updated together?
└── Zustand store with actions namespace

State has mutually exclusive modes (loading/idle/error/success)?
└── XState Machine — impossible states become impossible

State depends on API response?
└── React Query — manages loading/error/success for you
```

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Exporting raw Zustand store | Export custom selector hooks only |
| Object/array selectors without `useShallow` | Use atomic selectors or `useShallow` |
| `useEffect` syncing server → client state | Derive with `find()` or use RQ data directly |
| Static query keys with dynamic data | Embed all dependencies in query key |
| `refetch()` with arguments | Update state, let key drive fetch |
| `onSuccess` to sync to other state | v5: removed; use data directly or `select` |
| Copying query data into Context state | Pass query data directly as Context value |
| Context + useState for state management | Use Zustand or React Query instead |
| Manually specifying TypeScript generics in useQuery | Type the `queryFn` return, let TS infer |
| Inline `select` function causing re-renders | Stabilise with `useCallback` |
| Creating QueryClient inside component | Create outside component or with `useState(() => new QueryClient())` |
| Object selectors in Zustand without shallow | Use atomic selectors per value |
| Business logic scattered in components | Move to store actions |
| Conditional rendering soup (nested ternaries) | Layout component + early returns (one return = one visible state) |
| Optimistic updates everywhere | Use `invalidateQueries` by default; optimistic only when UX justifies complexity |

---

## Review Checklist

- [ ] Server state uses React Query, not `useState` + `useEffect`
- [ ] Client state uses Zustand with exported selector hooks (not raw store)
- [ ] Query keys include all variables they depend on
- [ ] `queryOptions()` factory used for reusable queries
- [ ] No `refetch()` calls with arguments — state drives keys
- [ ] No `useEffect` syncing server state to local state
- [ ] Status checks: `data` check before `error` check (data-first pattern)
- [ ] Error handling uses one of: `isError` flag, `throwOnError`, or `QueryCache onError`
- [ ] Mutations use `invalidateQueries` (not `setQueryData`) unless response is complete
- [ ] TypeScript types inferred from `queryFn` return, not manually specified with generics
- [ ] Zustand selectors are atomic (single values) or use `useShallow` for objects
- [ ] Actions namespace used for Zustand actions
- [ ] Context is used as DI (wrapping data, not managing mutable state)
- [ ] Global `QueryClient` and `MutationCache` handle cross-cutting concerns
- [ ] `QueryClient` created once (outside component or via `useState(() => ...)`)
- [ ] Component async states use early returns + layout component, not conditional soup

---

## Reference Files

- **[zustand.md](zustand.md)** — 6 tips (export hooks, atomic selectors, actions namespace, event model, small scope, Zustand+Context pattern) + TypeScript patterns
- **[react-query.md](react-query.md)** — Mental model, staleTime, query key factories, `queryOptions` API, status checks, error strategies, `select`, mutations, render optimisations, TypeScript, testing, prefetching
- **[react-context.md](react-context.md)** — Context as DI, explicit dependency pattern, `useSuspenseQuery` alternative, waterfall tradeoff, anti-patterns, component composition
- **[xstate.md](xstate.md)** — XState Store (lightweight), XState Machine (full state machines), decision guide: Store vs Machine vs Zustand
- **[patterns.md](patterns.md)** — Zustand+RQ integration, Context+RQ, derive vs sync, Router loaders, global error/mutation handling, XState+RQ
