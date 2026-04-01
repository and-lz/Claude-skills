# Components & UI Patterns

## Navigation

### Top navigation bar
```tsx
// Semantic nav with skip link for accessibility
export function TopNav() {
  return (
    <>
      <a href="#main-content" className="sr-only focus:not-sr-only focus:fixed focus:top-4 focus:left-4 focus:z-50 focus:px-4 focus:py-2 focus:bg-white focus:rounded">
        Skip to content
      </a>
      <header className="sticky top-0 z-40 border-b bg-background/95 backdrop-blur supports-[backdrop-filter]:bg-background/60">
        <nav aria-label="Main navigation" className="container flex h-16 items-center justify-between">
          <Logo />
          <ul className="hidden md:flex items-center gap-6" role="list">
            <li><a href="/features" className="text-sm font-medium text-muted-foreground hover:text-foreground transition-colors">Features</a></li>
            <li><a href="/pricing" className="text-sm font-medium text-muted-foreground hover:text-foreground transition-colors">Pricing</a></li>
          </ul>
          <div className="flex items-center gap-2">
            <Button variant="ghost" asChild><a href="/login">Sign in</a></Button>
            <Button asChild><a href="/signup">Get started</a></Button>
          </div>
        </nav>
      </header>
    </>
  )
}
```

### Sidebar (app shell)
```tsx
'use client'
import { useUIStore } from '@/store/ui'

export function Sidebar() {
  const { sidebarOpen, toggleSidebar } = useUIStore()

  return (
    <>
      {/* Mobile overlay */}
      {sidebarOpen && (
        <div
          className="fixed inset-0 z-20 bg-black/50 md:hidden"
          onClick={toggleSidebar}
          aria-hidden="true"
        />
      )}

      <aside
        id="sidebar"
        className={cn(
          'fixed left-0 top-0 z-30 h-full w-64 border-r bg-background transition-transform duration-200',
          'md:sticky md:translate-x-0',
          sidebarOpen ? 'translate-x-0' : '-translate-x-full'
        )}
        aria-label="Sidebar navigation"
      >
        <nav className="flex flex-col gap-1 p-4">
          <SidebarLink href="/dashboard" icon={LayoutDashboard}>Dashboard</SidebarLink>
          <SidebarLink href="/users" icon={Users}>Users</SidebarLink>
          <SidebarLink href="/settings" icon={Settings}>Settings</SidebarLink>
        </nav>
      </aside>
    </>
  )
}

function SidebarLink({ href, icon: Icon, children }: SidebarLinkProps) {
  return (
    <a
      href={href}
      className="flex items-center gap-3 rounded-md px-3 py-2 text-sm font-medium text-muted-foreground hover:bg-accent hover:text-accent-foreground transition-colors aria-[current=page]:bg-accent aria-[current=page]:text-accent-foreground"
    >
      <Icon className="h-4 w-4" aria-hidden="true" />
      {children}
    </a>
  )
}
```

### Breadcrumbs
```tsx
export function Breadcrumbs({ items }: { items: { label: string; href?: string }[] }) {
  return (
    <nav aria-label="Breadcrumb">
      <ol className="flex items-center gap-2 text-sm">
        {items.map((item, i) => (
          <li key={item.label} className="flex items-center gap-2">
            {i > 0 && <ChevronRight className="h-4 w-4 text-muted-foreground" aria-hidden="true" />}
            {item.href && i < items.length - 1 ? (
              <a href={item.href} className="text-muted-foreground hover:text-foreground transition-colors">
                {item.label}
              </a>
            ) : (
              <span aria-current={i === items.length - 1 ? 'page' : undefined}>{item.label}</span>
            )}
          </li>
        ))}
      </ol>
    </nav>
  )
}
```

## Buttons

