---
name: nextjs-testing
description: Sets up and writes tests for a Next.js App Router project, with Vitest and React Testing Library for components and server logic, and Playwright for end-to-end flows including async Server Components. Use when adding test tooling to a Next.js app, writing tests for a new page, form or Server Action, or deciding which layer a test belongs in.
license: MIT
---

# Testing a Next.js App Router project

## Pick the layer first

| What you are testing | Tool | Why |
|---|---|---|
| Pure functions, schemas, data helpers | Vitest | Fast, no DOM |
| Client Components and synchronous Server Components | Vitest with jsdom and Testing Library | Renders in memory |
| Server Action logic (validation, auth branches) | Vitest, with `next/*` modules mocked | The function is plain async code |
| Async Server Components, routing, caching, redirects, full forms | Playwright | Vitest can't render async Server Components |

If a test needs a real request, cookies or a redirect, write it in Playwright. Don't mock your way around the framework.

## Vitest setup

```bash
npm install -D vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/dom vite-tsconfig-paths
```

```ts
// vitest.config.mts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [tsconfigPaths(), react()],
  test: {
    environment: 'jsdom',
    exclude: ['e2e/**', 'node_modules/**'],
  },
})
```

Add `"test": "vitest"` to `package.json` scripts. Drop `vite-tsconfig-paths` in a JavaScript project.

## Component test

```tsx
// app/posts/new/post-form.test.tsx
import { render, screen } from '@testing-library/react'
import { expect, test, vi } from 'vitest'

vi.mock('../actions', () => ({ createPost: vi.fn() }))

import { PostForm } from './post-form'

test('renders the title field and save button', () => {
  render(<PostForm />)
  expect(screen.getByRole('textbox', { name: /title/i })).toBeDefined()
  expect(screen.getByRole('button', { name: 'Save' })).toBeDefined()
})
```

Query by role and accessible name. If a query fails because a field has no label, fix the markup.

## Server Action test

Mock what only exists inside a Next.js request.

```ts
// app/posts/actions.test.ts
import { beforeEach, expect, test, vi } from 'vitest'

vi.mock('next/cache', () => ({ revalidatePath: vi.fn() }))
vi.mock('next/navigation', () => ({
  redirect: vi.fn(() => { throw new Error('NEXT_REDIRECT') }),
}))
vi.mock('@/lib/auth', () => ({ auth: vi.fn() }))
vi.mock('@/lib/db', () => ({ db: { post: { create: vi.fn() } } }))

import { createPost } from './actions'
import { auth } from '@/lib/auth'
import { db } from '@/lib/db'

beforeEach(() => vi.clearAllMocks())

function form(fields: Record<string, string>) {
  const fd = new FormData()
  for (const [k, v] of Object.entries(fields)) fd.set(k, v)
  return fd
}

test('rejects an empty title without writing', async () => {
  vi.mocked(auth).mockResolvedValue({ user: { id: 'u1' } } as any)
  const state = await createPost({}, form({ title: '', body: 'x' }))
  expect(state.errors?.title).toBeDefined()
  expect(db.post.create).not.toHaveBeenCalled()
})

test('refuses anonymous users', async () => {
  vi.mocked(auth).mockResolvedValue(null as any)
  const state = await createPost({}, form({ title: 'a', body: 'b' }))
  expect(state.message).toMatch(/sign in/i)
  expect(db.post.create).not.toHaveBeenCalled()
})
```

Every action should have one test per guard: unauthenticated, unauthorized, invalid input, and success.

## Playwright setup

```bash
npm init playwright@latest
```

```ts
// playwright.config.ts
import { defineConfig } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  use: { baseURL: 'http://localhost:3000' },
  webServer: {
    command: 'npm run build && npm run start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 180_000,
  },
})
```

Test against a production build. The dev server compiles on demand and hides caching behaviour.

```ts
// e2e/posts.spec.ts
import { test, expect } from '@playwright/test'

test('a signed in user can publish a post', async ({ page }) => {
  await page.goto('/posts/new')
  await page.getByRole('textbox', { name: 'Title' }).fill('Hello')
  await page.getByRole('textbox', { name: 'Body' }).fill('First post')
  await page.getByRole('button', { name: 'Save' }).click()
  await expect(page).toHaveURL(/\/posts\/[\w-]+$/)
  await expect(page.getByRole('heading', { name: 'Hello' })).toBeVisible()
})
```

For signed in flows, log in once in a setup project and reuse `storageState`, rather than logging in for every test.

## Checks before you finish

- `npx vitest run` and `npx playwright test` both pass locally.
- Vitest doesn't pick up `e2e/` specs, and Playwright doesn't pick up `*.test.tsx`.
- No test renders an async Server Component with Testing Library.
- Each Server Action has tests for its auth and validation branches.
- Playwright assertions use web-first `expect(locator)` matchers, not fixed waits.
