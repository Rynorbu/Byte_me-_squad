# Complete CI/CD Pipeline Setup Guide

This guide walks you through setting up and configuring the complete CI/CD pipeline for your Next.js + Supabase application.

## Table of Contents

1. [Pipeline Overview](#pipeline-overview)
2. [Prerequisites](#prerequisites)
3. [GitHub Secrets Configuration](#github-secrets-configuration)
4. [SonarQube Setup](#sonarqube-setup)
5. [Docker Hub Configuration](#docker-hub-configuration)
6. [Render Deployment Setup](#render-deployment-setup)
7. [OWASP ZAP Configuration](#owasp-zap-configuration)
8. [Trivy Vulnerability Scanning](#trivy-vulnerability-scanning)
9. [Running the Pipeline](#running-the-pipeline)
10. [Monitoring and Troubleshooting](#monitoring-and-troubleshooting)

---

## Pipeline Overview

The CI/CD pipeline executes 10 stages in order:

```
1. Setup Environment
   ↓
2. Install & Test
   ↓
3. Build Next.js Application
   ↓
4. SonarQube Code Quality Analysis ⚠️ (Fails if quality gate fails)
   ↓
5. Trivy Filesystem Scan ⚠️ (Fails if critical vulnerabilities found)
   ↓
6. Build Docker Image
   ↓
7. Trivy Docker Image Scan ⚠️ (Fails if critical vulnerabilities found)
   ↓
8. Push Docker Image (Only on main branch)
   ↓
9. Deploy to Render (Only on main branch)
   ↓
10. OWASP ZAP Dynamic Security Scan
```

### Pipeline Protection Points
- ⚠️ SonarQube Quality Gate must pass
- ⚠️ No CRITICAL vulnerabilities in filesystem (Trivy)
- ⚠️ No CRITICAL vulnerabilities in Docker image (Trivy)
- ✅ Security recommendations from OWASP ZAP are reported but non-blocking

---

## Prerequisites

### Local Requirements
- Node.js 18+ installed
- Docker installed and running
- Git configured
- GitHub account with repository access

### Repository Requirements
- `.github/workflows/ci-cd.yml` - Main workflow file (provided)
- `Dockerfile.prod` - Production Dockerfile (provided)
- `package.json` with build scripts
- `next.config.js` or `next.config.ts` (if using custom Next.js config)

### Required Configuration Files

#### `next.config.ts` (Create if doesn't exist)
```typescript
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  output: 'standalone', // Required for Docker deployment
  // Supabase environment variables are injected at build time
  env: {
    NEXT_PUBLIC_SUPABASE_URL: process.env.NEXT_PUBLIC_SUPABASE_URL,
    NEXT_PUBLIC_SUPABASE_ANON_KEY: process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY,
  },
};

export default nextConfig;
```

#### `sonar-project.properties` (Create in repo root)
```properties
sonar.projectKey=byte-me-app
sonar.projectName=Byte Me Application
sonar.projectVersion=1.0.0
sonar.sources=src
sonar.tests=src/__tests__
sonar.test.inclusions=**/*.test.ts,**/*.test.tsx,**/*.spec.ts,**/*.spec.tsx
sonar.exclusions=node_modules/**,dist/**,.next/**
sonar.typescript.lcov.reportPaths=coverage/lcov.info
```

#### `.zap/rules.tsv` (Create for OWASP ZAP baseline configuration)
```tsv
10020	WARN
10021	WARN
10023	WARN
10024	WARN
10025	WARN
10026	WARN
10027	WARN
10028	WARN
10029	WARN
10030	WARN
10038	WARN
10039	WARN
10040	WARN
10041	WARN
10042	WARN
10043	WARN
10044	WARN
10045	WARN
10046	WARN
10047	WARN
10048	WARN
10049	WARN
10050	WARN
10051	WARN
10052	WARN
10053	WARN
10054	WARN
10055	WARN
10056	WARN
10057	WARN
10058	WARN
10059	WARN
10060	WARN
10061	WARN
10062	WARN
10063	WARN
10070	WARN
10071	WARN
10094	WARN
10095	WARN
10096	WARN
10097	WARN
10098	WARN
10099	WARN
10202	WARN
```

---

## GitHub Secrets Configuration

Add these secrets to your GitHub repository at **Settings > Secrets and variables > Actions > New repository secret**:

### Required Secrets

#### 1. Docker Hub Credentials
```
DOCKER_HUB_USERNAME = <your-docker-hub-username>
DOCKER_HUB_PASSWORD = <your-docker-hub-personal-access-token>
```
**How to generate Docker Hub token:**
1. Go to https://hub.docker.com/settings/security
2. Click "New Access Token"
3. Name it "github-actions"
4. Select "Read & Write" permissions
5. Copy and save the token

#### 2. Render Deployment
```
RENDER_SERVICE_ID = <your-render-service-id>
RENDER_API_KEY = <your-render-api-key>
RENDER_DEPLOYMENT_URL = https://your-app.onrender.com
```
**How to get Render credentials:**
1. Go to https://dashboard.render.com
2. Select your service > Settings > Copy Service ID
3. Go to Account Settings > API Keys > Create New Key
4. The deployment URL is shown in your Render service dashboard

#### 3. Supabase Environment Variables
```
NEXT_PUBLIC_SUPABASE_URL = <your-supabase-project-url>
NEXT_PUBLIC_SUPABASE_ANON_KEY = <your-supabase-anon-key>
```
**How to get Supabase credentials:**
1. Go to https://app.supabase.com
2. Select your project > Settings > API
3. Copy the Project URL and anon key

#### 4. SonarQube Configuration
```
SONAR_HOST_URL = https://your-sonarqube-instance.com
SONAR_LOGIN = <your-sonarqube-token>
```

---

## SonarQube Setup

### Option A: SonarQube Cloud (Recommended for GitHub)

1. **Sign up for SonarQube Cloud**
   - Go to https://sonarcloud.io
   - Sign in with GitHub
   - Authorize SonarQube Cloud

2. **Create Organization**
   - Click "Create Organization"
   - Import from GitHub
   - Select your repository organization

3. **Create Project**
   - Select "Create a new project"
   - Choose your repository
   - Select GitHub Actions as CI

4. **Generate Token**
   - Go to Account > Security > Generate Tokens
   - Name it "github-actions"
   - Copy the token to GitHub Secrets as `SONAR_LOGIN`

5. **Configure Project**
   - In your SonarQube project, set Quality Gate to "Sonar way"
   - Configure rules in the project settings

### Option B: Self-Hosted SonarQube

1. **Install SonarQube**
   ```bash
   docker run -d --name sonarqube -p 9000:9000 sonarqube:latest
   ```

2. **Login and Configure**
   - Access http://localhost:9000
   - Default credentials: admin/admin
   - Create new project

3. **Generate Token**
   - Administration > Security > Users > Tokens
   - Generate new token for CI/CD

### SonarQube Quality Gate Configuration

**Default Quality Gate Requirements:**
- Code coverage > 80%
- Duplicated lines < 3%
- Cyclomatic complexity < 15
- Cognitive complexity < 20
- No CRITICAL or BLOCKER issues

To adjust: Project Settings > Quality Gate > Edit

---

## Docker Hub Configuration

### 1. Create Docker Hub Account
- Go to https://hub.docker.com
- Sign up and verify email

### 2. Create Access Token
```bash
# Via web UI:
1. Navigate to Account Settings > Security
2. Click "New Access Token"
3. Name: "github-actions"
4. Permissions: Read & Write
```

### 3. Create Repository
```bash
# Via Docker CLI:
docker tag byte-me-app:latest <your-username>/byte-me-app:latest
docker push <your-username>/byte-me-app:latest
```

### 4. GitHub Secrets
- `DOCKER_HUB_USERNAME` = Your Docker Hub username
- `DOCKER_HUB_PASSWORD` = Your access token (NOT your password)

---

## Render Deployment Setup

### 1. Create Render Service
1. Go to https://dashboard.render.com
2. Click "New +" > "Web Service"
3. Connect your GitHub repository
4. Configure:
   - **Name:** byte-me-app
   - **Environment:** Docker
   - **Branch:** main
   - **Build Command:** `docker build -f Dockerfile.prod -t app .`
   - **Start Command:** `docker run -p 3000:3000 app`

### 2. Set Environment Variables
In Render dashboard > Environment:
```
NODE_ENV=production
NEXT_PUBLIC_SUPABASE_URL=<your-supabase-url>
NEXT_PUBLIC_SUPABASE_ANON_KEY=<your-supabase-key>
```

### 3. Get Service ID
- Service Settings > Copy Service ID

### 4. Generate API Key
- Account Settings > API Keys > Create New Key

### 5. Configure Deployment
- Service Settings > Deploy Hook
- Copy the deploy hook URL

### 6. GitHub Secrets
```
RENDER_SERVICE_ID = srv-xxxxxxxxxxxxx
RENDER_API_KEY = rnd_xxxxxxxxxxxxx
RENDER_DEPLOYMENT_URL = https://byte-me-app.onrender.com
```

---

## OWASP ZAP Configuration

### How It Works
- OWASP ZAP performs active security scanning on the deployed application
- It automatically generates security reports
- Reports are published as GitHub PR comments
- Findings include:
  - SQL Injection risks
  - XSS vulnerabilities
  - CSRF issues
  - Authentication/Session issues
  - Security Headers analysis

### Configuration in Workflow
The workflow uses `zaproxy/action-baseline@v0.7.0` with:
- Target: Your Render deployment URL
- Mode: Baseline (full scan)
- Options: `-a` (all rules)

### Custom Rules
Edit `.zap/rules.tsv` to control alert levels:
- **PASS**: Suppress alert
- **WARN**: Report as warning
- **FAIL**: Report as failure (stops pipeline)

### Interpreting Results
Common findings and remediation:
- **Missing Security Headers** → Add to Next.js headers config
- **Weak TLS Configuration** → Render handles this
- **Insecure Authentication** → Review Supabase auth setup
- **CORS Issues** → Configure Next.js CORS middleware

---

## Trivy Vulnerability Scanning

### What Trivy Scans

**Filesystem Scan:**
- Dependencies in package.json
- Known CVEs in npm packages
- OS-level vulnerabilities in Alpine base image

**Docker Image Scan:**
- Container layers
- Base image vulnerabilities
- Dependencies inside the image

### How Failures Work

Pipeline fails if **CRITICAL** vulnerabilities are found:
```yaml
# Automatic actions:
1. Scan completes
2. Results uploaded to GitHub Security tab
3. If CRITICAL found → Pipeline fails
4. PR blocked from merging
```

### Fixing Vulnerabilities

**For Dependencies:**
```bash
# Find vulnerable packages
npm audit

# Update packages
npm update
npm audit fix
```

**For Base Image Issues:**
```bash
# Switch to newer Alpine version
# In Dockerfile.prod:
FROM node:18-alpine  # Update to latest Alpine tag
```

**For Container Vulnerabilities:**
```dockerfile
# Apply security patches in Dockerfile
RUN apk update && apk upgrade
```

---

## Running the Pipeline

### Trigger Methods

**1. Automatic (on push to main):**
```bash
git push origin main
```

**2. Manual Trigger:**
- Go to GitHub Actions > Workflow name > Run workflow

**3. On Pull Requests:**
- Pipeline runs automatically (without deployment steps)

### Monitoring Pipeline Execution

1. **GitHub Actions Tab**
   - Go to your repository
   - Click Actions tab
   - Select the running workflow
   - Watch real-time logs

2. **Workflow Dashboard**
   - Shows each job status
   - Click on job to see detailed logs
   - Artifacts available for download

3. **PR Status Checks**
   - PR shows all checks
   - Red X = failure
   - Green checkmark = passed

### Typical Execution Time
- Stage 1-4: ~5-10 minutes
- Stage 5-7: ~5-15 minutes (scanning)
- Stage 8-10: ~10-15 minutes (deployment + security scan)
- **Total: 20-40 minutes**

---

## Monitoring and Troubleshooting

### Common Issues and Solutions

#### 1. "Docker login failed"
```
ERROR: denied: incorrect username/password

Solution:
1. Check Docker Hub credentials in GitHub Secrets
2. Verify using: docker login -u <username>
3. Regenerate Personal Access Token if needed
4. Update GitHub Secret
```

#### 2. "SonarQube Quality Gate Failed"
```
ERROR: QUALITY GATE STATUS: FAILED

Solution:
1. Fix code quality issues (review SonarQube dashboard)
2. Increase code coverage with tests
3. Reduce code duplication
4. Lower complexity scores
5. Fix security hotspots
```

#### 3. "Trivy found CRITICAL vulnerabilities"
```
ERROR: Critical vulnerabilities found in filesystem scan!

Solution:
1. Run: npm audit
2. Update vulnerable packages: npm update
3. Review: npm audit fix --force (use carefully)
4. Commit and push changes
```

#### 4. "Render deployment failed"
```
ERROR: Failed to verify deployment

Solution:
1. Check Render Service logs (Render dashboard)
2. Verify RENDER_SERVICE_ID is correct
3. Check Render API Key expiration
4. Ensure environment variables are set
5. Check application health endpoint /api/health
```

#### 5. "Health check timeout"
```
ERROR: Application health check failed

Solution:
1. Implement /api/health endpoint (Next.js):
   // pages/api/health.ts
   export default function handler(req, res) {
     res.status(200).json({ status: 'ok' });
   }
2. Check application startup logs
3. Verify PORT environment variable (should be 3000)
```

### Debug Mode

To add verbose logging, update workflow:
```yaml
- name: Debug Info
  if: failure()
  run: |
    echo "=== Environment ===" 
    node --version
    npm --version
    docker --version
    echo "=== Package ===" 
    cat package.json | grep -A2 scripts
```

### Artifact Downloads

After pipeline runs:
1. Go to Actions > Workflow run
2. Scroll to "Artifacts"
3. Download:
   - `test-results` - Coverage reports
   - `zap-report` - Security findings
   - `docker-image` - Built image

---

## Next Steps

### 1. Update Next.js Configuration
```bash
# Create/update next.config.ts with:
- output: 'standalone'
- Environment variable configuration
- CORS setup if needed
```

### 2. Configure Authentication
```bash
# Set Supabase auth environment:
export NEXT_PUBLIC_SUPABASE_URL=your_url
export NEXT_PUBLIC_SUPABASE_ANON_KEY=your_key
```

### 3. Add Health Check Endpoint
```typescript
// src/pages/api/health.ts
export default function handler(req, res) {
  res.status(200).json({ status: 'healthy', timestamp: new Date() });
}
```

### 4. Add Security Headers
```typescript
// next.config.ts or middleware
const securityHeaders = [
  {
    key: 'X-Content-Type-Options',
    value: 'nosniff'
  },
  {
    key: 'X-Frame-Options',
    value: 'DENY'
  },
  {
    key: 'X-XSS-Protection',
    value: '1; mode=block'
  }
];
```

### 5. Monitor in Production
- **GitHub Actions:** Check workflow runs
- **SonarQube:** Review quality metrics
- **Render:** Monitor application performance
- **Docker Hub:** Track image builds

---

## Security Best Practices

1. **Rotate Secrets Regularly**
   - Change API keys every 90 days
   - Regenerate access tokens quarterly

2. **Use Branch Protection**
   - Require PR reviews before merge
   - Require status checks to pass
   - Dismiss stale PR approvals

3. **Monitor Deployments**
   - Check logs for errors
   - Monitor application performance
   - Set up alerts for failures

4. **Review Security Findings**
   - Check OWASP ZAP reports
   - Fix high/critical issues immediately
   - Track CVE updates

5. **Secure Environment Variables**
   - Never commit secrets
   - Use GitHub Secrets only
   - Rotate credentials regularly

---

## Pipeline Metrics and KPIs

Track these metrics to measure pipeline health:

| Metric | Target |
|--------|--------|
| Build Success Rate | > 95% |
| Code Coverage | > 80% |
| SonarQube Quality Gate | PASSED |
| Deployment Time | < 20 minutes |
| Critical Vulnerabilities | 0 |
| High Vulnerabilities | < 3 |

---

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [SonarQube Quality Gates](https://docs.sonarqube.org/latest/user-guide/quality-gates/)
- [Trivy Vulnerability Scanner](https://github.com/aquasecurity/trivy)
- [OWASP ZAP](https://www.zaproxy.org/)
- [Next.js Docker Deployment](https://nextjs.org/docs/deployment/docker)
- [Render Documentation](https://render.com/docs)

---

## Support and Questions

For pipeline issues:
1. Check GitHub Actions logs
2. Review this guide's troubleshooting section
3. Check individual service documentation
4. Open GitHub issue with logs attached
