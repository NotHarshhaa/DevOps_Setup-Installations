# CI/CD Integration with Docker

![docker](https://spaceliftio.wpcomstaging.com/wp-content/uploads/2024/04/367.docker-ci-cd.png)

This guide covers integrating Docker with various CI/CD platforms including Jenkins, GitHub Actions, and GitLab CI/CD for automated building, testing, and deployment of containerized applications.

## Prerequisites

- **Docker** installed on the CI/CD server or runner
- A **Git repository** with your application's code and a **Dockerfile**
- **Docker Hub** or container registry account
- Appropriate permissions to push/pull images

---

## 1. Jenkins with Docker

### 1.1. Install Jenkins on Docker

```bash
# Pull Jenkins LTS image
docker pull jenkins/jenkins:lts

# Run Jenkins container with Docker socket mounted
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

Access Jenkins at `http://localhost:8080` and follow the setup wizard.

### 1.2. Install Required Plugins

1. Go to **Manage Jenkins > Manage Plugins**
2. Install:
   - **Pipeline**
   - **Git**
   - **Docker Pipeline**
   - **Docker Build Step**

### 1.3. Modern Jenkins Pipeline (Jenkinsfile)

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "your-dockerhub-username/your-app"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        REGISTRY = 'https://index.docker.io/v1/'
        CREDENTIALS_ID = 'dockerhub-credentials'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                    docker.build("${DOCKER_IMAGE}:latest")
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").inside {
                        sh '''
                            echo "Running tests..."
                            npm test || pytest || go test
                        '''
                    }
                }
            }
        }

        stage('Security Scan') {
            steps {
                script {
                    sh 'docker scout quickview ${DOCKER_IMAGE}:${DOCKER_TAG} || echo "Scout not available, skipping"'
                    sh 'trivy image ${DOCKER_IMAGE}:${DOCKER_TAG} || echo "Trivy not available, skipping"'
                }
            }
        }

        stage('Push to Registry') {
            steps {
                script {
                    docker.withRegistry(REGISTRY, CREDENTIALS_ID) {
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push()
                        docker.image("${DOCKER_IMAGE}:latest").push()
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh '''
                        docker stop myapp || true
                        docker rm myapp || true
                        docker run -d --name myapp -p 8080:8080 ${DOCKER_IMAGE}:latest
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up...'
            sh 'docker system prune -f'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check logs for details.'
        }
    }
}
```

### 1.4. Configure Docker Hub Credentials

1. Go to **Manage Jenkins > Manage Credentials**
2. Add credentials:
   - **Kind**: Username with password
   - **Scope**: Global
   - **ID**: `dockerhub-credentials`
   - **Username**: Your Docker Hub username
   - **Password**: Your Docker Hub password or access token

---

## 2. GitHub Actions with Docker

### 2.1. Basic GitHub Actions Workflow

Create `.github/workflows/docker.yml`:

```yaml
name: Docker Build and Push

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  DOCKER_IMAGE: your-dockerhub-username/your-app
  REGISTRY: docker.io

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Log in to Docker Hub
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.DOCKER_IMAGE }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: ${{ github.event_name != 'pull_request' }}
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=registry,ref=${{ env.DOCKER_IMAGE }}:buildcache
        cache-to: type=registry,ref=${{ env.DOCKER_IMAGE }}:buildcache,mode=max

    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.DOCKER_IMAGE }}:${{ github.sha }}
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: Upload Trivy results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v2
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'
```

### 2.2. Set Up GitHub Secrets

1. Go to your repository **Settings > Secrets and variables > Actions**
2. Add the following secrets:
   - `DOCKER_USERNAME`: Your Docker Hub username
   - `DOCKER_PASSWORD`: Your Docker Hub password or access token

### 2.3. Multi-Platform Build

```yaml
name: Multi-Platform Docker Build

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        platform: [linux/amd64, linux/arm64]

    steps:
    - uses: actions/checkout@v4

    - name: Set up QEMU
      uses: docker/setup-qemu-action@v3

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Log in to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        platforms: ${{ matrix.platform }}
        push: true
        tags: your-dockerhub-username/your-app:latest
```

---

## 3. GitLab CI/CD with Docker

### 3.1. Basic GitLab CI Pipeline

Create `.gitlab-ci.yml`:

```yaml
stages:
  - build
  - test
  - security
  - deploy

variables:
  DOCKER_IMAGE: your-dockerhub-username/your-app
  DOCKER_TAG: $CI_COMMIT_SHORT_SHA

before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t $DOCKER_IMAGE:$DOCKER_TAG .
    - docker build -t $DOCKER_IMAGE:latest .
    - docker push $DOCKER_IMAGE:$DOCKER_TAG
    - docker push $DOCKER_IMAGE:latest
  only:
    - main
    - develop

test:
  stage: test
  image: $DOCKER_IMAGE:$DOCKER_TAG
  script:
    - echo "Running tests..."
    - npm test || pytest || go test
  needs:
    - build

security:
  stage: security
  image: aquasec/trivy:latest
  script:
    - trivy image --severity HIGH,CRITICAL $DOCKER_IMAGE:$DOCKER_TAG
  needs:
    - build
  allow_failure: true

deploy:
  stage: deploy
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker pull $DOCKER_IMAGE:latest
    - docker stop myapp || true
    - docker rm myapp || true
    - docker run -d --name myapp -p 8080:8080 $DOCKER_IMAGE:latest
  only:
    - main
  when: manual
```

### 3.2. Configure GitLab CI/CD Variables

1. Go to **Settings > CI/CD > Variables**
2. Add the following variables:
   - `CI_REGISTRY_USER`: Your Docker Hub username
   - `CI_REGISTRY_PASSWORD`: Your Docker Hub password or access token
   - Mark them as **Masked** and **Protected**

### 3.3. Using GitLab Container Registry

```yaml
variables:
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$IMAGE_TAG .
    - docker push $CI_REGISTRY_IMAGE:$IMAGE_TAG
```

---

## 4. Docker Security Scanning

### 4.1. Using Trivy

```bash
# Scan local image
trivy image your-image:tag

# Scan with severity threshold
trivy image --severity HIGH,CRITICAL your-image:tag

# Generate SARIF report
trivy image --format sarif --output trivy-results.sarif your-image:tag
```

### 4.2. Using Docker Scout

```bash
# Quick view of vulnerabilities
docker scout quickview your-image:tag

# CVEs report
docker scout cves your-image:tag

# Compare two images
docker scout compare your-image:v1.0 your-image:v2.0
```

### 4.3. Using Snyk

```bash
# Install Snyk
npm install -g snyk

# Authenticate
snyk auth YOUR_TOKEN

# Scan Docker image
snyk container test your-image:tag
```

---

## 5. Best Practices

### 5.1. Image Tagging Strategy

- Use semantic versioning: `v1.0.0`, `v1.0.1`
- Use Git commit SHA for reproducibility: `abc1234`
- Use `latest` tag for the most recent stable build
- Use branch names for development builds: `develop`, `feature-xyz`

### 5.2. Multi-Stage Builds

```dockerfile
# Build stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm ci --only=production
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

### 5.3. Caching Strategy

- Use BuildKit cache mounts
- Implement layer caching in CI/CD
- Use `.dockerignore` to exclude unnecessary files

### 5.4. Security Best Practices

- Scan images for vulnerabilities before deployment
- Use non-root users in containers
- Minimize image size (use alpine variants)
- Keep base images updated
- Don't include secrets in images
- Use specific image versions, not `latest` in production

### 5.5. CI/CD Pipeline Best Practices

- Run tests before building images
- Use parallel stages for faster builds
- Implement rollback strategies
- Monitor pipeline performance
- Use infrastructure as code for deployment
- Implement proper notification systems

---

## 6. Troubleshooting

### Issue: Docker Build Fails in CI/CD

**Solution**:
- Check Docker daemon is running
- Verify build context is correct
- Check for syntax errors in Dockerfile
- Ensure all required files are included in build context

### Issue: Authentication Failures

**Solution**:
- Verify credentials are correctly configured
- Check if credentials have expired
- Use access tokens instead of passwords
- Ensure registry URL is correct

### Issue: Image Push Fails

**Solution**:
- Check registry permissions
- Verify image name format
- Ensure sufficient disk space
- Check network connectivity

---

## 7. Additional Resources

- [Docker Build Documentation](https://docs.docker.com/build/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [Jenkins Pipeline Documentation](https://www.jenkins.io/doc/book/pipeline/)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)

## Author by:

![test](https://imgur.com/2j6Aoyl.png)

> [!NOTE]
> **Join Our** [Telegram Community](https://t.me/prodevopsguy) // [Follow me](https://github.com/NotHarshhaa) **for more DevOps & Cloud content.**
