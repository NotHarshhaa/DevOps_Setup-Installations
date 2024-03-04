# Setup Containerization Tools on Jenkins

Certainly! Below are the detailed steps for installing Docker and Kubernetes CLI (kubectl) on a Jenkins server:

1. **Installing Docker**:

   Docker provides a convenient script for installing Docker Engine on Linux systems. You can execute this script to install Docker on your Jenkins server:

   ```bash
   curl -fsSL https://get.docker.com -o get-docker.sh
   sudo sh get-docker.sh
   sudo usermod -aG docker $USER
   ```

   After running these commands, you may need to restart your Jenkins server for the changes to take effect. Additionally, make sure that the Docker service is started and enabled to start on boot:

   ```bash
   sudo systemctl start docker
   sudo systemctl enable docker
   ```

   Verify the Docker installation by running the following command:

   ```bash
   docker --version
   ```

2. **Installing Kubernetes CLI (kubectl)**:

   The Kubernetes CLI (kubectl) is a separate binary that you can install on your Jenkins server to interact with Kubernetes clusters. You can install kubectl using the following steps:

   - Download the kubectl binary:

     ```bash
     sudo curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
     ```

   - Make the kubectl binary executable:

     ```bash
     sudo chmod +x ./kubectl
     ```

   - Move the kubectl binary to a directory in your PATH, such as `/usr/local/bin`:

     ```bash
     sudo mv ./kubectl /usr/local/bin/kubectl
     ```

   Verify the kubectl installation by running the following command:

   ```bash
   kubectl version --client
   ```

   This command should display the client version of kubectl.

After completing these steps, Docker and Kubernetes CLI (kubectl) should be installed and available on your Jenkins server. You can use them in your Jenkins jobs and pipelines to build and deploy Docker containers and interact with Kubernetes clusters.
