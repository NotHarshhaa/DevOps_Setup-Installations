# Installing and Integrating Notification and Collaboration Tools on Jenkins

This guide provides detailed steps for installing and integrating Slack, Microsoft Teams, Discord, PagerDuty, and Email notifications on Jenkins.

## Prerequisites

- Jenkins server with administrative access
- Account credentials for notification platforms
- Webhook URLs or API tokens for integrations

## 1. Slack Integration

### Option A: Using Slack Plugin (Recommended)

1. **Install Slack Plugin**:
   - Navigate to `Manage Jenkins` > `Manage Plugins` > `Available` tab
   - Search for "Slack Notification" plugin
   - Select and install the plugin
   - Restart Jenkins if required

2. **Get Slack Webhook URL**:
   - Go to https://api.slack.com/apps
   - Create a new app or use existing one
   - Enable "Incoming Webhooks"
   - Add a new webhook to your workspace
   - Copy the webhook URL

3. **Configure Slack in Jenkins**:
   - Go to `Manage Jenkins` > `Configure System`
   - Scroll to "Slack" section
   - Configure:
     - Workspace: `your-workpace-name`
     - Credential: Add Slack token credential
     - Default channel: `#build-notifications`

4. **Add to Jenkins Pipeline**:

```groovy
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                sh 'echo "Building..."
            }
        }
    }
    
    post {
        success {
            slackSend(
                channel: '#build-notifications',
                color: 'good',
                message: "Build successful: ${env.JOB_NAME} ${env.BUILD_NUMBER} (${env.BUILD_URL})"
            )
        }
        failure {
            slackSend(
                channel: '#build-notifications',
                color: 'danger',
                message: "Build failed: ${env.JOB_NAME} ${env.BUILD_NUMBER} (${env.BUILD_URL})"
            )
        }
    }
}
```

### Option B: Using Webhook Directly

```groovy
pipeline {
    agent any
    
    environment {
        SLACK_WEBHOOK_URL = credentials('slack-webhook-url')
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'echo "Building..."'
            }
        }
    }
    
    post {
        always {
            script {
                def color = currentBuild.result == 'SUCCESS' ? 'good' : 'danger'
                def message = "Build ${currentBuild.result}: ${env.JOB_NAME} ${env.BUILD_NUMBER} (${env.BUILD_URL})"
                
                sh '''
                    curl -X POST -H 'Content-type: application/json' \
                    --data '{"text":"'"${message}"'","attachments":[{"color":"'"${color}"'"}]}' \
                    ${SLACK_WEBHOOK_URL}
                '''
            }
        }
    }
}
```

## 2. Microsoft Teams Integration

### Option A: Using Teams Plugin

1. **Install Teams Plugin**:
   - Navigate to `Manage Jenkins` > `Manage Plugins` > `Available` tab
   - Search for "Microsoft Teams Notification" plugin
   - Select and install the plugin
   - Restart Jenkins if required

2. **Get Teams Webhook URL**:
   - Go to your Teams channel
   - Click "..." > "Connectors"
   - Search for "Incoming Webhook"
   - Configure and copy the webhook URL

3. **Configure in Jenkins**:
   - Go to job configuration
   - Add post-build action "Microsoft Teams Notification"
   - Paste webhook URL
   - Customize message format

### Option B: Using Webhook Directly

```groovy
pipeline {
    agent any
    
    environment {
        TEAMS_WEBHOOK_URL = credentials('teams-webhook-url')
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'echo "Building..."'
            }
        }
    }
    
    post {
        always {
            script {
                def status = currentBuild.result == 'SUCCESS' ? 'Succeeded' : 'Failed'
                def color = currentBuild.result == 'SUCCESS' ? '00ff00' : 'ff0000'
                
                sh '''
                    curl -X POST -H 'Content-Type: application/json' \
                    -d '{
                        "@type": "MessageCard",
                        "@context": "https://schema.org/extensions",
                        "summary": "Jenkins Build Notification",
                        "themeColor": "'"${color}"'",
                        "title": "Build '"${status}"': '"${env.JOB_NAME}"",
                        "sections": [{
                            "activityTitle": "Build '"${env.BUILD_NUMBER}"'",
                            "activitySubtitle": "'"${env.BUILD_URL}"'",
                            "text": "Build '"${status}"' at '"${currentBuild.currentResult}"'"
                        }]
                    }' \
                    ${TEAMS_WEBHOOK_URL}
                '''
            }
        }
    }
}
```

