![jenkins-pipeline](https://www.jenkins.io/images/pipeline/realworld-pipeline-flow.png)

# Jenkins Pipeline Templates Guide

## 1. Declarative Pipeline Templates

### 1.1. Basic Java Application Pipeline

```groovy
pipeline {
    agent any
    
    tools {
        maven 'Maven 3.8.6'
        jdk 'JDK 17'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/yourusername/your-repo.git'
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean package'
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
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                sh 'docker build -t myapp:${BUILD_NUMBER} .'
                sh 'docker push myapp:${BUILD_NUMBER}'
                sh 'kubectl apply -f k8s/staging/'
            }
        }
    }
    
    post {
        success {
            slackSend channel: '#deployments',
                      color: 'good',
                      message: "Pipeline succeeded: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
        failure {
            slackSend channel: '#deployments',
                      color: 'danger',
                      message: "Pipeline failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
    }
}
```

### 1.2. Node.js Application Pipeline

```groovy
pipeline {
    agent {
        docker {
            image 'node:16'
        }
    }
    
    stages {
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        
        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }
        
        stage('Test') {
            steps {
                sh 'npm test'
            }
        }
        
        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
        
        stage('Docker Build & Push') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                        def app = docker.build("myapp:${env.BUILD_NUMBER}")
                        app.push()
                    }
                }
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
    
    stage('Preparation') {
        mvnHome = tool 'Maven 3.8.6'
        checkout scm
    }
    
    stage('Build & Test') {
        try {
            sh "'${mvnHome}/bin/mvn' clean test"
        } catch (err) {
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
    }
    
    stage('Build Docker Image') {
        dockerImage = docker.build("myapp:${env.BRANCH_NAME}-${env.BUILD_NUMBER}")
    }
    
    if (env.BRANCH_NAME == 'main') {
        stage('Deploy to Production') {
            dockerImage.push()
            sh 'kubectl apply -f k8s/production/'
        }
    } else if (env.BRANCH_NAME == 'develop') {
        stage('Deploy to Staging') {
            dockerImage.push()
            sh 'kubectl apply -f k8s/staging/'
        }
    }
}
```

## 3. Advanced Pipeline Features

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

## 4. Best Practices

### 4.1. Pipeline Organization
- Keep pipelines simple and focused
- Use shared libraries for common functionality
- Implement proper error handling
- Use environment variables for configuration
- Implement proper cleanup in post sections

### 4.2. Security Best Practices
- Use credentials management for sensitive data
- Implement proper access controls
- Regular security scanning of dependencies
- Use approved base images for containers
- Implement proper secrets management

### 4.3. Performance Optimization
- Use parallel execution where possible
- Implement proper caching strategies
- Clean up workspace regularly
- Use lightweight containers
- Implement proper resource constraints

## 5. Troubleshooting

### 5.1. Common Issues
1. Pipeline syntax errors
2. Resource constraints
3. Network connectivity issues
4. Permission problems
5. Tool version mismatches

### 5.2. Debugging Tips
1. Use `sh 'set -x'` for debug output
2. Check Jenkins logs
3. Validate Jenkinsfile syntax
4. Test scripts locally first
5. Use pipeline replay feature 