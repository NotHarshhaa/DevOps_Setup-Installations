# Docker Installation Guide

![docker](https://vpsie.com/wp-content/uploads/2021/08/Install-Docker-on-linux-And-Windows.png)

This comprehensive guide covers Docker installation on Windows, macOS, and Linux distributions (Ubuntu/Debian and CentOS/RHEL).

## Prerequisites

- **Windows**: Windows 10/11 64-bit with Pro, Enterprise, or Education editions
- **macOS**: macOS 10.15 or newer
- **Linux**: 64-bit system with kernel 3.10 or higher
- **System Requirements**: Minimum 4GB RAM, 20GB free disk space
- **Internet Connection**: Required for downloading Docker images

## 1. Docker Installation

### 1.1. For Windows

#### Step 1: Download Docker Desktop

- Go to [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop) and download the installer.

#### Step 2: Install Docker Desktop

1. Double-click the downloaded installer file to start the installation.
2. Follow the installation prompts:
   - Ensure that **"Use WSL 2 instead of Hyper-V"** is selected for better performance.
   - Check **"Add shortcut to desktop"** if desired.
3. Click **Finish** when the installation completes.
4. Restart your system if prompted.

#### Step 3: Verify Docker Desktop Installation

1. Open **PowerShell** or **Command Prompt** and run:

   ```bash
   docker --version
   docker compose version
   ```

2. To ensure everything works, run a test container:

   ```bash
   docker run hello-world
   ```

#### Step 4: Configure WSL 2 (If not enabled)

1. Open **PowerShell** as Administrator and run:

   ```bash
   wsl --install
   wsl --set-default-version 2
   ```

2. Restart your computer and install a Linux distribution from the Microsoft Store (Ubuntu recommended).

### 1.2. For macOS

#### Step 1: Download Docker Desktop

- Download from the [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop) page.

#### Step 2: Install Docker Desktop

1. Open the `.dmg` file and drag **Docker.app** to the **Applications** folder.
2. Start Docker from **Applications** or Spotlight search.
3. Docker should appear in the menu bar; wait for it to initialize (whale icon stops animating).
4. Accept the Docker Desktop license agreement.

#### Step 3: Verify Installation

1. Open **Terminal** and run:

   ```bash
   docker --version
   docker compose version
   ```

2. Test Docker with the following command:

   ```bash
   docker run hello-world
   ```

### 1.3. For Linux (Ubuntu/Debian)

#### Step 1: Update Packages

```bash
sudo apt-get update
sudo apt-get upgrade -y
```

#### Step 2: Install Required Packages

```bash
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    apt-transport-https
```

#### Step 3: Add Docker's Official GPG Key

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

#### Step 4: Set up the Stable Repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

#### Step 5: Install Docker Engine

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### Step 6: Verify Docker Installation

```bash
sudo docker --version
sudo docker run hello-world
```

### 1.4. For Linux (CentOS/RHEL/Rocky Linux)

#### Step 1: Update Packages

```bash
sudo yum update -y
```

#### Step 2: Install Required Packages

```bash
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

#### Step 3: Add Docker's Official Repository

```bash
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

#### Step 4: Install Docker Engine

```bash
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### Step 5: Start and Enable Docker

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

#### Step 6: Verify Docker Installation

```bash
sudo docker --version
sudo docker run hello-world
```

### 1.5. For Linux (Fedora)

#### Step 1: Update Packages

```bash
sudo dnf update -y
```

#### Step 2: Install Required Packages

```bash
sudo dnf -y install dnf-plugins-core
```

#### Step 3: Add Docker's Official Repository

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
```

#### Step 4: Install Docker Engine

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### Step 5: Start and Enable Docker

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

#### Step 6: Verify Docker Installation

```bash
sudo docker --version
sudo docker run hello-world
```

## 2. Post-Installation Steps

### 2.1. Manage Docker as a Non-root User (Linux)

By default, Docker requires `sudo` privileges. To run Docker as a non-root user:

#### Step 1: Create the Docker Group

```bash
sudo groupadd docker
```

#### Step 2: Add Your User to the Docker Group

```bash
sudo usermod -aG docker $USER
```

#### Step 3: Apply Changes

Log out and log back in so that your group membership is re-evaluated.

#### Step 4: Verify

```bash
docker run hello-world
```

### 2.2. Configure Docker to Start on Boot (Linux)

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

### 2.3. Configure Docker Daemon (Linux)

You can adjust Docker's behavior by modifying the daemon configuration file:

```bash
sudo nano /etc/docker/daemon.json
```

Example configuration:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "registry-mirrors": [
    "https://mirror.gcr.io"
  ],
  "insecure-registries": [],
  "live-restore": true
}
```

After making changes, restart Docker:

```bash
sudo systemctl restart docker
```

### 2.4. Verify Docker Installation

```bash
# Check Docker version
docker --version

