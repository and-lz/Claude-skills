# Layout & Responsive Design

## Spacing system

4px base grid — all spacing is a multiple of 4px. Use Tailwind's spacing scale:

| Token | px | Tailwind |
|-------|----|---------|
| 1 | 4px | `p-1`, `m-1`, `gap-1` |
| 2 | 8px | `p-2`, `gap-2` |
| 3 | 12px | `p-3` |
| 4 | 16px | `p-4` (base unit) |
| 6 | 24px | `p-6` |
| 8 | 32px | `p-8` |
| 12 | 48px | `p-12` |
| 16 | 64px | `p-16` |
| 24 | 96px | `p-24` |

**Rules:**
- Component padding: `p-4` (16px) minimum, `p-6` (24px) comfortable
- Section spacing: `gap-6` to `gap-8` between components
- Page section padding: `py-16` to `py-24` for marketing pages
- Never use `margin: auto` — use `mx-auto` for centering containers

## Breakpoints

Tailwind v4 default breakpoints (min-width, mobile-first):

| Name | Width | Use for |
|------|-------|---------|
| `sm:` | 640px | Large phones, small tablets |
| `md:` | 768px | Tablets |
| `lg:` | 1024px | Laptops |
| `xl:` | 1280px | Desktops |
| `2xl:` | 1536px | Wide screens |

```tsx
// Mobile-first — base = smallest, add breakpoints upward
<div className="flex flex-col md:flex-row gap-4">
  <aside className="w-full md:w-64">Sidebar</aside>
  <main className="flex-1">Content</main>
</div>
```

### Custom breakpoints in Tailwind v4
```css
@theme {
  --breakpoint-xs: 480px;
  --breakpoint-3xl: 1920px;
}
```

## Mobile-first methodology

```tsx
// ✅ Mobile-first — build for small, enhance for large
<div className="
  grid grid-cols-1      /* 1 column on mobile */
  sm:grid-cols-2        /* 2 columns on small tablets */
  lg:grid-cols-3        /* 3 columns on desktop */
  gap-4
">

// ❌ Desktop-first — avoid
<div className="
  grid grid-cols-3
  max-md:grid-cols-2
  max-sm:grid-cols-1
">
```

## CSS Grid patterns

### 12-column grid
```css
.grid-12 {
  display: grid;
  grid-template-columns: repeat(12, minmax(0, 1fr));
  gap: 1.5rem;
}
```

```tsx
// Common spans
<div className="grid grid-cols-12 gap-6">
  <aside className="col-span-12 md:col-span-3">Sidebar</aside>
  <main  className="col-span-12 md:col-span-9">Content</main>
</div>

<div className="grid grid-cols-12 gap-6">
  <div className="col-span-12 lg:col-span-8">Main article</div>
  <div className="col-span-12 lg:col-span-4">Related</div>
</div>
```

### Auto-fill / auto-fit grid (responsive without breakpoints)
```tsx
// auto-fill — maintains minimum card width, fills space
<div className="grid gap-4" style={{ gridTemplateColumns: 'repeat(auto-fill, minmax(280px, 1fr))' }}>
  {items.map(item => <Card key={item.id} {...item} />)}
</div>

// Tailwind equivalent
<div className="grid grid-cols-[repeat(auto-fill,minmax(280px,1fr))] gap-4">
```

### Named grid areas
```css
.page-layout {
  display: grid;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
  grid-template-columns: 16rem 1fr;
  grid-template-rows: auto 1fr auto;
  min-height: 100dvh;
}

header { grid-area: header; }
aside  { grid-area: sidebar; }
main   { grid-area: main; }
footer { grid-area: footer; }

@media (max-width: 768px) {
  .page-layout {
    grid-template-areas:
      "header"
      "main"
      "sidebar"
      "footer";
    grid-template-columns: 1fr;
  }
}
```

## Flexbox patterns

### Holy grail layout
```tsx
// Sticky footer, sidebar, main content
<div className="flex flex-col min-h-dvh">
  <header className="shrink-0">Header</header>
  <div className="flex flex-1 overflow-hidden">
    <aside className="w-64 shrink-0 overflow-y-auto">Sidebar</aside>
    <main className="flex-1 overflow-y-auto p-6">Content</main>
  </div>
  <footer className="shrink-0">Footer</footer>
</div>
```

### Centering
```tsx
// Horizontal + vertical center
<div className="flex items-center justify-center min-h-screen">
  <Card />
</div>

// Center in a fixed container
<div className="fixed inset-0 flex items-center justify-center z-50">
  <Modal />
</div>
```

### Equal-height columns
```tsx
// Flexbox — all children same height
<div className="flex gap-4 items-stretch">
  <Card className="flex-1" />
  <Card className="flex-1" />
  <Card className="flex-1" />
</div>
```

## Container queries

```tsx
// @container — component adapts to available space
<div className="@container">
  <div className="flex flex-col @md:flex-row gap-4 p-4">
    <img className="w-full @md:w-48 rounded-lg object-cover" src={product.image} alt={product.name} />
    <div className="flex-1">
      <h3 className="text-base @md:text-lg font-semibold">{product.name}</h3>
      <p className="text-sm text-muted-foreground mt-1">{product.description}</p>
      <div className="mt-4 flex flex-col @sm:flex-row gap-2">
        <Button className="flex-1">Add to cart</Button>
        <Button variant="outline">Save</Button>
      </div>
    </div>
  </div>
</div>
```

