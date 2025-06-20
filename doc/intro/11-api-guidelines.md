# API Guidelines 🌐

This guide outlines our API design principles, patterns, and best practices for building and consuming APIs in our Next.js application.

## API Design Principles

### 1. RESTful Endpoints 🎯

```typescript
// app/api/users/route.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { db } from '@/lib/db'
import { validateUser } from '@/lib/validators'

// GET /api/users
export async function GET(request: NextRequest) {
  try {
    const users = await db.users.findMany()
    return NextResponse.json(users)
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to fetch users' },
      { status: 500 }
    )
  }
}

// POST /api/users
export async function POST(request: NextRequest) {
  try {
    const body = await request.json()
    const validatedData = validateUser(body)

    const user = await db.users.create({
      data: validatedData,
    })

    return NextResponse.json(user, { status: 201 })
  } catch (error) {
    if (error instanceof ValidationError) {
      return NextResponse.json(
        { error: error.message },
        { status: 400 }
      )
    }
    return NextResponse.json(
      { error: 'Failed to create user' },
      { status: 500 }
    )
  }
}
```

### 2. Resource Routes 📍

```typescript
// app/api/users/[id]/route.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { db } from '@/lib/db'

// GET /api/users/:id
export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  try {
    const user = await db.users.findUnique({
      where: { id: params.id },
    })

    if (!user) {
      return NextResponse.json(
        { error: 'User not found' },
        { status: 404 }
      )
    }

    return NextResponse.json(user)
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to fetch user' },
      { status: 500 }
    )
  }
}

// PATCH /api/users/:id
export async function PATCH(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  try {
    const body = await request.json()
    const user = await db.users.update({
      where: { id: params.id },
      data: body,
    })

    return NextResponse.json(user)
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to update user' },
      { status: 500 }
    )
  }
}

// DELETE /api/users/:id
export async function DELETE(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  try {
    await db.users.delete({
      where: { id: params.id },
    })

    return new NextResponse(null, { status: 204 })
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to delete user' },
      { status: 500 }
    )
  }
}
```

## API Patterns

### 1. Route Handlers 🛣️

```typescript
// app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth'
import { authOptions } from '@/lib/auth'

const handler = NextAuth(authOptions)

export { handler as GET, handler as POST }

// app/api/webhooks/stripe/route.ts
import { headers } from 'next/headers'
import { NextResponse } from 'next/server'
import Stripe from 'stripe'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!)
const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET!

export async function POST(request: NextRequest) {
  const body = await request.text()
  const signature = headers().get('stripe-signature')!

  try {
    const event = stripe.webhooks.constructEvent(
      body,
      signature,
      webhookSecret
    )

    switch (event.type) {
      case 'payment_intent.succeeded':
        await handlePaymentSuccess(event.data.object)
        break
      case 'payment_intent.failed':
        await handlePaymentFailure(event.data.object)
        break
    }

    return NextResponse.json({ received: true })
  } catch (error) {
    return NextResponse.json(
      { error: 'Webhook error' },
      { status: 400 }
    )
  }
}
```

### 2. API Middleware 🔒

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { getToken } from 'next-auth/jwt'

export async function middleware(request: NextRequest) {
  const token = await getToken({ req: request })

  // Protect API routes
  if (request.nextUrl.pathname.startsWith('/api/')) {
    if (!token) {
      return NextResponse.json(
        { error: 'Unauthorized' },
        { status: 401 }
      )
    }

    // Add user info to request headers
    const requestHeaders = new Headers(request.headers)
    requestHeaders.set('x-user-id', token.sub!)
    requestHeaders.set('x-user-role', token.role as string)

    return NextResponse.next({
      request: {
        headers: requestHeaders,
      },
    })
  }

  return NextResponse.next()
}

export const config = {
  matcher: [
    '/api/:path*',
    '/dashboard/:path*',
  ],
}
```

### 3. API Utilities 🛠️

```typescript
// lib/api-utils.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { ZodError } from 'zod'

export class ApiError extends Error {
  constructor(
    message: string,
    public statusCode: number,
    public code: string
  ) {
    super(message)
    this.name = 'ApiError'
  }
}