# Check Docker system information
docker info

# Test Docker with hello-world
docker run hello-world

# Check running containers
docker ps

# Check Docker images
docker images
```

## 3. Common Docker Commands

### Container Management

```bash
# Run a container
docker run -d -p 80:80 --name my-nginx nginx

# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# Stop a container
docker stop <container_id>

# Start a stopped container
docker start <container_id>

# Remove a container
docker rm <container_id>

# Remove all stopped containers
docker container prune
```

### Image Management

```bash
# Pull an image
docker pull nginx:latest

# List images
docker images

# Build an image from Dockerfile
docker build -t myapp:1.0 .

# Remove an image
docker rmi <image_id>

# Remove unused images
docker image prune -a
```

### Logs and Debugging

```bash
# View container logs
docker logs <container_id>

# Follow logs in real-time
docker logs -f <container_id>

# Execute command in running container
docker exec -it <container_id> /bin/bash

# Inspect container details
docker inspect <container_id>
```

### System Management

```bash
# Check Docker disk usage
docker system df

# Remove unused data (containers, networks, images, build cache)
docker system prune -a

# View Docker system information
docker info
```

## 4. Troubleshooting

### Issue: Permission Denied

**Problem**: Cannot run Docker without sudo

**Solution**:
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Issue: Docker Daemon Not Running

**Problem**: "Cannot connect to the Docker daemon"

**Solution**:
```bash
sudo systemctl start docker
sudo systemctl status docker
```

### Issue: WSL 2 Issues on Windows

**Problem**: Docker Desktop fails to start on WSL 2

**Solution**:
1. Update WSL 2 kernel
2. Run in PowerShell as Administrator:
   ```powershell
   wsl --update
   wsl --shutdown
   ```

### Issue: Port Already in Use

**Problem**: Container fails to start due to port conflict

**Solution**:
```bash
# Find process using the port
sudo lsof -i :80
# Or use a different port
docker run -d -p 8080:80 nginx
```

### Issue: Out of Disk Space

**Problem**: Docker uses too much disk space

**Solution**:
```bash
# Clean up unused resources
docker system prune -a --volumes

# Check disk usage
docker system df
```

### Issue: Container Cannot Access Internet

**Problem**: Container has no network connectivity

**Solution**:
```bash
# Restart Docker daemon
sudo systemctl restart docker

# Check DNS configuration in daemon.json
sudo nano /etc/docker/daemon.json
# Add: {"dns": ["8.8.8.8", "8.8.4.4"]}
sudo systemctl restart docker
```

## 5. Uninstalling Docker

### Windows

1. Go to **Settings > Apps > Apps & features**
2. Find **Docker Desktop** and click **Uninstall**
3. Delete the Docker data directories (optional):
   - `C:\ProgramData\Docker`
   - `C:\Users\<your-user>\AppData\Roaming\Docker`

### macOS

1. Quit Docker Desktop from the menu bar
2. Move **Docker.app** from Applications to Trash
3. Remove Docker data directories (optional):
   ```bash
   rm -rf ~/Library/Containers/com.docker.docker
   rm -rf ~/Library/Application\ Support/Docker\ Desktop
   ```

### Linux (Ubuntu/Debian)

```bash
sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo apt-get autoremove -y
sudo rm -rf /var/lib/docker
sudo rm -rf /etc/docker
```

### Linux (CentOS/RHEL)

```bash
sudo yum remove docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo rm -rf /var/lib/docker
sudo rm -rf /etc/docker
```

## 6. Additional Resources

- [Official Docker Documentation](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Docker Security](https://docs.docker.com/engine/security/)

## Author by:

![test](https://imgur.com/2j6Aoyl.png)

> [!NOTE]
> **Join Our** [Telegram Community](https://t.me/prodevopsguy) // [Follow me](https://github.com/NotHarshhaa) **for more DevOps & Cloud content.**