```tsx
// Button variants (shadcn/ui pattern)
const buttonVariants = cva(
  'inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent hover:text-accent-foreground',
        secondary: 'bg-secondary text-secondary-foreground hover:bg-secondary/80',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline',
      },
      size: {
        default: 'h-9 px-4 py-2',
        sm: 'h-8 rounded-md px-3 text-xs',
        lg: 'h-10 rounded-md px-8',
        icon: 'h-9 w-9',
      },
    },
    defaultVariants: { variant: 'default', size: 'default' },
  }
)

// Icon button — always needs aria-label
<button className={buttonVariants({ size: 'icon', variant: 'ghost' })} aria-label="Open settings">
  <Settings className="h-4 w-4" aria-hidden="true" />
</button>

// Loading button
function LoadingButton({ loading, children, ...props }: ButtonProps & { loading?: boolean }) {
  return (
    <button {...props} disabled={loading || props.disabled} aria-busy={loading}>
      {loading && <Loader2 className="h-4 w-4 animate-spin" aria-hidden="true" />}
      {children}
    </button>
  )
}
```

## Cards

```tsx
// Stat card
function StatCard({ label, value, change, icon: Icon }: StatCardProps) {
  const isPositive = change >= 0
  return (
    <article className="rounded-lg border bg-card p-6 shadow-sm">
      <div className="flex items-center justify-between">
        <span className="text-sm font-medium text-muted-foreground">{label}</span>
        <Icon className="h-4 w-4 text-muted-foreground" aria-hidden="true" />
      </div>
      <div className="mt-2 flex items-end justify-between">
        <span className="text-2xl font-bold tabular-nums">{value}</span>
        <span className={cn('text-xs font-medium', isPositive ? 'text-green-600' : 'text-red-600')}>
          <span className="sr-only">{isPositive ? 'Increased by' : 'Decreased by'}</span>
          {isPositive ? '+' : ''}{change}%
        </span>
      </div>
    </article>
  )
}
```

## Data table

```tsx
// Accessible data table with sorting
function DataTable<T>({ data, columns }: DataTableProps<T>) {
  const [sort, setSort] = useState<{ key: keyof T; dir: 'asc' | 'desc' } | null>(null)

  const sorted = useMemo(() => {
    if (!sort) return data
    return [...data].sort((a, b) => {
      const cmp = String(a[sort.key]).localeCompare(String(b[sort.key]))
      return sort.dir === 'asc' ? cmp : -cmp
    })
  }, [data, sort])

  function toggleSort(key: keyof T) {
    setSort(prev =>
      prev?.key === key
        ? { key, dir: prev.dir === 'asc' ? 'desc' : 'asc' }
        : { key, dir: 'asc' }
    )
  }

  return (
    <div className="rounded-md border overflow-x-auto">
      <table className="w-full text-sm" aria-label="Data table">
        <thead>
          <tr className="border-b bg-muted/50">
            {columns.map(col => (
              <th
                key={String(col.key)}
                scope="col"
                className="px-4 py-3 text-left font-medium"
                aria-sort={sort?.key === col.key ? (sort.dir === 'asc' ? 'ascending' : 'descending') : 'none'}
              >
                {col.sortable ? (
                  <button
                    onClick={() => toggleSort(col.key)}
                    className="flex items-center gap-1 hover:text-foreground"
                  >
                    {col.label}
                    <ChevronsUpDown className="h-3 w-3" aria-hidden="true" />
                  </button>
                ) : col.label}
              </th>
            ))}
          </tr>
        </thead>
        <tbody>
          {sorted.map((row, i) => (
            <tr key={i} className="border-b last:border-0 hover:bg-muted/30 transition-colors">
              {columns.map(col => (
                <td key={String(col.key)} className="px-4 py-3">
                  {col.render ? col.render(row) : String(row[col.key])}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  )
}
```

## Modal / Dialog

