---
name: web-frontend
description: ALWAYS use for ANY web frontend, React, Next.js, Tailwind CSS, or modern CSS request — non-negotiable. Triggers on React, Next.js, App Router, RSC, Server Components, Server Actions, use(), useOptimistic, useFormStatus, useActionState, Suspense, Tailwind CSS, @theme, @variant, container queries, :has(), subgrid, view transitions, scroll-driven animations, color-mix(), oklch, CSS layers, CSS nesting, anchor positioning, TypeScript, Core Web Vitals, LCP, CLS, INP, Lighthouse, bundle optimization, image optimization, next/image, next/font, Vitest, Playwright, Testing Library, WCAG, ARIA, semantic HTML, keyboard navigation, screen reader, React 19, Next.js 15, Tailwind v4, middleware, ISR, streaming, RSC payload, Zustand, React Query, TanStack, Zod, React Hook Form, Radix, shadcn/ui, Framer Motion, CSS modules, PostCSS, responsive design, mobile-first, dark mode, color scheme, design tokens, i18n, next-intl, route groups, parallel routes, intercepting routes, generateMetadata, opengraph-image, sitemap, robots, edge runtime, turbopack, Vercel, Netlify, Cloudflare Pages, SSR, SSG, ISR, PPR, partial prerendering. Also triggers on "my app/page/component", "build/design a page", navigation, sidebar, modal, toast, data table, form, dashboard, landing page, auth flow, checkout, blog, portfolio, e-commerce, SaaS, dark mode toggle, theme switcher, infinite scroll, virtualized list, drag and drop, command palette, breadcrumbs, tabs, accordion, carousel, skeleton loading, error boundary, loading.tsx, error.tsx, not-found.tsx, layout.tsx, page.tsx, .tsx files, .css files, tailwind.config, next.config, postcss.config, tsconfig. If it touches web frontend USE THIS SKILL. Covers components, layout, typography, color, animation, accessibility, i18n, engineering, architecture, state management, data fetching, caching, Next.js routing, RSC, Server Actions, middleware, performance, Core Web Vitals, forms, validation, testing, auth, security, and modern CSS. Enforces requirements-gathering via structured questions before any code generation.
---

# Web Frontend Design & Engineering Skill

Design, architect, and build modern web applications using Next.js 15+, React 19+, Tailwind CSS v4+, and TypeScript that follow web standards, deliver excellent Core Web Vitals, and are accessible, performant, and production-ready.

Before writing any frontend code, read the relevant reference files listed below. Multiple references will often apply to a single page.

## Reference files — read before building

| File | Read when |
|------|-----------|
| `components-ui-patterns.md` | Building any UI component — buttons, cards, modals, data tables, navigation, sidebars, command palettes, toasts, dropdowns, tabs, accordions, and 60+ common page patterns |
| `layout-responsive.md` | Laying out any page — breakpoints, container queries, grid, flexbox, mobile-first, sidebar layouts, dashboard grids, spacing system, z-index layers |
| `typography-color.md` | Setting fonts, type scale, next/font, design tokens, color systems, oklch, dark mode, theme switching, the 80/20 accent rule |
| `animation-transitions.md` | Adding motion, view transitions, scroll-driven animations, Framer Motion, CSS transitions, loading states, skeleton screens, micro-interactions |
| `accessibility-i18n.md` | Accessibility requirements, WCAG 2.2 AA, ARIA patterns, semantic HTML, keyboard navigation, screen readers, focus management, i18n, next-intl, RTL |
| `engineering.md` | App architecture, state management (Zustand, React Query, use()), TypeScript patterns, React 19 features, data fetching, caching, error handling, project structure, code splitting |
| `nextjs.md` | Routing, RSC, Server Actions, middleware, ISR, streaming, PPR, metadata, OG images, edge runtime, deployment, next.config |
| `performance.md` | Core Web Vitals, bundle optimization, code splitting, image optimization, font optimization, lazy loading, Lighthouse, caching strategies |
| `forms-validation.md` | Form patterns, React Hook Form, Zod validation, Server Actions for forms, useActionState, useFormStatus, useOptimistic, file uploads, multi-step forms |
| `testing.md` | Vitest unit tests, Playwright E2E, Testing Library component tests, MSW mocking, accessibility testing, visual regression, CI integration |
| `auth-security.md` | Authentication patterns, Auth.js, middleware protection, CSRF, CSP, XSS prevention, CORS, rate limiting, environment variables, secrets |
| `modern-css.md` | CSS-first Tailwind v4, @theme, @variant, container queries, :has(), subgrid, CSS layers, nesting, anchor positioning, scroll-driven animations, color-mix(), oklch |

