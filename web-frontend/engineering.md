# Engineering — Architecture, State, TypeScript, React 19, Data Fetching

## Project structure

Feature-based co-location — keep everything related to a feature together:

```
src/
├── app/                          # Next.js App Router
│   ├── (marketing)/              # Route group — no URL segment
│   │   ├── page.tsx
│   │   └── layout.tsx
│   ├── (app)/                    # Authenticated app shell
│   │   ├── dashboard/
│   │   │   ├── page.tsx
│   │   │   ├── loading.tsx
│   │   │   └── error.tsx
│   │   └── layout.tsx
│   ├── api/                      # Route Handlers
│   └── layout.tsx                # Root layout
├── features/                     # Feature modules
│   ├── auth/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── actions.ts            # Server Actions
│   │   └── types.ts
│   ├── dashboard/
│   └── users/
├── components/                   # Shared UI components
│   ├── ui/                       # shadcn/ui primitives
│   └── layout/                   # Shell, sidebar, nav
├── lib/                          # Shared utilities
│   ├── db.ts                     # Database client
│   ├── auth.ts                   # Auth config
│   └── utils.ts
└── types/                        # Global TypeScript types
```

**Rules:**
- Co-locate components, hooks, actions, and tests inside `features/`
- No barrel `index.ts` re-exports in `features/` — import directly to enable tree-shaking
- `components/ui/` is for design system primitives only; feature components live in `features/`
- `lib/` is for singletons and shared utilities; not for feature code

## TypeScript patterns

### Strict config
```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true
  }
}
```

### Zod for runtime validation — infer types from schemas
```typescript
import { z } from 'zod'

const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  name: z.string().min(1).max(100),
  role: z.enum(['admin', 'member', 'viewer']),
  createdAt: z.coerce.date(),
})

// Infer — single source of truth
type User = z.infer<typeof UserSchema>

// Validate at boundary
const result = UserSchema.safeParse(apiResponse)
if (!result.success) {
  console.error(result.error.flatten())
  throw new Error('Invalid user data')
}
const user = result.data // fully typed
```

### Discriminated unions for state
```typescript
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string }

// Usage — exhaustive narrowing
function render(state: AsyncState<User[]>) {
  switch (state.status) {
    case 'idle': return null
    case 'loading': return <Skeleton />
    case 'success': return <UserList users={state.data} />
    case 'error': return <ErrorMessage message={state.error} />
  }
}
```

### Branded types for domain safety
```typescript
type UserId = string & { readonly _brand: 'UserId' }
type PostId = string & { readonly _brand: 'PostId' }

function createUserId(id: string): UserId {
  return id as UserId
}

// Prevents passing wrong ID type
function getUser(id: UserId): Promise<User> { ... }
function getPost(id: PostId): Promise<Post> { ... }
```

### `server-only` module guard
```typescript
// lib/db.ts
import 'server-only' // throws at build if imported in client bundle

import { PrismaClient } from '@prisma/client'
export const db = new PrismaClient()
```

## React 19 features

### `use()` for promises
```typescript
// Server Component passes promise to Client Component
async function Page() {
  const userPromise = fetchUser() // don't await — pass the promise
  return <UserProfile promise={userPromise} />
}

// Client Component unwraps it with use()
'use client'
import { use } from 'react'

function UserProfile({ promise }: { promise: Promise<User> }) {
  const user = use(promise) // suspends until resolved
  return <div>{user.name}</div>
}
```

### `use()` for context (conditionally)
```typescript
'use client'
import { use } from 'react'

function Component({ condition }: { condition: boolean }) {
  // use() works in conditionals — useContext does not
  if (condition) {
    const theme = use(ThemeContext)
    return <div style={{ color: theme.primary }}>...</div>
  }
  return null
}
```

### `useActionState`
```typescript
'use client'
import { useActionState } from 'react'
import { submitForm } from './actions'

type FormState = { error?: string; success?: boolean }

function ContactForm() {
  const [state, action, isPending] = useActionState<FormState, FormData>(
    submitForm,
    { error: undefined, success: false }
  )

  return (
    <form action={action}>
      <input name="email" type="email" required />
      <button disabled={isPending}>
        {isPending ? 'Sending...' : 'Send'}
      </button>
      {state.error && <p role="alert">{state.error}</p>}
      {state.success && <p>Sent successfully!</p>}
    </form>
  )
}
```

