# Security Guidelines 🔒

This guide outlines our security best practices and standards for building secure Next.js applications.

## Authentication & Authorization

### 1. Authentication Flow 🔑

```typescript
// lib/auth.ts
import { NextAuthOptions } from 'next-auth'
import CredentialsProvider from 'next-auth/providers/credentials'
import { db } from '@/lib/db'
import { compare, hash } from 'bcrypt'
import { z } from 'zod'

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})

export const authOptions: NextAuthOptions = {
  providers: [
    CredentialsProvider({
      name: 'Credentials',
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        try {
          const { email, password } = loginSchema.parse(credentials)

          const user = await db.users.findUnique({
            where: { email },
            select: {
              id: true,
              email: true,
              password: true,
              role: true,
              isActive: true,
            },
          })

          if (!user || !user.isActive) {
            throw new Error('Invalid credentials')
          }

          const isValid = await compare(password, user.password)
          if (!isValid) {
            throw new Error('Invalid credentials')
          }

          // Log successful login
          await db.loginAttempts.create({
            data: {
              userId: user.id,
              success: true,
              ipAddress: request.headers.get('x-forwarded-for') ?? 'unknown',
            },
          })

          return {
            id: user.id,
            email: user.email,
            role: user.role,
          }
        } catch (error) {
          // Log failed login attempt
          if (credentials?.email) {
            await db.loginAttempts.create({
              data: {
                email: credentials.email,
                success: false,
                ipAddress: request.headers.get('x-forwarded-for') ?? 'unknown',
              },
            })
          }

          throw error
        }
      },
    }),
  ],
  session: {
    strategy: 'jwt',
    maxAge: 24 * 60 * 60, // 24 hours
  },
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.role = user.role
        token.id = user.id
      }
      return token
    },
    async session({ session, token }) {
      if (token) {
        session.user.role = token.role
        session.user.id = token.id
      }
      return session
    },
  },
  pages: {
    signIn: '/login',
    error: '/auth/error',
  },
  events: {
    async signOut({ token }) {
      // Clear any server-side sessions
      await db.sessions.deleteMany({
        where: { userId: token.id },
      })
    },
  },
}
```

### 2. Role-Based Access Control 👥

```typescript
// lib/rbac.ts
type Role = 'admin' | 'manager' | 'user'

interface Permission {
  action: 'create' | 'read' | 'update' | 'delete'
  resource: string
}

const rolePermissions: Record<Role, Permission[]> = {
  admin: [
    { action: 'create', resource: '*' },
    { action: 'read', resource: '*' },
    { action: 'update', resource: '*' },
    { action: 'delete', resource: '*' },
  ],
  manager: [
    { action: 'create', resource: 'users' },
    { action: 'read', resource: '*' },
    { action: 'update', resource: 'users' },
    { action: 'delete', resource: 'users' },
  ],
  user: [
    { action: 'read', resource: 'users' },
    { action: 'update', resource: 'users' },
  ],
}

export function hasPermission(
  role: Role,
  action: Permission['action'],
  resource: string
): boolean {
  const permissions = rolePermissions[role]
  return permissions.some(
    p =>
      (p.action === action || p.action === '*') &&
      (p.resource === resource || p.resource === '*')
  )
}

// Usage in API route
export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const session = await getServerSession(authOptions)
  if (!session) {
    throw new UnauthorizedError()
  }

  if (!hasPermission(session.user.role, 'read', 'users')) {
    throw new ForbiddenError()
  }

  // Continue with request
}
```

### 3. Session Management 🔄

