# XState Reference
---

## Two Tools: Store vs Machine

XState ships two distinct tools:

| | XState Store (`@xstate/store`) | XState Machine (`xstate`) |
|---|---|---|
| Complexity | Low | Medium–High |
| Mental model | Event-driven key-value state | Finite state machine + statecharts |
| States | No explicit states | Explicit, mutually exclusive states |
| Best for | Zustand replacement with event semantics | Complex async flows, multi-step UIs |
| Upgrade path | → XState Machine | — |
| TypeScript | Automatic from initial context | Defined in machine config |

---

## XState Store: Lightweight Event-Driven State

### When to choose over Zustand

- You want **automatic TypeScript inference** (no manual generic setup)
- Your team thinks in **events** (`increasePopulation`) rather than setters (`setBears`)
- You want a **natural upgrade path** to full XState machines later
- You want `useSelector` **enforced** (preventing accidental full-store subscriptions)
- The state is **framework-agnostic** (same store used in React, Vue, vanilla JS)

### Basic usage

```typescript
import { createStore } from '@xstate/store'
import { useSelector } from '@xstate/store/react'

const store = createStore({
  // Initial context (state values)
  context: {
    bears: 0,
    fish: 0,
  },
  // Event handlers — each key is an event type
  on: {
    increasePopulation: {
      bears: (context, event: { by: number }) => context.bears + event.by,
    },
    eatFish: {
      fish: (context) => context.fish - 1,
    },
    reset: {
      bears: () => 0,
      fish: () => 0,
    },
  },
})

// Send events (not function calls)
store.send({ type: 'increasePopulation', by: 10 })
store.send({ type: 'eatFish' })

// React integration — selector is required (enforces subscription hygiene)
function BearCounter() {
  const bears = useSelector(store, (state) => state.context.bears)
  return <div>{bears}</div>
}

function Controls() {
  return (
    <button onClick={() => store.send({ type: 'increasePopulation', by: 1 })}>
      Add Bear
    </button>
  )
}
```

### TypeScript — automatic inference

```typescript
// ✅ No manual type setup needed — all inferred from initial context
const store = createStore({
  context: {
    count: 0,           // inferred: number
    name: '',           // inferred: string
    items: [] as Item[], // inferred: Item[]
  },
  on: {
    increment: {
      count: (context, event: { by: number }) => context.count + event.by,
      // `context` is typed, `event` typed from annotation
    },
  },
})

// store.getSnapshot().context.count is typed as number
// store.send({ type: 'increment', by: 'wrong' }) — TypeScript error
```

### Framework-agnostic core

```typescript
// Same store works without React
const snapshot = store.getSnapshot()
console.log(snapshot.context.bears)

store.subscribe((snapshot) => {
  console.log('State changed:', snapshot.context)
})

// React integration is a thin adapter
import { useSelector } from '@xstate/store/react'
```

### Comparison: XState Store vs Zustand

| Feature | XState Store | Zustand |
|---------|--------------|---------|
| TypeScript inference | Automatic | Manual generic setup |
| Update pattern | `store.send({ type, payload })` | Direct function calls |
| Selector enforcement | Required | Optional (easy footgun) |
| Framework binding | Opt-in adapter | React-specific by default |
| Upgrade to state machine | Natural path | Full rewrite |
| Actions namespace | Built-in via `on` | Manual convention |
| Middleware (persist, devtools) | Via XState inspect | Via Zustand middleware |
| Multiple instances | Recreate store | Zustand + Context pattern |

---

## XState Machines: Full State Machines

### When to reach for machines

- UI has **mutually exclusive modes** (idle / loading / error / success) — impossible states should be impossible
- **Multi-step flows** (wizard, checkout, auth flow)
- **Complex async operations** with branching success/failure paths
- **Actor model** needed (child machines, spawned actors)
- You want to **visualise** the state logic (XState Inspector / Stately Studio)
- State transitions need **guards** (conditional transitions based on context)

### Core concepts

```
State machine =
  finite set of states +
  transitions between states (triggered by events) +
  actions (side effects on enter/exit/transition) +
  context (extended state — arbitrary data) +
  guards (conditions that gate transitions)
```

### Basic machine

```typescript
import { createMachine, assign } from 'xstate'
import { useMachine } from '@xstate/react'

const fetchMachine = createMachine({
  id: 'fetch',
  initial: 'idle',
  context: {
    data: null as Todo[] | null,
    error: null as Error | null,
  },
  states: {
    idle: {
      on: {
        FETCH: 'loading', // event → new state
      },
    },
    loading: {
      invoke: {
        src: 'fetchTodos',    // service (async function)
        onDone: {
          target: 'success',
          actions: assign({ data: ({ event }) => event.output }),
        },
        onError: {
          target: 'failure',
          actions: assign({ error: ({ event }) => event.error as Error }),
        },
      },
    },
    success: {
      on: {
        FETCH: 'loading', // allow refetch
      },
    },
    failure: {
      on: {
        RETRY: 'loading',
      },
    },
  },
})
```

### React integration

