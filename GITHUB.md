# GitHub Repository Setup Guide

This guide shows how to create a private GitHub repository from your local project while protecting sensitive information.

## Step 1: Create .gitignore File

Create a `.gitignore` file to protect sensitive data:

```bash
cat > .gitignore << 'EOF'
# Sensitive files
secrets.yaml
dynakube-secrets.yaml
*.pem
*.key

# AWS credentials
.aws/
aws-credentials*

# Environment files
.env
.env.local
.env.production

# Logs
*.log
logs/

# Temporary files
*.tmp
*.temp
.DS_Store
Thumbs.db

# IDE files
.vscode/
.idea/
*.swp
*.swo

# Node modules (if any)
node_modules/

# Python cache (if any)
__pycache__/
*.pyc

# Dynamic status files
AmazonQ.md

# AWS CLI working files
eks-service-role-trust-policy.json
nodegroup-role-trust-policy.json
EOF
```

## Step 2: Initialize Git Repository

```bash
# Initialize git repository
git init

# Check what files will be tracked (sensitive files should NOT appear)
git status

# Add all files
git add .

# Make initial commit
git commit -m "Initial commit: Online Shop Demo microservices deployment on AWS EKS with Dynatrace monitoring"
```

## Step 3: Create Private GitHub Repository

### Option A: Using GitHub CLI (if available)
```bash
# Create private repository
gh repo create online-shop-demo --private --description "Online Shop Demo: Cloud-native microservices e-commerce platform on AWS EKS with Dynatrace observability"

# Push code
git push -u origin main
```

### Option B: Manual Creation
1. Go to https://github.com/new
2. Repository name: `online-shop-demo`
3. Set to **Private**
4. Don't initialize with README (you already have one)
5. Click "Create repository"

Then push your code:
```bash
# Add remote (replace YOUR_USERNAME with your GitHub username)
git remote add origin https://github.com/YOUR_USERNAME/online-shop-demo.git

# Push code
git branch -M main
git push -u origin main
```

## Verification

After pushing, verify that sensitive files are protected:

1. Check your GitHub repository - sensitive files should NOT be visible:
   - `dynakube-secrets.yaml` (Dynatrace credentials)
   - `AmazonQ.md` (dynamic deployment status)
   - `*.pem` files (SSH keys)
   - AWS policy JSON files
2. Verify `.gitignore` is working: `git status` should not show ignored files
3. Confirm repository is private in GitHub settings

## Important Security Notes

- **Never commit dynakube-secrets.yaml** - contains Dynatrace API tokens
- **Never commit AmazonQ.md** - contains dynamic deployment information
- **Never commit *.pem files** - contains SSH private keys
- **Never commit AWS policy files** - may contain account-specific information
- **Always verify .gitignore** before first commit
- **Keep repository private** for security

## Repository Structure

Your repository will contain:
- `README.md` - Complete deployment guide and documentation
- `AGENTS.md` - Agent context and deployment strategies
- `GITHUB.md` - This setup guide
- `.gitignore` - File protection rules
- Deployment scripts and configuration files (non-sensitive)

## Future Updates

To update the repository:
```bash
git add .
git commit -m "Description of changes"
git push
```

The .gitignore will continue protecting sensitive files automatically.

## Project Description

This repository contains deployment scripts and documentation for the Online Shop Demo - a cloud-native microservices e-commerce platform featuring:

- 🏗️ 11 interconnected microservices
- 🚀 AWS EKS Kubernetes deployment
- 📊 Dynatrace full-stack monitoring
- 🛒 Complete e-commerce workflows
- 🤖 Built-in load generation
- 💥 Production-ready architecture

Perfect for demonstrating modern cloud-native applications, microservices architecture, and comprehensive observability solutions.
