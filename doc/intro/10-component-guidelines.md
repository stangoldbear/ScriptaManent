# Component Guidelines 🧩

This guide outlines our component patterns, best practices, and reusable solutions for building maintainable React components.

## Component Types

### 1. Presentational Components 🎨

```typescript
// components/ui/Card.tsx
interface CardProps {
  title: string
  children: React.ReactNode
  className?: string
}

export function Card({ title, children, className }: CardProps) {
  return (
    <div className={cn('rounded-lg border p-4 shadow-sm', className)}>
      <h3 className="mb-2 text-lg font-semibold">{title}</h3>
      <div>{children}</div>
    </div>
  )
}
```

### 2. Container Components 📦

```typescript
// components/UserProfile.tsx
export function UserProfile({ userId }: { userId: string }) {
  const { data: user, isLoading, error } = useSWR(`/api/users/${userId}`, fetcher)

  if (isLoading) return <LoadingSpinner />
  if (error) return <ErrorMessage error={error} />
  if (!user) return null

  return (
    <Card title="User Profile">
      <UserDetails user={user} />
      <UserActions userId={userId} />
    </Card>
  )
}
```

### 3. Layout Components 📐

```typescript
// components/layout/PageLayout.tsx
interface PageLayoutProps {
  header?: React.ReactNode
  sidebar?: React.ReactNode
  children: React.ReactNode
  footer?: React.ReactNode
}

export function PageLayout({
  header,
  sidebar,
  children,
  footer,
}: PageLayoutProps) {
  return (
    <div className="min-h-screen bg-gray-50">
      {header && <header className="sticky top-0 z-10">{header}</header>}
      <div className="container mx-auto flex gap-4 p-4">
        {sidebar && <aside className="w-64">{sidebar}</aside>}
        <main className="flex-1">{children}</main>
      </div>
      {footer && <footer>{footer}</footer>}
    </div>
  )
}
```

## Component Patterns

### 1. Compound Components 🎯

```typescript
// components/ui/Accordion.tsx
interface AccordionContextType {
  isOpen: boolean
  toggle: () => void
}

const AccordionContext = createContext<AccordionContextType | null>(null)

function useAccordion() {
  const context = useContext(AccordionContext)
  if (!context) {
    throw new Error('Accordion components must be used within Accordion')
  }
  return context
}

interface AccordionProps {
  children: React.ReactNode
  defaultOpen?: boolean
}

export function Accordion({ children, defaultOpen = false }: AccordionProps) {
  const [isOpen, setIsOpen] = useState(defaultOpen)
  const toggle = useCallback(() => setIsOpen(prev => !prev), [])

  return (
    <AccordionContext.Provider value={{ isOpen, toggle }}>
      <div className="border rounded-lg">{children}</div>
    </AccordionContext.Provider>
  )
}

Accordion.Header = function AccordionHeader({ children }: { children: React.ReactNode }) {
  const { isOpen, toggle } = useAccordion()
  return (
    <button
      className="flex w-full items-center justify-between p-4"
      onClick={toggle}
    >
      {children}
      <ChevronIcon className={cn('transition-transform', isOpen && 'rotate-180')} />
    </button>
  )
}

Accordion.Content = function AccordionContent({ children }: { children: React.ReactNode }) {
  const { isOpen } = useAccordion()
  if (!isOpen) return null
  return <div className="p-4 border-t">{children}</div>
}
```

### 2. Render Props 🎭

```typescript
// components/ui/DataTable.tsx
interface DataTableProps<T> {
  data: T[]
  columns: Column<T>[]
  renderRow: (item: T, index: number) => React.ReactNode
  renderEmpty?: () => React.ReactNode
}

export function DataTable<T>({
  data,
  columns,
  renderRow,
  renderEmpty,
}: DataTableProps<T>) {
  if (data.length === 0) {
    return renderEmpty?.() ?? <EmptyState />
  }

  return (
    <table className="w-full">
      <thead>
        <tr>
          {columns.map(column => (
            <th key={column.key} className="text-left p-2">
              {column.header}
            </th>
          ))}
        </tr>
      </thead>
      <tbody>
        {data.map((item, index) => renderRow(item, index))}
      </tbody>
    </table>
  )
}
```