```typescript
// lib/session.ts
import { Redis } from '@upstash/redis'
import { getToken } from 'next-auth/jwt'

const redis = new Redis({
  url: process.env.REDIS_URL!,
  token: process.env.REDIS_TOKEN!,
})

export async function validateSession(request: NextRequest) {
  const token = await getToken({ req: request })
  if (!token) {
    throw new UnauthorizedError()
  }

  // Check if session is blacklisted
  const isBlacklisted = await redis.get(`blacklist:${token.jti}`)
  if (isBlacklisted) {
    throw new UnauthorizedError('Session invalidated')
  }

  // Check session activity
  const lastActivity = await redis.get(`session:${token.jti}:lastActivity`)
  if (lastActivity) {
    const inactiveTime = Date.now() - Number(lastActivity)
    if (inactiveTime > 30 * 60 * 1000) { // 30 minutes
      await invalidateSession(token.jti)
      throw new UnauthorizedError('Session expired')
    }
  }

  // Update last activity
  await redis.set(`session:${token.jti}:lastActivity`, Date.now())
  return token
}

export async function invalidateSession(jti: string) {
  await redis.set(`blacklist:${jti}`, true, { ex: 24 * 60 * 60 }) // 24 hours
  await redis.del(`session:${jti}:lastActivity`)
}

// Usage in middleware
export async function middleware(request: NextRequest) {
  try {
    await validateSession(request)
    return NextResponse.next()
  } catch (error) {
    if (error instanceof UnauthorizedError) {
      return NextResponse.redirect(new URL('/login', request.url))
    }
    throw error
  }
}
```

## Data Security

### 1. Password Security 🔐

```typescript
// lib/password.ts
import { hash, compare, genSalt } from 'bcrypt'
import { z } from 'zod'

const passwordSchema = z
  .string()
  .min(8)
  .regex(/[A-Z]/, 'Password must contain at least one uppercase letter')
  .regex(/[a-z]/, 'Password must contain at least one lowercase letter')
  .regex(/[0-9]/, 'Password must contain at least one number')
  .regex(/[^A-Za-z0-9]/, 'Password must contain at least one special character')

export async function hashPassword(password: string): Promise<string> {
  const validatedPassword = passwordSchema.parse(password)
  const salt = await genSalt(12)
  return hash(validatedPassword, salt)
}

export async function verifyPassword(
  password: string,
  hashedPassword: string
): Promise<boolean> {
  return compare(password, hashedPassword)
}

// Usage in registration
export async function registerUser(data: RegisterInput) {
  const hashedPassword = await hashPassword(data.password)
  return db.users.create({
    data: {
      ...data,
      password: hashedPassword,
    },
  })
}
```

### 2. Data Encryption 🔏

```typescript
// lib/encryption.ts
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto'

const ENCRYPTION_KEY = Buffer.from(process.env.ENCRYPTION_KEY!, 'hex')
const ALGORITHM = 'aes-256-gcm'

export function encrypt(text: string): string {
  const iv = randomBytes(12)
  const cipher = createCipheriv(ALGORITHM, ENCRYPTION_KEY, iv)
  
  let encrypted = cipher.update(text, 'utf8', 'hex')
  encrypted += cipher.final('hex')
  
  const authTag = cipher.getAuthTag()
  
  return `${iv.toString('hex')}:${encrypted}:${authTag.toString('hex')}`
}

export function decrypt(encryptedText: string): string {
  const [ivHex, encrypted, authTagHex] = encryptedText.split(':')
  
  const iv = Buffer.from(ivHex, 'hex')
  const authTag = Buffer.from(authTagHex, 'hex')
  
  const decipher = createDecipheriv(ALGORITHM, ENCRYPTION_KEY, iv)
  decipher.setAuthTag(authTag)
  
  let decrypted = decipher.update(encrypted, 'hex', 'utf8')
  decrypted += decipher.final('utf8')
  
  return decrypted
}

// Usage with sensitive data
export async function storeSensitiveData(userId: string, data: SensitiveData) {
  const encryptedData = encrypt(JSON.stringify(data))
  return db.sensitiveData.create({
    data: {
      userId,
      encryptedData,
    },
  })
}
```

### 3. Input Validation 🛡️

