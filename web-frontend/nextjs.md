# Next.js — Routing, RSC, Server Actions, Middleware, Deployment

## App Router file conventions

| File | Purpose |
|------|---------|
| `page.tsx` | Unique UI for a route segment — makes route publicly accessible |
| `layout.tsx` | Shared UI wrapping children — persists across navigations |
| `loading.tsx` | Instant loading UI — auto-wraps page in `<Suspense>` |
| `error.tsx` | Error UI boundary — must be `'use client'` |
| `not-found.tsx` | 404 UI — triggered by `notFound()` |
| `template.tsx` | Like layout but re-mounts on navigation (use rarely) |
| `default.tsx` | Fallback for parallel routes when slot has no match |
| `route.ts` | API endpoint (Route Handler) — no UI |
| `middleware.ts` | Runs before request — auth, redirects, headers |

## Routing patterns

### Route groups — organize without affecting URL
```
app/
├── (marketing)/          # URL: /
│   ├── page.tsx          # /
│   ├── about/page.tsx    # /about
│   └── layout.tsx        # marketing layout (no auth)
├── (app)/                # URL: /dashboard
│   ├── dashboard/page.tsx
│   ├── settings/page.tsx
│   └── layout.tsx        # app shell layout (with auth)
└── layout.tsx            # root layout
```

### Dynamic routes
```
app/
├── blog/
│   ├── page.tsx                    # /blog
│   └── [slug]/page.tsx             # /blog/my-post
├── shop/
│   └── [...categories]/page.tsx   # /shop/a/b/c (catch-all)
├── docs/
│   └── [[...slug]]/page.tsx       # /docs and /docs/a/b (optional catch-all)
```

```typescript
// Dynamic segment params are passed as props
export default async function BlogPost({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params // Next.js 15 — params is a Promise
  const post = await getPost(slug)
  if (!post) notFound()
  return <Article post={post} />
}

// Generate static params for SSG
export async function generateStaticParams() {
  const posts = await getAllPosts()
  return posts.map(post => ({ slug: post.slug }))
}
```

### Parallel routes — show multiple pages simultaneously
```
app/
└── dashboard/
    ├── @analytics/page.tsx    # @slot
    ├── @team/page.tsx         # @slot
    ├── page.tsx
    └── layout.tsx             # receives slots as props
```

```typescript
// layout.tsx receives slot content as props
export default function DashboardLayout({
  children,
  analytics,
  team,
}: {
  children: React.ReactNode
  analytics: React.ReactNode
  team: React.ReactNode
}) {
  return (
    <div className="grid grid-cols-2">
      {children}
      {analytics}
      {team}
    </div>
  )
}
```

### Intercepting routes — modals with shareable URLs
```
app/
├── feed/
│   └── page.tsx
├── photo/
│   └── [id]/page.tsx           # /photo/123 (direct access = full page)
└── (.)photo/
    └── [id]/page.tsx           # intercepts /photo/123 from /feed = modal
```

## Server Components vs Client Components

### Decision tree
```
Does it need:
  - onClick, onChange, other event handlers? → 'use client'
  - useState, useEffect, useRef, custom hooks? → 'use client'
  - Browser APIs (localStorage, window, navigator)? → 'use client'
  - Context (consuming)? → 'use client'
  
Default: Server Component — no directive needed
```

### Composition pattern — push 'use client' to leaves
```typescript
// ✅ Good — Server Component fetches data, Client Component handles interaction
async function ProductPage({ id }: { id: string }) {
  const product = await db.product.findUnique({ where: { id } })
  return (
    <div>
      <h1>{product.name}</h1>
      <ProductImages images={product.images} />  {/* Server Component */}
      <AddToCartButton productId={id} />          {/* Client Component */}
    </div>
  )
}

// ✅ Pass Server Components as children to Client Components
'use client'
function InteractiveWrapper({ children }: { children: React.ReactNode }) {
  const [expanded, setExpanded] = useState(false)
  return (
    <div>
      <button onClick={() => setExpanded(!expanded)}>Toggle</button>
      {expanded && children}  {/* children can be Server Components! */}
    </div>
  )
}
```

### Serialization boundary rules
```typescript
// ❌ Cannot pass non-serializable props across the boundary
<ClientComponent fn={() => {}} />              // functions not serializable
<ClientComponent date={new Date()} />          // Date not serializable

// ✅ Pass serializable data
<ClientComponent dateString={date.toISOString()} />
<ClientComponent onClick="deleteUser" />       // pass action name, not fn
```

