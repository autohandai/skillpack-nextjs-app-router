---
name: nextjs-data-caching
description: Chooses and applies the right data fetching and caching approach in a Next.js 16 App Router project, covering Cache Components ('use cache', cacheLife, cacheTag), the earlier fetch and segment config model, Suspense streaming, and revalidation. Use when data is stale, a page is unexpectedly dynamic or static, a build fails on caching config, or when adding a new data source to a page.
license: MIT
---

# Next.js data fetching and caching

Next.js 16 has two caching models, and the right code depends on which one the project uses. Find out first.

## Step 1: detect the model

Open `next.config.ts` (or `.js`/`.mjs`).

- `cacheComponents: true` means the project uses Cache Components. Follow section A.
- No flag means the project uses the earlier model. Follow section B.

Don't mix the two. With `cacheComponents` on, route segments that export `dynamic`, `revalidate`, `fetchCache`, `dynamicParams` or `runtime` fail the build.

## Section A: Cache Components

Data is dynamic by default. You opt in to caching with `'use cache'`.

```tsx
// lib/products.ts
import { cacheLife, cacheTag } from 'next/cache'
import { db } from '@/lib/db'

export async function getProducts() {
  'use cache'
  cacheLife('hours')
  cacheTag('products')
  return db.product.findMany()
}
```

```tsx
// app/shop/page.tsx
import { Suspense } from 'react'
import { getProducts } from '@/lib/products'
import { Cart } from './cart'

export default async function Page() {
  const products = await getProducts()        // cached, part of the static shell
  return (
    <>
      <ProductGrid products={products} />
      <Suspense fallback={<p>Loading cart</p>}>
        <Cart />                               {/* reads cookies(), streams per request */}
      </Suspense>
    </>
  )
}
```

Rules:
1. Put `'use cache'` at the top of an async function, component or file. Its arguments and closed-over values become part of the cache key, so they must be serializable.
2. Don't call `cookies()`, `headers()` or read `searchParams` inside a cached scope. Read them outside and pass plain values in.
3. Wrap anything that reads request data, or that is uncached, in `<Suspense>`. Otherwise the build reports that uncached data was accessed outside a Suspense boundary.
4. For values that must differ per request but don't come from request data (`Date.now()`, `crypto.randomUUID()`), call `await connection()` from `next/server` first, inside a Suspense boundary.
5. Tag anything a write can change, and invalidate the tag in the Server Action that writes it.
6. `generateStaticParams` must return at least one entry when Cache Components is on.

## Section B: the earlier model (no cacheComponents flag)

`fetch` is not cached by default (it hasn't been since Next.js 15). Opt in for each request or each segment.

```tsx
// cache this request for an hour and tag it
const res = await fetch('https://api.example.com/products', {
  next: { revalidate: 3600, tags: ['products'] },
})

// cache until invalidated
const res2 = await fetch(url, { cache: 'force-cache' })
```

```tsx
// app/blog/page.tsx: segment config
export const revalidate = 600          // seconds; false means cache indefinitely
// export const dynamic = 'force-dynamic'  // render on every request
```

For non-fetch data (ORM calls), the page is static unless it reads request data. Use `revalidate` segment config to refresh it, or `export const dynamic = 'force-dynamic'`.

## Invalidation (both models)

| After a write you want | Call (from `next/cache`) |
|---|---|
| The user who wrote it to see it right away | `updateTag('products')` inside the Server Action |
| Everyone to get fresh data soon, serving stale meanwhile | `revalidateTag('products', 'max')` |
| A specific page re-rendered | `revalidatePath('/shop')` |
| The current page re-rendered without touching caches | `refresh()` inside the Server Action |

`updateTag` and `refresh` work only in Server Actions. `revalidateTag` and `revalidatePath` also work in Route Handlers, for example a CMS webhook.

## Fetching patterns

- Fetch in the Server Component that renders the data. Don't add an internal API route just to call it from your own server code.
- Start independent requests together: `const [a, b] = await Promise.all([getA(), getB()])`.
- Pass a promise to a client component and unwrap it with `use()` when the client part should stream in.
- Wrap a database helper in React's `cache()` to dedupe calls within one render. This is separate from `'use cache'`, which persists across requests.

## Diagnosing stale or surprising data

1. Run `next build` and read the route table. It marks each route as static, dynamic or partially prerendered.
2. Data stale after a write: find the tag or path the write invalidates, and check it matches the one the read uses. A missing `cacheTag` is the most common cause.
3. Page dynamic when you expected static: search the tree for `cookies()`, `headers()`, `searchParams` or `connection()` outside Suspense.
4. Build fails on segment config: you are on Cache Components with section B exports. Replace `revalidate` with `cacheLife`, and remove `dynamic`.

## Checks before you finish

- The model is identified and only that model's APIs are used.
- Every cached read that a write can change has a tag, and that write invalidates it.
- No request data is read inside `'use cache'`.
- Personalised data is never cached in a shared scope.
