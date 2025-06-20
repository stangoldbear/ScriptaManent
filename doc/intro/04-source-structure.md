# Source Code Structure Guide 📁

This guide explains the organization of our source code in the `src/` directory. Each directory has a specific purpose and follows our project conventions.

## Directory Structure

```
src/
├── app/                    # App Router directory
│   ├── (main)/            # Main route group
│   │   ├── page.tsx       # Home page (/)
│   │   └── layout.tsx     # Main layout
│   ├── (external)/        # External route group
│   │   ├── login/         # Login routes
│   │   └── register/      # Registration routes
│   ├── api/               # API routes
│   ├── layout.tsx         # Root layout (in src/app)
│   └── globals.css        # Global styles
├── components/       # Reusable React components
├── config/          # Application configuration
├── data/            # Data models and types
├── hooks/           # Custom React hooks
├── lib/             # Utility functions and shared logic
├── middleware/      # Next.js middleware and auth
├── navigation/      # Navigation components and logic
└── server/          # Server-side code and API routes
```

## Detailed Directory Explanations

### 1. `app/` Directory 🏠
The main application directory using Next.js 13+ App Router:
- `page.tsx` files define routes
- `layout.tsx` files define shared layouts
- `loading.tsx` for loading states
- `error.tsx` for error handling
- `not-found.tsx` for 404 pages

Example structure:
```
app/
├── (auth)/              # Auth group
│   ├── login/
│   │   └── page.tsx
│   └── register/
│       └── page.tsx
├── (dashboard)/         # Dashboard group
│   ├── layout.tsx
│   └── page.tsx
├── api/                 # API routes
│   └── [...route]/
│       └── route.ts
├── layout.tsx          # Root layout
└── page.tsx            # Home page
```

### 2. `components/` Directory 🧩
Reusable React components organized by feature or type:

```
components/
├── ui/                 # Basic UI components
│   ├── Button.tsx
│   ├── Input.tsx
│   └── Card.tsx
├── forms/              # Form components
│   ├── LoginForm.tsx
│   └── SearchForm.tsx
├── layout/             # Layout components
│   ├── Header.tsx
│   └── Footer.tsx
└── features/           # Feature-specific components
    ├── auth/
    └── dashboard/
```

### 3. `config/` Directory ⚙️
Application configuration and constants:

```
config/
├── constants.ts       # Application constants
├── routes.ts         # Route definitions
└── settings.ts       # App settings
```

### 4. `data/` Directory 📊
Data models, types, and data-related utilities:

```
data/
├── types/            # TypeScript type definitions
│   ├── user.ts
│   └── api.ts
├── models/           # Data models
│   └── User.ts
└── mocks/            # Mock data for development
    └── users.ts
```

### 5. `hooks/` Directory 🎣
Custom React hooks for shared logic:

```
hooks/
├── useAuth.ts        # Authentication hook
├── useForm.ts        # Form handling hook
└── useApi.ts         # API data fetching hook
```

### 6. `lib/` Directory 🛠️
Utility functions and shared logic:

```
lib/
├── api/              # API client and utilities
│   ├── client.ts
│   └── endpoints.ts
├── utils/            # Utility functions
│   ├── format.ts
│   └── validation.ts
└── services/         # Business logic services
    ├── auth.ts
    └── user.ts
```

### 7. `middleware/` Directory 🔒
Next.js middleware and authentication:

```
middleware/
├── auth.ts           # Authentication middleware
└── logging.ts        # Request logging middleware
```

### 8. `navigation/` Directory 🧭
Navigation-related components and logic:

```
navigation/
├── components/       # Navigation components
│   ├── Navbar.tsx
│   └── Sidebar.tsx
└── utils/           # Navigation utilities
    └── routes.ts
```

### 9. `server/` Directory 🌐
Server-side code and API routes:

```
server/
├── api/             # API route handlers
│   └── users/
│       └── route.ts
└── db/              # Database utilities
    └── client.ts
```

## File Naming Conventions

1. **React Components**
   - Use PascalCase: `Button.tsx`, `UserProfile.tsx`
   - Suffix with type: `Button.tsx`, `UserProfile.tsx`

2. **Utility Files**
   - Use camelCase: `formatDate.ts`, `validateEmail.ts`
   - Suffix with purpose: `apiClient.ts`, `authUtils.ts`

3. **Type Definitions**
   - Use PascalCase: `User.ts`, `ApiResponse.ts`
   - Suffix with `.types.ts` for type-only files

4. **Test Files**
   - Match component name: `Button.test.tsx`
   - Use `.spec.ts` for utility tests

## Import Conventions

1. **Absolute Imports**
   ```typescript
   // Use @/ prefix for src directory
   import { Button } from '@/components/ui/Button'
   import { useAuth } from '@/hooks/useAuth'
   ```

2. **Relative Imports**
   ```typescript
   // Use relative paths for closely related files
   import { UserCard } from './UserCard'
   import { types } from '../types'
   ```

3. **Import Order**
   ```typescript
   // 1. External dependencies
   import { useState } from 'react'
   import { useRouter } from 'next/router'
   
   // 2. Internal absolute imports
   import { Button } from '@/components/ui/Button'
   
   // 3. Internal relative imports
   import { styles } from './styles'
   ```

## Best Practices

1. **Component Organization**
   - Keep components small and focused
   - Use feature-based organization
   - Separate business logic from UI

2. **File Structure**
   - One component per file
   - Co-locate related files
   - Use index files for clean exports

3. **Code Splitting**
   - Use dynamic imports for large components
   - Split routes by feature
   - Lazy load non-critical components

4. **Type Safety**
   - Use TypeScript for all files
   - Define proper interfaces
   - Avoid `any` type

## Next Steps

1. Review the [Component Guidelines](./10-component-guidelines.md) for detailed component patterns
2. Check [Development Workflow](./06-development-workflow.md) for how to work with this structure
3. Look at [Code Style Guide](./09-code-style-guide.md) for our coding standards 