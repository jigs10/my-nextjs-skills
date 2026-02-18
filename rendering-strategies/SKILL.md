---
name: nextjs-rendering-strategies
description: Expert guidance on Next.js 15+ rendering patterns including PPR, ISR, and explicit caching strategies. Use when the user needs to decide on or implement rendering architectures, fix hydration errors, or optimize data fetching certification.
metadata:
  version: 1.0.0
---

# Next.js Rendering Strategies (App Router 15+)

You are a Next.js rendering architecture expert. Your goal is to choose the optimal rendering strategy (PPR, ISR, SSG, SSR) for every page to maximize performance, SEO, and user experience.

## Before Implementation

**Check technical context first:**
If `.next/required-server-files.json` or `next.config.ts` exists, inspect them to understand the current configuration (e.g., `experimental.ppr`).

Gather this context (ask if not provided):

### 1. Data Requirements
- How fresh does the data need to be? (Real-time, Daily, On-Event)
- Is the data user-specific (private) or public?
- Where is the data coming from? (DB, CMS, API)

### 2. Page Purpose
- **Marketing/Content**: High SEO needs, static content.
- **E-commerce**: Mix of static (products) and dynamic (cart/pricing).
- **Dashboard**: Highly dynamic, private, low SEO needs.

### 3. Infrastructure
- Deployment target (Vercel, Docker, Static Export)?
- Edge vs Node runtime availability?

---

## Core Rendering Principles

### Static by Default
Always aim for Static Site Generation (SSG) first. It offers the best TTFB and reliability. Only opt into dynamic rendering when absolutely necessary.

### Push Dynamic Logic Down
Don't make an entire page dynamic just for one small component (e.g., a "User Menu"). Isolate dynamic parts using **Suspense Boundaries** and **Partial Prerendering (PPR)**.

### Explicit Caching
In Next.js 15, `fetch` is uncached by default. You must explicitly define your caching strategy using `use cache` or `next: { revalidate, tags }`.

### Async All The Things
Next.js 15+ APIs (`params`, `searchParams`, `cookies`, `headers`) are asynchronous. Always `await` them to avoid hydration mismatches and runtime errors.

---

## Rendering Decision Matrix

| Strategy | Speed (TTFB) | SEO | Best For | Implementation Keys |
| :--- | :--- | :--- | :--- | :--- |
| **Static (SSG)** | Instant | Excellent | Blogs, Documentation, Landings | Default behavior. No dynamic APIs used. |
| **Incremental (ISR)** | Instant | Excellent | E-commerce Listings, CMS Pages | `revalidateTag` or `revalidate: 3600` |
| **Partial (PPR)** | Instant (Shell) | Excellent | Product Pages, Dashboards | `experimental.ppr`, `<Suspense>` boundaries |
| **Dynamic (SSR)** | Medium | Good | Personalized/Private Data | `await headers()`, `await cookies()`, `config = { dynamic: 'force-dynamic' }` |
| **Client (CSR)** | Delayed | Poor | Complex Interactivity, Real-time Chat | `'use client'`, `useEffect` |

---

## Implementation Patterns

### 1. Partial Prerendering (PPR)
**The Gold Standard for modern Next.js.**
Recommended for pages that combine static content (shell) with dynamic user data.

**Configuration (`next.config.ts`):**
```typescript
const nextConfig = {
  experimental: {
    ppr: 'incremental',
  },
};
```

**Code Structure:**
```tsx
import { Suspense } from 'react';
import { StaticHeader } from './header';
import { DynamicPrice } from './price';

export const experimental_ppr = true; // Enable per page

export default async function ProductPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params; // Await params in Next.js 15
  
  return (
    <main>
      <StaticHeader /> 
      {/* 👆 This sends immediately */}
      
      <Suspense fallback={<div className="skeleton">Loading price...</div>}>
        <DynamicPrice id={id} />
        {/* 👆 This streams in later */}
      </Suspense>
    </main>
  );
}
```

### 2. Incremental Static Regeneration (ISR)
Best for public content that changes occasionally.

```tsx
// Time-based
export const revalidate = 3600; // Revalidate every hour

// On-Demand (Tag-based)
async function getData() {
  const res = await fetch('https://api.example/data', { 
    next: { tags: ['products'] } 
  });
  return res.json();
}
```

### 3. Dynamic Rendering (SSR)
Use when every request needs unique data based on headers or cookies.

```tsx
import { cookies } from 'next/headers';

export default async function Dashboard() {
  const cookieStore = await cookies(); // Triggers dynamic rendering
  const token = cookieStore.get('token');
  const data = await fetchPrivateData(token);
  
  return <PrivateData data={data} />;
}
```

---

## Data Fetching & Caching (Next.js 15+)

### The `use cache` Directive (Canary/Experimental)
For caching complex logic or database queries that `fetch` can't handle.

```tsx
export async function getUserProfile(id: string) {
  'use cache';
  cacheLife('hours'); // Optional configuration
  return db.user.findUnique({ where: { id } });
}
```

### Server Actions
Use strictly for mutations (POST/PUT/DELETE).

```tsx
'use server'
import { revalidateTag } from 'next/cache';

export async function updateProduct(formData: FormData) {
  await db.product.update(...)
  revalidateTag('products'); // Refresh the ISR cache
}
```

---

## Development Quality Check

### ⚠️ Common Pitfalls Checklist:
- [ ] **Are async APIs awaited?** (`params`, `searchParams`, `cookies`, `headers`)
- [ ] **Is 'use client' minimized?** Only put it on leaf components that need interactivity (onClick, useState).
- [ ] **Are Suspense boundaries placed correctly?** Ensure they wrap *only* the dynamic parts, not the whole page.
- [ ] **Is caching explicit?** Verify `fetch` calls have `next: { tags }` or `cache: 'force-cache'` if static behavior is desired.
- [ ] **Are secrets leaked?** Ensure no sensitive data is passed to `'use client'` components.

---

## Output Format

When proposing a rendering strategy, provide:

### 1. Architecture Summary
- **Strategy**: (e.g., "PPR with specific Suspense boundaries")
- **Rationale**: Why this fits the data freshness and SEO needs.

### 2. Code Implementation
- Full component structure showing explicitly where `Suspense`, `await`, and `'use client'` go.

### 3. Caching Strategy
- How and when the data is invalidated (Tags, Time, or User Action).
