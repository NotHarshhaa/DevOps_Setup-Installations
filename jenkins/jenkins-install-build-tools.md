# Build Tools Installation and Setup on Jenkins

This guide provides detailed steps for installing Git, Maven, Gradle, Node.js, and npm on a Jenkins server.

## Prerequisites

- Ubuntu/Debian or CentOS/RHEL system
- Root or sudo access
- Internet connection
- Java JDK (for Maven and Gradle)

## 1. Installing Git

### For Ubuntu/Debian:

```bash
# Update package index
sudo apt update

# Install Git
sudo apt install -y git

# Configure Git (optional)
git config --global user.name "Jenkins"
git config --global user.email "jenkins@yourdomain.com"
```

### For CentOS/RHEL:

```bash
# Install Git
sudo yum install -y git

# Configure Git (optional)
git config --global user.name "Jenkins"
git config --global user.email "jenkins@yourdomain.com"
```

### Verify Installation:

```bash
git --version
```

### Jenkins Git Configuration:

1. Navigate to `Manage Jenkins` > `Global Tool Configuration`
2. Scroll to "Git" section
3. Add Git installation:
   - Name: `Git`
   - Path to Git executable: `/usr/bin/git` (auto-detected)

## 2. Installing Maven

### Option A: Using Package Manager (Ubuntu/Debian)

```bash
# Install Maven
sudo apt update
sudo apt install -y maven

# Verify installation
mvn --version
```

### Option B: Manual Installation (Recommended for Latest Version)

```bash
# Install dependencies
sudo apt install -y wget

# Download latest Maven
wget https://downloads.apache.org/maven/maven-3/3.9.5/binaries/apache-maven-3.9.5-bin.tar.gz

# Extract to /opt directory
sudo tar -xzf apache-maven-3.9.5-bin.tar.gz -C /opt
sudo mv /opt/apache-maven-3.9.5 /opt/maven

# Add Maven to PATH
echo 'export M2_HOME=/opt/maven' | sudo tee -a /etc/profile.d/maven.sh
echo 'export PATH=$M2_HOME/bin:$PATH' | sudo tee -a /etc/profile.d/maven.sh
source /etc/profile.d/maven.sh

# Set environment variables
sudo tee -a /etc/environment << EOF
M2_HOME=/opt/maven
MAVEN_HOME=/opt/maven
EOF

# Verify installation
mvn --version

# Clean up
rm apache-maven-3.9.5-bin.tar.gz
```

### For CentOS/RHEL:

```bash
# Download Maven
wget https://downloads.apache.org/maven/maven-3/3.9.5/binaries/apache-maven-3.9.5-bin.tar.gz

# Extract to /opt directory
sudo tar -xzf apache-maven-3.9.5-bin.tar.gz -C /opt
sudo mv /opt/apache-maven-3.9.5 /opt/maven

# Add Maven to PATH
echo 'export M2_HOME=/opt/maven' | sudo tee -a /etc/profile.d/maven.sh
echo 'export PATH=$M2_HOME/bin:$PATH' | sudo tee -a /etc/profile.d/maven.sh
source /etc/profile.d/maven.sh

# Verify installation
mvn --version

# Clean up
rm apache-maven-3.9.5-bin.tar.gz
```

### Jenkins Maven Configuration:

1. Navigate to `Manage Jenkins` > `Global Tool Configuration`
2. Scroll to "Maven" section
3. Add Maven installation:
   - Name: `Maven 3.9.5`
   - MAVEN_HOME: `/opt/maven`

## 3. Installing Gradle

### Manual Installation (Recommended):

```bash
# Install dependencies
sudo apt install -y wget unzip

# Download latest Gradle
wget https://services.gradle.org/distributions/gradle-8.5-bin.zip

# Extract to /opt directory
sudo unzip -q gradle-8.5-bin.zip -d /opt
sudo mv /opt/gradle-8.5 /opt/gradle

# Add Gradle to PATH
echo 'export GRADLE_HOME=/opt/gradle' | sudo tee -a /etc/profile.d/gradle.sh
echo 'export PATH=$GRADLE_HOME/bin:$PATH' | sudo tee -a /etc/profile.d/gradle.sh
source /etc/profile.d/gradle.sh

# Verify installation
gradle --version

# Clean up
rm gradle-8.5-bin.zip
```

### Using SDKMAN (Alternative):

```bash
# Install SDKMAN
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

# Install Gradle
sdk install gradle 8.5

# Verify
gradle --version
```

### Jenkins Gradle Configuration:

1. Navigate to `Manage Jenkins` > `Global Tool Configuration`
2. Scroll to "Gradle" section
3. Add Gradle installation:
   - Name: `Gradle 8.5`
   - GRADLE_HOME: `/opt/gradle`

## 4. Installing Node.js and npm

### Option A: Using NodeSource Repository (Recommended)

