# Animation & Transitions

## Motion principles

1. **Purposeful** — every animation communicates something (state change, hierarchy, direction)
2. **Fast** — 150–300ms for most UI transitions; 400–600ms for page transitions
3. **Natural** — ease-out for entering, ease-in for leaving, ease-in-out for both
4. **Interruptible** — never lock the user out while animating
5. **Reducible** — always respect `prefers-reduced-motion`

## CSS transitions (use for simple state changes)

```css
/* Base transition utilities */
.transition-colors {
  transition-property: color, background-color, border-color;
  transition-duration: 150ms;
  transition-timing-function: ease;
}

.transition-transform {
  transition-property: transform;
  transition-duration: 200ms;
  transition-timing-function: cubic-bezier(0.34, 1.56, 0.64, 1); /* spring */
}

/* Custom easing functions */
--ease-spring:   cubic-bezier(0.34, 1.56, 0.64, 1);  /* bouncy */
--ease-smooth:   cubic-bezier(0.4, 0, 0.2, 1);        /* material */
--ease-snappy:   cubic-bezier(0.25, 0.46, 0.45, 0.94);
```

```tsx
// Tailwind — transitions on interactive elements
<button className="
  bg-primary text-primary-foreground
  transition-colors duration-150
  hover:bg-primary/90
  active:scale-95 transition-transform
">
  Save
</button>

// Card hover lift
<div className="
  rounded-lg border bg-card p-6
  transition-all duration-200
  hover:-translate-y-1 hover:shadow-lg
">
```

## CSS @keyframes

```css
/* Fade in */
@keyframes fade-in {
  from { opacity: 0; }
  to   { opacity: 1; }
}

/* Slide in from bottom */
@keyframes slide-up {
  from { opacity: 0; transform: translateY(1rem); }
  to   { opacity: 1; transform: translateY(0); }
}

/* Scale in */
@keyframes scale-in {
  from { opacity: 0; transform: scale(0.95); }
  to   { opacity: 1; transform: scale(1); }
}

/* Skeleton shimmer */
@keyframes shimmer {
  from { background-position: -200% 0; }
  to   { background-position: 200% 0; }
}

.skeleton {
  background: linear-gradient(
    90deg,
    var(--muted) 25%,
    color-mix(in oklch, var(--muted) 70%, var(--background)) 50%,
    var(--muted) 75%
  );
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
}

/* Spin */
@keyframes spin {
  to { transform: rotate(360deg); }
}
```

```css
/* Register as Tailwind utilities */
@theme {
  --animate-fade-in:  fade-in 200ms ease-out both;
  --animate-slide-up: slide-up 300ms ease-out both;
  --animate-scale-in: scale-in 200ms ease-out both;
}
```

## View Transitions API

For page-to-page and in-page transitions without JavaScript animation libraries.

```tsx
// Trigger a view transition
'use client'
import { useRouter } from 'next/navigation'
import { useTransition } from 'react'

function TransitionLink({ href, children }: { href: string; children: React.ReactNode }) {
  const router = useRouter()
  const [isPending, startTransition] = useTransition()

  function handleClick(e: React.MouseEvent) {
    e.preventDefault()
    if (!document.startViewTransition) {
      router.push(href)
      return
    }
    document.startViewTransition(() => {
      startTransition(() => router.push(href))
    })
  }

  return (
    <a href={href} onClick={handleClick} aria-busy={isPending}>
      {children}
    </a>
  )
}
```

```css
/* Customize the transition */
@keyframes slide-in-from-right {
  from { transform: translateX(100%); opacity: 0; }
  to   { transform: translateX(0);    opacity: 1; }
}

@keyframes slide-out-to-left {
  from { transform: translateX(0);    opacity: 1; }
  to   { transform: translateX(-30%); opacity: 0; }
}

::view-transition-new(root) {
  animation: slide-in-from-right 300ms ease-out;
}

::view-transition-old(root) {
  animation: slide-out-to-left 200ms ease-in;
}

/* Shared element transition — product image grows into detail page */
.product-image {
  view-transition-name: var(--product-id);  /* unique per item */
}

::view-transition-group(product-image) {
  animation-duration: 300ms;
  animation-timing-function: cubic-bezier(0.34, 1.56, 0.64, 1);
}

/* Skip animation for reduced motion */
@media (prefers-reduced-motion: reduce) {
  ::view-transition-old(*), ::view-transition-new(*) {
    animation: none;
  }
}
```