## Core design principles

These principles are the foundation of every good web application:

### 1. Content first
Content drives the interface. Typography is readable, whitespace is purposeful, decoration is minimal. UI recedes so users focus on what matters. Structure content with semantic HTML before adding any visual treatment.

### 2. Progressive enhancement
Build on a foundation of semantic HTML. Layer CSS for presentation and JS for interaction. The core experience works without client-side JavaScript where possible. Use Server Components by default — they ship zero JS to the browser.

### 3. Performance by default
Every page meets Core Web Vitals thresholds (LCP < 2.5s, CLS < 0.1, INP < 200ms). Ship less JavaScript. Prefer server rendering. Optimize images, fonts, and bundles. Performance is a feature, not an afterthought.

### 4. Responsive and adaptive
Design mobile-first, then enhance for larger viewports. Use container queries for component-level responsiveness. Support landscape, portrait, touch, mouse, and keyboard. Every layout works from 320px to 2560px.

### 5. Accessible to all
WCAG 2.2 AA compliance is baseline, not a stretch goal. Semantic HTML, keyboard navigation, screen reader support, color contrast, focus management, and reduced motion preferences are required for every component.

### 6. Type safety everywhere
TypeScript strict mode from the start. Validate at boundaries with Zod (API responses, form data, URL params). Types are documentation. `any` is banned. Infer types from schemas — don't duplicate definitions.

## RULE #1: Never assume — always ask first

**Before writing a single line of code or making any design decision, gather requirements from the user.** Do not assume page layout, color scheme, content structure, data sources, rendering strategy, or any design/engineering choice. What feels "obvious" often isn't — a "dashboard" could be a simple stats overview, a complex analytics platform, or an admin panel depending on the app.

Use the `AskUserQuestion` tool to present structured questions. Collect all needed context in as few turns as possible (batch 1–4 questions per call, each with 2–4 focused options). Only proceed to implementation after the user has confirmed the key decisions.

### What to ask — question bank by category

Pick the questions relevant to the request. Scale questioning to scope.

**For new pages or features:**
- What type of page is this? (landing, dashboard, data table, form, detail/profile, settings, auth, blog/content, e-commerce, empty state, error page)
- What's the primary user goal? (browse, create, edit, consume, configure, search, purchase)
- What navigation context? (app shell with sidebar, top nav bar, standalone page, modal flow, tab-based sections)
- What data drives this page? (REST API, GraphQL, database via Server Actions, static/CMS, real-time WebSocket, user input)
- What rendering strategy? (SSR default, SSG for static, ISR for semi-static, client-side for highly interactive)

**For design decisions:**
- What's the visual tone? (minimal/clean, content-dense, data-heavy, marketing/editorial, playful, professional)
- Does the app have an existing design system or component library? (shadcn/ui, custom, none)
- Is dark mode required? (system preference only, manual toggle, both, light only)
- What brand colors are established? If none, what personality?
- Responsive priority? (mobile-first default, desktop-first, equal)

**For engineering decisions:**
- What's the data model? (entities, relationships, persistence)
- Is there an existing API? If so, what does the response shape look like?
- What state needs to be shared across pages vs local to this component?
- Caching strategy? (static generation, time-based revalidation, on-demand, no cache)
- Authentication required? What provider? (Auth.js, custom JWT, third-party)
- Real-time updates? (polling, SSE, WebSocket, none)

