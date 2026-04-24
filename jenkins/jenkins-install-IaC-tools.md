# Setup & Installation for Infrastructure as Code (IaC) Tools on Jenkins

This guide provides detailed steps for installing Terraform, AWS CLI, Azure CLI, and Google Cloud SDK (gcloud) on a Jenkins server.

## Prerequisites

- Ubuntu/Debian or CentOS/RHEL system
- Root or sudo access
- Internet connection

## 1. Installing Terraform

Terraform can be installed using the official HashiCorp repository for automatic updates.

### For Ubuntu/Debian:

```bash
# Install dependencies
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl

# Add HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Add HashiCorp repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# Update and install Terraform
sudo apt update
sudo apt install -y terraform
```

### For CentOS/RHEL:

```bash
# Install yum-utils
sudo yum install -y yum-utils

# Add HashiCorp repository
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo

# Install Terraform
sudo yum install -y terraform
```

### Verify Installation:

```bash
terraform version
```

## 2. Installing AWS CLI v2

The AWS CLI v2 is the recommended version with improved features.

### For Ubuntu/Debian:

```bash
# Download AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-$(uname -m).zip" -o "awscliv2.zip"

# Uninstall previous version if exists
sudo apt remove -y awscli 2>/dev/null || true

# Extract and install
unzip awscliv2.zip
sudo ./aws/install

# Clean up
rm -rf aws awscliv2.zip
```

### For CentOS/RHEL:

```bash
# Download AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-$(uname -m).zip" -o "awscliv2.zip"

# Uninstall previous version if exists
sudo yum remove -y awscli 2>/dev/null || true

# Extract and install
unzip awscliv2.zip
sudo ./aws/install

# Clean up
rm -rf aws awscliv2.zip
```

### Verify Installation:

```bash
aws --version
```

### Configure AWS CLI:

```bash
aws configure
```

You'll be prompted to enter:
- AWS Access Key ID
- AWS Secret Access Key
- Default region name
- Default output format

## 3. Installing Azure CLI

### For Ubuntu/Debian:

```bash
# Install dependencies
sudo apt-get update
sudo apt-get install -y ca-certificates curl apt-transport-https lsb-release gnupg

# Download and install Microsoft signing key
curl -sL https://packages.microsoft.com/keys/microsoft.asc | sudo gpg --dearmor -o /usr/share/keyrings/microsoft-archive-keyring.gpg

# Add Azure CLI repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/microsoft-archive-keyring.gpg] https://packages.microsoft.com/repos/azure-cli/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/azure-cli.list

# Update and install Azure CLI
sudo apt-get update
sudo apt-get install -y azure-cli
```

### For CentOS/RHEL:

```bash
# Download and install Microsoft signing key
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc

# Add Azure CLI repository
sudo sh -c 'echo -e "[azure-cli]
name=Azure CLI
baseurl=https://packages.microsoft.com/yumrepos/azure-cli
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/azure-cli.repo'

# Install Azure CLI
sudo yum install -y azure-cli
```

### Verify Installation:

```bash
az --version
```

### Login to Azure:

```bash
az login
```

## 4. Installing Google Cloud SDK (gcloud)

### For Ubuntu/Debian:

```bash
# Create environment variable for distribution
export CLOUD_SDK_REPO="cloud-sdk-$(lsb_release -cs)"

# Add the Google Cloud SDK distribution URI as a package source
echo "deb [signed-by=/usr/share/keyrings/cloud.google.com.gpg] http://packages.cloud.google.com/apt $CLOUD_SDK_REPO main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

# Import the Google Cloud Platform public key
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.com.gpg add -

# Update the package list and install the Cloud SDK
sudo apt-get update && sudo apt-get install -y google-cloud-sdk

# Optional: Install additional components
sudo apt-get install -y google-cloud-sdk-app-engine-python google-cloud-sdk-app-engine-java
```

### For CentOS/RHEL:

```bash
# Create repo file
sudo tee -a /etc/yum.repos.d/google-cloud-sdk.repo << EOM
[google-cloud-sdk]
name=Google Cloud SDK
baseurl=https://packages.cloud.google.com/yum/repos/cloud-sdk-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOM

# Install Cloud SDK
sudo yum install -y google-cloud-sdk
```

### Verify Installation:

```bash
gcloud version
```

### Initialize gcloud:

```bash
gcloud init
```

## Jenkins Configuration

After installing these tools, configure them in Jenkins:

1. **Global Tool Configuration**:
   - Navigate to `Manage Jenkins` > `Global Tool Configuration`
   - Add installations for Terraform, AWS CLI, Azure CLI, and gcloud
   - Specify installation paths or let Jenkins auto-install

2. **Credentials Management**:
   - Store cloud credentials in Jenkins Credentials
   - Use AWS credentials, Azure service principals, and GCP service accounts securely

3. **Environment Variables**:
   - Set necessary environment variables in Jenkins system configuration
   - Example: `PATH` to include tool binaries

## Verification

Create a test Jenkins job to verify all installations:

```groovy
pipeline {
    agent any
    stages {
        stage('Verify Tools') {
            steps {
                sh 'terraform version'
                sh 'aws --version'
                sh 'az --version'
                sh 'gcloud version'
            }
        }
    }
}
```

## Troubleshooting

### Permission Issues

```bash
# Fix permissions for Jenkins user
sudo usermod -aG docker jenkins  # For Docker access
sudo chmod +x /usr/local/bin/terraform  # If binary not executable
```

### PATH Issues

Add tool paths to Jenkins environment:

```bash
# In Jenkins configuration
export PATH=$PATH:/usr/local/bin:/opt/terraform
```

After completing these steps, Terraform, AWS CLI, Azure CLI, and Google Cloud SDK should be installed and available on your Jenkins server. You can use them in your Jenkins jobs and pipelines to manage infrastructure on various cloud platforms.
