# React Query Reference
---

## Mental Model: Async State Manager, Not a Fetching Library

React Query does not care how you fetch. It manages the **state** around async operations — loading, error, background refresh, caching, deduplication.

**The problems it solves (that `useEffect` gets wrong):**

```typescript
// ❌ The "useEffect fetch" pattern — broken by default
function Component() {
  const [data, setData] = useState(null)
  const [loading, setLoading] = useState(false)

  useEffect(() => {
    setLoading(true)
    fetch('/api/todos')
      .then((r) => r.json())
      .then((data) => {
        setData(data)    // BUG: may arrive after component unmounts
        setLoading(false) // BUG: race condition if effect re-runs
      })
  }, [])
  // Missing: error handling, cancellation, refetch on focus, stale data, cache
}

// ✅ React Query handles all of this
function Component() {
  const { data, isPending, isError, error } = useQuery({
    queryKey: ['todos'],
    queryFn: () => fetch('/api/todos').then((r) => r.json()),
  })
}
```

React Query handles: race conditions, cancellation, deduplication, background refetching, stale-while-revalidate, error retry, loading states, cache sharing across components.

---

## Defaults & staleTime

Understanding defaults prevents unnecessary network requests and confusing behaviour:

| Default | Value | Meaning |
|---------|-------|---------|
| `staleTime` | `0` | Data is immediately stale; background refetch on every mount/focus |
| `gcTime` | `5 * 60 * 1000` | Unused cache entries garbage collected after 5 minutes |
| `retry` | `3` | Failed queries retry 3 times with exponential backoff |
| `refetchOnWindowFocus` | `true` | Refetches when user returns to tab |
| `refetchOnMount` | `true` | Refetches if data is stale when component mounts |

**`staleTime` is your primary lever.** Setting it higher reduces network requests.

```typescript
// Per-query
useQuery({
  queryKey: ['config'],
  queryFn: fetchConfig,
  staleTime: Infinity,      // Never refetch — config doesn't change
})

useQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId),
  staleTime: 5 * 60 * 1000, // Fresh for 5 minutes
})

// Global default — applies to all queries unless overridden
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000, // 1 minute
      retry: 1,             // reduce retries for faster failure feedback
    },
  },
})
```

---

## Query Keys as Dependency Arrays

Query keys are the most important concept in React Query. They:
1. Identify entries in the cache
2. Drive automatic refetching when they change
3. Enable targeted cache invalidation

**Everything that affects the query output must be in the key.**

```typescript
// ❌ Static key with dynamic data — cache never updates
useQuery({
  queryKey: ['tasks'],
  queryFn: () => fetchTasks(status, page), // status and page not in key!
})

// ✅ Dynamic key — cache updates when dependencies change
useQuery({
  queryKey: ['tasks', { status, page }],
  queryFn: () => fetchTasks(status, page),
})
```

### Key structure conventions

```typescript
// Recommended: hierarchical from general to specific
['todos']                           // all todos
['todos', todoId]                   // specific todo
['todos', 'list', filters]          // filtered list
['todos', 'detail', todoId]         // detail view
['user', userId, 'todos']           // user-scoped todos
```

### Query Key Factories pattern

Centralise key construction to avoid typos and enable targeted invalidation:

```typescript
// queryKeys.ts
export const todoKeys = {
  all: ['todos'] as const,
  lists: () => [...todoKeys.all, 'list'] as const,
  list: (filters: TodoFilters) => [...todoKeys.lists(), filters] as const,
  details: () => [...todoKeys.all, 'detail'] as const,
  detail: (id: string) => [...todoKeys.details(), id] as const,
}

// Usage
useQuery({
  queryKey: todoKeys.detail(todoId),
  queryFn: () => fetchTodo(todoId),
})

// Invalidate all todos
queryClient.invalidateQueries({ queryKey: todoKeys.all })

// Invalidate only lists (not details)
queryClient.invalidateQueries({ queryKey: todoKeys.lists() })
```

---

## The `queryOptions` API

`queryOptions()` is the v5 helper for creating reusable, composable query definitions. Prefer it over custom hooks when queries are shared across components and route loaders.

