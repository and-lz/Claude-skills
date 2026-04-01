# Performance & Core Web Vitals

## Core Web Vitals targets

| Metric | Good | Needs Improvement | Poor |
|--------|------|-------------------|------|
| LCP (Largest Contentful Paint) | < 2.5s | 2.5–4s | > 4s |
| CLS (Cumulative Layout Shift) | < 0.1 | 0.1–0.25 | > 0.25 |
| INP (Interaction to Next Paint) | < 200ms | 200–500ms | > 500ms |

## LCP optimization

LCP is usually the hero image, headline, or above-the-fold content.

```tsx
// 1. Priority flag on LCP image — skips lazy loading
<Image
  src="/hero.jpg"
  alt="Hero"
  width={1200}
  height={630}
  priority              // adds <link rel="preload"> in <head>
  quality={85}
/>

// 2. Preload critical font
// next/font handles this automatically — always use it

// 3. Minimize server response time
// Use ISR or static generation for marketing pages
// Avoid heavy database queries on initial render — stream them

// 4. Avoid render-blocking resources
// next/script with afterInteractive or lazyOnload for non-critical scripts
```

## CLS optimization

CLS is caused by content shifting after initial render.

```tsx
// 1. Always size images — never let them load unsized
<Image src={src} alt={alt} width={800} height={450} />          // explicit
<div className="relative aspect-video"><Image fill /></div>     // fill in sized container

// 2. Reserve space for dynamic content
<div className="min-h-[120px]">  {/* reserve space before data loads */}
  {data ? <Content /> : <Skeleton />}
</div>

// 3. next/font eliminates FOIT/FOUT layout shift
// font-display: swap + preloaded — no shift

// 4. Don't insert content above existing content
// Banners, cookie notices — use position: fixed or sticky
```

## INP optimization

INP measures responsiveness — time from user interaction to next frame.

```tsx
// 1. Minimize client-side JS bundle
// Use Server Components — they ship zero JS

// 2. startTransition for non-urgent updates
import { startTransition, useState } from 'react'

function SearchPage() {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState([])

  function handleInput(value: string) {
    setQuery(value)  // urgent — update input immediately
    startTransition(() => {
      setResults(search(value))  // non-urgent — can be interrupted
    })
  }
}

// 3. Break up long tasks with scheduler.yield()
async function processLargeList(items: Item[]) {
  for (let i = 0; i < items.length; i++) {
    processItem(items[i])
    if (i % 50 === 0) {
      await scheduler.yield()  // yield to browser every 50 items
    }
  }
}

// 4. Debounce expensive operations
import { useDeferredValue } from 'react'
function SearchResults({ query }: { query: string }) {
  const deferredQuery = useDeferredValue(query)  // stale while typing
  return <Results query={deferredQuery} />
}
```

## Bundle optimization

```bash
# Analyze bundle
npm install @next/bundle-analyzer
```

```tsx
// next.config.ts
import bundleAnalyzer from '@next/bundle-analyzer'
const withBundleAnalyzer = bundleAnalyzer({ enabled: process.env.ANALYZE === 'true' })
export default withBundleAnalyzer(config)

// Run: ANALYZE=true npm run build
```

```tsx
// ❌ Barrel file — imports entire module even if you need one function
import { formatDate } from '@/utils'  // if utils/index.ts re-exports everything

// ✅ Direct import — tree-shakeable
import { formatDate } from '@/utils/date'

// ❌ Named import from lodash — still bundles everything
import { debounce } from 'lodash'

// ✅ Deep import
import debounce from 'lodash/debounce'
// Or use native alternatives: lodash.debounce, radash, or write it yourself
```

## Dynamic imports

```tsx
import dynamic from 'next/dynamic'

// Heavy client component — loads only when needed
const RichTextEditor = dynamic(() => import('@/components/RichTextEditor'), {
  loading: () => <Skeleton className="h-64" />,
  ssr: false,  // skip SSR for browser-only (Monaco, draft-js, etc.)
})

const Chart = dynamic(() => import('@/components/Chart'), {
  loading: () => <ChartSkeleton />,
})

// Conditional — only load when feature is enabled
const DevTools = dynamic(() => import('@/components/DevTools'), { ssr: false })
{isDev && <DevTools />}
```

