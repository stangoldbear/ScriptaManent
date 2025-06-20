# Next.js Basics for TypeScript/JavaScript Developers 🚀

## What is Next.js?
Next.js is a React framework that provides a robust foundation for building web applications. If you know React, think of Next.js as React with superpowers! It adds features like:
- Server-side rendering (SSR)
- Static site generation (SSG)
- API routes
- File-based routing
- Built-in optimizations

## Project Structure

Our project follows the modern Next.js 13+ structure with the App Router:

```
src/
├── app/                    # App Router directory
│   ├── (main)/            # Main route group
│   │   └── page.tsx       # Home page (/)
│   ├── (external)/        # External route group
│   │   ├── login/         # Login routes
│   │   └── register/      # Registration routes
│   ├── api/               # API routes
│   ├── layout.tsx         # Root layout (in src/app)
│   └── globals.css        # Global styles
```

Note: This is our actual project structure. While Next.js supports different directory organizations, we've chosen this structure for better code organization and maintainability.

## Key Concepts

### 1. App Router and Pages

The App Router uses a file-system based routing where:
- `page.tsx` files define routes
- `layout.tsx` files define shared layouts
- `loading.tsx` files define loading states
- `error.tsx` files define error boundaries

> **Note on Root URL Behavior**: When accessing the root URL (`http://localhost:3000`), our application implements a two-step redirect chain:
> 1. The root page (`src/app/(external)/page.tsx`) performs a redirect to `/dashboard`
> 2. The Next.js configuration (`next.config.mjs`) then redirects `/dashboard` to `/dashboard/default`
> This creates a seamless user experience by automatically directing users to our default dashboard view.

