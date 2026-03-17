# Zustand Reference
---

## Tip 1: Export Custom Hooks, Never Raw Stores

The store itself is private. Components subscribe via exported selector hooks.

```typescript
// ❌ Raw store export — subscribes to ALL state changes
export const useBearStore = create<BearState>((set) => ({
  bears: 0,
  fish: 0,
  actions: { ... },
}))

// Component re-renders on ANY state change, even unrelated fields
const { bears } = useBearStore()
```

```typescript
// ✅ Private store, exported selector hooks
const useBearStore = create<BearState>((set) => ({
  bears: 0,
  fish: 0,
  actions: {
    increasePopulation: (by: number) =>
      set((state) => ({ bears: state.bears + by })),
    eatFish: () =>
      set((state) => ({ fish: state.fish - 1 })),
  },
}))

// Export granular hooks — each re-renders only when its slice changes
export const useBears = () => useBearStore((state) => state.bears)
export const useFish = () => useBearStore((state) => state.fish)
export const useBearActions = () => useBearStore((state) => state.actions)
```

**Why it matters:** Zustand re-renders subscribers whenever selected value changes via `Object.is`. Exporting the raw store makes it trivial to accidentally subscribe to everything.

---

## Tip 2: Prefer Atomic Selectors; Use `shallow` for Objects

### Atomic selectors (preferred)

```typescript
// ✅ One hook per primitive value — zero wasted renders
const bears = useBears()
const fish = useFish()
```

### When you must return an object

Object selectors create a new reference every render, which always triggers a re-render since `Object.is({}, {}) === false`.

```typescript
// ❌ New object every render — always re-renders
const { bears, fish } = useBearStore((state) => ({
  bears: state.bears,
  fish: state.fish,
}))

// ✅ shallow comparison — only re-renders when values change
import { useShallow } from 'zustand/react/shallow'

const { bears, fish } = useBearStore(
  useShallow((state) => ({ bears: state.bears, fish: state.fish }))
)
```

> **Note:** In Zustand v4 the import is `import { shallow } from 'zustand/shallow'` as the second argument. In Zustand v5 use `useShallow` from `zustand/react/shallow` as a wrapper.

### Array selectors

```typescript
// ✅ Same rule — useShallow for arrays
const items = useStore(useShallow((state) => state.items))
```

---

## Tip 3: Separate Actions from State

Group all mutations under an `actions` namespace. Actions functions are stable references — they never change — so selecting the entire `actions` object is safe without `shallow`.

```typescript
const useStore = create<StoreState>((set, get) => ({
  // --- State ---
  bears: 0,
  fish: 0,

  // --- Actions namespace (always stable) ---
  actions: {
    increasePopulation: (by: number) =>
      set((state) => ({ bears: state.bears + by })),
    eatFish: () =>
      set((state) => ({ fish: state.fish - 1 })),
    removeAllBears: () => set({ bears: 0 }),
    // Access other state via get()
    feedBears: () => {
      const { fish } = get()
      if (fish > 0) set((state) => ({ bears: state.bears + 1, fish: state.fish - 1 }))
    },
  },
}))

// Single hook for ALL actions — no re-render issues (actions are static)
export const useStoreActions = () => useStore((state) => state.actions)
```

**Why the actions namespace is safe to select together:** Functions are created once at store initialisation and never change. `Object.is(fn, fn) === true` always, so Zustand never triggers a re-render even though you're selecting an object.

---

## Tip 4: Model Actions as Events, Not Setters

Actions should describe **what happened** in domain language, not what to do to the store.

```typescript
// ❌ Setter pattern — leaks store implementation into components
// Component has to know about internal structure
const setBears = (n: number) => set({ bears: n })
// Called from component: setBears(bears + 1) — business logic in component

// ✅ Event pattern — encapsulates business logic in store
increasePopulation: (by: number) =>
  set((state) => ({ bears: state.bears + by }))

// Multi-step business logic stays in store
completeOrder: (orderId: string) =>
  set((state) => ({
    orders: state.orders.map((o) =>
      o.id === orderId ? { ...o, status: 'completed', completedAt: Date.now() } : o
    ),
    completedCount: state.completedCount + 1,
  })),
```

**Naming guide:**
- `increasePopulation` not `setPopulation`
- `addToCart` not `setCartItems`
- `submitForm` not `setFormSubmitting`
- `clearFilters` not `setFilters({})`

---

## Tip 5: Keep Store Scope Small

One monolithic store creates coupling and forces irrelevant re-renders. Prefer multiple focused stores combined with custom hooks.

