# CI/CD Pipeline - Quick Start Checklist

## Pre-Setup Requirements

- [ ] GitHub repository created and cloned
- [ ] Node.js 18+ installed locally
- [ ] Docker installed and running
- [ ] GitHub account with repository access
- [ ] Docker Hub account created
- [ ] Render account created and logged in
- [ ] Supabase project created

---

## Step 1: Repository Setup (5 minutes)

- [ ] Copy `.github/workflows/ci-cd.yml` to your repository
- [ ] Copy `Dockerfile.prod` to your repository root
- [ ] Copy `sonar-project.properties` to your repository root
- [ ] Create `.zap/rules.tsv` directory and file
- [ ] Commit and push these files: `git add . && git commit -m "Add CI/CD pipeline" && git push`

---

## Step 2: External Accounts Setup (30 minutes)

### Docker Hub

- [ ] Log in to https://hub.docker.com
- [ ] Go to Account Settings > Security
- [ ] Create New Access Token
  - Name: `github-actions`
  - Permissions: Read & Write
- [ ] Save the token (you won't see it again)

### Render

- [ ] Go to https://dashboard.render.com
- [ ] Create New Web Service
  - Connect your GitHub repository
  - Branch: `main`
- [ ] In Service Settings:
  - [ ] Copy **Service ID** (format: `srv-xxxxx`)
- [ ] In Account Settings:
  - [ ] Create API Key
  - [ ] Copy the **API Key**
- [ ] Get your **Deployment URL** from the service dashboard

### SonarQube (Choose ONE option)

#### Option A: SonarQube Cloud (Recommended)
- [ ] Go to https://sonarcloud.io
- [ ] Sign in with GitHub
- [ ] Create Organization from your GitHub org
- [ ] Create Project for your repository
- [ ] Go to Account > Security > Generate Tokens
  - Name: `github-actions`
- [ ] Copy the **token**
- [ ] Note the SonarQube Cloud URL: `https://sonarcloud.io`

#### Option B: Self-Hosted SonarQube
- [ ] Run: `docker run -d --name sonarqube -p 9000:9000 sonarqube:latest`
- [ ] Access: http://localhost:9000
- [ ] Create a new project
- [ ] Generate token in Administration > Security > Users > Tokens
- [ ] Note your **server URL** (e.g., http://your-server:9000)

### Supabase

- [ ] Go to https://app.supabase.com
- [ ] Select your project
- [ ] Go to Settings > API
- [ ] Copy:
  - [ ] **Project URL** (`NEXT_PUBLIC_SUPABASE_URL`)
  - [ ] **Anon Key** (`NEXT_PUBLIC_SUPABASE_ANON_KEY`)

---

## Step 3: GitHub Secrets Configuration (10 minutes)

In your GitHub repository:
1. Go to **Settings > Secrets and variables > Actions**
2. Click **New repository secret**
3. Add each secret from the table below:

| Secret Name | Value | Source |
|------------|-------|--------|
| `DOCKER_HUB_USERNAME` | Your Docker Hub username | Docker Hub account |
| `DOCKER_HUB_PASSWORD` | Docker Hub access token | Docker Hub Security page |
| `RENDER_SERVICE_ID` | srv-xxxxx | Render Service Settings |
| `RENDER_API_KEY` | rnd_xxxxx | Render Account Settings |
| `RENDER_DEPLOYMENT_URL` | https://your-app.onrender.com | Render dashboard |
| `NEXT_PUBLIC_SUPABASE_URL` | https://xxxx.supabase.co | Supabase Settings > API |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | eyJhbGc... | Supabase Settings > API |
| `SONAR_HOST_URL` | https://sonarcloud.io | SonarQube instance |
| `SONAR_LOGIN` | squ_xxxx | SonarQube Security > Tokens |

---

## Step 4: Configure Application (20 minutes)

### Add Health Check Endpoint

Create `src/pages/api/health.ts` (or `src/app/api/health/route.ts` for App Router):

```typescript
import type { NextApiRequest, NextApiResponse } from 'next';

export default function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'GET') {
    return res.status(405).json({ error: 'Method not allowed' });
  }
  
  res.status(200).json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
  });
}
```

### Create next.config.ts

Copy from `next.config.example.ts` or use:

```typescript
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  output: 'standalone',
  env: {
    NEXT_PUBLIC_SUPABASE_URL: process.env.NEXT_PUBLIC_SUPABASE_URL,
    NEXT_PUBLIC_SUPABASE_ANON_KEY: process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY,
  },
};

export default nextConfig;
```

### Update package.json

Ensure your `package.json` has these scripts:

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint . --ext .ts,.tsx",
    "test": "jest",
    "test:coverage": "jest --coverage"
  }
}
```

---

## Step 5: Test Locally (15 minutes)

```bash
# 1. Install dependencies
npm ci

