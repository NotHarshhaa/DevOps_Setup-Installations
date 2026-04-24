# Azure DevOps Security Configuration Guide

![Azure DevOps Security](https://imgur.com/D1OFqlv.png)

Comprehensive guide for securing Azure DevOps organizations, projects, pipelines, and resources.

## Table of Contents
1. [Introduction](#1-introduction)
2. [Identity and Access Management](#2-identity-and-access-management)
3. [Project Security](#3-project-security)
4. [Pipeline Security](#4-pipeline-security)
5. [Repository Security](#5-repository-security)
6. [Secrets Management](#6-secrets-management)
7. [Service Connections](#7-service-connections)
8. [Agent Security](#8-agent-security)
9. [Compliance and Auditing](#9-compliance-and-auditing)
10. [Security Best Practices](#10-security-best-practices)

---

## 1. Introduction

### 1.1. Security Overview

Azure DevOps provides multiple layers of security to protect your code, pipelines, and infrastructure:
- **Identity and Access Management**: Control who can access resources
- **Project Security**: Configure permissions at project and team level
- **Pipeline Security**: Secure CI/CD processes and artifacts
- **Repository Security**: Protect source code with branch policies
- **Secrets Management**: Safely store and use sensitive data
- **Service Connections**: Securely connect to external services
- **Agent Security**: Protect build and deployment agents

---

## 2. Identity and Access Management

### 2.1. Personal Access Tokens (PATs)

#### Create PAT with Minimal Permissions

1. Navigate to **User Settings** > **Personal access tokens**
2. Click "New token"
3. Configure token settings:
   - **Name**: Descriptive name (e.g., "CI/CD Pipeline Token")
   - **Organization**: Select organization
   - **Expiration**: Set appropriate expiration (recommended: 90 days or less)
   - **Scopes**: Select only required permissions

#### Recommended PAT Scopes

| Purpose | Scopes |
|---------|--------|
| Build/Release Pipelines | Build (Read & Execute), Release (Read & Execute) |
| Code Access | Code (Read), Code (Write) |
| Work Items | Work Items (Read), Work Items (Write) |
| Artifacts | Packaging (Read), Packaging (Write) |
| Agent Pools | Agent Pools (Read & Manage) |

#### Rotate PATs Regularly

```bash
# Script to check PAT expiration (PowerShell)
$patExpiryDate = Get-Date "2024-01-01" # Your PAT expiry date
$daysUntilExpiry = ($patExpiryDate - (Get-Date)).Days

if ($daysUntilExpiry -lt 30) {
    Write-Host "Warning: PAT expires in $daysUntilExpiry days"
    # Send notification or create work item
}
```

### 2.2. Azure Active Directory Integration

#### Configure AAD Integration

1. Go to **Organization Settings** > **Azure Active Directory**
2. Click "Connect directory"
3. Select your Azure AD tenant
4. Configure policies:
   - **Require MFA**: Enable multi-factor authentication
   - **Conditional Access**: Set up access policies
   - **Guest Access**: Configure external user policies

#### Enforce MFA for Users

```json
{
  "displayName": "Require MFA for Azure DevOps",
  "state": "enabled",
  "conditions": {
    "applications": {
      "includeApplications": ["0000000f-0000-0000-c000-000000000000"]
    },
    "users": {
      "includeUsers": ["All"]
    },
    "clientAppTypes": ["all"],
    "locations": {
      "includeLocations": ["All"]
    }
  },
  "grantControls": {
    "operator": "OR",
    "builtInControls": ["mfa"]
  }
}
```

### 2.3. Group-Based Access Control

#### Create Security Groups

1. Navigate to **Organization Settings** > **Users** > **Groups**
2. Create groups based on roles:
   - **DevOps Administrators**: Full administrative access
   - **Developers**: Code and pipeline access
   - **Release Managers**: Deployment permissions
   - **Readers**: Read-only access

#### Assign Permissions to Groups

```powershell
# Use Azure DevOps REST API to manage group permissions
$orgUrl = "https://dev.azure.com/{organization}"
$project = "{project}"
$groupDescriptor = "{group-descriptor}"

# Get project ID
$projectUrl = "$orgUrl/_apis/projects/$project?api-version=6.0"
$project = Invoke-RestMethod -Uri $projectUrl -Method Get

# Set permissions
$permissionUrl = "$orgUrl/_apis/securitynamespaces/5a27515b-ccd7-42c9-84f1-54c998f03866/aces"
$body = @{
    token = "project/$($project.id)"
    merge = $true
    accessControlEntries = @(
        @{
            descriptor = $groupDescriptor
            allow = @(
                1,  # View project-level information
                2,  # Edit project-level information
                4,  # Delete team project
                8,  # Rename team project
                16, # Manage project properties
                32, # Manage project memberships
                64  # Delete test results
            )
            deny = @()
        }
    )
} | ConvertTo-Json -Depth 10

Invoke-RestMethod -Uri $permissionUrl -Method Put -Body $body -ContentType "application/json"
```

---

## 3. Project Security

### 3.1. Project-Level Permissions

#### Configure Project Permissions

1. Go to **Project Settings** > **Permissions**
2. Review and configure permissions for each group:
   - **Project Administrators**: Full control (limit to 2-3 users)
   - **Project Contributors**: Code and work item access
   - **Project Readers**: Read-only access
   - **Project Valid Users**: All users with project access

#### Recommended Permission Settings

| Permission | Admins | Contributors | Readers |
|------------|--------|--------------|---------|
| View project-level information | ✓ | ✓ | ✓ |
| Edit project-level information | ✓ | ✗ | ✗ |
| Delete team project | ✓ | ✗ | ✗ |
| Rename team project | ✓ | ✗ | ✗ |
| Manage project properties | ✓ | ✗ | ✗ |
| Manage project memberships | ✓ | ✗ | ✗ |
| Delete test results | ✓ | ✓ | ✗ |

### 3.2. Team-Level Permissions

#### Configure Team Settings

1. Navigate to **Project Settings** > **Teams**
2. Select team > **Team Settings**
3. Configure:
   - **Team administrators**: Limit team admin access
   - **Area paths**: Restrict work item visibility
   - **Iteration paths**: Control sprint access

#### Area Path Security

```yaml
# Example area path structure
Project/
├── Frontend/
│   ├── Components/
│   └── Services/
├── Backend/
│   ├── API/
│   └── Database/
└── Infrastructure/
    ├── DevOps/
    └── Security/

# Assign different teams to different area paths
# Frontend Team: Project/Frontend/*
# Backend Team: Project/Backend/*
# DevOps Team: Project/Infrastructure/*
```

---

## 4. Pipeline Security

### 4.1. Pipeline Permissions

#### Configure Pipeline Access

1. Go to **Pipelines** > **Pipelines**
2. Select pipeline > **Settings**
3. Configure security:
   - **Who can run**: Limit to specific users or groups
   - **Protected branches**: Require approval for specific branches
   - **YAML validation**: Require approval for YAML changes

#### Pipeline Security Settings

```yaml
# azure-pipelines.yml
# Add security checks to pipeline

# Require approval for production deployments
resources:
  repositories:
  - repository: self
    type: git
    name: $(Build.Repository.Name)

# Set pipeline permissions
trigger:
  branches:
    include:
    - main
  paths:
    exclude:
    - docs/*

# Require approval for deployment
stages:
- stage: Deploy
  displayName: 'Deploy to Production'
  dependsOn: Build
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
              azureSubscription: '$(azureSubscription)'
              appName: '$(appName)'
              package: '$(Pipeline.Workspace)/drop/*.zip'
```

### 4.2. Pipeline Approvals

#### Configure Manual Approvals

1. Navigate to **Pipelines** > **Environments**
2. Create or select environment
3. Go to **Approvals and checks**
4. Add **Approvals**:
   - **Approvers**: Select required approvers
   - **Timeout**: Set approval timeout (recommended: 7 days)
   - **Allow approvers to approve their own runs**: Disable

#### Configure Branch Protection

```yaml
# Add branch policy to main branch
# Navigate to: Repos > Branches > main > Branch policies

# Required reviewers: 2
# Check for linked work items: Enable
# Check for comment resolution: Enable
# Limit merge types: Squash merge only
# Build validation: Require successful build
```

### 4.3. Pipeline Artifacts Security

#### Configure Artifact Retention

```yaml
# azure-pipelines.yml
# Configure artifact retention policies

- task: PublishBuildArtifacts@1
  displayName: 'Publish Artifacts'
  inputs:
    PathtoPublish: '$(Build.ArtifactStagingDirectory)'
    ArtifactName: 'drop'
    publishLocation: 'Container'

# Or use retention policies in pipeline settings
# Navigate to: Pipelines > Pipeline > Settings > Retention
```

#### Secure Artifact Downloads

```yaml
# Require authentication for artifact downloads
# Navigate to: Project Settings > Settings

# Enable "Require authentication for artifact downloads"
# This prevents anonymous access to build artifacts
```

---

## 5. Repository Security

### 5.1. Branch Policies

#### Configure Branch Protection

1. Navigate to **Repos** > **Branches**
2. Select branch (e.g., main)
3. Click "Branch policies"
4. Configure policies:

```yaml
# Recommended branch policies for main branch

# 1. Require a minimum number of reviewers
# Minimum number of reviewers: 2
# When new changes are pushed: Require one approval
# Request from: Reviewers who added files

# 2. Check for linked work items
# Require: At least one linked work item

# 3. Check for comment resolution
# Require: All conversations resolved

# 4. Limit merge types
# Allow: Squash merge only
# Block: Merge commit, Rebase

# 5. Build validation
# Require: Successful build from specific pipeline

# 6. Status checks
# Require: External status checks (e.g., SonarQube, security scans)
```

### 5.2. Pull Request Policies

#### Configure PR Templates

Create `.azuredevops/pull_request_template.md`:

```markdown
## Description
<!-- Describe your changes in detail -->

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update
- [ ] Performance improvement
- [ ] Code refactoring

## Testing
<!-- Describe how you tested your changes -->
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing completed
- [ ] Test environment: [Dev/Staging/Prod]

## Checklist
- [ ] Code follows the style guidelines
- [ ] Self-review completed
- [ ] Comments added for complex code
- [ ] Documentation updated
- [ ] No new warnings generated
- [ ] All tests passing

## Security
- [ ] No sensitive data committed
- [ ] Dependencies checked for vulnerabilities
- [ ] Security review completed (if applicable)

## Breaking Changes
<!-- List any breaking changes -->
```

#### Configure PR Workflows

```yaml
# Require work item linking
# Navigate to: Repos > Branches > Policies

# Enable "Require work item linking"
# Minimum number of work items: 1

# Configure required reviewers
# Add specific users as required reviewers for certain file paths
# Example: Require security team for changes to authentication code
```

### 5.3. Code Review Policies

#### Configure Code Review Requirements

```yaml
# Require code review for all changes
# Minimum reviewers: 2
# Allow completion with comments: No

# Require review from specific teams
# Example: Require database team for schema changes
# Example: Require security team for authentication changes
```

#### Configure Protected Branches

```bash
# Use git branch protection rules
# Navigate to: Repos > Branches

# Protect main branch:
# - No direct pushes (require PR)
# - Require status checks to pass
# - Require branches to be up to date
# - Require signed commits
```

---

## 6. Secrets Management

### 6.1. Azure Key Vault Integration

#### Create Azure Key Vault

```bash
# Create Azure Key Vault
az keyvault create \
  --name my-devops-kv \
  --resource-group my-resource-group \
  --location eastus \
  --enable-soft-delete true \
  --enable-purge-protection true

# Add secrets
az keyvault secret set \
  --vault-name my-devops-kv \
  --name database-connection-string \
  --value "Server=myserver;Database=mydb;User Id=myuser;Password=mypassword;"

az keyvault secret set \
  --vault-name my-devops-kv \
  --name api-key \
  --value "my-api-key-value"
```

#### Configure Key Vault in Azure DevOps

1. Go to **Project Settings** > **Service connections**
2. Click "New service connection"
3. Select "Azure Resource Manager"
4. Choose authentication method
5. Grant Azure DevOps access to Key Vault:

```bash
# Get Azure DevOps service principal object ID
az ad sp show --id <service-principal-id> --query objectId -o tsv

# Grant access to Key Vault
az keyvault set-policy \
  --name my-devops-kv \
  --object-id <service-principal-object-id> \
  --secret-permissions get list
```

#### Use Key Vault Secrets in Pipeline

```yaml
# azure-pipelines.yml
variables:
  # Reference Key Vault secrets
  - group: MyKeyVaultVariables

steps:
- script: |
    echo "Database connection: $(database-connection-string)"
    echo "API key: $(api-key)"
  displayName: 'Use secrets'
```

### 6.2. Pipeline Variables and Secrets

#### Create Secret Variables

1. Go to **Pipelines** > **Library**
2. Click "Variable group"
3. Add variables:
   - **Name**: Variable name
   - **Value**: Secret value
   - **Lock**: Click the lock icon to mark as secret

#### Use Secret Variables in Pipeline

```yaml
# azure-pipelines.yml
variables:
  - group: MySecretVariables
  - name: my-secret
    value: $[variables.MY_SECRET]

steps:
- script: |
    echo "Using secret variable"
    # Secret variables are automatically masked in logs
  displayName: 'Use secret'
  env:
    MY_SECRET: $(my-secret)
```

#### Secure File Variables

```yaml
# Upload secure file
# Navigate to: Pipelines > Library > Secure files

# Use secure file in pipeline
steps:
- task: DownloadSecureFile@1
  displayName: 'Download secure file'
  name: mySecureFile
  inputs:
    secureFile: 'my-certificate.pfx'

- script: |
    # Use the secure file
    ls -la $(mySecureFile.secureFilePath)
  displayName: 'Use secure file'
```

### 6.3. Azure DevOps Secrets Scanner

#### Enable Secret Scanning

1. Navigate to **Project Settings** > **General**
2. Enable "Secret scanning"
3. Configure scan scope:
   - Scan all repositories
   - Scan pull requests
   - Scan branches

#### Configure Secret Scanning Rules

```yaml
# Create .azuredevops/secret-scanning.yml (if custom rules needed)
# Note: Azure DevOps uses built-in secret detection

# Common patterns detected:
# - API keys
# - Passwords
# - Connection strings
# - SSH keys
# - Certificates
```

---

## 7. Service Connections

### 7.1. Azure Service Connections

#### Create Service Principal (Manual)

```bash
# Create service principal
az ad sp create-for-rbac \
  --name "MyDevOpsServicePrincipal" \
  --role "Contributor" \
  --scopes /subscriptions/{subscription-id} \
  --json-auth

# Save the output JSON for service connection creation
```

#### Create Service Connection

1. Go to **Project Settings** > **Service connections**
2. Click "New service connection"
3. Select "Azure Resource Manager"
4. Choose authentication method:
   - **Service principal (automatic)**: Recommended for most scenarios
   - **Service principal (manual)**: For advanced scenarios
   - **Managed identity**: For Azure DevOps Server

#### Grant Least Privilege Access

```bash
# Create custom role for service connection
az role definition create \
  --role-definition '{
    "Name": "DevOps Deployment Role",
    "Description": "Custom role for Azure DevOps deployments",
    "Actions": [
      "Microsoft.Resources/deployments/*",
      "Microsoft.Web/sites/*",
      "Microsoft.ContainerRegistry/registries/*",
      "Microsoft.ContainerService/managedClusters/*"
    ],
    "NotActions": [
      "Microsoft.Web/sites/config/*",
      "Microsoft.Web/sites/delete"
    ],
    "AssignableScopes": [
      "/subscriptions/{subscription-id}/resourceGroups/{resource-group}"
    ]
  }'

# Assign custom role to service principal
az role assignment create \
  --assignee {service-principal-id} \
  --role "DevOps Deployment Role" \
  --scope /subscriptions/{subscription-id}/resourceGroups/{resource-group}
```

### 7.2. GitHub Service Connections

#### Configure GitHub OAuth

1. Go to **Project Settings** > **Service connections**
2. Click "New service connection"
3. Select "GitHub"
4. Authorize Azure DevOps to access your GitHub account
5. Configure permissions:
   - Repository access: Select specific repositories
   - Permissions: Read-only for code, Read/Write for deployments

#### Use GitHub PAT

```bash
# Create GitHub PAT
# Navigate to: GitHub Settings > Developer settings > Personal access tokens

# Required scopes:
# - repo: Full control of private repositories
# - workflow: Update GitHub Action workflows
# - read:org: Read org and team membership
# - user:user:email: Read user email address
```

### 7.3. Other Service Connections

#### Docker Registry Connection

```yaml
# Create Docker registry service connection
# Navigate to: Project Settings > Service connections

# For Azure Container Registry:
# 1. Select "Docker Registry"
# 2. Choose "Azure Container Registry"
# 3. Select your ACR
# 4. Grant Azure DevOps access

# For Docker Hub:
# 1. Select "Docker Registry"
# 2. Choose "Docker Hub"
# 3. Enter Docker Hub credentials
# 4. Test connection
```

#### Kubernetes Service Connection

```yaml
# Create Kubernetes service connection
# Navigate to: Project Settings > Service connections

# Options:
# 1. Subscription: Use Azure subscription credentials
# 2. Service Account: Use Kubernetes service account (recommended)
# 3. Kubeconfig: Upload kubeconfig file (not recommended for production)

# For AKS with Azure AD integration:
# 1. Select "Azure Subscription"
# 2. Choose your AKS cluster
# 3. Configure Azure AD authentication
```

---

## 8. Agent Security

### 8.1. Microsoft-Hosted Agents

#### Security Considerations

- **Isolation**: Each pipeline run gets a fresh agent
- **Ephemeral**: Agents are deleted after pipeline completion
- **No persistence**: No data persists between runs
- **Managed by Microsoft**: Security patches applied automatically

#### Use Microsoft-Hosted Agents When

- Building open-source projects
- Running untrusted code
- Need for multiple OS/platforms
- No special software requirements

### 8.2. Self-Hosted Agents

#### Secure Self-Hosted Agent Configuration

```powershell
# Run agent as non-administrator user
# Create service account
New-LocalUser -Name "AzDevOpsAgent" -Password (ConvertTo-SecureString "SecurePassword123!" -AsPlainText -Force) -Description "Azure DevOps Agent Service Account"

# Add to necessary groups
Add-LocalGroupMember -Group "Docker Users" -Member "AzDevOpsAgent"
Add-LocalGroupMember -Group "Performance Monitor Users" -Member "AzDevOpsAgent"

# Configure agent to run as service account
.\config.cmd
# When prompted, use the service account credentials
```

#### Network Isolation for Agents

```powershell
# Configure firewall rules for agent
# Allow outbound to Azure DevOps
New-NetFirewallRule -DisplayName "Azure DevOps Agent" -Direction Outbound -Action Allow -Protocol TCP -RemoteAddress "*.visualstudio.com","dev.azure.com"

# Block unnecessary inbound connections
New-NetFirewallRule -DisplayName "Block Inbound" -Direction Inbound -Action Block

# Allow only necessary ports
New-NetFirewallRule -DisplayName "Allow SSH" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 22
```

#### Agent Pool Security

```yaml
# Configure agent pool security
# Navigate to: Project Settings > Agent pools

# Create separate pools for:
# - Build agents: For compilation and testing
# - Deployment agents: For deployments to production
# - Security-scoped agents: For security-sensitive operations

# Set pool permissions:
# - Who can queue: Restrict to specific users/groups
# - Service account: Use least privilege service account
```

### 8.3. Agent Capabilities and Demands

#### Configure Agent Capabilities

```yaml
# Use demands to control which agents can run pipelines
# azure-pipelines.yml

pool:
  demands:
  - agent.os -equals Linux
  - docker
  - java

steps:
- script: echo "Running on Linux with Docker and Java"
```

#### Limit Agent Access

```yaml
# Use agent pools with restricted access
# Navigate to: Project Settings > Agent pools > Pool > Security

# Configure:
# - User roles: Who can use the pool
# - Service roles: Who can manage the pool
# - Project scope: Which projects can use the pool
```

---

## 9. Compliance and Auditing

### 9.1. Audit Logs

#### Enable Audit Logging

1. Navigate to **Organization Settings** > **Audit**
2. Audit logs are automatically enabled
3. Configure log retention:
   - Default: 90 days
   - Extended: Up to 480 days (with premium license)

#### Query Audit Logs

```bash
# Use Azure DevOps REST API to query audit logs
$orgUrl = "https://auditservice.dev.azure.com/{organization}"
$apiVersion = "6.0-preview.1"

# Query for pipeline runs
$auditUrl = "$orgUrl/_apis/audit/auditlog?api-version=$apiVersion&startTime=2024-01-01T00:00:00Z&endTime=2024-01-31T23:59:59Z&pipelineId={pipeline-id}"

$response = Invoke-RestMethod -Uri $auditUrl -Method Get -Headers @{
    Authorization = "Bearer $PAT"
}

$response.auditLogEntries | ForEach-Object {
    Write-Host "$($_.operationName) - $($_.actor) - $($_.timestamp)"
}
```

### 9.2. Policy Enforcement

#### Configure Organization Policies

1. Navigate to **Organization Settings** > **Policies**
2. Configure policies:
   - **Anonymous access**: Disable for security
   - **Public projects**: Require approval
   - **External user access**: Restrict guest access
   - **IP filtering**: Restrict access by IP range

#### Configure Project Policies

1. Navigate to **Project Settings** > **Settings**
2. Configure:
   - **Pipeline permissions**: Require approval for new pipelines
   - **Artifact permissions**: Restrict artifact downloads
   - **Service connection permissions**: Require approval for new connections

### 9.3. Compliance Scanning

#### Integrate Security Scanners

```yaml
# Add security scanning to pipeline
# azure-pipelines.yml

stages:
- stage: SecurityScan
  displayName: 'Security Scanning'
  jobs:
  - job: SAST
    displayName: 'Static Application Security Testing'
    steps:
    - task: SonarQubePrepare@5
      displayName: 'Prepare SonarQube Analysis'
      inputs:
        SonarQube: 'SonarQubeConnection'
        scannerMode: 'MSBuild'
        projectKey: 'my-project-key'

    - task: SonarQubeAnalyze@5
      displayName: 'Run SonarQube Analysis'

    - task: SonarQubePublish@5
      displayName: 'Publish SonarQube Results'
      inputs:
        pollingTimeoutSec: '300'

  - job: DependencyCheck
    displayName: 'Dependency Vulnerability Scan'
    steps:
    - task: Npm@1
      displayName: 'Run npm audit'
      inputs:
        command: 'custom'
        customCommand: 'audit --audit-level=moderate'

  - job: ContainerScan
    displayName: 'Container Image Scan'
    steps:
    - task: Docker@2
      displayName: 'Pull Image'
      inputs:
        command: pull
        arguments: myregistry.azurecr.io/myapp:latest

    - script: |
        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
          aquasec/trivy image --severity HIGH,CRITICAL \
          --exit-code 1 myregistry.azurecr.io/myapp:latest
      displayName: 'Scan Container Image'
```

---

## 10. Security Best Practices

### 10.1. General Best Practices

- **Use least privilege access**: Grant minimum required permissions
- **Enable MFA**: Require multi-factor authentication for all users
- **Regularly rotate secrets**: Update PATs, service principal credentials
- **Monitor audit logs**: Regularly review audit logs for suspicious activity
- **Implement branch policies**: Protect main branches with policies
- **Use secret scanning**: Enable secret scanning in repositories
- **Secure service connections**: Use service principals with limited scope
- **Isolate environments**: Use separate agents for dev/staging/prod

### 10.2. Pipeline Security Checklist

- [ ] All secrets stored in Azure Key Vault or secret variables
- [ ] Pipeline approvals configured for production deployments
- [ ] Branch policies enabled for main branches
- [ ] Pull request templates configured
- [ ] Code review required for all changes
- [ ] Security scanning integrated in CI/CD
- [ ] Artifact retention policies configured
- [ ] Agent pool permissions restricted
- [ ] Service connections use least privilege
- [ ] Pipeline YAML files reviewed and approved

### 10.3. Incident Response

#### Security Incident Response Plan

1. **Detection**: Monitor audit logs and alerts
2. **Containment**: Immediately revoke compromised credentials
3. **Investigation**: Analyze audit logs to determine scope
4. **Remediation**: Rotate all potentially compromised credentials
5. **Recovery**: Restore from clean backups if necessary
6. **Post-Incident**: Review and improve security measures

#### Emergency Access Revocation

```bash
# Revoke all PATs for a user
# Navigate to: Organization Settings > Users > Select user > Revoke all tokens

# Disable user account
az ad user update --id {user-id} --account-enabled false

# Remove user from all groups
az ad user remove-from-group --group {group-id} --member-id {user-id}
```

---

## References

- [Azure DevOps Security Documentation](https://docs.microsoft.com/azure/devops/security/)
- [Azure Key Vault Documentation](https://docs.microsoft.com/azure/key-vault/)
- [Azure DevOps Audit Logs](https://docs.microsoft.com/azure/devops/organizations/audit/overview)
- [Azure Security Best Practices](https://docs.microsoft.com/azure/security/fundamentals/best-practices)

---

## By [Harshhaa Reddy](https://www.github.com/NotHarshhaa)
