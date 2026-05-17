# CI/CD Pipeline Implementation Guide

This file contains implementation details, code examples, and additional setup instructions.

## Table of Contents

1. [Health Check Endpoint](#health-check-endpoint)
2. [API Security Middleware](#api-security-middleware)
3. [Environment Configuration](#environment-configuration)
4. [Testing Setup](#testing-setup)
5. [Build Script Configuration](#build-script-configuration)
6. [Deployment Verification](#deployment-verification)

---

## Health Check Endpoint

The pipeline requires a health check endpoint to verify successful deployment.

### Implementation in Next.js

**File: `src/pages/api/health.ts`**

```typescript
import type { NextApiRequest, NextApiResponse } from 'next';

type ResponseData = {
  status: 'healthy' | 'unhealthy';
  timestamp: string;
  version: string;
  checks: {
    database: 'connected' | 'disconnected';
    cache: 'ok' | 'failed';
  };
};

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse<ResponseData>
) {
  if (req.method !== 'GET') {
    res.status(405).json({
      status: 'unhealthy',
      timestamp: new Date().toISOString(),
      version: process.env.npm_package_version || '1.0.0',
      checks: {
        database: 'disconnected',
        cache: 'failed',
      },
    });
    return;
  }

  try {
    // Check Supabase connection
    const databaseHealthy = await checkSupabaseConnection();
    
    // Check cache/memory
    const cacheHealthy = true; // Add cache check as needed
    
    const overallHealth = databaseHealthy && cacheHealthy;

    res.status(overallHealth ? 200 : 503).json({
      status: overallHealth ? 'healthy' : 'unhealthy',
      timestamp: new Date().toISOString(),
      version: process.env.npm_package_version || '1.0.0',
      checks: {
        database: databaseHealthy ? 'connected' : 'disconnected',
        cache: cacheHealthy ? 'ok' : 'failed',
      },
    });
  } catch (error) {
    console.error('Health check failed:', error);
    res.status(503).json({
      status: 'unhealthy',
      timestamp: new Date().toISOString(),
      version: process.env.npm_package_version || '1.0.0',
      checks: {
        database: 'disconnected',
        cache: 'failed',
      },
    });
  }
}

async function checkSupabaseConnection(): Promise<boolean> {
  try {
    // Attempt a simple query to Supabase
    // This depends on your Supabase setup
    // Example: ping the auth endpoint
    const response = await fetch(
      `${process.env.NEXT_PUBLIC_SUPABASE_URL}/auth/v1/health`,
      {
        method: 'GET',
        timeout: 5000,
      }
    );
    return response.ok;
  } catch (error) {
    console.error('Supabase connection check failed:', error);
    return false;
  }
}
```

### Alternative Implementation (App Router - Next.js 13+)

**File: `src/app/api/health/route.ts`**

```typescript
import { NextResponse } from 'next/server';

export async function GET() {
  try {
    const databaseHealthy = await checkSupabaseConnection();
    const overallHealth = databaseHealthy;

    return NextResponse.json(
      {
        status: overallHealth ? 'healthy' : 'unhealthy',
        timestamp: new Date().toISOString(),
        checks: {
          database: databaseHealthy ? 'connected' : 'disconnected',
        },
      },
      {
        status: overallHealth ? 200 : 503,
      }
    );
  } catch (error) {
    return NextResponse.json(
      {
        status: 'unhealthy',
        timestamp: new Date().toISOString(),
        error: 'Internal server error',
      },
      { status: 503 }
    );
  }
}

async function checkSupabaseConnection(): Promise<boolean> {
  try {
    const response = await fetch(
      `${process.env.NEXT_PUBLIC_SUPABASE_URL}/auth/v1/health`,
      { method: 'GET', signal: AbortSignal.timeout(5000) }
    );
    return response.ok;
  } catch {
    return false;
  }
}
```

---

## API Security Middleware

### CORS Configuration Middleware

**File: `src/middleware.ts`**

```typescript
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const response = NextResponse.next();

  // CORS headers
  response.headers.set('Access-Control-Allow-Origin', process.env.NEXT_PUBLIC_APP_URL || '*');
  response.headers.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
  response.headers.set('Access-Control-Allow-Headers', 'Content-Type, Authorization');
  response.headers.set('Access-Control-Max-Age', '86400');

  // Security headers
  response.headers.set('X-Content-Type-Options', 'nosniff');
  response.headers.set('X-Frame-Options', 'DENY');
  response.headers.set('X-XSS-Protection', '1; mode=block');
  response.headers.set('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
  response.headers.set('Content-Security-Policy', "default-src 'self'");

  // Handle OPTIONS
  if (request.method === 'OPTIONS') {
    return new NextResponse(null, {
      status: 200,
      headers: response.headers,
    });
  }

  return response;
}

export const config = {
  matcher: ['/api/:path*'],
};
```

### Rate Limiting Middleware

**File: `src/lib/rateLimit.ts`**

```typescript
import type { NextApiRequest, NextApiResponse } from 'next';

const store = new Map<string, { count: number; resetTime: number }>();

export function rateLimit(
  maxRequests: number = 100,
  windowMs: number = 60000 // 1 minute
) {
  return (req: NextApiRequest, res: NextApiResponse, next: Function) => {
    const ip = req.headers['x-forwarded-for'] || req.socket.remoteAddress || 'unknown';
    const key = String(ip);

    const now = Date.now();
    const record = store.get(key);

    if (!record || now > record.resetTime) {
      store.set(key, { count: 1, resetTime: now + windowMs });
    } else {
      record.count++;

      if (record.count > maxRequests) {
        res.status(429).json({ error: 'Too many requests' });
        return;
      }
    }

    // Cleanup old entries
    if (store.size > 1000) {
      for (const [k, v] of store.entries()) {
        if (now > v.resetTime) {
          store.delete(k);
        }
      }
    }

    next();
  };
}
```

---

## Environment Configuration

### Local Development (.env.local)

```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxxxxxxxxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# API
NEXT_PUBLIC_API_URL=http://localhost:3000

# Application
NODE_ENV=development
```

### Production (.env.production)

```bash
# Supabase (from GitHub Secrets)
NEXT_PUBLIC_SUPABASE_URL={NEXT_PUBLIC_SUPABASE_URL}
NEXT_PUBLIC_SUPABASE_ANON_KEY={NEXT_PUBLIC_SUPABASE_ANON_KEY}

# API
NEXT_PUBLIC_API_URL=https://your-app.onrender.com

# Application
NODE_ENV=production
```

### GitHub Actions Secrets Required

```
✓ DOCKER_HUB_USERNAME
✓ DOCKER_HUB_PASSWORD
✓ RENDER_SERVICE_ID
✓ RENDER_API_KEY
✓ RENDER_DEPLOYMENT_URL
✓ NEXT_PUBLIC_SUPABASE_URL
✓ NEXT_PUBLIC_SUPABASE_ANON_KEY
✓ SONAR_HOST_URL
✓ SONAR_LOGIN
```

---

## Testing Setup

### Jest Configuration

**File: `jest.config.js`**

```javascript
const nextJest = require('next/jest');

const createJestConfig = nextJest({
  // Provide the path to your Next.js app
  dir: './',
});

// Add any custom config to be passed to Jest
const customJestConfig = {
  setupFilesAfterEnv: ['<rootDir>/jest.setup.js'],
  testEnvironment: 'jest-environment-jsdom',
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
  },
  collectCoverageFrom: [
    'src/**/*.{js,jsx,ts,tsx}',
    '!src/**/*.d.ts',
    '!src/**/*.stories.{js,jsx,ts,tsx}',
    '!src/**/__tests__/**',
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};

module.exports = createJestConfig(customJestConfig);
```

### Jest Setup

**File: `jest.setup.js`**

```javascript
import '@testing-library/jest-dom';

// Mock environment variables
process.env.NEXT_PUBLIC_SUPABASE_URL = 'https://test.supabase.co';
process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY = 'test-key';
```

### Example Test

**File: `src/pages/api/__tests__/health.test.ts`**

```typescript
import { createMocks } from 'node-mocks-http';
import handler from '../health';

describe('/api/health', () => {
  it('returns 200 with healthy status', async () => {
    const { req, res } = createMocks({
      method: 'GET',
    });

    await handler(req, res);

    expect(res._getStatusCode()).toBe(200);
    const data = JSON.parse(res._getData());
    expect(data.status).toBe('healthy');
  });

  it('returns 405 for non-GET requests', async () => {
    const { req, res } = createMocks({
      method: 'POST',
    });

    await handler(req, res);

    expect(res._getStatusCode()).toBe(405);
  });
});
```

### package.json Test Scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "lint": "eslint src --ext .ts,.tsx",
    "lint:fix": "eslint src --ext .ts,.tsx --fix",
    "build": "next build",
    "dev": "next dev"
  }
}
```

---

## Build Script Configuration

### package.json Build Scripts

```json
{
  "name": "byte-me-app",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint src --ext .ts,.tsx",
    "lint:fix": "eslint src --ext .ts,.tsx --fix",
    "test": "jest",
    "test:coverage": "jest --coverage",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "@supabase/auth-helpers-nextjs": "latest",
    "@supabase/supabase-js": "latest",
    "next": "^14.0.0",
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  },
  "devDependencies": {
    "@testing-library/jest-dom": "latest",
    "@testing-library/react": "latest",
    "@types/jest": "latest",
    "@types/node": "latest",
    "@types/react": "latest",
    "eslint": "latest",
    "jest": "latest",
    "jest-environment-jsdom": "latest",
    "typescript": "latest"
  },
  "engines": {
    "node": ">=18.0.0",
    "npm": ">=8.0.0"
  }
}
```

---

## Deployment Verification

### Pre-Deployment Checklist

Before pushing to main:

```bash
# 1. Run tests
npm run test:coverage

# 2. Check types
npm run type-check

# 3. Lint code
npm run lint

# 4. Build application
npm run build

# 5. Build Docker image locally
docker build -f Dockerfile.prod -t byte-me-app:test .

# 6. Run Docker image
docker run -p 3000:3000 byte-me-app:test

# 7. Test health endpoint
curl http://localhost:3000/api/health
```

### Post-Deployment Checklist

After deployment to Render:

```bash
# 1. Check application status
curl https://your-app.onrender.com/api/health

# 2. Verify Supabase connection
# - Check user authentication
# - Verify database queries work
# - Test file uploads if applicable

# 3. Monitor application logs
# - Check Render dashboard logs
# - Look for errors or warnings

# 4. Verify GitHub Actions workflow
# - All stages completed
# - No pipeline failures
# - Artifacts available

# 5. Check security scan results
# - Review OWASP ZAP findings
# - Check Trivy vulnerability reports
# - Verify SonarQube quality gate
```

### Monitoring Script

**File: `scripts/monitor-deployment.sh`**

```bash
#!/bin/bash

set -e

DEPLOYMENT_URL="${1:-https://your-app.onrender.com}"
MAX_RETRIES=10
RETRY_COUNT=0

echo "🔍 Monitoring deployment at $DEPLOYMENT_URL"

while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
  echo "Attempt $((RETRY_COUNT + 1))/$MAX_RETRIES..."
  
  if curl -f -s "$DEPLOYMENT_URL/api/health" > /dev/null 2>&1; then
    echo "✅ Application is healthy!"
    
    # Additional checks
    RESPONSE=$(curl -s "$DEPLOYMENT_URL/api/health")
    echo "Response: $RESPONSE"
    
    exit 0
  fi
  
  RETRY_COUNT=$((RETRY_COUNT + 1))
  
  if [ $RETRY_COUNT -lt $MAX_RETRIES ]; then
    echo "❌ Health check failed, waiting 10 seconds..."
    sleep 10
  fi
done

echo "❌ Deployment verification failed after $MAX_RETRIES attempts"
exit 1
```

---

## Troubleshooting Commands

### Docker Build Issues

```bash
# Build with verbose output
docker build -f Dockerfile.prod -t byte-me-app:debug . --no-cache --progress=plain

# Check image layers
docker history byte-me-app:debug

# Run with debug
docker run -it byte-me-app:debug sh
```

### Next.js Build Issues

```bash
# Clean and rebuild
rm -rf .next node_modules
npm ci
npm run build -- --debug

# Check build output
du -sh .next/
```

### GitHub Actions Debugging

Add to workflow for debugging:

```yaml
- name: Debug Environment
  if: failure()
  run: |
    echo "Node version: $(node --version)"
    echo "NPM version: $(npm --version)"
    echo "Docker version: $(docker --version)"
    echo "Current directory: $(pwd)"
    echo "Directory contents:"
    ls -la
```

---

## Performance Optimization

### Next.js Optimization

```typescript
// next.config.ts
export default {
  swcMinify: true,
  compress: true,
  productionBrowserSourceMaps: false,
  experimental: {
    optimizePackageImports: ['@supabase/supabase-js'],
  },
};
```

### Docker Optimization

In `Dockerfile.prod`:
- ✅ Multi-stage build (reduces image size)
- ✅ Alpine base image (minimal size)
- ✅ Layer caching optimization
- ✅ Non-root user (security)
- ✅ Health checks enabled

### SonarQube Performance

```properties
# sonar-project.properties
sonar.exclusions=node_modules/**,dist/**,.next/**
sonar.pullrequest.github.repository=your-org/repo
```

---

## Support Resources

- 📚 [Next.js Documentation](https://nextjs.org/docs)
- 🐳 [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- 🔒 [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- 📊 [SonarQube Rules](https://rules.sonarsource.com)
- 🛡️ [Trivy Vulnerability DB](https://github.com/aquasecurity/trivy)
- 🚀 [Render Deployment Docs](https://render.com/docs)
