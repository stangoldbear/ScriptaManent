# Performance Guidelines ⚡

This guide outlines our performance optimization strategies and best practices for building fast and efficient Next.js applications.

## Core Web Vitals

### 1. Loading Performance 🚀

```typescript
// src/app/layout.tsx
import { Metadata } from 'next'
import { Inter } from 'next/font/google'
import { Suspense } from 'react'
import { LoadingSpinner } from '@/components/ui/loading'

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-inter',
})

export const metadata: Metadata = {
  title: {
    template: '%s | My App',
    default: 'My App',
  },
  description: 'A high-performance Next.js application',
  viewport: 'width=device-width, initial-scale=1',
  icons: {
    icon: '/favicon.ico',
    apple: '/apple-touch-icon.png',
  },
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en" className={inter.variable}>
      <head>
        <link
          rel="preconnect"
          href="https://fonts.googleapis.com"
          crossOrigin="anonymous"
        />
        <link
          rel="preconnect"
          href="https://fonts.gstatic.com"
          crossOrigin="anonymous"
        />
      </head>
      <body>
        <Suspense fallback={<LoadingSpinner />}>
          <main>{children}</main>
        </Suspense>
      </body>
    </html>
  )
}
```

### 2. Image Optimization 🖼️

```typescript
// components/OptimizedImage.tsx
import Image from 'next/image'
import { useState } from 'react'

interface OptimizedImageProps {
  src: string
  alt: string
  width: number
  height: number
  priority?: boolean
  className?: string
}

export function OptimizedImage({
  src,
  alt,
  width,
  height,
  priority,
  className,
}: OptimizedImageProps) {
  const [isLoading, setIsLoading] = useState(true)

  return (
    <div className={cn('relative overflow-hidden', className)}>
      <Image
        src={src}
        alt={alt}
        width={width}
        height={height}
        priority={priority}
        quality={75}
        loading={priority ? 'eager' : 'lazy'}
        onLoadingComplete={() => setIsLoading(false)}
        className={cn(
          'duration-700 ease-in-out',
          isLoading
            ? 'scale-110 blur-2xl grayscale'
            : 'scale-100 blur-0 grayscale-0'
        )}
      />
      {isLoading && (
        <div className="absolute inset-0 bg-gray-200 animate-pulse" />
      )}
    </div>
  )
}

// Usage
function ProductCard({ product }: { product: Product }) {
  return (
    <div className="rounded-lg overflow-hidden">
      <OptimizedImage
        src={product.image}
        alt={product.name}
        width={400}
        height={300}
        priority={product.featured}
        className="w-full h-48 object-cover"
      />
      {/* Product details */}
    </div>
  )
}
```

### 3. Font Optimization 📝

```typescript
// lib/fonts.ts
import { Inter, Roboto_Mono } from 'next/font/google'
import localFont from 'next/font/local'

export const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-inter',
  preload: true,
  fallback: ['system-ui', 'arial'],
})

export const robotoMono = Roboto_Mono({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-roboto-mono',
  preload: true,
})

export const customFont = localFont({
  src: [
    {
      path: '../fonts/custom-regular.woff2',
      weight: '400',
      style: 'normal',
    },
    {
      path: '../fonts/custom-bold.woff2',
      weight: '700',
      style: 'normal',
    },
  ],
  variable: '--font-custom',
  display: 'swap',
  preload: true,
})

// Usage in layout
export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html
      lang="en"
      className={cn(
        inter.variable,
        robotoMono.variable,
        customFont.variable
      )}
    >
      <body className={inter.className}>{children}</body>
    </html>
  )
}
```

## Code Optimization

### 1. Code Splitting 📦

```typescript
// components/Dashboard.tsx
import dynamic from 'next/dynamic'
import { Suspense } from 'react'
import { LoadingSpinner } from '@/components/ui/LoadingSpinner'

// Dynamically import heavy components
const UserProfile = dynamic(() => import('./UserProfile'), {
  loading: () => <LoadingSpinner />,
  ssr: false, // Disable SSR for client-only components
})

const UserSettings = dynamic(() => import('./UserSettings'), {
  loading: () => <LoadingSpinner />,
})

const UserAnalytics = dynamic(() => import('./UserAnalytics'), {
  loading: () => <LoadingSpinner />,
})

export function Dashboard() {
  return (
    <div className="space-y-4">
      <Suspense fallback={<LoadingSpinner />}>
        <UserProfile />
      </Suspense>
      <Suspense fallback={<LoadingSpinner />}>
        <UserSettings />
      </Suspense>
      <Suspense fallback={<LoadingSpinner />}>
        <UserAnalytics />
      </Suspense>
    </div>
  )
}
```

