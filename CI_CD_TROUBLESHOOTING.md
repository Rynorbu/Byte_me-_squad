# CI/CD Pipeline - Troubleshooting Guide

## Quick Diagnosis

### Pipeline Status Colors
- 🟢 **Green** - All stages passed successfully
- 🔴 **Red** - One or more stages failed
- 🟡 **Yellow** - Pipeline currently running
- ⚪ **Gray** - Queued or not started

### Where to Check

1. **GitHub Actions** → Your repo → Actions tab → Select workflow
2. **Render Dashboard** → Your service → Deployments tab
3. **Docker Hub** → Your repository → Tags tab
4. **SonarQube** → Your project → Overview tab

---

## Common Issues & Solutions

### 1. "Docker login failed: incorrect username/password"

**Symptoms:**
```
ERROR: denied: incorrect username/password
Exit code: 1
```

**Root Causes:**
- ❌ Using Docker Hub password instead of Personal Access Token
- ❌ Token expired or revoked
- ❌ Wrong credentials in GitHub Secrets
- ❌ Token has insufficient permissions

**Solution:**

```bash
# Step 1: Generate new Docker Hub token
1. Go to https://hub.docker.com/settings/security
2. Click "New Access Token"
3. Name it: "github-actions"
4. Select: "Read & Write"
5. Copy the token

# Step 2: Update GitHub Secret
1. Go to Settings > Secrets and variables > Actions
2. Click on "DOCKER_HUB_PASSWORD"
3. Update with the new token
4. Re-run the workflow
```

**Verification:**
```bash
# Test locally
docker login -u YOUR_USERNAME
# Paste the token when prompted
docker logout
```

---

### 2. "SonarQube Quality Gate Failed"

**Symptoms:**
```
ERROR: QUALITY GATE STATUS: FAILED
ERROR: The project did not pass the SonarQube Quality Gate
```

**Root Causes:**
- ❌ Code coverage below 80%
- ❌ Too many code duplications
- ❌ High complexity scores
- ❌ CRITICAL or BLOCKER issues found

**Solution:**

```bash
# Step 1: Review SonarQube dashboard
1. Go to your SonarQube project
2. Look for red indicators (failures)
3. Review the specific issues

# Step 2: Common fixes

# Fix coverage issues
npm run test:coverage

# Reduce code duplication
# - Extract common functions
# - Use utility files
# - Follow DRY principle

# Reduce complexity
# - Break down large functions
# - Simplify if-else chains
# - Use guard clauses

# Fix security hotspots
# - Review flagged lines
# - Add security headers
# - Fix input validation

# Step 3: Commit and push
git add .
git commit -m "Fix SonarQube quality gate issues"
git push origin main
```

**Prevention:**
```typescript
// Use extract function when complexity gets high
// Keep functions focused (single responsibility)
// Add comments for complex logic
// Write unit tests for edge cases
```

---

### 3. "Trivy found CRITICAL vulnerabilities"

**Symptoms:**
```
❌ Critical vulnerabilities found in filesystem scan!
ERROR: Pipeline failed at Trivy Scan
```

**Root Causes:**
- ❌ Outdated npm packages with known CVEs
- ❌ Vulnerable dependencies in package.json
- ❌ Base Docker image has vulnerabilities

**Solution:**

```bash
# Step 1: Identify vulnerable packages
npm audit

# Step 2: Review the output
# Look for CRITICAL and HIGH severity issues

# Step 3: Fix vulnerabilities (choose one approach)

# Option A: Automatic fix (safest)
npm audit fix
npm audit fix --force  # Use with caution

# Option B: Manual update
npm update  # Updates within version constraints
npm install package@latest  # Install latest version

# Option C: Review and decide per package
npm audit
# Then manually update specific packages

# Step 4: Verify fixes
npm audit  # Should show no CRITICAL

# Step 5: Test application
npm run build
npm run test

# Step 6: Commit and push
git add package*.json
git commit -m "Fix npm audit vulnerabilities"
git push origin main
```

**Prevention:**
```bash
# Regular updates
npm update
npm audit fix

# Check before committing
npm audit

# Use Dependabot
# Enable in GitHub Settings > Security > Dependabot
```

**For Docker Image Vulnerabilities:**

```dockerfile
# Update Alpine base image in Dockerfile.prod
# FROM node:18-alpine → FROM node:18.16.0-alpine

RUN apk update && apk upgrade  # Keep packages updated
```

---

### 4. "Deployment verification failed"

**Symptoms:**
```
Attempt 1/5 failed, retrying...
Attempt 2/5 failed, retrying...
...
❌ Deployment verification failed after 5 attempts
```

**Root Causes:**
- ❌ Application not starting on Render
- ❌ Health endpoint not implemented or returning wrong status
- ❌ Port configuration incorrect
- ❌ Environment variables not set

**Solution:**

