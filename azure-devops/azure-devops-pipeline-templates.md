# Azure DevOps Pipeline Templates

![Azure Pipelines](https://imgur.com/D1OFqlv.png)

Collection of reusable Azure DevOps pipeline templates for common CI/CD scenarios across different languages and frameworks.

## Table of Contents
1. [Introduction](#1-introduction)
2. [Basic Templates](#2-basic-templates)
3. [Language-Specific Templates](#3-language-specific-templates)
4. [Deployment Templates](#4-deployment-templates)
5. [Security Templates](#5-security-templates)
6. [Multi-Stage Templates](#6-multi-stage-templates)
7. [Template Usage](#7-template-usage)

---

## 1. Introduction

### 1.1. What are Pipeline Templates?

Pipeline templates are reusable YAML fragments that can be included in multiple pipelines. They help:
- Reduce code duplication
- Standardize build and deployment processes
- Enforce consistency across teams
- Simplify pipeline maintenance

### 1.2. Template Types

- **Step Templates**: Reusable steps within a job
- **Job Templates**: Reusable jobs with multiple steps
- **Stage Templates**: Reusable stages with multiple jobs
- **Variable Templates**: Reusable variable definitions

---

## 2. Basic Templates

### 2.1. Build Template

```yaml
# templates/build-template.yml
parameters:
  buildConfiguration: 'Release'
  projects: '**/*.csproj'
  dotnetVersion: '6.x'

steps:
- task: UseDotNet@2
  displayName: 'Install .NET SDK'
  inputs:
    packageType: 'sdk'
    version: ${{ parameters.dotnetVersion }}

- task: DotNetCoreCLI@2
  displayName: 'Restore Dependencies'
  inputs:
    command: 'restore'
    projects: ${{ parameters.projects }}

- task: DotNetCoreCLI@2
  displayName: 'Build Project'
  inputs:
    command: 'build'
    projects: ${{ parameters.projects }}
    arguments: '--configuration ${{ parameters.buildConfiguration }}'
```

**Usage:**
```yaml
# azure-pipelines.yml
steps:
- template: templates/build-template.yml
  parameters:
    buildConfiguration: 'Debug'
    projects: 'src/**/*.csproj'
    dotnetVersion: '7.x'
```

### 2.2. Test Template

```yaml
# templates/test-template.yml
parameters:
  testProjects: '**/*Tests/*.csproj'
  codeCoverageEnabled: true

steps:
- task: DotNetCoreCLI@2
  displayName: 'Run Tests'
  inputs:
    command: 'test'
    projects: ${{ parameters.testProjects }}
    arguments: '--no-build --configuration $(BuildConfiguration)'

- ${{ if eq(parameters.codeCoverageEnabled, true) }}:
  - task: PublishCodeCoverageResults@1
    displayName: 'Publish Code Coverage'
    inputs:
      codeCoverageTool: 'Cobertura'
      summaryFileLocation: '$(System.DefaultWorkingDirectory)/**/coverage.cobertura.xml'
```

**Usage:**
```yaml
steps:
- template: templates/test-template.yml
  parameters:
    testProjects: 'tests/**/*.csproj'
    codeCoverageEnabled: true
```

### 2.3. Docker Build Template

```yaml
# templates/docker-build-template.yml
parameters:
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'
  imageName: 'myapp'
  imageTag: '$(Build.BuildId)'
  buildContext: '$(Build.SourcesDirectory)'

steps:
- task: Docker@2
  displayName: 'Build Docker Image'
  inputs:
    repository: ${{ parameters.imageName }}
    command: build
    Dockerfile: ${{ parameters.dockerfilePath }}
    buildContext: ${{ parameters.buildContext }}
    tags: |
      ${{ parameters.imageTag }}
      latest
```

**Usage:**
```yaml
steps:
- template: templates/docker-build-template.yml
  parameters:
    dockerfilePath: 'Dockerfile'
    imageName: 'mycompany/myapp'
    imageTag: 'v1.0.0'
    buildContext: '.'
```

---

## 3. Language-Specific Templates

### 3.1. Node.js Template

```yaml
# templates/nodejs-template.yml
parameters:
  nodeVersion: '18.x'
  installCommand: 'install'
  scriptCommand: 'build'
  testCommand: 'test'

steps:
- task: UseNode@1
  displayName: 'Install Node.js'
  inputs:
    version: ${{ parameters.nodeVersion }}

- task: Npm@1
  displayName: 'Install Dependencies'
  inputs:
    command: ${{ parameters.installCommand }}

- task: Npm@1
  displayName: 'Build'
  inputs:
    command: 'custom'
    customCommand: ${{ parameters.scriptCommand }}

- task: Npm@1
  displayName: 'Run Tests'
  inputs:
    command: 'custom'
    customCommand: ${{ parameters.testCommand }}
```

**Usage:**
```yaml
steps:
- template: templates/nodejs-template.yml
  parameters:
    nodeVersion: '20.x'
    installCommand: 'ci'
    scriptCommand: 'run build'
    testCommand: 'run test:ci'
```

### 3.2. Python Template

```yaml
# templates/python-template.yml
parameters:
  pythonVersion: '3.10'
  requirementsFile: 'requirements.txt'
  testCommand: 'pytest'
  lintCommand: 'flake8'

steps:
- task: UsePythonVersion@0
  displayName: 'Install Python'
  inputs:
    versionSpec: ${{ parameters.pythonVersion }}

- script: |
    python -m pip install --upgrade pip
    pip install -r ${{ parameters.requirementsFile }}
  displayName: 'Install Dependencies'

- script: |
    ${{ parameters.lintCommand }} . --count --select=E9,F63,F7,F82 --show-source --statistics
  displayName: 'Lint Code'
  continueOnError: true

- script: |
    ${{ parameters.testCommand }} --junitxml=test-results.xml
  displayName: 'Run Tests'

- task: PublishTestResults@2
  displayName: 'Publish Test Results'
  inputs:
    testResultsFiles: '**/test-results.xml'
    testRunTitle: 'Python Tests'
  condition: succeededOrFailed()
```

**Usage:**
```yaml
steps:
- template: templates/python-template.yml
  parameters:
    pythonVersion: '3.11'
    requirementsFile: 'requirements-dev.txt'
    testCommand: 'pytest tests/ --cov'
    lintCommand: 'pylint'
```

### 3.3. Java/Maven Template

```yaml
# templates/maven-template.yml
parameters:
  mavenPOMFile: 'pom.xml'
  mavenOptions: '-Xmx3072m'
  javaHomeOption: 'JDKVersion'
  jdkVersionOption: '1.17'
  jdkArchitectureOption: 'x64'
  publishJUnitResults: true
  testResultsFiles: '**/surefire-reports/TEST-*.xml'

steps:
- task: Maven@3
  displayName: 'Maven Build'
  inputs:
    mavenPOMFile: ${{ parameters.mavenPOMFile }}
    mavenOptions: ${{ parameters.mavenOptions }}
    javaHomeOption: ${{ parameters.javaHomeOption }}
    jdkVersionOption: ${{ parameters.jdkVersionOption }}
    jdkArchitectureOption: ${{ parameters.jdkArchitectureOption }}
    publishJUnitResults: ${{ parameters.publishJUnitResults }}
    testResultsFiles: ${{ parameters.testResultsFiles }}
```

**Usage:**
```yaml
steps:
- template: templates/maven-template.yml
  parameters:
    mavenPOMFile: 'pom.xml'
    mavenOptions: '-Xmx4096m'
    jdkVersionOption: '1.21'
```

### 3.4. Go Template

```yaml
# templates/go-template.yml
parameters:
  goVersion: '1.21'
  packages: './...'
  testPackages: './...'
  buildOutput: 'main'

steps:
- task: GoTool@0
  displayName: 'Install Go'
  inputs:
    version: ${{ parameters.goVersion }}

- script: |
    go mod download
    go mod verify
  displayName: 'Download Dependencies'

- script: |
    go test -v -race -coverprofile=coverage.txt -covermode=atomic ${{ parameters.testPackages }}
  displayName: 'Run Tests'

- script: |
    go build -v -o ${{ parameters.buildOutput }} ${{ parameters.packages }}
  displayName: 'Build'

- task: PublishCodeCoverageResults@1
  displayName: 'Publish Code Coverage'
  inputs:
    codeCoverageTool: 'Cobertura'
    summaryFileLocation: '$(System.DefaultWorkingDirectory)/coverage.txt'
  condition: succeededOrFailed()
```

**Usage:**
```yaml
steps:
- template: templates/go-template.yml
  parameters:
    goVersion: '1.22'
    packages: './cmd/...'
    testPackages: './...'
    buildOutput: 'bin/app'
```

---

## 4. Deployment Templates

### 4.1. Azure Web App Deployment

```yaml
# templates/azure-webapp-deploy.yml
parameters:
  azureSubscription: ''
  appName: ''
  package: '$(Pipeline.Workspace)/drop/*.zip'
  runtimeStack: 'DOTNETCORE|6.0'

steps:
- task: AzureWebApp@1
  displayName: 'Deploy to Azure Web App'
  inputs:
    azureSubscription: ${{ parameters.azureSubscription }}
    appName: ${{ parameters.appName }}
    package: ${{ parameters.package }}
    runtimeStack: ${{ parameters.runtimeStack }}
```

**Usage:**
```yaml
steps:
- template: templates/azure-webapp-deploy.yml
  parameters:
    azureSubscription: 'My Azure Service Connection'
    appName: 'my-webapp'
    package: '$(Build.ArtifactStagingDirectory)/*.zip'
    runtimeStack: 'DOTNETCORE|7.0'
```

### 4.2. Kubernetes Deployment

```yaml
# templates/k8s-deploy.yml
parameters:
  kubernetesServiceConnection: ''
  manifests: []
  containers: []
  imagePullSecrets: []

steps:
- task: KubernetesManifest@0
  displayName: 'Deploy to Kubernetes'
  inputs:
    action: deploy
    kubernetesServiceConnection: ${{ parameters.kubernetesServiceConnection }}
    manifests: ${{ parameters.manifests }}
    containers: ${{ parameters.containers }}
    imagePullSecrets: ${{ parameters.imagePullSecrets }}
```

**Usage:**
```yaml
steps:
- template: templates/k8s-deploy.yml
  parameters:
    kubernetesServiceConnection: 'My K8s Connection'
    manifests: |
      $(Pipeline.Workspace)/manifests/deployment.yaml
      $(Pipeline.Workspace)/manifests/service.yaml
    containers: |
      myregistry.azurecr.io/myapp:$(Build.BuildId)
    imagePullSecrets: |
      my-registry-secret
```

### 4.3. Azure Function Deployment

```yaml
# templates/azure-function-deploy.yml
parameters:
  azureSubscription: ''
  appType: 'functionApp'
  appName: ''
  package: '$(Pipeline.Workspace)/drop/*.zip'
  runtimeStack: 'DOTNET|6.0'

steps:
- task: AzureFunctionApp@1
  displayName: 'Deploy to Azure Function'
  inputs:
    azureSubscription: ${{ parameters.azureSubscription }}
    appType: ${{ parameters.appType }}
    appName: ${{ parameters.appName }}
    package: ${{ parameters.package }}
    runtimeStack: ${{ parameters.runtimeStack }}
```

**Usage:**
```yaml
steps:
- template: templates/azure-function-deploy.yml
  parameters:
    azureSubscription: 'My Azure Service Connection'
    appName: 'my-function-app'
    package: '$(Build.ArtifactStagingDirectory)/*.zip'
    runtimeStack: 'DOTNET|7.0'
```

### 4.4. VM Deployment

```yaml
# templates/vm-deploy.yml
parameters:
  azureSubscription: ''
  resourceGroupName: ''
  vmNames: ''
  scripts: []
  workingDirectory: ''

steps:
- task: AzureVMShell@0
  displayName: 'Deploy to VM'
  inputs:
    azureSubscription: ${{ parameters.azureSubscription }}
    ResourceGroupName: ${{ parameters.resourceGroupName }}
    VMs: ${{ parameters.vmNames }}
    ScriptType: 'InlineScript'
    InlineScript: |
      ${{ parameters.scripts }}
    ScriptPath: ''
    ScriptArguments: ''
    InitializeScriptPath: ''
    InitializeScriptArguments: ''
    RunOptions: 'inline'
    workingDirectory: ${{ parameters.workingDirectory }}
```

**Usage:**
```yaml
steps:
- template: templates/vm-deploy.yml
  parameters:
    azureSubscription: 'My Azure Service Connection'
    resourceGroupName: 'MyResourceGroup'
    vmNames: 'my-vm-1,my-vm-2'
    scripts: |
      cd /home/azure/app
      git pull origin main
      docker-compose down
      docker-compose up -d
    workingDirectory: '/home/azure/app'
```

---

## 5. Security Templates

### 5.1. SAST Scan Template

```yaml
# templates/sast-scan-template.yml
parameters:
  scanType: 'SonarQube'
  sonarQubeServiceConnection: ''
  projectKey: ''
  projectName: ''

steps:
- ${{ if eq(parameters.scanType, 'SonarQube') }}:
  - task: SonarQubePrepare@5
    displayName: 'Prepare SonarQube Analysis'
    inputs:
      SonarQube: ${{ parameters.sonarQubeServiceConnection }}
      scannerMode: 'MSBuild'
      projectKey: ${{ parameters.projectKey }}
      projectName: ${{ parameters.projectName }}

  - task: SonarQubeAnalyze@5
    displayName: 'Run SonarQube Analysis'

  - task: SonarQubePublish@5
    displayName: 'Publish SonarQube Results'
    inputs:
      pollingTimeoutSec: '300'
```

**Usage:**
```yaml
steps:
- template: templates/sast-scan-template.yml
  parameters:
    scanType: 'SonarQube'
    sonarQubeServiceConnection: 'My SonarQube Connection'
    projectKey: 'my-project-key'
    projectName: 'My Project'
```

### 5.2. Container Scan Template

```yaml
# templates/container-scan-template.yml
parameters:
  imageName: ''
  severityThreshold: 'HIGH'
  failOnSeverity: 'CRITICAL'

steps:
- task: Docker@2
  displayName: 'Pull Image for Scanning'
  inputs:
    command: pull
    arguments: ${{ parameters.imageName }}

- script: |
    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
      aquasec/trivy image --severity ${{ parameters.severityThreshold }} \
      --exit-code 1 ${{ parameters.imageName }}
  displayName: 'Scan Container Image'
  continueOnError: ${{ eq(parameters.failOnSeverity, '') }}
```

**Usage:**
```yaml
steps:
- template: templates/container-scan-template.yml
  parameters:
    imageName: 'mycompany/myapp:latest'
    severityThreshold: 'HIGH,CRITICAL'
    failOnSeverity: 'CRITICAL'
```

### 5.3. Dependency Check Template

```yaml
# templates/dependency-check-template.yml
parameters:
  scanType: 'npm'
  failOnVulnerabilities: true

steps:
- ${{ if eq(parameters.scanType, 'npm') }}:
  - task: Npm@1
    displayName: 'Run npm audit'
    inputs:
      command: 'custom'
      customCommand: 'audit --audit-level=moderate'
    continueOnError: ${{ not(parameters.failOnVulnerabilities) }}

- ${{ if eq(parameters.scanType, 'nuget') }}:
  - script: |
      dotnet tool install --global NuGetKeyVaultSignTool
      dotnet list package --vulnerable
    displayName: 'Check NuGet Vulnerabilities'
    continueOnError: ${{ not(parameters.failOnVulnerabilities) }}

- ${{ if eq(parameters.scanType, 'pip') }}:
  - script: |
      pip install safety
      safety check --json
    displayName: 'Check Python Dependencies'
    continueOnError: ${{ not(parameters.failOnVulnerabilities) }}
```

**Usage:**
```yaml
steps:
- template: templates/dependency-check-template.yml
  parameters:
    scanType: 'npm'
    failOnVulnerabilities: true
```

---

## 6. Multi-Stage Templates

### 6.1. Build and Deploy Template

```yaml
# templates/build-deploy-template.yml
parameters:
  buildConfiguration: 'Release'
  deployEnvironment: 'Production'
  azureSubscription: ''
  appName: ''

stages:
- stage: Build
  displayName: 'Build Stage'
  jobs:
  - job: Build
    displayName: 'Build Job'
    steps:
    - template: build-template.yml
      parameters:
        buildConfiguration: ${{ parameters.buildConfiguration }}

    - template: test-template.yml

    - task: PublishBuildArtifacts@1
      displayName: 'Publish Artifacts'
      inputs:
        PathtoPublish: '$(Build.ArtifactStagingDirectory)'
        ArtifactName: 'drop'

- stage: Deploy
  displayName: 'Deploy Stage'
  dependsOn: Build
  condition: succeeded()
  jobs:
  - deployment: Deploy
    displayName: 'Deploy Job'
    environment: ${{ parameters.deployEnvironment }}
    strategy:
      runOnce:
        deploy:
          steps:
          - template: azure-webapp-deploy.yml
            parameters:
              azureSubscription: ${{ parameters.azureSubscription }}
              appName: ${{ parameters.appName }}
```

**Usage:**
```yaml
# azure-pipelines.yml
trigger:
- main

pool:
  vmImage: 'ubuntu-latest'

extends:
  template: templates/build-deploy-template.yml
  parameters:
    buildConfiguration: 'Release'
    deployEnvironment: 'Production'
    azureSubscription: 'My Azure Connection'
    appName: 'my-webapp'
```

### 6.2. Multi-Environment Template

```yaml
# templates/multi-env-deploy-template.yml
parameters:
  environments: []
  buildConfiguration: 'Release'

stages:
- stage: Build
  displayName: 'Build Stage'
  jobs:
  - job: Build
    steps:
    - template: build-template.yml
      parameters:
        buildConfiguration: ${{ parameters.buildConfiguration }}

    - task: PublishBuildArtifacts@1
      displayName: 'Publish Artifacts'
      inputs:
        PathtoPublish: '$(Build.ArtifactStagingDirectory)'
        ArtifactName: 'drop'

- ${{ each env in parameters.environments }}:
  - stage: Deploy_${{ env.name }}
    displayName: 'Deploy to ${{ env.displayName }}'
    dependsOn: Build
    condition: succeeded()
    jobs:
    - deployment: Deploy
      displayName: 'Deploy to ${{ env.displayName }}'
      environment: ${{ env.name }}
      strategy:
        runOnce:
          deploy:
            steps:
            - template: azure-webapp-deploy.yml
              parameters:
                azureSubscription: ${{ env.azureSubscription }}
                appName: ${{ env.appName }}
```

**Usage:**
```yaml
# azure-pipelines.yml
trigger:
- main

pool:
  vmImage: 'ubuntu-latest'

extends:
  template: templates/multi-env-deploy-template.yml
  parameters:
    buildConfiguration: 'Release'
    environments:
      - name: 'Dev'
        displayName: 'Development'
        azureSubscription: 'Dev Azure Connection'
        appName: 'myapp-dev'
      - name: 'QA'
        displayName: 'QA'
        azureSubscription: 'QA Azure Connection'
        appName: 'myapp-qa'
      - name: 'Production'
        displayName: 'Production'
        azureSubscription: 'Prod Azure Connection'
        appName: 'myapp-prod'
```

---

## 7. Template Usage

### 7.1. Variable Templates

```yaml
# templates/variables.yml
variables:
  buildConfiguration: 'Release'
  dotnetVersion: '6.x'
  dockerRegistry: 'myregistry.azurecr.io'
  imageName: 'myapp'
```

**Usage:**
```yaml
# azure-pipelines.yml
variables:
- template: templates/variables.yml

pool:
  vmImage: 'ubuntu-latest'

steps:
- script: echo $(buildConfiguration)
```

### 7.2. Conditional Templates

```yaml
# azure-pipelines.yml
parameters:
- name: runTests
  type: boolean
  default: true
- name: runSecurityScan
  type: boolean
  default: false

steps:
- template: build-template.yml

- ${{ if eq(parameters.runTests, true) }}:
  - template: test-template.yml

- ${{ if eq(parameters.runSecurityScan, true) }}:
  - template: sast-scan-template.yml
```

**Usage:**
```yaml
# azure-pipelines.yml
extends:
  template: templates/pipeline-with-params.yml
  parameters:
    runTests: true
    runSecurityScan: true
```

### 7.3. Template Loops

```yaml
# templates/multi-project-build.yml
parameters:
  projects: []

steps:
- ${{ each project in parameters.projects }}:
  - task: DotNetCoreCLI@2
    displayName: 'Build ${{ project.name }}'
    inputs:
      command: 'build'
      projects: ${{ project.path }}
      arguments: '--configuration ${{ project.configuration }}'
```

**Usage:**
```yaml
steps:
- template: templates/multi-project-build.yml
  parameters:
    projects:
      - name: 'API'
        path: 'src/Api/Api.csproj'
        configuration: 'Release'
      - name: 'Web'
        path: 'src/Web/Web.csproj'
        configuration: 'Release'
      - name: 'Worker'
        path: 'src/Worker/Worker.csproj'
        configuration: 'Debug'
```

---

## 8. Best Practices

### 8.1. Template Organization

```
templates/
├── steps/
│   ├── build-step.yml
│   ├── test-step.yml
│   └── deploy-step.yml
├── jobs/
│   ├── build-job.yml
│   └── deploy-job.yml
├── stages/
│   ├── build-stage.yml
│   └── deploy-stage.yml
└── variables/
    ├── common-variables.yml
    └── environment-variables.yml
```

### 8.2. Template Naming Conventions

- Use descriptive names: `build-dotnet-template.yml`
- Include parameters in filename: `build-with-params-template.yml`
- Group by function: `security/`, `deployment/`, `build/`

### 8.3. Template Documentation

```yaml
# templates/build-template.yml
# Description: Build template for .NET projects
# Parameters:
#   - buildConfiguration: Debug or Release (default: Release)
#   - projects: Glob pattern for project files (default: **/*.csproj)
#   - dotnetVersion: .NET SDK version (default: 6.x)
# Usage:
#   - template: templates/build-template.yml
#     parameters:
#       buildConfiguration: 'Debug'
#       projects: 'src/**/*.csproj'
parameters:
  buildConfiguration: 'Release'
  projects: '**/*.csproj'
  dotnetVersion: '6.x'

steps:
  # ... template content
```

---

## References

- [Azure DevOps Templates Documentation](https://docs.microsoft.com/azure/devops/pipelines/yaml-template)
- [Template Schema Reference](https://docs.microsoft.com/azure/devops/pipelines/yaml-schema)
- [Reusable Pipeline Templates](https://docs.microsoft.com/azure/devops/pipelines/process/templates)

---

## By [Harshhaa Reddy](https://www.github.com/NotHarshhaa)