```typescript
// lib/validation.ts
import { z } from 'zod'

// User input validation
export const userSchema = z.object({
  email: z.string().email(),
  name: z.string().min(2).max(100),
  role: z.enum(['admin', 'manager', 'user']),
  settings: z.object({
    theme: z.enum(['light', 'dark']),
    notifications: z.boolean(),
  }),
})

// File upload validation
export const fileUploadSchema = z.object({
  file: z
    .instanceof(File)
    .refine(file => file.size <= 5 * 1024 * 1024, 'File size must be less than 5MB')
    .refine(
      file => ['image/jpeg', 'image/png', 'application/pdf'].includes(file.type),
      'File must be JPEG, PNG, or PDF'
    ),
})

// API request validation
export const apiRequestSchema = z.object({
  method: z.enum(['GET', 'POST', 'PUT', 'DELETE']),
  path: z.string().regex(/^\/api\/[a-z0-9/-]+$/),
  headers: z.record(z.string()),
  query: z.record(z.string().optional()),
  body: z.unknown(),
})

// Usage in API route
export async function POST(request: NextRequest) {
  const body = await request.json()
  const validatedData = userSchema.parse(body)
  
  // Process validated data
  const user = await db.users.create({
    data: validatedData,
  })
  
  return NextResponse.json(user)
}
```

## API Security

### 1. Rate Limiting ⏱️

```typescript
// lib/rate-limit.ts
import { Redis } from '@upstash/redis'
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

const redis = new Redis({
  url: process.env.REDIS_URL!,
  token: process.env.REDIS_TOKEN!,
})

interface RateLimitConfig {
  limit: number
  window: number
  blockDuration?: number
}

const rateLimits: Record<string, RateLimitConfig> = {
  'api/auth': { limit: 5, window: 60 * 1000, blockDuration: 15 * 60 * 1000 }, // 5 attempts per minute, block for 15 minutes
  'api/users': { limit: 100, window: 60 * 1000 }, // 100 requests per minute
  default: { limit: 50, window: 60 * 1000 }, // 50 requests per minute
}

export async function rateLimit(
  request: NextRequest,
  path: string
): Promise<NextResponse | null> {
  const ip = request.ip ?? '127.0.0.1'
  const config = rateLimits[path] ?? rateLimits.default
  const key = `rate-limit:${path}:${ip}`
  const blockKey = `rate-limit:${path}:${ip}:blocked`

  // Check if IP is blocked
  const isBlocked = await redis.get(blockKey)
  if (isBlocked) {
    return NextResponse.json(
      { error: 'Too many requests. Please try again later.' },
      { status: 429 }
    )
  }

  const now = Date.now()
  const windowStart = now - config.window

  // Get request count in current window
  const requests = await redis.zrangebyscore(key, windowStart, now)
  const count = requests.length

  if (count >= config.limit) {
    // Block IP if blockDuration is configured
    if (config.blockDuration) {
      await redis.set(blockKey, true, { ex: config.blockDuration / 1000 })
    }

    return NextResponse.json(
      { error: 'Too many requests' },
      { status: 429 }
    )
  }

  // Add request to window
  await redis.zadd(key, { score: now, member: now.toString() })
  await redis.expire(key, config.window / 1000)

  return null
}

// Usage in middleware
export async function middleware(request: NextRequest) {
  if (request.nextUrl.pathname.startsWith('/api/')) {
    const rateLimitResult = await rateLimit(
      request,
      request.nextUrl.pathname
    )
    if (rateLimitResult) return rateLimitResult
  }
  return NextResponse.next()
}
```

### 2. CORS Configuration 🌐

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

const allowedOrigins = [
  process.env.NEXT_PUBLIC_APP_URL!,
  'https://staging.example.com',
]

const allowedMethods = ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS']
const allowedHeaders = [
  'Content-Type',
  'Authorization',
  'X-Requested-With',
]

