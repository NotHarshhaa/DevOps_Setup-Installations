# **CI/CD integration with Docker**

![docker](https://spaceliftio.wpcomstaging.com/wp-content/uploads/2024/04/367.docker-ci-cd.png)

## **Prerequisites**

Before proceeding, ensure the following prerequisites are met:

- **Docker** is installed on the Jenkins server.
- **Jenkins** is installed and running.
- The Jenkins user has **Docker** permissions.
- You have a **Git repository** with your application’s code and a **Dockerfile**.

---

## **1. Setting Up Jenkins with Docker**

### **1.1. Install Jenkins on Docker (Optional)**

If you want to run Jenkins inside Docker, here’s how to install it:

1. **Pull Jenkins Docker Image:**

   ```bash
   docker pull jenkins/jenkins:lts
   ```

2. **Run Jenkins Container:**

   ```bash
   docker run -d --name jenkins -p 8080:8080 -p 50000:50000 \
   -v jenkins_home:/var/jenkins_home \
   -v /var/run/docker.sock:/var/run/docker.sock \
   jenkins/jenkins:lts
   ```

   This mounts the Docker socket inside the Jenkins container so Jenkins can run Docker commands.

3. **Access Jenkins:**
   - Visit `http://<your-server-ip>:8080` in your browser to access Jenkins.
   - Follow the installation steps, install the required plugins, and set up an admin user.

### **1.2. Install Required Plugins**

1. In Jenkins, go to **Manage Jenkins > Manage Plugins**.
2. Install the following plugins:
   - **Pipeline** (for Jenkins pipelines).
   - **Git** (for Git repository integration).
   - **Docker Pipeline** (to integrate Docker into Jenkins pipelines).

---

## **2. Setting Up a Jenkins Pipeline for Docker**

### **2.1. Write a Jenkinsfile**

A `Jenkinsfile` defines the CI/CD pipeline, including steps for building Docker images and running containers.

#### **Sample Jenkinsfile**

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'your_dockerhub_username/your_image_name'
    }

    stages {
        stage('Clone Repository') {
            steps {
                git 'https://github.com/yourusername/your-repo.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${env.BUILD_NUMBER}")
                }
            }
        }

        stage('Test Docker Image') {
            steps {
                script {
                    docker.image("${DOCKER_IMAGE}:${env.BUILD_NUMBER}").inside {
                        sh 'echo "Run your test commands here"'
                    }
                }
            }
        }

        stage('Push Docker Image to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials') {
                        docker.image("${DOCKER_IMAGE}:${env.BUILD_NUMBER}").push()
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    docker.image("${DOCKER_IMAGE}:${env.BUILD_NUMBER}").run('-p 8080:8080')
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up...'
            sh 'docker system prune -f'
        }
    }
}
```

### **Explanation of the Jenkinsfile**

- **Pipeline Stages**:
  - **Clone Repository**: Pulls the code from a Git repository.
  - **Build Docker Image**: Builds a Docker image using the `Dockerfile` in your repository.
  - **Test Docker Image**: Runs any test commands inside a temporary container.
  - **Push Docker Image to Docker Hub**: Pushes the Docker image to Docker Hub.
  - **Deploy**: Runs the newly built Docker container.
  
- **DOCKER_IMAGE**: An environment variable used to define the Docker image name.
- **docker.build**: Builds the Docker image with a tag using the build number.
- **docker.withRegistry**: Pushes the image to Docker Hub using Docker credentials saved in Jenkins.
- **docker.run**: Runs the container on port `8080`.

---

### **2.2. Setting Up Docker Hub Credentials in Jenkins**

1. In Jenkins, go to **Manage Jenkins > Manage Credentials**.
2. Add your **Docker Hub** credentials:
   - **Kind**: Username with password.
   - **Scope**: Global.
   - **ID**: `dockerhub-credentials` (this is used in the Jenkinsfile).

---

## **3. Setting Up the Jenkins Pipeline Job**

### **3.1. Create a New Pipeline Job**

1. In Jenkins, click **New Item**.
2. Choose **Pipeline** and give it a name.
3. In the **Pipeline** section:
   - Select **Pipeline script from SCM**.
   - Choose **Git** and add your repository URL.
   - Set the branch (e.g., `main` or `master`).
   - Set the path to your `Jenkinsfile`.

### **3.2. Triggering Builds Automatically**

You can configure Jenkins to trigger builds automatically when changes are pushed to the repository.

1. Set up **webhooks** in GitHub or GitLab to trigger Jenkins when there’s a code change.
2. In Jenkins, enable **Poll SCM** to check the repository periodically for updates:
   - Go to your job’s configuration.
   - Under **Build Triggers**, check **Poll SCM** and set a schedule (e.g., `H/5 * * * *` to check every 5 minutes).

---

## **4. Running the Pipeline**

1. After configuring the pipeline, you can run the job manually or trigger it automatically based on the SCM configuration.
2. Jenkins will:
   - Clone the repository.
   - Build the Docker image.
   - Run any tests (if specified).
   - Push the Docker image to Docker Hub.
   - Deploy the Docker container.

You can monitor the job’s progress and logs in Jenkins.

---

## **5. Testing and Validation**

- **Check Docker Image**: Ensure that the Docker image has been pushed to your Docker Hub repository.
- **Check Running Containers**: Verify that the Docker container is running:

   ```bash
   docker ps
   ```

   You should see the container running and accessible on the specified port (e.g., port 8080).

- **Access Application**: Open `http://<your-server-ip>:8080` in a browser to access the application running inside the Docker container.

---

## **6. Post-Build Cleanup**

After each build, it’s good practice to clean up unused images, containers, and volumes to save space. The `docker system prune -f` command in the **post** section of the Jenkinsfile will ensure this.

---

## **7. Security Considerations**

- Ensure your Docker Daemon is secured, especially if running Jenkins and Docker on the same server.
- Limit Docker privileges and set up proper user permissions in the Docker group.
- Implement scanning of Docker images for vulnerabilities before pushing them to Docker Hub, using tools like **Docker Scout** or **Trivy**.

---

This setup will give you a fully functional CI/CD pipeline integrated with Docker, allowing you to build, test, and deploy containerized applications automatically.

## **Author by:**

![test](https://imgur.com/2j6Aoyl.png)

> [!NOTE]
> **Join Our** [Telegram Community](https://t.me/prodevopsguy) // [Follow me](https://github.com/NotHarshhaa) **for more DevOps & Cloud content.**