```bash
# Step 1: Check health endpoint locally
npm run build
npm run start
curl http://localhost:3000/api/health
# Should return: {"status":"healthy",...}

# Step 2: Verify in Docker
docker build -f Dockerfile.prod -t app:test .
docker run -p 3000:3000 app:test
# In another terminal:
curl http://localhost:3000/api/health

# Step 3: Check Render logs
1. Go to https://dashboard.render.com
2. Select your service
3. Click "Logs" tab
4. Look for errors or warnings

# Step 4: Verify environment variables in Render
1. Service Settings > Environment
2. Verify all required variables are set
3. Check NEXT_PUBLIC_SUPABASE_URL and key are present

# Step 5: Create/verify health endpoint
# File: src/pages/api/health.ts
import type { NextApiRequest, NextApiResponse } from 'next';

export default function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'GET') {
    res.status(405).json({ error: 'Method not allowed' });
    return;
  }
  res.status(200).json({ status: 'healthy' });
}

# Step 6: Update Dockerfile health check
# In Dockerfile.prod, ensure:
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/api/health', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"

# Step 7: Commit and push
git add .
git commit -m "Fix health check endpoint"
git push origin main
```

---

### 5. "Docker image build fails"

**Symptoms:**
```
ERROR: failed to solve: ...
Exit code: 1
```

**Root Causes:**
- ❌ npm ci fails (missing dependencies)
- ❌ npm run build fails (build error)
- ❌ Alpine packages missing (python, gcc)
- ❌ Out of disk space

**Solution:**

```bash
# Step 1: Test build locally
npm ci
npm run build
npm audit
npm run test  # if available

# Step 2: Test Docker build
docker build -f Dockerfile.prod -t app:debug . --no-cache --progress=plain

# Step 3: Check Dockerfile
# Verify:
# - FROM image exists
# - All RUN commands succeed
# - File paths are correct
# - Permissions are set correctly

# Step 4: Common fixes
# Add missing packages in Dockerfile.prod:
RUN apk add --no-cache python3 make g++

# Use correct working directory
WORKDIR /app

# Copy files correctly
COPY package*.json ./
COPY . .

# Step 5: Try building again
docker build -f Dockerfile.prod -t app:test .

# Step 6: If successful, push
git add Dockerfile.prod
git commit -m "Fix Docker build issues"
git push origin main
```

---

### 6. "OWASP ZAP scan timeout"

**Symptoms:**
```
OWASP ZAP scan taking too long
Timeout after 30 minutes
```

**Root Causes:**
- ❌ Application very large
- ❌ Slow network connection
- ❌ Application unresponsive
- ❌ Default timeout too short

**Solution:**

```yaml
# In .github/workflows/ci-cd.yml, increase timeout:

- name: Run OWASP ZAP Scan
  uses: zaproxy/action-baseline@v0.7.0
  with:
    target: ${{ secrets.RENDER_DEPLOYMENT_URL }}
    rules_file_name: '.zap/rules.tsv'
    cmd_options: '-a'
    continue_on_error: true
  timeout-minutes: 45  # Increase from 30 to 45

# Alternative: Use quick scan instead
cmd_options: ''  # Removes aggressive scanning
```

---

### 7. "GitHub Secrets not found in workflow"

**Symptoms:**
```
GitHub Actions error: Secret 'DOCKER_HUB_PASSWORD' not found
ERROR: Undefined variable reference
```

**Root Causes:**
- ❌ Secret name typo in workflow
- ❌ Secret not created in GitHub
- ❌ Workflow references wrong repo
- ❌ Secret name case-sensitive

**Solution:**

```bash
# Step 1: Verify secret exists
1. Go to GitHub Settings > Secrets and variables > Actions
2. Look for your secret name
3. Verify it's spelled exactly as in workflow

# Step 2: Check workflow references
In .github/workflows/ci-cd.yml:
- ${{ secrets.DOCKER_HUB_USERNAME }}  ✓ Correct
- ${{ secrets.DOCKER_HUB_PASSWORD }}  ✓ Correct
- ${{ secrets.docker_hub_password }}  ✗ Wrong (case-sensitive)

# Step 3: Create missing secrets
1. Settings > Secrets and variables > Actions
2. "New repository secret"
3. Add each required secret (see CI_CD_QUICK_START.md)

# Step 4: Verify all secrets
Required secrets:
- DOCKER_HUB_USERNAME
- DOCKER_HUB_PASSWORD
- RENDER_SERVICE_ID
- RENDER_API_KEY
- RENDER_DEPLOYMENT_URL
- NEXT_PUBLIC_SUPABASE_URL
- NEXT_PUBLIC_SUPABASE_ANON_KEY
- SONAR_HOST_URL
- SONAR_LOGIN
```

---

### 8. "Port already in use (Docker)"

**Symptoms:**
```
docker: Error response from daemon: driver failed programming external connectivity on endpoint
Bind for 0.0.0.0:3000 failed: port is already allocated
```

**Solution:**

```bash
# Step 1: Find what's using port 3000
# On Windows:
netstat -ano | findstr :3000

# On macOS/Linux:
lsof -i :3000

# Step 2: Kill the process
# Windows (from result above, note the PID):
taskkill /PID <PID> /F

# macOS/Linux:
kill -9 <PID>

# Step 3: Or use different port
docker run -p 3001:3000 app:test
curl http://localhost:3001/api/health
```

