# React Context Reference
---

## Core Rule: Context Is Dependency Injection, Not State Management

React Context is not a state manager. It is a **dependency injection** mechanism — it lets you define what a subtree needs and lets parents provide it, skipping prop drilling.

```
Context = "who provides this data"
State management = "how this data changes over time"
```

Use Context to make data dependencies **explicit and type-safe**, not to hold mutable state.

---

## Pattern: Explicit Dependency via Context Provider

**The problem:** Components that call `useQuery` internally create an implicit, invisible dependency on that data being available. If the component is used outside the expected subtree, you get a silent failure or undefined data.

```typescript
// ❌ Implicit dependency — runtime error risk
function UserProfile() {
  const { data: user } = useQuery({ queryKey: ['user'], queryFn: fetchUser })
  // What if user is undefined? What if this renders before auth completes?
  return <div>{user.name}</div> // TypeError: Cannot read properties of undefined
}
```

```typescript
// ✅ Explicit dependency via Context Provider
import { createContext, useContext, ReactNode } from 'react'
import { useQuery } from '@tanstack/react-query'

interface User {
  id: string
  name: string
  settings: UserSettings
}

const CurrentUserContext = createContext<User | null>(null)

// Invariant hook — throws descriptively if used outside provider
export const useCurrentUser = (): User => {
  const user = useContext(CurrentUserContext)
  if (!user) {
    throw new Error(
      'useCurrentUser must be used within CurrentUserContextProvider. ' +
      'Ensure the component is rendered inside the authenticated area.'
    )
  }
  return user
}

// Provider owns loading/error states — children see only happy path
export const CurrentUserContextProvider = ({
  children,
}: {
  children: ReactNode
}) => {
  const { data, isPending, isError, error } = useQuery({
    queryKey: ['currentUser'],
    queryFn: fetchCurrentUser,
    staleTime: 5 * 60 * 1000,
  })

  if (isPending) return <SkeletonLoader />
  if (isError) return <ErrorMessage error={error} />

  // data is guaranteed to be defined here
  return (
    <CurrentUserContext.Provider value={data}>
      {children}
    </CurrentUserContext.Provider>
  )
}

// Usage — TypeScript knows `user` is User, not User | undefined
function UserProfile() {
  const user = useCurrentUser()
  return <div>{user.name}</div>
}
```

**Benefits:**
- TypeScript narrowing: `useCurrentUser()` returns `User`, not `User | undefined`
- Self-documenting: the provider in the tree communicates what data is available
- Fail-fast: missing provider throws a clear error, not a cryptic undefined access
- Centralised loading/error UI for the whole subtree

---

## Pattern: useSuspenseQuery Alternative

React Query v5's `useSuspenseQuery` achieves the same type-safety without a Context wrapper.

```typescript
import { useSuspenseQuery } from '@tanstack/react-query'

function UserProfile() {
  // data is guaranteed to be defined — component suspends until loaded
  const { data: user } = useSuspenseQuery({
    queryKey: ['currentUser'],
    queryFn: fetchCurrentUser,
  })
  // TypeScript knows user is User (not User | undefined)
  return <div>{user.name}</div>
}

// Wrap with Suspense + ErrorBoundary at the appropriate level
import { Suspense } from 'react'
import { ErrorBoundary } from 'react-error-boundary'

function App() {
  return (
    <ErrorBoundary fallback={<ErrorMessage />}>
      <Suspense fallback={<SkeletonLoader />}>
        <UserProfile />
      </Suspense>
    </ErrorBoundary>
  )
}
```

### Context Provider vs useSuspenseQuery

| Consideration | Context Provider | useSuspenseQuery |
|---------------|-----------------|-----------------|
| Loading/error centralisation | At provider level | At Suspense/ErrorBoundary level |
| Multiple components need same data | Single fetch, shared via context | Each component fetches (deduped by RQ) |
| Waterfall risk | Yes — subtree waits for provider | No — components fetch in parallel |
| Works without Suspense setup | Yes | No — requires Suspense boundary |
| Data used in many disconnected components | Better | Equivalent (RQ cache deduplication) |

**Recommendation:** For data that the **entire subtree** depends on (auth user, tenant config), use the Context Provider pattern to avoid repetition. For data used by a few components, use `useSuspenseQuery` directly.

---

## The Waterfall Tradeoff

Context providers that wait for data before rendering children create **request waterfalls** — child components cannot fire their own queries until the parent data resolves.

```typescript
// ⚠️ Waterfall: user loads first, THEN todos can start loading
<CurrentUserContextProvider>    {/* T=0: starts user fetch */}
  <TodoList />                  {/* T=user_done: starts todos fetch */}
</CurrentUserContextProvider>

// ✅ Parallel: both start immediately
<QueryClientProvider client={queryClient}>
  <UserProfile />   {/* T=0: starts user fetch */}
  <TodoList />      {/* T=0: starts todos fetch — no dependency */}
</QueryClientProvider>
```