# 2. Run tests (if available)
npm run test

# 3. Lint code
npm run lint

# 4. Build application
npm run build

# 5. Build Docker image
docker build -f Dockerfile.prod -t byte-me-app:test .

# 6. Run Docker image
docker run -p 3000:3000 byte-me-app:test

# 7. Test health endpoint (in another terminal)
curl http://localhost:3000/api/health

# Expected response:
# {"status":"healthy","timestamp":"2024-05-17T..."}
```

---

## Step 6: Push to GitHub (Trigger Pipeline)

```bash
# 1. Commit changes
git add .
git commit -m "Setup CI/CD pipeline and health check"

# 2. Push to main (this triggers the pipeline!)
git push origin main
```

---

## Step 7: Monitor Pipeline Execution

1. Go to GitHub repository
2. Click **Actions** tab
3. Select the running workflow
4. Watch real-time execution:
   - ✅ Green = Success
   - ❌ Red = Failed
   - ⏳ Yellow = Running

---

## Step 8: Verify Deployment

After pipeline completes successfully:

```bash
# 1. Check health endpoint
curl https://your-app.onrender.com/api/health

# 2. Check application loads
open https://your-app.onrender.com

# 3. Verify Supabase connection
# - Test authentication
# - Test database queries
# - Check file uploads (if applicable)
```

---

## Common Issues & Quick Fixes

### ❌ Docker login failed
```
✓ Check DOCKER_HUB_PASSWORD is a token, not password
✓ Regenerate token at hub.docker.com/settings/security
✓ Update GitHub secret
```

### ❌ SonarQube Quality Gate Failed
```
✓ Review SonarQube dashboard
✓ Fix code quality issues
✓ Increase test coverage
✓ Reduce code complexity
```

### ❌ Trivy found vulnerabilities
```
✓ Run: npm audit
✓ Run: npm update
✓ Run: npm audit fix
✓ Commit and push
```

### ❌ Deployment verification failed
```
✓ Check Render service logs
✓ Verify health endpoint works: curl https://app-url/api/health
✓ Check environment variables in Render
✓ Check Render deployment history
```

### ❌ OWASP ZAP scan failed
```
✓ Review ZAP findings
✓ Fix security headers
✓ Add security configurations
✓ Re-run pipeline
```

---

## Success Indicators

✅ All stages passed in GitHub Actions
✅ Docker image pushed to Docker Hub
✅ Application deployed to Render
✅ Health check returns 200 OK
✅ Application loads without errors
✅ SonarQube quality gate passed
✅ No CRITICAL vulnerabilities found
✅ OWASP ZAP scan completed

---

## Next Steps

1. **Set up branch protection**
   - Go to Settings > Branches > Add rule
   - Require status checks to pass
   - Require pull request reviews

2. **Configure monitoring**
   - Set up alerts in Render dashboard
   - Configure GitHub notifications
   - Monitor SonarQube metrics

3. **Document deployment process**
   - Create runbook for team
   - Document manual deployment steps
   - Create incident response procedures

4. **Schedule security reviews**
   - Weekly OWASP ZAP review
   - Monthly dependency updates
   - Quarterly security audit

---

## Documentation Files

- 📖 **CI_CD_SETUP_GUIDE.md** - Comprehensive setup guide
- 🔧 **CI_CD_IMPLEMENTATION_GUIDE.md** - Code examples and implementation details
- 📋 **This file** - Quick start checklist

---

## Support

For questions or issues:
1. Check the troubleshooting sections in setup guides
2. Review GitHub Actions logs
3. Check individual service documentation
4. Review OWASP ZAP and Trivy reports

---

**Last Updated:** May 17, 2024
**Pipeline Version:** 1.0.0
**Status:** Production Ready ✅
