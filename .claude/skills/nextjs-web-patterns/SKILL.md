---
name: nextjs-web-patterns
description: Best practices for the web admin dashboard in shopoo (Next.js App Router + TypeScript). Use when building or reviewing the web/ repo — covers Server vs Client Components, data fetching (server fetch + React Query on the client), route handlers/server actions proxying to the Admin Service, admin auth with httpOnly cookies + middleware, forms with react-hook-form + zod, env handling, and loading/error boundaries. Pair with marketplace-conventions; pair with design-taste-frontend for visual quality.
---

# Next.js Web Admin Patterns

> `web/` is the admin dashboard (moderation, users, reports, analytics). It talks to **Admin Service** over REST. Admin-only — security and clean data tables matter more than marketing polish.

## App Router model
- **Server Components by default.** Add `'use client'` only for interactivity (forms, charts, React Query, event handlers). Push client boundaries down the tree — keep pages as server components that pass data in.
- Use `app/` segments with `layout.tsx`, `loading.tsx`, `error.tsx`, `not-found.tsx`. Group routes: `app/(auth)/login`, `app/(dashboard)/...`.

## Data fetching
- **Reads:** server components fetch directly (with the admin's token from the cookie) and stream with Suspense. For interactive/refetching tables, use **TanStack Query** in client components hitting Next **route handlers** that proxy to Admin Service (keeps the API base + token server-side).
- **Mutations:** Server Actions or route handlers; invalidate React Query keys / `revalidatePath` after.
- Centralize the API client (base URL from server env, inject `Authorization`). Never call the backend directly from the browser with a long-lived token.

## Auth
- Login posts to Admin Service, store the access token in an **httpOnly, Secure, SameSite cookie** (not localStorage). A `middleware.ts` guards `(dashboard)` routes — redirect to `/login` if missing/expired, and enforce `role === 'admin'`.
- Refresh via a route handler; never expose the refresh token to client JS.

## Forms & UI
- **react-hook-form + zod** (`zodResolver`) for every form; show server validation errors inline. Optimistic updates only where safe.
- UI kit: **shadcn/ui** + Tailwind; charts with a self-contained lib. Reusable `DataTable`, `StatCard`, `ConfirmDialog`. For any visual-quality work, **invoke the `design-taste-frontend` skill** — but this is a dashboard, so favour clarity, density, and scannability over hero/marketing aesthetics.

## Env & config
- `NEXT_PUBLIC_*` only for values safe in the browser. API base URL + secrets stay server-side (plain env, read in server components/route handlers). Commit `.env.example`.

## Performance & quality
- Use `next/image`, `next/font`. Avoid client components that re-fetch what the server already has. Type everything; share request/response types with the backend contract where practical.

## Don't
Don't put the API token or backend base URL in client bundles · don't `'use client'` an entire page to make one button work · don't fetch in the client when a server component can · don't skip `loading`/`error` files on async routes.