### 3. Higher-Order Components 🎪

```typescript
// components/hoc/withErrorBoundary.tsx
interface ErrorBoundaryProps {
  fallback: React.ReactNode
  onError?: (error: Error) => void
}

function withErrorBoundary<P extends object>(
  WrappedComponent: React.ComponentType<P>,
  { fallback, onError }: ErrorBoundaryProps
) {
  return function WithErrorBoundary(props: P) {
    const [hasError, setHasError] = useState(false)
    const [error, setError] = useState<Error | null>(null)

    useEffect(() => {
      const handleError = (error: Error) => {
        setHasError(true)
        setError(error)
        onError?.(error)
      }

      window.addEventListener('error', handleError)
      return () => window.removeEventListener('error', handleError)
    }, [onError])

    if (hasError) {
      return fallback
    }

    return <WrappedComponent {...props} />
  }
}
```

## State Management

### 1. Local State 🏠

```typescript
// components/forms/FormField.tsx
interface FormFieldProps {
  label: string
  name: string
  type?: 'text' | 'email' | 'password'
  required?: boolean
  error?: string
}

export function FormField({
  label,
  name,
  type = 'text',
  required,
  error,
}: FormFieldProps) {
  const [isFocused, setIsFocused] = useState(false)
  const [value, setValue] = useState('')

  return (
    <div className="space-y-1">
      <label
        htmlFor={name}
        className={cn(
          'block text-sm font-medium',
          error ? 'text-red-600' : 'text-gray-700'
        )}
      >
        {label}
        {required && <span className="text-red-500 ml-1">*</span>}
      </label>
      <input
        id={name}
        name={name}
        type={type}
        value={value}
        onChange={e => setValue(e.target.value)}
        onFocus={() => setIsFocused(true)}
        onBlur={() => setIsFocused(false)}
        className={cn(
          'w-full rounded-md border p-2',
          isFocused && 'ring-2 ring-blue-500',
          error && 'border-red-500'
        )}
      />
      {error && <p className="text-sm text-red-600">{error}</p>}
    </div>
  )
}
```

### 2. Context State 🌍

```typescript
// contexts/ThemeContext.tsx
type Theme = 'light' | 'dark'

interface ThemeContextType {
  theme: Theme
  toggleTheme: () => void
}

const ThemeContext = createContext<ThemeContextType | null>(null)

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>(() => {
    if (typeof window !== 'undefined') {
      return (localStorage.getItem('theme') as Theme) ?? 'light'
    }
    return 'light'
  })

  const toggleTheme = useCallback(() => {
    setTheme(prev => {
      const next = prev === 'light' ? 'dark' : 'light'
      localStorage.setItem('theme', next)
      return next
    })
  }, [])

  useEffect(() => {
    document.documentElement.classList.remove('light', 'dark')
    document.documentElement.classList.add(theme)
  }, [theme])

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  )
}
```

### 3. Form State 📝

```typescript
// components/forms/LoginForm.tsx
interface LoginFormData {
  email: string
  password: string
  rememberMe: boolean
}

export function LoginForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<LoginFormData>({
    defaultValues: {
      email: '',
      password: '',
      rememberMe: false,
    },
  })

  const onSubmit = async (data: LoginFormData) => {
    try {
      await login(data)
    } catch (error) {
      // Handle error
    }
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <FormField
        label="Email"
        type="email"
        {...register('email', {
          required: 'Email is required',
          pattern: {
            value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
            message: 'Invalid email address',
          },
        })}
        error={errors.email?.message}
      />
      <FormField
        label="Password"
        type="password"
        {...register('password', {
          required: 'Password is required',
          minLength: {
            value: 8,
            message: 'Password must be at least 8 characters',
          },
        })}
        error={errors.password?.message}
      />
      <label className="flex items-center">
        <input
          type="checkbox"
          {...register('rememberMe')}
          className="mr-2"
        />
        Remember me
      </label>
      <button
        type="submit"
        disabled={isSubmitting}
        className="w-full rounded-md bg-blue-500 px-4 py-2 text-white"
      >
        {isSubmitting ? 'Logging in...' : 'Login'}
      </button>
    </form>
  )
}
```

