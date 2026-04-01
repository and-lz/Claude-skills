# Testing

## Testing philosophy

- **Testing trophy**: E2E > integration > unit (not the pyramid)
- **Test behavior, not implementation**: test what users see and do
- **Co-locate tests**: `Button.test.tsx` next to `Button.tsx`
- **Query priority**: getByRole > getByLabelText > getByText > getByTestId

## Vitest setup

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov'],
      thresholds: { lines: 80, functions: 80 },
    },
  },
})

// src/test/setup.ts
import '@testing-library/jest-dom'
import { cleanup } from '@testing-library/react'
import { afterEach } from 'vitest'

afterEach(cleanup)
```

## Unit tests

```typescript
// utils/format.test.ts
import { describe, it, expect } from 'vitest'
import { formatCurrency, truncate } from './format'

describe('formatCurrency', () => {
  it('formats USD', () => {
    expect(formatCurrency(1234.56, 'en-US', 'USD')).toBe('$1,234.56')
  })
  it('formats zero', () => {
    expect(formatCurrency(0, 'en-US', 'USD')).toBe('$0.00')
  })
})

// Zod schema tests
import { UserSchema } from './schemas'

describe('UserSchema', () => {
  it('accepts valid user', () => {
    expect(UserSchema.safeParse({ email: 'a@b.com', name: 'Alice' }).success).toBe(true)
  })
  it('rejects invalid email', () => {
    const result = UserSchema.safeParse({ email: 'not-an-email', name: 'Alice' })
    expect(result.success).toBe(false)
    expect(result.error?.flatten().fieldErrors.email).toBeDefined()
  })
})
```

## Component tests with Testing Library

```tsx
// components/LoginForm.test.tsx
import { render, screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { vi } from 'vitest'
import { LoginForm } from './LoginForm'
import { signIn } from '@/features/auth/actions'

vi.mock('@/features/auth/actions')

describe('LoginForm', () => {
  it('submits with valid credentials', async () => {
    vi.mocked(signIn).mockResolvedValue({ success: true })
    const user = userEvent.setup()

    render(<LoginForm />)

    await user.type(screen.getByLabelText(/email/i), 'test@example.com')
    await user.type(screen.getByLabelText(/password/i), 'password123')
    await user.click(screen.getByRole('button', { name: /sign in/i }))

    await waitFor(() => {
      expect(signIn).toHaveBeenCalledWith(expect.any(Object), expect.any(FormData))
    })
  })

  it('shows field errors on invalid input', async () => {
    const user = userEvent.setup()
    render(<LoginForm />)

    await user.click(screen.getByRole('button', { name: /sign in/i }))

    expect(await screen.findByText(/email is required/i)).toBeInTheDocument()
  })

  it('shows server error', async () => {
    vi.mocked(signIn).mockResolvedValue({ error: 'Invalid credentials' })
    const user = userEvent.setup()

    render(<LoginForm />)
    await user.type(screen.getByLabelText(/email/i), 'wrong@example.com')
    await user.type(screen.getByLabelText(/password/i), 'wrongpassword')
    await user.click(screen.getByRole('button', { name: /sign in/i }))

    expect(await screen.findByRole('alert')).toHaveTextContent(/invalid credentials/i)
  })
})
```

## MSW for API mocking

```typescript
// src/test/mocks/handlers.ts
import { http, HttpResponse } from 'msw'

export const handlers = [
  http.get('/api/users', () => {
    return HttpResponse.json([
      { id: '1', name: 'Alice', email: 'alice@example.com' },
      { id: '2', name: 'Bob',   email: 'bob@example.com' },
    ])
  }),

  http.post('/api/users', async ({ request }) => {
    const body = await request.json() as { name: string; email: string }
    return HttpResponse.json({ id: '3', ...body }, { status: 201 })
  }),

  http.delete('/api/users/:id', () => {
    return new HttpResponse(null, { status: 204 })
  }),
]

// src/test/mocks/server.ts
import { setupServer } from 'msw/node'
import { handlers } from './handlers'
export const server = setupServer(...handlers)

// src/test/setup.ts
import { server } from './mocks/server'
import { beforeAll, afterAll, afterEach } from 'vitest'

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

## Mocking Next.js

```typescript
// src/test/setup.ts — mock next/navigation
vi.mock('next/navigation', () => ({
  useRouter: () => ({ push: vi.fn(), replace: vi.fn(), back: vi.fn(), prefetch: vi.fn() }),
  usePathname: () => '/',
  useSearchParams: () => new URLSearchParams(),
  useParams: () => ({}),
  redirect: vi.fn(),
  notFound: vi.fn(),
}))

// Mock next-intl
vi.mock('next-intl', () => ({
  useTranslations: () => (key: string) => key,
  useLocale: () => 'en',
}))
```

## Playwright E2E

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'Mobile Safari', use: { ...devices['iPhone 14'] } },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
})
```

```typescript
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Authentication', () => {
  test('user can log in', async ({ page }) => {
    await page.goto('/login')

    await page.getByLabel('Email').fill('test@example.com')
    await page.getByLabel('Password').fill('password123')
    await page.getByRole('button', { name: 'Sign in' }).click()

    await expect(page).toHaveURL('/dashboard')
    await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible()
  })

  test('shows error for invalid credentials', async ({ page }) => {
    await page.goto('/login')

    await page.getByLabel('Email').fill('wrong@example.com')
    await page.getByLabel('Password').fill('wrongpass')
    await page.getByRole('button', { name: 'Sign in' }).click()

    await expect(page.getByRole('alert')).toContainText('Invalid credentials')
  })
})

// Page Object Model
class LoginPage {
  constructor(private page: Page) {}

  async goto() { await this.page.goto('/login') }
  async login(email: string, password: string) {
    await this.page.getByLabel('Email').fill(email)
    await this.page.getByLabel('Password').fill(password)
    await this.page.getByRole('button', { name: 'Sign in' }).click()
  }
  get errorAlert() { return this.page.getByRole('alert') }
}
```

```typescript
// Reuse auth state across tests
// e2e/fixtures.ts
import { test as base } from '@playwright/test'

export const test = base.extend({
  authenticatedPage: async ({ page }, use) => {
    // Log in once, reuse session
    await page.goto('/login')
    await page.getByLabel('Email').fill(process.env.TEST_USER_EMAIL!)
    await page.getByLabel('Password').fill(process.env.TEST_USER_PASSWORD!)
    await page.getByRole('button', { name: 'Sign in' }).click()
    await page.waitForURL('/dashboard')
    await use(page)
  },
})
```

## Accessibility testing

```typescript
// Component test with axe
import { axe, toHaveNoViolations } from 'jest-axe'
expect.extend(toHaveNoViolations)

test('ContactForm has no a11y violations', async () => {
  const { container } = render(<ContactForm />)
  expect(await axe(container)).toHaveNoViolations()
})

// Playwright a11y
import { checkA11y, injectAxe } from 'axe-playwright'

test('home page is accessible', async ({ page }) => {
  await page.goto('/')
  await injectAxe(page)
  await checkA11y(page, undefined, {
    detailedReport: true,
    detailedReportOptions: { html: true },
  })
})
```

## CI workflow

```yaml
# .github/workflows/test.yml
name: Test
on: [push, pull_request]

jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npm run test:coverage
      - uses: codecov/codecov-action@v4

  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npm run build
      - run: npx playwright test --shard=${{ matrix.shardIndex }}/${{ matrix.shardTotal }}
    strategy:
      matrix:
        shardIndex: [1, 2, 3]
        shardTotal: [3]
```
