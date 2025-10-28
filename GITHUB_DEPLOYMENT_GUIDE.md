# GitHub & Azure Deployment Guide

This guide walks you through downloading the HR Admin Portal code and deploying it to Azure App Service from GitHub.

## Part 1: Download the Project Code

### Option A: Download as ZIP (Easiest)

1. **Go to the Manus Project Dashboard**
   - Click the "Code" button in the Management UI (right panel)
   - Click "Download all files" button
   - This will download the entire project as a ZIP file

2. **Extract the ZIP file**
   ```bash
   unzip hr_admin_portal.zip
   cd hr_admin_portal
   ```

### Option B: Clone from Manus (If available)

If you have access to the Manus CLI or API, you can clone directly:

```bash
manus-cli clone hr_admin_portal
cd hr_admin_portal
```

## Part 2: Prepare for GitHub

### Step 1: Initialize Git Repository (if not already done)

```bash
cd hr_admin_portal
git init
git config user.name "Your Name"
git config user.email "your.email@example.com"
```

### Step 2: Create .gitignore File

The project should already have a `.gitignore` file, but verify it includes:

```
node_modules/
dist/
build/
.env
.env.local
.env.*.local
*.log
.DS_Store
.idea/
.vscode/
```

**IMPORTANT:** Never commit `.env` files or secrets to GitHub!

### Step 3: Add All Files to Git

```bash
git add .
git commit -m "Initial commit: HR Admin Portal with login and document management"
```

## Part 3: Create GitHub Repository

### Step 1: Create Repository on GitHub

