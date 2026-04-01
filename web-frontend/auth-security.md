# Auth & Security

## Authentication overview

| Approach | Storage | XSS risk | CSRF risk | Use when |
|----------|---------|----------|-----------|----------|
| httpOnly cookies | Server | None (JS can't read) | Use SameSite=Lax | Default — most secure |
| localStorage JWT | Browser | High (XSS steals it) | None | Avoid for auth tokens |
| Memory (variable) | JS heap | Low | None | Short-lived, resets on refresh |

**Always use httpOnly cookies for auth tokens.**

## Auth.js (NextAuth.js v5)

```typescript
// auth.ts
import NextAuth from 'next-auth'
import { DrizzleAdapter } from '@auth/drizzle-adapter'
import Credentials from 'next-auth/providers/credentials'
import GitHub from 'next-auth/providers/github'
import Google from 'next-auth/providers/google'
import { db } from '@/lib/db'
import bcrypt from 'bcryptjs'

export const { handlers, signIn, signOut, auth } = NextAuth({
  adapter: DrizzleAdapter(db),
  session: { strategy: 'jwt' },

  providers: [
    GitHub({ clientId: process.env.GITHUB_ID!, clientSecret: process.env.GITHUB_SECRET! }),
    Google({ clientId: process.env.GOOGLE_ID!, clientSecret: process.env.GOOGLE_SECRET! }),

    Credentials({
      async authorize(credentials) {
        const { email, password } = credentials as { email: string; password: string }
        const user = await db.query.users.findFirst({ where: eq(users.email, email) })
        if (!user?.passwordHash) return null
        const valid = await bcrypt.compare(password, user.passwordHash)
        return valid ? user : null
      },
    }),
  ],

  callbacks: {
    jwt({ token, user }) {
      if (user) token.role = user.role  // add custom claims
      return token
    },
    session({ session, token }) {
      session.user.role = token.role as string
      return session
    },
  },

  pages: {
    signIn: '/login',
    error: '/login',
  },
})

// types/next-auth.d.ts — augment session type
declare module 'next-auth' {
  interface User { role: string }
  interface Session { user: { role: string } & DefaultSession['user'] }
}
```

```typescript
// app/api/auth/[...nextauth]/route.ts
import { handlers } from '@/auth'
export const { GET, POST } = handlers
```

## Middleware auth guard

```typescript
// middleware.ts
import { auth } from '@/auth'
import { NextResponse } from 'next/server'

export default auth((req) => {
  const { pathname } = req.nextUrl
  const session = req.auth

  const publicPaths = ['/', '/login', '/register', '/about']
  const isPublic = publicPaths.includes(pathname) || pathname.startsWith('/api/auth')

  if (!isPublic && !session) {
    const loginUrl = new URL('/login', req.url)
    loginUrl.searchParams.set('callbackUrl', pathname)
    return NextResponse.redirect(loginUrl)
  }

  // Role-based access
  if (pathname.startsWith('/admin') && session?.user.role !== 'admin') {
    return NextResponse.redirect(new URL('/403', req.url))
  }

  return NextResponse.next()
})

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
}
```

## Server-side auth checks

```typescript
// In Server Components
import { auth } from '@/auth'
import { redirect } from 'next/navigation'

export default async function ProtectedPage() {
  const session = await auth()
  if (!session) redirect('/login')
  // session.user is fully typed
  return <div>Welcome, {session.user.name}</div>
}

// In Server Actions — always re-verify
'use server'
export async function deletePost(id: string) {
  const session = await auth()
  if (!session) throw new Error('Unauthorized')
  if (session.user.role !== 'admin') throw new Error('Forbidden')
  await db.post.delete({ where: { id } })
}

// In Route Handlers
export async function DELETE(req: Request, { params }: { params: { id: string } }) {
  const session = await auth()
  if (!session) return Response.json({ error: 'Unauthorized' }, { status: 401 })
  await db.post.delete({ where: { id: params.id } })
  return new Response(null, { status: 204 })
}
```

## Client-side auth state

```tsx
// app/layout.tsx — wrap with SessionProvider
import { SessionProvider } from 'next-auth/react'
export default function RootLayout({ children, session }) {
  return <SessionProvider session={session}>{children}</SessionProvider>
}

// Client Component
'use client'
import { useSession, signIn, signOut } from 'next-auth/react'

function UserMenu() {
  const { data: session, status } = useSession()

  if (status === 'loading') return <Skeleton className="h-8 w-8 rounded-full" />

  if (!session) {
    return <button onClick={() => signIn()}>Sign in</button>
  }

  return (
    <DropdownMenu>
      <DropdownMenuTrigger>
        <Avatar src={session.user.image} alt={session.user.name ?? ''} />
      </DropdownMenuTrigger>
      <DropdownMenuContent>
        <DropdownMenuItem>{session.user.email}</DropdownMenuItem>
        <DropdownMenuItem onClick={() => signOut()}>Sign out</DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  )
}
```

## CSP headers

```typescript
// next.config.ts
async headers() {
  const csp = [
    "default-src 'self'",
    "script-src 'self' 'nonce-{nonce}'",   // use nonces for inline scripts
    "style-src 'self' 'unsafe-inline'",     // Tailwind needs this or use nonce
    "img-src 'self' data: https://cdn.example.com",
    "font-src 'self'",
    "connect-src 'self' https://api.example.com",
    "frame-ancestors 'none'",
  ].join('; ')

  return [{
    source: '/(.*)',
    headers: [
      { key: 'Content-Security-Policy', value: csp },
      { key: 'X-Frame-Options', value: 'DENY' },
      { key: 'X-Content-Type-Options', value: 'nosniff' },
      { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
      { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
    ],
  }]
}
```

## XSS prevention

```tsx
// React auto-escapes — safe by default
<p>{userInput}</p>  // safe — React escapes HTML entities

// dangerouslySetInnerHTML — only use when absolutely necessary
import DOMPurify from 'dompurify'

function RichContent({ html }: { html: string }) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['p', 'strong', 'em', 'ul', 'ol', 'li', 'a'],
    ALLOWED_ATTR: ['href', 'target', 'rel'],
  })
  return <div dangerouslySetInnerHTML={{ __html: clean }} />
}

// URL validation — prevent javascript: URLs
function SafeLink({ href, children }: { href: string; children: React.ReactNode }) {
  const isSafe = href.startsWith('/') || href.startsWith('https://') || href.startsWith('http://')
  if (!isSafe) return <span>{children}</span>
  return <a href={href} rel="noopener noreferrer">{children}</a>
}
```

## Input validation (server-side)

```typescript
// Always validate with Zod on server — regardless of client validation
'use server'
import { z } from 'zod'

const UpdateProfileSchema = z.object({
  name:    z.string().min(1).max(100).trim(),
  bio:     z.string().max(500).trim().optional(),
  website: z.string().url().optional().or(z.literal('')),
})

export async function updateProfile(_prev: State, formData: FormData) {
  const session = await auth()
  if (!session) return { error: 'Unauthorized' }

  // Validate every field — never trust FormData
  const parsed = UpdateProfileSchema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) return { fieldErrors: parsed.error.flatten().fieldErrors }

  // Prisma/Drizzle use parameterized queries — no SQL injection
  await db.user.update({
    where: { id: session.user.id },
    data: parsed.data,
  })
}
```

## Environment variables

```bash
# .env.local — never commit to git
DATABASE_URL="postgresql://..."
AUTH_SECRET="random-secret-at-least-32-chars"
GITHUB_ID="..."
GITHUB_SECRET="..."

# Public (exposed to browser) — only for non-sensitive config
NEXT_PUBLIC_APP_URL="https://example.com"
NEXT_PUBLIC_POSTHOG_KEY="phc_..."
```

```typescript
// Prevent server-only modules from being imported client-side
// lib/db.ts
import 'server-only'  // throws at build time if imported in client bundle
import { drizzle } from 'drizzle-orm/postgres-js'
```

## Rate limiting

```typescript
// middleware.ts — with Upstash
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '10 s'),  // 10 requests per 10 seconds
})

export default auth(async (req) => {
  // Rate limit API routes
  if (req.nextUrl.pathname.startsWith('/api/')) {
    const ip = req.headers.get('x-forwarded-for') ?? '127.0.0.1'
    const { success, limit, remaining } = await ratelimit.limit(ip)

    if (!success) {
      return Response.json(
        { error: 'Too many requests' },
        {
          status: 429,
          headers: {
            'X-RateLimit-Limit': String(limit),
            'X-RateLimit-Remaining': String(remaining),
            'Retry-After': '10',
          },
        }
      )
    }
  }
})
```

## Security headers complete example

```typescript
// next.config.ts — production-ready security headers
async headers() {
  return [{
    source: '/(.*)',
    headers: [
      // Prevent clickjacking
      { key: 'X-Frame-Options', value: 'DENY' },
      // Prevent MIME sniffing
      { key: 'X-Content-Type-Options', value: 'nosniff' },
      // Referrer policy
      { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
      // Disable browser features
      { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=(), interest-cohort=()' },
      // HSTS — HTTPS only (enable after confirming HTTPS works)
      { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },
    ],
  }]
}
```