## Scroll-driven animations

No JavaScript needed — pure CSS.

```css
/* Progress bar that tracks page scroll */
@keyframes fill-width {
  from { width: 0%; }
  to   { width: 100%; }
}

.scroll-progress {
  position: fixed;
  top: 0; left: 0;
  height: 3px;
  background: var(--color-brand-500);
  animation: fill-width linear both;
  animation-timeline: scroll(root block);
}

/* Reveal on scroll — element animates as it enters viewport */
@keyframes reveal {
  from { opacity: 0; transform: translateY(2rem); }
  to   { opacity: 1; transform: translateY(0); }
}

.reveal {
  animation: reveal linear both;
  animation-timeline: view();
  animation-range: entry 0% entry 40%;  /* start when 0%, complete at 40% in view */
}

/* Parallax effect */
@keyframes parallax-up {
  from { transform: translateY(0); }
  to   { transform: translateY(-20%); }
}

.parallax-bg {
  animation: parallax-up linear;
  animation-timeline: scroll(root);
  animation-range: 0% 100%;
}

/* Always disable for reduced motion */
@media (prefers-reduced-motion: reduce) {
  .reveal, .parallax-bg {
    animation: none;
    opacity: 1;
    transform: none;
  }
}
```

## Framer Motion

Use for complex orchestration: exit animations, layout animations, stagger, drag.

### Basic usage
```tsx
import { motion, AnimatePresence } from 'framer-motion'

// Simple enter animation
<motion.div
  initial={{ opacity: 0, y: 10 }}
  animate={{ opacity: 1, y: 0 }}
  transition={{ duration: 0.2, ease: 'easeOut' }}
>
  Content
</motion.div>
```

### Exit animations (AnimatePresence required)
```tsx
<AnimatePresence mode="wait">
  {isVisible && (
    <motion.div
      key="modal"
      initial={{ opacity: 0, scale: 0.95 }}
      animate={{ opacity: 1, scale: 1 }}
      exit={{ opacity: 0, scale: 0.95 }}
      transition={{ duration: 0.15 }}
    >
      Modal content
    </motion.div>
  )}
</AnimatePresence>

// Toast notifications with exit
<AnimatePresence>
  {toasts.map(toast => (
    <motion.li
      key={toast.id}
      layout
      initial={{ opacity: 0, x: 100 }}
      animate={{ opacity: 1, x: 0 }}
      exit={{ opacity: 0, x: 100, height: 0 }}
      transition={{ type: 'spring', stiffness: 500, damping: 40 }}
    >
      <Toast {...toast} />
    </motion.li>
  ))}
</AnimatePresence>
```

### Stagger children
```tsx
const container = {
  hidden: {},
  show: {
    transition: {
      staggerChildren: 0.05,
      delayChildren: 0.1,
    },
  },
}

const item = {
  hidden: { opacity: 0, y: 10 },
  show:   { opacity: 1, y: 0, transition: { ease: 'easeOut', duration: 0.2 } },
}

<motion.ul variants={container} initial="hidden" animate="show">
  {items.map(i => (
    <motion.li key={i.id} variants={item}>
      <Card {...i} />
    </motion.li>
  ))}
</motion.ul>
```

### Layout animations
```tsx
// Animate layout changes (reorder, size change) automatically
<motion.div layout className="flex flex-col gap-2">
  {items.map(item => (
    <motion.div key={item.id} layout layoutId={`item-${item.id}`}>
      {item.content}
    </motion.div>
  ))}
</motion.div>

// Shared element transition between list and detail
// List item
<motion.div layoutId={`card-${id}`} className="cursor-pointer" onClick={() => setSelected(id)}>
  <motion.img layoutId={`image-${id}`} src={image} />
  <motion.h3 layoutId={`title-${id}`}>{title}</motion.h3>
</motion.div>

// Detail overlay
<AnimatePresence>
  {selected && (
    <motion.div layoutId={`card-${selected}`} className="fixed inset-0 z-50 bg-background">
      <motion.img layoutId={`image-${selected}`} className="w-full h-64 object-cover" />
      <motion.h1 layoutId={`title-${selected}`} className="text-2xl font-bold p-6">{title}</motion.h1>
    </motion.div>
  )}
</AnimatePresence>
```

