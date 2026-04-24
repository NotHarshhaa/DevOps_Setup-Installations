![jenkins-pipeline](https://www.jenkins.io/images/pipeline/realworld-pipeline-flow.png)

# Jenkins Pipeline Templates Guide

This guide provides comprehensive pipeline templates for various application types and deployment scenarios using modern Jenkins Pipeline features.

## Table of Contents

1. [Declarative Pipeline Templates](#1-declarative-pipeline-templates)
2. [Scripted Pipeline Templates](#2-scripted-pipeline-templates)
3. [Advanced Pipeline Features](#3-advanced-pipeline-features)
4. [Multi-Environment Pipelines](#4-multi-environment-pipelines)
5. [Best Practices](#5-best-practices)
6. [Troubleshooting](#6-troubleshooting)

## 1. Declarative Pipeline Templates

### 1.1. Basic Java Application Pipeline

```groovy
pipeline {
    agent any
    
    tools {
        maven 'Maven 3.9.5'
        jdk 'JDK 17'
    }
    
    environment {
        DOCKER_IMAGE = "myapp:${BUILD_NUMBER}"
        DOCKER_REGISTRY = "registry.example.com"
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/yourusername/your-repo.git'
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('Code Quality') {
            parallel {
                stage('Checkstyle') {
                    steps {
                        sh 'mvn checkstyle:check'
                    }
                }
                stage('PMD') {
                    steps {
                        sh 'mvn pmd:check'
                    }
                }
                stage('SpotBugs') {
                    steps {
                        sh 'mvn spotbugs:check'
                    }
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        
        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build(DOCKER_IMAGE)
                }
            }
        }
        
        stage('Push Docker Image') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                script {
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-registry-credentials') {
                        docker.image(DOCKER_IMAGE).push()
                        docker.image(DOCKER_IMAGE).push('latest')
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh 'kubectl apply -f k8s/staging/'
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: "Pipeline succeeded: ${env.JOB_NAME} ${env.BUILD_NUMBER} (${env.BUILD_URL})"
            )
        }
        failure {
            slackSend(
                channel: '#deployments',
                color: 'danger',
                message: "Pipeline failed: ${env.JOB_NAME} ${env.BUILD_NUMBER} (${env.BUILD_URL})"
            )
        }
    }
}
```

### 1.2. Node.js Application Pipeline

```groovy
pipeline {
    agent {
        docker {
            image 'node:20'
        }
    }
    
    environment {
        NODE_ENV = 'production'
    }
    
    stages {
        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }
        
        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }
        
        stage('Format Check') {
            steps {
                sh 'npm run format:check'
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh 'npm run test:unit'
            }
            post {
                always {
                    publishHTML(target: [
                        reportDir: 'coverage',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }
        
        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
        
        stage('E2E Tests') {
            steps {
                sh 'npm run test:e2e'
            }
        }
        
        stage('Docker Build & Push') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                        def app = docker.build("myapp:${env.BUILD_NUMBER}")
                        app.push()
                        app.push('latest')
                    }
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}
```

### 1.3. Python Application Pipeline

```groovy
pipeline {
    agent {
        docker {
            image 'python:3.11'
        }
    }
    
    stages {
        stage('Install Dependencies') {
            steps {
                sh 'pip install -r requirements.txt'
            }
        }
        
        stage('Lint') {
            steps {
                sh 'flake8 src/'
            }
        }
        
        stage('Security Scan') {
            steps {
                sh 'bandit -r src/'
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh 'pytest tests/unit/ --cov=src --cov-report=html'
            }
            post {
                always {
                    publishHTML(target: [
                        reportDir: 'htmlcov',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }
        
        stage('Package') {
            steps {
                sh 'python setup.py sdist bdist_wheel'
            }
        }
    }
}
```

## 2. Scripted Pipeline Templates

### 2.1. Multi-Branch Pipeline

```groovy
node {
    def mvnHome
    def dockerImage
    def branchName = env.BRANCH_NAME
    
    stage('Preparation') {
        mvnHome = tool 'Maven 3.9.5'
        checkout scm
    }
    
    stage('Build & Test') {
        try {
            sh "'${mvnHome}/bin/mvn' clean test"
        } catch (err) {
            currentBuild.result = 'FAILURE'
            throw err
        } finally {
            junit '**/target/surefire-reports/*.xml'
        }
    }
    
    stage('Code Quality') {
        def scannerHome = tool 'SonarQubeScanner'
        withSonarQubeEnv('SonarQube') {
            sh "${scannerHome}/bin/sonar-scanner"
        }
        
        // Wait for quality gate
        timeout(time: 5, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
    
    stage('Build Docker Image') {
        dockerImage = docker.build("myapp:${branchName}-${env.BUILD_NUMBER}")
    }
    
    if (branchName == 'main') {
        stage('Deploy to Production') {
            dockerImage.push()
            dockerImage.push('latest')
            withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                sh 'kubectl apply -f k8s/production/'
            }
        }
    } else if (branchName == 'develop') {
        stage('Deploy to Staging') {
            dockerImage.push()
            withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                sh 'kubectl apply -f k8s/staging/'
            }
        }
    } else {
        stage('Build Only') {
            echo "Building feature branch: ${branchName}"
        }
    }
}
```

### 3.1. Parallel Execution

```groovy
pipeline {
    agent any
    
    stages {
        stage('Parallel Tests') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm run test:unit'
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'npm run test:integration'
                    }
                }
                stage('E2E Tests') {
                    steps {
                        sh 'npm run test:e2e'
                    }
                }
            }
        }
    }
}
```

### 3.2. Shared Library Usage

```groovy
@Library('my-shared-library') _

pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                customBuildStep()
            }
        }
        
        stage('Deploy') {
            steps {
                customDeployStep(
                    environment: 'production',
                    region: 'us-east-1'
                )
            }
        }
    }
}
```

### 3.3. Matrix Build

```groovy
pipeline {
    agent none
    
    axes {
        axis {
            name 'PLATFORM'
            values 'linux', 'windows', 'macos'
        }
        axis {
            name 'JDK'
            values '11', '17', '21'
        }
    }
    
    stages {
        stage('Build') {
            matrix {
                stages {
                    stage('Build') {
                        agent { label "${PLATFORM}" }
                        tools {
                            jdk "JDK ${JDK}"
                        }
                        steps {
                            sh "echo Building on ${PLATFORM} with JDK ${JDK}"
                            sh './gradlew build'
                        }
                    }
                }
            }
        }
    }
}
```

### 3.4. Input and Approval

```groovy
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                sh './deploy.sh staging'
            }
        }
        
        stage('Promote to Production') {
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                timeout(time: 1, unit: 'HOURS') {
                    input message: 'Confirm production deployment', ok: 'Confirm'
                }
            }
        }
        
        stage('Deploy to Production') {
            steps {
                sh './deploy.sh production'
            }
        }
    }
}
```

## 4. Multi-Environment Pipelines

### 4.1. Blue-Green Deployment

```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = "myapp:${BUILD_NUMBER}"
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
                docker.build(DOCKER_IMAGE)
            }
        }
        
        stage('Deploy Blue') {
            steps {
                sh '''
                    helm upgrade --install myapp-blue ./helm-chart \
                        --set image.tag=${BUILD_NUMBER} \
                        --set color=blue \
                        --namespace production
                '''
            }
        }
        
        stage('Health Check Blue') {
            steps {
                sh './scripts/health-check.sh blue'
            }
        }
        
        stage('Switch Traffic') {
            steps {
                sh './scripts/switch-traffic.sh blue'
            }
        }
        
        stage('Cleanup Green') {
            steps {
                sh 'helm uninstall myapp-green -n production'
            }
        }
    }
}
```

### 4.2. Canary Deployment

```groovy
pipeline {
    agent any
    
    stages {
        stage('Deploy Canary') {
            steps {
                sh '''
                    helm upgrade --install myapp-canary ./helm-chart \
                        --set image.tag=${BUILD_NUMBER} \
                        --set replicaCount=1 \
                        --namespace production \
                        --values values-canary.yaml
                '''
            }
        }
        
        stage('Monitor Canary') {
            steps {
                script {
                    sleep(time: 10, unit: 'MINUTES')
                    def metrics = sh(script: './scripts/get-metrics.sh', returnStdout: true)
                    if (metrics.toInteger() < 95) {
                        error "Canary metrics below threshold"
                    }
                }
            }
        }
        
        stage('Deploy Full Release') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                sh '''
                    helm upgrade --install myapp ./helm-chart \
                        --set image.tag=${BUILD_NUMBER} \
                        --namespace production \
                        --values values-production.yaml
                '''
            }
        }
        
        stage('Cleanup Canary') {
            steps {
                sh 'helm uninstall myapp-canary -n production'
            }
        }
    }
}
```

## 5. Best Practices

### 5.1. Pipeline Organization
- Keep pipelines simple and focused
- Use shared libraries for common functionality
- Implement proper error handling
- Use environment variables for configuration
- Implement proper cleanup in post sections
- Use descriptive stage names
- Document complex pipeline logic

### 5.2. Security Best Practices
- Use credentials management for sensitive data
- Implement proper access controls
- Regular security scanning of dependencies
- Use approved base images for containers
- Implement proper secrets management
- Use `withCredentials` for temporary credential access
- Never hardcode passwords or tokens

### 5.3. Performance Optimization
- Use parallel execution where possible
- Implement proper caching strategies
- Clean up workspace regularly
- Use lightweight containers
- Implement proper resource constraints
- Use incremental builds when possible
- Cache dependencies between builds

### 5.4. Maintainability
- Use version control for Jenkinsfiles
- Implement proper error messages
- Use meaningful variable names
- Add comments for complex logic
- Use pipeline parameters for flexibility
- Implement proper logging

## 6. Troubleshooting

### 6.1. Common Issues
1. Pipeline syntax errors
2. Resource constraints
3. Network connectivity issues
4. Permission problems
5. Tool version mismatches
6. Credential issues
7. Workspace cleanup failures

### 6.2. Debugging Tips
1. Use `sh 'set -x'` for debug output
2. Check Jenkins logs (`/var/log/jenkins/jenkins.log`)
3. Validate Jenkinsfile syntax using "Replay" feature
4. Test scripts locally first
5. Use `echo` statements for debugging
6. Check workspace contents with `dir` or `ls`
7. Use `catchError` to prevent pipeline failure

### 6.3. Pipeline Linter

```groovy
// Use the Jenkins Pipeline Linter to validate syntax
// Navigate to: http://your-jenkins/pipeline-syntax/
// Or use the REST API:

curl -X POST -H "Content-Type: text/x-groovy" \
  --data-binary @Jenkinsfile \
  http://your-jenkins/pipeline-model-converter/validate
```

### 6.4. Common Debugging Pipeline

```groovy
pipeline {
    agent any
    
    stages {
        stage('Debug') {
            steps {
                sh 'echo "Build Number: ${BUILD_NUMBER}"'
                sh 'echo "Job Name: ${JOB_NAME}"'
                sh 'echo "Workspace: ${WORKSPACE}"'
                sh 'echo "Node Name: ${NODE_NAME}"'
                sh 'pwd'
                sh 'ls -la'
                sh 'env'
            }
        }
    }
}
```

## Additional Resources

- [Jenkins Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Shared Libraries](https://www.jenkins.io/doc/book/pipeline/shared-libraries/)
- [Pipeline Examples](https://www.jenkins.io/doc/pipeline/tour/hello-world/)
- [Blue Ocean Plugin](https://www.jenkins.io/doc/book/pipeline/blue-ocean/)