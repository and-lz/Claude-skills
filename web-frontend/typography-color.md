# Typography & Color

## Font loading with next/font

Always use `next/font` — eliminates layout shift, self-hosts Google Fonts automatically.

```tsx
// app/layout.tsx
import { Inter, Playfair_Display } from 'next/font/google'
import localFont from 'next/font/local'

const inter = Inter({
  subsets: ['latin'],
  variable: '--font-sans',
  display: 'swap',
})

const playfair = Playfair_Display({
  subsets: ['latin'],
  variable: '--font-display',
  weight: ['400', '600', '700'],
  style: ['normal', 'italic'],
})

// Local / custom font
const geist = localFont({
  src: [
    { path: '../fonts/GeistVF.woff2', weight: '100 900' },  // variable
  ],
  variable: '--font-sans',
  display: 'swap',
})

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${inter.variable} ${playfair.variable}`}>
      <body className="font-sans antialiased">{children}</body>
    </html>
  )
}
```

```css
/* globals.css — use CSS variables */
@theme {
  --font-sans: var(--font-sans-loaded, ui-sans-serif, system-ui, sans-serif);
  --font-display: var(--font-display-loaded, Georgia, serif);
  --font-mono: ui-monospace, 'Cascadia Code', 'Source Code Pro', monospace;
}
```

## Type scale

A modular scale with consistent ratios. Define once, use everywhere.

### Tailwind defaults (use these)

| Class | Size | Line height | Use for |
|-------|------|-------------|---------|
| `text-xs` | 12px | 16px | Captions, labels |
| `text-sm` | 14px | 20px | Secondary text, metadata |
| `text-base` | 16px | 24px | Body text (default) |
| `text-lg` | 18px | 28px | Lead text, subheadings |
| `text-xl` | 20px | 28px | Card titles, section heads |
| `text-2xl` | 24px | 32px | Page subheadings |
| `text-3xl` | 30px | 36px | Section headings |
| `text-4xl` | 36px | 40px | Page headings |
| `text-5xl` | 48px | 1 | Hero subheadings |
| `text-6xl` | 60px | 1 | Hero headings |
| `text-7xl` | 72px | 1 | Display / landing hero |

### Fluid typography with clamp()

```css
/* Scales smoothly between viewport sizes — no breakpoint jumps */
h1 {
  font-size: clamp(2rem, 5vw + 1rem, 4.5rem);
  line-height: 1.1;
}

h2 {
  font-size: clamp(1.5rem, 3vw + 0.5rem, 2.5rem);
  line-height: 1.2;
}

p {
  font-size: clamp(1rem, 1vw + 0.75rem, 1.125rem);
}
```

## Readability rules

- **Line length**: 45–75 characters for body text. Use `max-w-prose` (65ch) or `max-w-xl`.
- **Line height**: 1.5–1.7 for body, 1.1–1.3 for headings.
- **Letter spacing**: Tight for headings (`tracking-tight`), normal for body.
- **Text wrap**: `text-balance` for headings (even line lengths), `text-pretty` for paragraphs (avoid orphans).

```tsx
<h1 className="text-4xl font-bold tracking-tight text-balance">
  Build faster, ship better
</h1>

<p className="text-base text-muted-foreground leading-relaxed text-pretty max-w-prose">
  Deploy and scale your applications with confidence. Built for teams who move fast.
</p>
```

## Design tokens

### CSS custom properties (single source of truth)

```css
@theme {
  /* Colors */
  --color-background: oklch(100% 0 0);
  --color-foreground: oklch(9% 0.02 264);
  --color-muted: oklch(96% 0.01 264);
  --color-muted-foreground: oklch(45% 0.02 264);
  --color-border: oklch(90% 0.01 264);
  --color-input: oklch(90% 0.01 264);
  --color-primary: oklch(20% 0.02 264);
  --color-primary-foreground: oklch(98% 0 0);
  --color-secondary: oklch(96% 0.01 264);
  --color-secondary-foreground: oklch(20% 0.02 264);
  --color-accent: oklch(96% 0.01 264);
  --color-accent-foreground: oklch(20% 0.02 264);
  --color-destructive: oklch(55% 0.22 27);
  --color-destructive-foreground: oklch(98% 0 0);
  --color-card: oklch(100% 0 0);
  --color-card-foreground: oklch(9% 0.02 264);
  --color-ring: oklch(20% 0.02 264);

  /* Brand */
  --color-brand-50:  oklch(97% 0.01 250);
  --color-brand-100: oklch(93% 0.03 250);
  --color-brand-200: oklch(86% 0.06 250);
  --color-brand-300: oklch(77% 0.10 250);
  --color-brand-400: oklch(66% 0.15 250);
  --color-brand-500: oklch(55% 0.18 250);  /* primary */
  --color-brand-600: oklch(47% 0.17 250);
  --color-brand-700: oklch(39% 0.14 250);
  --color-brand-800: oklch(31% 0.10 250);
  --color-brand-900: oklch(23% 0.06 250);
}
```

### Dark mode token switching

```css
/* Light mode (default) */
:root {
  --background: oklch(100% 0 0);
  --foreground: oklch(9% 0.02 264);
  --muted: oklch(96% 0.01 264);
  --muted-foreground: oklch(45% 0.02 264);
  --border: oklch(90% 0.01 264);
  --primary: oklch(20% 0.02 264);
  --primary-foreground: oklch(98% 0 0);
}

/* Dark mode — swap token values */
.dark {
  --background: oklch(9% 0.02 264);
  --foreground: oklch(98% 0 0);
  --muted: oklch(15% 0.02 264);
  --muted-foreground: oklch(64% 0.02 264);
  --border: oklch(20% 0.02 264);
  --primary: oklch(98% 0 0);
  --primary-foreground: oklch(9% 0.02 264);
}

