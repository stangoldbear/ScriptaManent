# Deployment Process Guide 🚀

This guide explains our deployment process and best practices for releasing code to production.

## Deployment Environments 🌍

We use three main environments:
1. **Development** - Local development
2. **Staging** - Pre-production testing
3. **Production** - Live environment

## Deployment Tools 🛠️

### 1. Netlify Configuration

```toml
# netlify.toml
[build]
  command = "npm run build"
  publish = ".next"
  environment = { NODE_VERSION = "18" }

[build.environment]
  NEXT_PUBLIC_API_URL = "https://api.example.com"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200

[context.production.environment]
  NEXT_PUBLIC_API_URL = "https://api.production.com"

[context.staging.environment]
  NEXT_PUBLIC_API_URL = "https://api.staging.com"
```

### 2. Environment Variables

```bash
# .env.production
NEXT_PUBLIC_API_URL=https://api.production.com
DATABASE_URL=postgresql://prod:pass@prod-db:5432/prod

# .env.staging
NEXT_PUBLIC_API_URL=https://api.staging.com
DATABASE_URL=postgresql://staging:pass@staging-db:5432/staging
```

## Deployment Process

### 1. Pre-deployment Checklist ✅

```bash
# 1. Run all tests
npm run test

# 2. Check types
npm run type-check

# 3. Lint code
npm run lint

# 4. Build application
npm run build

# 5. Run production build locally
npm run start
```

### 2. Staging Deployment 🌊

```bash
# 1. Create staging branch
git checkout -b staging

# 2. Update version
npm version patch

# 3. Push to staging
git push origin staging

# 4. Deploy to staging
npm run deploy:staging

# 5. Run staging tests
npm run test:staging
```

### 3. Production Deployment 🚀

```bash
# 1. Merge staging to main
git checkout main
git merge staging

# 2. Update version
npm version minor

# 3. Push to production
git push origin main

# 4. Deploy to production
npm run deploy:prod

# 5. Verify deployment
npm run verify:prod
```

## Deployment Scripts

### 1. Package.json Scripts 📦

```json
{
  "scripts": {
    "deploy:staging": "netlify deploy --env staging",
    "deploy:prod": "netlify deploy --prod",
    "verify:prod": "node scripts/verify-deployment.js",
    "rollback": "node scripts/rollback.js"
  }
}
```

### 2. Verification Script 🔍

```typescript
// scripts/verify-deployment.js
import { execSync } from 'child_process'
import { checkHealth } from './health-check'

async function verifyDeployment() {
  try {
    // Check build status
    const buildStatus = execSync('netlify status').toString()
    console.log('Build status:', buildStatus)

    // Check health endpoints
    await checkHealth({
      api: process.env.NEXT_PUBLIC_API_URL,
      cdn: process.env.NEXT_PUBLIC_CDN_URL,
    })

    // Run smoke tests
    execSync('npm run test:smoke')

    console.log('✅ Deployment verified successfully')
  } catch (error) {
    console.error('❌ Deployment verification failed:', error)
    process.exit(1)
  }
}

verifyDeployment()
```

## Monitoring and Logging

### 1. Health Checks 🏥

```typescript
// pages/api/health.ts
import type { NextApiRequest, NextApiResponse } from 'next'

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  try {
    // Check database connection
    await db.ping()

    // Check external services
    await checkExternalServices()

    res.status(200).json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      version: process.env.APP_VERSION,
    })
  } catch (error) {
    res.status(500).json({
      status: 'unhealthy',
      error: error.message,
    })
  }
}
```

### 2. Error Tracking 🚨

```typescript
// lib/error-tracking.ts
import * as Sentry from '@sentry/nextjs'

export function initErrorTracking() {
  Sentry.init({
    dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
    environment: process.env.NODE_ENV,
    tracesSampleRate: 1.0,
  })
}

export function captureError(error: Error) {
  Sentry.captureException(error)
}
```

## Rollback Process

### 1. Rollback Script 🔄

```typescript
// scripts/rollback.js
import { execSync } from 'child_process'

async function rollback() {
  try {
    // Get previous deployment
    const previousDeploy = execSync(
      'netlify deploy:list --json'
    ).toString()
    const { id } = JSON.parse(previousDeploy)[1]

    // Rollback to previous deployment
    execSync(`netlify deploy:rollback ${id}`)

    console.log('✅ Rollback successful')
  } catch (error) {
    console.error('❌ Rollback failed:', error)
    process.exit(1)
  }
}

rollback()
```

### 2. Emergency Rollback 🚨

```bash
# 1. Stop traffic to new deployment
netlify deploy:lock

# 2. Rollback to previous version
npm run rollback

# 3. Verify rollback
npm run verify:prod

# 4. Unlock deployment
netlify deploy:unlock
```

## Security Measures

### 1. Environment Security 🔒

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  // Add security headers
  const response = NextResponse.next()
  
  response.headers.set('X-Frame-Options', 'DENY')
  response.headers.set('X-Content-Type-Options', 'nosniff')
  response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin')
  response.headers.set(
    'Content-Security-Policy',
    "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline';"
  )

  return response
}
```

### 2. Access Control 🔑

```typescript
// lib/auth.ts
import { getSession } from 'next-auth/react'

export async function requireAuth(req, res, next) {
  const session = await getSession({ req })
  
  if (!session) {
    return res.status(401).json({ error: 'Unauthorized' })
  }
  
  next()
}
```

## Performance Optimization

### 1. Build Optimization ⚡

```javascript
// next.config.mjs
const nextConfig = {
  // Enable static optimization
  output: 'standalone',
  
  // Optimize images
  images: {
    domains: ['cdn.example.com'],
    formats: ['image/avif', 'image/webp'],
  },
  
  // Enable compression
  compress: true,
  
  // Configure caching
  headers: async () => [
    {
      source: '/static/:path*',
      headers: [
        {
          key: 'Cache-Control',
          value: 'public, max-age=31536000, immutable',
        },
      ],
    },
  ],
}

export default nextConfig
```

### 2. Monitoring Performance 📊

```typescript
// lib/performance.ts
export function trackPerformance(metric) {
  // Send to analytics
  analytics.track('performance', {
    name: metric.name,
    value: metric.value,
    rating: metric.rating,
  })

  // Log to monitoring
  if (metric.rating === 'poor') {
    logger.warn('Poor performance detected', metric)
  }
}
```

## Best Practices

1. **Deployment Safety**
   - Always deploy to staging first
   - Run all tests before deployment
   - Have a rollback plan ready
   - Monitor deployment health

2. **Environment Management**
   - Use different environments
   - Keep secrets secure
   - Document all variables
   - Use environment-specific configs

3. **Performance**
   - Optimize build size
   - Enable caching
   - Monitor metrics
   - Set up alerts

4. **Security**
   - Use HTTPS
   - Set security headers
   - Implement rate limiting
   - Monitor for attacks

## Troubleshooting

### Common Issues 🔧

1. **Build Failures**
   ```bash
   # Clear cache and rebuild
   npm run clean
   rm -rf .next
   npm run build
   ```

2. **Deployment Errors**
   ```bash
   # Check deployment logs
   netlify deploy:log
   
   # Verify environment
   netlify env:list
   ```

3. **Performance Issues**
   ```bash
   # Analyze bundle
   npm run analyze
   
   # Check performance
   npm run lighthouse
   ```

## Next Steps

1. Review the [Development Workflow](./06-development-workflow.md) for how deployment fits into our process
2. Check [Testing Strategy](./07-testing-strategy.md) for pre-deployment testing
3. Look at [Code Style Guide](./09-code-style-guide.md) for our coding standards 