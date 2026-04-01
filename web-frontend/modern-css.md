# Modern CSS — Tailwind v4, Container Queries, :has(), Subgrid, CSS Layers

## Tailwind CSS v4 — CSS-first configuration

### Setup
```css
/* app/globals.css */
@import "tailwindcss";

/* All configuration lives here — no tailwind.config.js needed */
@theme {
  /* Override or extend design tokens */
  --font-sans: 'Inter', sans-serif;
  --font-display: 'Playfair Display', serif;

  --color-brand-50: oklch(97% 0.01 250);
  --color-brand-500: oklch(55% 0.18 250);
  --color-brand-900: oklch(25% 0.08 250);

  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
  --radius-lg: 0.75rem;
  --radius-xl: 1rem;
}
```

### @variant — custom variants
```css
@import "tailwindcss";

/* Custom variant for a specific class */
@variant sidebar-open (&:is(.sidebar-open *));

/* Compound variants */
@variant hocus (&:hover, &:focus-visible);

/* Container-based variant */
@variant card-sm (@container (max-width: 400px));
```

```tsx
// Usage in JSX
<div className="hocus:bg-accent">Hover or focus</div>
<div className="sidebar-open:translate-x-0">Slides in when parent has .sidebar-open</div>
```

### Key changes from v3 to v4

| v3 | v4 |
|----|-----|
| `tailwind.config.js` | `@theme {}` in CSS |
| `theme.extend.colors` | `--color-*:` in `@theme` |
| `theme.extend.screens` | `--breakpoint-*:` in `@theme` |
| `@apply` only option | `@theme` + `@variant` + `@apply` |
| `content: ['./src/**/*.tsx']` | Auto-detection — no content config |
| `plugins: []` | `@variant` + `@utility` |

### Custom utilities
```css
@utility container-padded {
  width: 100%;
  max-width: var(--breakpoint-xl);
  margin-inline: auto;
  padding-inline: clamp(1rem, 4vw, 3rem);
}

@utility text-balance {
  text-wrap: balance;
}
```

## Container queries

Component-level responsiveness — adapts based on its container, not the viewport.

```css
/* Define a containment context */
.card-grid {
  container-type: inline-size;
  container-name: card-grid;
}

/* Query the container */
@container card-grid (min-width: 600px) {
  .card {
    display: grid;
    grid-template-columns: auto 1fr;
  }
}
```

```tsx
// Tailwind v4 container queries
<div className="@container">
  <div className="flex flex-col @md:flex-row gap-4">
    <img className="w-full @md:w-48" />
    <div className="flex-1">...</div>
  </div>
</div>

// Named containers
<div className="@container/sidebar">
  <nav className="hidden @sidebar/[200px]:block">...</nav>
</div>
```

**Use container queries when:**
- A component needs to adapt to its own available space, not the viewport
- The same component is used in different layout contexts (sidebar, main content, modal)
- You want truly reusable, portable components

## CSS :has() — parent selectors

```css
/* Style parent based on child state */
.form-field:has(input:invalid) label {
  color: var(--color-red-500);
}

/* Style parent when child is checked */
.card:has(input[type="checkbox"]:checked) {
  border-color: var(--color-brand-500);
  background-color: var(--color-brand-50);
}

/* Conditional layout — header with nav vs without */
header:has(nav) {
  grid-template-columns: auto 1fr auto;
}

header:not(:has(nav)) {
  grid-template-columns: auto auto;
}
```

```tsx
// Tailwind — has-* variant
<div className="has-[:checked]:bg-brand-50 has-[:checked]:border-brand-500 border rounded-md p-4">
  <input type="checkbox" id="agree" className="peer" />
  <label htmlFor="agree">I agree to the terms</label>
</div>
```

## CSS Subgrid

Align items across nested grid contexts.

```css
/* Parent grid */
.card-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1.5rem;
}

/* Cards inherit parent grid columns */
.card {
  display: grid;
  grid-column: span 1;
  grid-template-rows: subgrid;  /* key */
  grid-row: span 3;             /* 3 rows: image, title, body */
}

/* All card sections align across cards automatically */
.card-image  { grid-row: 1; }
.card-title  { grid-row: 2; }
.card-body   { grid-row: 3; }
```

```tsx
// Pattern: consistent card alignment with subgrid
<div className="grid grid-cols-3 gap-6 [&>*]:grid [&>*]:[grid-template-rows:subgrid] [&>*]:[grid-row:span_3]">
  {cards.map(card => (
    <article key={card.id}>
      <img src={card.image} alt="" />          {/* row 1 */}
      <h3>{card.title}</h3>                    {/* row 2 */}
      <p>{card.description}</p>               {/* row 3 */}
    </article>
  ))}
</div>
```

## CSS Layers

Control specificity without fighting it.

```css
@import "tailwindcss";

/* Declare layer order — lower = less specific */
@layer base, components, utilities;

@layer base {
  /* Low specificity resets */
  *, *::before, *::after { box-sizing: border-box; }
  body { font-family: var(--font-sans); }
}

@layer components {
  /* Component defaults — can be overridden by utilities */
  .btn {
    padding: 0.5rem 1rem;
    border-radius: var(--radius-md);
    font-weight: 500;
  }
  .btn-primary {
    background: var(--color-brand-500);
    color: white;
  }
}

@layer utilities {
  /* Highest specificity among layers */
  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
  }
}
```

**Tailwind's built-in layers:** `base` (preflight/resets) < `components` (component classes) < `utilities` (utility classes). Anything outside a layer has highest specificity — use for critical overrides.

## CSS Nesting

