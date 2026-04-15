# Tami Studio - Production Deployment Guide

Complete guide for deploying Tami Studio to DigitalOcean using Docker and nginx.

## 📋 Prerequisites

- DigitalOcean account
- Domain name (optional but recommended)
- SSH client
- Git installed locally

## 🚀 Step-by-Step Deployment

### Step 1: Create a DigitalOcean Droplet

1. **Log in to DigitalOcean** and click "Create" → "Droplets"

2. **Choose Configuration:**
   - **Distribution:** Ubuntu 22.04 LTS (recommended)
   - **Plan:** Basic - $6/month (1 GB RAM, 1 vCPU, 25 GB SSD)
   - **Datacenter:** Choose closest to your target audience
   - **Authentication:** SSH key (recommended) or password
   - **Hostname:** `tami-studio-prod` or similar

3. **Click "Create Droplet"** and wait ~60 seconds

4. **Note your Droplet's IP address** (shown in the dashboard)

### Step 2: Initial Server Setup

1. **Connect to your droplet:**
   ```bash
   ssh root@YOUR_DROPLET_IP
   ```

2. **Update system packages:**
   ```bash
   apt update && apt upgrade -y
   ```

3. **Install Docker:**
   ```bash
   # Install dependencies
   apt install -y apt-transport-https ca-certificates curl software-properties-common
   
   # Add Docker's GPG key
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
   
   # Add Docker repository
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
   
   # Install Docker
   apt update
   apt install -y docker-ce docker-ce-cli containerd.io
   
   # Verify installation
   docker --version
   ```

4. **Install Docker Compose:**
   ```bash
   # Download latest version
   curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   
   # Make executable
   chmod +x /usr/local/bin/docker-compose
   
   # Verify installation
   docker-compose --version
   ```

5. **Configure firewall:**
   ```bash
   # Enable UFW
   ufw allow OpenSSH
   ufw allow 80/tcp
   ufw allow 443/tcp
   ufw --force enable
   
   # Check status
   ufw status
   ```

6. **Create deployment directory:**
   ```bash
   mkdir -p /opt/tami-studio
   cd /opt/tami-studio
   ```

### Step 3: Deploy the Application

**Option A: Deploy via Git (Recommended)**

1. **Install Git:**
   ```bash
   apt install -y git
   ```

2. **Clone your repository:**
   ```bash
   cd /opt
   git clone YOUR_REPOSITORY_URL tami-studio
   cd tami-studio
   ```

3. **Build and start containers:**
   ```bash
   docker-compose up -d --build
   ```

**Option B: Deploy via SCP (Manual Upload)**

1. **From your local machine, upload files:**
   ```bash
   # From your project directory
   scp -r * root@YOUR_DROPLET_IP:/opt/tami-studio/
   ```

2. **On the server, build and start:**
   ```bash
   cd /opt/tami-studio
   docker-compose up -d --build
   ```

### Step 4: Verify Deployment

1. **Check container status:**
   ```bash
   docker-compose ps
   ```
   
   Should show `tami-studio-web` as "Up"

2. **Check logs:**
   ```bash
   docker-compose logs -f
   ```
   
   Press `Ctrl+C` to exit

3. **Test locally on the server:**
   ```bash
   curl http://localhost:8080
   ```
   
   Should return HTML content

4. **Test from your browser:**
   ```
   http://YOUR_DROPLET_IP:8080
   ```

### Step 5: Configure Reverse Proxy (Production Setup)

For production with SSL, use nginx as a reverse proxy on the host.

1. **Install nginx on host:**
   ```bash
   apt install -y nginx
   ```

2. **Create nginx configuration:**
   ```bash
   nano /etc/nginx/sites-available/tami-studio
   ```

3. **Add this configuration:**
   ```nginx
   server {
       listen 80;
       server_name YOUR_DOMAIN.com www.YOUR_DOMAIN.com;
   
       location / {
           proxy_pass http://localhost:8080;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
           proxy_cache_bypass $http_upgrade;
       }
   }
   ```

4. **Enable the site:**
   ```bash
   ln -s /etc/nginx/sites-available/tami-studio /etc/nginx/sites-enabled/
   nginx -t
   systemctl restart nginx
   ```

### Step 6: Configure SSL with Let's Encrypt (Recommended)

1. **Install Certbot:**
   ```bash
   apt install -y certbot python3-certbot-nginx
   ```

2. **Get SSL certificate:**
   ```bash
   certbot --nginx -d YOUR_DOMAIN.com -d www.YOUR_DOMAIN.com
   ```
   
   Follow the prompts (provide email, agree to terms)

3. **Test auto-renewal:**
   ```bash
   certbot renew --dry-run
   ```

4. **Your site is now accessible at:**
   ```
   https://YOUR_DOMAIN.com
   ```

### Step 7: Set Up Auto-Start on Reboot

Docker Compose services with `restart: unless-stopped` will auto-start, but create a systemd service for extra reliability:

