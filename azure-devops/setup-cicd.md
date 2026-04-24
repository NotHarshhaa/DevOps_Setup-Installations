# Azure DevOps Installation and Configuration Guide

![Azure DevOps](https://imgur.com/D1OFqlv.png)

Comprehensive guide for setting up Azure DevOps Services, self-hosted agents, CI/CD pipelines, and integration with various Azure services.

## Table of Contents
1. [Introduction](#1-introduction)
2. [Azure DevOps Services Setup](#2-azure-devops-services-setup)
3. [Azure DevOps Server Installation](#3-azure-devops-server-installation)
4. [Self-Hosted Agents Configuration](#4-self-hosted-agents-configuration)
5. [CI/CD Pipeline Configuration](#5-cicd-pipeline-configuration)
6. [Azure Repos Setup](#6-azure-repos-setup)
7. [Azure Artifacts Configuration](#7-azure-artifacts-configuration)
8. [Azure Boards Setup](#8-azure-boards-setup)
9. [Integration with Other Services](#9-integration-with-other-services)
10. [Best Practices](#10-best-practices)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Introduction

### 1.1. What is Azure DevOps?

Azure DevOps is a suite of development tools provided by Microsoft for software development teams. It includes:
- **Azure Boards**: Agile planning, work item tracking, and visualization
- **Azure Repos**: Git repositories and code reviews
- **Azure Pipelines**: CI/CD for build, test, and deployment
- **Azure Test Plans**: Manual and exploratory testing
- **Azure Artifacts**: Package management

### 1.2. Azure DevOps Services vs Azure DevOps Server

**Azure DevOps Services (Cloud)**
- Hosted by Microsoft
- No infrastructure management
- Automatic updates
- Free for up to 5 users

**Azure DevOps Server (On-Premises)**
- Self-hosted solution
- Full control over data
- Requires infrastructure maintenance
- formerly known as TFS (Team Foundation Server)

---

## 2. Azure DevOps Services Setup

### 2.1. Create Azure DevOps Account

#### Step 1: Sign Up
1. Navigate to [https://dev.azure.com](https://dev.azure.com)
2. Click on "Start free" or "Sign in"
3. Sign in with your Microsoft account or create a new one
4. Choose your preferred organization URL (e.g., `https://dev.azure.com/yourcompany`)

#### Step 2: Create a Project
1. After signing in, click "Create project"
2. Fill in the project details:
   - **Project name**: MyFirstProject
   - **Description**: Description of your project
   - **Visibility**: Private or Public
   - **Version control**: Git or TFVC (Git recommended)
   - **Work item process**: Agile, Scrum, or CMMI
3. Click "Create"

### 2.2. Configure Project Settings

#### Add Team Members
1. Go to **Project Settings** > **Teams**
2. Click on your team
3. Click **Add members**
4. Enter email addresses and select access level (Basic, Stakeholder, etc.)

#### Configure Permissions
1. Go to **Project Settings** > **Permissions**
2. Configure access for different groups:
   - **Project Administrators**: Full control
   - **Project Contributors**: Can contribute code and work items
   - **Project Readers**: Read-only access
   - **Project Valid Users**: All users with access

---

## 3. Azure DevOps Server Installation

### 3.1. System Requirements

**Hardware Requirements**
- **CPU**: 4-core processor (64-bit)
- **RAM**: 8 GB minimum (16 GB recommended)
- **Disk**: 100 GB for application tier, 500 GB for data tier
- **Network**: 1 Gbps

**Software Requirements**
- **OS**: Windows Server 2016/2019/2022
- **SQL Server**: SQL Server 2017/2019/2022 (Express/Standard/Enterprise)
- **Browser**: Latest version of Chrome, Edge, or Firefox

### 3.2. Installation Steps

#### Step 1: Download Azure DevOps Server
1. Navigate to [https://azure.microsoft.com/en-us/services/devops/server/](https://azure.microsoft.com/en-us/services/devops/server/)
2. Download the Azure DevOps Server installer

#### Step 2: Install SQL Server
```powershell
# Install SQL Server 2019 Express (if not already installed)
# Download from: https://www.microsoft.com/en-us/sql-server/sql-server-downloads
```

#### Step 3: Install Azure DevOps Server
1. Run the Azure DevOps Server installer as Administrator
2. Click on "Install Azure DevOps Server"
3. Accept the license terms
4. Choose installation type:
   - **Basic**: Single-server installation
   - **Advanced**: Custom configuration
5. Configure the following:
   - **Application Tier**: Configure web access URL
   - **Database**: Connect to SQL Server instance
   - **Search**: Configure search service (optional)
6. Click "Install" and wait for completion

#### Step 4: Configure Azure DevOps Server
1. Open the Azure DevOps Server Administration Console
2. Create a new collection:
   - Collection Name: DefaultCollection
   - Description: Default project collection
3. Configure the reporting and analysis services (optional)

#### Step 5: Install Build and Release Agents
```powershell
# Download and install build agents
# Navigate to: https://dev.azure.com/{organization}/_admin/_AgentPool
# Download the agent package
# Extract and run the config.cmd
```

---

## 4. Self-Hosted Agents Configuration

### 4.1. Windows Self-Hosted Agent

#### Step 1: Create Agent Pool
1. Go to **Project Settings** > **Agent pools**
2. Click "Add pool"
3. Select "Self-hosted"
4. Name the pool (e.g., "Windows Agents")
5. Click "Create"

#### Step 2: Install Agent on Windows
```powershell
# Create agent directory
mkdir C:\agent
cd C:\agent

# Download the agent
# Navigate to: https://dev.azure.com/{organization}/_settings/agentpools
# Download the Windows agent package

# Extract the agent
Add-Type -AssemblyName System.IO.Compression.FileSystem
[System.IO.Compression.ZipFile]::ExtractToDirectory("agent.zip", ".")

# Configure the agent
.\config.cmd

# Follow the prompts:
# - Enter server URL: https://dev.azure.com/{organization}
# - Enter authentication type: PAT
# - Enter personal access token
# - Enter agent pool name: Windows Agents
# - Enter agent name: (accept default or custom)
# - Enter work folder: _work (default)
# - Enter run as service: Y (recommended)
# - Enter service account: NETWORK SERVICE (default)

# Start the agent
.\run.cmd
```

#### Step 3: Configure Agent Capabilities
```powershell
# Install required software on the agent
# Example: Install Docker
Install-Module -Name DockerMsftProvider -Repository PSGallery -Force
Install-Package -Name docker -ProviderName DockerMsftProvider -Force

# Example: Install Node.js
choco install nodejs -y

# Example: Install Python
choco install python -y

# Example: Install .NET SDK
choco install dotnet-sdk -y
```

### 4.2. Linux Self-Hosted Agent

#### Step 1: Install Prerequisites
```bash
# Update system
sudo apt-get update
sudo apt-get upgrade -y

# Install required packages
sudo apt-get install -y curl libunwind8 gettext

# Install .NET Core Runtime (for Azure DevOps agent)
wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt-get update
sudo apt-get install -y dotnet-runtime-6.0
```

#### Step 2: Install Agent on Linux
```bash
# Create agent directory
mkdir ~/agent
cd ~/agent

# Download the agent
# Navigate to: https://dev.azure.com/{organization}/_settings/agentpools
# Download the Linux agent package

# Extract the agent
tar zxvf agent.tar.gz

# Configure the agent
./config.sh

# Follow the prompts:
# - Enter server URL: https://dev.azure.com/{organization}
# - Enter authentication type: PAT
# - Enter personal access token
# - Enter agent pool name: Linux Agents
# - Enter agent name: (accept default or custom)
# - Enter work folder: _work (default)

# Install the agent as a service
sudo ./svc.sh install
sudo ./svc.sh start
```

### 4.3. Docker Self-Hosted Agent

#### Step 1: Create Dockerfile
```dockerfile
FROM ubuntu:22.04

# Install dependencies
RUN apt-get update && apt-get install -y \
    curl \
    libunwind8 \
    gettext \
    && rm -rf /var/lib/apt/lists/*

# Install .NET Core Runtime
RUN wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb && \
    dpkg -i packages-microsoft-prod.deb && \
    apt-get update && \
    apt-get install -y dotnet-runtime-6.0 && \
    rm packages-microsoft-prod.deb

# Install additional tools
RUN apt-get update && apt-get install -y \
    docker.io \
    git \
    nodejs \
    npm \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Create agent directory
WORKDIR /agent

# Copy agent files (downloaded separately)
COPY . .

# Configure and run agent
CMD ./config.sh --unattended --url ${AZP_URL} --auth PAT --token ${AZP_TOKEN} --pool ${AZP_POOL} --agent ${AZP_AGENT_NAME} --acceptTeeEula && ./run.sh
```

#### Step 2: Build and Run Docker Agent
```bash
# Build the image
docker build -t azure-devops-agent .

# Run the container
docker run -d \
  -e AZP_URL=https://dev.azure.com/{organization} \
  -e AZP_TOKEN={your-pat-token} \
  -e AZP_POOL=Docker Agents \
  -e AZP_AGENT_NAME=docker-agent-1 \
  --name azure-devops-agent \
  azure-devops-agent

# Mount Docker socket for Docker-in-Docker
docker run -d \
  -e AZP_URL=https://dev.azure.com/{organization} \
  -e AZP_TOKEN={your-pat-token} \
  -e AZP_POOL=Docker Agents \
  -e AZP_AGENT_NAME=docker-agent-1 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --name azure-devops-agent \
  azure-devops-agent
```

---

## 5. CI/CD Pipeline Configuration

### 5.1. Classic Pipeline vs YAML Pipeline

**Classic Pipeline**
- GUI-based configuration
- Easier for beginners
- Limited flexibility
- Not version controlled

**YAML Pipeline** (Recommended)
- Code-based configuration
- Version controlled
- More flexible and powerful
- Supports multi-stage pipelines

### 5.2. Create YAML Pipeline

#### Step 1: Create Pipeline File
Create `azure-pipelines.yml` in your repository root:

```yaml
# Basic pipeline structure
trigger:
  branches:
    include:
    - main
    - develop
  paths:
    exclude:
    - docs/*
    - README.md

pr:
  branches:
    include:
    - main
    - develop

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  dotnetSdkVersion: '6.x'

stages:
- stage: Build
  displayName: 'Build Stage'
  jobs:
  - job: Build
    displayName: 'Build Job'
    steps:
    - task: UseDotNet@2
      displayName: 'Install .NET SDK'
      inputs:
        packageType: 'sdk'
        version: $(dotnetSdkVersion)

    - task: DotNetCoreCLI@2
      displayName: 'Restore Dependencies'
      inputs:
        command: 'restore'
        projects: '**/*.csproj'

    - task: DotNetCoreCLI@2
      displayName: 'Build Project'
      inputs:
        command: 'build'
        projects: '**/*.csproj'
        arguments: '--configuration $(buildConfiguration)'

    - task: DotNetCoreCLI@2
      displayName: 'Run Tests'
      inputs:
        command: 'test'
        projects: '**/*Tests/*.csproj'
        arguments: '--configuration $(buildConfiguration) --no-build'

    - task: PublishBuildArtifacts@1
      displayName: 'Publish Artifacts'
      inputs:
        PathtoPublish: '$(Build.ArtifactStagingDirectory)'
        ArtifactName: 'drop'
        publishLocation: 'Container'

- stage: Deploy
  displayName: 'Deploy Stage'
  dependsOn: Build
  condition: succeeded()
  jobs:
  - deployment: Deploy
    displayName: 'Deploy Job'
    environment: 'Production'
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: drop

          - task: AzureWebApp@1
            displayName: 'Deploy to Azure Web App'
            inputs:
              azureSubscription: 'Your Azure Service Connection'
              appName: 'Your-Web-App-Name'
              package: '$(Pipeline.Workspace)/drop/*.zip'
```

### 5.3. Python CI/CD Pipeline

```yaml
trigger:
- main

pool:
  vmImage: 'ubuntu-latest'

variables:
  pythonVersion: '3.10'

stages:
- stage: Build
  displayName: 'Build and Test'
  jobs:
  - job: Build
    displayName: 'Build Job'
    steps:
    - task: UsePythonVersion@0
      displayName: 'Install Python'
      inputs:
        versionSpec: $(pythonVersion)

    - script: |
        python -m venv venv
        source venv/bin/activate
        pip install --upgrade pip
        pip install -r requirements.txt
        pip install pytest pytest-cov flake8
      displayName: 'Setup Environment'

    - script: |
        flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
        flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics
      displayName: 'Lint Code'

    - script: |
        pytest tests/ --cov=. --cov-report=xml --cov-report=html
      displayName: 'Run Tests'

    - task: PublishCodeCoverageResults@1
      displayName: 'Publish Coverage'
      inputs:
        codeCoverageTool: 'Cobertura'
        summaryFileLocation: '$(System.DefaultWorkingDirectory)/**/coverage.xml'

    - script: |
        python setup.py sdist bdist_wheel
      displayName: 'Build Package'

    - task: PublishBuildArtifacts@1
      displayName: 'Publish Artifacts'
      inputs:
        PathtoPublish: '$(System.DefaultWorkingDirectory)/dist'
        ArtifactName: 'python-package'

- stage: Deploy
  displayName: 'Deploy to Azure Artifacts'
  dependsOn: Build
  condition: succeeded()
  jobs:
  - job: Deploy
    displayName: 'Deploy Job'
    steps:
    - task: UsePythonVersion@0
      inputs:
        versionSpec: $(pythonVersion)

    - script: |
        pip install twine
        twine upload --repository-url $(AZURE_ARTIFACTS_URL) --skip-existing dist/*
      displayName: 'Publish to Azure Artifacts'
      env:
        TWINE_USERNAME: $(AZURE_ARTIFACTS_USER)
        TWINE_PASSWORD: $(AZURE_ARTIFACTS_PASSWORD)
```

### 5.4. Docker CI/CD Pipeline

```yaml
trigger:
- main

pool:
  vmImage: 'ubuntu-latest'

variables:
  dockerRegistryServiceConnection: 'Your Docker Registry Service Connection'
  imageRepository: 'your-registry/your-image'
  containerRegistry: 'your-registry.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'
  tag: '$(Build.BuildId)'

stages:
- stage: Build
  displayName: 'Build Docker Image'
  jobs:
  - job: Build
    displayName: 'Build Job'
    steps:
    - task: Docker@2
      displayName: 'Build Image'
      inputs:
        repository: $(imageRepository)
        command: build
        Dockerfile: $(dockerfilePath)
        tags: |
          $(tag)
          latest

    - task: Docker@2
      displayName: 'Push Image'
      inputs:
        containerRegistry: $(dockerRegistryServiceConnection)
        repository: $(imageRepository)
        command: push
        tags: |
          $(tag)
          latest

- stage: Deploy
  displayName: 'Deploy to Kubernetes'
  dependsOn: Build
  condition: succeeded()
  jobs:
  - deployment: Deploy
    displayName: 'Deploy to K8s'
    environment: 'Production'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            displayName: 'Deploy to K8s'
            inputs:
              action: deploy
              kubernetesServiceConnection: 'Your K8s Service Connection'
              manifests: |
                $(Pipeline.Workspace)/manifests/deployment.yaml
                $(Pipeline.Workspace)/manifests/service.yaml
              containers: |
                $(containerRegistry)/$(imageRepository):$(tag)
```

---

## 6. Azure Repos Setup

### 6.1. Create a New Repository

#### Step 1: Create Repository
1. Navigate to **Repos** > **Files**
2. Click "New repository"
3. Choose repository type:
   - **Git** (recommended)
   - **TFVC** (legacy)
4. Enter repository name
5. Click "Create"

#### Step 2: Clone the Repository
```bash
# Clone using HTTPS
git clone https://dev.azure.com/{organization}/{project}/_git/{repository}

# Clone using SSH (requires SSH key setup)
git clone git@ssh.dev.azure.com:v3/{organization}/{project}/{repository}
```

### 6.2. Configure Git

#### Set Up Git User
```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

#### Set Up SSH Keys
```bash
# Generate SSH key
ssh-keygen -t rsa -b 4096 -C "your.email@example.com"

# Start SSH agent
eval $(ssh-agent -s)

# Add SSH key
ssh-add ~/.ssh/id_rsa

# Copy public key
cat ~/.ssh/id_rsa.pub
```

Add the public key to Azure DevOps:
1. Go to **User Settings** > **SSH public keys**
2. Click "New key"
3. Paste the public key
4. Click "Add"

### 6.3. Branch Policies

#### Configure Branch Protection
1. Navigate to **Repos** > **Branches**
2. Click on the branch (e.g., main)
3. Click "Branch policies"
4. Configure policies:
   - **Require a minimum number of reviewers**: Set to 2
   - **Check for linked work items**: Enable
   - **Check for comment resolution**: Enable
   - **Limit merge types**: Enable "Squash merge"
   - **Build validation**: Add pipeline validation

#### Configure Pull Request Templates
Create `.azuredevops/pull_request_template.md`:
```markdown
## Description
<!-- Describe your changes here -->

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Testing
<!-- Describe how you tested your changes -->

## Checklist
- [ ] Code follows the style guidelines
- [ ] Self-review completed
- [ ] Comments added for complex code
- [ ] Unit tests added/updated
- [ ] Documentation updated
```

---

## 7. Azure Artifacts Configuration

### 7.1. Create a Feed

#### Step 1: Create Feed
1. Navigate to **Artifacts** > **Create feed**
2. Choose feed type:
   - **Project-scoped**: Visible only within the project
   - **Organization-scoped**: Visible across the organization
3. Enter feed name and visibility
4. Click "Create"

### 7.2. Configure NuGet Feed

#### Step 1: Connect to Feed
```bash
# Add NuGet source
dotnet nuget add source https://pkgs.dev.azure.com/{organization}/{project}/_packaging/{feed}/nuget/v3/index.json \
  -n {feed} \
  -u {username} \
  -p {pat-token}
```

#### Step 2: Publish Package
```bash
# Pack the project
dotnet pack -c Release

# Push to Azure Artifacts
dotnet nuget push \
  --source https://pkgs.dev.azure.com/{organization}/{project}/_packaging/{feed}/nuget/v3/index.json \
  --api-key AzureDevOps \
  bin/Release/*.nupkg
```

### 7.3. Configure Python Feed

#### Step 1: Connect to Feed
```bash
# Install Azure Artifacts keyring
pip install keyring artifacts-keyring

# Add pip source
pip index versions \
  --index-url https://pkgs.dev.azure.com/{organization}/{project}/_packaging/{feed}/pypi/simple/
```

#### Step 2: Publish Package
```bash
# Install twine
pip install twine

# Publish to Azure Artifacts
twine upload \
  --repository-url https://pkgs.dev.azure.com/{organization}/{project}/_packaging/{feed}/pypi/upload/ \
  dist/*
```

### 7.4. Configure npm Feed

#### Step 1: Connect to Feed
```bash
# Create .npmrc file
echo "registry=https://pkgs.dev.azure.com/{organization}/{project}/_packaging/{feed}/npm/registry/" > .npmrc
echo "always-auth=true" >> .npmrc

# Set up authentication
npm config set //pkgs.dev.azure.com/{organization}/{project}/_packaging/{feed}/npm/registry/:_authToken {pat-token}
```

#### Step 2: Publish Package
```bash
# Publish to Azure Artifacts
npm publish
```

---

## 8. Azure Boards Setup

### 8.1. Configure Work Items

#### Create Work Item Types
1. Navigate to **Boards** > **Work Items**
2. Click "New Work Item"
3. Select work item type:
   - **User Story**: Feature requirements
   - **Bug**: Defects and issues
   - **Task**: Implementation tasks
   - **Epic**: Large initiatives
   - **Feature**: Group of user stories

#### Configure Work Item Templates
1. Go to **Project Settings** > **Process**
2. Select your process (Agile, Scrum, or CMMI)
3. Customize work item types and fields

### 8.2. Configure Sprints

#### Create Sprint Iterations
1. Navigate to **Boards** > **Sprints**
2. Click "New iteration"
3. Enter iteration name and dates
4. Add to the iteration path

#### Configure Sprint Board
1. Go to **Boards** > **Boards**
2. Select your sprint
3. Configure columns:
   - **New**: New work items
   - **Active**: Work in progress
   - **Resolved**: Completed but not tested
   - **Closed**: Completed and tested

### 8.3. Configure Dashboards

#### Create Custom Dashboard
1. Navigate to **Overview** > **Dashboards**
2. Click "New dashboard"
3. Add widgets:
   - **Velocity Chart**: Track sprint velocity
   - **Burndown Chart**: Track sprint progress
   - **Work Item Summary**: Overview of work items
   - **Build Summary**: Build status
   - **Pull Request Status**: PR status

---

## 9. Integration with Other Services

### 9.1. Azure Integration

#### Connect to Azure Resources
1. Go to **Project Settings** > **Service connections**
2. Click "New service connection"
3. Select "Azure Resource Manager"
4. Choose authentication method:
   - **Service principal (automatic)**: Recommended
   - **Service principal (manual)**: For advanced scenarios
   - **Managed identity**: For Azure DevOps Server

#### Deploy to Azure Web App
```yaml
- task: AzureWebApp@1
  displayName: 'Deploy to Azure Web App'
  inputs:
    azureSubscription: 'Your Azure Service Connection'
    appName: 'Your-Web-App-Name'
    package: '$(Pipeline.Workspace)/drop/*.zip'
```

#### Deploy to Azure Kubernetes Service
```yaml
- task: KubernetesManifest@0
  displayName: 'Deploy to AKS'
  inputs:
    action: deploy
    kubernetesServiceConnection: 'Your K8s Service Connection'
    manifests: |
      $(Pipeline.Workspace)/manifests/deployment.yaml
      $(Pipeline.Workspace)/manifests/service.yaml
```

### 9.2. GitHub Integration

#### Connect GitHub Repository
1. Go to **Project Settings** > **Service connections**
2. Click "New service connection"
3. Select "GitHub"
4. Authorize Azure DevOps to access your GitHub account

#### Build from GitHub
```yaml
trigger:
- main

resources:
  repositories:
  - repository: mygithubrepo
    type: github
    name: YourOrg/YourRepo
    endpoint: YourGitHubServiceConnection

pool:
  vmImage: 'ubuntu-latest'

steps:
- checkout: self
- checkout: mygithubrepo
```

### 9.3. Slack Integration

#### Configure Slack Notifications
1. Install the Azure DevOps app in Slack
2. In Azure DevOps, go to **Project Settings** > **Service hooks**
3. Click "New subscription"
4. Select "Slack"
5. Configure triggers:
   - Build completed
   - Pull request created
   - Work item created

---

## 10. Best Practices

### 10.1. Pipeline Best Practices

- **Use YAML pipelines** for version control and flexibility
- **Implement multi-stage pipelines** for separation of build and deployment
- **Use templates** for reusable pipeline components
- **Implement caching** to speed up builds
- **Use variables** for configuration management
- **Implement secrets management** with Azure Key Vault
- **Use pipeline artifacts** for sharing data between stages

### 10.2. Repository Best Practices

- **Implement branch policies** for main branches
- **Use pull request templates** for consistency
- **Implement code reviews** for all changes
- **Use git-flow** or trunk-based development
- **Implement branch naming conventions**
- **Use .gitignore** to exclude unnecessary files

### 10.3. Security Best Practices

- **Use Personal Access Tokens (PATs)** with minimal permissions
- **Rotate PATs regularly**
- **Implement branch protection rules**
- **Use Azure Key Vault** for secrets management
- **Implement security scanning in pipelines**
- **Use managed identities** for Azure resources
- **Enable audit logs** for compliance

### 10.4. Agent Best Practices

- **Use self-hosted agents** for specific requirements
- **Implement agent pools** for different environments
- **Use Docker agents** for isolation
- **Implement agent scaling** for high-demand scenarios
- **Keep agents updated** with latest patches
- **Monitor agent health** and performance

---

## 11. Troubleshooting

### 11.1. Common Pipeline Issues

**Issue: Pipeline Fails with Authentication Error**
```yaml
# Solution: Check service connection and credentials
# Verify PAT token has required permissions
# Update service connection with correct credentials
```

**Issue: Build Timeout**
```yaml
# Solution: Increase timeout in pipeline
timeoutInMinutes: 120

# Or use timeout at job level
jobs:
- job: Build
  timeoutInMinutes: 60
```

**Issue: Agent Unavailable**
```bash
# Solution: Check agent status
# Restart agent service
sudo ./svc.sh restart

# Reconfigure agent
./config.sh remove
./config.sh
```

### 11.2. Common Repository Issues

**Issue: Git Push Fails with Authentication Error**
```bash
# Solution: Update git credentials
git config --global credential.helper store
git push
# Enter credentials when prompted

# Or use credential manager
git config --global credential.helper manager-core
```

**Issue: Merge Conflicts**
```bash
# Solution: Resolve conflicts
git pull origin main
# Resolve conflicts in files
git add .
git commit -m "Resolve conflicts"
git push origin feature-branch
```

### 11.3. Common Agent Issues

**Issue: Agent Cannot Connect**
```bash
# Solution: Check network connectivity
ping dev.azure.com

# Check agent configuration
cat .agent

# Reconfigure agent
./config.sh remove
./config.sh
```

**Issue: Agent Out of Disk Space**
```bash
# Solution: Clean agent work directory
rm -rf _work/*

# Configure work directory size limit
# In agent configuration, set work folder size limit
```

---

## 12. Quick Reference

### 12.1. Common Commands

```bash
# Clone Azure Repos repository
git clone https://dev.azure.com/{organization}/{project}/_git/{repository}

# Create Personal Access Token
# Navigate to: User Settings > Personal Access Tokens

# Configure Git credential helper
git config --global credential.helper manager-core

# View pipeline runs
# Navigate to: Pipelines > Pipelines > Select pipeline > Runs

# View agent status
# Navigate to: Project Settings > Agent Pools > Select pool > Agents
```

### 12.2. Useful Pipeline Tasks

```yaml
# .NET Build
- task: DotNetCoreCLI@2
  inputs:
    command: 'build'
    projects: '**/*.csproj'

# Node.js Build
- task: Npm@1
  inputs:
    command: 'install'
    workingDir: '$(Build.SourcesDirectory)'

# Docker Build
- task: Docker@2
  inputs:
    repository: 'myimage'
    command: 'build'
    Dockerfile: '**/Dockerfile'

# Kubernetes Deploy
- task: KubernetesManifest@0
  inputs:
    action: 'deploy'
    manifests: '**/deployment.yaml'
```

---

## References

- [Azure DevOps Documentation](https://docs.microsoft.com/azure/devops/)
- [Azure DevOps REST API](https://docs.microsoft.com/rest/api/azure/devops/)
- [Azure DevOps CLI](https://docs.microsoft.com/azure/devops/cli/)
- [Azure DevOps Blog](https://devblogs.microsoft.com/devops/)
- [Azure DevOps YouTube Channel](https://www.youtube.com/c/AzureDevOps)

---

## By [Harshhaa Reddy](https://www.github.com/NotHarshhaa)
