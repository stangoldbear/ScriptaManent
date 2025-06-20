# Code Style Guide 🎨

This guide defines our coding standards and best practices for maintaining consistent, readable, and maintainable code.

## General Principles

### 1. Code Formatting 📝

We use Prettier for consistent code formatting:

```json
// .prettierrc
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "tabWidth": 2,
  "useTabs": false,
  "printWidth": 80,
  "bracketSpacing": true,
  "arrowParens": "always"
}
```

### 2. TypeScript Standards 📘

```typescript
// Use explicit types
interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
}

// Avoid any
// ❌ Bad
function processData(data: any) {}

// ✅ Good
function processData(data: unknown) {
  if (isValidData(data)) {
    // Process data
  }
}

// Use type inference when possible
// ❌ Bad
const name: string = 'John';

// ✅ Good
const name = 'John';
```

## React/Next.js Standards

### 1. Component Structure 🧩

```typescript
// components/Button.tsx
import { forwardRef } from 'react'
import type { ButtonHTMLAttributes } from 'react'
import { cn } from '@/lib/utils'

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary'
  size?: 'sm' | 'md' | 'lg'
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant = 'primary', size = 'md', children, ...props }, ref) => {
    return (
      <button
        ref={ref}
        className={cn(
          'rounded font-medium transition-colors',
          {
            'bg-blue-500 text-white hover:bg-blue-600': variant === 'primary',
            'bg-gray-200 text-gray-800 hover:bg-gray-300': variant === 'secondary',
            'px-3 py-1 text-sm': size === 'sm',
            'px-4 py-2': size === 'md',
            'px-6 py-3 text-lg': size === 'lg',
          },
          className
        )}
        {...props}
      >
        {children}
      </button>
    )
  }
)

Button.displayName = 'Button'
```

### 2. Hooks Usage 🎣

```typescript
// hooks/useAuth.ts
import { useState, useCallback } from 'react'
import type { User } from '@/types'

interface UseAuthReturn {
  user: User | null
  login: (email: string, password: string) => Promise<void>
  logout: () => Promise<void>
}

export function useAuth(): UseAuthReturn {
  const [user, setUser] = useState<User | null>(null)

  const login = useCallback(async (email: string, password: string) => {
    try {
      const user = await api.login(email, password)
      setUser(user)
    } catch (error) {
      throw new Error('Login failed')
    }
  }, [])

  const logout = useCallback(async () => {
    await api.logout()
    setUser(null)
  }, [])

  return { user, login, logout }
}
```

### 3. Page Structure 📄

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react'
import { DashboardHeader } from '@/components/dashboard/Header'
import { DashboardContent } from '@/components/dashboard/Content'
import { LoadingSpinner } from '@/components/ui/LoadingSpinner'

export const metadata = {
  title: 'Dashboard',
  description: 'User dashboard',
}

export default function DashboardPage() {
  return (
    <div className="container mx-auto p-4">
      <DashboardHeader />
      <Suspense fallback={<LoadingSpinner />}>
        <DashboardContent />
      </Suspense>
    </div>
  )
}
```

## Naming Conventions

### 1. Files and Directories 📁

```
src/
├── components/          # PascalCase for components
│   ├── Button.tsx
│   └── UserProfile.tsx
├── hooks/              # camelCase for hooks
│   ├── useAuth.ts
│   └── useForm.ts
├── utils/              # camelCase for utilities
│   ├── formatDate.ts
│   └── validateEmail.ts
└── types/              # PascalCase for types
    ├── User.ts
    └── ApiResponse.ts
```

### 2. Variables and Functions 📝

```typescript
// Variables
const userCount = 0
const isAuthenticated = true
const MAX_RETRY_COUNT = 3

// Functions
function calculateTotal(items: Item[]): number {
  return items.reduce((sum, item) => sum + item.price, 0)
}

// Async functions
async function fetchUserData(id: string): Promise<User> {
  const response = await api.get(`/users/${id}`)
  return response.data
}

// Event handlers
function handleSubmit(event: FormEvent) {
  event.preventDefault()
  // Handle form submission
}
```

## Code Organization

### 1. Imports Order 📦

```typescript
// 1. External dependencies
import { useState, useEffect } from 'react'
import { useRouter } from 'next/router'

