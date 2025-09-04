# SonarQube Production Setup with Docker Compose, PostgreSQL & Nginx SSL

## Overview

This guide sets up a production-ready SonarQube server with PostgreSQL database using Docker Compose, following Linux Filesystem Hierarchy Standard (FHS) best practices, and configures Nginx reverse proxy with SSL certificate.

**What we're building:**
- SonarQube server for code quality analysis
- PostgreSQL database for persistent data storage  
- Nginx reverse proxy with SSL/HTTPS
- All following production best practices

**Why these choices:**
- **Docker Compose**: Easy container orchestration and management
- **PostgreSQL**: Reliable database for SonarQube data
- **Linux FHS**: Proper file organization (config in /etc, data in /var/lib, logs in /var/log)
- **Nginx + SSL**: Secure web access with domain name and HTTPS
- **Best Practices**: Data persistence, security, monitoring, proper permissions

---

## Prerequisites

- Fresh Ubuntu EC2 instance (or Ubuntu server)
- Domain name pointing to your server's public IP
- Root/sudo access
- SSH access to your server

---

## Step 0: Install Docker and Docker Compose

**For a fresh Ubuntu EC2 instance, first install Docker:**

```bash
# Update system packages
sudo apt update
sudo apt upgrade -y

# Install required packages
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package index
sudo apt update

# Install Docker
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Add your user to docker group (so you don't need sudo for docker commands)
sudo usermod -aG docker $USER

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Verify installation
docker --version
docker-compose --version
```

**After installation, log out and log back in (or restart your session) for group changes to take effect.**

**Test Docker installation:**
```bash
# Test docker (should work without sudo)
docker run hello-world

# If you get permission errors, restart your SSH session
```

**Why we need Docker:**
- Containerization for consistent deployment
- Easy management of SonarQube and PostgreSQL
- Isolation and security
- Easy backup and restore

---

## Step 1: Create Directory Structure

Create directories following Linux FHS standards:

```bash
# Application and orchestration files
sudo mkdir -p /opt/sonarqube-stack

# Persistent data (databases, application data)
sudo mkdir -p /var/lib/sonarqube/{data,extensions}
sudo mkdir -p /var/lib/postgresql/sonarqube

# Configuration files
sudo mkdir -p /etc/sonarqube

# Log files
sudo mkdir -p /var/log/sonarqube
sudo mkdir -p /var/log/postgresql
```

**Why this structure:**
- `/opt`: Application files (FHS standard for optional software)
- `/var/lib`: Variable data that persists (database files, app data)
- `/etc`: Configuration files (system standard)
- `/var/log`: Log files (system standard)

---

## Step 2: Set Correct Ownership and Permissions

```bash
# Application directory (your user can manage)
sudo chown -R $USER:$USER /opt/sonarqube-stack

# Data directories (container users need access)
sudo chown -R 1000:1000 /var/lib/sonarqube    # SonarQube runs as UID 1000
sudo chown -R 70:70 /var/lib/postgresql       # PostgreSQL runs as UID 70

# Configuration (root owned, readable by services)
sudo chown -R root:root /etc/sonarqube
sudo chmod 755 /etc/sonarqube

# Logs (services need write access)
sudo chown -R 1000:1000 /var/log/sonarqube
sudo chown -R 70:70 /var/log/postgresql
```

**Why specific UIDs:**
- Container processes run as specific user IDs for security
- Proper ownership prevents permission errors

---

## Step 3: Configure System Settings

SonarQube requires specific system configurations for Elasticsearch:

```bash
# Configure system limits for SonarQube/Elasticsearch
sudo tee -a /etc/sysctl.conf << EOF
vm.max_map_count=524288
fs.file-max=131072
EOF

# Apply immediately
sudo sysctl -p
```

**Why needed:**
- SonarQube uses Elasticsearch internally which requires these kernel parameters

---

## Step 4: Create Environment Configuration

```bash
# Configuration file in proper location
sudo nano /etc/sonarqube/.env
```