1. Go to [GitHub.com](https://github.com) and sign in
2. Click the **+** icon in the top-right corner
3. Select **New repository**
4. Fill in the details:
   - **Repository name:** `hr-admin-portal`
   - **Description:** "HR Admin Portal for managing HR documents with Azure Blob Storage integration"
   - **Visibility:** Private (recommended for HR data)
   - **Initialize repository:** Leave unchecked (we'll push existing code)
5. Click **Create repository**

### Step 2: Add Remote and Push Code

After creating the repository, GitHub will show you commands. Run these in your terminal:

```bash
# Add the remote repository
git remote add origin https://github.com/YOUR_USERNAME/hr-admin-portal.git

# Rename branch to main (if needed)
git branch -M main

# Push code to GitHub
git push -u origin main
```

**Replace `YOUR_USERNAME` with your actual GitHub username.**

### Step 3: Verify on GitHub

- Go to your repository on GitHub
- You should see all the project files
- Check that `.env` and sensitive files are NOT included

## Part 4: Deploy to Azure App Service

### Prerequisites

- Azure subscription (free tier available)
- Azure CLI installed ([Download](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli))
- GitHub repository created (from Part 3)

### Step 1: Create Azure App Service

#### Using Azure Portal (GUI)

1. Go to [Azure Portal](https://portal.azure.com)
2. Click **Create a resource**
3. Search for **App Service** and click it
4. Click **Create**
5. Fill in the details:
   - **Resource Group:** Create new or select existing
   - **Name:** `hr-admin-portal` (must be globally unique)
   - **Publish:** Code
   - **Runtime stack:** Node.js 20 LTS
   - **Operating System:** Linux
   - **Region:** Choose closest to your location
   - **App Service Plan:** Create new or select existing (B1 or B2 tier recommended)
6. Click **Review + Create** → **Create**

#### Using Azure CLI

```bash
# Login to Azure
az login

# Create resource group
az group create --name hr-portal-rg --location eastus

# Create App Service plan
az appservice plan create \
  --name hr-portal-plan \
  --resource-group hr-portal-rg \
  --sku B2 \
  --is-linux

# Create App Service
az webapp create \
  --resource-group hr-portal-rg \
  --plan hr-portal-plan \
  --name hr-admin-portal \
  --runtime "NODE|20-lts"
```

### Step 2: Configure GitHub Deployment

#### Method A: GitHub Actions (Recommended)

1. In Azure Portal, go to your App Service
2. Navigate to **Deployment Center**
3. Select **GitHub** as source
4. Click **Authorize** and sign in to GitHub
5. Select your repository and branch (main)
6. Click **Save**

Azure will automatically create a GitHub Actions workflow file and deploy on every push.

#### Method B: Manual GitHub Connection

1. In Azure Portal, go to **Deployment Center**
2. Select **GitHub** as source
3. Authorize and select your repository
4. Click **Save**

### Step 3: Configure Environment Variables

1. In Azure Portal, go to your App Service
2. Navigate to **Configuration** → **Application settings**
3. Add the following environment variables:

```
AZURE_STORAGE_ACCOUNT_NAME=your_account_name
AZURE_STORAGE_ACCOUNT_KEY=your_account_key
AZURE_STORAGE_CONTAINER_NAME=your_container_name
NODE_ENV=production
```

**IMPORTANT:** Use your actual Azure Storage credentials here.

4. Click **Save**

### Step 4: Configure Build Settings

1. Still in **Configuration**, go to **General settings**
2. Set:
   - **Stack:** Node.js
   - **Node.js version:** 20 LTS
   - **Startup command:** `npm start` or `pnpm start`

3. Click **Save**

### Step 5: Create Build Script

Create a `package.json` build script if not present. Verify your `package.json` has:

```json
{
  "scripts": {
    "build": "pnpm build",
    "start": "node server/index.js",
    "dev": "pnpm dev"
  }
}
```

### Step 6: Deploy

#### Automatic Deployment (via GitHub Actions)

1. Make a commit and push to GitHub:
   ```bash
   git add .
   git commit -m "Configure for Azure deployment"
   git push
   ```

2. Azure will automatically trigger deployment
3. Monitor progress in Azure Portal → **Deployment Center** → **Logs**

#### Manual Deployment

```bash
# Using Azure CLI
az webapp up --name hr-admin-portal --resource-group hr-portal-rg
```

### Step 7: Verify Deployment

1. Go to Azure Portal → Your App Service
2. Click the **URL** in the overview (looks like `https://hr-admin-portal.azurewebsites.net`)
3. Test the login page and admin portal

## Part 5: Troubleshooting

### Application Won't Start

**Check logs:**
```bash
az webapp log tail --name hr-admin-portal --resource-group hr-portal-rg
```

**Common issues:**
- Missing dependencies: Run `pnpm install` before pushing
- Wrong Node.js version: Check Azure App Service settings
- Missing environment variables: Verify all are set in Configuration

### Build Failures

**Check GitHub Actions logs:**
1. Go to your GitHub repository
2. Click **Actions** tab
3. Click the failed workflow
4. View the logs to see the error

**Common fixes:**
- Install dependencies: `pnpm install`
- Check `pnpm-lock.yaml` is committed
- Verify all required environment variables

### Application Runs But Shows Errors

**Enable detailed logging:**
1. Azure Portal → App Service → **App Service logs**
2. Set **Application logging** to **File System**
3. Set level to **Verbose**
4. Click **Save**
5. Check logs in **Log stream**

## Part 6: Continuous Deployment Setup

### GitHub Actions Workflow

Azure automatically creates a workflow file at `.github/workflows/main_hr-admin-portal.yml`

Example workflow:

```yaml
name: Build and deploy Node.js app to Azure Web App

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Set up Node.js version
        uses: actions/setup-node@v1
        with:
          node-version: '20.x'
      
      - name: npm install and build
        run: |
          npm install -g pnpm
          pnpm install
          pnpm build
      
      - name: Upload artifact for deployment job
        uses: actions/upload-artifact@v2
        with:
          name: node-app
          path: .
      
      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v2
        with:
          app-name: 'hr-admin-portal'
          publish-profile: ${{ secrets.AZURE_PUBLISHPROFILE }}
          package: .
```

## Part 7: Post-Deployment Steps

### 1. Test All Features

- [ ] Login with demo credentials
- [ ] Upload a test document
- [ ] View documents in list
- [ ] Delete a document
- [ ] Logout and login again

### 2. Configure Azure AI Search Integration

Once deployed, configure your Azure AI Search indexer to point to the blob container:

1. Go to Azure Portal → Azure AI Search resource
2. Create an indexer that reads from your blob container
3. Set schedule to daily (as mentioned in your requirements)
4. Test indexing with a sample document

### 3. Set Up Monitoring

1. Azure Portal → App Service → **Application Insights**
2. Enable Application Insights
3. Monitor performance and errors

### 4. Configure Custom Domain (Optional)

1. Azure Portal → App Service → **Custom domains**
2. Add your domain (e.g., `hr-portal.yourdomain.com`)
3. Configure DNS records as instructed

### 5. Enable HTTPS

Azure automatically provides HTTPS with `.azurewebsites.net` domain. For custom domains, Azure provides free SSL certificates.

## Part 8: Updating Your Application

### Making Changes

1. Make changes locally
2. Test with `pnpm dev`
3. Commit and push:
   ```bash
   git add .
   git commit -m "Description of changes"
   git push
   ```
4. Azure automatically deploys the changes

### Rollback to Previous Version

If something breaks:

1. Azure Portal → App Service → **Deployment slots** or **Deployment Center**
2. Select previous successful deployment
3. Click **Swap** or **Redeploy**

## Part 9: Security Checklist

Before going to production:

- [ ] All secrets are in Azure Configuration (not in code)
- [ ] `.env` files are in `.gitignore`
- [ ] GitHub repository is set to Private
- [ ] Azure Storage account has proper access controls
- [ ] HTTPS is enforced
- [ ] Application Insights is enabled for monitoring
- [ ] Database backups are configured (if using database)
- [ ] Regular security updates are planned

## Part 10: Support & Resources

- [Azure App Service Documentation](https://learn.microsoft.com/en-us/azure/app-service/)
- [GitHub Actions for Azure](https://github.com/Azure/actions)
- [Azure CLI Reference](https://learn.microsoft.com/en-us/cli/azure/)
- [Node.js on Azure](https://learn.microsoft.com/en-us/azure/developer/nodejs/)

## Quick Reference Commands

```bash
# Clone your GitHub repo locally
git clone https://github.com/YOUR_USERNAME/hr-admin-portal.git
cd hr-admin-portal

# Install dependencies
pnpm install

# Run locally
pnpm dev

# Build for production
pnpm build

# Deploy to Azure
az webapp up --name hr-admin-portal --resource-group hr-portal-rg

# View logs
az webapp log tail --name hr-admin-portal --resource-group hr-portal-rg

# Update after making changes
git add .
git commit -m "Your message"
git push  # Automatically triggers Azure deployment
```

---

**Next Steps:**
1. Download the project code
2. Create a GitHub repository
3. Push code to GitHub
4. Create Azure App Service
5. Connect GitHub to Azure
6. Set environment variables
7. Deploy and test

Good luck with your deployment! 🚀
