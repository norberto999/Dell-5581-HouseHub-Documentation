# Dell 5581 - HouseHub Development Server Documentation

## Overview

The Dell 5581 (running Ubuntu 24.04.4 LTS) is configured as a **Secondary/Staging/CI Runner** device for the HouseHub ecosystem. This machine serves as:

- **Staging/Testing Environment**: Run integration tests and staging deployments
- **Build Runner**: Execute longer builds and Docker image creation
- **CI/CD Infrastructure**: Host GitHub Actions self-hosted runners or GitLab runners
- **Backup Development Server**: Cross-browser testing and compatibility verification
- **Supporting Services**: Internal Docker registry, PostgreSQL replicas, artifact storage

## Hardware Specifications

To verify hardware capabilities, run these commands:

```bash
lscpu              # CPU information
free -h            # RAM available
lsblk -o NAME,SIZE,MOUNTPOINT  # Storage layout
```

**Expected Specs** (as secondary machine):
- Multiple CPU cores (typically 4-8 for staging)
- 8GB+ RAM recommended
- SSD or HDD for storage

## Base Software Installation

Run these commands on the 5581 machine to bootstrap the environment:

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y build-essential git curl wget unzip htop \
  python3 python3-venv python3-pip openjdk-17-jdk

# Install VS Code (optional for remote editing)
sudo snap install --classic code

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Install Docker Compose plugin (modern)
sudo apt install -y docker-compose-plugin

# Log out and back in so your user can run docker without sudo
logout
```

## Git & SSH Configuration

Set up Git configuration and SSH keys for GitHub access:

```bash
# Configure Git user
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Generate SSH key for GitHub/GitLab
ssh-keygen -t ed25519 -C "you@example.com"

# Copy public key and add to GitHub/GitLab
cat ~/.ssh/id_ed25519.pub
```

## Project Structure

Organize projects on the 5581 machine under `/srv/projects`:

```
/srv/projects/
├── app-frontend/
├── app-api/
├── app-worker/
├── app-admin/
├── infrastructure/      # docker-compose, scripts
├── dotfiles/            # Configuration backups
└── backups/             # Database and volume backups
```

## Containerization & Docker Compose

The 5581 runs containerized applications for consistency across environments.

### Example docker-compose.yml

Place in `/srv/projects/app-stack/docker-compose.yml`:

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: devdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

  api:
    build: ./app-api
    command: npm run dev
    volumes:
      - ./app-api:/usr/src/app
    ports:
      - "4000:4000"
    environment:
      DATABASE_URL: postgres://dev:dev@postgres:5432/devdb
      NODE_ENV: staging
    depends_on:
      - postgres
    restart: unless-stopped

  frontend:
    build: ./app-frontend
    command: npm run dev
    volumes:
      - ./app-frontend:/usr/src/app
    ports:
      - "3000:3000"
    depends_on:
      - api
    restart: unless-stopped

  worker:
    build: ./app-worker
    command: npm run start
    volumes:
      - ./app-worker:/usr/src/app
    environment:
      DATABASE_URL: postgres://dev:dev@postgres:5432/devdb
    depends_on:
      - postgres
    restart: unless-stopped

volumes:
  pgdata:
```

### Starting Services

```bash
cd /srv/projects/app-stack
docker compose up --build
```

## Systemd Service for Auto-Start

Create a systemd service to automatically start the app stack on boot:

```bash
sudo nano /etc/systemd/system/app-stack.service
```

Add the following content:

```ini
[Unit]
Description=HouseHub App Stack (Staging)
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/srv/projects/app-stack
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
User=norberto999
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable the service:

```bash
sudo systemctl enable --now app-stack.service
sudo systemctl status app-stack.service
```

## CI/CD - Self-Hosted GitHub Actions Runner

Run GitHub Actions workflows on the 5581 for build and test operations:

### Download and Configure Runner

```bash
# Create runner directory
mkdir -p ~/actions-runner && cd ~/actions-runner

# Download latest runner
curl -o actions-runner-linux-x64.tar.gz \
  -L https://github.com/actions/runner/releases/download/v2.310.3/actions-runner-linux-x64-2.310.3.tar.gz