```css
/* Native CSS nesting (supported in all modern browsers) */
.card {
  border-radius: var(--radius-lg);
  padding: 1.5rem;

  & .card-title {
    font-size: 1.125rem;
    font-weight: 600;
  }

  &:hover {
    box-shadow: 0 4px 12px rgb(0 0 0 / 0.1);
  }

  @media (min-width: 768px) {
    display: grid;
    grid-template-columns: auto 1fr;
  }

  /* :is() for multiple selectors */
  &:is(:hover, :focus-within) {
    outline: 2px solid var(--color-brand-500);
  }
}
```

## CSS Anchor positioning

Position popovers/tooltips relative to a trigger without JavaScript.

```css
/* Define anchor */
.tooltip-trigger {
  anchor-name: --my-anchor;
}

/* Position relative to anchor */
.tooltip {
  position: absolute;
  position-anchor: --my-anchor;

  /* Position above the anchor */
  bottom: calc(anchor(top) + 8px);
  left: anchor(center);
  translate: -50% 0;
}

/* Fallback positions if tooltip overflows viewport */
position-try-fallbacks: flip-block, flip-inline, flip-block flip-inline;
```

## Scroll-driven animations

```css
/* Progress indicator that tracks scroll */
@keyframes grow-width {
  from { width: 0%; }
  to { width: 100%; }
}

.scroll-progress {
  position: fixed;
  top: 0;
  left: 0;
  height: 3px;
  background: var(--color-brand-500);
  animation: grow-width linear;
  animation-timeline: scroll(root block);
}

/* Reveal elements as they enter viewport */
@keyframes fade-up {
  from {
    opacity: 0;
    transform: translateY(1.5rem);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.reveal {
  animation: fade-up linear both;
  animation-timeline: view();
  animation-range: entry 0% entry 30%;
}

/* Respect reduced motion */
@media (prefers-reduced-motion: reduce) {
  .reveal {
    animation: none;
    opacity: 1;
    transform: none;
  }
}
```

## Color functions

### oklch — perceptually uniform colors
```css
@theme {
  /* oklch(lightness% chroma hue) */
  /* Consistent visual weight across hues */
  --color-blue-500:   oklch(55% 0.18 250);
  --color-green-500:  oklch(55% 0.18 145);
  --color-red-500:    oklch(55% 0.18 25);
  --color-yellow-500: oklch(55% 0.18 85);
}
```

### color-mix() — dynamic tints
```css
.surface-tinted {
  /* Mix brand color into white background */
  background: color-mix(in oklch, var(--color-brand-500) 8%, white);
}

/* Dynamic tint based on custom property */
.badge {
  --badge-color: var(--color-brand-500);
  background: color-mix(in oklch, var(--badge-color) 15%, white);
  color: color-mix(in oklch, var(--badge-color) 85%, black);
  border-color: color-mix(in oklch, var(--badge-color) 30%, white);
}
```

### Relative color syntax
```css
/* Adjust lightness of any color */
.surface-hover {
  background: oklch(from var(--color-brand-500) calc(l + 10%) c h);
}

/* Create a transparent version */
.overlay {
  background: oklch(from var(--color-brand-500) l c h / 20%);
}
```

## Logical properties — RTL-ready by default

```css
/* ❌ Physical — breaks RTL */
.card { padding-left: 1rem; margin-right: auto; }

/* ✅ Logical — RTL-aware */
.card { padding-inline-start: 1rem; margin-inline-end: auto; }
```

| Physical | Logical |
|---------|---------|
| `padding-left` | `padding-inline-start` |
| `margin-right` | `margin-inline-end` |
| `border-top` | `border-block-start` |
| `left` | `inset-inline-start` |
| `width` | `inline-size` |
| `height` | `block-size` |

```tsx
// Tailwind uses logical properties for padding/margin/border
// ps- = padding-inline-start, pe- = padding-inline-end
// ms- = margin-inline-start, me- = margin-inline-end
<div className="ps-4 me-auto border-s-2 border-primary" />
```

## View transitions

```css
/* Named view transitions for shared element animation */
.product-card {
  view-transition-name: var(--product-id);  /* unique per card */
}

/* Customize transition */
::view-transition-old(product-card) {
  animation: fade-out 200ms ease;
}
::view-transition-new(product-card) {
  animation: fade-in 200ms ease;
}
```

```tsx
// Next.js — trigger view transition on navigation
'use client'
import { useRouter } from 'next/navigation'

function ProductLink({ product }: { product: Product }) {
  const router = useRouter()

  function navigate() {
    if (!document.startViewTransition) {
      router.push(`/products/${product.id}`)
      return
    }
    document.startViewTransition(() => {
      router.push(`/products/${product.id}`)
    })
  }

  return (
    <button
      onClick={navigate}
      style={{ viewTransitionName: `product-${product.id}` }}
    >
      <img src={product.image} alt={product.name} />
    </button>
  )
}
```

## Modern selectors cheatsheet

```css
/* :where() — zero specificity, safe for base styles */
:where(h1, h2, h3, h4, h5, h6) {
  line-height: 1.2;
  text-wrap: balance;
}

/* :is() — groups selectors, takes highest specificity of list */
:is(article, section) h2 { font-size: 1.5rem; }

/* :not() with complex selectors */
a:not([href^="http"]):not([href^="mailto"]) {
  /* Internal links only */
}

/* :focus-visible — keyboard focus only, not mouse */
button:focus-visible {
  outline: 2px solid var(--color-brand-500);
  outline-offset: 2px;
}

/* :has() for interactive states */
.input-group:has(input:focus) label { color: var(--color-brand-500); }
.input-group:has(input:invalid:not(:placeholder-shown)) label { color: var(--color-red-500); }
```