### 2. Component Memoization 🧠

```typescript
// components/UserList.tsx
import { memo, useMemo } from 'react'

interface User {
  id: string
  name: string
  email: string
  role: string
}

// Memoize individual user card
const UserCard = memo(function UserCard({ user }: { user: User }) {
  return (
    <div className="rounded-lg border p-4">
      <h3 className="font-semibold">{user.name}</h3>
      <p className="text-gray-600">{user.email}</p>
      <span className="text-sm text-blue-600">{user.role}</span>
    </div>
  )
})

// Memoize filtered and sorted users
export function UserList({ users, filter, sortBy }: UserListProps) {
  const processedUsers = useMemo(() => {
    return users
      .filter(user => user.role === filter)
      .sort((a, b) => a[sortBy].localeCompare(b[sortBy]))
  }, [users, filter, sortBy])

  return (
    <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
      {processedUsers.map(user => (
        <UserCard key={user.id} user={user} />
      ))}
    </div>
  )
}
```

### 3. Hook Optimization 🎣

```typescript
// hooks/useOptimizedQuery.ts
import { useCallback, useMemo, useState } from 'react'
import { useQuery, useQueryClient } from '@tanstack/react-query'

interface UseOptimizedQueryOptions<T> {
  queryKey: string[]
  queryFn: () => Promise<T>
  staleTime?: number
  cacheTime?: number
  enabled?: boolean
}

export function useOptimizedQuery<T>({
  queryKey,
  queryFn,
  staleTime = 5 * 60 * 1000, // 5 minutes
  cacheTime = 30 * 60 * 1000, // 30 minutes
  enabled = true,
}: UseOptimizedQueryOptions<T>) {
  const queryClient = useQueryClient()
  const [isPrefetching, setIsPrefetching] = useState(false)

  // Memoize query function
  const memoizedQueryFn = useCallback(queryFn, [queryKey])

  // Optimistic updates
  const optimisticUpdate = useCallback(
    (updater: (oldData: T) => T) => {
      queryClient.setQueryData(queryKey, updater)
    },
    [queryClient, queryKey]
  )

  // Prefetch related data
  const prefetch = useCallback(
    async (relatedKey: string[]) => {
      setIsPrefetching(true)
      try {
        await queryClient.prefetchQuery(relatedKey, queryFn)
      } finally {
        setIsPrefetching(false)
      }
    },
    [queryClient, queryFn]
  )

  const query = useQuery({
    queryKey,
    queryFn: memoizedQueryFn,
    staleTime,
    cacheTime,
    enabled,
  })

  return {
    ...query,
    isPrefetching,
    optimisticUpdate,
    prefetch,
  }
}

// Usage
function UserProfile({ userId }: { userId: string }) {
  const { data: user, isLoading, prefetch } = useOptimizedQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  })

  // Prefetch related data on hover
  const handleHover = () => {
    prefetch(['user-posts', userId])
  }

  if (isLoading) return <LoadingSpinner />
  if (!user) return null

  return (
    <div onMouseEnter={handleHover}>
      <h2>{user.name}</h2>
      {/* User details */}
    </div>
  )
}
```

## Data Fetching

### 1. Server Components 🖥️

```typescript
// app/users/page.tsx
import { Suspense } from 'react'
import { UserList } from '@/components/UserList'
import { UserStats } from '@/components/UserStats'
import { db } from '@/lib/db'

async function getUsers() {
  const users = await db.users.findMany({
    select: {
      id: true,
      name: true,
      email: true,
      role: true,
      _count: {
        select: {
          posts: true,
          comments: true,
        },
      },
    },
  })
  return users
}

export default async function UsersPage() {
  const users = await getUsers()

  return (
    <div className="space-y-6">
      <Suspense fallback={<UserStatsSkeleton />}>
        <UserStats users={users} />
      </Suspense>
      <Suspense fallback={<UserListSkeleton />}>
        <UserList users={users} />
      </Suspense>
    </div>
  )
}
```

### 2. Data Caching 💾

```typescript
// lib/cache.ts
import { Redis } from '@upstash/redis'
import { unstable_cache } from 'next/cache'

const redis = new Redis({
  url: process.env.REDIS_URL!,
  token: process.env.REDIS_TOKEN!,
})

// Server-side caching
export const getCachedData = unstable_cache(
  async (key: string) => {
    const data = await fetchData(key)
    return data
  },
  ['cached-data'],
  {
    revalidate: 60, // Revalidate every minute
    tags: ['data'], // Cache tags for invalidation
  }
)

// Redis caching
export async function getRedisCachedData<T>(
  key: string,
  fetchFn: () => Promise<T>,
  ttl: number = 3600 // 1 hour
): Promise<T> {
  const cached = await redis.get<T>(key)
  if (cached) return cached

  const data = await fetchFn()
  await redis.set(key, data, { ex: ttl })
  return data
}

// Usage
async function getProductDetails(productId: string) {
  return getRedisCachedData(
    `product:${productId}`,
    () => db.products.findUnique({ where: { id: productId } }),
    60 * 5 // 5 minutes
  )
}
```

