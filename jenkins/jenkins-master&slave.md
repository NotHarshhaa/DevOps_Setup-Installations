# Install Jenkins on Ubuntu & Setup Jenkins Master-Agent Configuration

![jenkins](https://imgur.com/d4TaKyx.png)

This guide provides detailed steps for installing Jenkins on Ubuntu and setting up a master-agent (formerly master-slave) configuration for distributed builds.

## Prerequisites

- Ubuntu 20.04 or later
- Minimum 2GB RAM (4GB recommended)
- 10GB free disk space
- Root or sudo access
- Java JDK 11 or later

## Installing Jenkins on Ubuntu

### 1. Update Package Index

```bash
sudo apt update
sudo apt upgrade -y
```

### 2. Install Java Development Kit (JDK)

Jenkins requires Java 11 or later. Install OpenJDK 17:

```bash
sudo apt install -y openjdk-17-jdk

# Verify Java installation
java -version

# Set JAVA_HOME environment variable
echo 'export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64' | sudo tee -a /etc/environment
source /etc/environment
```

### 3. Add Jenkins Repository

```bash
# Install dependencies
sudo apt install -y curl gnupg

# Add Jenkins GPG key
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo gpg --dearmor -o /usr/share/keyrings/jenkins-keyring.gpg

# Add Jenkins repository
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.gpg] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

### 4. Install Jenkins

```bash
# Update package index
sudo apt update

# Install Jenkins
sudo apt install -y jenkins

# Start Jenkins service
sudo systemctl start jenkins

# Enable Jenkins to start on boot
sudo systemctl enable jenkins

# Check Jenkins service status
sudo systemctl status jenkins
```

### 5. Configure Firewall

```bash
# Allow Jenkins default port 8080
sudo ufw allow 8080/tcp

# Allow SSH (if not already allowed)
sudo ufw allow 22/tcp

# Enable firewall
sudo ufw enable
```

### 6. Access Jenkins Web Interface

1. Open your web browser and navigate to `http://your_server_ip:8080`
2. Retrieve the initial administrator password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

3. Enter the password to unlock Jenkins
4. Select "Install suggested plugins" or choose custom plugins
5. Create an admin user account
6. Set the Jenkins URL (auto-detected, but verify it's correct)
7. Click "Save and Finish" to complete setup

## Jenkins Master-Agent Configuration

### Configure Jenkins Master

1. **Access Jenkins Dashboard**: Navigate to `http://your_server_ip:8080`

2. **Install Required Plugins**:
   - Go to `Manage Jenkins` > `Manage Plugins` > `Available` tab
   - Install "SSH Build Agents" plugin
   - Restart Jenkins if required

### Configure Jenkins Agent (Slave)

#### Option A: Using SSH (Recommended)

**On Agent Machine:**

```bash
# Update package index
sudo apt update

# Install Java JDK
sudo apt install -y openjdk-17-jdk

# Verify Java installation
java -version

# Create Jenkins user
sudo useradd -m -s /bin/bash jenkins

# Set password for Jenkins user
sudo passwd jenkins

# Add Jenkins user to sudoers (optional, for specific tasks)
sudo usermod -aG sudo jenkins

# Create workspace directory
sudo mkdir -p /home/jenkins/workspace
sudo chown -R jenkins:jenkins /home/jenkins/workspace
```

**On Master Machine:**

```bash
# Generate SSH key pair for Jenkins user
sudo -u jenkins ssh-keygen -t rsa -b 4096 -f /var/lib/jenkins/.ssh/id_rsa -N ""

# Copy public key to agent machine
sudo -u jenkins ssh-copy-id jenkins@agent_server_ip

# Test SSH connection
sudo -u jenkins ssh jenkins@agent_server_ip
```

**Configure Agent in Jenkins UI:**

1. Go to `Manage Jenkins` > `Manage Nodes and Clouds`
2. Click "New Node"
3. Enter node name (e.g., `agent-1`)
4. Select "Permanent Agent"
5. Click "Create"
6. Configure the node:
   - **Number of executors**: `2` (or based on CPU cores)
   - **Remote root directory**: `/home/jenkins/workspace`
   - **Labels**: `linux docker maven` (comma-separated)
   - **Usage**: `Use this node as much as possible`
   - **Launch method**: `Launch agents via SSH`
   - **Host**: `agent_server_ip`
   - **Credentials**: Add SSH credentials (username: `jenkins`, private key from master)
   - **Host Key Verification Strategy**: `Known hosts file Verification Strategy`
7. Click "Save"
8. Click "Launch agent" to test connection

#### Option B: Using JNLP (Java Network Launch Protocol)

**On Agent Machine:**

```bash
# Create Jenkins user and workspace
sudo useradd -m -s /bin/bash jenkins
sudo mkdir -p /home/jenkins/workspace
sudo chown -R jenkins:jenkins /home/jenkins/workspace
```

**Configure Agent in Jenkins UI:**

1. Go to `Manage Jenkins` > `Manage Nodes and Clouds`
2. Click "New Node"
3. Enter node name and select "Permanent Agent"
4. Configure the node:
   - **Remote root directory**: `/home/jenkins/workspace`
   - **Launch method**: `Launch agent by connecting it to the controller`
5. Save configuration
6. Copy the agent command shown

**On Agent Machine:**

```bash
# Download agent.jar from master
wget http://master_server_ip:8080/jnlpJars/agent.jar

# Run the agent command (copy from Jenkins UI)
java -jar agent.jar -url http://master_server_ip:8080 -secret <secret_value> -name agent-1
```

To run as a service, create a systemd service file:

```bash
sudo nano /etc/systemd/system/jenkins-agent.service
```

Add the following content:

```ini
[Unit]
Description=Jenkins Agent
After=network.target

[Service]
User=jenkins
WorkingDirectory=/home/jenkins
ExecStart=/usr/bin/java -jar /home/jenkins/agent.jar -url http://master_server_ip:8080 -secret <secret_value> -name agent-1
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
# Reload systemd and start service
sudo systemctl daemon-reload
sudo systemctl start jenkins-agent
sudo systemctl enable jenkins-agent
```

### Configure Docker on Agent (Optional)

```bash
# On agent machine
sudo apt update
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker jenkins
```

## Jenkins Pipeline Configuration for Agents

### Using Node Labels

```groovy
pipeline {
    agent { label 'linux' }
    
    stages {
        stage('Build') {
            steps {
                sh 'echo "Building on ${env.NODE_NAME}"'
            }
        }
    }
}
```

### Using Multiple Agents

```groovy
pipeline {
    agent none
    
    stages {
        stage('Build on Linux') {
            agent { label 'linux' }
            steps {
                sh 'echo "Building on Linux agent"'
            }
        }
        
        stage('Build on Docker Agent') {
            agent { label 'docker' }
            steps {
                sh 'docker --version'
            }
        }
    }
}
```

### Parallel Execution on Multiple Agents

```groovy
pipeline {
    agent none
    
    stages {
        stage('Parallel Builds') {
            parallel {
                stage('Unit Tests') {
                    agent { label 'linux' }
                    steps {
                        sh 'mvn test'
                    }
                }
                stage('Integration Tests') {
                    agent { label 'docker' }
                    steps {
                        sh 'docker-compose up -d'
                    }
                }
            }
        }
    }
}
```

## Best Practices

1. **Use descriptive labels** for agents (e.g., `linux`, `docker`, `maven`, `nodejs`)
2. **Limit executors** based on CPU cores (typically 1-2 per core)
3. **Use SSH launch method** for better security and management
4. **Configure proper resource limits** on agents
5. **Monitor agent status** regularly
6. **Use agent templates** for cloud environments
7. **Implement proper cleanup** for workspaces
8. **Use tool locations** in Global Tool Configuration for agents

## Troubleshooting

### Agent Connection Issues

```bash
# Check Jenkins logs on master
sudo tail -f /var/log/jenkins/jenkins.log

# Check SSH connection from master to agent
sudo -u jenkins ssh -v jenkins@agent_server_ip

# Check agent logs on agent
sudo tail -f /var/log/syslog
```

### Permission Issues

```bash
# Fix permissions on agent
sudo chown -R jenkins:jenkins /home/jenkins/workspace
sudo chmod -R 755 /home/jenkins/workspace
```

### Java Version Mismatch

```bash
# Verify Java version on agent
java -version

# Set JAVA_HOME on agent
echo 'export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64' >> ~/.bashrc
source ~/.bashrc
```

### Service Not Starting

```bash
# Check Jenkins service status
sudo systemctl status jenkins

# Restart Jenkins service
sudo systemctl restart jenkins

# Check for errors in logs
sudo journalctl -u jenkins -n 50
```

## Advanced Configuration

### Configure Agent Clouds (AWS, Azure, GCP)

1. Install respective cloud plugins (EC2, Azure VM, GCE)
2. Go to `Manage Jenkins` > `Manage Nodes and Clouds` > `Configure Clouds`
3. Add cloud configuration with credentials
4. Configure agent templates with instance types and labels

### Configure Kubernetes Agents

1. Install Kubernetes plugin
2. Go to `Manage Jenkins` > `Manage Nodes and Clouds` > `Configure Clouds`
3. Add Kubernetes cloud configuration
4. Configure pod templates with containers and tools

### Configure Load Balancing

1. Use Nginx or HAProxy in front of Jenkins
2. Configure SSL/TLS termination
3. Set up health checks for Jenkins master

That's it! You now have Jenkins installed on Ubuntu with a master-agent configuration set up for distributed builds.