```bash
# Install dependencies
sudo apt update
sudo apt install -y curl

# Add NodeSource repository for Node.js 20.x (LTS)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -

# Install Node.js and npm
sudo apt install -y nodejs

# Verify installations
node --version
npm --version
```

### Option B: Using NVM (Node Version Manager)

```bash
# Install NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Load NVM
source ~/.bashrc

# Install Node.js LTS
nvm install --lts
nvm use --lts
nvm alias default lts/*

# Verify installations
node --version
npm --version
```

### For CentOS/RHEL:

```bash
# Install dependencies
sudo yum install -y curl

# Add NodeSource repository
curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -

# Install Node.js and npm
sudo yum install -y nodejs

# Verify installations
node --version
npm --version
```

### Jenkins Node.js Configuration:

1. Navigate to `Manage Jenkins` > `Global Tool Configuration`
2. Scroll to "NodeJS" section (requires NodeJS plugin)
3. Add NodeJS installation:
   - Name: `Node 20`
   - Installation directory: `/usr/bin` (auto-detected)

### Configure npm for Jenkins User:

```bash
# Configure npm to use global directory
sudo mkdir -p /usr/local/lib/node_modules
sudo chown -R jenkins:jenkins /usr/local/lib/node_modules
sudo mkdir -p /usr/local/bin
sudo chown -R jenkins:jenkins /usr/local/bin
```

## 5. Installing Yarn (Modern npm Alternative)

```bash
# Install Yarn globally
npm install -g yarn

# Verify installation
yarn --version
```

## 6. Installing Python (for Python Projects)

```bash
# For Ubuntu/Debian
sudo apt update
sudo apt install -y python3 python3-pip python3-venv

# For CentOS/RHEL
sudo yum install -y python3 python3-pip

# Verify installations
python3 --version
pip3 --version
```

## Jenkins Pipeline Configuration Examples

### Maven Project:

```groovy
pipeline {
    agent any
    
    tools {
        maven 'Maven 3.9.5'
        jdk 'JDK 17'
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
    }
}
```

### Gradle Project:

```groovy
pipeline {
    agent any
    
    tools {
        gradle 'Gradle 8.5'
        jdk 'JDK 17'
    }
    
    stages {
        stage('Build') {
            steps {
                sh './gradlew clean build'
            }
        }
        stage('Test') {
            steps {
                sh './gradlew test'
            }
        }
    }
}
```

### Node.js Project:

```groovy
pipeline {
    agent any
    
    tools {
        nodejs 'Node 20'
    }
    
    stages {
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
        stage('Test') {
            steps {
                sh 'npm test'
            }
        }
    }
}
```

### Multi-Language Project:

```groovy
pipeline {
    agent any
    
    stages {
        stage('Backend Build (Maven)') {
            tools {
                maven 'Maven 3.9.5'
                jdk 'JDK 17'
            }
            steps {
                dir('backend') {
                    sh 'mvn clean package'
                }
            }
        }
        
        stage('Frontend Build (Node.js)') {
            tools {
                nodejs 'Node 20'
            }
            steps {
                dir('frontend') {
                    sh 'npm install'
                    sh 'npm run build'
                }
            }
        }
    }
}
```

## Verification

Create a test Jenkins job to verify all installations:

```groovy
pipeline {
    agent any
    stages {
        stage('Verify Build Tools') {
            steps {
                sh 'git --version'
                sh 'mvn --version'
                sh 'gradle --version'
                sh 'node --version'
                sh 'npm --version'
                sh 'python3 --version'
            }
        }
    }
}
```

## Best Practices

1. **Use specific versions** rather than latest to ensure reproducibility
2. **Store tool versions in version control** using `.nvmrc`, `.sdkmanrc`, or similar
3. **Use Jenkins Global Tool Configuration** for centralized management
4. **Configure caching** for dependencies to speed up builds
5. **Set up proper permissions** for Jenkins user
6. **Use Docker agents** for isolated build environments
7. **Regularly update tools** for security patches and new features

## Troubleshooting

### Permission Issues:

```bash
# Fix permissions for Jenkins user
sudo chown -R jenkins:jenkins /opt/maven
sudo chown -R jenkins:jenkins /opt/gradle
sudo chown -R jenkins:jenkins /usr/local/lib/node_modules
```

### PATH Issues:

Add tool paths to Jenkins environment:

```bash
# In Jenkins system configuration
export PATH=$PATH:/opt/maven/bin:/opt/gradle/bin:/usr/local/bin
```

### Memory Issues:

Increase Maven memory in `~/.mavenrc`:

```bash
export MAVEN_OPTS="-Xmx2048m -XX:MaxPermSize=512m"
```

Once you've completed these steps, Git, Maven, Gradle, Node.js, and npm should be installed and available on your Jenkins server. You can verify the installations by running commands like `git --version`, `mvn --version`, `gradle --version`, `node --version`, and `npm --version` to ensure that the tools are properly installed and configured.
