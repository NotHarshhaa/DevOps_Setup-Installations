# Setup & Installation for Testing Tools on Jenkins

This guide provides detailed steps for installing and configuring testing frameworks including JUnit 5, Selenium, TestNG, Cypress, and other testing tools on Jenkins.

## Prerequisites

- Jenkins server with administrative access
- Build tools (Maven/Gradle/npm) installed
- Browser drivers for Selenium tests
- Sufficient resources for test execution

## 1. JUnit 5 Installation

### Using Maven

Add JUnit 5 dependencies to your `pom.xml`:

```xml
<dependencies>
    <!-- JUnit 5 Jupiter API -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-api</artifactId>
        <version>5.10.1</version>
        <scope>test</scope>
    </dependency>
    
    <!-- JUnit 5 Jupiter Engine -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-engine</artifactId>
        <version>5.10.1</version>
        <scope>test</scope>
    </dependency>
    
    <!-- JUnit 5 Jupiter Params -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-params</artifactId>
        <version>5.10.1</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.2</version>
        </plugin>
    </plugins>
</build>
```

### Using Gradle

Add JUnit 5 dependencies to your `build.gradle`:

```groovy
test {
    useJUnitPlatform()
}

dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter-api:5.10.1'
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.10.1'
}
```

### Jenkins Configuration

```groovy
pipeline {
    agent any
    
    tools {
        maven 'Maven 3.9.5'
        jdk 'JDK 17'
    }
    
    stages {
        stage('Test') {
            steps {
                sh 'mvn clean test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }
    }
}
```

## 2. TestNG Installation

### Using Maven

```xml
<dependencies>
    <dependency>
        <groupId>org.testng</groupId>
        <artifactId>testng</artifactId>
        <version>7.8.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.2</version>
            <configuration>
                <suiteXmlFiles>
                    <suiteXmlFile>testng.xml</suiteXmlFile>
                </suiteXmlFiles>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### Using Gradle

```groovy
dependencies {
    testImplementation 'org.testng:testng:7.8.0'
}

test {
    useTestNG()
}
```

## 3. Selenium Installation

### Using Maven

```xml
<dependencies>
    <!-- Selenium WebDriver -->
    <dependency>
        <groupId>org.seleniumhq.selenium</groupId>
        <artifactId>selenium-java</artifactId>
        <version>4.16.1</version>
    </dependency>
    
    <!-- WebDriver Manager for automatic driver management -->
    <dependency>
        <groupId>io.github.bonigarcia</groupId>
        <artifactId>webdrivermanager</artifactId>
        <version>5.6.2</version>
        <scope>test</scope>
    </dependency>
    
    <!-- JUnit 5 for Selenium tests -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-api</artifactId>
        <version>5.10.1</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-engine</artifactId>
        <version>5.10.1</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Install Browser Drivers on Jenkins Server

```bash
# For Ubuntu/Debian
sudo apt update
sudo apt install -y chromium-browser firefox

# Install ChromeDriver
wget https://chromedriver.storage.googleapis.com/114.0.5735.90/chromedriver_linux64.zip
sudo unzip chromedriver_linux64.zip -d /usr/local/bin/
sudo chmod +x /usr/local/bin/chromedriver

# Install GeckoDriver (for Firefox)
wget https://github.com/mozilla/geckodriver/releases/download/v0.34.0/geckodriver-v0.34.0-linux64.tar.gz
tar -xzf geckodriver-v0.34.0-linux64.tar.gz
sudo mv geckodriver /usr/local/bin/
sudo chmod +x /usr/local/bin/geckodriver
```

### Headless Browser Configuration

```java
import io.github.bonigarcia.wdm.WebDriverManager;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;

public class SeleniumTest {
    @Test
    public void testExample() {
        WebDriverManager.chromedriver().setup();
        
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--headless");
        options.addArguments("--no-sandbox");
        options.addArguments("--disable-dev-shm-usage");
        options.addArguments("--disable-gpu");
        
        WebDriver driver = new ChromeDriver(options);
        driver.get("https://example.com");
        
        // Your test logic here
        
        driver.quit();
    }
}
```

### Jenkins Pipeline for Selenium

```groovy
pipeline {
    agent any
    
    stages {
        stage('Selenium Tests') {
            steps {
                sh 'mvn clean test -Dtest=SeleniumTest'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }
    }
}
```

## 4. Cypress Installation

### Installation via npm

```bash
npm install --save-dev cypress
```

### Configuration

Create `cypress.config.js`:

```javascript
const { defineConfig } = require("cypress");

module.exports = defineConfig({
  e2e: {
    setupNodeEvents(on, config) {
      // implement node event listeners here
    },
    baseUrl: "https://example.com",
    video: false,
    screenshotOnRunFailure: true,
  },
});
```