/* System preference — no JS required */
@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --background: oklch(9% 0.02 264);
    /* ... */
  }
}
```

## The 80/20 color rule

**80% neutral, 20% accent.**

```
Neutral (80%):
  - Page backgrounds
  - Card surfaces
  - Text (primary, secondary, muted)
  - Borders, dividers
  - Icons (non-interactive)
  - Input backgrounds

Accent (20%):
  - Primary action buttons
  - Links
  - Focus rings
  - Active navigation items
  - Key indicators (badges, progress bars)
  - Highlighted selections
```

```tsx
// ❌ Over-accented — accent everywhere loses impact
<div className="bg-brand-500 text-white">
  <h1 className="text-brand-200">Dashboard</h1>
  <nav className="border-brand-400">
    <a className="text-brand-100">Home</a>

// ✅ Correct — neutral base, accent for actions only
<div className="bg-background text-foreground">
  <h1 className="text-foreground">Dashboard</h1>
  <nav className="border-border">
    <a className="text-muted-foreground hover:text-foreground">Home</a>
  </nav>
  <button className="bg-primary text-primary-foreground">Save</button>
```

## oklch — why and how

oklch (Oklab chroma/hue) gives perceptually uniform colors — equal chroma and lightness values look visually consistent across hues.

```
oklch(lightness% chroma hue)
  lightness: 0% (black) to 100% (white)
  chroma:    0 (gray) to ~0.37 (most saturated)
  hue:       0–360 degrees
```

```css
/* Create a palette with consistent visual weight */
--red:    oklch(55% 0.2 25);
--orange: oklch(55% 0.2 55);
--yellow: oklch(55% 0.2 85);
--green:  oklch(55% 0.2 145);
--blue:   oklch(55% 0.2 250);
--purple: oklch(55% 0.2 300);

/* All have the same perceived lightness and saturation */
```

### color-mix() for tints
```css
/* Generate tints/shades without hand-picking values */
.badge-blue { 
  background: color-mix(in oklch, var(--color-brand-500) 15%, var(--background));
  color: color-mix(in oklch, var(--color-brand-500) 90%, var(--foreground));
}
```

## Dark mode implementation

### Option 1: System preference only (simplest)
```css
/* CSS only — no JavaScript */
@media (prefers-color-scheme: dark) {
  :root { /* dark token values */ }
}
```

### Option 2: Manual toggle + system fallback
```tsx
// Theme provider (client component)
'use client'
import { createContext, useContext, useEffect, useState } from 'react'

type Theme = 'light' | 'dark' | 'system'

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useLocalStorage<Theme>('theme', 'system')

  useEffect(() => {
    const root = document.documentElement
    const systemDark = window.matchMedia('(prefers-color-scheme: dark)').matches
    const isDark = theme === 'dark' || (theme === 'system' && systemDark)
    root.classList.toggle('dark', isDark)
    root.setAttribute('data-theme', isDark ? 'dark' : 'light')
  }, [theme])

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  )
}
```

```tsx
// Theme toggle button
function ThemeToggle() {
  const { theme, setTheme } = useTheme()
  return (
    <button
      onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}
      aria-label={`Switch to ${theme === 'dark' ? 'light' : 'dark'} mode`}
    >
      {theme === 'dark' ? <Sun className="h-4 w-4" /> : <Moon className="h-4 w-4" />}
    </button>
  )
}
```

### Prevent flash of wrong theme (FOUC)

```tsx
// app/layout.tsx — inline script to apply theme before paint
<head>
  <script dangerouslySetInnerHTML={{
    __html: `
      (function() {
        var theme = localStorage.getItem('theme') || 'system';
        var dark = theme === 'dark' || (theme === 'system' && window.matchMedia('(prefers-color-scheme: dark)').matches);
        document.documentElement.classList.toggle('dark', dark);
      })();
    `
  }} />
</head>
```

## Semantic color usage

```tsx
// Use semantic tokens — they adapt to dark mode automatically
<p className="text-foreground">Primary text</p>
<p className="text-muted-foreground">Secondary / helper text</p>
<span className="text-destructive">Error message</span>

<div className="bg-background">Page background</div>
<div className="bg-card border border-border rounded-lg">Card surface</div>
<div className="bg-muted">Subtle background (code blocks, table headers)</div>

<button className="bg-primary text-primary-foreground">Primary action</button>
<button className="bg-secondary text-secondary-foreground">Secondary</button>
<button className="bg-destructive text-destructive-foreground">Delete</button>

// Focus ring
<input className="focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2" />
```

## Contrast requirements

| Text type | Min ratio | Notes |
|-----------|-----------|-------|
| Normal text (< 18pt / 14pt bold) | 4.5:1 | WCAG AA |
| Large text (≥ 18pt / 14pt bold) | 3:1 | WCAG AA |
| UI components, icons | 3:1 | Borders, icons |
| Decorative | None | Purely visual |

**Tools:**
- `oklch` color values make it easier to maintain consistent contrast across hues
- Use browser DevTools accessibility panel to check contrast
- `axe-core` in tests catches contrast issues programmatically

## Font weight guidelines

```tsx
// Weights and their purposes
<span className="font-normal">400 — body text, descriptions</span>
<span className="font-medium">500 — labels, UI text, navigation</span>
<span className="font-semibold">600 — subheadings, card titles, emphasis</span>
<span className="font-bold">700 — headings, important numbers</span>

// Avoid very light weights (300) in body text — poor contrast on non-retina displays
```
