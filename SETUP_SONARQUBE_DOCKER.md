# SonarQube & Docker Hub Setup Guide

## PART 1: SonarQube Cloud Setup

### Step 1: Sign Up for SonarQube Cloud (FREE)

1. Go to: https://sonarcloud.io
2. Click **"Sign up"** button (top right)
3. Choose **"GitHub"** to sign up with GitHub
4. You'll be redirected to GitHub - Click **"Authorize SonarSource"**
5. Check your email for verification (if needed)
6. You're now logged into SonarCloud

### Step 2: Create Organization

1. After login, you should see a popup to create organization
2. Click **"Create Organization"**
3. Choose **"Import an organization from GitHub"**
4. You'll see your GitHub organizations/user
5. Click your organization/username
6. Give it a name (can be same as your username)
7. Click **"Continue"**

### Step 3: Create Project

1. Click **"Analyze new project"** or **"Create project"**
2. Select your **GitHub repository** (`Byte_me-_squad`)
3. Click **"Set Up"**
4. You'll be asked to choose a plan - Select **"Free"** (top option)
5. Click **"Create project"**

### Step 4: Generate Authentication Token

1. Go to: https://sonarcloud.io/account/security
   - Or: Click your profile icon (top right) → Account → Security

2. In the **"Tokens"** section, enter a token name:
   ```
   github-actions
   ```

3. Click **"Generate"**

