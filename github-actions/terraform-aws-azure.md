# Terraform with GitHub Actions for AWS and Azure

![Terraform](https://www.terraform.io/img/og-image.png)

Comprehensive guide for managing infrastructure deployments using Terraform for AWS and Azure with GitHub Actions.

## Table of Contents
1. [Introduction](#1-introduction)
2. [AWS Terraform Workflows](#2-aws-terraform-workflows)
3. [Azure Terraform Workflows](#3-azure-terraform-workflows)
4. [Multi-Cloud Workflows](#4-multi-cloud-workflows)
5. [State Management](#5-state-management)
6. [Security Best Practices](#6-security-best-practices)
7. [Advanced Features](#7-advanced-features)

---

## 1. Introduction

### 1.1. Terraform with GitHub Actions

Using Terraform with GitHub Actions enables:
- Automated infrastructure provisioning
- Infrastructure as Code (IaC) best practices
- Multi-cloud deployment strategies
- Automated testing and validation
- GitOps workflows

### 1.2. Prerequisites

- Terraform configuration files (`*.tf`)
- GitHub repository
- AWS or Azure account
- GitHub Secrets for credentials

---

## 2. AWS Terraform Workflows

### 2.1. Basic AWS Workflow

```yaml
name: Terraform AWS

on:
  push:
    branches: [ main ]
    paths:
    - 'terraform/aws/**'
  pull_request:
    branches: [ main ]
    paths:
    - 'terraform/aws/**'
  workflow_dispatch:

jobs:
  terraform:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ./terraform/aws

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
        aws-region: us-east-1

    - name: Terraform Init
      run: terraform init

    - name: Terraform Format
      run: terraform fmt -check

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
```

### 2.2. AWS with Remote State

```yaml
name: Terraform AWS with S3 Backend

on:
  push:
    branches: [ main ]
    paths:
    - 'terraform/aws/**'

jobs:
  terraform:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ./terraform/aws

    steps:
    - uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: '1.6.0'

    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1

    - name: Terraform Init with S3 Backend
      run: |
        terraform init \
          -backend-config="bucket=my-terraform-state" \
          -backend-config="key=terraform.tfstate" \
          -backend-config="region=us-east-1"

    - name: Terraform Plan
      run: terraform plan -out=tfplan

    - name: Terraform Apply
      if: github.ref == 'refs/heads/main'
      run: terraform apply -auto-approve tfplan
```

### 2.3. AWS with Workspaces

```yaml
name: Terraform AWS Workspaces

on:
  push:
    branches: [ main ]
  workflow_dispatch:
    inputs:
      workspace:
        description: 'Terraform workspace'
        required: true
        default: 'dev'
        type: choice
        options:
          - dev
          - staging
          - prod

jobs:
  terraform:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ./terraform/aws

    steps:
    - uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3

    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1

    - name: Select Workspace
      run: terraform workspace select ${{ github.event.inputs.workspace || 'dev' }}

    - name: Terraform Init
      run: terraform init

    - name: Terraform Plan
      run: terraform plan -out=tfplan

    - name: Terraform Apply
      if: github.ref == 'refs/heads/main'
      run: terraform apply -auto-approve tfplan
```

---

## 3. Azure Terraform Workflows

### 3.1. Basic Azure Workflow

```yaml
name: Terraform Azure

on:
  push:
    branches: [ main ]
    paths:
    - 'terraform/azure/**'
  pull_request:
    branches: [ main ]
    paths:
    - 'terraform/azure/**'
  workflow_dispatch:

jobs:
  terraform:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ./terraform/azure

    steps:
    - uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: '1.6.0'
        terraform_wrapper: false

    - name: Login to Azure
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Terraform Init
      run: terraform init

    - name: Terraform Format
      run: terraform fmt -check

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

    - name: Logout from Azure
      if: always()
      run: az logout
```

### 3.2. Azure with Remote State

```yaml
name: Terraform Azure with Storage Backend

on:
  push:
    branches: [ main ]
    paths:
    - 'terraform/azure/**'

jobs:
  terraform:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ./terraform/azure

    steps:
    - uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: '1.6.0'

    - name: Login to Azure
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Terraform Init with Azure Storage
      run: |
        terraform init \
          -backend-config="resource_group_name=${{ secrets.AZURE_RESOURCE_GROUP }}" \
          -backend-config="storage_account_name=${{ secrets.AZURE_STORAGE_ACCOUNT }}" \
          -backend-config="container_name=tfstate" \
          -backend-config="key=terraform.tfstate"

    - name: Terraform Plan
      run: terraform plan -out=tfplan

    - name: Terraform Apply
      if: github.ref == 'refs/heads/main'
      run: terraform apply -auto-approve tfplan
```

### 3.3. Azure with Service Principal

```yaml
name: Terraform Azure Service Principal

on:
  push:
    branches: [ main ]
    paths:
    - 'terraform/azure/**'

jobs:
  terraform:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ./terraform/azure

    steps:
    - uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3

    - name: Azure Login with Service Principal
      run: |
        az login --service-principal \
          --username ${{ secrets.AZURE_CLIENT_ID }} \
          --password ${{ secrets.AZURE_CLIENT_SECRET }} \
          --tenant ${{ secrets.AZURE_TENANT_ID }}

    - name: Terraform Init
      run: terraform init

    - name: Terraform Plan
      run: terraform plan -out=tfplan

    - name: Terraform Apply
      if: github.ref == 'refs/heads/main'
      run: terraform apply -auto-approve tfplan
```

---

## 4. Multi-Cloud Workflows

### 4.1. AWS and Azure Parallel Deployment

```yaml
name: Multi-Cloud Terraform

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  terraform-aws:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ./terraform/aws

    steps:
    - uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3

    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1

    - name: Terraform Init
      run: terraform init

    - name: Terraform Plan
      run: terraform plan -out=tfplan

    - name: Terraform Apply
      if: github.ref == 'refs/heads/main'
      run: terraform apply -auto-approve tfplan

  terraform-azure:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ./terraform/azure

    steps:
    - uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3

    - name: Login to Azure
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Terraform Init
      run: terraform init

    - name: Terraform Plan
      run: terraform plan -out=tfplan

    - name: Terraform Apply
      if: github.ref == 'refs/heads/main'
      run: terraform apply -auto-approve tfplan
```

### 4.2. Multi-Environment Deployment

```yaml
name: Multi-Environment Terraform

on:
  push:
    branches: [ main ]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        default: 'dev'
        type: choice
        options:
          - dev
          - staging
          - prod

jobs:
  terraform:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        cloud: [aws, azure]
        environment: [dev, staging, prod]
        exclude:
          - cloud: azure
            environment: prod

    defaults:
      run:
        working-directory: ./terraform/${{ matrix.cloud }}

    steps:
    - uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3

    - name: Configure ${{ matrix.cloud }} Credentials
      if: matrix.cloud == 'aws'
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1

    - name: Login to Azure
      if: matrix.cloud == 'azure'
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Terraform Init
      run: terraform init

    - name: Terraform Plan
      run: terraform plan -out=tfplan

    - name: Terraform Apply
      if: github.event.inputs.environment == matrix.environment
      run: terraform apply -auto-approve tfplan
```

---

## 5. State Management

### 5.1. AWS S3 State Backend

```hcl
# terraform/aws/backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### 5.2. Azure Storage State Backend

```hcl
# terraform/azure/backend.tf
terraform {
  backend "azurerm" {
    resource_group_name  = "my-resource-group"
    storage_account_name = "mystorageaccount"
    container_name       = "tfstate"
    key                  = "terraform.tfstate"
  }
}
```

### 5.3. State Locking

```yaml
- name: Terraform Apply with State Lock
  run: terraform apply -auto-approve -lock-timeout=5m tfplan
```

---

## 6. Security Best Practices

### 6.1. Use GitHub Secrets

Store sensitive information in GitHub Secrets:
- AWS credentials
- Azure service principals
- API keys
- Database passwords

### 6.2. Enable Encryption

```hcl
# AWS S3 encryption
terraform {
  backend "s3" {
    encrypt = true
  }
}
```

### 6.3. Use IAM Roles

Instead of using access keys, use IAM roles with temporary credentials.

### 6.4. Implement Approval Gates

```yaml
- name: Terraform Apply
  if: github.ref == 'refs/heads/main'
  environment: production
  run: terraform apply -auto-approve tfplan
```

---

## 7. Advanced Features

### 7.1. Drift Detection

```yaml
- name: Terraform Drift Detection
  run: |
    terraform plan -detailed-exitcode
```

### 7.2. Import Existing Resources

```yaml
- name: Terraform Import
  run: |
    terraform import aws_instance.example i-1234567890abcdef0
```

### 7.3. Terraform Compliance

```yaml
- name: Run Terraform Compliance
  uses: terraform-compliance/terraform-compliance-action@master
  with:
    features: tests/features/
```

---

## References

- [Terraform Documentation](https://www.terraform.io/docs)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [AWS Provider Documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Azure Provider Documentation](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)

---

## By [Harshhaa Reddy](https://www.github.com/NotHarshhaa)
