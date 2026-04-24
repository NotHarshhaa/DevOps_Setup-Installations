# GitHub Actions Installation and Configuration Guide

![GitHub Actions](https://github.blog/wp-content/uploads/2019/08/DL-V2-LinkedIn_FB.png)

Comprehensive guide for setting up GitHub Actions, configuring workflows, self-hosted runners, and implementing CI/CD pipelines.

## Table of Contents
1. [Introduction](#1-introduction)
2. [Getting Started with GitHub Actions](#2-getting-started-with-github-actions)
3. [Workflow Syntax](#3-workflow-syntax)
4. [Common Actions](#4-common-actions)
5. [CI/CD Pipelines](#5-cicd-pipelines)
6. [Self-Hosted Runners](#6-self-hosted-runners)
7. [Secrets and Security](#7-secrets-and-security)
8. [Advanced Features](#8-advanced-features)
9. [Best Practices](#9-best-practices)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Introduction

### 1.1. What is GitHub Actions?

GitHub Actions is a CI/CD platform that allows you to automate your build, test, and deployment pipelines. You can create workflows that build and test every pull request or deploy merged code to production.

### 1.2. Key Features

- **Free for public repositories**: Unlimited minutes and storage
- **Customizable workflows**: YAML-based configuration
- **Marketplace**: Thousands of pre-built actions
- **Matrix builds**: Test across multiple OS and versions
- **Self-hosted runners**: Run on your own infrastructure
- **Secrets management**: Secure storage for sensitive data
- **Artifact caching**: Speed up builds with dependency caching

### 1.3. Pricing (Private Repositories)

| Plan | Free | Pro | Team | Enterprise |
|------|------|-----|------|------------|
| Minutes/month | 2,000 | 3,000 | 10,000 | 50,000 |
| Storage | 500 MB | 1 GB | 2 GB | 10 GB |

---

## 2. Getting Started with GitHub Actions

### 2.1. Enable GitHub Actions

#### For New Repositories

1. Create a new repository on GitHub
2. GitHub Actions is automatically enabled

#### For Existing Repositories

1. Navigate to repository **Settings** > **Actions** > **General**
2. Under "Actions permissions", select:
   - **Allow all actions and reusable workflows**
   - Or restrict to specific actions

### 2.2. Create Your First Workflow

```yaml
# .github/workflows/hello-world.yml
name: Hello World

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run a one-line script
        run: echo Hello, world!

      - name: Run a multi-line script
        run: |
          echo Add other actions to build,
          echo test, and deploy your project.
```

### 2.3. Workflow File Structure

```
your-repo/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── cd.yml
│       └── security.yml
├── src/
└── README.md
```

---

## 3. Workflow Syntax

### 3.1. Basic Workflow Structure

```yaml
name: Workflow Name                    # Workflow name
on:                                    # Triggers
  push:                                # Push events
    branches: [main, develop]          # Branches to trigger on
  pull_request:                        # Pull request events
    branches: [main]
  schedule:                            # Scheduled runs
    - cron: '0 0 * * *'                # Daily at midnight
  workflow_dispatch:                   # Manual trigger

jobs:                                  # Jobs
  job-name:                            # Job identifier
    runs-on: ubuntu-latest             # Runner type
    steps:                             # Steps
      - name: Step name                # Step name
        uses: action@v1                # Use an action
      - name: Run command              # Run shell command
        run: echo "Hello"
```

### 3.2. Event Triggers

#### Push Events

```yaml
on:
  push:
    branches: [main, develop]
    tags:
      - 'v*'
    paths:
      - 'src/**'
      - '.github/workflows/**'
```

#### Pull Request Events

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [main]
```

#### Scheduled Events

```yaml
on:
  schedule:
    - cron: '0 2 * * *'    # Daily at 2 AM
    - cron: '0 0 * * 0'    # Weekly on Sunday
```

#### Manual Trigger

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
```

### 3.3. Job Configuration

#### Sequential Jobs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
```

#### Parallel Jobs

```yaml
jobs:
  test-linux:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

  test-windows:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4

  test-macos:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
```

#### Matrix Strategy

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [14.x, 16.x, 18.x]
        exclude:
          - os: macos-latest
            node-version: 14.x
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
```

---

## 4. Common Actions

### 4.1. Checkout Code

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0              # Full history
    lfs: true                   # Include LFS files
    submodules: recursive       # Include submodules
    token: ${{ secrets.GITHUB_TOKEN }}
```

### 4.2. Setup Languages

#### Node.js

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '18'
    cache: 'npm'                # Cache dependencies
```

#### Python

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: '3.11'
    cache: 'pip'
```

#### Java

```yaml
- uses: actions/setup-java@v4
  with:
    distribution: 'temurin'
    java-version: '17'
    cache: 'maven'
```

#### Go

```yaml
- uses: actions/setup-go@v5
  with:
    go-version: '1.21'
    cache: true
```

#### .NET

```yaml
- uses: actions/setup-dotnet@v4
  with:
    dotnet-version: '7.x'
```

### 4.3. Cache Dependencies

```yaml
- name: Cache Node modules
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

### 4.4. Upload/Download Artifacts

```yaml
- name: Upload artifact
  uses: actions/upload-artifact@v4
  with:
    name: my-artifact
    path: dist/

- name: Download artifact
  uses: actions/download-artifact@v4
  with:
    name: my-artifact
    path: ./dist
```

### 4.5. Docker Actions

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Login to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: myuser/myapp:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

---

## 5. CI/CD Pipelines

### 5.1. Java Application CI/CD

```yaml
name: Java CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
          cache: 'maven'

      - name: Build with Maven
        run: mvn -B clean package

      - name: Run tests
        run: mvn test

      - name: Generate coverage report
        run: mvn jacoco:report

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: ./target/site/jacoco/jacoco.xml

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: package
          path: target/*.jar

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          name: package

      - name: Set up SSH
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Deploy to server
        run: |
          ssh -o StrictHostKeyChecking=no user@server.com 'cd /app && systemctl stop myapp'
          scp target/*.jar user@server.com:/app/
          ssh -o StrictHostKeyChecking=no user@server.com 'cd /app && systemctl start myapp'
```

### 5.2. Node.js Application CI/CD

```yaml
name: Node.js CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests
        run: npm test

      - name: Build application
        run: npm run build

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ matrix.node-version }}
          path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build application
        run: npm run build

      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
```

### 5.3. Python Application CI/CD

```yaml
name: Python CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11']

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest pytest-cov flake8

      - name: Lint with flake8
        run: |
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
          flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

      - name: Test with pytest
        run: |
          pytest --cov=. --cov-report=xml --cov-report=html

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.xml

      - name: Build package
        run: python -m build

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist-${{ matrix.python-version }}
          path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install build twine

      - name: Build package
        run: python -m build

      - name: Publish to PyPI
        env:
          TWINE_USERNAME: __token__
          TWINE_PASSWORD: ${{ secrets.PYPI_API_TOKEN }}
        run: twine upload dist/*
```

### 5.4. Go Application CI/CD

```yaml
name: Go CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        go-version: ['1.20', '1.21', '1.22']

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go ${{ matrix.go-version }}
        uses: actions/setup-go@v5
        with:
          go-version: ${{ matrix.go-version }}
          cache: true

      - name: Download dependencies
        run: go mod download

      - name: Verify dependencies
        run: go mod verify

      - name: Run go vet
        run: go vet ./...

      - name: Run tests
        run: go test -v -race -coverprofile=coverage.txt -covermode=atomic ./...

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.txt

      - name: Build application
        run: go build -v -o app ./...

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: app-${{ matrix.go-version }}
          path: app

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.21'

      - name: Build application
        run: go build -v -o app ./...

      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: myuser/myapp:latest
          registry: docker.io
```

---

## 6. Self-Hosted Runners

### 6.1. Add Self-Hosted Runner

#### Step 1: Navigate to Runner Settings

1. Go to repository **Settings** > **Actions** > **Runners**
2. Click "New self-hosted runner"
3. Select OS (Linux, macOS, Windows)

#### Step 2: Install Runner on Linux

```bash
# Create a directory
mkdir actions-runner && cd actions-runner

# Download the latest runner package
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz

# Extract the installer
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz

# Configure the runner
./config.sh --url https://github.com/YOUR_ORG/YOUR_REPO --token YOUR_TOKEN

# Run the runner
./run.sh
```

#### Step 3: Install Runner as Service

```bash
# Install as service
sudo ./svc.sh install

# Start the service
sudo ./svc.sh start

# Check status
sudo ./svc.sh status
```

#### Step 4: Install Runner on Windows

```powershell
# Create a directory
mkdir actions-runner
cd actions-runner

# Download the latest runner package
Invoke-WebRequest -Uri https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-win-x64-2.311.0.zip -OutFile actions-runner-win-x64-2.311.0.zip

# Extract the installer
Add-Type -AssemblyName System.IO.Compression.FileSystem
[System.IO.Compression.ZipFile]::ExtractToDirectory("$PWD\actions-runner-win-x64-2.311.0.zip", "$PWD")

# Configure the runner
.\config.cmd --url https://github.com/YOUR_ORG/YOUR_REPO --token YOUR_TOKEN

# Run the runner
.\run.cmd
```

### 6.2. Configure Runner Labels

```bash
# Add labels during configuration
./config.sh --url https://github.com/YOUR_ORG/YOUR_REPO \
  --token YOUR_TOKEN \
  --labels self-hosted,linux,x64,docker

# Or add labels after configuration
# Navigate to: Settings > Actions > Runners > Select runner > Settings
```

### 6.3. Use Self-Hosted Runner in Workflow

```yaml
jobs:
  build:
    runs-on: [self-hosted, linux, x64]

    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: make build
```

---

## 7. Secrets and Security

### 7.1. Create Secrets

#### Repository Secrets

1. Navigate to repository **Settings** > **Secrets and variables** > **Actions**
2. Click "New repository secret"
3. Add secret name and value
4. Click "Add secret"

#### Organization Secrets

1. Navigate to organization **Settings** > **Secrets** > **Actions**
2. Click "New organization secret"
3. Add secret name and value
4. Select repositories with access
5. Click "Add secret"

### 7.2. Use Secrets in Workflows

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
          echo "Deploying with API key"
          # Use secrets in your deployment script
```

### 7.3. Encrypted Secrets

```yaml
# Use encrypted secrets for sensitive data
- name: Decrypt secrets
  run: |
    echo "${{ secrets.ENCRYPTED_SECRET }}" | base64 -d > secret.txt
```

---

## 8. Advanced Features

### 8.1. Reusable Workflows

```yaml
# .github/workflows/reusable-build.yml
name: Reusable Build

on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string
    secrets:
      npm-token:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
      - run: npm ci
        env:
          NODE_AUTH_TOKEN: ${{ secrets.npm-token }}
```

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]

jobs:
  call-build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      node-version: '18'
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
```

### 8.2. Composite Actions

```yaml
# .github/actions/setup-build/action.yml
name: 'Setup Build Environment'
description: 'Sets up the build environment with common tools'

inputs:
  node-version:
    description: 'Node.js version'
    required: true
    default: '18'

runs:
  using: "composite"
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - name: Install dependencies
      run: npm ci
      shell: bash
```

```yaml
# Use composite action
steps:
  - uses: ./.github/actions/setup-build
    with:
      node-version: '20'
```

### 8.3. Environments

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - uses: actions/checkout@v4
      - run: deploy-to-staging.sh

  deploy-production:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - uses: actions/checkout@v4
      - run: deploy-to-production.sh
```

---

## 9. Best Practices

### 9.1. Workflow Organization

- Use descriptive names for workflows and jobs
- Group related steps together
- Use conditions for environment-specific steps
- Implement proper error handling
- Use environment variables for configuration

### 9.2. Security Best Practices

- Use secrets for sensitive data
- Implement proper access controls
- Regular security scanning
- Use minimum required permissions
- Implement proper secrets rotation

### 9.3. Performance Optimization

- Use caching for dependencies
- Implement proper job dependencies
- Use matrix builds efficiently
- Clean up workspace regularly
- Use self-hosted runners when appropriate

---

## 10. Troubleshooting

### 10.1. Common Issues

1. Workflow syntax errors
2. Secret configuration issues
3. Permission problems
4. Dependency conflicts
5. Timeout issues

### 10.2. Debugging Tips

1. Use debug logging
2. Check workflow run logs
3. Validate workflow syntax
4. Test locally with act
5. Use workflow visualization tools

---

## References

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Workflow Syntax](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)

---

## By [Harshhaa Reddy](https://www.github.com/NotHarshhaa)