```typescript
import { queryOptions } from '@tanstack/react-query'

// Define once, use everywhere
const todoListOptions = (filters: TodoFilters) =>
  queryOptions({
    queryKey: todoKeys.list(filters),
    queryFn: () => fetchTodos(filters),
    staleTime: 60 * 1000,
  })

const todoDetailOptions = (id: string) =>
  queryOptions({
    queryKey: todoKeys.detail(id),
    queryFn: () => fetchTodo(id),
    staleTime: 5 * 60 * 1000,
  })

// In component
const { data } = useQuery(todoDetailOptions(todoId))

// In route loader (React Router v6.4+)
export const loader = ({ params }: LoaderFunctionArgs) =>
  queryClient.ensureQueryData(todoDetailOptions(params.id!))

// Prefetch
queryClient.prefetchQuery(todoDetailOptions(todoId))
```

Benefits over custom hooks:
- Works in non-component contexts (loaders, prefetching)
- Type inference is carried through automatically
- No hook dependency ordering issues

---

## Status Checks: Data-First Pattern

React Query v5 status model:

| Status | Meaning |
|--------|---------|
| `isPending` | No cached data, query hasn't resolved yet |
| `isSuccess` | Query resolved successfully; `data` is defined |
| `isError` | Query failed; `error` is defined |
| `isFetching` | Background refetch in progress (may have data) |

**The data-first pattern** avoids showing error states when data already exists:

```typescript
// ❌ isPending-first — hides data during background refetch
const { data, isPending, isError } = useQuery({ ... })

if (isPending) return <Skeleton />
if (isError) return <ErrorMessage />  // ← shown even when we have stale data!
return <DataDisplay data={data} />

// ✅ data-first — keeps showing data while refetching
const { data, isPending, isError, error } = useQuery({ ... })

if (data) {
  return <DataDisplay data={data} />  // Show data immediately if available
}
if (isPending) return <Skeleton />
if (isError) return <ErrorMessage error={error} />

// OR: check isPending first but show data regardless of isError
if (isPending) return <Skeleton />
// isError during background refetch still shows data
return <DataDisplay data={data} error={isError ? error : null} />
```

---

## Error Handling: 3 Strategies

### Strategy 1: `isError` flag (component-level)

```typescript
const { data, isError, error } = useQuery({ ... })

if (isError) {
  return <ErrorMessage message={error.message} />
}
```

Best for: inline errors that belong to a specific component's UI.

### Strategy 2: `throwOnError` (Error Boundary)

```typescript
useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  throwOnError: true,  // throws to nearest ErrorBoundary
})

// Or conditionally (throw only for 5xx, not 4xx)
throwOnError: (error) => error.status >= 500,
```

Best for: critical errors that should unmount the whole subtree.

### Strategy 3: `QueryCache onError` (global side effects)

```typescript
const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error, query) => {
      // Only toast for background refetch errors (data already exists)
      if (query.state.data !== undefined) {
        toast.error(`Background update failed: ${error.message}`)
      }
    },
  }),
})
```

Best for: toast notifications for background errors — avoids duplicate toasts since this fires once globally.

---

## Data Transformations: Use `select`

Four approaches to transforming server data, in order of preference:

```typescript
// 1. ✅ In queryFn (server → frontend type transformation)
queryFn: async () => {
  const data = await fetchTodos()
  return data.map(transformTodo) // runs on every fetch
}

// 2. ✅ select option (component-level derived data) — preferred for UI concerns
const { data: todoCount } = useQuery({
  queryKey: todoKeys.all,
  queryFn: fetchTodos,
  select: (todos) => todos.length, // data is now number
})

// Component only re-renders when count changes, not when other todo fields change
const { data: completedTodos } = useQuery({
  queryKey: todoKeys.all,
  queryFn: fetchTodos,
  select: (todos) => todos.filter((t) => t.completed),
})
```

### Memoize `select` for stable references

`select` runs on every render if it's an inline function. For objects/arrays, stabilise it:

```typescript
// ❌ New function every render — may cause extra re-renders
useQuery({
  select: (todos) => todos.filter((t) => t.status === status),
})

// ✅ Stable with useCallback
const selectCompleted = useCallback(
  (todos: Todo[]) => todos.filter((t) => t.status === status),
  [status]
)

useQuery({
  queryKey: todoKeys.list({ status }),
  queryFn: () => fetchTodos({ status }),
  select: selectCompleted,
})
```

---

## Mutations: Invalidation over Direct Updates

### invalidateQueries vs setQueryData