export function middleware(request: NextRequest) {
  const origin = request.headers.get('origin')
  const response = NextResponse.next()

  // Handle CORS
  if (origin && allowedOrigins.includes(origin)) {
    response.headers.set('Access-Control-Allow-Origin', origin)
    response.headers.set(
      'Access-Control-Allow-Methods',
      allowedMethods.join(', ')
    )
    response.headers.set(
      'Access-Control-Allow-Headers',
      allowedHeaders.join(', ')
    )
    response.headers.set('Access-Control-Max-Age', '86400')
    response.headers.set('Access-Control-Allow-Credentials', 'true')
  }

  // Handle preflight
  if (request.method === 'OPTIONS') {
    return new NextResponse(null, { status: 204 })
  }

  // Security headers
  response.headers.set('X-Content-Type-Options', 'nosniff')
  response.headers.set('X-Frame-Options', 'DENY')
  response.headers.set('X-XSS-Protection', '1; mode=block')
  response.headers.set(
    'Strict-Transport-Security',
    'max-age=31536000; includeSubDomains'
  )
  response.headers.set(
    'Content-Security-Policy',
    "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';"
  )

  return response
}

export const config = {
  matcher: '/api/:path*',
}
```

### 3. Request Validation 🔍

```typescript
// lib/request-validation.ts
import { z } from 'zod'
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function validateRequest<T extends z.ZodType>(
  schema: T,
  request: NextRequest
): Promise<z.infer<T>> {
  return schema.parseAsync(request)
}

// API route schemas
const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(2),
  role: z.enum(['user', 'admin']),
})

const updateUserSchema = createUserSchema.partial()

// Usage in API route
export async function POST(request: NextRequest) {
  try {
    const body = await request.json()
    const data = await validateRequest(createUserSchema, body)
    
    const user = await db.users.create({ data })
    return NextResponse.json(user)
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { error: 'Validation error', details: error.errors },
        { status: 400 }
      )
    }
    throw error
  }
}
```

## Security Headers

### 1. Security Middleware 🛡️

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const response = NextResponse.next()

  // Security headers
  const securityHeaders = {
    'X-DNS-Prefetch-Control': 'on',
    'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
    'X-Frame-Options': 'SAMEORIGIN',
    'X-Content-Type-Options': 'nosniff',
    'Referrer-Policy': 'strict-origin-when-cross-origin',
    'Permissions-Policy': [
      'camera=()',
      'microphone=()',
      'geolocation=()',
      'interest-cohort=()',
    ].join(', '),
    'Content-Security-Policy': [
      "default-src 'self'",
      "script-src 'self' 'unsafe-inline' 'unsafe-eval'",
      "style-src 'self' 'unsafe-inline'",
      "img-src 'self' data: https:",
      "font-src 'self'",
      "connect-src 'self' https://api.example.com",
      "frame-ancestors 'none'",
      "base-uri 'self'",
      "form-action 'self'",
    ].join('; '),
  }

  Object.entries(securityHeaders).forEach(([key, value]) => {
    response.headers.set(key, value)
  })

  return response
}

export const config = {
  matcher: [
    /*
     * Match all request paths except:
     * 1. /api routes
     * 2. /_next (Next.js internals)
     * 3. /_static (static files)
     * 4. /favicon.ico, /sitemap.xml (static files)
     */
    '/((?!api|_next|_static|favicon.ico|sitemap.xml).*)',
  ],
}
```

### 2. CSP Configuration 📜

```typescript
// lib/csp.ts
const cspDirectives = {
  defaultSrc: ["'self'"],
  scriptSrc: [
    "'self'",
    "'unsafe-inline'",
    "'unsafe-eval'",
    'https://cdn.example.com',
  ],
  styleSrc: ["'self'", "'unsafe-inline'", 'https://fonts.googleapis.com'],
  imgSrc: ["'self'", 'data:', 'https:'],
  fontSrc: ["'self'", 'https://fonts.gstatic.com'],
  connectSrc: ["'self'", 'https://api.example.com'],
  frameSrc: ["'none'"],
  objectSrc: ["'none'"],
  mediaSrc: ["'self'"],
  frameAncestors: ["'none'"],
  baseUri: ["'self'"],
  formAction: ["'self'"],
  upgradeInsecureRequests: [],
}

export function generateCSP(nonce: string) {
  const directives = Object.entries(cspDirectives)
    .map(([key, values]) => {
      const value = values
        .map(v => (v === nonce ? `'nonce-${nonce}'` : v))
        .join(' ')
      return `${key} ${value}`
    })
    .join('; ')

  return directives
}

// Usage in middleware
export function middleware(request: NextRequest) {
  const nonce = randomBytes(16).toString('base64')
  const csp = generateCSP(nonce)

  const response = NextResponse.next()
  response.headers.set('Content-Security-Policy', csp)
  response.headers.set('X-Nonce', nonce)

  return response
}
```