**Rule:** Only use the Context Provider pattern for data that is **mandatory** for the entire subtree and truly has no alternative rendering path — typically:
- Authenticated user object (the whole app needs it after login)
- Tenant/organisation config
- Feature flags loaded once at startup

Do NOT wrap independent data fetches in Context providers just to avoid prop drilling — let components fetch their own data in parallel.

---

## Anti-Patterns

### Context + useState for state management

```typescript
// ❌ Context is not a state manager
const [user, setUser] = useState<User | null>(null)
const UserContext = createContext({ user, setUser })

// Problems:
// - All consumers re-render on every state change
// - No loading/error handling
// - No cache, no deduplication, no stale-time
// Use Zustand for global client state, React Query for server state
```

### Copying query data into Context state

```typescript
// ❌ Creates a second source of truth that can drift
const { data } = useQuery({ queryKey: ['user'], queryFn: fetchUser })
const [contextValue, setContextValue] = useState(data)
useEffect(() => setContextValue(data), [data])

// ✅ Pass query data directly as Context value
const CurrentUserContext = createContext<User | null>(null)

const Provider = ({ children }: { children: ReactNode }) => {
  const { data } = useQuery({ queryKey: ['user'], queryFn: fetchUser })
  // Pass data directly — one source of truth
  return (
    <CurrentUserContext.Provider value={data ?? null}>
      {children}
    </CurrentUserContext.Provider>
  )
}
```

### Silent failures (if (!data) return null)

```typescript
// ❌ Silently renders nothing — invisible to the developer
function UserProfile() {
  const { data: user } = useQuery({ queryKey: ['user'], queryFn: fetchUser })
  if (!data) return null  // No explanation why; no loading state shown
  return <div>{user.name}</div>
}

// ✅ Explicit states — one return per visible state
function UserProfile() {
  const { data: user, isPending, isError, error } = useQuery({
    queryKey: ['user'],
    queryFn: fetchUser,
  })

  if (isPending) return <Skeleton />
  if (isError) return <ErrorMessage error={error} />

  return <div>{user.name}</div>
}
```

---

## Component Composition for State States

React's power is composition. Use a **layout component** pattern with early returns instead of conditional rendering soup.

### Anti-pattern: conditional rendering soup

```typescript
// ❌ Hard to read; TypeScript cannot narrow types inside JSX conditionals
export function ShoppingList() {
  const { data, isPending, isError } = useQuery({ ... })

  return (
    <Card>
      <CardHeading>Shopping List</CardHeading>
      <CardContent>
        {isPending ? <Skeleton /> : null}
        {isError ? <ErrorMessage /> : null}
        {!data && !isPending && !isError ? <EmptyScreen /> : null}
        {data && data.items.length > 0 ? (
          data.items.map((item) => <ShoppingItem key={item.id} {...item} />)
        ) : null}
        {data && data.items.length === 0 ? <EmptyScreen /> : null}
      </CardContent>
    </Card>
  )
}
```

### Solution: layout component + early returns

```typescript
// ✅ Clear mental model — one return = one visible state
function ShoppingListLayout({ children }: { children: ReactNode }) {
  return (
    <Card>
      <CardHeading>Shopping List</CardHeading>
      <CardContent>{children}</CardContent>
    </Card>
  )
}

export function ShoppingList() {
  const { data, isPending, isError, error } = useQuery({
    queryKey: ['shopping-list'],
    queryFn: fetchShoppingList,
  })

  if (isPending) {
    return <ShoppingListLayout><Skeleton /></ShoppingListLayout>
  }

  if (isError) {
    return <ShoppingListLayout><ErrorMessage error={error} /></ShoppingListLayout>
  }

  if (data.items.length === 0) {
    return <ShoppingListLayout><EmptyScreen message="No items yet" /></ShoppingListLayout>
  }

  // TypeScript knows: data exists and has items
  return (
    <ShoppingListLayout>
      {data.assignee && <UserInfo {...data.assignee} />}
      {data.items.map((item) => (
        <ShoppingItem key={item.id} {...item} />
      ))}
    </ShoppingListLayout>
  )
}
```

Repeating `<ShoppingListLayout>` in each branch is **intentional and correct**:
- Each branch can diverge its layout independently in the future
- Intent is immediately clear
- Avoids the "wrong abstraction" where one layout tries to handle all cases

---

## When to Use Each Pattern

| Scenario | Pattern |
|----------|---------|
| Entire subtree requires guaranteed data | Context Provider with `useQuery` |
| Few components need type-safe data | `useSuspenseQuery` + Suspense boundary |
| Independent per-component server data | Direct `useQuery` in component |
| Multiple instances need isolated state | Zustand + Context (store instance) |
| Global mutable client state | Zustand |
| Complex flows with distinct modes | XState |