### Jenkins Pipeline for Cypress

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
        
        stage('Cypress Tests') {
            steps {
                sh 'npm run cypress:run'
            }
            post {
                always {
                    publishHTML(target: [
                        reportDir: 'cypress/reports',
                        reportFiles: 'index.html',
                        reportName: 'Cypress Report'
                    ])
                }
            }
        }
    }
}
```

### Using Cypress Dashboard

```groovy
pipeline {
    agent any
    
    environment {
        CYPRESS_RECORD_KEY = credentials('cypress-record-key')
    }
    
    stages {
        stage('Cypress Tests') {
            steps {
                sh '''
                    npm run cypress:run --record \
                    --key ${CYPRESS_RECORD_KEY} \
                    --parallel
                '''
            }
        }
    }
}
```

## 5. Playwright Installation

### Installation via npm

```bash
npm install --save-dev @playwright/test
n```

### Install Browsers

```bash
npx playwright install
```

### Configuration

Create `playwright.config.ts`:

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'https://example.com',
    trace: 'on-first-retry',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
  ],
});
```

### Jenkins Pipeline for Playwright

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
        
        stage('Install Playwright Browsers') {
            steps {
                sh 'npx playwright install --with-deps'
            }
        }
        
        stage('Playwright Tests') {
            steps {
                sh 'npx playwright test'
            }
            post {
                always {
                    publishHTML(target: [
                        reportDir: 'playwright-report',
                        reportFiles: 'index.html',
                        reportName: 'Playwright Report'
                    ])
                }
            }
        }
    }
}
```

## 6. Jest Installation

### Installation via npm

```bash
npm install --save-dev jest @types/jest
```

### Configuration

Create `jest.config.js`:

```javascript
module.exports = {
  testEnvironment: 'node',
  coverageDirectory: 'coverage',
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/**/*.test.js',
  ],
};
```

### Jenkins Pipeline for Jest

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
        
        stage('Jest Tests') {
            steps {
                sh 'npm test -- --coverage'
            }
            post {
                always {
                    publishHTML(target: [
                        reportDir: 'coverage/lcov-report',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }
    }
}
```

## 7. pytest Installation

### Installation via pip

```bash
pip install pytest pytest-cov pytest-html
```

### Jenkins Pipeline for pytest

```groovy
pipeline {
    agent any
    
    stages {
        stage('Install Dependencies') {
            steps {
                sh 'pip install -r requirements.txt'
            }
        }
        
        stage('pytest Tests') {
            steps {
                sh '''
                    pytest --cov=. --cov-report=html \
                    --html=test-report.html \
                    --self-contained-html
                '''
            }
            post {
                always {
                    publishHTML(target: [
                        reportDir: '.',
                        reportFiles: 'test-report.html',
                        reportName: 'pytest Report'
                    ])
                    publishHTML(target: [
                        reportDir: 'htmlcov',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }
    }
}
```

## Parallel Test Execution

```groovy
pipeline {
    agent any
    
    stages {
        stage('Parallel Tests') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'mvn test -Dtest=UnitTest'
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'mvn test -Dtest=IntegrationTest'
                    }
                }
                stage('E2E Tests') {
                    steps {
                        sh 'mvn test -Dtest=E2ETest'
                    }
                }
            }
        }
    }
}
```

## Test Reporting Plugins

### Install Required Plugins

1. **JUnit Plugin** - For JUnit test reports
2. **HTML Publisher Plugin** - For HTML test reports
3. **Cucumber Reports Plugin** - For Cucumber test reports
4. **Allure Plugin** - For Allure test reports

### Allure Report Configuration

```groovy
pipeline {
    agent any
    
    stages {
        stage('Test') {
            steps {
                sh 'mvn clean test'
            }
        }
    }
    
    post {
        always {
            script {
                allure includeProperties: false,
                       jdk: '',
                       results: [[path: 'target/allure-results']]
            }
        }
    }
}
```

## Best Practices

1. **Use headless browsers** for CI/CD environments
2. **Implement parallel test execution** to reduce build time
3. **Use test containers** for isolated test environments
4. **Implement proper test data management**
5. **Use test reporting** for visibility
6. **Implement retry logic** for flaky tests
7. **Use test coverage thresholds** to enforce quality
8. **Mock external dependencies** for reliable tests

## Troubleshooting

### Browser Driver Issues:

```bash
# Check browser driver versions
chromedriver --version
geckodriver --version

# Update browser drivers
sudo apt update
sudo apt upgrade chromium-browser firefox
```

### Headless Browser Issues:

```bash
# Install xvfb for virtual display
sudo apt install -y xvfb

# Run tests with xvfb
xvfb-run mvn test
```

### Permission Issues:

```bash
# Fix permissions for Jenkins user
sudo chown -R jenkins:jenkins /var/lib/jenkins/.cache
```

By following these steps, you should be able to install and configure modern testing frameworks on your Jenkins server. Remember to configure your Jenkins build jobs to execute the appropriate test commands and integrate test reporting for visibility into test results.