# Extract
tar xzf ./actions-runner-linux-x64.tar.gz

# Configure runner (requires GitHub personal access token)
./config.sh --url https://github.com/norberto999/YOUR_REPO --token YOUR_TOKEN --labels hp-staging,runner-5581

# Install and start as service
./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
```

### Use in GitHub Actions

In `.github/workflows/your-workflow.yml`:

```yaml
jobs:
  build:
    runs-on: [self-hosted, hp-staging]
    steps:
      - uses: actions/checkout@v3
      - name: Run tests
        run: npm test
      - name: Build
        run: npm run build
```

## Backups & Data Protection

### PostgreSQL Backup Script

Create `/usr/local/bin/pg_backup.sh`:

```bash
#!/bin/bash

BACKUP_DIR="/srv/backups"
DATE=$(date +%F)

mkdir -p $BACKUP_DIR

# Dump all PostgreSQL databases
docker exec -t app-stack-postgres-1 pg_dumpall -c -U dev > \
  $BACKUP_DIR/pg_backup_$DATE.sql

echo "Backup completed: $BACKUP_DIR/pg_backup_$DATE.sql"
```

Make executable:

```bash
sudo chmod +x /usr/local/bin/pg_backup.sh
```

### Automated Daily Backup Cron

```bash
crontab -e
```

Add:

```
0 2 * * * /usr/local/bin/pg_backup.sh >> /var/log/pg_backup.log 2>&1
```

## Firewall & Security

### UFW Configuration

```bash
# Install UFW (if not already installed)
sudo apt install ufw

# Allow SSH access
sudo ufw allow OpenSSH

# Allow application ports
sudo ufw allow 3000/tcp      # Frontend
sudo ufw allow 4000/tcp      # API
sudo ufw allow 5432/tcp      # PostgreSQL (if external access needed)

# Enable firewall
sudo ufw enable
sudo ufw status
```

### Automatic Security Updates

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

## Environment Variables

Create `.env.template` in each project directory:

```
NODE_ENV=staging
DATABASE_URL=postgres://dev:dev@postgres:5432/devdb
API_PORT=4000
FRONTEND_PORT=3000
LOG_LEVEL=info
```

Actual `.env` file (never committed):

```bash
cp .env.template .env
# Edit with actual values
nano .env
```

## Monitoring & Logs

### View Docker Logs

```bash
# View all services
docker compose logs -f

# View specific service
docker compose logs -f api

# View last 100 lines
docker compose logs --tail 100
```

### System Monitoring

```bash
# Real-time system stats
htop

# Docker resource usage
docker stats

# Disk usage
df -h
du -sh /srv/projects/*
```

## Troubleshooting

### Service Won't Start

```bash
# Check service status
sudo systemctl status app-stack.service

# View logs
sudo journalctl -u app-stack.service -n 50

# Restart service
sudo systemctl restart app-stack.service
```

### Docker Issues

```bash
# Check Docker daemon
sudo systemctl status docker

# Verify user permissions
groups $USER

# Prune unused images/containers (careful!)
docker system prune -a
```

### Network/Port Issues

```bash
# Check which process uses port
sudo lsof -i :3000
sudo netstat -tlnp | grep 3000

# Test connectivity
netstat -tuln | grep LISTEN
```

## Quick Reference Commands

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# Rebuild and restart
docker compose up --build -d

# SSH into container
docker exec -it <container-name> /bin/bash

# Check system uptime
uptime

# Get IP address
ip addr show
ip route show | grep default
```

## Useful Docs & Resources

- [Ubuntu 24.04 LTS Documentation](https://ubuntu.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [GitHub Actions Self-Hosted Runners](https://docs.github.com/en/actions/hosting-your-own-runners)
- [PostgreSQL Backup Documentation](https://www.postgresql.org/docs/current/backup.html)

## Support & Updates

For issues or updates to this documentation, refer to the HouseHub main project documentation.

**Last Updated**: February 2026  
**Device**: Dell 5581 - Ubuntu 24.04.4 LTS  
**Status**: Production Ready