```tsx
// Using Radix UI Dialog for accessibility
import * as Dialog from '@radix-ui/react-dialog'

function Modal({ trigger, title, description, children }: ModalProps) {
  return (
    <Dialog.Root>
      <Dialog.Trigger asChild>{trigger}</Dialog.Trigger>
      <Dialog.Portal>
        <Dialog.Overlay className="fixed inset-0 z-50 bg-black/50 data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0" />
        <Dialog.Content className="fixed left-1/2 top-1/2 z-50 w-full max-w-lg -translate-x-1/2 -translate-y-1/2 rounded-lg bg-background p-6 shadow-lg data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95">
          <Dialog.Title className="text-lg font-semibold">{title}</Dialog.Title>
          {description && (
            <Dialog.Description className="mt-1 text-sm text-muted-foreground">
              {description}
            </Dialog.Description>
          )}
          <div className="mt-4">{children}</div>
          <Dialog.Close className="absolute right-4 top-4 rounded-sm opacity-70 hover:opacity-100 focus:outline-none focus:ring-2 focus:ring-ring" aria-label="Close">
            <X className="h-4 w-4" />
          </Dialog.Close>
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  )
}
```

## Toast notifications

```tsx
// Using sonner (recommended with Next.js)
import { toast } from 'sonner'

// Usage
toast.success('User created successfully')
toast.error('Failed to delete item')
toast.promise(saveUser(), {
  loading: 'Saving...',
  success: 'Saved!',
  error: 'Failed to save',
})

// Setup in root layout
import { Toaster } from 'sonner'
<Toaster position="bottom-right" richColors />
```

## Command palette

```tsx
// Using cmdk
'use client'
import { Command } from 'cmdk'
import { useEffect, useState } from 'react'

export function CommandPalette() {
  const [open, setOpen] = useState(false)

  useEffect(() => {
    const handler = (e: KeyboardEvent) => {
      if ((e.metaKey || e.ctrlKey) && e.key === 'k') {
        e.preventDefault()
        setOpen(prev => !prev)
      }
    }
    document.addEventListener('keydown', handler)
    return () => document.removeEventListener('keydown', handler)
  }, [])

  return (
    <Command.Dialog
      open={open}
      onOpenChange={setOpen}
      label="Command palette"
      className="fixed top-1/4 left-1/2 z-50 w-full max-w-xl -translate-x-1/2 rounded-xl border bg-background shadow-2xl overflow-hidden"
    >
      <Command.Input
        placeholder="Type a command or search..."
        className="h-12 w-full border-b bg-transparent px-4 text-sm outline-none placeholder:text-muted-foreground"
      />
      <Command.List className="max-h-72 overflow-y-auto p-2">
        <Command.Empty className="py-6 text-center text-sm text-muted-foreground">
          No results found.
        </Command.Empty>
        <Command.Group heading="Navigation">
          <Command.Item onSelect={() => { router.push('/dashboard'); setOpen(false) }}
            className="flex items-center gap-2 rounded-md px-3 py-2 text-sm cursor-pointer data-[selected]:bg-accent">
            <LayoutDashboard className="h-4 w-4" aria-hidden="true" />
            Dashboard
          </Command.Item>
        </Command.Group>
      </Command.List>
    </Command.Dialog>
  )
}
```

## Tabs

```tsx
import * as TabsPrimitive from '@radix-ui/react-tabs'

function Tabs({ items }: { items: { value: string; label: string; content: React.ReactNode }[] }) {
  return (
    <TabsPrimitive.Root defaultValue={items[0]?.value}>
      <TabsPrimitive.List
        className="flex border-b"
        aria-label="Tab navigation"
      >
        {items.map(item => (
          <TabsPrimitive.Trigger
            key={item.value}
            value={item.value}
            className="px-4 py-2 text-sm font-medium text-muted-foreground transition-colors hover:text-foreground data-[state=active]:text-foreground data-[state=active]:border-b-2 data-[state=active]:border-primary -mb-px"
          >
            {item.label}
          </TabsPrimitive.Trigger>
        ))}
      </TabsPrimitive.List>
      {items.map(item => (
        <TabsPrimitive.Content key={item.value} value={item.value} className="mt-4">
          {item.content}
        </TabsPrimitive.Content>
      ))}
    </TabsPrimitive.Root>
  )
}
```

## Skeleton loading