## 3. Discord Integration

### Get Discord Webhook URL:

1. Go to your Discord server settings
2. Create a webhook in your channel
3. Copy the webhook URL

### Jenkins Pipeline Integration:

```groovy
pipeline {
    agent any
    
    environment {
        DISCORD_WEBHOOK_URL = credentials('discord-webhook-url')
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'echo "Building..."'
            }
        }
    }
    
    post {
        always {
            script {
                def color = currentBuild.result == 'SUCCESS' ? 5763719 : 15548997
                def status = currentBuild.result ?: 'UNSTABLE'
                
                sh '''
                    curl -X POST -H 'Content-Type: application/json' \
                    -d '{
                        "embeds": [{
                            "title": "Jenkins Build '"${status}"'",
                            "url": "'"${env.BUILD_URL}"'",
                            "color": '"${color}'",
                            "fields": [
                                {"name": "Job", "value": "'"${env.JOB_NAME}"'", "inline": true},
                                {"name": "Build", "value": "'"${env.BUILD_NUMBER}"'", "inline": true},
                                {"name": "Status", "value": "'"${status}"'", "inline": true}
                            ],
                            "timestamp": "'"${new Date().format("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'", TimeZone.getTimeZone('UTC'))}"'
                        }]
                    }' \
                    ${DISCORD_WEBHOOK_URL}
                '''
            }
        }
    }
}
```

## 4. PagerDuty Integration

### Install PagerDuty Plugin:

1. Navigate to `Manage Jenkins` > `Manage Plugins` > `Available` tab
2. Search for "PagerDuty" plugin
3. Select and install the plugin
4. Restart Jenkins if required

### Configure PagerDuty:

1. Go to `Manage Jenkins` > `Configure System`
2. Scroll to "PagerDuty" section
3. Add PagerDuty API credentials
4. Configure integration key

### Jenkins Pipeline Integration:

```groovy
pipeline {
    agent any
    
    environment {
        PAGERDUTY_INTEGRATION_KEY = credentials('pagerduty-integration-key')
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'echo "Building..."'
            }
        }
    }
    
    post {
        failure {
            script {
                sh '''
                    curl -X POST -H 'Content-Type: application/json' \
                    -H 'Accept: application/vnd.pagerduty+json;version=2' \
                    -d '{
                        "routing_key": "'"${PAGERDUTY_INTEGRATION_KEY}"'",
                        "event_action": "trigger",
                        "payload": {
                            "summary": "Build failed: '"${env.JOB_NAME}"' # '"${env.BUILD_NUMBER}"'",
                            "severity": "critical",
                            "source": "jenkins",
                            "custom_details": {
                                "job_name": "'"${env.JOB_NAME}"'",
                                "build_number": "'"${env.BUILD_NUMBER}"'",
                                "build_url": "'"${env.BUILD_URL}"'",
                                "console_output": "'"${currentBuild.rawBuild.getLog(100)}"'"
                            }
                        }
                    }' \
                    https://events.pagerduty.com/v2/enqueue
                '''
            }
        }
    }
}
```

## 5. Email Notification (SMTP)

### Configure SMTP in Jenkins:

1. Go to `Manage Jenkins` > `Configure System`
2. Scroll to "E-mail Notification" section
3. Configure:
   - SMTP server: `smtp.gmail.com` (or your SMTP server)
   - SMTP port: `587` (or your port)
   - Use SMTP authentication: Check and enter credentials
   - Use SSL: Check if required
   - Reply-to address: `jenkins@yourdomain.com`
   - Default suffix: `@yourdomain.com`
4. Click "Test configuration" to verify

### Add to Jenkins Pipeline:

```groovy
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                sh 'echo "Building..."'
            }
        }
    }
    
    post {
        failure {
            emailext (
                subject: "Build Failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                body: """
                    <p>Build failed for job: ${env.JOB_NAME}</p>
                    <p>Build Number: ${env.BUILD_NUMBER}</p>
                    <p>Build URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    <p>Console Output: <a href="${env.BUILD_URL}console">View Console</a></p>
                """,
                to: 'team@example.com',
                mimeType: 'text/html'
            )
        }
    }
}
```

### Using Gmail SMTP:

```groovy
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                sh 'echo "Building..."'
            }
        }
    }
    
    post {
        always {
            script {
                def subject = "Build ${currentBuild.result}: ${env.JOB_NAME} - ${env.BUILD_NUMBER}"
                def body = """
                    Build Status: ${currentBuild.result}
                    Job: ${env.JOB_NAME}
                    Build Number: ${env.BUILD_NUMBER}
                    Build URL: ${env.BUILD_URL}
                """
                
                mail to: 'team@example.com',
                     subject: subject,
                     body: body
            }
        }
    }
}
```

## 6. Telegram Integration

### Get Telegram Bot Token:

1. Create a bot via @BotFather on Telegram
2. Copy the bot token
3. Get your chat ID via @userinfobot

### Jenkins Pipeline Integration:

```groovy
pipeline {
    agent any
    
    environment {
        TELEGRAM_BOT_TOKEN = credentials('telegram-bot-token')
        TELEGRAM_CHAT_ID = credentials('telegram-chat-id')
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'echo "Building..."'
            }
        }
    }
    
    post {
        always {
            script {
                def message = "Build ${currentBuild.result}: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n${env.BUILD_URL}"
                
                sh '''
                    curl -X POST \
                    "https://api.telegram.org/bot'"${TELEGRAM_BOT_TOKEN}"'/sendMessage" \
                    -d "chat_id='"${TELEGRAM_CHAT_ID}"'" \
                    -d "text='"${message}"'"
                '''
            }
        }
    }
}
```

## Multi-Channel Notification Example

```groovy
def notifySlack(String message, String color) {
    slackSend(
        channel: '#build-notifications',
        color: color,
        message: message
    )
}

def notifyTeams(String message, String status) {
    // Add Teams notification logic
}

def notifyEmail(String subject, String body) {
    emailext(
        subject: subject,
        body: body,
        to: 'team@example.com',
        mimeType: 'text/html'
    )
}

pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                sh 'echo "Building..."'
            }
        }
    }
    
    post {
        success {
            script {
                def message = "Build successful: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
                notifySlack(message, 'good')
                notifyTeams(message, 'Success')
                notifyEmail("Build Success: ${env.JOB_NAME}", message)
            }
        }
        failure {
            script {
                def message = "Build failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
                notifySlack(message, 'danger')
                notifyTeams(message, 'Failed')
                notifyEmail("Build Failed: ${env.JOB_NAME}", message)
            }
        }
    }
}
```

## Best Practices

1. **Use credentials management** - Store webhook URLs and tokens in Jenkins Credentials
2. **Implement rate limiting** - Avoid spamming notification channels
3. **Use conditional notifications** - Only notify on specific events
4. **Include relevant information** - Job name, build number, URL, console output snippet
5. **Use consistent formatting** - Maintain professional notification appearance
6. **Test notifications** - Verify webhook configurations before production use
7. **Handle notification failures** - Don't fail builds if notifications fail
8. **Use appropriate channels** - Separate channels for different environments/projects

## Troubleshooting

### Webhook Issues:

```bash
# Test webhook manually
curl -X POST -H 'Content-type: application/json' \
--data '{"text":"Test message"}' \
https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

### Email Issues:

```bash
# Test SMTP connection
telnet smtp.gmail.com 587
# Check Jenkins logs
tail -f /var/log/jenkins/jenkins.log
```

### Plugin Issues:

```bash
# Check plugin installation
# Manage Jenkins > Manage Plugins > Installed
# Restart Jenkins
sudo systemctl restart jenkins
```

Once you've completed these steps, Jenkins will be able to send notifications to Slack, Microsoft Teams, Discord, PagerDuty, Telegram, and via email using an SMTP server. You can configure these notifications in your Jenkins job configurations to alert users about build statuses, failures, or other events.