```typescript
import { useMachine } from '@xstate/react'

function TodoList() {
  const [state, send] = useMachine(fetchMachine, {
    actors: {
      fetchTodos: () => fetchTodosFromApi(),
    },
  })

  // State checks are exhaustive and type-safe
  if (state.matches('loading')) return <Skeleton />
  if (state.matches('failure')) {
    return (
      <div>
        <ErrorMessage error={state.context.error} />
        <button onClick={() => send({ type: 'RETRY' })}>Retry</button>
      </div>
    )
  }
  if (state.matches('idle')) {
    return <button onClick={() => send({ type: 'FETCH' })}>Load</button>
  }

  // state.matches('success') — data is guaranteed
  return <ul>{state.context.data?.map(...)}</ul>
}
```

### Guards: conditional transitions

```typescript
const checkoutMachine = createMachine({
  // ...
  states: {
    cart: {
      on: {
        CHECKOUT: {
          target: 'payment',
          guard: ({ context }) => context.items.length > 0, // only if cart not empty
        },
      },
    },
    payment: {
      on: {
        SUBMIT: [
          {
            target: 'processing',
            guard: ({ context }) => context.paymentMethod !== null,
          },
          { target: 'payment' }, // fallback — stay in payment if no method
        ],
      },
    },
  },
})
```

### Actions: side effects

```typescript
const machine = createMachine({
  // ...
  states: {
    success: {
      entry: [
        // Runs when entering success state
        ({ context }) => toast.success('Saved!'),
        assign({ savedAt: () => Date.now() }),
      ],
      exit: [
        // Runs when leaving success state
        ({ context }) => analytics.track('form_saved'),
      ],
    },
  },
})
```

### Parallel states (multiple active states)

```typescript
const playerMachine = createMachine({
  type: 'parallel', // both child states active simultaneously
  states: {
    playback: {
      initial: 'paused',
      states: {
        paused: { on: { PLAY: 'playing' } },
        playing: { on: { PAUSE: 'paused' } },
      },
    },
    volume: {
      initial: 'unmuted',
      states: {
        unmuted: { on: { MUTE: 'muted' } },
        muted: { on: { UNMUTE: 'unmuted' } },
      },
    },
  },
})

// state.matches({ playback: 'playing', volume: 'muted' }) — both axes
```

### Actors: spawning child machines

```typescript
import { createMachine, spawn, assign } from 'xstate'

const parentMachine = createMachine({
  context: { childRef: null as ActorRef<any> | null },
  states: {
    active: {
      entry: assign({
        childRef: ({ spawn: spawnFn }) => spawnFn(childMachine),
      }),
    },
  },
})
```

### Visualising with XState Inspector

```typescript
import { inspect } from '@xstate/inspect'

// In development only
if (process.env.NODE_ENV === 'development') {
  inspect({ iframe: false }) // opens in new window
}

// Pass inspect option to useMachine
const [state, send] = useMachine(machine, { inspect })
```

---

## Decision: Store vs Machine vs Zustand

```
State has distinct modes (loading/idle/error/success)?
├── Yes, AND transitions between them have business rules (guards)?
│   └── XState Machine
├── Yes, but transitions are simple?
│   ├── Will you need actors or spawning? → XState Machine
│   └── No → XState Store or Zustand (with careful status field)
└── No (just values that change)
    ├── Want automatic TS inference + event-driven pattern? → XState Store
    ├── Need per-instance state with props init? → Zustand + Context
    └── Standard global client state? → Zustand
```

### Feature comparison table

| Feature | Zustand | XState Store | XState Machine |
|---------|---------|--------------|----------------|
| TypeScript setup | Manual generics | Automatic | Config-based |
| Explicit states | No | No | Yes |
| Impossible states enforced | No | No | Yes |
| Guards (conditional transitions) | No | No | Yes |
| Parallel states | No | No | Yes |
| Actor model / child machines | No | No | Yes |
| Visualisation | No | No | Yes (Stately) |
| Upgrade path | → Zustand+Context | → XState Machine | — |
| Bundle size | Small | Small | Medium |
| Learning curve | Low | Low | Medium |

---

## XState v5 Migration Notes

XState v5 (current) changed the API significantly from v4:

```typescript
// v4 (old)
import { createMachine, assign } from 'xstate'
const machine = createMachine({
  schema: { context: {} as Context, events: {} as Event },
  context: { count: 0 },
  states: {
    active: {
      on: { INCREMENT: { actions: assign({ count: (ctx) => ctx.count + 1 }) } },
    },
  },
})

// v5 (current) — destructured context from event, no schema needed
const machine = createMachine({
  context: { count: 0 },
  states: {
    active: {
      on: {
        INCREMENT: {
          actions: assign({ count: ({ context }) => context.count + 1 }),
          // ^^ v5: context comes from destructured object, not first arg
        },
      },
    },
  },
})
```

Key v5 changes:
- `assign` receives `({ context, event })` object, not `(context, event)` args
- `invoke.src` can be an inline actor creator
- `useMachine` from `@xstate/react` (unchanged import)
- `createActor(machine)` replaces `interpret(machine)` for vanilla JS
