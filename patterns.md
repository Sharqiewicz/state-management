# Integration Patterns Reference

Cross-cutting patterns that combine Zustand, React Query, Context, XState, and React Router.

---

## Zustand + React Query: Declarative Integration

The golden rule: **Zustand owns the filters; React Query owns the data.** Never pass arguments to `refetch()`.

```typescript
// filterStore.ts
const useFilterStore = create<FilterState>((set) => ({
  filters: {
    status: 'all' as TodoStatus,
    page: 1,
    search: '',
  },
  actions: {
    setStatus: (status: TodoStatus) =>
      set((state) => ({ filters: { ...state.filters, status, page: 1 } })),
    setSearch: (search: string) =>
      set((state) => ({ filters: { ...state.filters, search, page: 1 } })),
    nextPage: () =>
      set((state) => ({ filters: { ...state.filters, page: state.filters.page + 1 } })),
  },
}))

export const useFilters = () => useFilterStore((state) => state.filters)
export const useFilterActions = () => useFilterStore((state) => state.actions)

// todos.queries.ts
export const useFilteredTodos = () => {
  const filters = useFilters()
  return useQuery({
    queryKey: ['todos', filters],   // filters in key — drives all refetching
    queryFn: () => fetchTodos(filters),
    placeholderData: keepPreviousData, // smooth pagination
  })
}

// Component — filter change triggers automatic refetch via key change
function FilterBar() {
  const { setStatus, setSearch } = useFilterActions()
  return (
    <div>
      <input onChange={(e) => setSearch(e.target.value)} />
      <select onChange={(e) => setStatus(e.target.value as TodoStatus)}>
        <option value="all">All</option>
        <option value="active">Active</option>
        <option value="done">Done</option>
      </select>
    </div>
  )
}

function TodoList() {
  const { data, isPending, isError } = useFilteredTodos()
  // No need to wire up filter changes — the key handles it
}
```

**Never do this:**
```typescript
// ❌ refetch() doesn't accept arguments — this pattern doesn't work
const { refetch } = useQuery({ queryKey: ['todos'], queryFn: fetchTodos })
refetch({ filters }) // BUG: filters ignored
```

---

## Context + React Query: Explicit Data Dependencies

Use when an entire subtree needs guaranteed, type-safe access to server data.

```typescript
// auth.context.tsx
interface AuthUser {
  id: string
  name: string
  role: 'admin' | 'user'
  tenantId: string
}

const AuthUserContext = createContext<AuthUser | null>(null)

export const useAuthUser = (): AuthUser => {
  const user = useContext(AuthUserContext)
  if (!user) throw new Error('useAuthUser must be within AuthUserProvider')
  return user
}

export const AuthUserProvider = ({ children }: { children: ReactNode }) => {
  const { data, isPending, isError, error } = useQuery({
    queryKey: ['auth', 'me'],
    queryFn: fetchCurrentUser,
    staleTime: 5 * 60 * 1000,
    retry: 1,
  })

  if (isPending) return <AppSkeleton />
  if (isError) return <LoginRedirect error={error} />

  return (
    <AuthUserContext.Provider value={data}>
      {children}
    </AuthUserContext.Provider>
  )
}

// Any child component gets guaranteed AuthUser — no loading checks needed
function UserMenu() {
  const user = useAuthUser() // AuthUser, not AuthUser | undefined
  return <span>Hello, {user.name}</span>
}

// Child queries can use the guaranteed user data
function UserTodos() {
  const { tenantId } = useAuthUser()
  const { data: todos } = useQuery({
    queryKey: ['todos', tenantId],
    queryFn: () => fetchTodosForTenant(tenantId),
  })
  // ...
}
```

---

## Deriving State Instead of Syncing

Compute values from server data at render time. Never `useEffect` to maintain a synchronised copy.

```typescript
// ❌ Sync pattern — two sources of truth, stale state risk
const { data: users } = useQuery({ ... })
const [selectedUserId, setSelectedUserId] = useState<string | null>(null)
const [selectedUser, setSelectedUser] = useState<User | null>(null)

useEffect(() => {
  // Complex sync logic that misses edge cases
  const user = users?.find((u) => u.id === selectedUserId) ?? null
  setSelectedUser(user)
}, [users, selectedUserId])

// ✅ Derive at render time — single source of truth
const { data: users } = useQuery({ ... })
const [selectedUserId, setSelectedUserId] = useState<string | null>(null)

// Computed every render — always fresh, no sync needed
const selectedUser = users?.find((u) => u.id === selectedUserId) ?? null
const isValidSelection = selectedUserId === null || selectedUser !== null

// Benefits:
// - If user is deleted from server, selectedUser automatically becomes null
// - If user reappears (undo delete), selection is automatically restored
// - No stale state between renders
```

### The `validSelected` pattern

```typescript
// Extract into a custom hook for reuse
export const useUserSelection = () => {
  const [selectedId, setSelectedId] = useState<string | null>(null)
  const { data: users, ...queryResult } = useQuery({
    queryKey: userKeys.all,
    queryFn: fetchUsers,
  })

  const selectedUser = users?.find((u) => u.id === selectedId) ?? null
  const isValidSelection = selectedId === null || selectedUser !== null

  // Auto-select first user when none selected and data loads
  const effectiveUser = selectedUser ?? users?.[0] ?? null

  return {
    ...queryResult,
    users,
    selectedUser: effectiveUser,
    selectedId,
    isValidSelection,
    setSelectedId,
  }
}
```