**For performance decisions:**
- Target audience? (global vs regional, mobile-dominant vs desktop, low-bandwidth concerns)
- Image-heavy? (gallery, product images, user uploads, icons only)
- SEO critical? (marketing/blog = yes, authenticated app = less so)
- Third-party scripts? (analytics, chat, payment forms)

**For iteration on existing code:**
- What specifically needs to change? (visual polish, new feature, bug fix, accessibility, performance)
- Architecture changes acceptable?

### How to ask effectively

Use the `AskUserQuestion` tool — it renders an interactive UI with selectable options. Set `multiSelect: true` when choices aren't mutually exclusive. Each question needs a short `header` (≤12 chars). Batch up to 4 questions per call. Example:

```
AskUserQuestion({
  questions: [
    {
      header: "Page type",
      question: "What kind of page is this?",
      multiSelect: false,
      options: [
        { label: "Dashboard", description: "Overview with stats, charts, and activity" },
        { label: "Data table", description: "Sortable/filterable rows of records" },
        { label: "Form", description: "Input fields for creating or editing" },
        { label: "Landing", description: "Marketing page with hero, features, CTA" }
      ]
    },
    {
      header: "Data source",
      question: "Where does the data come from?",
      multiSelect: false,
      options: [
        { label: "REST API", description: "Fetch from external or internal API" },
        { label: "Server Action", description: "Direct database query server-side" },
        { label: "Static/CMS", description: "Hardcoded or CMS-sourced content" },
        { label: "Real-time", description: "WebSocket or SSE for live updates" }
      ]
    },
    {
      header: "Rendering",
      question: "What rendering strategy?",
      multiSelect: false,
      options: [
        { label: "SSR", description: "Server-rendered on each request (default)" },
        { label: "SSG", description: "Static at build time" },
        { label: "ISR", description: "Static with timed revalidation" },
        { label: "Client", description: "Fully client-rendered (SPA-like)" }
      ]
    }
  ]
})
```

After receiving answers, summarize briefly: "Got it — I'll build a [type] page using [data source], rendered via [strategy]. Here's the approach..."

### When you CAN skip detailed questioning
- **Trivial changes**: "make the button blue", "add padding to this card", "fix the aria label" — just do it.
- **Explicit specs provided**: Detailed spec or Figma reference with clear requirements — don't re-ask.
- **Follow-up iterations**: Active back-and-forth with established context.
- **Code review / fix requests**: Diagnose first, ask only if ambiguous.

## Decision framework: building a web frontend page

After gathering requirements, follow this sequence for every page:

### Step 1 — Identify the page type
Landing/marketing, dashboard/overview, data table, form/input, detail/profile, settings, auth (login/register), content/blog, e-commerce (product grid, cart, checkout), empty state, or error page. This determines component selection and layout.

### Step 2 — Choose the rendering strategy
- **Server Component (default)** → zero client JS, direct data access, streaming with Suspense
- **Client Component** → interactivity, browser APIs, React state/effects (`'use client'` only when needed)
- **SSG** → static content that rarely changes (`generateStaticParams`)
- **ISR** → static with timed revalidation (`revalidate: 3600`)
- **Streaming** → progressive loading with `<Suspense>` boundaries
- **PPR** → static shell with dynamic holes (Partial Prerendering)
- Read `nextjs.md` for full patterns.

### Step 3 — Choose the navigation structure
- **App shell with sidebar** → dashboard apps
- **Top navigation bar** → marketing, content, SaaS
- **Breadcrumb hierarchy** → deep content (docs, e-commerce categories)
- **Tab-based sections** → settings, profile, multi-section detail views
- **Modal flows** → wizards, confirmations, quick actions
- Read `components-ui-patterns.md`.

### Step 4 — Apply the 80/20 color rule
80% neutral/system colors (backgrounds, text, borders). 20% accent color for primary actions, links, focus rings, active states only. Read `typography-color.md`.

### Step 5 — Set up typography
Use `next/font` for zero-layout-shift font loading. Type scale via CSS custom properties or Tailwind. Readable line lengths (45–75 characters). Never hardcode pixel sizes. Read `typography-color.md`.