### Reduced motion in Framer Motion
```tsx
import { useReducedMotion } from 'framer-motion'

function AnimatedCard({ children }: { children: React.ReactNode }) {
  const prefersReduced = useReducedMotion()

  return (
    <motion.div
      initial={prefersReduced ? false : { opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      transition={prefersReduced ? { duration: 0 } : { duration: 0.3 }}
    >
      {children}
    </motion.div>
  )
}
```

## Loading states

### Skeleton screens (preferred over spinners)
```tsx
// Skeleton that matches the real content shape
function PostCardSkeleton() {
  return (
    <div className="rounded-lg border bg-card p-6 space-y-3" aria-hidden="true">
      <Skeleton className="h-4 w-3/4" />
      <Skeleton className="h-4 w-1/2" />
      <Skeleton className="h-20 w-full" />
      <div className="flex gap-2">
        <Skeleton className="h-8 w-20" />
        <Skeleton className="h-8 w-20" />
      </div>
    </div>
  )
}

// With loading.tsx in Next.js App Router
// app/posts/loading.tsx
export default function PostsLoading() {
  return (
    <div aria-label="Loading posts...">
      {Array.from({ length: 6 }).map((_, i) => (
        <PostCardSkeleton key={i} />
      ))}
    </div>
  )
}
```

### Inline loading states
```tsx
// Button loading
<button disabled={isPending} aria-busy={isPending}>
  {isPending ? (
    <>
      <Loader2 className="h-4 w-4 animate-spin" aria-hidden="true" />
      <span>Saving...</span>
    </>
  ) : 'Save'}
</button>

// Inline content loading
<div aria-live="polite" aria-busy={isLoading}>
  {isLoading ? <Spinner /> : <Content />}
</div>
```

## Micro-interactions

```tsx
// Checkbox check animation
<label className="flex items-center gap-3 cursor-pointer group">
  <div className="relative">
    <input type="checkbox" className="peer sr-only" />
    <div className="h-5 w-5 rounded border-2 border-input peer-checked:bg-primary peer-checked:border-primary transition-colors duration-150" />
    <Check className="absolute inset-0 m-auto h-3 w-3 text-primary-foreground opacity-0 peer-checked:opacity-100 transition-opacity duration-100 scale-50 peer-checked:scale-100 transition-transform" />
  </div>
  <span>Option label</span>
</label>

// Number counter animation
'use client'
import { useSpring, animated } from '@react-spring/web'

function AnimatedNumber({ value }: { value: number }) {
  const spring = useSpring({ value, from: { value: 0 }, config: { tension: 300, friction: 40 } })
  return <animated.span>{spring.value.to(v => Math.round(v).toLocaleString())}</animated.span>
}
```

## Performance

```css
/* Only animate transform and opacity — no layout triggers */

/* ✅ GPU-composited — fast */
.card:hover {
  transform: translateY(-4px);
  opacity: 0.9;
}

/* ❌ Triggers layout — slow */
.card:hover {
  top: -4px;     /* layout */
  width: 105%;   /* layout */
  margin-top: -4px; /* layout */
}

/* will-change — hint browser to promote to GPU layer */
/* Use sparingly — overuse increases memory */
.animated-element {
  will-change: transform;
}

/* Remove after animation completes */
element.addEventListener('animationend', () => {
  element.style.willChange = 'auto'
})
```

```tsx
// Defer non-critical animations
import { useIntersectionObserver } from '@/hooks/useIntersectionObserver'

function AnimatedSection({ children }: { children: React.ReactNode }) {
  const { ref, isIntersecting } = useIntersectionObserver({ threshold: 0.1, once: true })

  return (
    <motion.div
      ref={ref}
      initial={false}
      animate={isIntersecting ? 'visible' : 'hidden'}
      variants={{
        hidden: { opacity: 0, y: 30 },
        visible: { opacity: 1, y: 0, transition: { duration: 0.4 } },
      }}
    >
      {children}
    </motion.div>
  )
}
```
