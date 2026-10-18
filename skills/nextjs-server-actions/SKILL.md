---
name: nextjs-server-actions
description: Writes Next.js App Router Server Actions (Server Functions) that authenticate, authorize and validate input with a schema before mutating data, then revalidate and redirect correctly. Use when adding a form submit, a mutation, a delete button, or when reviewing an existing 'use server' function for security and error handling.
license: MIT
---

# Next.js Server Actions with validation

A Server Action is a public HTTP endpoint. Anyone can call it with any payload, whether or not your UI shows the form. Write every action as if it were an API route.

## Steps

1. Put actions in a file that starts with `'use server'`, for example `app/posts/actions.ts`. Inline `'use server'` inside a function body works too, but a separate file is easier to review.
2. Inside each action, in this order:
   1. Authenticate. Get the session and fail if there is none.
   2. Authorize. Check that this user may touch this record.
   3. Validate. Parse the input with a schema (Zod or Valibot). Never trust `FormData` types.
   4. Mutate.
   5. Revalidate what changed.
   6. Redirect, if needed, as the last line.
3. Return plain, serializable data that the client may see. Never return the raw record or an error object that contains internals.
4. Wire the form with `useActionState` when you need to show field errors or a pending state.

## Example (Zod 4)

```ts
// app/posts/actions.ts
'use server'

import { z } from 'zod'
import { redirect } from 'next/navigation'
import { revalidatePath } from 'next/cache'
import { auth } from '@/lib/auth'
import { db } from '@/lib/db'

const PostSchema = z.object({
  title: z.string().trim().min(1, 'Title is required').max(200),
  body: z.string().trim().min(1, 'Body is required'),
})

export type PostFormState = {
  errors?: { title?: string[]; body?: string[] }
  message?: string
}

export async function createPost(
  _prev: PostFormState,
  formData: FormData,
): Promise<PostFormState> {
  const session = await auth()
  if (!session?.user) return { message: 'Please sign in.' }

  const parsed = PostSchema.safeParse({
    title: formData.get('title'),
    body: formData.get('body'),
  })
  if (!parsed.success) {
    return { errors: z.flattenError(parsed.error).fieldErrors }
  }

  const post = await db.post.create({
    data: { ...parsed.data, authorId: session.user.id },
  })

  revalidatePath('/posts')
  redirect(`/posts/${post.id}`)
}
```

```tsx
// app/posts/new/post-form.tsx
'use client'

import { useActionState } from 'react'
import { createPost, type PostFormState } from '../actions'

const initial: PostFormState = {}

export function PostForm() {
  const [state, formAction, pending] = useActionState(createPost, initial)
  return (
    <form action={formAction}>
      <label htmlFor="title">Title</label>
      <input id="title" name="title" aria-describedby="title-error" />
      {state.errors?.title && <p id="title-error">{state.errors.title[0]}</p>}
      <label htmlFor="body">Body</label>
      <textarea id="body" name="body" />
      {state.errors?.body && <p>{state.errors.body[0]}</p>}
      {state.message && <p role="alert">{state.message}</p>}
      <button disabled={pending}>{pending ? 'Saving' : 'Save'}</button>
    </form>
  )
}
```

`useActionState` comes from `react`, not `react-dom`. With it, the action receives the previous state first and the `FormData` second.

## Revalidating after a write

| Call | Import | Use it when |
|---|---|---|
| `revalidatePath('/posts')` | `next/cache` | A known path shows the changed data. Pass `'page'` or `'layout'` as the second argument for a dynamic path such as `/posts/[id]` |
| `updateTag('posts')` | `next/cache` | Cached data is tagged and the user must see their own write right away. Server Actions only |
| `revalidateTag('posts', 'max')` | `next/cache` | Stale data may be served while it refreshes in the background. The one-argument form is deprecated in 16 |
| `refresh()` | `next/cache` | Data you never cached needs a fresh render of the current page. Server Actions only |

Call revalidation before `redirect()`. `redirect` throws, so nothing after it runs. Don't wrap `redirect` in a `try` block, or the catch will swallow it.

## Authorization example

```ts
export async function deletePost(postId: string) {
  const session = await auth()
  if (!session?.user) throw new Error('Unauthorized')
  const id = z.uuid().parse(postId)
  const post = await db.post.findUnique({ where: { id } })
  if (!post || post.authorId !== session.user.id) throw new Error('Forbidden')
  await db.post.delete({ where: { id } })
  revalidatePath('/posts')
}
```

Arguments that are bound with `.bind(null, id)` or passed from a client component are user input too. Validate them the same way.

## Checks before you finish

- Each exported function in a `'use server'` file checks auth itself. A check on the page that renders the form does not protect the action.
- Input goes through `safeParse` (for field errors) or `parse` (for a thrown error), never `as` casts.
- The return value contains no secrets, stack traces or full database rows.
- Revalidation matches what the user will look at next.
- `redirect` is outside any `try`/`catch`.
- Expected failures (validation, permission) are returned as state. Unexpected ones throw, so the nearest `error.tsx` handles them.

## Notes

- Zod 4 deprecates `error.flatten()` and drops `invalid_type_error`. Older guides show those, so use `z.flattenError(error)` and the `error` option instead.
- The client currently dispatches actions one at a time, so they are a poor fit for reads. Fetch in a Server Component or a Route Handler instead.