Add this content (change passwords to strong ones):

```env
# Database Configuration
POSTGRES_DB=sonarqube
POSTGRES_USER=sonarqube
POSTGRES_PASSWORD=YourStrongPassword123!

# SonarQube Database Connection
SONARQUBE_JDBC_USERNAME=sonarqube
SONARQUBE_JDBC_PASSWORD=YourStrongPassword123!
SONARQUBE_JDBC_URL=jdbc:postgresql://postgres:5432/sonarqube

# Security
POSTGRES_HOST_AUTH_METHOD=md5
```

Secure the file:
```bash
sudo chmod 600 /etc/sonarqube/.env
sudo chown $USER:$USER /etc/sonarqube/.env

```

**Why separate env file:**
- Keeps secrets separate from code
- Follows security best practices
- Easy to manage different environments

---

## Step 5: Create Docker Compose Configuration

```bash
cd /opt/sonarqube-stack

# Create symbolic link to maintain best practices (config in /etc)
ln -s /etc/sonarqube/.env .env

nano docker-compose.yml
```

Choose the configuration that best fits your environment and requirements:

### **Quick Comparison:**

| Feature | Basic Setup | Production Setup |
|---------|-------------|------------------|
| **Complexity** | Simple | Slightly more complex |
| **Resource Control** | System managed | Explicit limits & reservations |
| **Memory Safety** | Relies on system limits | Guaranteed memory protection |
| **Best For** | Dedicated servers, testing | Production, shared servers, small instances |
| **Stability** | Good | Excellent |
| **Performance** | Full system resources | Controlled resource allocation |

---

### **Option 1: Basic Setup (Recommended for dedicated servers)**

**Use this version if:**
- You want the simplest setup
- Your server has dedicated resources (t3.medium+ with no other services)
- You prefer minimal configuration

```yaml
services:
  postgres:
    image: postgres:15-alpine
    container_name: sonarqube-postgres
    restart: unless-stopped
    env_file:
      - .env
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_HOST_AUTH_METHOD: ${POSTGRES_HOST_AUTH_METHOD}
    volumes:
      # Data persistence in standard location
      - /var/lib/postgresql/sonarqube:/var/lib/postgresql/data
      # Logs in standard location
      - /var/log/postgresql:/var/log/postgresql
    networks:
      - sonarnet
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 30s
      timeout: 10s
      retries: 3

  sonarqube:
    image: sonarqube:10-community
    container_name: sonarqube-app
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    env_file:
      - .env
    environment:
      SONAR_JDBC_URL: ${SONARQUBE_JDBC_URL}
      SONAR_JDBC_USERNAME: ${SONARQUBE_JDBC_USERNAME}
      SONAR_JDBC_PASSWORD: ${SONARQUBE_JDBC_PASSWORD}
    volumes:
      # Data persistence in standard locations
      - /var/lib/sonarqube/data:/opt/sonarqube/data
      - /var/lib/sonarqube/extensions:/opt/sonarqube/extensions
      # Logs in standard location
      - /var/log/sonarqube:/opt/sonarqube/logs
    ports:
      - "9000:9000"
    networks:
      - sonarnet
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9000/api/system/status || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 90s

networks:
  sonarnet:
    driver: bridge
```

**Key features:**
- **Health checks**: Automatic restart if services fail
- **Proper volumes**: Data persists between container restarts
- **Network isolation**: Services communicate securely
- **Dependency management**: SonarQube waits for database to be ready
- **Simple configuration**: No resource constraints, uses all available system resources

---

### **Option 2: Production Setup with Resource Limits (Recommended for production)**

**Use this version if:**
- This is a production environment
- You're running on shared instances or small EC2 instances
- You want maximum stability and predictability
- You're running multiple services on the same server
- You want to prevent out-of-memory issues

