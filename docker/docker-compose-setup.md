# Docker Compose Setup Guide

![docker-compose](https://quintagroup.com/cms/technology/Images/docker-compose-button.jpg)

Docker Compose is a tool for defining and running multi-container Docker applications using a single YAML file. This guide covers installation and setup for all platforms.

## Important Note: Docker Compose V2

Since Docker Desktop 3.4.0 and Docker Engine 20.10.0, Docker Compose V2 is included as a plugin. The new syntax is `docker compose` (without hyphen) instead of `docker-compose`. This guide uses the modern V2 syntax.

## 1. Docker Compose Installation

### 1.1. For Windows and macOS

Docker Compose V2 comes pre-installed with Docker Desktop. If Docker Desktop is installed, you already have Docker Compose.

Verify the installation:

```bash
docker compose version
```

### 1.2. For Linux (Ubuntu/Debian)

Docker Compose V2 is included with the Docker Engine installation. If you installed Docker using the official repository, you already have it.

Verify the installation:

```bash
docker compose version
```

If you need to install it separately:

```bash
# Download the latest version
sudo curl -SL https://github.com/docker/compose/releases/download/v2.24.5/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose

# Apply executable permissions
sudo chmod +x /usr/local/bin/docker-compose

# Verify
docker-compose --version
```

### 1.3. For Linux (CentOS/RHEL)

Docker Compose V2 is included with Docker Engine installation. Verify:

```bash
docker compose version
```

### 1.4. Legacy Docker Compose V1 (Deprecated)

If you need the standalone V1 version (not recommended):

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

## 2. Writing a Docker Compose YAML File

Docker Compose uses a YAML file to define services, networks, and volumes. Modern Compose files (V2) don't require a version declaration.

### 2.1. Basic Example: Web Server + Database

```yaml
# docker-compose.yml
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
    networks:
      - webnet
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: exampledb
      MYSQL_USER: user
      MYSQL_PASSWORD: password
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - webnet
    restart: unless-stopped

volumes:
  db_data:
    driver: local

networks:
  webnet:
    driver: bridge
```

### 2.2. Example with Environment Variables

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:latest
    build: .
    environment:
      - DATABASE_URL=mysql://user:password@db:3306/exampledb
      - REDIS_URL=redis://redis:6379
    env_file:
      - .env
    ports:
      - "3000:3000"
    depends_on:
      - db
      - redis

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
    volumes:
      - db_data:/var/lib/mysql

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

volumes:
  db_data:
```

### 2.3. Example with Custom Network

```yaml
services:
  frontend:
    image: react-app:latest
    ports:
      - "80:3000"
    networks:
      - frontend-network

  backend:
    image: api-server:latest
    ports:
      - "8080:8080"
    networks:
      - frontend-network
      - backend-network

  database:
    image: postgres:15
    networks:
      - backend-network
    volumes:
      - postgres_data:/var/lib/postgresql/data

networks:
  frontend-network:
    driver: bridge
  backend-network:
    driver: bridge
    internal: true

volumes:
  postgres_data:
```

## 3. Running Docker Compose Applications

### 3.1. Start Services

```bash
# Start services in foreground
docker compose up

# Start services in background (detached mode)
docker compose up -d

# Start specific service
docker compose up web

# Force rebuild of images
docker compose up -d --build
```

### 3.2. Manage Services

```bash
# List running services
docker compose ps

# View logs
docker compose logs

# Follow logs in real-time
docker compose logs -f

# View logs for specific service
docker compose logs -f web

# Stop services
docker compose stop

# Start stopped services
docker compose start

# Restart services
docker compose restart

# Restart specific service
docker compose restart web
```

### 3.3. Stop and Remove

```bash
# Stop and remove containers, networks
docker compose down

# Stop and remove containers, networks, and volumes
docker compose down -v

# Remove images as well
docker compose down --rmi all
```

## 4. Scaling Services

```bash
# Scale a service to multiple instances
docker compose up -d --scale web=3

# Note: You must remove port conflicts when scaling
# Use a load balancer or remove fixed ports
```

Example with scaling support:

```yaml
services:
  web:
    image: nginx:latest
    # Don't specify ports when scaling
    # or use a load balancer
    deploy:
      replicas: 3
```

## 5. Common Docker Compose Commands

### Build Commands

```bash
# Build services
docker compose build

# Build specific service
docker compose build web

# Build with no cache
docker compose build --no-cache

# Build and push images
docker compose build --push
```

### Execution Commands

```bash
# Execute command in running service
docker compose exec web bash

# Execute one-time command
docker compose run web python script.py

# Run command in new container
docker compose run --rm web npm install
```

### Maintenance Commands

```bash
# View resource usage
docker compose top

# View service logs since specific time
docker compose logs --since 1h

# Pause services
docker compose pause

# Unpause services
docker compose unpause

# Remove stopped containers
docker compose rm

# Remove all containers (including running)
docker compose rm -f
```

### Configuration Commands

```bash
# Validate compose file
docker compose config

# View resolved configuration
docker compose config --resolve-image-digests

# Create services from compose file (without starting)
docker compose create
```

## 6. Using Environment Variables

### 6.1. Create .env File

```env
# .env
MYSQL_ROOT_PASSWORD=securepassword
MYSQL_DATABASE=myapp
MYSQL_USER=dbuser
MYSQL_PASSWORD=dbpass
REDIS_HOST=redis
REDIS_PORT=6379
```

### 6.2. Reference in docker-compose.yml

```yaml
services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
```

### 6.3. Override Environment Variables

```bash
# Override specific variable
MYSQL_ROOT_PASSWORD=newpassword docker compose up

# Use different env file
docker compose --env-file .env.prod up
```

## 7. Multi-Stage Builds

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    image: myapp:latest
```

## 8. Health Checks

```yaml
services:
  web:
    image: nginx:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  app:
    image: myapp:latest
    depends_on:
      web:
        condition: service_healthy
```

## 9. Docker Compose Best Practices

- **Use environment variables**: Store sensitive data in `.env` files or Docker secrets
- **Use volumes for data persistence**: Ensure database data persists between container restarts
- **Specify resource limits**: Prevent containers from consuming all system resources
- **Use specific image versions**: Avoid `latest` tag for production
- **Organize with multiple compose files**: Use `docker-compose.override.yml` for local development
- **Use health checks**: Ensure services are healthy before dependent services start
- **Implement proper restart policies**: Use `restart: unless-stopped` for production
- **Label your resources**: Add labels for better organization and management

### Example with Best Practices

```yaml
services:
  web:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./html:/usr/share/nginx/html:ro
    networks:
      - frontend
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    labels:
      - "com.example.description=Web server"
      - "com.example.department=IT"

networks:
  frontend:
    driver: bridge
```

## 10. Troubleshooting

### Issue: Port Already in Use

```bash
# Find what's using the port
sudo lsof -i :80

# Change port in docker-compose.yml
# Or stop the conflicting service
```

### Issue: Container Won't Start

```bash
# Check logs
docker compose logs

# Check configuration
docker compose config

# Rebuild without cache
docker compose build --no-cache
```

### Issue: Volume Permission Issues

```bash
# Fix ownership of mounted volumes
sudo chown -R $USER:$USER ./data

# Or specify user in compose file
user: "${UID}:${GID}"
```

## 11. Additional Resources

- [Official Docker Compose Documentation](https://docs.docker.com/compose/)
- [Compose File Reference](https://docs.docker.com/compose/compose-file/)
- [Docker Compose Samples](https://github.com/docker/awesome-compose)

## Author by:

![test](https://imgur.com/2j6Aoyl.png)

> [!NOTE]
> **Join Our** [Telegram Community](https://t.me/prodevopsguy) // [Follow me](https://github.com/NotHarshhaa) **for more DevOps & Cloud content.**
