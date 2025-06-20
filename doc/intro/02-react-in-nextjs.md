# React in Next.js Guide ⚛️

This guide explains how we use React in our Next.js application.

## Server vs Client Components

### Server Components (Our Implementation)
```typescript
// src/app/(main)/page.tsx
import { getData } from '@/lib/data'

export default async function HomePage() {
  const data = await getData()
  return (
    <main>
      <h1>{data.title}</h1>
      <p>{data.description}</p>
    </main>
  )
}
```

### Client Components (Our Implementation)
```typescript
// src/components/ui/button.tsx
'use client'

import { useState } from 'react'

export function Button({ children, onClick }: { children: React.ReactNode; onClick?: () => void }) {
  const [isLoading, setIsLoading] = useState(false)
  
  const handleClick = async () => {
    if (onClick) {
      setIsLoading(true)
      await onClick()
      setIsLoading(false)
    }
  }

  return (
    <button 
      onClick={handleClick}
      disabled={isLoading}
      className="px-4 py-2 bg-blue-500 text-white rounded"
    >
      {isLoading ? 'Loading...' : children}
    </button>
  )
}
```

### Generic Examples (Not Part of Our Project)
Below are some generic examples to illustrate React patterns in Next.js:

```typescript
// This is a generic example, not part of our project
// Example of a product listing page
// src/app/(main)/products/page.tsx
import { ProductList } from '@/components/products'
import { getProducts } from '@/lib/products'

export default async function ProductsPage() {
  const products = await getProducts()
  return <ProductList products={products} />
}

// This is a generic example, not part of our project
// Example of a product detail page with client-side interactivity
// src/app/(main)/products/[id]/page.tsx
'use client'

import { useState } from 'react'
import { useCart } from '@/hooks/useCart'

export default function ProductPage({ params }: { params: { id: string } }) {
  const [quantity, setQuantity] = useState(1)
  const { addToCart } = useCart()

  return (
    <div>
      <h1>Product {params.id}</h1>
      <input 
        type="number" 
        value={quantity} 
        onChange={(e) => setQuantity(Number(e.target.value))} 
      />
      <button onClick={() => addToCart(params.id, quantity)}>
        Add to Cart
      </button>
    </div>
  )
}
```

## State Management

### React Query (Our Implementation)
```typescript
// src/hooks/useData.ts
import { useQuery } from '@tanstack/react-query'
import { fetchData } from '@/lib/api'

export function useData() {
  return useQuery({
    queryKey: ['data'],
    queryFn: fetchData,
  })
}

// src/app/(main)/page.tsx
import { useData } from '@/hooks/useData'

export default function HomePage() {
  const { data, isLoading, error } = useData()

  if (isLoading) return <div>Loading...</div>
  if (error) return <div>Error loading data</div>

  return (
    <main>
      <h1>{data.title}</h1>
    </main>
  )
}
```

### Context API (Our Implementation)
```typescript
// src/context/theme.tsx
'use client'

import { createContext, useContext, useState } from 'react'

type Theme = 'light' | 'dark'

const ThemeContext = createContext<{
  theme: Theme
  toggleTheme: () => void
}>({
  theme: 'light',
  toggleTheme: () => {},
})

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light')

  const toggleTheme = () => {
    setTheme(prev => prev === 'light' ? 'dark' : 'light')
  }

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  )
}

export const useTheme = () => useContext(ThemeContext)
```

### Generic Examples (Not Part of Our Project)
Below are some generic examples to illustrate state management patterns:

```typescript
// This is a generic example, not part of our project
// Example of a shopping cart context
// src/context/cart.tsx
'use client'

import { createContext, useContext, useReducer } from 'react'

type CartItem = {
  id: string
  quantity: number
}

type CartState = {
  items: CartItem[]
}

type CartAction = 
  | { type: 'ADD_ITEM'; payload: CartItem }
  | { type: 'REMOVE_ITEM'; payload: string }
  | { type: 'UPDATE_QUANTITY'; payload: { id: string; quantity: number } }

const CartContext = createContext<{
  state: CartState
  dispatch: React.Dispatch<CartAction>
}>({
  state: { items: [] },
  dispatch: () => {},
})

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'ADD_ITEM':
      return {
        ...state,
        items: [...state.items, action.payload],
      }
    // ... other cases
    default:
      return state
  }
}

export function CartProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(cartReducer, { items: [] })

  return (
    <CartContext.Provider value={{ state, dispatch }}>
      {children}
    </CartContext.Provider>
  )
}

export const useCart = () => useContext(CartContext)
```