```typescript
const mutation = useMutation({
  mutationFn: (newTodo: NewTodo) => createTodo(newTodo),
  onSuccess: () => {
    // ✅ Preferred: invalidate and let React Query refetch
    queryClient.invalidateQueries({ queryKey: todoKeys.lists() })
  },
})

// setQueryData is appropriate ONLY when:
// - The mutation response is the complete, up-to-date resource
// - You want to skip the refetch network request
onSuccess: (createdTodo) => {
  queryClient.setQueryData(todoKeys.detail(createdTodo.id), createdTodo)
  // Still invalidate list to include the new item
  queryClient.invalidateQueries({ queryKey: todoKeys.lists() })
},
```

### `mutate` vs `mutateAsync`

```typescript
// mutate — fire-and-forget, callbacks for side effects
mutation.mutate(newTodo, {
  onSuccess: () => toast.success('Created'),
  onError: (error) => toast.error(error.message),
})

// mutateAsync — for await/try-catch, chaining promises
try {
  const todo = await mutation.mutateAsync(newTodo)
  router.push(`/todos/${todo.id}`)
} catch (error) {
  // Handle error
}
```

### Optimistic updates (use sparingly)

```typescript
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (updatedTodo) => {
    // Cancel in-flight queries to prevent overwriting optimistic update
    await queryClient.cancelQueries({ queryKey: todoKeys.detail(updatedTodo.id) })

    // Snapshot previous value for rollback
    const previousTodo = queryClient.getQueryData(todoKeys.detail(updatedTodo.id))

    // Optimistically update
    queryClient.setQueryData(todoKeys.detail(updatedTodo.id), updatedTodo)

    return { previousTodo }
  },
  onError: (err, updatedTodo, context) => {
    // Rollback on error
    queryClient.setQueryData(
      todoKeys.detail(updatedTodo.id),
      context?.previousTodo
    )
  },
  onSettled: (_, __, updatedTodo) => {
    // Always refetch to sync with server
    queryClient.invalidateQueries({ queryKey: todoKeys.detail(updatedTodo.id) })
  },
})
```

**Only use optimistic updates when the UX benefit clearly justifies the complexity.** For most mutations, invalidate-and-refetch is simpler and correct.

### Global MutationCache callbacks

```typescript
const queryClient = new QueryClient({
  mutationCache: new MutationCache({
    onSuccess: (data, variables, context, mutation) => {
      // Global success handling (e.g., show toast for specific mutations)
      if (mutation.meta?.successMessage) {
        toast.success(mutation.meta.successMessage as string)
      }
    },
    onError: (error) => {
      toast.error(`Something went wrong: ${error.message}`)
    },
  }),
})

// Usage in mutation
useMutation({
  mutationFn: createTodo,
  meta: { successMessage: 'Todo created!' },
})
```

---

## Never Sync Server State to Client State

This is the most important rule in React Query:

```typescript
// ❌ NEVER DO THIS — destroys single source of truth
const { data } = useQuery({ queryKey: ['user'], queryFn: fetchUser })

useEffect(() => {
  if (data) setLocalUser(data)  // Creates stale copy
}, [data])

// ❌ Also wrong — onSuccess removed in v5; but pattern is wrong in v4 too
onSuccess: (data) => {
  setLocalUser(data) // Two sources of truth that can diverge
}

// ✅ Use React Query data directly everywhere
const { data: user } = useQuery({ queryKey: ['user'], queryFn: fetchUser })
// Pass user directly to child components, derive values from it
const displayName = user?.firstName + ' ' + user?.lastName
```

**If you need to share server state across many components:** Use React Query's cache — all `useQuery` calls with the same key share a single cached value. No extra state needed.

---

## Render Optimizations

### Tracked queries (default in v5)

React Query v5 tracks which properties you access and only re-renders when those properties change:

```typescript
// Only re-renders when data changes, not when isFetching changes
const { data } = useQuery({ ... })

// Opt into more granular tracking
const { data, isFetching } = useQuery({
  notifyOnChangeProps: ['data', 'isFetching'], // explicit whitelist
})

// Always re-render on any change
const result = useQuery({
  notifyOnChangeProps: 'all',
})
```

### Structural sharing

React Query preserves object references for unchanged data. If a refetch returns data where 90% is unchanged, only the changed parts get new references. This means:

```typescript
// These references are stable across refetches if their data didn't change
const { data } = useQuery({ queryKey: ['todos'], queryFn: fetchTodos })
// data[0] is the SAME object reference if todo[0] didn't change on server
// Downstream memoization (useMemo, React.memo) works correctly
```

