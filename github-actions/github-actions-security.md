# GitHub Actions Security Configuration Guide

![GitHub Actions Security](https://github.blog/wp-content/uploads/2019/08/DL-V2-LinkedIn_FB.png)

Comprehensive guide for securing GitHub Actions workflows, repositories, and CI/CD pipelines.

## Table of Contents
1. [Introduction](#1-introduction)
2. [Repository Security](#2-repository-security)
3. [Workflow Security](#3-workflow-security)
4. [Secrets Management](#4-secrets-management)
5. [Access Control](#5-access-control)
6. [Supply Chain Security](#6-supply-chain-security)
7. [Runner Security](#7-runner-security)
8. [Audit and Compliance](#8-audit-and-compliance)
9. [Security Best Practices](#9-security-best-practices)
10. [Incident Response](#10-incident-response)

---

## 1. Introduction

### 1.1. GitHub Actions Security Overview

GitHub Actions provides multiple layers of security to protect your workflows and infrastructure:
- **Repository-level security**: Branch protection, code review requirements
- **Workflow security**: Secret management, access controls
- **Supply chain security**: Verified actions, dependency scanning
- **Runner security**: Isolated environments, self-hosted options

### 1.2. Security Threats

Common security threats in CI/CD:
- Secrets leakage in logs
- Supply chain attacks via malicious actions
- Unauthorized access to runners
- Credential theft
- Dependency vulnerabilities

---

## 2. Repository Security

### 2.1. Branch Protection Rules

#### Configure Branch Protection

1. Navigate to repository **Settings** > **Branches**
2. Click "Add rule" for main branch
3. Configure settings:

```yaml
# Required settings for main branch:
- Require pull request reviews before merging
  - Required approvers: 2
  - Dismiss stale reviews when new commits are pushed
  - Require review from CODEOWNERS
- Require status checks to pass before merging
  - Require branches to be up to date before merging
- Require branches to be up to date before merging
- Require conversation resolution before merging
- Limit who can push to matching branches
- Do not allow bypassing the above settings
```

#### Protect Tags

```yaml
# Tag protection rules:
- Only allow users with admin access to push tags
- Require pattern matching for tags (e.g., v*)
```

### 2.2. CODEOWNERS Configuration

Create `.github/CODEOWNERS`:

```yaml
# Default: Everyone
* @default-team

# Documentation
/docs/ @documentation-team

# Infrastructure
/terraform/ @infrastructure-team
/kubernetes/ @infrastructure-team

# Security
/security/ @security-team

# CI/CD
.github/workflows/ @devops-team
```

### 2.3. Dependency Review

#### Enable Dependency Review

```yaml
name: Dependency Review

on:
  pull_request:

permissions:
  contents: read

jobs:
  dependency-review:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Dependency Review
        uses: actions/dependency-review-action@v4
```

---

## 3. Workflow Security

### 3.1. Workflow Permissions

#### Configure Workflow Permissions

1. Navigate to repository **Settings** > **Actions** > **General**
2. Under "Workflow permissions", select:
   - **Read and write permissions**: Required for deployments
   - **Read repository contents permission**: For read-only workflows

#### Scoped Permissions

```yaml
# Use permissions at workflow level
permissions:
  contents: read
  issues: read
  pull-requests: read
  deployments: write
```

### 3.2. Pin Action Versions

#### Use Specific Versions

```yaml
# Bad: Using @main or @master
- uses: actions/checkout@main

# Good: Using specific version
- uses: actions/checkout@v4.1.1

# Better: Using commit SHA
- uses: actions/checkout@a81bbbf8298c0fa03ea29cdc473d45769f953675
```

#### Use Dependabot for Action Updates

Create `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    labels:
      - "dependencies"
      - "github-actions"
```

### 3.3. Environment Protection Rules

#### Create Protected Environments

1. Navigate to repository **Settings** > **Environments**
2. Click "New environment"
3. Configure:
   - **Name**: production
   - **Environment secrets**: Add production-specific secrets
   - **Deployment branches**: Restrict to main branch
   - **Required reviewers**: Add required reviewers
   - **Wait timer**: Set wait time (e.g., 30 minutes)

#### Use Environments in Workflows

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - uses: actions/checkout@v4
      - run: deploy.sh
```

---

## 4. Secrets Management

### 4.1. Repository Secrets

#### Create Repository Secrets

1. Navigate to repository **Settings** > **Secrets and variables** > **Actions**
2. Click "New repository secret"
3. Add secret name and value
4. Click "Add secret"

#### Use Secrets in Workflows

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        env:
          API_KEY: ${{ secrets.API_KEY }}
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
        run: |
          # Use secrets in your deployment script
```

### 4.2. Organization Secrets

#### Create Organization Secrets

1. Navigate to organization **Settings** > **Secrets** > **Actions**
2. Click "New organization secret"
3. Add secret name and value
4. Select repositories with access
5. Click "Add secret"

#### Benefits of Organization Secrets

- Centralized management
- Consistent secrets across repositories
- Reduced duplication
- Easier rotation

### 4.3. Environment Secrets

#### Configure Environment Secrets

1. Navigate to repository **Settings** > **Environments**
2. Select environment
3. Add environment secrets

#### Use Environment Secrets

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy
        env:
          PRODUCTION_API_KEY: ${{ secrets.PRODUCTION_API_KEY }}
        run: |
          # Environment-specific secrets
```

### 4.4. Secrets Rotation

#### Automate Secret Rotation

```yaml
name: Rotate Secrets

on:
  schedule:
    - cron: '0 0 1 * *'  # Monthly

jobs:
  rotate:
    runs-on: ubuntu-latest
    steps:
      - name: Generate New Secret
        id: generate
        run: |
          NEW_SECRET=$(openssl rand -base64 32)
          echo "secret=$NEW_SECRET" >> $GITHUB_OUTPUT

      - name: Update Secret
        uses: hmanzur/action-set-secret@v2
        with:
          name: API_KEY
          value: ${{ steps.generate.outputs.secret }}
          repository: ${{ github.repository }}
          token: ${{ secrets.GITHUB_TOKEN }}
```

---

## 5. Access Control

### 5.1. User Permissions

#### Configure User Access Levels

| Permission Level | Capabilities |
|-----------------|--------------|
| Admin | Full control, settings, billing |
| Maintain | Issues, pull requests, wikis |
| Write | Push to all branches |
| Triage | Manage issues and pull requests |
| Read | Read-only access |

#### Best Practices

- Grant least privilege required
- Regularly review and revoke access
- Use teams for group permissions
- Enable two-factor authentication

### 5.2. Team Access

#### Create Teams

1. Navigate to organization **Settings** > **Teams**
2. Click "New team"
3. Configure team permissions
4. Add members to team

#### Assign Repository Access to Teams

```yaml
# Team-based access:
- @devops-team: Write access to all repositories
- @security-team: Read access to all repositories, Write to security repos
- @developers: Write access to application repositories
```

### 5.3. SSH Key Management

#### Deploy Keys for Deployment

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Setup SSH Key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -H github.com >> ~/.ssh/known_hosts

      - name: Deploy via SSH
        run: |
          ssh user@server.com 'deploy.sh'
```

---

## 6. Supply Chain Security

### 6.1. Verified Actions

#### Use Verified Actions

```yaml
# Check if action is verified
# Look for the "Verified creator" badge in the marketplace

# Example of verified actions:
- uses: actions/checkout@v4              # Verified
- uses: docker/setup-buildx-action@v3   # Verified
- uses: aws-actions/configure-aws-credentials@v4  # Verified
```

#### Use Action References

```yaml
# Use full reference including owner
- uses: actions/checkout@v4

# Avoid using unverified actions
# - uses: random-user/random-action@v1
```

### 6.2. Action Audit

#### Audit Workflow Actions

```yaml
name: Audit Actions

on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Audit Workflow Actions
        uses: trstringer/manual-approval@v1
        with:
          secret: ${{ secrets.GITHUB_TOKEN }}
          approvers: security-team
          minimum-approvals: 1
```

### 6.3. Dependency Scanning

#### GitHub Dependency Graph

Enable dependency graph:
1. Navigate to repository **Settings** > **Security & analysis**
2. Enable "Dependency graph"

#### Dependabot Alerts

Enable Dependabot alerts:
1. Navigate to repository **Settings** > **Security & analysis**
2. Enable "Dependabot alerts"

#### Dependabot Security Updates

Create `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "daily"
    versioning-strategy: increase
    open-pull-requests-limit: 10
```

---

## 7. Runner Security

### 7.1. GitHub-Hosted Runners

#### Security Features

- Ephemeral: Fresh runner for each job
- Isolated: Network and resource isolation
- Managed: Security updates by GitHub
- Audited: Regular security audits

#### Best Practices

- Use GitHub-hosted runners for public repositories
- Avoid storing sensitive data on runners
- Clean up temporary files after jobs
- Don't cache sensitive data

### 7.2. Self-Hosted Runners

#### Secure Self-Hosted Runner Configuration

```bash
# Run as non-root user
sudo useradd -m -s /bin/bash runner
sudo usermod -aG docker runner

# Configure runner as service
sudo ./svc.sh install
sudo ./svc.sh start
```

#### Network Isolation

```yaml
# Configure firewall rules
sudo ufw allow from 192.168.1.0/24 to any port 22
sudo ufw allow from 192.168.1.0/24 to any port 443
sudo ufw deny incoming
sudo ufw enable
```

#### Runner Hardening

```bash
# Update system regularly
sudo apt update && sudo apt upgrade -y

# Install security tools
sudo apt install -y fail2ban ufw

# Configure automatic security updates
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

### 7.3. Runner Groups

#### Create Runner Groups

1. Navigate to organization **Settings** > **Actions** > **Runner groups**
2. Click "New runner group"
3. Configure:
   - **Name**: production-runners
   - **Visibility**: Private
   - **Repositories**: Select repositories with access

#### Use Runner Groups

```yaml
jobs:
  deploy:
    runs-on:
      group: production-runners
      labels: [self-hosted, linux, x64]
    steps:
      - uses: actions/checkout@v4
```

---

## 8. Audit and Compliance

### 8.1. Audit Logs

#### Access Audit Logs

1. Navigate to organization **Settings** > **Audit log**
2. Filter by:
   - User
   - Repository
   - Action
   - Date range

#### Export Audit Logs

```bash
# Use GitHub API to export logs
curl -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/orgs/YOUR_ORG/audit-log \
  > audit-log.json
```

### 8.2. Security Advisories

#### Create Security Advisory

1. Navigate to repository **Security** > **Advisories**
2. Click "New draft security advisory"
3. Fill in:
   - **Summary**: Brief description
   - **Severity**: Critical, High, Medium, Low
   - **Affected versions**: Version range
   - **CVE ID**: If applicable

### 8.3. Compliance Reporting

#### Generate Compliance Report

```yaml
name: Compliance Report

on:
  schedule:
    - cron: '0 0 1 * *'  # Monthly

jobs:
  compliance:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate Report
        run: |
          echo "# Compliance Report" > report.md
          echo "## Repository: ${{ github.repository }}" >> report.md
          echo "## Date: $(date)" >> report.md
          echo "## Branch Protection: Enabled" >> report.md

      - name: Upload Report
        uses: actions/upload-artifact@v4
        with:
          name: compliance-report
          path: report.md
```

---

## 9. Security Best Practices

### 9.1. Workflow Security

- Use least privilege permissions
- Pin action versions
- Use environment protection rules
- Implement approval workflows
- Regularly audit workflows

### 9.2. Secrets Management

- Use environment-specific secrets
- Rotate secrets regularly
- Never log secrets
- Use secret scanning
- Implement secret rotation automation

### 9.3. Supply Chain Security

- Use verified actions only
- Enable Dependabot alerts
- Implement dependency review
- Use signed commits
- Enable branch protection

### 9.4. Runner Security

- Prefer GitHub-hosted runners
- Isolate self-hosted runners
- Regularly update runners
- Monitor runner logs
- Implement network segmentation

---

## 10. Incident Response

### 10.1. Security Incident Response Plan

#### Detection

1. Monitor audit logs
2. Enable security alerts
3. Configure notifications
4. Set up anomaly detection

#### Containment

1. Revoke compromised credentials
2. Disable affected workflows
3. Isolate affected runners
4. Suspend integrations

#### Investigation

1. Analyze audit logs
2. Review workflow runs
3. Check for unauthorized access
4. Identify affected resources

#### Recovery

1. Rotate all secrets
2. Update workflows
3. Patch vulnerabilities
4. Implement additional controls

### 10.2. Emergency Procedures

#### Revoke GitHub Token

```bash
# Revoke all tokens
# Navigate to: Developer settings > Personal access tokens
# Delete all tokens

# Revoke OAuth apps
# Navigate to: Settings > Applications > Authorized OAuth Apps
# Revoke access
```

#### Disable Workflows

```yaml
# Add temporary disable
name: Disabled Workflow

on:
  push:
    branches: [ main ]

jobs:
  disabled:
    runs-on: ubuntu-latest
    steps:
      - name: Workflow Disabled
        run: echo "Workflow is disabled for security reasons"
```

---

## References

- [GitHub Actions Security Documentation](https://docs.github.com/en/actions/security-guides)
- [Securing Your Workflow](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [Encrypted Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [Dependabot Documentation](https://docs.github.com/en/code-security/dependabot)

---

## By [Harshhaa Reddy](https://www.github.com/NotHarshhaa)