### 3. Infinite Loading 📜

```typescript
// components/InfiniteScroll.tsx
import { useInfiniteQuery } from '@tanstack/react-query'
import { useInView } from 'react-intersection-observer'
import { useEffect } from 'react'

interface InfiniteScrollProps<T> {
  queryKey: string[]
  fetchFn: (page: number) => Promise<T[]>
  renderItem: (item: T) => React.ReactNode
  getNextPageParam: (lastPage: T[]) => number | undefined
}

export function InfiniteScroll<T>({
  queryKey,
  fetchFn,
  renderItem,
  getNextPageParam,
}: InfiniteScrollProps<T>) {
  const { ref, inView } = useInView()

  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    status,
  } = useInfiniteQuery({
    queryKey,
    queryFn: ({ pageParam = 1 }) => fetchFn(pageParam),
    getNextPageParam,
    keepPreviousData: true,
  })

  useEffect(() => {
    if (inView && hasNextPage && !isFetchingNextPage) {
      fetchNextPage()
    }
  }, [inView, hasNextPage, isFetchingNextPage, fetchNextPage])

  if (status === 'loading') return <LoadingSpinner />
  if (status === 'error') return <ErrorMessage />

  return (
    <div className="space-y-4">
      {data.pages.map((page, i) => (
        <div key={i} className="space-y-4">
          {page.map(item => renderItem(item))}
        </div>
      ))}
      <div ref={ref} className="h-10">
        {isFetchingNextPage && <LoadingSpinner />}
      </div>
    </div>
  )
}

// Usage
function PostList() {
  return (
    <InfiniteScroll
      queryKey={['posts']}
      fetchFn={page => fetchPosts(page)}
      getNextPageParam={lastPage =>
        lastPage.length === 10 ? lastPage.length + 1 : undefined
      }
      renderItem={post => <PostCard key={post.id} post={post} />}
    />
  )
}
```

## Build Optimization

### 1. Bundle Analysis 📊

```json
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
})

module.exports = withBundleAnalyzer({
  // Next.js config
  webpack: (config, { dev, isServer }) => {
    // Optimize bundle size
    if (!dev && !isServer) {
      config.optimization.splitChunks = {
        chunks: 'all',
        minSize: 20000,
        maxSize: 244000,
        minChunks: 1,
        maxAsyncRequests: 30,
        maxInitialRequests: 30,
        cacheGroups: {
          defaultVendors: {
            test: /[\\/]node_modules[\\/]/,
            priority: -10,
            reuseExistingChunk: true,
          },
          default: {
            minChunks: 2,
            priority: -20,
            reuseExistingChunk: true,
          },
        },
      }
    }
    return config
  },
})
```

### 2. Tree Shaking 🌳

```typescript
// lib/utils.ts
// Export individual functions for better tree shaking
export function formatDate(date: Date): string {
  return date.toLocaleDateString()
}

export function formatCurrency(amount: number): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
  }).format(amount)
}

// Usage
import { formatDate } from '@/lib/utils'
// Only formatDate will be included in the bundle
```

### 3. Dynamic Imports 📦

```typescript
// components/HeavyComponent.tsx
import dynamic from 'next/dynamic'

// Dynamically import heavy dependencies
const Chart = dynamic(() => import('@/components/Chart'), {
  loading: () => <ChartSkeleton />,
  ssr: false,
})

const Map = dynamic(() => import('@/components/Map'), {
  loading: () => <MapSkeleton />,
  ssr: false,
})

// Dynamically import based on props
const DynamicComponent = dynamic(
  () =>
    import('@/components/DynamicComponent').then(mod => {
      if (process.env.NODE_ENV === 'development') {
        console.log('Dynamic component loaded')
      }
      return mod
    }),
  {
    loading: () => <LoadingSpinner />,
    ssr: false,
  }
)

export function Dashboard() {
  return (
    <div className="grid gap-4 md:grid-cols-2">
      <Chart />
      <Map />
      <DynamicComponent />
    </div>
  )
}
```

## Monitoring

### 1. Performance Monitoring 📈