> **Route Groups**: Our project uses route groups (denoted by parentheses) to organize routes. While Next.js allows any name for route groups (as long as they're wrapped in parentheses), we follow a specific convention:
> - `(external)`: Contains routes that are accessible without authentication, such as the root page (`/`), login, and registration pages. These pages share a common layout optimized for unauthenticated users.
> - `(main)`: Contains authenticated routes that require user login, such as dashboard pages and user settings. These pages share a layout with navigation and user-specific features.
> 
> Route groups are a Next.js feature that:
> - Don't affect the URL structure (the parentheses are excluded from the URL)
> - Allow sharing layouts between specific routes
> - Help organize code by grouping related routes
> - Can be used to split your application into different sections
> 
> While `(external)` and `(main)` are our project's convention, you can use any meaningful name for route groups. For example, you might see other projects use `(auth)`, `(dashboard)`, or `(marketing)` depending on their needs.
> 
> For more details, see the [Next.js Route Groups documentation](https://nextjs.org/docs/app/building-your-application/routing/route-groups).

Here's our actual project structure:
```
src/app/
├── (main)/            # Main route group
│   └── page.tsx       # Home page (/)
├── (external)/        # External route group
│   ├── login/         # Login routes
│   └── register/      # Registration routes
├── api/               # API routes
├── layout.tsx         # Root layout
└── globals.css        # Global styles
```

Below are some generic examples of how Next.js routing works (these are not part of our current project):

```typescript
// Example 1: Basic page
// This is a generic example, not part of our project
// src/app/(main)/about/page.tsx
export default function AboutPage() {
  return <h1>About Us</h1>
}

// Example 2: Dynamic route
// This is a generic example, not part of our project
// src/app/(main)/blog/[slug]/page.tsx
export default function BlogPost({ params }: { params: { slug: string } }) {
  return <article>Blog post: {params.slug}</article>
}

// Example 3: Loading state
// This is a generic example, not part of our project
// src/app/(main)/blog/loading.tsx
export default function BlogLoading() {
  return <div>Loading blog posts...</div>
}

// Example 4: Error boundary
// This is a generic example, not part of our project
// src/app/(main)/blog/error.tsx
'use client'

export default function BlogError({
  error,
  reset,
}: {
  error: Error
  reset: () => void
}) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={reset}>Try again</button>
    </div>
  )
}
```

### 2. Page Types

Next.js supports different types of pages:

1. **Static Pages** (example of how we would implement it):
```typescript
// Note: This is an example of how we would implement a home page
// Currently, we don't have a page.tsx in (main) route group
// When we implement it, it would be at: src/app/(main)/page.tsx
export default function HomePage() {
  return (
    <main>
      <h1>Welcome to Our App</h1>
    </main>
  )
}
```

2. **Dynamic Pages** (generic example, not part of our project):
```typescript
// This is a generic example showing dynamic routes
// Not part of our current project
// src/app/(main)/blog/[slug]/page.tsx
export async function generateStaticParams() {
  return [
    { slug: 'post-1' },
    { slug: 'post-2' },
  ]
}

export default function BlogPost({ params }: { params: { slug: string } }) {
  return <article>Blog post: {params.slug}</article>
}
```

3. **Server Components** (example of how we would implement it):
```typescript
// Note: This is an example of how we would implement a server component
// Currently, we don't have a page.tsx in (main) route group
// When we implement it, it would be at: src/app/(main)/page.tsx
import { getData } from '@/lib/data'

export default async function HomePage() {
  const data = await getData()
  return (
    <main>
      <h1>{data.title}</h1>
    </main>
  )
}
```

4. **Client Components** (generic example, not part of our project):
```typescript
// This is a generic example showing client components
// Not part of our current project
// src/app/(main)/blog/page.tsx
'use client'

import { useState } from 'react'

export default function BlogPage() {
  const [posts, setPosts] = useState([])
  // ... rest of the component
}
```

### 3. Data Fetching 🔄

In our project, we use a combination of data fetching methods:

1. **Server Components** (Default in App Router)
   ```typescript
   // src/app/(main)/blog/page.tsx
   async function getPosts() {
     const res = await fetch('https://api.example.com/posts', {
       next: { revalidate: 3600 } // Revalidate every hour
     })
     return res.json()
   }
   
   export default async function BlogPage() {
     const posts = await getPosts()
     return <PostList posts={posts} />
   }
   ```

2. **Client Components** (When needed)
   ```typescript
   // src/components/PostList.tsx
   'use client'
   
   import { useQuery } from '@tanstack/react-query'
   
   export function PostList() {
     const { data: posts } = useQuery({
       queryKey: ['posts'],
       queryFn: () => fetch('/api/posts').then(res => res.json())
     })
     
     return <div>{/* Render posts */}</div>
   }
   ```

### 4. API Routes 🛣️

In our project, we use the new App Router API routes:

```typescript
// src/app/(external)/api/posts/route.ts
import { NextResponse } from 'next/server'
import { db } from '@/lib/db'

export async function GET() {
  const posts = await db.posts.findMany()
  return NextResponse.json(posts)
}

export async function POST(request: Request) {
  const data = await request.json()
  const post = await db.posts.create({ data })
  return NextResponse.json(post)
}
```

### 5. Built-in Components 🧩

We use Next.js built-in components extensively:

```typescript
// src/components/ProductCard.tsx
import Image from 'next/image'
import Link from 'next/link'

export function ProductCard({ product }: { product: Product }) {
  return (
    <Link href={`/products/${product.slug}`}>
      <div className="rounded-lg overflow-hidden">
        <Image
          src={product.image}
          alt={product.name}
          width={400}
          height={300}
          className="object-cover"
        />
        <h2>{product.name}</h2>
      </div>
    </Link>
  )
}
```

### 6. Styling 🎨

Our project uses a combination of styling approaches:

1. **Tailwind CSS** (Primary styling method)
   ```typescript
   // src/components/Button.tsx
   export function Button({ children }: { children: React.ReactNode }) {
     return (
       <button className="px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600">
         {children}
       </button>
     )
   }
   ```

2. **CSS Modules** (For complex components)
   ```typescript
   // src/components/ComplexComponent.module.css
   .container { /* styles */ }
   
   // src/components/ComplexComponent.tsx
   import styles from './ComplexComponent.module.css'
   export function ComplexComponent() {
     return <div className={styles.container}>Content</div>
   }
   ```

3. **Global Styles**
   ```typescript
   // src/app/globals.css
   @tailwind base;
   @tailwind components;
   @tailwind utilities;
   
   /* Custom global styles */
   ```

## Development vs Production

Our project uses specific scripts defined in `package.json`:

- **Development**: `npm run dev` - Starts the development server with hot reloading
- **Production Build**: `npm run build` - Creates an optimized production build
- **Production Start**: `npm start` - Runs the production build
- **Linting**: `npm run lint` - Runs ESLint for code quality
- **Type Checking**: `npm run type-check` - Runs TypeScript compiler

## Key Differences from Plain React

1. **App Router**: We use the new App Router instead of Pages Router
2. **Server Components**: Default to server components for better performance
3. **API Routes**: Modern API routes with the App Router
4. **Styling**: Tailwind CSS as our primary styling solution
5. **TypeScript**: Strict TypeScript configuration for better type safety

## Next Steps

1. Check out the [React in Next.js](./02-react-in-nextjs.md) guide to understand how React concepts apply in our Next.js project
2. Look at [Source Code Structure](./04-source-structure.md) to see how these concepts are implemented in our project
3. Review [Component Guidelines](./10-component-guidelines.md) for our specific component patterns
4. Read [Performance Guidelines](./13-performance-guidelines.md) for optimization strategies

Remember: While Next.js adds powerful features, all your React knowledge still applies! We just use Next.js's conventions and features to build better applications. 