### Step 6 — Design for every context
Light mode, dark mode, `prefers-reduced-motion`, `prefers-contrast`. Mobile (320px), tablet (768px), desktop (1280px), ultrawide (2560px). Touch and mouse input.

### Step 7 — Verify accessibility
Every interactive element has an accessible name. Color contrast meets WCAG AA (4.5:1 normal, 3:1 large text/UI). Focus order is logical. Keyboard navigation works. ARIA is minimal and correct — prefer semantic HTML. Read `accessibility-i18n.md`.

### Step 8 — Implement data flow
Server Components for fetching by default. React Query/SWR for client-side caching. Zustand for client-only UI state. Server Actions for mutations. URL search params for shareable state. Read `engineering.md`.

### Step 9 — Add motion purposefully
CSS transitions for simple state changes. View Transitions API for page navigation. Framer Motion for complex orchestration. Always respect `prefers-reduced-motion`. Read `animation-transitions.md`.

### Step 10 — Optimize performance
Measure Core Web Vitals. Optimize LCP (priority images, font preloading). Minimize CLS (size images, reserve space). Reduce INP (minimize client JS, `startTransition`). Read `performance.md`.

### Step 11 — Handle forms and validation
Client-side validation for UX, server-side validation for security. Progressive enhancement with Server Actions. `useActionState` + `useFormStatus` for submission states. Read `forms-validation.md`.

### Step 12 — Test everything
Vitest for utilities, Testing Library for components, Playwright for E2E, axe-core for accessibility. Read `testing.md`.

### Step 13 — Secure the application
Auth.js for authentication. Middleware for route protection. CSP headers. Zod for all server-side input validation. `httpOnly` cookies for auth tokens. Read `auth-security.md`.

## Quick reference: what changed in React 19 + Next.js 15 vs previous

| Aspect | Previous (React 18 / Next.js 14) | Current (React 19 / Next.js 15) |
|--------|-----------------------------------|----------------------------------|
| Server Components | Stable but cautious adoption | Default, mature, streaming everywhere |
| Data fetching | `fetch()` in SC, `getServerSideProps` legacy | Server Components + `use()` + Server Actions |
| Forms | Client-side only, manual fetch | `useActionState`, `useFormStatus`, Server Actions |
| Optimistic UI | Manual implementation | `useOptimistic` built-in |
| Caching | Aggressive by default (fetch cached) | Opt-in (`fetch` no longer cached by default) |
| `ref` prop | `forwardRef` wrapper required | `ref` is a regular prop — no wrapper |
| Context | `useContext(MyContext)` | `use(MyContext)` — works in conditionals/loops |
| Promises in render | Not possible | `use(promise)` — unwrap in render |
| Tailwind config | `tailwind.config.js` | CSS-first `@theme` — no JS config |
| Bundler (dev) | Webpack | Turbopack stable |
| Partial prerendering | Experimental | Stable PPR |
| `<form>` action | Not supported | `<form action={serverAction}>` native |

## What has NOT changed (carry forward)

### Design fundamentals
- **Semantic HTML** — `nav`, `main`, `section`, `article`, `aside`, `header`, `footer`, `button`, `a`
- **WCAG AA contrast** — 4.5:1 normal text, 3:1 large text and UI components
- **CSS Box Model** — Flexbox, Grid, `box-sizing: border-box` everywhere
- **Mobile-first** — base styles = smallest viewport, layer up with `min-width`
- **Dark mode** — `prefers-color-scheme`, semantic color tokens
- **44×44px minimum touch targets** — on touch devices
- **Focus indicators** — visible `:focus-visible` outlines on all interactive elements

### Engineering fundamentals
- **React hooks** — `useState`, `useEffect`, `useRef`, `useMemo`, `useCallback`, `useContext`
- **JSX/TSX component model** — props, children, composition
- **File-based routing** — `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`, `not-found.tsx`
- **`next/image`** — optimized images with `width`/`height` or `fill`
- **`next/link`** — client-side navigation with prefetching
- **`next/font`** — zero-layout-shift font loading
- **Middleware** — `middleware.ts` at project root
- **Environment variables** — `.env.local`, `NEXT_PUBLIC_` prefix for client-exposed vars
- **HTTP fundamentals** — REST patterns, status codes, caching headers
- **Zod** — schema validation at every system boundary

