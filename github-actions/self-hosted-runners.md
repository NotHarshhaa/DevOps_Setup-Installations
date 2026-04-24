# Self-Hosted Runners Installation and Configuration Guide

![Self-Hosted Runners](https://github.blog/wp-content/uploads/2019/08/DL-V2-LinkedIn_FB.png)

Comprehensive guide for installing, configuring, and managing GitHub Actions self-hosted runners.

## Table of Contents
1. [Introduction](#1-introduction)
2. [Linux Runner Installation](#2-linux-runner-installation)
3. [Windows Runner Installation](#3-windows-runner-installation)
4. [macOS Runner Installation](#4-macos-runner-installation)
5. [Docker Runner Installation](#5-docker-runner-installation)
6. [Runner Configuration](#6-runner-configuration)
7. [Runner Management](#7-runner-management)
8. [Runner Security](#8-runner-security)
9. [Troubleshooting](#9-troubleshooting)
10. [Best Practices](#10-best-practices)

---

## 1. Introduction

### 1.1. What are Self-Hosted Runners?

Self-hosted runners are servers that you set up and manage to execute GitHub Actions workflows. They provide:
- Custom hardware specifications
- Local network access
- Specific software dependencies
- Cost control for large workloads
- Data residency compliance

### 1.2. Self-Hosted vs GitHub-Hosted

| Feature | Self-Hosted | GitHub-Hosted |
|---------|-------------|---------------|
| Cost | You pay for infrastructure | Free for public repos |
| Customization | Full control | Limited |
| Maintenance | Your responsibility | Managed by GitHub |
| Security | Your responsibility | Managed by GitHub |
| Performance | Dependent on your hardware | Standardized |

### 1.3. When to Use Self-Hosted Runners

- Need for specific hardware (GPU, specialized processors)
- Local network access requirements
- Data residency or compliance requirements
- Cost optimization for large workloads
- Need for pre-installed software dependencies

---

## 2. Linux Runner Installation

### 2.1. Prerequisites

```bash
# System requirements
# - Ubuntu 20.04+ / Debian 11+ / CentOS 8+ / RHEL 8+
# - Minimum 2 CPU cores
# - Minimum 4 GB RAM
# - Minimum 20 GB disk space
# - Network access to GitHub

# Update system
sudo apt update && sudo apt upgrade -y  # Ubuntu/Debian
# or
sudo yum update -y  # CentOS/RHEL
```

### 2.2. Install Runner

#### Step 1: Create Runner Directory

```bash
mkdir actions-runner
cd actions-runner
```

#### Step 2: Download Runner Package

```bash
# Download the latest runner package
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz

# Or use wget
wget https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
```

#### Step 3: Extract Runner

```bash
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz
```

#### Step 4: Configure Runner

```bash
# Run the configuration script
./config.sh --url https://github.com/YOUR_ORG/YOUR_REPO \
  --token YOUR_TOKEN \
  --name my-runner \
  --labels self-hosted,linux,x64 \
  --work _work
```

#### Step 5: Install as Service

```bash
# Install as a service
sudo ./svc.sh install

# Start the service
sudo ./svc.sh start

# Check status
sudo ./svc.sh status
```

### 2.3. Systemd Service Configuration

```bash
# Create systemd service file
sudo nano /etc/systemd/system/actions-runner.service
```

```ini
[Unit]
Description=GitHub Actions Runner
After=network.target

[Service]
Type=simple
User=runner
WorkingDirectory=/home/runner/actions-runner
ExecStart=/home/runner/actions-runner/run.sh
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable actions-runner
sudo systemctl start actions-runner
```

---

## 3. Windows Runner Installation

### 3.1. Prerequisites

```powershell
# System requirements
# - Windows 10/11 or Windows Server 2019/2022
# - Minimum 2 CPU cores
# - Minimum 4 GB RAM
# - Minimum 20 GB disk space
# - Network access to GitHub
# - .NET Framework 4.8 or later
```

### 3.2. Install Runner

#### Step 1: Create Runner Directory

```powershell
mkdir actions-runner
cd actions-runner
```

#### Step 2: Download Runner Package

```powershell
# Download the latest runner package
Invoke-WebRequest -Uri https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-win-x64-2.311.0.zip -OutFile actions-runner-win-x64-2.311.0.zip
```

#### Step 3: Extract Runner

```powershell
# Extract the installer
Add-Type -AssemblyName System.IO.Compression.FileSystem
[System.IO.Compression.ZipFile]::ExtractToDirectory("$PWD\actions-runner-win-x64-2.311.0.zip", "$PWD")
```

#### Step 4: Configure Runner

```powershell
# Run the configuration script
.\config.cmd --url https://github.com/YOUR_ORG/YOUR_REPO `
  --token YOUR_TOKEN `
  --name my-runner `
  --labels self-hosted,windows,x64 `
  --work _work
```

#### Step 5: Install as Service

```powershell
# Install as a service
.\svc.sh install

# Start the service
.\svc.sh start

# Check status
.\svc.sh status
```

### 3.3. Windows Service Configuration

```powershell
# Configure service to run as specific user
# Navigate to: Services > GitHub Actions Runner > Properties
# Set "Log On As" to your service account
```

---

## 4. macOS Runner Installation

### 4.1. Prerequisites

```bash
# System requirements
# - macOS 11 (Big Sur) or later
# - Minimum 2 CPU cores
# - Minimum 4 GB RAM
# - Minimum 20 GB disk space
# - Network access to GitHub
# - Xcode Command Line Tools
```

### 4.2. Install Runner

#### Step 1: Install Xcode Command Line Tools

```bash
xcode-select --install
```

#### Step 2: Create Runner Directory

```bash
mkdir actions-runner
cd actions-runner
```

#### Step 3: Download Runner Package

```bash
# Download the latest runner package
curl -o actions-runner-osx-x64-2.311.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-osx-x64-2.311.0.tar.gz
```

#### Step 4: Extract Runner

```bash
tar xzf ./actions-runner-osx-x64-2.311.0.tar.gz
```

#### Step 5: Configure Runner

```bash
# Run the configuration script
./config.sh --url https://github.com/YOUR_ORG/YOUR_REPO \
  --token YOUR_TOKEN \
  --name my-runner \
  --labels self-hosted,macos,x64 \
  --work _work
```

#### Step 6: Install as Service

```bash
# Install as a service
sudo ./svc.sh install

# Start the service
sudo ./svc.sh start

# Check status
sudo ./svc.sh status
```

---

## 5. Docker Runner Installation

### 5.1. Docker Container Runner

#### Step 1: Create Dockerfile

```dockerfile
FROM ubuntu:22.04

# Install dependencies
RUN apt-get update && apt-get install -y \
    curl \
    git \
    jq \
    && rm -rf /var/lib/apt/lists/*

# Create runner user
RUN useradd -m runner

# Set working directory
WORKDIR /home/runner

# Download and install runner
RUN curl -o actions-runner-linux-x64-2.311.0.tar.gz -L \
    https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz && \
    tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz && \
    rm actions-runner-linux-x64-2.311.0.tar.gz

# Set permissions
RUN chown -R runner:runner /home/runner

# Switch to runner user
USER runner

# Configure runner (will be done at runtime)
ENTRYPOINT ["/home/runner/entrypoint.sh"]
```

#### Step 2: Create Entry Point Script

```bash
#!/bin/bash
set -e

# Configure runner
/home/runner/actions-runner/config.sh \
  --url ${RUNNER_URL} \
  --token ${RUNNER_TOKEN} \
  --name ${RUNNER_NAME} \
  --labels ${RUNNER_LABELS} \
  --work _work

# Run runner
/home/runner/actions-runner/run.sh
```

#### Step 3: Build and Run Container

```bash
# Build image
docker build -t github-runner:latest .

# Run container
docker run -d \
  --name github-runner \
  -e RUNNER_URL=https://github.com/YOUR_ORG/YOUR_REPO \
  -e RUNNER_TOKEN=YOUR_TOKEN \
  -e RUNNER_NAME=docker-runner \
  -e RUNNER_LABELS=self-hosted,docker,linux \
  -v /var/run/docker.sock:/var/run/docker.sock \
  github-runner:latest
```

### 5.2. Docker Compose Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  runner:
    image: github-runner:latest
    container_name: github-runner
    environment:
      RUNNER_URL: https://github.com/YOUR_ORG/YOUR_REPO
      RUNNER_TOKEN: YOUR_TOKEN
      RUNNER_NAME: docker-runner
      RUNNER_LABELS: self-hosted,docker,linux
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - runner-work:/home/runner/_work
    restart: unless-stopped

volumes:
  runner-work:
```

```bash
# Start runner
docker-compose up -d

# View logs
docker-compose logs -f
```

---

## 6. Runner Configuration

### 6.1. Runner Labels

#### Add Labels During Configuration

```bash
./config.sh --labels self-hosted,linux,x64,docker,gpu
```

#### Add Labels After Configuration

```bash
# Navigate to: Settings > Actions > Runners > Select runner > Settings
# Add labels through the UI
```

#### Use Labels in Workflows

```yaml
jobs:
  build:
    runs-on: [self-hosted, linux, x64, docker]
    steps:
      - uses: actions/checkout@v4
```

### 6.2. Runner Groups

#### Create Runner Group

1. Navigate to organization **Settings** > **Actions** > **Runner groups**
2. Click "New runner group"
3. Configure:
   - **Name**: production-runners
   - **Visibility**: Private
   - **Repositories**: Select repositories

#### Assign Runner to Group

```bash
# Configure runner with group
./config.sh --runnergroup production-runners
```

### 6.3. Runner Capabilities

#### Display Runner Capabilities

```yaml
jobs:
  check-capabilities:
    runs-on: self-hosted
    steps:
      - name: Display capabilities
        run: |
          echo "Runner OS: ${{ runner.os }}"
          echo "Runner Arch: ${{ runner.arch }}"
          echo "Runner Name: ${{ runner.name }}"
```

#### Install Custom Software

```bash
# Install additional software on runner
sudo apt install -y docker-compose
sudo apt install -y nodejs npm
sudo apt install -y python3 python3-pip
```

---

## 7. Runner Management

### 7.1. Update Runner

```bash
# Stop runner
sudo ./svc.sh stop

# Download new version
curl -o actions-runner-linux-x64-NEW_VERSION.tar.gz -L \
  https://github.com/actions/runner/releases/download/vNEW_VERSION/actions-runner-linux-x64-NEW_VERSION.tar.gz

# Extract new version
tar xzf ./actions-runner-linux-x64-NEW_VERSION.tar.gz

# Start runner
sudo ./svc.sh start
```

### 7.2. Remove Runner

```bash
# Stop runner
sudo ./svc.sh stop

# Unconfigure runner
./config.sh remove --token YOUR_TOKEN

# Remove service
sudo ./svc.sh uninstall

# Remove files
cd ..
rm -rf actions-runner
```

### 7.3. Monitor Runner Health

#### Check Runner Status

```bash
# Check service status
sudo ./svc.sh status

# Check runner logs
sudo ./svc.sh log

# Check runner in GitHub
# Navigate to: Settings > Actions > Runners
```

#### Health Check Script

```bash
#!/bin/bash
# health-check.sh

# Check if runner service is running
if sudo systemctl is-active --quiet actions-runner; then
    echo "Runner service is running"
else
    echo "Runner service is not running"
    exit 1
fi

# Check if runner can reach GitHub
if curl -s -o /dev/null -w "%{http_code}" https://github.com | grep -q "200"; then
    echo "GitHub is reachable"
else
    echo "GitHub is not reachable"
    exit 1
fi

echo "All checks passed"
```

---

## 8. Runner Security

### 8.1. Isolate Runner

#### Use Dedicated User

```bash
# Create dedicated user for runner
sudo useradd -m -s /bin/bash github-runner
sudo usermod -aG docker github-runner

# Run as dedicated user
sudo -u github-runner ./config.sh --url ...
```

#### Network Isolation

```bash
# Configure firewall
sudo ufw allow from 192.168.1.0/24 to any port 22
sudo ufw allow from 192.168.1.0/24 to any port 443
sudo ufw deny incoming
sudo ufw enable
```

### 8.2. Runner Hardening

#### System Hardening

```bash
# Update system regularly
sudo apt update && sudo apt upgrade -y

# Install security tools
sudo apt install -y fail2ban ufw

# Configure fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

#### Runner Configuration

```bash
# Configure runner with security options
./config.sh \
  --url https://github.com/YOUR_ORG/YOUR_REPO \
  --token YOUR_TOKEN \
  --ephemeral \
  --disableupdate
```

### 8.3. Access Control

#### Restrict Runner Access

```yaml
# Only allow specific workflows to use runner
# Navigate to: Settings > Actions > Runner groups > Select group > Settings
# Configure access control
```

#### Use Runner Groups

```yaml
# Create separate runner groups for different environments
# - dev-runners: Development environments
# - staging-runners: Staging environments
# - prod-runners: Production environments
```

---

## 9. Troubleshooting

### 9.1. Common Issues

#### Runner Won't Start

```bash
# Check service status
sudo systemctl status actions-runner

# Check logs
sudo journalctl -u actions-runner -n 50

# Check configuration
cat .runner
```

#### Runner Can't Connect to GitHub

```bash
# Test network connectivity
curl -I https://github.com

# Check DNS resolution
nslookup github.com

# Check firewall rules
sudo ufw status
```

#### Runner Jobs Fail

```bash
# Check runner logs
sudo ./svc.sh log

# Check workflow logs in GitHub
# Navigate to: Actions > Select workflow run > View logs
```

### 9.2. Debug Mode

```bash
# Run runner in debug mode
./run.sh --debug

# Enable verbose logging
./config.sh --url ... --debug
```

### 9.3. Reinstall Runner

```bash
# Remove existing configuration
./config.sh remove --token YOUR_TOKEN

# Reconfigure
./config.sh --url ... --token NEW_TOKEN
```

---

## 10. Best Practices

### 10.1. Runner Management

- Use runner groups for organization
- Implement regular updates
- Monitor runner health
- Use ephemeral runners when possible
- Implement auto-scaling for variable workloads

### 10.2. Security

- Isolate runners from production systems
- Use dedicated service accounts
- Regularly update runner software
- Implement network segmentation
- Monitor runner logs for suspicious activity

### 10.3. Performance

- Allocate sufficient resources
- Use SSD storage for better performance
- Implement caching for dependencies
- Clean up workspace regularly
- Monitor resource usage

### 10.4. Maintenance

- Regular system updates
- Log rotation
- Disk space monitoring
- Regular security patches
- Backup runner configuration

---

## References

- [Self-Hosted Runners Documentation](https://docs.github.com/en/actions/hosting-your-own-runners)
- [Monitoring and Troubleshooting](https://docs.github.com/en/actions/hosting-your-own-runners/monitoring-and-troubleshooting-self-hosted-runners)
- [Runner Security](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners)

---

## By [Harshhaa Reddy](https://www.github.com/NotHarshhaa)
