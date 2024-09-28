![docker](https://vpsie.com/wp-content/uploads/2021/08/Install-Docker-on-linux-And-Windows.png)

## **1. Docker Installation**

### **1.1. For Windows**

#### **Step 1: Download Docker Desktop**

- Go to [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop) and download the installer.

#### **Step 2: Install Docker Desktop**

1. Double-click the downloaded installer file to start the installation.
2. Follow the installation prompts:
   - Ensure that **"Install required components for WSL 2"** is selected.
   - Select **Enable WSL 2 instead of Hyper-V** for better performance.
3. Click **Finish** when the installation completes, and restart your system if prompted.

#### **Step 3: Verify Docker Desktop Installation**

1. Open **PowerShell** and run the following command to verify the installation:

   ```bash
   docker --version
   ```

2. To ensure everything works, run a test container:

   ```bash
   docker run hello-world
   ```

#### **Step 4: Optional - Enable WSL 2**

1. Open **PowerShell** as Administrator and run:

   ```bash
   wsl --set-default-version 2
   ```

2. Install a Linux distribution from the Microsoft Store (Ubuntu, Debian, etc.).

### **1.2. For macOS**

#### **Step 1: Download Docker Desktop**

- Download from the [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop) page.

#### **Step 2: Install Docker Desktop**

1. Open the `.dmg` file and drag **Docker.app** to the **Applications** folder.
2. Start Docker from **Applications** or Spotlight search.
3. Docker should appear in the status bar; wait for it to initialize.

#### **Step 3: Verify Installation**

1. Open **Terminal** and run:

   ```bash
   docker --version
   ```

2. Test Docker with the following command:

   ```bash
   docker run hello-world
   ```

### **1.3. For Linux (Ubuntu Example)**

#### **Step 1: Update Packages**

1. Open a terminal and update your package index:

   ```bash
   sudo apt-get update
   ```

#### **Step 2: Install Required Packages**

```bash
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

#### **Step 3: Add Docker's Official GPG Key**

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

#### **Step 4: Set up the Stable Repository**

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

#### **Step 5: Install Docker Engine**

1. Update the package index and install Docker Engine:

   ```bash
   sudo apt-get update
   sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```

#### **Step 6: Verify Docker Installation**

1. Run the following command:

   ```bash
   sudo docker --version
   ```

2. Test Docker installation:

   ```bash
   sudo docker run hello-world
   ```

---

## **2. Post-Installation Steps**

### **2.1. Manage Docker as a Non-root User (Linux)**

By default, Docker requires `sudo` privileges. To run Docker as a non-root user:

#### **Step 1: Create the Docker Group**

1. Create a Docker group:

   ```bash
   sudo groupadd docker
   ```

#### **Step 2: Add Your User to the Docker Group**

1. Add your user to the Docker group:

   ```bash
   sudo usermod -aG docker $USER
   ```

#### **Step 3: Apply Changes**

1. Log out and log back in so that your group membership is re-evaluated.

#### **Step 4: Verify**

1. Run the following command without `sudo`:

   ```bash
   docker run hello-world
   ```

### **2.2. Verify Docker Installation**

After installation, verify Docker by running the following command:

```bash
docker --version
```

To test if Docker is running correctly:

```bash
docker run hello-world
```

If successful, Docker will pull and run a test image.

### **2.3. Start Docker on Boot (Linux)**

To enable Docker to start automatically on boot:

```bash
sudo systemctl enable docker
```

### **2.4. Optional: Configure Docker Daemon (Linux)**

- You can adjust Docker’s behavior by modifying the daemon configuration file located at:

  ```bash
  /etc/docker/daemon.json
  ```

  Example:

  ```json
  {
    "log-driver": "json-file",
    "log-level": "warn"
  }
  ```

- After making changes, restart Docker:

  ```bash
  sudo systemctl restart docker
  ```

---

## **Author by:**

![](https://imgur.com/2j6Aoyl.png)

> [!NOTE]
> **Join Our** [Telegram Community](https://t.me/prodevopsguy) // [Follow me](https://github.com/NotHarshhaa) **for more DevOps & Cloud content.**