export function withErrorHandler(
  handler: (req: NextRequest, context: any) => Promise<NextResponse>
) {
  return async (req: NextRequest, context: any) => {
    try {
      return await handler(req, context)
    } catch (error) {
      if (error instanceof ApiError) {
        return NextResponse.json(
          { error: error.message, code: error.code },
          { status: error.statusCode }
        )
      }

      if (error instanceof ZodError) {
        return NextResponse.json(
          { error: 'Validation error', details: error.errors },
          { status: 400 }
        )
      }

      console.error('API Error:', error)
      return NextResponse.json(
        { error: 'Internal server error' },
        { status: 500 }
      )
    }
  }
}

export function withAuth(
  handler: (req: NextRequest, context: any) => Promise<NextResponse>
) {
  return async (req: NextRequest, context: any) => {
    const token = await getToken({ req })
    if (!token) {
      return NextResponse.json(
        { error: 'Unauthorized' },
        { status: 401 }
      )
    }

    return handler(req, { ...context, token })
  }
}
```

## API Client

### 1. API Client Setup 🏗️

```typescript
// lib/api-client.ts
import axios from 'axios'

const api = axios.create({
  baseURL: '/api',
  headers: {
    'Content-Type': 'application/json',
  },
})

// Request interceptor
api.interceptors.request.use(
  config => {
    // Add auth token
    const token = localStorage.getItem('token')
    if (token) {
      config.headers.Authorization = `Bearer ${token}`
    }
    return config
  },
  error => Promise.reject(error)
)

// Response interceptor
api.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      // Handle unauthorized
      window.location.href = '/login'
    }
    return Promise.reject(error)
  }
)

export default api
```

### 2. API Hooks 🎣

```typescript
// hooks/useApi.ts
import { useState, useCallback } from 'react'
import api from '@/lib/api-client'

interface UseApiOptions<T> {
  onSuccess?: (data: T) => void
  onError?: (error: Error) => void
}

export function useApi<T = any>() {
  const [data, setData] = useState<T | null>(null)
  const [error, setError] = useState<Error | null>(null)
  const [isLoading, setIsLoading] = useState(false)

  const execute = useCallback(
    async <D = any>(
      method: 'get' | 'post' | 'put' | 'delete',
      url: string,
      options?: UseApiOptions<D>,
      config?: any
    ) => {
      setIsLoading(true)
      setError(null)

      try {
        const response = await api[method](url, config)
        const result = response.data
        setData(result as T)
        options?.onSuccess?.(result)
        return result
      } catch (err) {
        const error = err as Error
        setError(error)
        options?.onError?.(error)
        throw error
      } finally {
        setIsLoading(false)
      }
    },
    []
  )

  return {
    data,
    error,
    isLoading,
    execute,
  }
}

// Usage example
function UserProfile({ userId }: { userId: string }) {
  const { data: user, error, isLoading, execute } = useApi<User>()

  useEffect(() => {
    execute('get', `/users/${userId}`, {
      onError: error => {
        toast.error('Failed to fetch user')
      },
    })
  }, [userId, execute])

  if (isLoading) return <LoadingSpinner />
  if (error) return <ErrorMessage error={error} />
  if (!user) return null

  return <UserCard user={user} />
}
```

### 3. API Mutations 🔄

```typescript
// hooks/useMutation.ts
import { useState, useCallback } from 'react'
import api from '@/lib/api-client'

interface UseMutationOptions<T, V> {
  onSuccess?: (data: T, variables: V) => void
  onError?: (error: Error, variables: V) => void
}

export function useMutation<T = any, V = any>(
  method: 'post' | 'put' | 'delete',
  url: string,
  options?: UseMutationOptions<T, V>
) {
  const [isLoading, setIsLoading] = useState(false)
  const [error, setError] = useState<Error | null>(null)

  const mutate = useCallback(
    async (variables: V) => {
      setIsLoading(true)
      setError(null)

      try {
        const response = await api[method](url, variables)
        const result = response.data
        options?.onSuccess?.(result, variables)
        return result
      } catch (err) {
        const error = err as Error
        setError(error)
        options?.onError?.(error, variables)
        throw error
      } finally {
        setIsLoading(false)
      }
    },
    [method, url, options]
  )

  return {
    mutate,
    isLoading,
    error,
  }
}