```yaml
services:
  postgres:
    image: postgres:15-alpine
    container_name: sonarqube-postgres
    restart: unless-stopped
    env_file:
      - .env
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_HOST_AUTH_METHOD: ${POSTGRES_HOST_AUTH_METHOD}
    volumes:
      # Data persistence in standard location
      - /var/lib/postgresql/sonarqube:/var/lib/postgresql/data
      # Logs in standard location
      - /var/log/postgresql:/var/log/postgresql
    networks:
      - sonarnet
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 30s
      timeout: 10s
      retries: 3
    # Resource limits to prevent PostgreSQL from consuming excessive memory
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1.0'
        reservations:
          memory: 256M
          cpus: '0.5'

  sonarqube:
    image: sonarqube:10-community
    container_name: sonarqube-app
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    env_file:
      - .env
    environment:
      SONAR_JDBC_URL: ${SONARQUBE_JDBC_URL}
      SONAR_JDBC_USERNAME: ${SONARQUBE_JDBC_USERNAME}
      SONAR_JDBC_PASSWORD: ${SONARQUBE_JDBC_PASSWORD}
    volumes:
      # Data persistence in standard locations
      - /var/lib/sonarqube/data:/opt/sonarqube/data
      - /var/lib/sonarqube/extensions:/opt/sonarqube/extensions
      # Logs in standard location
      - /var/log/sonarqube:/opt/sonarqube/logs
    ports:
      - "9000:9000"
    networks:
      - sonarnet
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9000/api/system/status || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 90s
    # Resource limits to prevent SonarQube from consuming all server memory
    deploy:
      resources:
        limits:
          memory: 3G
          cpus: '2.0'
        reservations:
          memory: 2G
          cpus: '1.0'

networks:
  sonarnet:
    driver: bridge
```

**Key features:**
- **All Basic Setup features** +
- **Memory limits**: Prevents services from consuming all server memory
- **CPU limits**: Controls CPU usage for better system stability
- **Resource reservations**: Guarantees minimum resources for each service
- **Production stability**: Prevents one service from starving others
- **Perfect for small instances**: Works great with swap memory configuration
- **Predictable performance**: Resources are controlled and allocated properly

---

### **Which Setup Should You Choose?**

**Choose Basic Setup if:**
- You have a dedicated server (t3.medium or larger)
- Only running SonarQube (no other major services)
- You want the simplest possible configuration
- You're in a testing/development environment

**Choose Production Setup if:**
- You're deploying to production
- Using small EC2 instances (t3.small, t3.micro)
- Running other services on the same server
- You configured swap memory (Step 0.5)
- You want maximum stability and predictability

### **Pro Tip:**
You can always start with the Basic Setup and migrate to Production Setup later by simply updating your docker-compose.yml file and running `docker compose up -d` to apply the changes.
## Step 6: Start the Services

```bash
cd /opt/sonarqube-stack

# Start services
docker compose up -d

# Check status
docker compose ps

# View startup logs
docker compose logs -f
```

Wait 2-3 minutes for SonarQube to fully initialize.

**Verify everything works:**
```bash
# Test database connection
docker exec sonarqube-postgres psql -U sonarqube -d sonarqube -c "SELECT version();"

# Test SonarQube is responding
curl http://localhost:9000
```

---

## Step 7: Configure Nginx Reverse Proxy

Install and configure Nginx:

```bash
# Install nginx
sudo apt update
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx

# Clean up default configuration
sudo rm -f /etc/nginx/sites-enabled/default
```

Create SonarQube site configuration:
```bash
sudo nano /etc/nginx/sites-available/sonarqube
```

Add this configuration:
```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable the site:
```bash
sudo ln -s /etc/nginx/sites-available/sonarqube /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

**Why Nginx:**
- Provides domain name access
- Better performance for static files
- SSL termination
- Security headers

---

## Step 8: Configure SSL Certificate

Install certbot and get SSL certificate:

```bash
# Install certbot
sudo apt install certbot python3-certbot-nginx -y

# Get SSL certificate (replace with your domain)
sudo certbot --nginx -d your-domain.com
```

