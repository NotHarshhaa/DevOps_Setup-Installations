# GitHub Actions Workflow Templates and Best Practices

![banner](https://github.blog/wp-content/uploads/2019/08/DL-V2-LinkedIn_FB.png)

Comprehensive collection of reusable GitHub Actions workflow templates for various programming languages, frameworks, and deployment scenarios.

## Table of Contents
1. [Introduction](#1-introduction)
2. [CI/CD Pipeline Templates](#2-cicd-pipeline-templates)
3. [Security Scanning Templates](#3-security-scanning-templates)
4. [Infrastructure as Code Templates](#4-infrastructure-as-code-templates)
5. [Testing Templates](#5-testing-templates)
6. [Release Management Templates](#6-release-management-templates)
7. [Code Quality Templates](#7-code-quality-templates)
8. [Deployment Templates](#8-deployment-templates)
9. [Notification Templates](#9-notification-templates)
10. [Best Practices](#10-best-practices)

---

## 1. Introduction

### 1.1. About This Collection

This collection provides production-ready workflow templates that you can copy and adapt for your projects. Each template includes:
- Complete YAML configuration
- Best practices implementation
- Security considerations
- Optimization techniques

### 1.2. How to Use These Templates

1. Copy the template to `.github/workflows/`
2. Customize variables and secrets
3. Adjust for your specific requirements
4. Test in a development environment first

---

## 2. CI/CD Pipeline Templates

### 2.1. Java Spring Boot Application

```yaml
name: Java Spring Boot CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4

    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
        cache: maven

    - name: Build with Maven
      run: mvn -B clean package --file pom.xml

    - name: Run Tests
      run: mvn test

    - name: Generate Coverage Report
      run: mvn jacoco:report

    - name: Upload Coverage to Codecov
      uses: codecov/codecov-action@v4
      with:
        files: ./target/site/jacoco/jacoco.xml

    - name: Build Docker Image
      run: docker build -t myapp:${{ github.sha }} .

    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}

    - name: Push to Docker Hub
      run: |
        docker tag myapp:${{ github.sha }} ${{ secrets.DOCKERHUB_USERNAME }}/myapp:latest
        docker push ${{ secrets.DOCKERHUB_USERNAME }}/myapp:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://myapp.example.com

    steps:
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-west-2

    - name: Update ECS Service
      run: |
        aws ecs update-service --cluster production --service myapp --force-new-deployment

    - name: Wait for Deployment
      run: |
        aws ecs wait services-stable --cluster production --services myapp
```

### 2.2. Node.js Application

```yaml
name: Node.js CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [18.x, 20.x]
        os: [ubuntu-latest]

    steps:
    - uses: actions/checkout@v4

    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'

    - name: Install Dependencies
      run: npm ci

    - name: Run Linter
      run: npm run lint

    - name: Run Tests
      run: npm test -- --coverage

    - name: Upload Coverage to Codecov
      uses: codecov/codecov-action@v4
      with:
        files: ./coverage/coverage-final.json

    - name: Build Application
      run: npm run build

    - name: Upload Build Artifacts
      uses: actions/upload-artifact@v4
      with:
        name: build-${{ matrix.node-version }}
        path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://myapp.vercel.app

    steps:
    - uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'

    - name: Install Dependencies
      run: npm ci

    - name: Build Application
      run: npm run build

    - name: Deploy to Vercel
      uses: amondnet/vercel-action@v25
      with:
        vercel-token: ${{ secrets.VERCEL_TOKEN }}
        vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
        vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
        vercel-args: '--prod'
```

### 2.3. Python Application

```yaml
name: Python CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        python-version: ['3.10', '3.11', '3.12']

    steps:
    - uses: actions/checkout@v4

    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v5
      with:
        python-version: ${{ matrix.python-version }}
        cache: 'pip'

    - name: Install Dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install pytest pytest-cov flake8 mypy

    - name: Lint with Flake8
      run: |
        flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
        flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics

    - name: Type Check with MyPy
      run: mypy src/

    - name: Run Tests
      run: pytest --cov=src --cov-report=xml --cov-report=html

    - name: Upload Coverage to Codecov
      uses: codecov/codecov-action@v4
      with:
        files: ./coverage.xml

    - name: Build Package
      run: python -m build

    - name: Upload Artifacts
      uses: actions/upload-artifact@v4
      with:
        name: dist-${{ matrix.python-version }}
        path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment:
      name: production

    steps:
    - uses: actions/checkout@v4

    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'

    - name: Install Dependencies
      run: |
        python -m pip install --upgrade pip
        pip install build twine

    - name: Build Package
      run: python -m build

    - name: Publish to PyPI
      env:
        TWINE_USERNAME: __token__
        TWINE_PASSWORD: ${{ secrets.PYPI_API_TOKEN }}
      run: twine upload dist/*
```

### 2.4. Go Application

```yaml
name: Go CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        go-version: ['1.21', '1.22']
        os: [ubuntu-latest, windows-latest, macos-latest]

    steps:
    - uses: actions/checkout@v4

    - name: Set up Go ${{ matrix.go-version }}
      uses: actions/setup-go@v5
      with:
        go-version: ${{ matrix.go-version }}
        cache: true

    - name: Download Dependencies
      run: go mod download

    - name: Verify Dependencies
      run: go mod verify

    - name: Run Go Vet
      run: go vet ./...

    - name: Run Tests
      run: go test -v -race -coverprofile=coverage.txt -covermode=atomic ./...

    - name: Upload Coverage to Codecov
      uses: codecov/codecov-action@v4
      with:
        files: ./coverage.txt

    - name: Build Application
      run: go build -v -o app ./...

    - name: Upload Artifacts
      uses: actions/upload-artifact@v4
      with:
        name: app-${{ matrix.os }}-${{ matrix.go-version }}
        path: app

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
    - uses: actions/checkout@v4

    - name: Set up Go
      uses: actions/setup-go@v5
      with:
        go-version: '1.22'

    - name: Build Application
      run: go build -v -o app ./...

    - name: Build and Push Docker Image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: myuser/myapp:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

---

## 3. Security Scanning Templates

### 3.1. Comprehensive Security Scan

```yaml
name: Security Scan

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 0 * * 0'  # Weekly on Sunday

jobs:
  security:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4

    - name: Run Snyk Security Scan
      uses: snyk/actions/node@master
      continue-on-error: true
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

    - name: Run Trivy Vulnerability Scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: Upload Trivy Results to GitHub Security
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: 'trivy-results.sarif'

    - name: Run CodeQL Analysis
      uses: github/codeql-action/analyze@v3
      with:
        languages: javascript, python

    - name: Run OWASP Dependency Check
      uses: dependency-check/Dependency-Check_Action@main
      with:
        project: 'my-project'
        path: '.'
        format: 'HTML'
        out: 'dependency-check-report'

    - name: Upload Dependency Check Report
      uses: actions/upload-artifact@v4
      with:
        name: dependency-check-report
        path: dependency-check-report
```

### 3.2. Container Security Scan

```yaml
name: Container Security Scan

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  scan:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4

    - name: Build Docker Image
      run: docker build -t myapp:${{ github.sha }} .

    - name: Run Trivy Image Scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'myapp:${{ github.sha }}'
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'

    - name: Upload Trivy Results to GitHub Security
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: 'trivy-results.sarif'

    - name: Run Grype Scanner
      uses: anchore/scan-action@v3
      with:
        image: 'myapp:${{ github.sha }}'
        format: 'sarif'
        output: 'grype-results.sarif'

    - name: Upload Grype Results to GitHub Security
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: 'grype-results.sarif'
```

---

## 4. Infrastructure as Code Templates

### 4.1. Terraform Workflow

```yaml
name: Terraform

on:
  push:
    branches: [ main ]
    paths:
    - 'terraform/**'
  pull_request:
    branches: [ main ]
    paths:
    - 'terraform/**'
  workflow_dispatch:

jobs:
  terraform:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ./terraform

    steps:
    - uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: '1.6.0'
        terraform_wrapper: false

    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-west-2

    - name: Terraform Format
      run: terraform fmt -check

    - name: Terraform Init
      run: terraform init

    - name: Terraform Validate
      run: terraform validate

    - name: Terraform Plan
      id: plan
      run: terraform plan -no-color -out=tfplan

    - name: Terraform Plan Summary
      if: always()
      run: |
        echo "## Terraform Plan Summary" >> $GITHUB_STEP_SUMMARY
        echo '```' >> $GITHUB_STEP_SUMMARY
        terraform show -no-color tfplan >> $GITHUB_STEP_SUMMARY
        echo '```' >> $GITHUB_STEP_SUMMARY

    - name: Terraform Apply
      if: github.ref == 'refs/heads/main' && github.event_name == 'push'
      run: terraform apply -auto-approve tfplan

    - name: Terraform Output
      if: always()
      run: terraform output -json
```

### 4.2. Ansible Workflow

```yaml
name: Ansible

on:
  push:
    branches: [ main ]
    paths:
    - 'ansible/**'
  workflow_dispatch:

jobs:
  ansible:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4

    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'

    - name: Install Ansible
      run: |
        python -m pip install --upgrade pip
        pip install ansible ansible-lint

    - name: Run Ansible Lint
      run: ansible-lint ansible/

    - name: Set up SSH Key
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
        chmod 600 ~/.ssh/id_rsa
        ssh-keyscan -H ${{ secrets.ANSIBLE_HOST }} >> ~/.ssh/known_hosts

    - name: Run Ansible Playbook
      run: |
        ansible-playbook -i ansible/inventory ansible/playbook.yml \
          --extra-vars "ansible_user=${{ secrets.ANSIBLE_USER }}"
```

---

## 5. Testing Templates

### 5.1. End-to-End Testing with Cypress

```yaml
name: E2E Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  cypress:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
    - uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'

    - name: Install Dependencies
      run: npm ci

    - name: Start Application
      run: npm start &
      env:
        DATABASE_URL: postgresql://test:test@localhost:5432/testdb

    - name: Wait for Application
      run: npx wait-on http://localhost:3000

    - name: Run Cypress Tests
      uses: cypress-io/github-action@v6
      with:
        browser: chrome
        headless: true
        record: true
      env:
        CYPRESS_RECORD_KEY: ${{ secrets.CYPRESS_RECORD_KEY }}

    - name: Upload Screenshots
      uses: actions/upload-artifact@v4
      if: failure()
      with:
        name: cypress-screenshots
        path: cypress/screenshots

    - name: Upload Videos
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: cypress-videos
        path: cypress/videos
```

### 5.2. Playwright Testing

```yaml
name: Playwright Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'

    - name: Install Dependencies
      run: npm ci

    - name: Install Playwright Browsers
      run: npx playwright install --with-deps

    - name: Run Playwright Tests
      run: npx playwright test

    - name: Upload Test Results
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: playwright-report
        path: playwright-report/
```

---

## 6. Release Management Templates

### 6.1. Automated Release

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'

    - name: Install Dependencies
      run: npm ci

    - name: Build Application
      run: npm run build

    - name: Create Release
      uses: softprops/action-gh-release@v1
      with:
        draft: false
        prerelease: false
        generate_release_notes: true
        files: |
          dist/*
          CHANGELOG.md
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

    - name: Publish to NPM
      run: npm publish
      env:
        NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### 6.2. Semantic Release

```yaml
name: Semantic Release

on:
  push:
    branches: [ main ]

jobs:
  release:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
        persist-credentials: false

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'

    - name: Install Dependencies
      run: npm ci

    - name: Run Semantic Release
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
      run: npx semantic-release
```

---

## 7. Code Quality Templates

### 7.1. Code Quality Checks

```yaml
name: Code Quality

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  quality:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'

    - name: Install Dependencies
      run: npm ci

    - name: SonarCloud Scan
      uses: SonarSource/sonarcloud-github-action@master
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

    - name: Run ESLint
      run: npm run lint

    - name: Check Code Formatting
      run: npm run format:check

    - name: Run Type Check
      run: npm run type-check

    - name: Run Prettier Check
      run: npx prettier --check "**/*.{js,jsx,ts,tsx,json,md}"
```

---

## 8. Deployment Templates

### 8.1. Deploy to AWS ECS

```yaml
name: Deploy to AWS ECS

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.example.com

    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-west-2

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build and Push Docker Image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: |
          ${{ steps.login-ecr.outputs.registry }}/myapp:${{ github.sha }}
          ${{ steps.login-ecr.outputs.registry }}/myapp:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Update ECS Service
      run: |
        aws ecs update-service \
          --cluster production \
          --service myapp \
          --force-new-deployment \
          --task-definition myapp:${{ github.sha }}

    - name: Wait for Deployment
      run: |
        aws ecs wait services-stable \
          --cluster production \
          --services myapp
```

### 8.2. Deploy to Kubernetes

```yaml
name: Deploy to Kubernetes

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production

    steps:
    - uses: actions/checkout@v4

    - name: Configure kubectl
      uses: azure/k8s-set-context@v4
      with:
        method: kubeconfig
        kubeconfig: ${{ secrets.KUBE_CONFIG }}

    - name: Build and Push Docker Image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: myregistry/myapp:${{ github.sha }}

    - name: Update Kubernetes Deployment
      run: |
        kubectl set image deployment/myapp \
          myapp=myregistry/myapp:${{ github.sha }} \
          --namespace=production

    - name: Verify Deployment
      run: |
        kubectl rollout status deployment/myapp --namespace=production
```

---

## 9. Notification Templates

### 9.1. Slack Notifications

```yaml
name: Slack Notification

on:
  workflow_run:
    workflows: ["CI/CD"]
    types:
      - completed

jobs:
  notify:
    runs-on: ubuntu-latest
    if: github.event.workflow_run.conclusion == 'failure'

    steps:
    - name: Send Slack Notification
      uses: slackapi/slack-github-action@v1.25.0
      with:
        payload: |
          {
            "text": "Workflow failed",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Workflow ${{ github.event.workflow_run.name }}* failed\n\nRepository: ${{ github.repository }}\nRun: ${{ github.event.workflow_run.html_url }}"
                }
              }
            ]
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### 9.2. Email Notifications

```yaml
name: Email Notification

on:
  workflow_run:
    workflows: ["CI/CD"]
    types:
      - completed

jobs:
  notify:
    runs-on: ubuntu-latest

    steps:
    - name: Send Email
      uses: dawidd6/action-send-mail@v3
      with:
        server_address: smtp.gmail.com
        server_port: 465
        username: ${{ secrets.EMAIL_USERNAME }}
        password: ${{ secrets.EMAIL_PASSWORD }}
        subject: Workflow ${{ github.event.workflow_run.conclusion }} - ${{ github.repository }}
        to: team@example.com
        from: github-actions@example.com
        body: |
          Workflow ${{ github.event.workflow_run.name }} has ${{ github.event.workflow_run.conclusion }}.

          Repository: ${{ github.repository }}
          Run: ${{ github.event.workflow_run.html_url }}
```

---

## 10. Best Practices

### 10.1. Workflow Organization

- Use descriptive names for workflows and jobs
- Group related steps together
- Use conditions for environment-specific steps
- Implement proper error handling
- Use environment variables for configuration

### 10.2. Security Best Practices

- Use secrets for sensitive data
- Implement proper access controls
- Regular security scanning
- Use minimum required permissions
- Implement proper secrets rotation
- Review and update actions regularly

### 10.3. Performance Optimization

- Use caching for dependencies
- Implement proper job dependencies
- Use matrix builds efficiently
- Clean up workspace regularly
- Use self-hosted runners when appropriate
- Parallelize independent jobs

### 10.4. Maintainability

- Use reusable workflows
- Create composite actions for common tasks
- Document complex workflows
- Use environment variables for configuration
- Implement proper error handling
- Add comments for complex logic

---

## References

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Workflow Syntax Reference](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
- [Security Hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)

---

## By [Harshhaa Reddy](https://www.github.com/NotHarshhaa)