1. **Create systemd service:**
   ```bash
   nano /etc/systemd/system/tami-studio.service
   ```

2. **Add this content:**
   ```ini
   [Unit]
   Description=Tami Studio Docker Compose
   Requires=docker.service
   After=docker.service
   
   [Service]
   Type=oneshot
   RemainAfterExit=yes
   WorkingDirectory=/opt/tami-studio
   ExecStart=/usr/local/bin/docker-compose up -d
   ExecStop=/usr/local/bin/docker-compose down
   TimeoutStartSec=0
   
   [Install]
   WantedBy=multi-user.target
   ```

3. **Enable and start:**
   ```bash
   systemctl daemon-reload
   systemctl enable tami-studio
   systemctl start tami-studio
   ```

## 🔄 Updating the Application

### When you make changes to the code:

1. **Connect to your server:**
   ```bash
   ssh root@YOUR_DROPLET_IP
   cd /opt/tami-studio
   ```

2. **Pull latest changes (if using Git):**
   ```bash
   git pull
   ```
   
   **Or upload new files (if using SCP):**
   ```bash
   # From local machine
   scp -r * root@YOUR_DROPLET_IP:/opt/tami-studio/
   ```

3. **Rebuild and restart:**
   ```bash
   docker-compose down
   docker-compose up -d --build
   ```

4. **Clean up old images (optional):**
   ```bash
   docker image prune -f
   ```

## 🛠 Useful Commands

### Container Management

```bash
# View running containers
docker-compose ps

# View logs
docker-compose logs -f

# Restart service
docker-compose restart

# Stop service
docker-compose down

# Start service
docker-compose up -d

# Rebuild and restart
docker-compose up -d --build

# View resource usage
docker stats
```

### Debugging

```bash
# Enter container shell
docker exec -it tami-studio-web sh

# View nginx config inside container
docker exec tami-studio-web cat /etc/nginx/conf.d/default.conf

# Test nginx config
docker exec tami-studio-web nginx -t

# View container logs
docker logs tami-studio-web

# Check health status
docker inspect --format='{{.State.Health.Status}}' tami-studio-web
```

### System Monitoring

```bash
# Disk usage
df -h

# Memory usage
free -h

# Docker disk usage
docker system df
```

## 🔐 Security Best Practices

1. **Create a non-root user:**
   ```bash
   adduser deploy
   usermod -aG sudo deploy
   usermod -aG docker deploy
   ```

2. **Disable root SSH login:**
   ```bash
   nano /etc/ssh/sshd_config
   # Set: PermitRootLogin no
   systemctl restart sshd
   ```

3. **Set up automatic security updates:**
   ```bash
   apt install -y unattended-upgrades
   dpkg-reconfigure --priority=low unattended-upgrades
   ```

4. **Regular backups:**
   ```bash
   # Create backup script
   nano /root/backup.sh
   ```
   
   ```bash
   #!/bin/bash
   tar -czf /backups/tami-studio-$(date +%Y%m%d).tar.gz /opt/tami-studio
   find /backups -name "tami-studio-*.tar.gz" -mtime +7 -delete
   ```
   
   ```bash
   chmod +x /root/backup.sh
   # Add to crontab: 0 2 * * * /root/backup.sh
   ```

## 🚀 Future: Adding FastAPI Backend

When you're ready to add FastAPI:

1. **Update docker-compose.yml** to include a FastAPI service
2. **Modify nginx.conf** to proxy `/api` requests to FastAPI
3. **Keep the same network** so containers can communicate

Example addition to docker-compose.yml:
```yaml
  api:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: tami-studio-api
    restart: unless-stopped
    ports:
      - "8000:8000"
    networks:
      - tami-network
```

## 📊 Monitoring and Maintenance

### Regular tasks:

- **Weekly:** Check logs for errors
- **Monthly:** Update system packages and Docker
- **Quarterly:** Review and clean up old Docker images

### Useful monitoring commands:

```bash
# Check uptime
uptime

# Check logs for errors
docker-compose logs --tail=100 | grep -i error

# Check disk space
df -h
```

## 🆘 Troubleshooting

### Container won't start

```bash
# Check logs
docker-compose logs

# Check docker daemon
systemctl status docker

# Rebuild from scratch
docker-compose down -v
docker-compose up -d --build
```

### Can't access website

```bash
# Check if container is running
docker ps

# Check port binding
netstat -tulpn | grep 8080

# Check firewall
ufw status

# Check nginx (if using reverse proxy)
systemctl status nginx
nginx -t
```

### High memory usage

```bash
# Check resource usage
docker stats

# Adjust limits in docker-compose.yml
# Restart containers
docker-compose restart
```

## 📞 Support

For issues specific to:
- **DigitalOcean:** Check their documentation at docs.digitalocean.com
- **Docker:** docs.docker.com
- **nginx:** nginx.org/en/docs/

---

**Last Updated:** April 2026  
**Version:** 1.0.0