**Important:** Your domain must point to your server's public IP address before running certbot.

Follow the prompts:
- Enter your email address
- Agree to terms of service  
- Choose whether to share email (optional)

**Certbot automatically:**
- Gets free SSL certificate from Let's Encrypt
- Updates nginx configuration for HTTPS
- Sets up HTTP to HTTPS redirect
- Configures automatic renewal

---

## Step 9: Configure Firewall (Security)

```bash
# Allow HTTP and HTTPS traffic
sudo ufw allow 'Nginx Full'
sudo ufw allow ssh
sudo ufw --force enable

# Check firewall status
sudo ufw status
```

**Why firewall:**
- Blocks unwanted traffic
- Only allows necessary ports (80, 443, 22)

---

## Step 10: Final Verification

```bash
# Test nginx configuration
sudo nginx -t

# Check all services are running
sudo systemctl status nginx
docker compose ps

# Test domain access
curl -I https://your-domain.com
```

**Access SonarQube:**
- Open browser to `https://your-domain.com`
- Default login: admin/admin
- Change password when prompted

---

## Maintenance Commands

### Managing Services
```bash
# Stop/start SonarQube stack
cd /opt/sonarqube-stack
docker compose down
docker compose up -d

# Restart nginx
sudo systemctl restart nginx

# View logs
docker compose logs -f sonarqube
docker compose logs -f postgres
sudo tail -f /var/log/nginx/error.log
```

### Backup Commands
```bash
# Backup database
docker exec sonarqube-postgres pg_dump -U sonarqube sonarqube > sonarqube_backup.sql

# Backup application data
tar -czf sonarqube_data_backup.tar.gz /var/lib/sonarqube/
```

### SSL Certificate Renewal
```bash
# Test renewal (automatic renewal is configured)
sudo certbot renew --dry-run

# Check certificate expiry
sudo certbot certificates
```

---

## Troubleshooting

### Common Issues

**SonarQube won't start:**
```bash
# Check logs
docker compose logs sonarqube

# Common fix: permission issues
sudo chown -R 1000:1000 /var/lib/sonarqube
sudo chown -R 1000:1000 /var/log/sonarqube
```

**Database connection issues:**
```bash
# Check postgres logs
docker compose logs postgres

# Test database connectivity
docker exec sonarqube-postgres psql -U sonarqube -d sonarqube -c "\l"
```

**Nginx issues:**
```bash
# Check nginx configuration
sudo nginx -t

# Check nginx logs
sudo tail -f /var/log/nginx/error.log

# Test nginx is serving correctly
curl -I http://localhost
```

**SSL certificate issues:**
```bash
# Check certificate status
sudo certbot certificates

# Renew certificate manually
sudo certbot renew

# Check nginx SSL configuration
sudo nginx -t
```

---

## Security Best Practices Applied

1. **File Organization**: Follows Linux FHS standards
2. **Permissions**: Proper file ownership and permissions
3. **Secrets Management**: Environment variables in protected files
4. **Network Security**: Docker network isolation
5. **SSL/HTTPS**: Encrypted communication
6. **Firewall**: Only necessary ports open
7. **Auto-restart**: Services restart on failure
8. **Health Checks**: Automatic failure detection

---

## Directory Structure Summary

```
/opt/sonarqube-stack/          # Application files
├── docker-compose.yml         # Container orchestration
└── .env -> /etc/sonarqube/.env # Symlink to config

/etc/sonarqube/                # Configuration
└── .env                       # Environment variables

/var/lib/sonarqube/            # Application data
├── data/                      # SonarQube data
└── extensions/                # SonarQube plugins

/var/lib/postgresql/           # Database data
└── sonarqube/                 # PostgreSQL data

/var/log/                      # Log files
├── sonarqube/                 # SonarQube logs
├── postgresql/                # Database logs
└── nginx/                     # Web server logs
```

This setup provides a production-ready SonarQube installation with proper security, data persistence, and ease of maintenance.