## Image optimization

```tsx
// Responsive image with srcset
<Image
  src="/product.jpg"
  alt={product.name}
  fill
  sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw"
  className="object-cover"
/>

// Blur placeholder — prevents CLS and looks polished
<Image
  src={post.coverImage}
  alt={post.title}
  width={800}
  height={450}
  placeholder="blur"
  blurDataURL={post.blurDataURL}  // tiny base64 image
/>

// Generate blur placeholder at build time
import { getPlaiceholder } from 'plaiceholder'
const { base64 } = await getPlaiceholder(imagePath)
```

```tsx
// next.config.ts — remote image patterns
images: {
  remotePatterns: [
    { protocol: 'https', hostname: 'cdn.example.com', pathname: '/images/**' },
  ],
  formats: ['image/avif', 'image/webp'],  // AVIF first — better compression
}
```

## Font optimization

```tsx
// Always next/font — zero layout shift, self-hosted, preloaded
import { Inter } from 'next/font/google'

const inter = Inter({
  subsets: ['latin'],       // only load needed character sets
  variable: '--font-sans',
  display: 'swap',
})

// Variable font — single file for all weights
import localFont from 'next/font/local'
const geist = localFont({
  src: '../fonts/GeistVF.woff2',
  variable: '--font-sans',
})
```

## Caching strategies

```tsx
// 1. Static (SSG) — cached indefinitely, regenerated at build
export const revalidate = false  // or omit

// 2. Time-based ISR — cached, regenerated after N seconds
export const revalidate = 3600  // 1 hour

// 3. On-demand — cached until manually invalidated
const data = await fetch(url, { next: { tags: ['products'] } })
// After mutation:
revalidateTag('products')
revalidatePath('/products')

// 4. No cache — always fresh
const data = await fetch(url, { cache: 'no-store' })

// 5. Browser cache via headers
// next.config.ts
async headers() {
  return [{
    source: '/static/(.*)',
    headers: [{ key: 'Cache-Control', value: 'public, max-age=31536000, immutable' }],
  }]
}
```

## Third-party scripts

```tsx
import Script from 'next/script'

// Analytics — load after page is interactive
<Script src="https://www.googletagmanager.com/gtag/js" strategy="afterInteractive" />

// Chat widget — load lazily
<Script src="https://cdn.intercom.io/widget.js" strategy="lazyOnload" />

// Partytown — run in web worker (zero main thread impact)
<Script src="/analytics.js" strategy="worker" />
```

## Prefetching

```tsx
// next/link — prefetches on hover by default in production
<Link href="/dashboard">Dashboard</Link>

// Disable prefetch for rarely-visited pages
<Link href="/admin" prefetch={false}>Admin</Link>

// Programmatic prefetch
import { useRouter } from 'next/navigation'
const router = useRouter()
// On hover
onMouseEnter={() => router.prefetch('/dashboard')}
```

```html
<!-- Resource hints in layout.tsx -->
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="dns-prefetch" href="https://cdn.example.com" />
```

## Monitoring

```tsx
// web-vitals — report to analytics
import { onCLS, onINP, onLCP } from 'web-vitals'

function reportWebVitals({ name, value, rating }: Metric) {
  // Send to your analytics
  analytics.track('web_vital', { name, value, rating })
}

onCLS(reportWebVitals)
onINP(reportWebVitals)
onLCP(reportWebVitals)

// Next.js built-in — app/layout.tsx
export function reportWebVitals(metric: NextWebVitalsMetric) {
  console.log(metric)
}
```

```yaml
# Lighthouse CI — .github/workflows/lhci.yml
- name: Run Lighthouse CI
  run: |
    npm install -g @lhci/cli
    lhci autorun
  env:
    LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
```