## Performance Optimization

### 1. Memoization 🧠

```typescript
// components/UserList.tsx
interface User {
  id: string
  name: string
  email: string
}

const UserCard = memo(function UserCard({ user }: { user: User }) {
  return (
    <div className="rounded-lg border p-4">
      <h3 className="font-semibold">{user.name}</h3>
      <p className="text-gray-600">{user.email}</p>
    </div>
  )
})

export function UserList({ users }: { users: User[] }) {
  const sortedUsers = useMemo(() => {
    return [...users].sort((a, b) => a.name.localeCompare(b.name))
  }, [users])

  return (
    <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
      {sortedUsers.map(user => (
        <UserCard key={user.id} user={user} />
      ))}
    </div>
  )
}
```

### 2. Code Splitting 📦

```typescript
// components/Dashboard.tsx
const UserProfile = lazy(() => import('./UserProfile'))
const UserSettings = lazy(() => import('./UserSettings'))
const UserAnalytics = lazy(() => import('./UserAnalytics'))

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

### 3. Virtualization 📊

```typescript
// components/VirtualList.tsx
interface VirtualListProps<T> {
  items: T[]
  height: number
  itemHeight: number
  renderItem: (item: T, index: number) => React.ReactNode
}

export function VirtualList<T>({
  items,
  height,
  itemHeight,
  renderItem,
}: VirtualListProps<T>) {
  const [scrollTop, setScrollTop] = useState(0)

  const visibleItems = useMemo(() => {
    const startIndex = Math.floor(scrollTop / itemHeight)
    const endIndex = Math.min(
      startIndex + Math.ceil(height / itemHeight),
      items.length
    )
    return items.slice(startIndex, endIndex)
  }, [items, scrollTop, itemHeight, height])

  return (
    <div
      style={{ height, overflow: 'auto' }}
      onScroll={e => setScrollTop(e.currentTarget.scrollTop)}
    >
      <div style={{ height: items.length * itemHeight }}>
        <div
          style={{
            transform: `translateY(${Math.floor(scrollTop / itemHeight) * itemHeight}px)`,
          }}
        >
          {visibleItems.map((item, index) => renderItem(item, index))}
        </div>
      </div>
    </div>
  )
}
```

## Accessibility

### 1. Keyboard Navigation ⌨️

```typescript
// components/ui/Modal.tsx
interface ModalProps {
  isOpen: boolean
  onClose: () => void
  title: string
  children: React.ReactNode
}

export function Modal({ isOpen, onClose, title, children }: ModalProps) {
  const overlayRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    if (!isOpen) return

    const handleEscape = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose()
    }

    const handleTab = (e: KeyboardEvent) => {
      if (e.key === 'Tab') {
        const focusableElements = overlayRef.current?.querySelectorAll(
          'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
        )
        if (!focusableElements?.length) return

        const firstElement = focusableElements[0] as HTMLElement
        const lastElement = focusableElements[focusableElements.length - 1] as HTMLElement

        if (e.shiftKey) {
          if (document.activeElement === firstElement) {
            lastElement.focus()
            e.preventDefault()
          }
        } else {
          if (document.activeElement === lastElement) {
            firstElement.focus()
            e.preventDefault()
          }
        }
      }
    }

    document.addEventListener('keydown', handleEscape)
    document.addEventListener('keydown', handleTab)
    return () => {
      document.removeEventListener('keydown', handleEscape)
      document.removeEventListener('keydown', handleTab)
    }
  }, [isOpen, onClose])

  if (!isOpen) return null

  return (
    <div
      ref={overlayRef}
      className="fixed inset-0 z-50 flex items-center justify-center bg-black bg-opacity-50"
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
    >
      <div className="w-full max-w-md rounded-lg bg-white p-6">
        <div className="mb-4 flex items-center justify-between">
          <h2 id="modal-title" className="text-lg font-semibold">
            {title}
          </h2>
          <button
            onClick={onClose}
            className="rounded-full p-1 hover:bg-gray-100"
            aria-label="Close modal"
          >
            <XIcon className="h-5 w-5" />
          </button>
        </div>
        {children}
      </div>
    </div>
  )
}
```

### 2. ARIA Attributes ♿

```typescript
// components/ui/Tabs.tsx
interface TabsProps {
  tabs: { id: string; label: string; content: React.ReactNode }[]
  defaultTab?: string
}