```typescript
// lib/performance.ts
import { onCLS, onFID, onLCP } from 'web-vitals'

export function reportWebVitals(metric: any) {
  // Send to analytics
  console.log(metric)
}

export function initPerformanceMonitoring() {
  // Monitor Core Web Vitals
  onCLS(metric => reportWebVitals(metric))
  onFID(metric => reportWebVitals(metric))
  onLCP(metric => reportWebVitals(metric))

  // Custom performance marks
  performance.mark('app-start')
  window.addEventListener('load', () => {
    performance.mark('app-loaded')
    performance.measure(
      'app-load-time',
      'app-start',
      'app-loaded'
    )
  })
}

// Usage in app
export default function App({ Component, pageProps }: AppProps) {
  useEffect(() => {
    initPerformanceMonitoring()
  }, [])

  return <Component {...pageProps} />
}
```

### 2. Error Tracking 🚨

```typescript
// lib/error-tracking.ts
import * as Sentry from '@sentry/nextjs'

export function initErrorTracking() {
  Sentry.init({
    dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
    tracesSampleRate: 1.0,
    environment: process.env.NODE_ENV,
    integrations: [
      new Sentry.BrowserTracing({
        tracePropagationTargets: ['localhost', process.env.NEXT_PUBLIC_APP_URL!],
      }),
    ],
  })
}

export function trackError(error: Error, context?: Record<string, any>) {
  Sentry.withScope(scope => {
    if (context) {
      Object.entries(context).forEach(([key, value]) => {
        scope.setExtra(key, value)
      })
    }
    Sentry.captureException(error)
  })
}

// Usage in error boundary
class ErrorBoundary extends React.Component {
  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    trackError(error, {
      componentStack: errorInfo.componentStack,
    })
  }

  render() {
    return this.props.children
  }
}
```

## Next Steps

1. Review the [Code Style Guide](./09-code-style-guide.md) for performance coding standards
2. Check [Development Workflow](./06-development-workflow.md) for performance review process
3. Look at [Testing Strategy](./07-testing-strategy.md) for performance testing patterns 

## Server Components

### Our Implementation
```typescript
// src/app/layout.tsx
import { Suspense } from 'react'
import { LoadingSpinner } from '@/components/ui/loading'

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body>
        <Suspense fallback={<LoadingSpinner />}>
          {children}
        </Suspense>
      </body>
    </html>
  )
}
```

### Generic Examples (Not Part of Our Project)
Below are some generic examples to illustrate performance optimization patterns:

```typescript
// This is a generic example, not part of our project
// Example of a data-heavy page with streaming
// src/app/(main)/dashboard/page.tsx
import { Suspense } from 'react'
import { DashboardHeader } from './DashboardHeader'
import { DashboardStats } from './DashboardStats'
import { DashboardCharts } from './DashboardCharts'

export default function DashboardPage() {
  return (
    <div>
      <DashboardHeader />
      <Suspense fallback={<div>Loading stats...</div>}>
        <DashboardStats />
      </Suspense>
      <Suspense fallback={<div>Loading charts...</div>}>
        <DashboardCharts />
      </Suspense>
    </div>
  )
}

// This is a generic example, not part of our project
// Example of a component with dynamic imports
// src/app/(main)/reports/page.tsx
import dynamic from 'next/dynamic'

const ReportViewer = dynamic(() => import('@/components/ReportViewer'), {
  loading: () => <div>Loading report viewer...</div>,
  ssr: false,
})

export default function ReportsPage() {
  return (
    <div>
      <h1>Reports</h1>
      <ReportViewer />
    </div>
  )
}
```

## Image Optimization

### Our Implementation
```typescript
// src/components/ui/image.tsx
import Image from 'next/image'

export function OptimizedImage({
  src,
  alt,
  width,
  height,
}: {
  src: string
  alt: string
  width: number
  height: number
}) {
  return (
    <Image
      src={src}
      alt={alt}
      width={width}
      height={height}
      className="rounded-lg"
      priority={false}
    />
  )
}
```

### Generic Examples (Not Part of Our Project)
Below are some generic examples to illustrate image optimization patterns:

```typescript
// This is a generic example, not part of our project
// Example of a gallery component with optimized images
// src/app/(main)/gallery/page.tsx
import Image from 'next/image'

export default function GalleryPage() {
  return (
    <div className="grid grid-cols-3 gap-4">
      {images.map((image) => (
        <Image
          key={image.id}
          src={image.url}
          alt={image.description}
          width={400}
          height={300}
          placeholder="blur"
          blurDataURL={image.blurDataUrl}
          loading="lazy"
        />
      ))}
    </div>
  )
}
``` 