## Anti-patterns: what NOT to do

### Process
0. **Don't assume requirements** — This is the #1 anti-pattern. Never guess page layout, color palette, data source, rendering strategy, or component choices. Use `AskUserQuestion` with structured options first.

### Design
1. **Don't use div soup** — use semantic HTML. Screen readers and SEO depend on it.
2. **Don't hardcode colors** — use CSS custom properties or Tailwind design tokens. Raw hex values break dark mode and consistency.
3. **Don't skip dark mode** — design the dark palette upfront, not as an afterthought.
4. **Don't ignore mobile** — design mobile-first. Desktop-first creates painful responsive retrofits.
5. **Don't use fixed dimensions for text containers** — use `max-width` with `ch` units or percentages.
6. **Don't nest interactive elements** — no links inside buttons, no buttons inside links. Invalid HTML and breaks a11y.
7. **Don't rely solely on color to convey meaning** — add icons, text, or patterns for colorblind users.
8. **Don't use CSS `!important`** — fix specificity with layers or more specific selectors.
9. **Don't create layout shift** — size all images, reserve space for dynamic content.

### Engineering
10. **Don't use `'use client'` by default** — Server Components are the default. Only add when truly needed.
11. **Don't fetch data in Client Components when a Server Component can** — move fetching to the server boundary.
12. **Don't use `useEffect` for data fetching** — causes waterfalls and race conditions. Use Server Components or React Query.
13. **Don't use `any` in TypeScript** — use `unknown` and narrow, or define proper types.
14. **Don't mutate state directly** — immutable patterns only (spread, `map`, `filter`).
15. **Don't put secrets in client code** — no `NEXT_PUBLIC_` for secrets. Use server-only modules.
16. **Don't skip error boundaries** — every route needs `error.tsx`. Every async op needs `try/catch`.
17. **Don't use index as key in dynamic lists** — use stable unique identifiers.
18. **Don't import heavy libraries client-side without code splitting** — use `dynamic()`.
19. **Don't skip server-side form validation** — client validation is UX; server validation is security.
20. **Don't use `localStorage` for auth tokens** — use `httpOnly` cookies. `localStorage` is XSS-vulnerable.

## Code generation guidelines

**Pre-check: Have you gathered requirements?** If there's any ambiguity about page type, navigation, data source, rendering strategy, or visual style — stop and ask. See RULE #1.

### UI layer
1. Use Server Components by default. Add `'use client'` only when interactivity is required.
2. Use semantic HTML elements throughout.
3. Use Tailwind v4 utilities with `@theme` tokens. No arbitrary values when a token exists.
4. Apply responsive classes mobile-first (base → `sm:` → `md:` → `lg:` → `xl:`).
5. Supply `aria-label` on interactive elements without visible text.
6. Use `next/image` for all images with explicit `width`/`height` or `fill`.
7. Use `next/font` for all fonts.
8. Wrap async client content in `<Suspense>` with meaningful fallbacks.
9. Respect `prefers-reduced-motion`, `prefers-color-scheme`, and `prefers-contrast`.
10. Never disable focus outlines without a visible alternative.

### Architecture layer
11. TypeScript strict mode. No `any`. Validate external data with Zod.
12. Fetch data in Server Components, pass as props to Client Components.
13. Use Server Actions for mutations. Validate server-side with Zod.
14. Use `useActionState` + `useFormStatus` for form states.
15. Use React Query/SWR for client-side server state with caching needs.
16. Use Zustand for client-only UI state (theme, sidebar, preferences).
17. Co-locate components, tests, and styles by feature.
18. Handle errors with `error.tsx` and typed error responses.
19. Use middleware for auth guards, redirects, and response headers.
20. Write tests: Vitest for units, Testing Library for components, Playwright for E2E.