### `useFormStatus`
```typescript
'use client'
import { useFormStatus } from 'react-dom'

// Must be a child of the <form> — not in the same component
function SubmitButton() {
  const { pending } = useFormStatus()
  return (
    <button type="submit" disabled={pending} aria-busy={pending}>
      {pending ? 'Saving...' : 'Save'}
    </button>
  )
}
```

### `useOptimistic`
```typescript
'use client'
import { useOptimistic, useTransition } from 'react'

function TodoList({ todos }: { todos: Todo[] }) {
  const [optimisticTodos, addOptimistic] = useOptimistic(
    todos,
    (state, newTodo: Todo) => [...state, newTodo]
  )
  const [isPending, startTransition] = useTransition()

  async function handleAdd(formData: FormData) {
    const text = formData.get('text') as string
    const tempTodo = { id: crypto.randomUUID(), text, done: false }

    startTransition(async () => {
      addOptimistic(tempTodo)
      await createTodo(text) // Server Action
    })
  }

  return (
    <div>
      {optimisticTodos.map(todo => <TodoItem key={todo.id} todo={todo} />)}
      <form action={handleAdd}>
        <input name="text" />
        <button>Add</button>
      </form>
    </div>
  )
}
```

### `ref` as prop (no more `forwardRef`)
```typescript
// React 19 — ref is a regular prop
function Input({ ref, ...props }: React.ComponentProps<'input'> & {
  ref?: React.Ref<HTMLInputElement>
}) {
  return <input ref={ref} {...props} />
}

// Usage
const inputRef = useRef<HTMLInputElement>(null)
<Input ref={inputRef} />
```

## State management hierarchy

Choose the right tool for each state type:

| State type | Tool | Example |
|-----------|------|---------|
| Server data (fetched) | Server Component + `fetch` | User list from API |
| Server data (client cache) | React Query / SWR | Paginated posts, polling |
| URL / shareable state | `useSearchParams` / `nuqs` | Filters, search, pagination |
| Form state | `useActionState` + Zod | Contact form, settings form |
| Local UI state | `useState` | Dropdown open, accordion |
| Shared client UI state | Zustand | Sidebar collapsed, theme |
| Async derived state | `use(promise)` | Passed server promises |

**Never use global state for server data.** React Query/SWR manage server state with caching, deduplication, background refresh, and loading/error states — Zustand cannot.

### Zustand store (client-only UI state)
```typescript
// store/ui.ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface UIStore {
  sidebarOpen: boolean
  theme: 'light' | 'dark' | 'system'
  toggleSidebar: () => void
  setTheme: (theme: UIStore['theme']) => void
}

export const useUIStore = create<UIStore>()(
  persist(
    (set) => ({
      sidebarOpen: true,
      theme: 'system',
      toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
      setTheme: (theme) => set({ theme }),
    }),
    { name: 'ui-store' }
  )
)
```

### React Query for client-side server state
```typescript
// features/users/hooks.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'

export function useUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: () => fetch('/api/users').then(r => r.json()),
    staleTime: 5 * 60 * 1000, // 5 minutes
  })
}

export function useUpdateUser() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (user: Partial<User> & { id: string }) =>
      fetch(`/api/users/${user.id}`, {
        method: 'PATCH',
        body: JSON.stringify(user),
      }),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['users'] }),
  })
}
```

### URL state with nuqs (shareable/bookmarkable)
```typescript
'use client'
import { useQueryState, parseAsInteger, parseAsString } from 'nuqs'

function ProductFilters() {
  const [page, setPage] = useQueryState('page', parseAsInteger.withDefault(1))
  const [search, setSearch] = useQueryState('q', parseAsString.withDefault(''))
  const [sort, setSort] = useQueryState('sort', parseAsString.withDefault('created'))

  return (
    <div>
      <input value={search} onChange={e => setSearch(e.target.value)} />
      <select value={sort} onChange={e => setSort(e.target.value)}>
        <option value="created">Newest</option>
        <option value="price">Price</option>
      </select>
    </div>
  )
}
```

## Data fetching patterns

### Server Component fetch (default)
```typescript
// app/users/page.tsx — Server Component, no 'use client'
async function UsersPage() {
  // Direct fetch — deduped by React, cached by Next.js Data Cache
  const users = await fetch('https://api.example.com/users', {
    next: { revalidate: 60, tags: ['users'] }, // ISR-style caching
  }).then(r => r.json())

  return <UserList users={users} />
}
```

### Parallel fetching (avoid waterfalls)
```typescript
async function DashboardPage() {
  // Initiate all fetches in parallel — don't await serially
  const [users, stats, activity] = await Promise.all([
    fetchUsers(),
    fetchStats(),
    fetchActivity(),
  ])

  return <Dashboard users={users} stats={stats} activity={activity} />
}
```