4. **IMPORTANT: Copy the token immediately!** (You won't see it again)
   - Token looks like: `squ_1234567890abcdefghijklmnop`

5. Store it temporarily (we'll use it in next section)

### Step 5: Get Your SonarCloud Organization Key

1. Go to: https://sonarcloud.io/organizations
2. Click your organization
3. Go to **"Organization Settings"** (bottom left)
4. Copy the **"Key"** - looks like: `your-org-name`
5. Note this down (you'll need it)

---

## PART 2: Add Secrets to GitHub

### Step 1: Open GitHub Secrets Settings

1. Go to your GitHub repository: https://github.com/Rynorbu/Byte_me-_squad
2. Click **"Settings"** (top menu)
3. Left sidebar → Click **"Secrets and variables"**
4. Click **"Actions"**

### Step 2: Add SonarQube Secrets

**Add Secret #1: SONAR_HOST_URL**

1. Click **"New repository secret"** (green button)
2. Name: `SONAR_HOST_URL`
3. Value: `https://sonarcloud.io`
4. Click **"Add secret"**

**Add Secret #2: SONAR_LOGIN**

1. Click **"New repository secret"** again
2. Name: `SONAR_LOGIN`
3. Value: **Paste the token you generated above**
   - Example: `squ_1234567890abcdefghijklmnop`
4. Click **"Add secret"**

✅ **SonarQube Secrets Added!**

You should now have these 2 secrets:
```
✓ SONAR_HOST_URL = https://sonarcloud.io
✓ SONAR_LOGIN = squ_xxxxxxxxxxxxxxxxxxxx
```

---

## PART 3: Docker Hub Setup

### Step 1: Create Docker Hub Account (FREE)

1. Go to: https://hub.docker.com
2. Click **"Sign up"** (top right)
3. Enter:
   - Username (memorable, will be in image name)
   - Email
   - Password
4. Check your email and verify
5. You're now logged in to Docker Hub

### Step 2: Create Personal Access Token

1. Log in to Docker Hub: https://hub.docker.com
2. Click your profile icon (top right) → **"Account Settings"**
3. Left sidebar → Click **"Security"**
4. Click **"New Access Token"** (blue button)

**Configure the token:**

- Token name: `github-actions`
- Access Permissions: 
  - Select **"Read & Write"** 
  - ✅ Make sure it includes write permission
  
5. Click **"Generate"**

6. **IMPORTANT: Copy the token!** (You won't see it again)
   - Token looks like: `dckr_pat_abcdefghijklmnopqrstuvwxyz123`

7. **Note your username** (shown in Account Settings)
   - Example: `johndoe`

### Step 3: Add Docker Hub Secrets to GitHub

**Add Secret #3: DOCKER_HUB_USERNAME**

1. Go to GitHub → Settings → Secrets and variables → Actions
2. Click **"New repository secret"**
3. Name: `DOCKER_HUB_USERNAME`
4. Value: **Your Docker Hub username**
   - Example: `johndoe`
5. Click **"Add secret"**

**Add Secret #4: DOCKER_HUB_PASSWORD**

1. Click **"New repository secret"** again
2. Name: `DOCKER_HUB_PASSWORD`
3. Value: **Paste the token you generated**
   - Example: `dckr_pat_xxxxxxxxxxxxxxxxxxxxxxxxxxxx`
4. Click **"Add secret"**

✅ **Docker Hub Secrets Added!**

You should now have these 2 secrets:
```
✓ DOCKER_HUB_USERNAME = johndoe
✓ DOCKER_HUB_PASSWORD = dckr_pat_xxxxxxxxxxxxxxxxxxxx
```

---

## PART 4: Update Supabase Secrets (If Not Done)

### Step 1: Get Supabase Credentials

1. Go to: https://app.supabase.com
2. Select your project: **Byte Me**
3. Click **"Settings"** (bottom left)
4. Click **"API"**
5. Copy these two values:
   - **Project URL** - starts with `https://`
   - **Anon Key** - long key starting with `eyJ...`

### Step 2: Add Supabase Secrets to GitHub

**Add Secret #5: NEXT_PUBLIC_SUPABASE_URL**

1. Go to GitHub Secrets
2. Click **"New repository secret"**
3. Name: `NEXT_PUBLIC_SUPABASE_URL`
4. Value: **Your Supabase Project URL**
   - Example: `https://xxxxxxxxxxxxxx.supabase.co`
5. Click **"Add secret"**

**Add Secret #6: NEXT_PUBLIC_SUPABASE_ANON_KEY**

1. Click **"New repository secret"**
2. Name: `NEXT_PUBLIC_SUPABASE_ANON_KEY`
3. Value: **Your Supabase Anon Key**
   - Starts with `eyJ...`
4. Click **"Add secret"**

---

## PART 5: Render Setup (If Not Done)

### Step 1: Get Render Credentials

1. Go to: https://dashboard.render.com
2. Select your service (or create new one)
3. Go to **"Settings"** tab
4. Copy **Service ID** - looks like: `srv-xxxxxxxxxxxxxxxxx`

### Step 2: Create Render API Key

1. Click your profile icon (top right)
2. Go to **"Account"** or **"Account Settings"**
3. Click **"API Keys"**
4. Click **"Create API Key"**
5. Name it: `github-actions`
6. Copy the key

### Step 3: Add Render Secrets to GitHub

**Add Secret #7: RENDER_SERVICE_ID**

1. Go to GitHub Secrets
2. Click **"New repository secret"**
3. Name: `RENDER_SERVICE_ID`
4. Value: **Your Render Service ID**
   - Example: `srv-xxxxxxxxxxxxxxxxx`
5. Click **"Add secret"**

**Add Secret #8: RENDER_API_KEY**

1. Click **"New repository secret"**
2. Name: `RENDER_API_KEY`
3. Value: **Your Render API Key**
4. Click **"Add secret"**

**Add Secret #9: RENDER_DEPLOYMENT_URL**

1. Click **"New repository secret"**
2. Name: `RENDER_DEPLOYMENT_URL`
3. Value: **Your Render service URL**
   - Example: `https://byte-me-app.onrender.com`
4. Click **"Add secret"**

---

## FINAL VERIFICATION

### Check All Secrets in GitHub

1. Go to GitHub → Settings → Secrets and variables → Actions
2. You should see all 9 secrets:

```
✓ SONAR_HOST_URL
✓ SONAR_LOGIN
✓ DOCKER_HUB_USERNAME
✓ DOCKER_HUB_PASSWORD
✓ NEXT_PUBLIC_SUPABASE_URL
✓ NEXT_PUBLIC_SUPABASE_ANON_KEY
✓ RENDER_SERVICE_ID
✓ RENDER_API_KEY
✓ RENDER_DEPLOYMENT_URL
```

### Verify SonarQube Project

1. Go to https://sonarcloud.io
2. Go to your organization
3. You should see your project listed: `Byte_me-_squad`

### Verify Docker Hub

1. Go to https://hub.docker.com
2. You should see your profile with username

---

## QUICK REFERENCE

### SonarQube Tokens
| Name | Format |
|------|--------|
| SONAR_HOST_URL | https://sonarcloud.io |
| SONAR_LOGIN | squ_xxxxxxxxxxxxxxxxxxxx |

### Docker Hub Tokens
| Name | Format |
|------|--------|
| DOCKER_HUB_USERNAME | your_username |
| DOCKER_HUB_PASSWORD | dckr_pat_xxxxxxxxxxxxxxxxxxxx |

### Important Notes

⚠️ **NEVER share these tokens**
- Treat them like passwords
- Never commit them to GitHub
- GitHub Secrets are encrypted

✅ **Token Security**
- You can regenerate tokens anytime
- Old tokens stop working immediately
- Regenerate if you accidentally share one

🔄 **If You Lose a Token**
- Go to the service and regenerate it
- Update the GitHub Secret
- Pipeline will use the new token on next run

---

## TROUBLESHOOTING

### "SonarQube token not working"
1. Go to https://sonarcloud.io/account/security
2. Check if token still exists (not deleted)
3. Generate a new one if needed
4. Update GitHub Secret with new token

### "Docker Hub login failed"
1. Go to https://hub.docker.com/settings/security
2. Verify token is still active
3. Make sure you used the TOKEN, not your password
4. Generate new token if needed

### "Render deployment failed"
1. Go to https://dashboard.render.com
2. Check service settings
3. Verify API key is still active
4. Copy correct Service ID from settings

---

## NEXT STEPS

1. ✅ Follow all steps above
2. ✅ Verify all 9 secrets are in GitHub
3. ✅ Commit and push to main branch
4. ✅ Go to GitHub Actions tab
5. ✅ Watch pipeline execute
6. ✅ Check SonarQube for analysis results
7. ✅ Check Docker Hub for uploaded image

You're ready to deploy! 🚀
