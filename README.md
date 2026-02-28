# Gitea Deployment on AWS EC2 with Docker Compose and Nginx Reverse Proxy

## Architecture Summary

This project deploys Gitea on an AWS EC2 Ubuntu instance using Docker
Compose. Nginx runs as a reverse proxy container exposed on port 80 and
forwards traffic internally to the Gitea container on port 3000. Gitea's
data directory is bind-mounted to `/home/ubuntu/data`, which resides on
an attached EBS volume to ensure persistent storage beyond container
lifecycle events. Only ports 22 (SSH) and 80 (HTTP) are publicly
accessible, improving security posture.

------------------------------------------------------------------------

## Architecture Diagram

    Internet
       │
       ▼
    EC2 Security Group (Allow 22, 80)
       │
       ▼
    Nginx Container (Port 80)
       │   Reverse Proxy
       ▼
    Gitea Container (Port 3000 - internal only)
       │
       ▼
    Bind Mount → /home/ubuntu/data
       │
       ▼
    EBS Volume (Persistent Storage)
       │
       ▼
    Backup Script (backup.sh)
       │
       ▼
    Tar Archive
       │
       ▼
Amazon S3 Bucket (s3-gitea-server)


------------------------------------------------------------------------

# Deployment Instructions (Step-by-Step)

## 1. Launch EC2 Instance

-   Ubuntu 22.04 LTS
-   t2.micro (or similar)
-   Security Group:
    -   Port 22 → Your IP only
    -   Port 80 → 0.0.0.0/0

SSH into instance:

ssh -i your-key.pem ubuntu@<PUBLIC-IP>

------------------------------------------------------------------------

## 2. Attach and Mount EBS Volume

Check device:

lsblk

Format (first time only):

sudo mkfs.ext4 /dev/nvme1n1

Create mount point:

mkdir -p /home/ubuntu/data

Mount volume:

sudo mount /dev/nvme1n1 /home/ubuntu/data

Set permissions:

sudo chown -R ubuntu:ubuntu /home/ubuntu/data

Verify:

df -h

------------------------------------------------------------------------

## 3. Install Docker (Official Repository)

sudo apt update
sudo apt install ca-certificates curl gnupg -y

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu   $(. /etc/os-release && echo "$VERSION_CODENAME") stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

sudo usermod -aG docker ubuntu

Log out and back in.

Verify:

docker --version
docker compose version

------------------------------------------------------------------------

## Install AWS CLI v2 (Official Repository)

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install

aws --version
aws s3 ls

------------------------------------------------------------------------

## 5. Project Structure

mkdir ~/gitea-stack
cd ~/gitea-stack
mkdir nginx

------------------------------------------------------------------------

## 5. docker-compose.yml

Create `docker-compose.yml`:

services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: always
    volumes:
      - /home/ubuntu/data:/data
    expose:
      - "3000"
    networks:
      - gitea-net

  nginx:
    image: nginx:latest
    container_name: gitea-nginx
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - gitea
    networks:
      - gitea-net

networks:
  gitea-net:
    driver: bridge

------------------------------------------------------------------------

## 6. Nginx Reverse Proxy Config

Create `nginx/nginx.conf`:

server {
    listen 80;
    server_name YOUR_PUBLIC_IP;

    location / {
        proxy_pass http://gitea:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

------------------------------------------------------------------------

## 7. Start the Stack

docker compose up -d
docker ps

Access:

    http://<PUBLIC-IP>

------------------------------------------------------------------------

## 8. Configure ROOT_URL

Edit:

    /home/ubuntu/data/gitea/conf/app.ini

Ensure:

[server]
PROTOCOL = http
DOMAIN = YOUR_PUBLIC_IP
ROOT_URL = http://YOUR_PUBLIC_IP/
HTTP_PORT = 3000

Restart:

docker restart gitea

------------------------------------------------------------------------

# Backup and Restore

## Backup Script (backup.sh)

#!/usr/bin/env bash
set -euo pipefail
TS="$(date -u +%Y%m%dT%H%M%SZ)"
ARCHIVE="/tmp/gitea-backup-${TS}.tar.gz"
sudo tar -czf "${ARCHIVE}" -C "$HOME/data" .
echo "Created backup archive: ${ARCHIVE}"

Run:

chmod +x backup.sh
./backup.sh

------------------------------------------------------------------------

## Upload to S3

aws s3 cp <archive-name>.tar.gz s3://YOUR-BUCKET/backups/
aws s3 ls s3://YOUR-BUCKET/backups/

------------------------------------------------------------------------

## Restore Procedure (EBS)

1.  Stop containers:

docker compose down

2.  Check URL:

curl -I http://<---->

3.  Restart:

docker compose up -d

4.  Check URL:

curl -I http://<---->

5.  Verify repository in browser.

------------------------------------------------------------------------

# Persistence Test (EBS)

docker compose down
docker compose up -d

Repository remains intact due to EBS-backed bind mount.

------------------------------------------------------------------------

## Restore Procedure (S3)

1.  Stop containers:

docker compose down

2.  Check URL:

curl -I http://<---->

3.  Download backup:

aws s3 cp s3://YOUR-BUCKET/backups/<archive>.tar.gz /tmp/

4.  Extract:

sudo tar -xzf /tmp/<archive>.tar.gz -C /home/ubuntu/data

5.  Restart:

docker compose up -d

6.  Check URL:

curl -I http://<---->

7.  Verify repository in browser.

------------------------------------------------------------------------

# Persistence Test (S3)

docker compose down
docker compose up -d

Repository remains intact after S3 restore.

------------------------------------------------------------------------

# Security Notes

-   Port 3000 is not publicly exposed.
-   SSH restricted to your IP.
-   Reverse proxy handles external access.
-   EBS ensures durable storage.

------------------------------------------------------------------------

# 👤 Author

**Mohammed Golam Kaisar Hossain Bhuyan**  
GitHub: [https://kaisarhossain.github.io/portfolio/]
LinkedIn: [https://www.linkedin.com/in/kaisarhossain/]
Email: kaisar.hossain@gmail.com
CUA Email: hossainbhuyan@cua.edu