**When to use container queries vs media queries:**
- **Container queries** — component adapts to its container (cards in sidebar vs main area)
- **Media queries** — layout changes based on viewport (page structure, navigation patterns)

## Common layout patterns

### Dashboard grid
```tsx
export function DashboardLayout() {
  return (
    <div className="flex h-dvh overflow-hidden bg-background">
      {/* Sidebar */}
      <aside className="hidden md:flex md:w-64 md:flex-col border-r">
        <div className="flex h-16 items-center px-6 border-b">
          <Logo />
        </div>
        <nav className="flex-1 overflow-y-auto py-4 px-3">
          <SidebarNav />
        </nav>
      </aside>

      {/* Main */}
      <div className="flex flex-1 flex-col overflow-hidden">
        <header className="flex h-16 items-center gap-4 border-b px-6">
          <MobileMenuButton className="md:hidden" />
          <div className="flex-1" />
          <UserMenu />
        </header>
        <main className="flex-1 overflow-y-auto">
          <div className="container max-w-7xl mx-auto p-6">
            {/* Page content */}
          </div>
        </main>
      </div>
    </div>
  )
}
```

### Marketing page
```tsx
export function MarketingLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="min-h-dvh flex flex-col">
      <TopNav />
      <main id="main-content" className="flex-1">
        {children}
      </main>
      <Footer />
    </div>
  )
}

// Constrained content width with full-bleed sections
export function Section({ children, className }: SectionProps) {
  return (
    <section className={cn('py-16 md:py-24', className)}>
      <div className="container mx-auto px-4 max-w-7xl">
        {children}
      </div>
    </section>
  )
}
```

### Card grid
```tsx
// Responsive grid — adapts without breakpoints
<ul
  className="grid gap-4"
  style={{ gridTemplateColumns: 'repeat(auto-fill, minmax(min(280px, 100%), 1fr))' }}
  role="list"
>
  {items.map(item => (
    <li key={item.id}>
      <Card {...item} />
    </li>
  ))}
</ul>
```

### Sticky header with scrollable content
```tsx
<div className="flex flex-col h-dvh">
  <header className="sticky top-0 z-40 border-b bg-background/95 backdrop-blur">
    <div className="container flex h-16 items-center">...</div>
  </header>
  <div className="flex-1 overflow-y-auto">
    <main className="container py-8">...</main>
  </div>
</div>
```

## Max-width and reading width

```tsx
// Container widths
<div className="container mx-auto px-4">           {/* max-w based on breakpoint */}
<div className="max-w-7xl mx-auto px-4">           {/* 1280px — wide dashboard */}
<div className="max-w-5xl mx-auto px-4">           {/* 1024px — content page */}
<div className="max-w-3xl mx-auto px-4">           {/* 768px — article, blog */}
<div className="max-w-prose mx-auto px-4">         {/* 65ch — reading width */}

// Prose (reading) width — never wider than 75 characters
<article className="prose prose-neutral dark:prose-invert max-w-prose mx-auto">
  {content}
</article>
```

## Viewport units

```css
/* ✅ Use dvh — adapts to mobile browser chrome */
.full-height { height: 100dvh; }
.min-full { min-height: 100dvh; }

/* ❌ Avoid vh — 100vh breaks on iOS Safari (URL bar cuts content) */
.broken { height: 100vh; }

/* svh — smallest viewport (URL bar visible) */
.safe-height { height: 100svh; }

/* lvh — largest viewport (URL bar hidden) */
.max-height { height: 100lvh; }
```

```tsx
// Tailwind — dvh available natively
<div className="min-h-dvh flex flex-col">
```

## Z-index system

Define a semantic z-index scale to avoid z-index wars:

```css
@theme {
  --z-base:      0;
  --z-raised:    10;    /* Cards on hover, sticky table headers */
  --z-dropdown:  100;   /* Dropdowns, popovers */
  --z-sticky:    200;   /* Sticky headers */
  --z-overlay:   300;   /* Modal overlays */
  --z-modal:     400;   /* Modal content */
  --z-toast:     500;   /* Toast notifications */
  --z-tooltip:   600;   /* Tooltips — always on top */
}
```

```tsx
// Tailwind custom z values
<div className="z-[100]">  // inline arbitrary value when needed
```

## Overflow handling

```tsx
// Scrollable sidebar with fixed header
<div className="flex h-dvh overflow-hidden">
  <aside className="w-64 overflow-y-auto">
    <nav className="p-4">...</nav>
  </aside>
  <main className="flex-1 overflow-y-auto">
    <div className="p-6">...</div>
  </main>
</div>

// Horizontal scroll for tables on mobile
<div className="overflow-x-auto -mx-4 px-4">
  <table className="min-w-[600px] w-full">...</table>
</div>

// Truncate text
<p className="truncate">Very long text that should be truncated</p>
<p className="line-clamp-3">Text truncated after 3 lines...</p>
```

## Aspect ratios

```tsx
// Preserve aspect ratio — responsive images/videos
<div className="aspect-video relative">           {/* 16:9 */}
  <Image src={src} alt={alt} fill className="object-cover" />
</div>

<div className="aspect-square">                   {/* 1:1 */}
<div className="aspect-[4/3]">                   {/* 4:3 */}
<div className="aspect-[3/4]">                   {/* Portrait */}

// Video embed
<div className="aspect-video">
  <iframe className="w-full h-full" src={embedUrl} title={title} allowFullScreen />
</div>
```
