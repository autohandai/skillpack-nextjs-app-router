---
name: nextjs-routes-layouts
description: Adds or restructures routes in a Next.js App Router project (Next.js 16), including pages, layouts, dynamic segments, route groups, loading and error boundaries, and not-found pages. Use when creating a new page or section under app/, moving routes, adding a shared layout, or fixing params typing errors.
license: MIT
---

# Next.js routes and layouts

Use this for work under `app/` in a Next.js 16 App Router project. Read `package.json` before you start. If `next` is older than 16, check that version's docs, because `params` typing and some file names differ.

## Steps

1. Find the root. It is `app/` or `src/app/`. Never create a second one.
2. Map the URL to folders. Each folder is a segment, and a segment only becomes a route once it has a `page.tsx` (or a `route.ts` for an HTTP handler). A folder can hold one or the other, never both.
3. Pick the folder pattern from the table below.
4. Add the special files the route needs, and nothing more.
5. Type `params` and `searchParams` as Promises and await them.
6. Run `next build` (or `next typegen`) so the generated route types catch typos in paths.

| You need | Folder pattern | Example |
|---|---|---|
| One dynamic value | `[slug]` | `app/blog/[slug]/page.tsx` serves `/blog/hello` |
| Any depth | `[...slug]` | `app/docs/[...slug]/page.tsx` serves `/docs/a/b` |
| Any depth, including none | `[[...slug]]` | `app/docs/[[...slug]]/page.tsx` also serves `/docs` |
| Shared layout, no URL change | `(group)` | `app/(marketing)/about/page.tsx` serves `/about` |
| Colocated files that are not routes | `_folder` | `app/blog/_components/Card.tsx` |
| Two views in one layout | `@slot` | `app/dashboard/@stats/page.tsx`, passed to the layout as `stats` |

## Special files

| File | Job | Notes |
|---|---|---|
| `layout.tsx` | Wraps child routes and keeps its state across navigation | Root layout must render `<html>` and `<body>` |
| `page.tsx` | The UI for the route | A Server Component unless it starts with `'use client'` |
| `loading.tsx` | Instant fallback while the segment streams | Wraps the page in Suspense for you |
| `error.tsx` | Error boundary for the segment | Must start with `'use client'` |
| `not-found.tsx` | UI shown when `notFound()` is called | Import `notFound` from `next/navigation` |
| `template.tsx` | Like a layout, but remounts on every navigation | Use only when state must reset |
| `default.tsx` | Fallback for a parallel `@slot` on hard navigation | Needed for each slot that can be unmatched |

## Typed params (Next.js 16)

`params` and `searchParams` are Promises. Await them in an async Server Component. The `PageProps` and `LayoutProps` helpers are global and generated from your routes, so no import is needed.

```tsx
// app/blog/[slug]/page.tsx
import { notFound } from 'next/navigation'
import { getPost } from '@/lib/posts'

export default async function Page(props: PageProps<'/blog/[slug]'>) {
  const { slug } = await props.params
  const post = await getPost(slug)
  if (!post) notFound()
  return <article><h1>{post.title}</h1></article>
}
```

```tsx
// app/dashboard/layout.tsx with an @stats slot
export default function Layout(props: LayoutProps<'/dashboard'>) {
  return (
    <section>
      {props.children}
      {props.stats}
    </section>
  )
}
```

In a Client Component, read params with `useParams()` from `next/navigation`, or pass `params` down and unwrap it with React's `use()`.

## Error boundary

```tsx
// app/blog/error.tsx
'use client'

export default function Error({ error, retry }: { error: Error & { digest?: string }; retry: () => void }) {
  return (
    <div>
      <p>Something went wrong loading this page.</p>
      <button onClick={() => retry()}>Try again</button>
    </div>
  )
}
```

The `retry` prop is stable from 16.3. On 16.2 it was `unstable_retry`, and on older versions use `reset`. Errors thrown in Server Components arrive with a generic message and a `digest` that matches the server log, so don't show `error.message` to users and expect detail.

## Checks before you finish

- Each new route folder has exactly one `page.tsx` or `route.ts`.
- No `'use client'` on a layout or page unless it needs state, effects or browser APIs. Push the client boundary down to the smallest component.
- Every `params` and `searchParams` access is awaited.
- A route group doesn't produce two pages for the same URL. `(a)/about` and `(b)/about` will fail the build.
- Each `@slot` has a `default.tsx` if it can be unmatched.
- `proxy.ts` (formerly `middleware.ts`, renamed in 16) is not used for auth on its own. Check auth again where the data is read.

## Common mistakes

- Putting a `page.tsx` inside `_components`. Private folders never route.
- Fetching the same data in a layout and a page and passing it through context. Fetch it in both places instead. `fetch` calls with the same URL and options are memoized within one render, and you can wrap a database helper in React's `cache()` to get the same effect.
- Expecting a layout to re-render on navigation between its children. It doesn't. Use `template.tsx` or move the logic into the page.