---

### 9. "Render deployment URL not working"

**Symptoms:**
```
curl: (7) Failed to connect
Connection refused
Application unreachable
```

**Solution:**

```bash
# Step 1: Get correct URL
1. Go to https://dashboard.render.com
2. Select your service
3. Look for "URL" in the service header
4. Format: https://your-app-name.onrender.com

# Step 2: Test endpoint
curl https://your-app.onrender.com/api/health

# Step 3: Verify in GitHub Secrets
Settings > Secrets > RENDER_DEPLOYMENT_URL
Should be: https://your-app-name.onrender.com
NOT: https://your-app-name.onrender.com/  (no trailing slash)

# Step 4: Update and re-run
1. Fix the URL in GitHub Secrets
2. Go to Actions tab
3. Select the last workflow
4. Click "Re-run all jobs"
```

---

### 10. "Render deployment still in progress after 10 minutes"

**Symptoms:**
```
Deployment status: building
Status: deploying
Time elapsed: 10+ minutes
```

**Solution:**

```bash
# Step 1: Check Render logs
1. Dashboard > Your service > Logs
2. Look for errors in build or deploy logs

# Step 2: Common issues
- Docker build taking too long
- npm install timing out
- Large node_modules

# Step 3: Optimize
# In package.json:
"build": "next build"  # Fast build

# In Dockerfile.prod:
RUN npm ci --only=production  # Faster than npm install

# Step 4: Rebuild Render service
1. Service Settings > Redeploy
2. Click "Manual Deploy"
3. Select latest commit

# Step 5: If still slow, check Render plan
- Free tier: Slower builds
- Paid tier: Faster builds
```

---

## Debug Commands

### Enable Verbose Logging in Workflow

Add this to your workflow:

```yaml
- name: Debug Information
  if: failure()
  run: |
    echo "=== Node Version ==="
    node --version
    
    echo "=== NPM Version ==="
    npm --version
    
    echo "=== Docker Version ==="
    docker --version
    
    echo "=== Package Scripts ==="
    cat package.json | grep -A5 '"scripts"'
    
    echo "=== Directory Structure ==="
    ls -la
    
    echo "=== Build Output ==="
    [ -d .next ] && du -sh .next/
    [ -d dist ] && du -sh dist/
```

### Local Debugging

```bash
# Test each stage locally

# Stage 1-2: Setup and Test
npm ci
npm run lint
npm run test

# Stage 3: Build
npm run build

# Stage 4: Check build output
du -sh .next/
ls -la .next/

# Stage 5: Verify Docker
docker build -f Dockerfile.prod -t app:test . --progress=plain
docker run -p 3000:3000 app:test
# In another terminal:
curl http://localhost:3000/api/health

# Stage 6: Check Docker image size
docker images
docker history app:test
```

---

## Performance Optimization

### Reduce Build Time

```yaml
# In .github/workflows/ci-cd.yml

# Enable GitHub Actions cache
- name: Restore npm cache
  uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}

# Enable Docker layer caching
with:
  cache-from: type=gha
  cache-to: type=gha,mode=max
```

### Reduce Image Size

```dockerfile
# In Dockerfile.prod

# Use minimal Alpine
FROM node:18-alpine

# Clean up after installs
RUN apk add --no-cache python3 make g++ && \
    npm ci && \
    apk del python3 make g++

# Remove dev dependencies
RUN npm ci --only=production
```

---

## Getting Help

### Information to Gather

Before asking for help, collect:
1. **Workflow logs** - Copy full error message
2. **GitHub Actions URL** - Link to failed workflow
3. **Service logs** - Render/Docker/SonarQube logs
4. **Steps to reproduce** - What did you do
5. **Expected vs actual** - What should happen vs what happened

### Where to Get Help

1. **Check this guide** - Most issues are covered
2. **Review CI_CD_SETUP_GUIDE.md** - Detailed explanations
3. **Check tool documentation** - SonarQube, Trivy, OWASP ZAP docs
4. **GitHub Issues** - Search similar issues
5. **Stack Overflow** - Tag relevant tools

---

## Quick Reference

| Issue | File | Solution |
|-------|------|----------|
| Docker login | Workflow | Update DOCKER_HUB_PASSWORD |
| Quality gate fails | Workflow | Fix code quality issues |
| Trivy vulnerabilities | Local | Run `npm audit fix` |
| Deployment fails | Render | Check logs, add health endpoint |
| Secrets not found | Workflow | Verify secret names and spelling |
| Port in use | Local | Kill process or use different port |
| Slow builds | Workflow | Enable caching, optimize Docker |

---

## Success Indicators

✅ All workflow stages show green checkmarks  
✅ Docker image pushed to Docker Hub  
✅ Application deployed to Render  
✅ Health endpoint returns 200 OK  
✅ SonarQube shows quality gate PASSED  
✅ No CRITICAL vulnerabilities in reports  
✅ OWASP ZAP scan completed successfully  

---

**Last Updated:** May 17, 2024  
**Troubleshooting Guide Version:** 1.0.0
