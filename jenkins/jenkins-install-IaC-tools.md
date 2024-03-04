# Setup & Installation for Infrastructure as Code (IaC) Tools on Jenkins

Certainly! Below are the detailed steps for installing Terraform, AWS CLI, Azure CLI, and Google Cloud SDK (gcloud) on a Jenkins server:

1. **Installing Terraform**:

   Terraform can be installed by downloading the binary distribution and placing it in a directory that is included in the system's PATH.

   ```bash
   # Download Terraform binary
   wget https://releases.hashicorp.com/terraform/1.0.10/terraform_1.0.10_linux_amd64.zip

   # Extract the downloaded file
   unzip terraform_1.0.10_linux_amd64.zip

   # Move Terraform binary to a directory included in the PATH
   sudo mv terraform /usr/local/bin/
   ```

   Verify the Terraform installation by running the following command:

   ```bash
   terraform version
   ```

2. **Installing AWS CLI**:

   The AWS CLI can be installed using the Python package manager, pip.

   ```bash
   # Install pip if not already installed
   sudo apt update
   sudo apt install python3-pip

   # Install AWS CLI using pip
   sudo pip3 install awscli
   ```

   Verify the AWS CLI installation by running the following command:

   ```bash
   aws --version
   ```

3. **Installing Azure CLI**:

   The Azure CLI can be installed using the package manager for the respective Linux distribution.

   - For Ubuntu:

     ```bash
     curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
     ```

   - For CentOS/RHEL:

     ```bash
     sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
     sudo sh -c 'echo -e "[azure-cli]
     name=Azure CLI
     baseurl=https://packages.microsoft.com/yumrepos/azure-cli
     enabled=1
     gpgcheck=1
     gpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/azure-cli.repo'
     sudo yum install azure-cli
     ```

   Verify the Azure CLI installation by running the following command:

   ```bash
   az --version
   ```

4. **Installing Google Cloud SDK (gcloud)**:

   The Google Cloud SDK provides the gcloud command-line tool, which can be installed using a package manager or by downloading the binary distribution.

   ```bash
   # Add the Google Cloud SDK distribution URI as a package source
   export CLOUD_SDK_REPO="cloud-sdk-$(lsb_release -c -s)"
   echo "deb http://packages.cloud.google.com/apt $CLOUD_SDK_REPO main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

   # Import the Google Cloud Platform public key
   curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

   # Update the package list and install the Cloud SDK
   sudo apt update && sudo apt install google-cloud-sdk
   ```

   Verify the Google Cloud SDK installation by running the following command:

   ```bash
   gcloud version
   ```

After completing these steps, Terraform, AWS CLI, Azure CLI, and Google Cloud SDK should be installed and available on your Jenkins server. You can use them in your Jenkins jobs and pipelines to manage infrastructure on various cloud platforms.