```tsx
function Skeleton({ className }: { className?: string }) {
  return (
    <div
      className={cn('animate-pulse rounded-md bg-muted', className)}
      aria-hidden="true"  // decorative — screen readers skip
    />
  )
}

// Page-level skeleton
function DashboardSkeleton() {
  return (
    <div aria-label="Loading dashboard..." aria-busy="true">
      <div className="grid grid-cols-4 gap-4">
        {Array.from({ length: 4 }).map((_, i) => (
          <Skeleton key={i} className="h-28" />
        ))}
      </div>
      <Skeleton className="mt-6 h-80" />
    </div>
  )
}
```

## Empty state

```tsx
function EmptyState({
  icon: Icon,
  title,
  description,
  action,
}: {
  icon: React.ElementType
  title: string
  description: string
  action?: React.ReactNode
}) {
  return (
    <div className="flex flex-col items-center justify-center py-16 text-center">
      <div className="rounded-full bg-muted p-4 mb-4">
        <Icon className="h-8 w-8 text-muted-foreground" aria-hidden="true" />
      </div>
      <h3 className="text-lg font-semibold">{title}</h3>
      <p className="mt-1 text-sm text-muted-foreground max-w-sm">{description}</p>
      {action && <div className="mt-4">{action}</div>}
    </div>
  )
}
```

## Accordion

```tsx
import * as AccordionPrimitive from '@radix-ui/react-accordion'

function Accordion({ items }: { items: { value: string; trigger: string; content: React.ReactNode }[] }) {
  return (
    <AccordionPrimitive.Root type="single" collapsible className="divide-y border rounded-md">
      {items.map(item => (
        <AccordionPrimitive.Item key={item.value} value={item.value}>
          <AccordionPrimitive.Header>
            <AccordionPrimitive.Trigger className="flex w-full items-center justify-between px-4 py-3 text-sm font-medium hover:bg-muted/50 transition-colors [&[data-state=open]>svg]:rotate-180">
              {item.trigger}
              <ChevronDown className="h-4 w-4 transition-transform duration-200" aria-hidden="true" />
            </AccordionPrimitive.Trigger>
          </AccordionPrimitive.Header>
          <AccordionPrimitive.Content className="overflow-hidden text-sm data-[state=open]:animate-accordion-down data-[state=closed]:animate-accordion-up">
            <div className="px-4 pb-4 pt-2 text-muted-foreground">{item.content}</div>
          </AccordionPrimitive.Content>
        </AccordionPrimitive.Item>
      ))}
    </AccordionPrimitive.Root>
  )
}
```

## Common page patterns

### Dashboard layout
```tsx
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex h-screen overflow-hidden">
      <Sidebar />
      <div className="flex flex-1 flex-col overflow-hidden">
        <TopBar />
        <main id="main-content" className="flex-1 overflow-y-auto p-6">
          {children}
        </main>
      </div>
    </div>
  )
}
```

### Landing page hero
```tsx
export function Hero() {
  return (
    <section className="container mx-auto px-4 py-24 text-center">
      <h1 className="text-4xl font-bold tracking-tight sm:text-6xl lg:text-7xl">
        Build faster,{' '}
        <span className="text-primary">ship better</span>
      </h1>
      <p className="mt-6 text-lg text-muted-foreground max-w-2xl mx-auto">
        The platform for modern web development. Deploy, scale, and monitor your applications with ease.
      </p>
      <div className="mt-10 flex flex-col sm:flex-row items-center justify-center gap-4">
        <Button size="lg" asChild>
          <a href="/signup">Get started free</a>
        </Button>
        <Button size="lg" variant="outline" asChild>
          <a href="/docs">Read the docs</a>
        </Button>
      </div>
    </section>
  )
}
```

### Auth page
```tsx
export default function LoginPage() {
  return (
    <main className="flex min-h-screen items-center justify-center p-4">
      <div className="w-full max-w-sm">
        <div className="text-center mb-8">
          <Logo className="mx-auto h-10 w-auto" />
          <h1 className="mt-4 text-2xl font-bold">Sign in to your account</h1>
          <p className="mt-2 text-sm text-muted-foreground">
            Don't have an account?{' '}
            <a href="/signup" className="font-medium text-primary hover:underline">Sign up</a>
          </p>
        </div>
        <LoginForm />
      </div>
    </main>
  )
}
```
