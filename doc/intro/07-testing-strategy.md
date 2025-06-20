# Testing Strategy Guide 🧪

This guide outlines our testing strategy and best practices for ensuring code quality.

## Testing Pyramid 📊

Our testing approach follows the testing pyramid:
1. **Unit Tests** (60%) - Testing individual components and functions
2. **Integration Tests** (30%) - Testing component interactions
3. **End-to-End Tests** (10%) - Testing complete user flows

## Testing Tools 🛠️

### 1. Core Testing Libraries

```json
{
  "devDependencies": {
    "@testing-library/react": "latest",
    "@testing-library/jest-dom": "latest",
    "@testing-library/user-event": "latest",
    "jest": "latest",
    "cypress": "latest"
  }
}
```

### 2. Testing Utilities

```typescript
// test-utils.tsx
import { render as rtlRender } from '@testing-library/react'
import { ThemeProvider } from '@/components/ThemeProvider'

function render(ui: React.ReactElement, options = {}) {
  return rtlRender(ui, {
    wrapper: ({ children }) => (
      <ThemeProvider>{children}</ThemeProvider>
    ),
    ...options,
  })
}

export * from '@testing-library/react'
export { render }
```

## Unit Testing

### 1. Component Testing 🧩

```typescript
// components/Button.test.tsx
import { render, screen, fireEvent } from '@/test-utils'
import Button from './Button'

describe('Button', () => {
  it('renders with correct text', () => {
    render(<Button>Click me</Button>)
    expect(screen.getByRole('button')).toHaveTextContent('Click me')
  })

  it('handles click events', () => {
    const handleClick = jest.fn()
    render(<Button onClick={handleClick}>Click me</Button>)
    fireEvent.click(screen.getByRole('button'))
    expect(handleClick).toHaveBeenCalledTimes(1)
  })

  it('applies custom className', () => {
    render(<Button className="custom-class">Click me</Button>)
    expect(screen.getByRole('button')).toHaveClass('custom-class')
  })
})
```

### 2. Hook Testing 🎣

```typescript
// hooks/useCounter.test.ts
import { renderHook, act } from '@testing-library/react'
import { useCounter } from './useCounter'

describe('useCounter', () => {
  it('initializes with default value', () => {
    const { result } = renderHook(() => useCounter())
    expect(result.current.count).toBe(0)
  })

  it('increments counter', () => {
    const { result } = renderHook(() => useCounter())
    act(() => {
      result.current.increment()
    })
    expect(result.current.count).toBe(1)
  })
})
```

### 3. Utility Function Testing 🔧

```typescript
// utils/format.test.ts
import { formatDate, formatCurrency } from './format'

describe('formatDate', () => {
  it('formats date correctly', () => {
    const date = new Date('2024-01-01')
    expect(formatDate(date)).toBe('01/01/2024')
  })

  it('handles invalid dates', () => {
    expect(formatDate(null)).toBe('Invalid Date')
  })
})
```

## Integration Testing

### 1. Component Integration 🎯

```typescript
// components/UserProfile.test.tsx
import { render, screen, waitFor } from '@/test-utils'
import UserProfile from './UserProfile'
import { UserProvider } from '@/contexts/UserContext'

describe('UserProfile', () => {
  it('loads and displays user data', async () => {
    render(
      <UserProvider>
        <UserProfile userId="123" />
      </UserProvider>
    )

    // Wait for data to load
    await waitFor(() => {
      expect(screen.getByText('Loading...')).not.toBeInTheDocument()
    })

    // Check if user data is displayed
    expect(screen.getByText('John Doe')).toBeInTheDocument()
  })
})
```

### 2. API Integration 🌐

```typescript
// api/users.test.ts
import { rest } from 'msw'
import { setupServer } from 'msw/node'
import { fetchUser } from './users'

const server = setupServer(
  rest.get('/api/users/:id', (req, res, ctx) => {
    return res(
      ctx.json({
        id: req.params.id,
        name: 'John Doe',
      })
    )
  })
)

beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())

describe('fetchUser', () => {
  it('fetches user data', async () => {
    const user = await fetchUser('123')
    expect(user).toEqual({
      id: '123',
      name: 'John Doe',
    })
  })
})
```

## End-to-End Testing

### 1. Cypress Tests 🌟