## Custom Hooks

We create custom hooks for reusable logic:

```typescript
// src/hooks/useForm.ts
import { useForm as useHookForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

export function useForm<T extends z.ZodType>(schema: T) {
  return useHookForm<z.infer<T>>({
    resolver: zodResolver(schema),
    mode: 'onChange',
  })
}

// Usage
const productSchema = z.object({
  name: z.string().min(1),
  price: z.number().positive(),
  description: z.string().optional(),
})

function ProductForm() {
  const form = useForm(productSchema)
  
  return (
    <form onSubmit={form.handleSubmit(data => console.log(data))}>
      {/* Form fields */}
    </form>
  )
}
```

## Component Patterns

### Compound Components
```typescript
// src/components/ui/Accordion.tsx
'use client'

import { createContext, useContext, useState } from 'react'

type AccordionContextType = {
  isOpen: boolean
  toggle: () => void
}

const AccordionContext = createContext<AccordionContextType | null>(null)

export function Accordion({ children }: { children: React.ReactNode }) {
  const [isOpen, setIsOpen] = useState(false)

  return (
    <AccordionContext.Provider value={{ isOpen, toggle: () => setIsOpen(!isOpen) }}>
      <div className="border rounded-lg">{children}</div>
    </AccordionContext.Provider>
  )
}

export function AccordionTrigger({ children }: { children: React.ReactNode }) {
  const context = useContext(AccordionContext)
  if (!context) throw new Error('AccordionTrigger must be used within Accordion')

  return (
    <button
      onClick={context.toggle}
      className="w-full p-4 text-left font-medium"
    >
      {children}
    </button>
  )
}

export function AccordionContent({ children }: { children: React.ReactNode }) {
  const context = useContext(AccordionContext)
  if (!context) throw new Error('AccordionContent must be used within Accordion')

  if (!context.isOpen) return null

  return <div className="p-4 border-t">{children}</div>
}

// Usage
function Example() {
  return (
    <Accordion>
      <AccordionTrigger>Click me</AccordionTrigger>
      <AccordionContent>Content here</AccordionContent>
    </Accordion>
  )
}
```

### Higher-Order Components
```typescript
// src/components/hoc/withAuth.tsx
'use client'

import { useSession } from 'next-auth/react'
import { useRouter } from 'next/navigation'
import { useEffect } from 'react'

export function withAuth<P extends object>(
  WrappedComponent: React.ComponentType<P>
) {
  return function WithAuth(props: P) {
    const { data: session, status } = useSession()
    const router = useRouter()

    useEffect(() => {
      if (status === 'unauthenticated') {
        router.push('/login')
      }
    }, [status, router])

    if (status === 'loading') {
      return <div>Loading...</div>
    }

    if (!session) {
      return null
    }

    return <WrappedComponent {...props} />
  }
}

// Usage
const ProtectedPage = withAuth(function Page() {
  return <div>Protected Content</div>
})
```

## Best Practices

1. **Server Components by Default**
   - Use Server Components unless you need client-side interactivity
   - Keep client-side JavaScript minimal

2. **State Management**
   - Use React Query for server state
   - Use Context for UI state
   - Keep state as local as possible

3. **Performance**
   - Memoize expensive computations
   - Use proper dependency arrays
   - Implement proper loading states

4. **Type Safety**
   - Use TypeScript for all components
   - Define proper interfaces and types
   - Use Zod for runtime validation

## Next Steps

1. Review [Component Guidelines](./10-component-guidelines.md) for more component patterns
2. Check [Performance Guidelines](./13-performance-guidelines.md) for optimization tips
3. Read [Code Style Guide](./09-code-style-guide.md) for coding standards

Remember: While Next.js adds powerful features, all React concepts still apply. We just use them in a way that's optimized for Next.js and our specific needs. 