## Server Actions

```typescript
// features/users/actions.ts
'use server'
import { z } from 'zod'
import { revalidateTag, revalidatePath } from 'next/cache'
import { redirect } from 'next/navigation'
import { auth } from '@/lib/auth'

const CreateUserSchema = z.object({
  name: z.string().min(1, 'Name is required').max(100),
  email: z.string().email('Invalid email'),
})

export type CreateUserState = {
  error?: string
  fieldErrors?: Record<string, string[]>
  success?: boolean
}

export async function createUser(
  _prevState: CreateUserState,
  formData: FormData
): Promise<CreateUserState> {
  // Auth check
  const session = await auth()
  if (!session) return { error: 'Unauthorized' }

  // Validate
  const parsed = CreateUserSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
  })
  if (!parsed.success) {
    return { fieldErrors: parsed.error.flatten().fieldErrors }
  }

  // Mutate
  try {
    await db.user.create({ data: parsed.data })
  } catch (e) {
    if (e instanceof PrismaClientKnownRequestError && e.code === 'P2002') {
      return { error: 'Email already in use' }
    }
    return { error: 'Failed to create user' }
  }

  // Revalidate
  revalidateTag('users')

  // Redirect (throws internally — don't wrap in try/catch)
  redirect('/users')
}
```

### Server Action — inline (for simple mutations)
```typescript
// Can define inline in Server Components
async function ProductCard({ product }: { product: Product }) {
  async function deleteProduct() {
    'use server'
    await db.product.delete({ where: { id: product.id } })
    revalidatePath('/products')
  }

  return (
    <div>
      <h2>{product.name}</h2>
      <form action={deleteProduct}>
        <button type="submit">Delete</button>
      </form>
    </div>
  )
}
```

## Middleware

```typescript
// middleware.ts — runs on every matching request
import { NextResponse, type NextRequest } from 'next/server'
import { auth } from '@/lib/auth'

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl

  // Auth guard
  const session = await auth()
  const isAuthPage = pathname.startsWith('/login') || pathname.startsWith('/register')
  const isProtected = pathname.startsWith('/dashboard') || pathname.startsWith('/settings')

  if (isProtected && !session) {
    const loginUrl = new URL('/login', request.url)
    loginUrl.searchParams.set('callbackUrl', pathname)
    return NextResponse.redirect(loginUrl)
  }

  if (isAuthPage && session) {
    return NextResponse.redirect(new URL('/dashboard', request.url))
  }

  // Add security headers
  const response = NextResponse.next()
  response.headers.set('X-Frame-Options', 'DENY')
  response.headers.set('X-Content-Type-Options', 'nosniff')
  return response
}

export const config = {
  matcher: [
    // Skip static files and API routes
    '/((?!_next/static|_next/image|favicon.ico|api/).*)',
  ],
}
```

## Metadata and SEO

```typescript
// Static metadata
export const metadata: Metadata = {
  title: 'My App',
  description: 'App description',
}

// Dynamic metadata
export async function generateMetadata({
  params,
}: {
  params: Promise<{ slug: string }>
}): Promise<Metadata> {
  const { slug } = await params
  const post = await getPost(slug)
  if (!post) return { title: 'Not Found' }

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [{ url: post.coverImage, width: 1200, height: 630 }],
    },
    alternates: {
      canonical: `/blog/${slug}`,
    },
  }
}
```

### Dynamic OG image
```typescript
// app/blog/[slug]/opengraph-image.tsx
import { ImageResponse } from 'next/og'

export const size = { width: 1200, height: 630 }
export const contentType = 'image/png'

export default async function OGImage({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params
  const post = await getPost(slug)

  return new ImageResponse(
    <div style={{ display: 'flex', flexDirection: 'column', padding: 60, background: '#fff' }}>
      <p style={{ fontSize: 48, fontWeight: 700 }}>{post?.title}</p>
    </div>
  )
}
```

### Sitemap and robots
```typescript
// app/sitemap.ts
export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts = await getAllPosts()
  return [
    { url: 'https://example.com', lastModified: new Date() },
    ...posts.map(post => ({
      url: `https://example.com/blog/${post.slug}`,
      lastModified: post.updatedAt,
    })),
  ]
}

