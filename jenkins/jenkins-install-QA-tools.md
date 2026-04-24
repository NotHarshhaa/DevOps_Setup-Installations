# Setup & Integrate Code Quality and Analysis Tools on Jenkins

This guide provides detailed steps for installing and configuring code quality tools including SonarQube, Checkstyle, PMD, SpotBugs, and modern linters on Jenkins.

## Prerequisites

- Jenkins server with administrative access
- Build tools (Maven/Gradle/npm) installed
- Sufficient disk space for analysis reports

## 1. SonarQube Scanner Installation

### Option A: Using SonarQube Scanner for Jenkins Plugin (Recommended)

1. **Install the Plugin**:
   - Navigate to `Manage Jenkins` > `Manage Plugins` > `Available` tab
   - Search for "SonarQube Scanner for Jenkins"
   - Select and install the plugin
   - Restart Jenkins if required

2. **Configure SonarQube Server**:
   - Go to `Manage Jenkins` > `Configure System`
   - Scroll to "SonarQube servers" section
   - Add server with:
     - Name: `SonarQube`
     - Server URL: `http://your-sonarqube-server:9000`
     - Server authentication token (stored in Jenkins Credentials)

3. **Configure SonarQube Scanner**:
   - Go to `Manage Jenkins` > `Global Tool Configuration`
   - Scroll to "SonarQube Scanner" section
   - Add SonarQube Scanner installation:
     - Name: `SonarQubeScanner`
     - Select "Install automatically" or specify installation directory

### Option B: Manual Installation

```bash
# Download SonarQube Scanner
wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip

# Extract to /opt directory
sudo unzip sonar-scanner-cli-5.0.1.3006-linux.zip -d /opt
sudo mv /opt/sonar-scanner-cli-5.0.1.3006-linux /opt/sonar-scanner

# Add to PATH
echo 'export PATH=$PATH:/opt/sonar-scanner/bin' | sudo tee -a /etc/profile.d/sonar-scanner.sh
source /etc/profile.d/sonar-scanner.sh

# Verify installation
sonar-scanner --version
```

### Configure sonar-scanner.properties:

```properties
sonar.host.url=http://your-sonarqube-server:9000
sonar.login=your-auth-token
sonar.projectKey=my-project
sonar.sources=src
sonar.java.binaries=target/classes
```

## 2. Checkstyle Installation

### Using Maven Plugin (Recommended for Java Projects)

Add to your `pom.xml`:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>3.3.0</version>
    <configuration>
        <configLocation>google_checks.xml</configLocation>
        <consoleOutput>true</consoleOutput>
        <failsOnError>false</failsOnError>
    </configuration>
</plugin>
```

### Using Jenkins Plugin

1. Install "Checkstyle" plugin from `Manage Jenkins` > `Manage Plugins`
2. Configure in job settings under "Post-build Actions"
3. Specify checkstyle report pattern: `**/target/checkstyle-result.xml`

### Standalone Installation

```bash
# Download Checkstyle
wget https://github.com/checkstyle/checkstyle/releases/download/checkstyle-10.12.5/checkstyle-10.12.5-all.jar

# Move to /opt directory
sudo mv checkstyle-10.12.5-all.jar /opt/checkstyle.jar

# Create wrapper script
echo 'java -jar /opt/checkstyle.jar "$@"' | sudo tee /usr/local/bin/checkstyle
sudo chmod +x /usr/local/bin/checkstyle

# Verify
checkstyle -version
```

## 3. PMD Installation

### Using Maven Plugin (Recommended for Java Projects)

Add to your `pom.xml`:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-pmd-plugin</artifactId>
    <version>3.21.0</version>
    <configuration>
        <linkXRef>false</linkXRef>
        <sourceEncoding>utf-8</sourceEncoding>
        <minimumTokens>100</minimumTokens>
    </configuration>
</plugin>
```

### Using Jenkins Plugin

1. Install "PMD" plugin from `Manage Jenkins` > `Manage Plugins`
2. Configure in job settings under "Post-build Actions"
3. Specify PMD report pattern: `**/target/pmd.xml`

### Standalone Installation

```bash
# Download PMD
wget https://github.com/pmd/pmd/releases/download/pmd_releases%2F7.0.0/pmd-dist-7.0.0-bin.zip

# Extract to /opt
sudo unzip pmd-dist-7.0.0-bin.zip -d /opt
sudo mv /opt/pmd-dist-7.0.0 /opt/pmd

# Add to PATH
echo 'export PATH=$PATH:/opt/pmd/bin' | sudo tee -a /etc/profile.d/pmd.sh
source /etc/profile.d/pmd.sh

# Verify
run.sh pmd --version
```

