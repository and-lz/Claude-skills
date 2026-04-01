# Accessibility & Internationalization

## WCAG 2.2 AA — key requirements

| Criterion | Requirement | How |
|-----------|-------------|-----|
| 1.1.1 | Alt text for images | `alt` on `<img>`, `aria-label` on SVG icons |
| 1.3.1 | Info via structure | Semantic HTML, proper heading hierarchy |
| 1.4.3 | Contrast 4.5:1 | Check with DevTools / axe |
| 1.4.4 | Resize to 200% | No fixed heights on text containers |
| 1.4.11 | UI contrast 3:1 | Borders, icons, focus rings |
| 2.1.1 | Keyboard accessible | All interactions work without mouse |
| 2.4.3 | Focus order | Logical tab order matches visual order |
| 2.4.7 | Focus visible | `:focus-visible` outlines on all interactive elements |
| 4.1.2 | Name/role/value | Interactive elements have name, role, state |
| 4.1.3 | Status messages | `aria-live` for dynamic content updates |

## Semantic HTML

```tsx
// Page structure — use these elements
<header>        // Site header
<nav aria-label="Main navigation">  // Navigation — aria-label if multiple
<main id="main-content">            // Main content — one per page
<section>       // Thematic group
<article>       // Self-contained content
<aside>         // Complementary / sidebar
<footer>

// Interactive elements
// <button> for actions, <a href> for navigation — never swap
// Never put <button> inside <a> or vice versa
// Never use <div onClick> — use <button>
```

## ARIA — use minimally

The first rule of ARIA: don't use ARIA if native HTML works.

```tsx
// Live regions for dynamic content
<div aria-live="polite">{statusMessage}</div>    // non-urgent
<div aria-live="assertive">{errorMessage}</div>  // critical only

// Icon buttons
<button aria-label="Close dialog">
  <X className="h-4 w-4" aria-hidden="true" />
</button>

// Expandable controls
<button aria-expanded={isOpen} aria-controls="menu-id">Options</button>
<ul id="menu-id" hidden={!isOpen}>...</ul>

// Loading
<div aria-busy={isLoading} aria-label="Loading users">
  {isLoading ? <Spinner /> : <UserList />}
</div>

// Modal dialog
<div role="dialog" aria-modal="true" aria-labelledby="title-id">
  <h2 id="title-id">Confirm deletion</h2>
</div>
```

## Keyboard navigation

```tsx
// Skip link — always first element
<a href="#main-content" className="sr-only focus:not-sr-only focus:fixed focus:top-4 focus:left-4 focus:z-50 focus:px-4 focus:py-2 focus:bg-background focus:rounded-md focus:shadow-lg">
  Skip to main content
</a>

// Focus trap for modals
function useFocusTrap(isActive: boolean) {
  const ref = useRef<HTMLDivElement>(null)

  useEffect(() => {
    if (!isActive || !ref.current) return
    const focusable = ref.current.querySelectorAll<HTMLElement>(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    )
    focusable[0]?.focus()

    function onKeyDown(e: KeyboardEvent) {
      if (e.key !== 'Tab') return
      const first = focusable[0]
      const last = focusable[focusable.length - 1]
      if (e.shiftKey ? document.activeElement === first : document.activeElement === last) {
        e.preventDefault()
        ;(e.shiftKey ? last : first).focus()
      }
    }
    document.addEventListener('keydown', onKeyDown)
    return () => document.removeEventListener('keydown', onKeyDown)
  }, [isActive])

  return ref
}

// Return focus to trigger after modal closes
const triggerRef = useRef<HTMLButtonElement>(null)
useEffect(() => {
  if (!isOpen) triggerRef.current?.focus()
}, [isOpen])
```

## Focus styles

```css
/* :focus-visible — keyboard only, not mouse */
:focus-visible {
  outline: 2px solid var(--color-ring);
  outline-offset: 2px;
  border-radius: 4px;
}
```

```tsx
// Tailwind
<button className="focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2">
```

## Forms accessibility