export function Tabs({ tabs, defaultTab }: TabsProps) {
  const [activeTab, setActiveTab] = useState(defaultTab ?? tabs[0].id)

  return (
    <div>
      <div role="tablist" className="flex border-b">
        {tabs.map(tab => (
          <button
            key={tab.id}
            role="tab"
            aria-selected={activeTab === tab.id}
            aria-controls={`panel-${tab.id}`}
            id={`tab-${tab.id}`}
            onClick={() => setActiveTab(tab.id)}
            className={cn(
              'px-4 py-2 font-medium',
              activeTab === tab.id
                ? 'border-b-2 border-blue-500 text-blue-600'
                : 'text-gray-500 hover:text-gray-700'
            )}
          >
            {tab.label}
          </button>
        ))}
      </div>
      {tabs.map(tab => (
        <div
          key={tab.id}
          role="tabpanel"
          id={`panel-${tab.id}`}
          aria-labelledby={`tab-${tab.id}`}
          hidden={activeTab !== tab.id}
          className="p-4"
        >
          {tab.content}
        </div>
      ))}
    </div>
  )
}
```

## Testing

### 1. Component Testing 🧪

```typescript
// components/ui/Button.test.tsx
import { render, screen, fireEvent } from '@testing-library/react'
import { Button } from './Button'

describe('Button', () => {
  it('renders with correct text', () => {
    render(<Button>Click me</Button>)
    expect(screen.getByRole('button')).toHaveTextContent('Click me')
  })

  it('calls onClick when clicked', () => {
    const handleClick = jest.fn()
    render(<Button onClick={handleClick}>Click me</Button>)
    fireEvent.click(screen.getByRole('button'))
    expect(handleClick).toHaveBeenCalledTimes(1)
  })

  it('applies variant styles', () => {
    render(<Button variant="secondary">Click me</Button>)
    expect(screen.getByRole('button')).toHaveClass('bg-gray-200')
  })

  it('is disabled when loading', () => {
    render(<Button loading>Click me</Button>)
    expect(screen.getByRole('button')).toBeDisabled()
    expect(screen.getByRole('button')).toHaveTextContent('Loading...')
  })
})
```

### 2. Integration Testing 🔄

```typescript
// components/forms/LoginForm.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react'
import { LoginForm } from './LoginForm'
import { login } from '@/lib/api'

jest.mock('@/lib/api')

describe('LoginForm', () => {
  beforeEach(() => {
    jest.clearAllMocks()
  })

  it('submits form data correctly', async () => {
    const mockLogin = login as jest.Mock
    mockLogin.mockResolvedValueOnce({ user: { id: '1', name: 'John' } })

    render(<LoginForm />)

    fireEvent.change(screen.getByLabelText(/email/i), {
      target: { value: 'john@example.com' },
    })
    fireEvent.change(screen.getByLabelText(/password/i), {
      target: { value: 'password123' },
    })
    fireEvent.click(screen.getByRole('button', { name: /login/i }))

    await waitFor(() => {
      expect(mockLogin).toHaveBeenCalledWith({
        email: 'john@example.com',
        password: 'password123',
        rememberMe: false,
      })
    })
  })

  it('shows validation errors', async () => {
    render(<LoginForm />)

    fireEvent.click(screen.getByRole('button', { name: /login/i }))

    expect(await screen.findByText(/email is required/i)).toBeInTheDocument()
    expect(await screen.findByText(/password is required/i)).toBeInTheDocument()
  })
})
```

## Next Steps

1. Review the [Code Style Guide](./09-code-style-guide.md) for general coding standards
2. Check [Development Workflow](./06-development-workflow.md) for component development process
3. Look at [Testing Strategy](./07-testing-strategy.md) for more testing patterns 