---

## React Router + React Query: Loaders

Loaders prefetch data before the component mounts, eliminating the loading state entirely.

```typescript
// queryOptions factory — shared between loader and component
// todos.options.ts
export const todoDetailOptions = (id: string) =>
  queryOptions({
    queryKey: todoKeys.detail(id),
    queryFn: () => fetchTodo(id),
    staleTime: 5 * 60 * 1000,
  })

// router.ts
export const router = createBrowserRouter([
  {
    path: '/todos/:id',
    loader: async ({ params }) => {
      // ensureQueryData: uses cache if fresh, fetches if stale
      await queryClient.ensureQueryData(todoDetailOptions(params.id!))
      return null // data is now in the RQ cache
    },
    Component: TodoDetailPage,
  },
])

// TodoDetailPage — no loading state needed (data pre-loaded by router)
function TodoDetailPage() {
  const { id } = useParams()
  const { data: todo } = useQuery(todoDetailOptions(id!))
  // data is guaranteed — component only renders after loader resolves
  return <TodoDetail todo={todo!} />
}
```

### List → detail prefetch pattern

```typescript
// Hover over list item → prefetch detail
function TodoListItem({ todo }: { todo: Todo }) {
  const queryClient = useQueryClient()

  return (
    <Link
      to={`/todos/${todo.id}`}
      onMouseEnter={() => {
        queryClient.prefetchQuery(todoDetailOptions(todo.id))
      }}
    >
      {todo.title}
    </Link>
  )
}
```

---

## Component Composition for Async State

Each early return represents one visible state. Use a layout component to avoid repeating shared structure.

```typescript
// Layout component extracts shared structure
function TodoDetailLayout({ children }: { children: ReactNode }) {
  return (
    <PageContainer>
      <BackButton />
      <main>{children}</main>
    </PageContainer>
  )
}

function TodoDetailPage() {
  const { id } = useParams()
  const { data: todo, isPending, isError, error } = useQuery(todoDetailOptions(id!))

  if (isPending) {
    return <TodoDetailLayout><TodoDetailSkeleton /></TodoDetailLayout>
  }

  if (isError) {
    return (
      <TodoDetailLayout>
        <ErrorMessage error={error} />
        <RetryButton />
      </TodoDetailLayout>
    )
  }

  // TypeScript knows todo is Todo (not Todo | undefined) here
  return (
    <TodoDetailLayout>
      <TodoDetailContent todo={todo} />
    </TodoDetailLayout>
  )
}
```

---

## Global Error & Mutation Handling

Centralise cross-cutting concerns in QueryClient configuration instead of duplicating logic in every component.

```typescript
// queryClient.ts
export const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error, query) => {
      // Only show toast for background errors (where data already exists)
      // First-load errors are handled by component isError checks
      if (query.state.data !== undefined) {
        toast.error(`Update failed: ${error.message}`)
      }
    },
  }),

  mutationCache: new MutationCache({
    onSuccess: (_data, _variables, _context, mutation) => {
      // Opt-in success messages via meta
      if (mutation.meta?.successMessage) {
        toast.success(mutation.meta.successMessage as string)
      }
    },
    onError: (error, _variables, _context, mutation) => {
      // Global mutation error handling
      if (mutation.meta?.errorMessage) {
        toast.error(mutation.meta.errorMessage as string)
      } else {
        toast.error(`Something went wrong: ${error.message}`)
      }
    },
  }),
})

// Usage — opt into global toasts via meta
const createTodoMutation = useMutation({
  mutationFn: createTodo,
  meta: {
    successMessage: 'Todo created!',
    errorMessage: 'Failed to create todo',
  },
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: todoKeys.lists() })
  },
})
```

---

## XState + React Query: Complex Async Flows

When a user interaction triggers multiple dependent async operations with branching logic, combine XState machine with React Query mutations.

```typescript
// checkout.machine.ts
const checkoutMachine = createMachine({
  id: 'checkout',
  initial: 'cart',
  context: {
    orderId: null as string | null,
    error: null as Error | null,
  },
  states: {
    cart: {
      on: {
        PROCEED: {
          target: 'payment',
          guard: ({ context }) => context.items?.length > 0,
        },
      },
    },
    payment: {
      on: {
        SUBMIT: 'processing',
        BACK: 'cart',
      },
    },
    processing: {
      invoke: {
        src: 'submitOrder',
        onDone: {
          target: 'confirmation',
          actions: assign({ orderId: ({ event }) => event.output.id }),
        },
        onError: {
          target: 'payment',
          actions: assign({ error: ({ event }) => event.error as Error }),
        },
      },
    },
    confirmation: {
      type: 'final',
    },
  },
})

// CheckoutFlow.tsx
function CheckoutFlow() {
  const [state, send] = useMachine(checkoutMachine, {
    actors: {
      // Wire React Query mutation as XState actor
      submitOrder: fromPromise(({ input }: { input: OrderInput }) =>
        submitOrderMutation.mutateAsync(input)
      ),
    },
  })

  if (state.matches('cart')) return <CartStep onProceed={() => send({ type: 'PROCEED' })} />
  if (state.matches('payment')) return <PaymentStep onSubmit={() => send({ type: 'SUBMIT' })} error={state.context.error} />
  if (state.matches('processing')) return <LoadingOverlay message="Processing order..." />
  if (state.matches('confirmation')) return <OrderConfirmation orderId={state.context.orderId} />
}
```