```tsx
<div className="space-y-2">
  <label htmlFor="email" className="text-sm font-medium">
    Email
    <span aria-hidden="true" className="text-destructive ml-1">*</span>
    <span className="sr-only">(required)</span>
  </label>
  <input
    id="email"
    type="email"
    required
    aria-required="true"
    aria-invalid={!!errors.email}
    aria-describedby={errors.email ? 'email-error' : 'email-hint'}
    className="w-full rounded-md border px-3 py-2 focus-visible:ring-2 focus-visible:ring-ring"
  />
  <p id="email-hint" className="text-xs text-muted-foreground">We'll never share your email.</p>
  {errors.email && (
    <p id="email-error" role="alert" className="text-xs text-destructive">{errors.email}</p>
  )}
</div>

// Group related inputs with fieldset
<fieldset>
  <legend className="text-sm font-medium mb-2">Notifications</legend>
  <label className="flex items-center gap-2">
    <input type="checkbox" name="email" />
    Email updates
  </label>
</fieldset>
```

## Reduced motion and preferences

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}

@media (prefers-contrast: more) {
  :root { --border: oklch(0% 0 0); }
}

@media (forced-colors: active) {
  .custom-element { forced-color-adjust: none; }
}
```

```tsx
// Tailwind
<div className="motion-safe:animate-bounce motion-reduce:animate-none" />
```

## Accessibility testing

```tsx
// jest-axe
import { axe, toHaveNoViolations } from 'jest-axe'
expect.extend(toHaveNoViolations)

test('LoginForm has no a11y violations', async () => {
  const { container } = render(<LoginForm />)
  expect(await axe(container)).toHaveNoViolations()
})

// Playwright
import { checkA11y } from 'axe-playwright'
test('home page is accessible', async ({ page }) => {
  await page.goto('/')
  await checkA11y(page)
})
```

### Manual checklist
- [ ] Tab through entire page — every element reachable?
- [ ] Shift+Tab — focus moves backwards?
- [ ] Enter/Space on buttons activate them?
- [ ] Escape closes modals and dropdowns?
- [ ] Screen reader: VoiceOver (Mac) or NVDA (Windows)
- [ ] Zoom 200% — content reflows without horizontal scroll?

## Internationalization with next-intl

```tsx
// i18n/routing.ts
import { defineRouting } from 'next-intl/routing'
export const routing = defineRouting({
  locales: ['en', 'pt', 'es'],
  defaultLocale: 'en',
})

// middleware.ts
import createMiddleware from 'next-intl/middleware'
import { routing } from './i18n/routing'
export default createMiddleware(routing)
export const config = { matcher: ['/((?!api|_next|.*\\..*).*)'] }

// messages/en.json
{
  "nav": { "home": "Home", "about": "About" },
  "users": {
    "count": "{count, plural, =0 {No users} one {# user} other {# users}}"
  }
}
```

```tsx
// Client Component
import { useTranslations, useFormatter } from 'next-intl'

function Nav() {
  const t = useTranslations('nav')
  return <a href="/">{t('home')}</a>
}

function UserCount({ count }: { count: number }) {
  const t = useTranslations('users')
  return <span>{t('count', { count })}</span>
}

function Price({ amount }: { amount: number }) {
  const format = useFormatter()
  return <span>{format.number(amount, { style: 'currency', currency: 'USD' })}</span>
}

// Server Component
import { getTranslations } from 'next-intl/server'
async function Page() {
  const t = await getTranslations('nav')
  return <h1>{t('home')}</h1>
}
```

## RTL support

```tsx
// app/layout.tsx
import { getLocale } from 'next-intl/server'
export default async function Layout({ children }) {
  const locale = await getLocale()
  const dir = ['ar', 'he', 'fa', 'ur'].includes(locale) ? 'rtl' : 'ltr'
  return <html lang={locale} dir={dir}><body>{children}</body></html>
}
```

```css
/* Use logical properties — automatically RTL-aware */
.card {
  padding-inline: 1.5rem;          /* left+right in LTR, right+left in RTL */
  border-inline-start: 4px solid;  /* left in LTR, right in RTL */
  margin-inline-end: auto;
}
```

```tsx
// Tailwind — use start/end variants
<div className="ps-4 pe-6 ms-auto border-s-4">   // ✅ RTL-safe
<div className="pl-4 pr-6 ml-auto border-l-4">   // ❌ breaks RTL
```