### Streaming with Suspense (progressive loading)
```typescript
import { Suspense } from 'react'

async function Page() {
  return (
    <div>
      <header>...</header>  {/* renders immediately */}
      <Suspense fallback={<StatsSkeleton />}>
        <Stats />  {/* streams in when ready */}
      </Suspense>
      <Suspense fallback={<TableSkeleton />}>
        <DataTable />  {/* independent — doesn't block Stats */}
      </Suspense>
    </div>
  )
}
```

### Error handling in async Server Components
```typescript
// Throw errors — caught by nearest error.tsx boundary
async function UserProfile({ id }: { id: string }) {
  const user = await db.user.findUnique({ where: { id } })
  if (!user) notFound() // triggers not-found.tsx
  return <Profile user={user} />
}
```

## Caching strategies

| Strategy | How | When to use |
|----------|-----|-------------|
| Static (SSG) | `fetch(url)` with no cache config | Marketing pages, docs |
| Time-based ISR | `fetch(url, { next: { revalidate: 60 } })` | Semi-static data (products, blog) |
| On-demand ISR | `revalidateTag('tag')` in Server Action | After mutations |
| No cache | `fetch(url, { cache: 'no-store' })` | Always-fresh data (auth, personal) |
| Client cache | React Query `staleTime` | Client-side with revalidation |

```typescript
// Tagging for on-demand revalidation
const data = await fetch('/api/products', {
  next: { tags: ['products'] }
})

// After mutation — Server Action
'use server'
import { revalidateTag } from 'next/cache'

async function deleteProduct(id: string) {
  await db.product.delete({ where: { id } })
  revalidateTag('products') // all fetches tagged 'products' revalidate
}
```

## Error handling patterns

### Route-level error boundary (error.tsx)
```typescript
// app/dashboard/error.tsx
'use client'

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <div role="alert">
      <h2>Something went wrong</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  )
}
```

### Typed error responses from Server Actions
```typescript
type ActionResult<T> =
  | { success: true; data: T }
  | { success: false; error: string; fields?: Record<string, string[]> }

async function createUser(formData: FormData): Promise<ActionResult<User>> {
  const parsed = CreateUserSchema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) {
    return {
      success: false,
      error: 'Validation failed',
      fields: parsed.error.flatten().fieldErrors,
    }
  }
  try {
    const user = await db.user.create({ data: parsed.data })
    revalidateTag('users')
    return { success: true, data: user }
  } catch (e) {
    return { success: false, error: 'Failed to create user' }
  }
}
```

## Code splitting and lazy loading

```typescript
import dynamic from 'next/dynamic'

// Lazy load heavy client components
const RichTextEditor = dynamic(() => import('@/components/RichTextEditor'), {
  loading: () => <Skeleton className="h-64" />,
  ssr: false, // skip SSR for browser-only components
})

const Chart = dynamic(() => import('@/components/Chart'), {
  loading: () => <ChartSkeleton />,
})

// React.lazy for non-Next.js contexts
import { lazy, Suspense } from 'react'
const HeavyComponent = lazy(() => import('./HeavyComponent'))
```

## API route patterns (Route Handlers)

```typescript
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { z } from 'zod'
import { auth } from '@/lib/auth'

const CreateUserBody = z.object({
  email: z.string().email(),
  name: z.string().min(1),
})

export async function POST(request: NextRequest) {
  const session = await auth()
  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const body = await request.json().catch(() => null)
  const parsed = CreateUserBody.safeParse(body)
  if (!parsed.success) {
    return NextResponse.json(
      { error: 'Invalid request', details: parsed.error.flatten() },
      { status: 400 }
    )
  }

  const user = await db.user.create({ data: parsed.data })
  return NextResponse.json(user, { status: 201 })
}
```

## Real-time patterns

### Server-Sent Events (one-way streaming)
```typescript
// app/api/events/route.ts
export async function GET() {
  const stream = new ReadableStream({
    start(controller) {
      const interval = setInterval(() => {
        const data = `data: ${JSON.stringify({ time: Date.now() })}\n\n`
        controller.enqueue(new TextEncoder().encode(data))
      }, 1000)

      // Clean up on close
      return () => clearInterval(interval)
    },
  })

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      Connection: 'keep-alive',
    },
  })
}

// Client
'use client'
useEffect(() => {
  const es = new EventSource('/api/events')
  es.onmessage = (e) => setData(JSON.parse(e.data))
  return () => es.close()
}, [])
```