```typescript
// ✅ Focused stores — independent concerns
const useCredentialsStore = create<CredentialsState>(...)
const useUsersStore = create<UsersState>(...)
const useFilterStore = create<FilterState>(...)

// Custom hook composes stores — still atomic subscriptions
export const useCurrentUserData = () => {
  const currentUserId = useCredentialsStore((state) => state.currentUserId)
  return useUsersStore((state) => state.users[currentUserId])
}

export const useFilteredUsers = () => {
  const filters = useFilterStore((state) => state.applied)
  return useUsersStore(
    useShallow((state) =>
      state.users.filter((u) => matchesFilters(u, filters))
    )
  )
}
```

Signs a store is too large:
- Actions in one part never touch state in another part
- Adding a feature requires reviewing the whole store
- Multiple teams/features share the same store file

---

## Tip 6: Zustand + React Context for Component-Scoped State

Use this pattern when you need:
- **Props-based initialisation** (store gets its initial values from parent)
- **Multiple independent instances** (e.g., multiple form instances on same page)
- **Testing isolation** (each test creates a fresh store without module mocking)

The key insight: put the **store instance** (not values) in Context.

```typescript
import { createStore, useStore, StoreApi } from 'zustand'
import { createContext, useContext, useRef, ReactNode } from 'react'

interface BearState {
  bears: number
  actions: {
    increasePopulation: (by: number) => void
    reset: () => void
  }
}

// Factory function (not a hook)
const createBearStore = (initialBears: number) =>
  createStore<BearState>((set) => ({
    bears: initialBears,
    actions: {
      increasePopulation: (by) =>
        set((state) => ({ bears: state.bears + by })),
      reset: () => set({ bears: initialBears }),
    },
  }))

// Context holds the store instance
const BearStoreContext = createContext<StoreApi<BearState> | null>(null)

// Provider creates ONE store instance per mount
export const BearStoreProvider = ({
  children,
  initialBears = 0,
}: {
  children: ReactNode
  initialBears?: number
}) => {
  // useRef so the store is created only once, even under StrictMode
  const storeRef = useRef<StoreApi<BearState>>(null)
  if (!storeRef.current) {
    storeRef.current = createBearStore(initialBears)
  }
  return (
    <BearStoreContext.Provider value={storeRef.current}>
      {children}
    </BearStoreContext.Provider>
  )
}

// Base hook — consumers always use selectors
const useBearStore = <T,>(selector: (state: BearState) => T): T => {
  const store = useContext(BearStoreContext)
  if (!store) throw new Error('useBearStore must be used within BearStoreProvider')
  return useStore(store, selector)
}

// Exported atomic hooks
export const useBears = () => useBearStore((state) => state.bears)
export const useBearActions = () => useBearStore((state) => state.actions)
```

Usage — two independent instances:

```tsx
function Page() {
  return (
    <div>
      <BearStoreProvider initialBears={5}>
        <ParkA />  {/* has its own bears: 5 */}
      </BearStoreProvider>

      <BearStoreProvider initialBears={10}>
        <ParkB />  {/* has its own bears: 10 */}
      </BearStoreProvider>
    </div>
  )
}
```

---

## TypeScript Patterns

### Typed state and actions

```typescript
interface StoreState {
  // State
  items: Item[]
  selectedId: string | null

  // Actions (always in namespace)
  actions: {
    addItem: (item: Item) => void
    selectItem: (id: string | null) => void
    removeItem: (id: string) => void
    reset: () => void
  }
}

const useStore = create<StoreState>((set, get) => ({
  items: [],
  selectedId: null,
  actions: {
    addItem: (item) =>
      set((state) => ({ items: [...state.items, item] })),
    selectItem: (id) =>
      set({ selectedId: id }),
    removeItem: (id) =>
      set((state) => ({ items: state.items.filter((i) => i.id !== id) })),
    reset: () =>
      set({ items: [], selectedId: null }),
  },
}))

export const useItems = () => useStore((state) => state.items)
export const useSelectedId = () => useStore((state) => state.selectedId)
export const useStoreActions = () => useStore((state) => state.actions)
```

### StoreApi type for Context pattern

```typescript
import { StoreApi } from 'zustand'

// When passing the store instance around
const MyContext = createContext<StoreApi<StoreState> | null>(null)

// Extracting state type from a store factory
type BearStore = ReturnType<typeof createBearStore>
type BearState = ReturnType<BearStore['getState']>
```

### Immer middleware (when state is deeply nested)

```typescript
import { immer } from 'zustand/middleware/immer'

const useStore = create<State>()(
  immer((set) => ({
    nested: { count: 0 },
    actions: {
      increment: () =>
        set((state) => {
          state.nested.count++ // mutate — immer handles immutability
        }),
    },
  }))
)
```

---

## Quick Decision: Zustand vs Zustand+Context

| Scenario | Use |
|----------|-----|
| Global singleton state (filters, theme, cart) | Plain Zustand |
| Needs props-based init at mount time | Zustand + Context |
| Multiple independent instances on same page | Zustand + Context |
| Testing isolation without module mocking | Zustand + Context |
| Complex async flows with states | Consider XState instead |