## 4. SpotBugs (Replacement for FindBugs)

**Note**: FindBugs is deprecated. Use SpotBugs instead.

### Using Maven Plugin (Recommended for Java Projects)

Add to your `pom.xml`:

```xml
<plugin>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-maven-plugin</artifactId>
    <version>4.7.3.6</version>
    <configuration>
        <effort>Max</effort>
        <threshold>Low</threshold>
        <failOnError>false</failOnError>
    </configuration>
</plugin>
```

### Using Jenkins Plugin

1. Install "SpotBugs" plugin from `Manage Jenkins` > `Manage Plugins`
2. Configure in job settings under "Post-build Actions"
3. Specify SpotBugs report pattern: `**/target/spotbugsXml.xml`

### Standalone Installation

```bash
# Download SpotBugs
wget https://github.com/spotbugs/spotbugs/releases/download/4.8.0/spotbugs-4.8.0.tgz

# Extract to /opt
sudo tar -xzf spotbugs-4.8.0.tgz -C /opt
sudo mv /opt/spotbugs-4.8.0 /opt/spotbugs

# Add to PATH
echo 'export PATH=$PATH:/opt/spotbugs/bin' | sudo tee -a /etc/profile.d/spotbugs.sh
source /etc/profile.d/spotbugs.sh

# Verify
spotbugs -version
```

## 5. ESLint (for JavaScript/TypeScript)

### Installation via npm

```bash
# Install ESLint globally
npm install -g eslint

# Or install locally in project
npm install --save-dev eslint
```

### Configuration

Create `.eslintrc.json` in your project:

```json
{
    "env": {
        "browser": true,
        "es2021": true,
        "node": true
    },
    "extends": "eslint:recommended",
    "parserOptions": {
        "ecmaVersion": "latest",
        "sourceType": "module"
    },
    "rules": {
        "no-unused-vars": "warn",
        "no-console": "off"
    }
}
```

### Jenkins Integration

```groovy
stage('ESLint') {
    steps {
        sh 'npm run lint'
    }
}
```

## 6. Prettier (for Code Formatting)

### Installation

```bash
npm install -g prettier
```

### Jenkins Integration

```groovy
stage('Prettier Check') {
    steps {
        sh 'prettier --check "src/**/*.{js,jsx,ts,tsx,json,css,md}"'
    }
}
```

## Jenkins Pipeline Integration Examples

### Maven Project with All Quality Tools

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
                sh 'mvn clean compile'
            }
        }
        
        stage('Code Quality Checks') {
            parallel {
                stage('Checkstyle') {
                    steps {
                        sh 'mvn checkstyle:check'
                    }
                }
                stage('PMD') {
                    steps {
                        sh 'mvn pmd:check'
                    }
                }
                stage('SpotBugs') {
                    steps {
                        sh 'mvn spotbugs:check'
                    }
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
    }
    
    post {
        always {
            checkstyle pattern: '**/target/checkstyle-result.xml'
            pmd pattern: '**/target/pmd.xml'
            spotbugs pattern: '**/target/spotbugsXml.xml'
        }
    }
}
```

### Node.js Project with ESLint

```groovy
pipeline {
    agent any
    
    stages {
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        
        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }
        
        stage('Format Check') {
            steps {
                sh 'npm run format:check'
            }
        }
    }
}
```

## Quality Gate Configuration

### Configure SonarQube Quality Gate in Jenkins

```groovy
stage('Quality Gate') {
    steps {
        script {
            timeout(time: 5, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
            }
        }
    }
}
```

## Best Practices

1. **Run quality checks in parallel** to reduce build time
2. **Fail builds on critical issues** but warn on minor issues
3. **Use consistent configuration** across all projects
4. **Regularly update tool versions** for latest rule sets
5. **Store configuration files** in version control
6. **Customize rules** based on project requirements
7. **Integrate with pull requests** for early feedback

## Troubleshooting

### Plugin Installation Issues

```bash
# Check Jenkins logs
tail -f /var/log/jenkins/jenkins.log

# Restart Jenkins if needed
sudo systemctl restart jenkins
```

### Permission Issues

```bash
# Fix permissions for Jenkins user
sudo chown -R jenkins:jenkins /opt/sonar-scanner
sudo chown -R jenkins:jenkins /opt/checkstyle.jar
```

### Memory Issues

Increase Jenkins heap memory in `/etc/default/jenkins`:

```bash
JAVA_ARGS="-Xmx4096m"
```

By following these steps, you should be able to install and configure modern code quality tools on your Jenkins server and integrate them into your Jenkins builds for comprehensive code analysis.
