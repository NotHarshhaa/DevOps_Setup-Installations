# AWS CLI Installation and Configuration Guide

![AWS CLI](https://d1.awsstatic.com/logos/aws-logo-lockups/poweredbyaws/Powered-by_AWS_logo_RGB_REV_SQ.6d4e9c4d0d9f8e8e9e8e9e8e9e8e9e8e9e8e9e8.png)

Comprehensive guide for installing, configuring, and optimizing AWS Command Line Interface (CLI) v2 across different operating systems.

## Table of Contents
1. [Introduction](#1-introduction)
2. [Installation](#2-installation)
3. [Configuration](#3-configuration)
4. [Advanced Configuration](#4-advanced-configuration)
5. [Common Commands](#5-common-commands)
6. [Best Practices](#6-best-practices)
7. [Troubleshooting](#7-troubleshooting)
8. [Integration with DevOps Tools](#8-integration-with-devops-tools)

---

## 1. Introduction

### 1.1. What is AWS CLI?

AWS Command Line Interface (CLI) is a unified tool to manage your AWS services. With just one tool to download and configure, you can control multiple AWS services from the command line and automate them through scripts.

### 1.2. Features

- **Unified Tool**: Control all AWS services from one tool
- **Multiple Formats**: Output in JSON, text, or table format
- **Scriptable**: Automate AWS tasks with scripts
- **Cross-Platform**: Works on Linux, macOS, and Windows
- **Customizable**: Extensible through plugins
- **Secure**: Supports MFA and role-based access

### 1.3. Prerequisites

- Internet connection
- AWS Account
- Appropriate IAM permissions
- Python 3.8+ (for some plugins)

---

## 2. Installation

### 2.1. Linux Installation

#### Method 1: Using the Bundled Installer (Recommended)

```bash
# Download the installer
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Unzip the package
unzip awscliv2.zip

# Run the installer
sudo ./aws/install

# Verify installation
aws --version
```

#### Method 2: Using Package Manager

**Ubuntu/Debian:**
```bash
# Update package list
sudo apt-get update

# Install AWS CLI
sudo apt-get install -y awscli

# Verify installation
aws --version
```

**CentOS/RHEL:**
```bash
# Enable EPEL repository
sudo yum install -y epel-release

# Install AWS CLI
sudo yum install -y awscli

# Verify installation
aws --version
```

**Fedora:**
```bash
# Install AWS CLI
sudo dnf install -y awscli

# Verify installation
aws --version
```

#### Method 3: Using pip

```bash
# Install pip if not already installed
sudo apt-get install -y python3-pip

# Install AWS CLI v2
pip3 install awscli --upgrade --user

# Add to PATH (add to ~/.bashrc or ~/.zshrc)
export PATH=$HOME/.local/bin:$PATH

# Verify installation
aws --version
```

### 2.2. macOS Installation

#### Method 1: Using the Bundled Installer (Recommended)

```bash
# Download the installer
curl "https://awscli.amazonaws.com/awscli-exe-macosx.zip" -o "awscliv2.zip"

# Unzip the package
unzip awscliv2.zip

# Run the installer
sudo ./aws/install

# Verify installation
aws --version
```

#### Method 2: Using Homebrew

```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install AWS CLI
brew install awscli

# Verify installation
aws --version
```

#### Method 3: Using pip

```bash
# Install using pip
pip3 install awscli --upgrade --user

# Add to PATH
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.zshrc
source ~/.zshrc

# Verify installation
aws --version
```

### 2.3. Windows Installation

#### Method 1: Using MSI Installer (Recommended)

1. Download the MSI installer from [AWS CLI v2](https://aws.amazon.com/cli/)
2. Run the installer
3. Follow the installation wizard
4. Verify installation by opening Command Prompt or PowerShell:
```powershell
aws --version
```

#### Method 2: Using Chocolatey

```powershell
# Install Chocolatey if not already installed
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Install AWS CLI
choco install awscli

# Verify installation
aws --version
```

#### Method 3: Using pip

```powershell
# Install Python if not already installed
# Download from https://www.python.org/downloads/

# Install AWS CLI using pip
pip install awscli --upgrade

# Verify installation
aws --version
```

### 2.4. Docker Installation

```bash
# Run AWS CLI in a Docker container
docker run --rm -it amazon/aws-cli --version

# Use AWS CLI via Docker
docker run --rm -it -v ~/.aws:/root/.aws amazon/aws-cli s3 ls

# Create an alias for convenience
alias aws='docker run --rm -it -v ~/.aws:/root/.aws -v $(pwd):/aws amazon/aws-cli'
```

---

## 3. Configuration

### 3.1. Basic Configuration

```bash
# Configure AWS CLI
aws configure

# You will be prompted for:
# AWS Access Key ID
# AWS Secret Access Key
# Default region name
# Default output format (json, text, table)
```

### 3.2. Named Profiles

```bash
# Configure a named profile
aws configure --profile production

# Configure another profile
aws configure --profile development

# List configured profiles
aws configure list-profiles

# Use a specific profile
aws s3 ls --profile production

# Set default profile
export AWS_PROFILE=production
```

### 3.3. Environment Variables

```bash
# Set AWS credentials via environment variables
export AWS_ACCESS_KEY_ID=your_access_key_id
export AWS_SECRET_ACCESS_KEY=your_secret_access_key
export AWS_DEFAULT_REGION=us-west-2
export AWS_DEFAULT_OUTPUT=json

# Set profile via environment variable
export AWS_PROFILE=production

# Temporary credentials for session
export AWS_SESSION_TOKEN=your_session_token
```

### 3.4. Credentials File

The credentials file is located at:
- Linux/macOS: `~/.aws/credentials`
- Windows: `C:\Users\USERNAME\.aws\credentials`

**Example credentials file:**
```ini
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[production]
aws_access_key_id = AKIAI44QH8DHBEXAMPLE
aws_secret_access_key = je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY
```

### 3.5. Config File

The config file is located at:
- Linux/macOS: `~/.aws/config`
- Windows: `C:\Users\USERNAME\.aws\config`

**Example config file:**
```ini
[default]
region = us-west-2
output = json

[profile production]
region = us-east-1
output = table

[profile development]
region = us-west-2
output = json
```

---

## 4. Advanced Configuration

### 4.1. IAM Role Configuration

```bash
# Configure to assume an IAM role
aws configure set role_arn arn:aws:iam::123456789012:role/MyRole --profile myrole
aws configure set source_profile default --profile myrole

# Use the role profile
aws s3 ls --profile myrole
```

### 4.2. MFA Configuration

```ini
[profile mfa-enabled]
region = us-west-2
output = json
source_profile = default
role_arn = arn:aws:iam::123456789012:role/MyRole
mfa_serial = arn:aws:iam::123456789012:mfa/MyMFA
```

### 4.3. S3 Configuration

```bash
# Configure S3 endpoint
aws configure set s3.endpoint_url https://s3.us-west-2.amazonaws.com

# Configure S3 signature version
aws configure set default.s3.signature_version s3v4

# Configure S3 max concurrent requests
aws configure set default.s3.max_concurrent_requests 20
```

### 4.4. Retry Configuration

```bash
# Set maximum retry attempts
aws configure set max_attempts 10

# Set retry mode (standard, legacy, adaptive)
aws configure set retry_mode standard
```

### 4.5. Proxy Configuration

```bash
# Set HTTP proxy
export HTTP_PROXY=http://proxy.example.com:8080

# Set HTTPS proxy
export HTTPS_PROXY=https://proxy.example.com:8080

# Set no proxy for certain hosts
export NO_PROXY=localhost,127.0.0.1,example.com

# Configure proxy in config file
aws configure set http_proxy http://proxy.example.com:8080
aws configure set https_proxy https://proxy.example.com:8080
```

---

## 5. Common Commands

### 5.1. EC2 Commands

```bash
# List instances
aws ec2 describe-instances

# List instances with specific filters
aws ec2 describe-instances \
    --filters "Name=instance-state-name,Values=running" \
    --query "Reservations[*].Instances[*].[InstanceId,InstanceType,State.Name]" \
    --output table

# Start an instance
aws ec2 start-instances --instance-ids i-1234567890abcdef0

# Stop an instance
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# Terminate an instance
aws ec2 terminate-instances --instance-ids i-1234567890abcdef0

# Create a key pair
aws ec2 create-key-pair --key-name MyKeyPair --query 'KeyMaterial' --output text > MyKeyPair.pem

# Create a security group
aws ec2 create-security-group \
    --group-name MySecurityGroup \
    --description "My security group"

# Add rules to security group
aws ec2 authorize-security-group-ingress \
    --group-id sg-903004f8 \
    --protocol tcp \
    --port 22 \
    --cidr 203.0.113.0/24
```

### 5.2. S3 Commands

```bash
# List buckets
aws s3 ls

# List objects in a bucket
aws s3 ls s3://my-bucket/

# Sync local directory to S3
aws s3 sync ./local-folder s3://my-bucket/remote-folder

# Copy file to S3
aws s3 cp myfile.txt s3://my-bucket/

# Copy file from S3
aws s3 cp s3://my-bucket/myfile.txt ./local-file.txt

# Delete file from S3
aws s3 rm s3://my-bucket/myfile.txt

# Set bucket policy
aws s3api put-bucket-policy \
    --bucket my-bucket \
    --policy file://policy.json

# Enable versioning
aws s3api put-bucket-versioning \
    --bucket my-bucket \
    --versioning-configuration Status=Enabled
```

### 5.3. IAM Commands

```bash
# List users
aws iam list-users

# Create user
aws iam create-user --user-name myuser

# Create access key
aws iam create-access-key --user-name myuser

# Attach policy to user
aws iam attach-user-policy \
    --user-name myuser \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# List attached policies
aws iam list-attached-user-policies --user-name myuser

# Create role
aws iam create-role \
    --role-name MyRole \
    --assume-role-policy-document file://trust-policy.json

# Attach policy to role
aws iam attach-role-policy \
    --role-name MyRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

### 5.4. EKS Commands

```bash
# List clusters
aws eks list-clusters

# Describe cluster
aws eks describe-cluster --name my-cluster

# Create cluster
aws eks create-cluster \
    --name my-cluster \
    --role-arn arn:aws:iam::123456789012:role/eksServiceRole \
    --resources-vpc-config subnetIds=subnet-12345,subnet-67890,securityGroupIds=sg-12345

# Delete cluster
aws eks delete-cluster --name my-cluster

# Update kubeconfig
aws eks update-kubeconfig --name my-cluster

# List node groups
aws eks list-nodegroups --cluster-name my-cluster
```

### 5.5. ECR Commands

```bash
# List repositories
aws ecr describe-repositories

# Create repository
aws ecr create-repository --repository-name my-repo

# Login to ECR
aws ecr get-login-password --region us-west-2 | \
    docker login --username AWS --password-stdin \
    123456789012.dkr.ecr.us-west-2.amazonaws.com

# Tag image
docker tag my-image:latest 123456789012.dkr.ecr.us-west-2.amazonaws.com/my-repo:latest

# Push image
docker push 123456789012.dkr.ecr.us-west-2.amazonaws.com/my-repo:latest

# Set lifecycle policy
aws ecr put-lifecycle-policy \
    --repository-name my-repo \
    --lifecycle-policy-text file://lifecycle-policy.json
```

---

## 6. Best Practices

### 6.1. Security Best Practices

```bash
# Use IAM roles instead of access keys when possible
# Configure MFA for sensitive operations
# Rotate access keys regularly
# Use least privilege principle
# Enable CloudTrail for API logging

# Example: Rotate access key
aws iam delete-access-key --access-key-id AKIAIOSFODNN7EXAMPLE --user-name myuser
aws iam create-access-key --user-name myuser
```

### 6.2. Naming Conventions

```bash
# Use consistent naming for profiles
aws configure --profile prod-admin
aws configure --profile prod-readonly
aws configure --profile dev-admin

# Use environment-specific configurations
aws configure --profile us-west-2-prod
aws configure --profile eu-central-1-prod
```

### 6.3. Output Formatting

```bash
# JSON output (default)
aws ec2 describe-instances --output json

# Table output
aws ec2 describe-instances --output table

# Text output
aws ec2 describe-instances --output text

# Query specific fields
aws ec2 describe-instances \
    --query "Reservations[*].Instances[*].[InstanceId,State.Name]" \
    --output table
```

### 6.4. Scripting Best Practices

```bash
#!/bin/bash
# Set error handling
set -e

# Set AWS profile
export AWS_PROFILE=production

# Use variables
BUCKET_NAME="my-bucket"
REGION="us-west-2"

# Create bucket
aws s3api create-bucket \
    --bucket $BUCKET_NAME \
    --region $REGION \
    --create-bucket-configuration LocationConstraint=$REGION

# Enable versioning
aws s3api put-bucket-versioning \
    --bucket $BUCKET_NAME \
    --versioning-configuration Status=Enabled

echo "Bucket created successfully"
```

### 6.5. Performance Optimization

```bash
# Increase max concurrent requests for S3
aws configure set default.s3.max_concurrent_requests 20

# Increase multipart threshold
aws configure set default.s3.multipart_threshold 64MB

# Increase multipart chunk size
aws configure set default.s3.multipart_chunksize 16MB
```

---

## 7. Troubleshooting

### 7.1. Common Issues

**Issue: "aws: command not found"**
```bash
# Check if AWS CLI is installed
which aws

# Add to PATH
export PATH=$PATH:/usr/local/bin

# Reinstall if necessary
sudo ./aws/install
```

**Issue: "Unable to locate credentials"**
```bash
# Check credentials file
cat ~/.aws/credentials

# Reconfigure
aws configure

# Check environment variables
env | grep AWS

# Set credentials manually
export AWS_ACCESS_KEY_ID=your_key
export AWS_SECRET_ACCESS_KEY=your_secret
```

**Issue: "SSL certificate problem"**
```bash
# Update CA certificates (Linux)
sudo apt-get install ca-certificates

# Disable SSL verification (not recommended)
aws configure set ssl_verify false
```

**Issue: "Connection timeout"**
```bash
# Check network connectivity
ping aws.amazon.com

# Configure proxy
export HTTP_PROXY=http://proxy.example.com:8080
export HTTPS_PROXY=https://proxy.example.com:8080

# Increase timeout
aws configure set cli_connect_timeout 120
```

### 7.2. Debug Mode

```bash
# Enable debug output
aws s3 ls --debug

# Enable verbose output
aws s3 ls --verbose

# Log to file
aws s3 ls --debug > aws-debug.log 2>&1
```

### 7.3. Version Check and Update

```bash
# Check current version
aws --version

# Update AWS CLI (Linux)
sudo ./aws/install --update

# Update AWS CLI (macOS)
brew upgrade awscli

# Update AWS CLI (Windows)
# Re-run the MSI installer

# Update via pip
pip install --upgrade awscli
```

---

## 8. Integration with DevOps Tools

### 8.1. Jenkins Integration

```groovy
pipeline {
    agent any
    environment {
        AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
    }
    stages {
        stage('Deploy to S3') {
            steps {
                sh 'aws s3 sync ./dist s3://my-bucket/ --delete'
            }
        }
    }
}
```

### 8.2. GitHub Actions Integration

```yaml
name: Deploy to AWS

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-west-2
    
    - name: Deploy to S3
      run: |
        aws s3 sync ./dist s3://my-bucket/ --delete
```

### 8.3. Terraform Integration

```hcl
# Configure AWS provider
provider "aws" {
  region = "us-west-2"
  
  # Use shared credentials file
  shared_credentials_file = "~/.aws/credentials"
  profile                 = "production"
}

# Use environment variables
provider "aws" {
  region = var.aws_region
}
```

### 8.4. Ansible Integration

```yaml
# ansible.cfg
[defaults]
host_key_checking = False
remote_user = ec2-user

# playbook.yml
---
- name: Deploy to AWS
  hosts: localhost
  connection: local
  tasks:
    - name: Create S3 bucket
      aws_s3:
        bucket: my-bucket
        region: us-west-2
        state: present
```

### 8.5. Docker Integration

```dockerfile
# Dockerfile
FROM amazon/aws-cli:latest

# Copy configuration
COPY .aws /root/.aws

# Set working directory
WORKDIR /aws

# Default command
CMD ["--version"]
```

```bash
# Build Docker image
docker build -t my-aws-cli .

# Run container
docker run --rm -it my-aws-cli s3 ls
```

---

## 9. AWS CLI Plugins

### 9.1. Installing Plugins

```bash
# Install a plugin
pip install awscli-plugin-endpoint

# List installed plugins
aws plugin list

# Configure plugin
aws configure set plugins.endpoint awscli_plugin_endpoint
```

### 9.2. Common Plugins

```bash
# Session manager plugin
aws ssm start-session --target i-1234567890abcdef0

# CloudFormation plugin
aws cloudformation deploy \
    --template-file template.yaml \
    --stack-name my-stack \
    --capabilities CAPABILITY_IAM
```

---

## 10. Quick Reference

### 10.1. Configuration Commands

```bash
aws configure                    # Configure default profile
aws configure --profile name     # Configure named profile
aws configure list               # List configuration
aws configure list-profiles      # List all profiles
aws configure get region         # Get specific setting
aws configure set region us-west-2  # Set specific setting
```

### 10.2. Common Options

```bash
--profile PROFILE_NAME           # Use specific profile
--region REGION_NAME             # Override default region
--output FORMAT                  # Override output format (json, text, table)
--query JMESPATH                 # Filter output
--debug                          # Enable debug output
--dry-run                        # Validate request without executing
```

### 10.3. Help Commands

```bash
aws help                         # General help
aws s3 help                      # Service help
aws s3 ls help                   # Command help
aws s3 ls help examples          # Examples
```

---

## References

- [AWS CLI Documentation](https://docs.aws.amazon.com/cli/)
- [AWS CLI GitHub Repository](https://github.com/aws/aws-cli)
- [AWS CLI Release Notes](https://github.com/aws/aws-cli/blob/v2/CHANGELOG.rst)
- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)

---

## By [Harshhaa Reddy](https://www.github.com/NotHarshhaa)
