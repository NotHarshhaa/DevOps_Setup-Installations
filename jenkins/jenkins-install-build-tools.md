# Build tools installation and Setup on Jenkins

Here's a detailed process for installing Git, Maven, Gradle, and npm on a Jenkins server:

1. **Installing Git**:
   - Jenkins typically comes with Git integration out of the box. However, if Git is not installed on your Jenkins server, you can install it using the package manager of your operating system.
     - For Ubuntu:
       ```bash
       sudo apt update
       sudo apt install git
       ```
     - For CentOS/RHEL:
       ```bash
       sudo yum install git
       ```

2. **Installing Maven**:
   - You can install Maven on your Jenkins server by downloading and extracting the Maven binary distribution.
     ```bash
     # Go to the /opt directory
     cd /opt
     
     # Download Maven
     sudo wget https://apache.mirror.digitalpacific.com.au/maven/maven-3/3.8.4/binaries/apache-maven-3.8.4-bin.tar.gz
     
     # Extract the downloaded file
     sudo tar -xf apache-maven-3.8.4-bin.tar.gz
     
     # Rename the extracted directory
     sudo mv apache-maven-3.8.4 maven
     
     # Add Maven bin directory to PATH
     echo 'export PATH=$PATH:/opt/maven/bin' | sudo tee -a /etc/profile.d/maven.sh
     
     # Reload the shell configuration
     source /etc/profile.d/maven.sh
     ```

3. **Installing Gradle**:
   - Similarly, you can install Gradle by downloading and extracting the Gradle binary distribution.
     ```bash
     # Go to the /opt directory
     cd /opt
     
     # Download Gradle
     sudo wget https://services.gradle.org/distributions/gradle-7.4-bin.zip
     
     # Extract the downloaded file
     sudo unzip gradle-7.4-bin.zip
     
     # Rename the extracted directory
     sudo mv gradle-7.4 gradle
     
     # Add Gradle bin directory to PATH
     echo 'export PATH=$PATH:/opt/gradle/bin' | sudo tee -a /etc/profile.d/gradle.sh
     
     # Reload the shell configuration
     source /etc/profile.d/gradle.sh
     ```

4. **Installing npm**:
   - To install npm, you'll first need to install Node.js on your Jenkins server, as npm comes bundled with Node.js.
     - For Ubuntu:
       ```bash
       sudo apt update
       sudo apt install nodejs npm
       ```
     - For CentOS/RHEL:
       ```bash
       sudo yum install epel-release
       sudo yum install nodejs npm
       ```

Once you've completed these steps, Git, Maven, Gradle, and npm should be installed and available on your Jenkins server. You can verify the installations by running commands like `git --version`, `mvn --version`, `gradle --version`, and `npm --version` to ensure that the tools are properly installed and configured.