```typescript
// cypress/e2e/auth.cy.ts
describe('Authentication', () => {
  beforeEach(() => {
    cy.visit('/login')
  })

  it('logs in successfully', () => {
    cy.get('[data-testid="email"]').type('user@example.com')
    cy.get('[data-testid="password"]').type('password123')
    cy.get('[data-testid="submit"]').click()
    
    cy.url().should('include', '/dashboard')
    cy.get('[data-testid="welcome-message"]')
      .should('contain', 'Welcome back')
  })

  it('shows error for invalid credentials', () => {
    cy.get('[data-testid="email"]').type('wrong@example.com')
    cy.get('[data-testid="password"]').type('wrongpass')
    cy.get('[data-testid="submit"]').click()
    
    cy.get('[data-testid="error-message"]')
      .should('be.visible')
      .and('contain', 'Invalid credentials')
  })
})
```

### 2. API E2E Tests 🔄

```typescript
// cypress/e2e/api.cy.ts
describe('API Endpoints', () => {
  it('creates a new user', () => {
    cy.request({
      method: 'POST',
      url: '/api/users',
      body: {
        name: 'New User',
        email: 'new@example.com',
      },
    }).then((response) => {
      expect(response.status).to.eq(201)
      expect(response.body).to.have.property('id')
    })
  })
})
```

## Test Coverage

### 1. Coverage Configuration 📊

```javascript
// jest.config.js
module.exports = {
  collectCoverageFrom: [
    'src/**/*.{js,jsx,ts,tsx}',
    '!src/**/*.d.ts',
    '!src/**/*.stories.{js,jsx,ts,tsx}',
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
}
```

### 2. Coverage Reports 📈

```bash
# Generate coverage report
npm run test:coverage

# View coverage in browser
npm run test:coverage:view
```

## Best Practices

### 1. Test Organization 📁

```
src/
├── components/
│   ├── Button/
│   │   ├── Button.tsx
│   │   ├── Button.test.tsx
│   │   └── Button.stories.tsx
│   └── UserProfile/
│       ├── UserProfile.tsx
│       ├── UserProfile.test.tsx
│       └── UserProfile.stories.tsx
└── __tests__/
    ├── setup.ts
    └── utils/
        └── test-utils.tsx
```

### 2. Testing Principles 🎯

1. **Test Behavior, Not Implementation**
   ```typescript
   // Good
   expect(screen.getByRole('button')).toBeEnabled()
   
   // Bad
   expect(button.props.disabled).toBe(false)
   ```

2. **Use Semantic Queries**
   ```typescript
   // Good
   screen.getByRole('button', { name: 'Submit' })
   
   // Bad
   screen.getByTestId('submit-button')
   ```

3. **Test User Interactions**
   ```typescript
   // Good
   await userEvent.click(screen.getByRole('button'))
   
   // Bad
   fireEvent.click(button)
   ```

### 3. Common Patterns 🔄

1. **Setup and Teardown**
   ```typescript
   describe('Component', () => {
    let mockData: MockData

    beforeEach(() => {
      mockData = createMockData()
    })

    afterEach(() => {
      cleanup()
    })

    it('does something', () => {
      // Test code
    })
   })
   ```

2. **Mocking**
   ```typescript
   // Mock API calls
   jest.mock('@/lib/api', () => ({
     fetchData: jest.fn(() => Promise.resolve(mockData))
   }))

   // Mock hooks
   jest.mock('@/hooks/useAuth', () => ({
     useAuth: () => ({
       user: mockUser,
       login: jest.fn()
     })
   }))
   ```

## Troubleshooting

### Common Issues 🔧

1. **Async Tests**
   ```typescript
   // Use async/await
   it('handles async operations', async () => {
     await waitFor(() => {
       expect(screen.getByText('Data loaded')).toBeInTheDocument()
     })
   })
   ```

2. **Mocking Modules**
   ```typescript
   // Reset mocks between tests
   beforeEach(() => {
     jest.resetAllMocks()
   })
   ```

3. **Testing Context**
   ```typescript
   // Wrap components with providers
   render(
     <ThemeProvider>
       <UserProvider>
         <Component />
       </UserProvider>
     </ThemeProvider>
   )
   ```

## Next Steps

1. Review the [Development Workflow](./06-development-workflow.md) for how testing fits into our process
2. Check [Component Guidelines](./10-component-guidelines.md) for component testing patterns
3. Look at [Code Style Guide](./09-code-style-guide.md) for our coding standards 