// Usage example
function CreateUserForm() {
  const { mutate, isLoading, error } = useMutation<User, CreateUserInput>(
    'post',
    '/users',
    {
      onSuccess: (user) => {
        toast.success('User created successfully')
        router.push(`/users/${user.id}`)
      },
      onError: (error) => {
        toast.error('Failed to create user')
      },
    }
  )

  const handleSubmit = async (data: CreateUserInput) => {
    try {
      await mutate(data)
    } catch (error) {
      // Error is handled by onError
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      {/* Form fields */}
      <button type="submit" disabled={isLoading}>
        {isLoading ? 'Creating...' : 'Create User'}
      </button>
      {error && <ErrorMessage error={error} />}
    </form>
  )
}
```

## Error Handling

### 1. Error Types 🚨

```typescript
// lib/errors.ts
export class ValidationError extends Error {
  constructor(message: string, public details: any) {
    super(message)
    this.name = 'ValidationError'
  }
}

export class NotFoundError extends Error {
  constructor(resource: string) {
    super(`${resource} not found`)
    this.name = 'NotFoundError'
  }
}

export class UnauthorizedError extends Error {
  constructor() {
    super('Unauthorized')
    this.name = 'UnauthorizedError'
  }
}

export class ForbiddenError extends Error {
  constructor() {
    super('Forbidden')
    this.name = 'ForbiddenError'
  }
}

// Usage in API route
export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const user = await db.users.findUnique({
    where: { id: params.id },
  })

  if (!user) {
    throw new NotFoundError('User')
  }

  if (!hasPermission(request, user)) {
    throw new ForbiddenError()
  }

  return NextResponse.json(user)
}
```

### 2. Error Responses 📤

```typescript
// lib/api-responses.ts
import { NextResponse } from 'next/server'
import type { ApiError } from './api-utils'

export function errorResponse(error: unknown) {
  if (error instanceof ApiError) {
    return NextResponse.json(
      {
        error: error.message,
        code: error.code,
        status: error.statusCode,
      },
      { status: error.statusCode }
    )
  }

  if (error instanceof ValidationError) {
    return NextResponse.json(
      {
        error: 'Validation error',
        details: error.details,
        status: 400,
      },
      { status: 400 }
    )
  }

  console.error('Unhandled error:', error)
  return NextResponse.json(
    {
      error: 'Internal server error',
      status: 500,
    },
    { status: 500 }
  )
}

export function successResponse<T>(data: T, status = 200) {
  return NextResponse.json(
    {
      data,
      status,
    },
    { status }
  )
}

// Usage in API route
export async function POST(request: NextRequest) {
  try {
    const body = await request.json()
    const user = await createUser(body)
    return successResponse(user, 201)
  } catch (error) {
    return errorResponse(error)
  }
}
```

## Security

### 1. Authentication 🔐

```typescript
// lib/auth.ts
import { NextAuthOptions } from 'next-auth'
import CredentialsProvider from 'next-auth/providers/credentials'
import { db } from '@/lib/db'
import { compare } from 'bcrypt'

export const authOptions: NextAuthOptions = {
  providers: [
    CredentialsProvider({
      name: 'Credentials',
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) {
          throw new Error('Missing credentials')
        }

        const user = await db.users.findUnique({
          where: { email: credentials.email },
        })

        if (!user) {
          throw new Error('Invalid credentials')
        }

        const isValid = await compare(credentials.password, user.password)

        if (!isValid) {
          throw new Error('Invalid credentials')
        }

        return {
          id: user.id,
          email: user.email,
          name: user.name,
          role: user.role,
        }
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.role = user.role
      }
      return token
    },
    async session({ session, token }) {
      if (token) {
        session.user.role = token.role
      }
      return session
    },
  },
  pages: {
    signIn: '/login',
    error: '/auth/error',
  },
}
```

### 2. Rate Limiting ⏱️

```typescript
// lib/rate-limit.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { Redis } from '@upstash/redis'

const redis = new Redis({
  url: process.env.REDIS_URL!,
  token: process.env.REDIS_TOKEN!,
})