## Security Monitoring

### 1. Security Logging 📝

```typescript
// lib/security-logger.ts
import { db } from '@/lib/db'
import type { NextRequest } from 'next/server'

interface SecurityEvent {
  type: 'login' | 'logout' | 'auth_failure' | 'rate_limit' | 'permission_denied'
  userId?: string
  ipAddress: string
  userAgent: string
  details: Record<string, any>
}

export async function logSecurityEvent(
  request: NextRequest,
  event: Omit<SecurityEvent, 'ipAddress' | 'userAgent'>
) {
  const securityEvent: SecurityEvent = {
    ...event,
    ipAddress: request.ip ?? 'unknown',
    userAgent: request.headers.get('user-agent') ?? 'unknown',
  }

  await db.securityLogs.create({
    data: securityEvent,
  })

  // Alert on suspicious activity
  if (isSuspiciousActivity(securityEvent)) {
    await sendSecurityAlert(securityEvent)
  }
}

function isSuspiciousActivity(event: SecurityEvent): boolean {
  // Check for multiple failed login attempts
  if (event.type === 'auth_failure') {
    return true
  }

  // Check for unusual IP addresses
  if (isUnusualIP(event.ipAddress)) {
    return true
  }

  // Check for suspicious user agents
  if (isSuspiciousUserAgent(event.userAgent)) {
    return true
  }

  return false
}

// Usage in auth flow
export async function login(request: NextRequest, credentials: LoginCredentials) {
  try {
    const user = await authenticateUser(credentials)
    await logSecurityEvent(request, {
      type: 'login',
      userId: user.id,
      details: { success: true },
    })
    return user
  } catch (error) {
    await logSecurityEvent(request, {
      type: 'auth_failure',
      details: {
        error: error.message,
        email: credentials.email,
      },
    })
    throw error
  }
}
```

### 2. Security Alerts 🚨

```typescript
// lib/security-alerts.ts
import { sendEmail } from '@/lib/email'
import { sendSlackMessage } from '@/lib/slack'

interface SecurityAlert {
  level: 'info' | 'warning' | 'critical'
  title: string
  message: string
  details: Record<string, any>
}

export async function sendSecurityAlert(alert: SecurityAlert) {
  // Log alert
  await db.securityAlerts.create({
    data: {
      level: alert.level,
      title: alert.title,
      message: alert.message,
      details: alert.details,
    },
  })

  // Send email for critical alerts
  if (alert.level === 'critical') {
    await sendEmail({
      to: process.env.SECURITY_ALERT_EMAIL!,
      subject: `[CRITICAL] ${alert.title}`,
      text: formatAlertEmail(alert),
    })
  }

  // Send Slack notification for all alerts
  await sendSlackMessage({
    channel: '#security-alerts',
    text: formatSlackMessage(alert),
  })
}

// Usage in security monitoring
async function monitorSecurityEvents() {
  const recentEvents = await db.securityLogs.findMany({
    where: {
      createdAt: {
        gte: new Date(Date.now() - 5 * 60 * 1000), // Last 5 minutes
      },
    },
  })

  const suspiciousEvents = recentEvents.filter(isSuspiciousActivity)
  if (suspiciousEvents.length > 0) {
    await sendSecurityAlert({
      level: 'warning',
      title: 'Suspicious Activity Detected',
      message: 'Multiple suspicious events detected in the last 5 minutes',
      details: {
        events: suspiciousEvents,
      },
    })
  }
}
```

## Next Steps

1. Review the [Code Style Guide](./09-code-style-guide.md) for security coding standards
2. Check [Development Workflow](./06-development-workflow.md) for security review process
3. Look at [Testing Strategy](./07-testing-strategy.md) for security testing patterns 