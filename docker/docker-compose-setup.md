# **Docker Compose Setup**

![docker-compose](https://quintagroup.com/cms/technology/Images/docker-compose-button.jpg)

**Docker Compose** is a tool that allows you to define and run multi-container Docker applications using a single YAML file. Here's how to install and set it up on different platforms.

---

## **1. Docker Compose Installation**

### **1.1. For Windows and macOS**

Docker Compose comes pre-installed with Docker Desktop. If Docker Desktop is already installed, Docker Compose is included.

1. To verify the installation, open **PowerShell** or **Terminal** and run:

   ```bash
   docker-compose --version
   ```

   If Docker Compose is installed, you will see the version.

### **1.2. For Linux**

#### **Step 1: Install Docker Compose**

1. Download the latest version of Docker Compose by running the following commands:

   ```bash
   sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   ```

2. Apply executable permissions to the binary:

   ```bash
   sudo chmod +x /usr/local/bin/docker-compose
   ```

3. Verify the installation:

   ```bash
   docker-compose --version
   ```

#### **Optional: Install Docker Compose from Package Manager (Alternative Method)**

For some Linux distributions, you can install Docker Compose from the official package manager:

   ```bash
   sudo apt-get install docker-compose
   ```

---

## **2. Writing a Docker Compose YAML File**

Docker Compose uses a YAML file to define services, networks, and volumes. Here's an example of a basic `docker-compose.yml` file that sets up a multi-container application with a web server and a database.

### **Example `docker-compose.yml`**

```yaml
version: '3'
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
    networks:
      - webnet

  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: exampledb
      MYSQL_USER: user
      MYSQL_PASSWORD: password
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - webnet

volumes:
  db_data:

networks:
  webnet:
```

This `docker-compose.yml` defines two services:

1. **web**: Uses the official `nginx` image, maps port 8080 on the host to port 80 on the container, and mounts a local directory to serve web content.
2. **db**: Uses the official `mysql:5.7` image, sets up environment variables for the database configuration, and mounts a volume to persist data.

---

## **3. Running a Docker Compose Application**

### **Step 1: Navigate to Your Project Directory**

In the terminal, navigate to the directory containing your `docker-compose.yml` file:

```bash
cd /path/to/your/project
```

### **Step 2: Start the Containers**

Run the following command to build and start the containers:

```bash
docker-compose up
```

This command will:

- Pull the necessary images from Docker Hub (if they aren't already available locally).
- Build the services and start the containers.

To start the containers in the background (detached mode), use:

```bash
docker-compose up -d
```

### **Step 3: Verify Services**

To verify that the containers are running, use:

```bash
docker-compose ps
```

This will list the containers, showing their status and port mappings.

### **Step 4: Stopping and Removing Containers**

To stop the services, run:

```bash
docker-compose stop
```

To stop and remove the containers, networks, and volumes created by `docker-compose`, run:

```bash
docker-compose down
```

---

## **4. Scaling Services**

Docker Compose allows you to scale services easily. For instance, to scale the **web** service to 3 instances, run:

```bash
docker-compose up --scale web=3 -d
```

This will create 3 instances of the **web** service, distributing traffic among them.

---

## **5. Docker Compose Commands**

Here are some useful Docker Compose commands:

- **Build or rebuild services:**

   ```bash
   docker-compose build
   ```

- **Check container logs:**

   ```bash
   docker-compose logs
   ```

- **View running services:**

   ```bash
   docker-compose ps
   ```

- **Restart services:**

   ```bash
   docker-compose restart
   ```

- **Remove stopped containers:**

   ```bash
   docker-compose rm
   ```

- **Execute commands in a running service:**

   ```bash
   docker-compose exec <service_name> <command>
   ```

   For example, to access the shell in a running **web** container:

   ```bash
   docker-compose exec web sh
   ```

---

## **6. Docker Compose Best Practices**

- **Use environment variables:** Store sensitive data and configuration variables in a `.env` file to keep the `docker-compose.yml` clean.
- **Use volumes for data persistence:** Ensure that data such as database files are persisted between container restarts using volumes.
- **Versioning:** Stick to a specific Compose file version (e.g., `version: '3'`) to ensure compatibility across environments.

---

This guide will help you install Docker Compose and set up multi-container Docker applications easily. By using `docker-compose.yml`, you can manage complex environments with multiple services, containers, and networks.

## **Author by:**

![test](https://imgur.com/2j6Aoyl.png)

> [!NOTE]
> **Join Our** [Telegram Community](https://t.me/prodevopsguy) // [Follow me](https://github.com/NotHarshhaa) **for more DevOps & Cloud content.**