export async function rateLimit(
  request: NextRequest,
  limit: number,
  window: number
) {
  const ip = request.ip ?? '127.0.0.1'
  const key = `rate-limit:${ip}`
  const now = Date.now()

  const requests = await redis.zrangebyscore(key, 0, now - window)
  await redis.zremrangebyscore(key, 0, now - window)

  const count = requests.length
  if (count >= limit) {
    return NextResponse.json(
      { error: 'Too many requests' },
      { status: 429 }
    )
  }

  await redis.zadd(key, { score: now, member: now.toString() })
  await redis.expire(key, window)

  return null
}

// Usage in API route
export async function POST(request: NextRequest) {
  const rateLimitResult = await rateLimit(request, 10, 60 * 1000) // 10 requests per minute
  if (rateLimitResult) return rateLimitResult

  // Continue with request handling
}
```

### 3. CORS Configuration 🌐

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const response = NextResponse.next()

  // CORS headers
  response.headers.set('Access-Control-Allow-Origin', process.env.NEXT_PUBLIC_APP_URL!)
  response.headers.set(
    'Access-Control-Allow-Methods',
    'GET, POST, PUT, DELETE, OPTIONS'
  )
  response.headers.set(
    'Access-Control-Allow-Headers',
    'Content-Type, Authorization'
  )
  response.headers.set('Access-Control-Max-Age', '86400')

  // Handle preflight requests
  if (request.method === 'OPTIONS') {
    return new NextResponse(null, { status: 204 })
  }

  return response
}

export const config = {
  matcher: '/api/:path*',
}
```

## Testing

### 1. API Route Testing 🧪

```typescript
// app/api/users/route.test.ts
import { createMocks } from 'node-mocks-http'
import { GET, POST } from './route'
import { db } from '@/lib/db'

jest.mock('@/lib/db')

describe('Users API', () => {
  beforeEach(() => {
    jest.clearAllMocks()
  })

  it('GET /api/users returns users', async () => {
    const mockUsers = [
      { id: '1', name: 'John' },
      { id: '2', name: 'Jane' },
    ]
    ;(db.users.findMany as jest.Mock).mockResolvedValue(mockUsers)

    const { req } = createMocks({
      method: 'GET',
    })

    const response = await GET(req)
    const data = await response.json()

    expect(response.status).toBe(200)
    expect(data).toEqual(mockUsers)
  })

  it('POST /api/users creates user', async () => {
    const mockUser = { id: '1', name: 'John' }
    ;(db.users.create as jest.Mock).mockResolvedValue(mockUser)

    const { req } = createMocks({
      method: 'POST',
      body: { name: 'John' },
    })

    const response = await POST(req)
    const data = await response.json()

    expect(response.status).toBe(201)
    expect(data).toEqual(mockUser)
  })
})
```

### 2. API Client Testing 🔄

```typescript
// lib/api-client.test.ts
import api from './api-client'
import { renderHook, act } from '@testing-library/react-hooks'
import { useApi } from './useApi'

describe('API Client', () => {
  beforeEach(() => {
    localStorage.clear()
    jest.clearAllMocks()
  })

  it('adds auth token to requests', async () => {
    localStorage.setItem('token', 'test-token')

    const { result } = renderHook(() => useApi())

    await act(async () => {
      await result.current.execute('get', '/users')
    })

    expect(api.defaults.headers.common.Authorization).toBe(
      'Bearer test-token'
    )
  })

  it('handles 401 responses', async () => {
    const mockAxios = jest.spyOn(api, 'get')
    mockAxios.mockRejectedValueOnce({
      response: { status: 401 },
    })

    const { result } = renderHook(() => useApi())

    await act(async () => {
      try {
        await result.current.execute('get', '/users')
      } catch (error) {
        // Error is expected
      }
    })

    expect(window.location.href).toBe('/login')
  })
})
```

## Next Steps

1. Review the [Code Style Guide](./09-code-style-guide.md) for API coding standards
2. Check [Development Workflow](./06-development-workflow.md) for API development process
3. Look at [Testing Strategy](./07-testing-strategy.md) for more API testing patterns 