// app/robots.ts
export default function robots(): MetadataRoute.Robots {
  return {
    rules: { userAgent: '*', allow: '/', disallow: '/api/' },
    sitemap: 'https://example.com/sitemap.xml',
  }
}
```

## Streaming and Suspense

```typescript
// Granular Suspense boundaries for independent data
export default function DashboardPage() {
  return (
    <main>
      {/* Renders immediately — no async */}
      <DashboardHeader />

      {/* Independent — suspends individually */}
      <Suspense fallback={<RevenueCardSkeleton />}>
        <RevenueCard />
      </Suspense>

      <div className="grid grid-cols-2 gap-4">
        <Suspense fallback={<ChartSkeleton />}>
          <SalesChart />
        </Suspense>
        <Suspense fallback={<TableSkeleton />}>
          <RecentOrders />
        </Suspense>
      </div>
    </main>
  )
}
```

## Image optimization

```typescript
import Image from 'next/image'

// Fixed dimensions
<Image
  src="/hero.jpg"
  alt="Hero image showing product dashboard"
  width={1200}
  height={630}
  priority          // LCP image — skip lazy loading
  quality={85}
/>

// Responsive fill (parent must have position: relative)
<div className="relative aspect-video">
  <Image
    src={post.coverImage}
    alt={post.title}
    fill
    sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
    className="object-cover"
  />
</div>

// Remote images — configure in next.config
// next.config.ts
images: {
  remotePatterns: [
    { protocol: 'https', hostname: 'cdn.example.com' },
  ],
}
```

## Font optimization

```typescript
// app/layout.tsx
import { Inter, Playfair_Display } from 'next/font/google'

const inter = Inter({
  subsets: ['latin'],
  variable: '--font-inter',  // expose as CSS variable
  display: 'swap',
})

const playfair = Playfair_Display({
  subsets: ['latin'],
  variable: '--font-playfair',
  weight: ['400', '700'],
})

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${inter.variable} ${playfair.variable}`}>
      <body className="font-sans">{children}</body>
    </html>
  )
}

// Local font
import localFont from 'next/font/local'
const myFont = localFont({
  src: [
    { path: '../fonts/MyFont-Regular.woff2', weight: '400' },
    { path: '../fonts/MyFont-Bold.woff2', weight: '700' },
  ],
  variable: '--font-my',
})
```

## Partial Prerendering (PPR)

```typescript
// next.config.ts — enable PPR
experimental: { ppr: true }

// page.tsx — static shell renders instantly, dynamic parts stream in
import { Suspense } from 'react'

export default function ProductPage({ params }: { params: { id: string } }) {
  return (
    <div>
      {/* Static — included in HTML shell */}
      <ProductDetails id={params.id} />

      {/* Dynamic — streams in after initial response */}
      <Suspense fallback={<RecommendationsSkeleton />}>
        <Recommendations productId={params.id} />
      </Suspense>

      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews productId={params.id} />
      </Suspense>
    </div>
  )
}
```

## next.config.ts reference

```typescript
import type { NextConfig } from 'next'

const config: NextConfig = {
  // Image optimization
  images: {
    remotePatterns: [{ protocol: 'https', hostname: 'cdn.example.com' }],
    formats: ['image/avif', 'image/webp'],
  },

  // Security headers
  async headers() {
    return [{
      source: '/(.*)',
      headers: [
        { key: 'X-Frame-Options', value: 'DENY' },
        { key: 'X-Content-Type-Options', value: 'nosniff' },
        { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
      ],
    }]
  },

  // Redirects
  async redirects() {
    return [{ source: '/old-path', destination: '/new-path', permanent: true }]
  },

  // Bundle analysis
  experimental: {
    ppr: true,                   // Partial Prerendering
    turbopack: true,             // Turbopack for dev
    typedRoutes: true,           // Type-safe route strings
  },

  // Standalone output for Docker
  output: 'standalone',
}

export default config
```

## Deployment

### Vercel (recommended)
- Zero-config deployment — push to git
- Edge Network for static assets and edge functions
- `VERCEL_URL` env var available automatically
- ISR and on-demand revalidation work out of the box

### Docker / self-hosted
```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

Requires `output: 'standalone'` in `next.config.ts`.

### Edge runtime (for Middleware and Route Handlers)
```typescript
// Opt specific Route Handlers into Edge Runtime
export const runtime = 'edge'

// Edge limitations — no Node.js APIs:
// ✗ fs, path, crypto (Node)
// ✗ Prisma, most ORMs
// ✓ fetch, Response, Request, URL, crypto (Web API)
// ✓ @vercel/kv, @vercel/postgres (edge-compatible)
```
