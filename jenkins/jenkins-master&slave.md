# Install Jenkins on Ubuntu & Setup Jenkins Master-Slave configuration

![jenkins](https://imgur.com/d4TaKyx.png)

To install Jenkins on Ubuntu and set up a master-slave configuration, you can follow these steps. Jenkins is a popular open-source automation server, and this guide will help you create a Jenkins master and connect one or more Jenkins slave nodes to distribute workloads.

### Installing Jenkins on Ubuntu:

1. **Update Package Index:**
   ```bash
   sudo apt update
   ```

2. **Install Java Development Kit (JDK):**
   Jenkins requires Java to run. You can install OpenJDK using the following command:
   ```bash
   sudo apt install openjdk-8-jdk
   ```

3. **Add Jenkins Repository Key:**
   ```bash
   wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
   ```

4. **Add Jenkins Repository to Your System:**
   ```bash
   sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
   ```

5. **Update Package Index Again:**
   ```bash
   sudo apt update
   ```

6. **Install Jenkins:**
   ```bash
   sudo apt install jenkins
   ```

7. **Start Jenkins Service:**
   ```bash
   sudo systemctl start jenkins
   ```

8. **Enable Jenkins to Start on Boot:**
   ```bash
   sudo systemctl enable jenkins
   ```

9. **Check Jenkins Service Status:**
   ```bash
   sudo systemctl status jenkins
   ```

   If Jenkins is running properly, you should see an active status.

10. **Configure Firewall (if necessary):**
    If you have UFW (Uncomplicated Firewall) enabled, you need to allow traffic on port 8080 (Jenkins default port).
    ```bash
    sudo ufw allow 8080
    ```

### Setting Up Jenkins Master-Slave Configuration:

#### Configure Jenkins Master:

1. **Access Jenkins Web Interface:**
   Open your web browser and navigate to `http://your_server_ip_or_domain:8080`. You will be prompted to enter the initial administrator password.

2. **Unlock Jenkins:**
   To retrieve the initial password, you can use the following command:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```

3. **Install Recommended Plugins:**
   Choose "Install suggested plugins" to install the recommended plugins.

4. **Create Admin User:** 
   Follow the instructions to create an administrator user for Jenkins.

5. **Set Jenkins URL:**
   After setting up the admin user, you'll be asked to set the Jenkins URL. Ensure it's correctly set based on your server's configuration.

6. **Finish Installation:**
   Once the setup is complete, you'll be redirected to the Jenkins dashboard.

#### Configure Jenkins Slave:

1. **Install Java Development Kit (JDK) on Slave Machine:**
   Follow the same steps as mentioned above to install Java on your slave machine.

2. **Enable SSH Access from Master to Slave:**
   Ensure that the master server can SSH into the slave without requiring a password. You might need to generate SSH keys and copy them to the slave.

3. **Add Slave Node in Jenkins:**
   - Go to Jenkins dashboard -> Manage Jenkins -> Manage Nodes and Clouds -> New Node.
   - Enter a name for your node and select "Permanent Agent".
   - Configure the remote root directory, labels, and other settings as per your requirement.
   - Under "Launch method", choose "Launch agent via SSH".
   - Enter the slave machine's SSH details (hostname, username, and private key).

4. **Save Configuration:**
   Save the configuration, and Jenkins will attempt to connect to the slave machine.

5. **Verify Connection:**
   After saving, Jenkins will attempt to connect to the slave machine. If the connection is successful, the node will appear in the list of nodes.

6. **Set Up Labels and Usage:**
   You can set labels on your slave nodes and configure Jenkins jobs to run on specific nodes based on these labels.

7. **Testing:**
   Create a Jenkins job and configure it to run on the slave node. Run the job to verify that the setup is working correctly.

That's it! You now have Jenkins installed on Ubuntu and a Master-Slave configuration set up.