// 2. Internal absolute imports
import { Button } from '@/components/ui/Button'
import { useAuth } from '@/hooks/useAuth'

// 3. Internal relative imports
import { styles } from './styles'
import type { Props } from './types'
```

### 2. Component Organization 🎯

```typescript
// 1. Imports
import { useState } from 'react'
import { Button } from '@/components/ui/Button'

// 2. Types
interface Props {
  initialCount: number
}

// 3. Component
export function Counter({ initialCount }: Props) {
  // 4. State
  const [count, setCount] = useState(initialCount)

  // 5. Handlers
  const handleIncrement = () => setCount(prev => prev + 1)

  // 6. Render
  return (
    <div>
      <p>Count: {count}</p>
      <Button onClick={handleIncrement}>Increment</Button>
    </div>
  )
}
```

## Best Practices

### 1. Error Handling 🚨

```typescript
// Use custom error classes
class ApiError extends Error {
  constructor(
    message: string,
    public statusCode: number,
    public code: string
  ) {
    super(message)
    this.name = 'ApiError'
  }
}

// Handle errors appropriately
async function fetchData() {
  try {
    const response = await api.get('/data')
    return response.data
  } catch (error) {
    if (error instanceof ApiError) {
      // Handle API errors
      logger.error('API error:', error)
      throw new Error('Failed to fetch data')
    }
    // Handle other errors
    logger.error('Unexpected error:', error)
    throw error
  }
}
```

### 2. Performance Optimization ⚡

```typescript
// Use memoization
const MemoizedComponent = memo(ExpensiveComponent)

// Use callbacks
const handleClick = useCallback(() => {
  // Handle click
}, [/* dependencies */])

// Use memo for expensive calculations
const processedData = useMemo(() => {
  return expensiveOperation(data)
}, [data])
```

### 3. Accessibility ♿

```typescript
// Use semantic HTML
<button
  type="button"
  aria-label="Close menu"
  onClick={handleClose}
>
  <span className="sr-only">Close menu</span>
  <XIcon />
</button>

// Add proper ARIA attributes
<div
  role="alert"
  aria-live="polite"
  aria-atomic="true"
>
  {message}
</div>
```

## Documentation

### 1. JSDoc Comments 📚

```typescript
/**
 * Fetches user data from the API
 * @param {string} id - The user ID
 * @returns {Promise<User>} The user data
 * @throws {ApiError} If the API request fails
 */
async function fetchUser(id: string): Promise<User> {
  // Implementation
}

/**
 * A button component that supports different variants
 * @component
 * @example
 * ```tsx
 * <Button variant="primary" onClick={handleClick}>
 *   Click me
 * </Button>
 * ```
 */
export const Button = forwardRef<HTMLButtonElement, ButtonProps>((props, ref) => {
  // Implementation
})
```

### 2. README Files 📖

```markdown
# Component Name

A brief description of the component.

## Usage

```tsx
import { Component } from '@/components/Component'

<Component prop1="value" prop2={42} />
```

## Props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| prop1 | string | - | Description of prop1 |
| prop2 | number | 0 | Description of prop2 |

## Examples

More detailed examples and use cases.
```

## Common Patterns

### 1. Data Fetching 🔄

```typescript
// Use SWR or React Query
function UserProfile({ userId }: { userId: string }) {
  const { data, error, isLoading } = useSWR(
    `/api/users/${userId}`,
    fetcher
  )

  if (isLoading) return <LoadingSpinner />
  if (error) return <ErrorMessage error={error} />
  if (!data) return null

  return <UserCard user={data} />
}
```

### 2. Form Handling 📝

```typescript
// Use react-hook-form
function LoginForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<LoginFormData>()

  const onSubmit = async (data: LoginFormData) => {
    try {
      await login(data)
    } catch (error) {
      // Handle error
    }
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input
        {...register('email', {
          required: 'Email is required',
          pattern: {
            value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
            message: 'Invalid email address',
          },
        })}
      />
      {errors.email && <span>{errors.email.message}</span>}
      <button type="submit">Login</button>
    </form>
  )
}
```

## Next Steps

1. Review the [Component Guidelines](./10-component-guidelines.md) for detailed component patterns
2. Check [Development Workflow](./06-development-workflow.md) for how to apply these standards
3. Look at [Testing Strategy](./07-testing-strategy.md) for testing standards 