---

## TypeScript: Let Inference Work

### Type the queryFn, not the hook generics

```typescript
// ❌ Manual generics — verbose, error-prone, redundant
const { data } = useQuery<Todo[], Error, Todo[]>({
  queryKey: todoKeys.all,
  queryFn: fetchTodos, // already returns Promise<Todo[]>
})

// ✅ Type the queryFn return — TS infers everything else
const fetchTodos = async (): Promise<Todo[]> => {
  const response = await fetch('/api/todos')
  return response.json()
}

const { data } = useQuery({
  queryKey: todoKeys.all,
  queryFn: fetchTodos,
  // data: Todo[] | undefined — inferred automatically
})
```

### Zod for runtime validation

```typescript
import { z } from 'zod'

const TodoSchema = z.object({
  id: z.string(),
  title: z.string(),
  completed: z.boolean(),
})

const fetchTodo = async (id: string): Promise<Todo> => {
  const response = await fetch(`/api/todos/${id}`)
  const data = await response.json()
  return TodoSchema.parse(data) // throws if shape doesn't match
}
```

### Conditional queries with `skipToken`

```typescript
import { skipToken } from '@tanstack/react-query'

// ✅ v5 pattern — type-safe conditional queries
const { data } = useQuery({
  queryKey: todoKeys.detail(todoId),
  queryFn: todoId ? () => fetchTodo(todoId) : skipToken,
  // data: Todo | undefined; query disabled when todoId is null/undefined
})

// ❌ enabled option still works but loses some type inference
useQuery({
  queryKey: todoKeys.detail(todoId),
  queryFn: () => fetchTodo(todoId!),
  enabled: !!todoId,
})
```

---

## Testing

```typescript
// Per-test QueryClient — prevents shared state between tests
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'

const createTestQueryClient = () =>
  new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,     // Don't retry in tests — fail fast
        staleTime: Infinity, // Don't refetch during test
        gcTime: Infinity,
      },
    },
    logger: {
      log: console.log,
      warn: console.warn,
      error: () => {},  // Suppress error logs in tests
    },
  })

const renderWithQueryClient = (ui: ReactElement) => {
  const queryClient = createTestQueryClient()
  return render(
    <QueryClientProvider client={queryClient}>
      {ui}
    </QueryClientProvider>
  )
}

// Use Mock Service Worker (MSW) to intercept network requests
// msw provides realistic mocks without coupling to implementation details
```

---

## Prefetching & Seeding

### Prefetch in route loaders

```typescript
// router.ts — React Router v6.4+
export const loader = async ({ params }: LoaderFunctionArgs) => {
  // ensureQueryData: returns cached if fresh, fetches if stale
  await queryClient.ensureQueryData(todoDetailOptions(params.id!))
  return null
}

// Component — data is already in cache from loader
function TodoDetail() {
  const { id } = useParams()
  const { data } = useQuery(todoDetailOptions(id!))
  // No loading state needed — data is pre-loaded
}
```

### initialData vs placeholderData

```typescript
// initialData — treated as real cache data; sets updatedAt timestamp
useQuery({
  queryKey: todoKeys.detail(id),
  queryFn: () => fetchTodo(id),
  initialData: listData?.find((t) => t.id === id), // from list cache
  initialDataUpdatedAt: queryClient.getQueryState(todoKeys.lists())?.dataUpdatedAt,
})

// placeholderData — shown while loading; never put in cache
useQuery({
  queryKey: todoKeys.list(filters),
  queryFn: () => fetchTodos(filters),
  placeholderData: keepPreviousData, // show previous results while new page loads
})
```

**Use `placeholderData: keepPreviousData`** for pagination — keeps the current page visible while the next page loads, preventing layout shift.

---

## QueryClient Instantiation

```typescript
// ❌ Created inside component — new client on every render, cache lost
function App() {
  const queryClient = new QueryClient() // wrong!
  return <QueryClientProvider client={queryClient}>...</QueryClientProvider>
}

// ✅ Created outside component (module scope)
const queryClient = new QueryClient({ ... })

function App() {
  return <QueryClientProvider client={queryClient}>...</QueryClientProvider>
}

// ✅ Or lazy with useState (for SSR or testing)
function App() {
  const [queryClient] = useState(() => new QueryClient({ ... }))
  return <QueryClientProvider client={queryClient}>...</QueryClientProvider>
}
```
