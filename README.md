# 🚀 Django Production Deployment (Step-by-Step)

Docker + PostgreSQL + GitHub Actions (CI/CD) + AWS EC2 + Nginx + Gunicorn + Custom Domain + SSL

This repository demonstrates how to deploy a Django application from local development to production using:

- Django
- Docker & Docker Compose
- PostgreSQL
- GitHub Actions (CI/CD)
- AWS EC2 (Free Tier / Student Account)
- Nginx
- Gunicorn
- Custom Domain
- SSL (Let's Encrypt)

You will go step-by-step from:

**Local → Docker → GitHub → AWS EC2 → Domain → HTTPS**

---

## 🧰 Prerequisites

Install the following on your system:

- Git
- Python 3.10+
- pip
- Docker Desktop
- VS Code (recommended)
- AWS Student Account (AWS Academy / AWS Educate / GitHub Student Pack — any works)

---

## 📦 Step 1 — Clone the Project

```bash
git clone https://github.com/dev-rathankumar/django_clickmart_
cd django_clickmart_
```

**Step 2 - Remove Git history**

```bash
rm -rf .git
```

This wipes your commit history & remote. Now it is just files in your local computer, not a repo.

**Create your own GitHub repository**

Go to GitHub → Click New Repository → Name: django-clickmart

**Re-initialize Git**

```bash
git init
git add .
git commit -m "Initial project setup"
git branch -M main
git remote add origin https://github.com/<YOUR-USERNAME>/<REPOSITORY-NAME>.git
git push -u origin main
```

Now you have the full source code in your own repo.

---

## Run Django Locally (Without Docker)

**Create virtual environment**

```bash
cd backend-drf
python3 -m venv env
source env/bin/activate     # Mac / Linux
# OR
env\Scripts\activate        # Windows
```

**Install dependencies**

```bash
pip install -r requirements.txt
```

**Create .env file**

```
DEBUG=True
SECRET_KEY=<YOUR-SECRET-KEY>

# Database Settings
DB_NAME=<DATABASE-NAME>
DB_USER=<POSTGRES-USERNAME>
DB_PASSWORD=<YOUR-PASSWORD>
DB_HOST=localhost
DB_PORT=5432

# Email Configuration
EMAIL_HOST_USER=<YOUR-EMAIL-ADDRESS>
EMAIL_HOST_PASSWORD=<PASSWORD> # USE APP PASSWORD IF YOU ARE USING GMAIL
```

**Create database tables and run the Django server**

```bash
python manage.py migrate
python manage.py runserver
```

**Create .env file inside /frontend/ directory and write:**

```
VITE_SERVER_BASE_URL=http://127.0.0.1:8000/api/v1
```

**And run the frontend - React**

```bash
npm install
npm run dev
```

Go to http://localhost:5173/

Optional: You can now create superuser and add some products.

To learn about deployment, continue to next step...

---

## Install and verify Docker and Docker Compose

```bash
docker --version
docker compose version
```

_(All Dockerfile and docker-compose.yml steps remain the same as before. Continue until you have completed the "Create superuser inside Docker container" step.)_

```bash
docker compose exec backend python manage.py createsuperuser
```

✅ You have completed the local Docker setup. Now let's move to AWS.

---

## Create AWS EC2 Server & SSH Key

### 👉 Login to your AWS Student Account

- Go to [https://aws.amazon.com](https://aws.amazon.com)
- Login with your student/sandbox credentials
- Make sure you are in a region close to you (e.g., `us-east-1`, `ap-south-1`)

---

### Launch an EC2 Instance

**Go to EC2 Dashboard**

AWS Console → Search "EC2" → Click "Launch Instance"

**Fill in the details:**

| Field         | Value                                        |
| ------------- | -------------------------------------------- |
| Name          | django-clickmart                             |
| AMI (OS)      | Ubuntu Server 22.04 LTS (Free Tier eligible) |
| Instance Type | t2.micro (Free Tier eligible)                |
| Key pair      | Create new key pair (see below)              |
| Storage       | 20 GB gp2 (default is fine)                  |

---

### Create SSH Key Pair

When asked for Key Pair during EC2 launch:

- Click **"Create new key pair"**
- Name: `clickmart-aws`
- Key pair type: `ED25519`
- Private key file format: `.pem`
- Click **"Create key pair"** — it will auto-download a `.pem` file

Save the `.pem` file safely. You cannot download it again.

Move it to your SSH folder:

```bash
mv ~/Downloads/clickmart-aws.pem ~/.ssh/clickmart-aws.pem
chmod 400 ~/.ssh/clickmart-aws.pem
```

---

### Configure Security Group (Firewall)

During EC2 launch, under **"Network settings"** → Click **"Edit"**

Add the following inbound rules:

| Type       | Port | Source                          |
| ---------- | ---- | ------------------------------- |
| SSH        | 22   | My IP (or Anywhere for testing) |
| HTTP       | 80   | Anywhere                        |
| Custom TCP | 8000 | Anywhere                        |
| Custom TCP | 5173 | Anywhere                        |

> ⚠️ If ports are not opened, the app will run but won't be accessible from the browser.

Click **"Launch Instance"**.

---

### Get Your EC2 Public IP

Go to EC2 → Instances → Click your instance → Copy **Public IPv4 address**

---

### SSH into EC2

```bash
ssh -i ~/.ssh/clickmart-aws.pem ubuntu@<EC2_PUBLIC_IP>
```

> Note: AWS Ubuntu instances use `ubuntu` as the username, not `root`.

**Update the server:**

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Install Required Software on EC2

**Install Docker:**

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu
newgrp docker
docker --version
```

**Install Docker Compose:**

```bash
sudo apt install docker-compose-plugin -y
docker compose version
```

**Install Git:**

```bash
sudo apt install git -y
git --version
```

✅ Docker, Docker Compose, and Git installed successfully.

---

## Clone Project into /opt

Reconnect to SSH (if disconnected):

```bash
ssh -i ~/.ssh/clickmart-aws.pem ubuntu@<EC2_PUBLIC_IP>
```

```bash
cd /opt
sudo mkdir clickmart
sudo chown ubuntu:ubuntu clickmart
cd clickmart
git clone https://github.com/your-repo.git .
```

Repo is now cloned inside `/opt/clickmart`

---

## Update Frontend Environment Variable

In `docker-compose.yml`:

```yaml
VITE_SERVER_BASE_URL="http://<EC2_PUBLIC_IP>:8000/api/v1"
```

Push changes:

```bash
git push origin main
```

---

## Create Environment Files on EC2

```bash
nano backend-drf/.env.production
nano backend-drf/.env.docker
```

Add required environment variables inside each file.

---

## Build & Run Docker Containers

```bash
docker compose up --build -d
docker compose ps
```

**Test in browser:**

- Backend: `http://<EC2_PUBLIC_IP>:8000/`
- Frontend: `http://<EC2_PUBLIC_IP>:5173/`

---

## Fix Django ALLOWED_HOSTS

**In local `settings.py`:**

```python
ALLOWED_HOSTS = os.getenv("ALLOWED_HOSTS", "").split(",")

CORS_ALLOWED_ORIGINS = [
    'http://localhost:5173',
    'http://<EC2_PUBLIC_IP>:5173'
]
```

**In local `.env.docker`:**

```
ALLOWED_HOSTS=<EC2_PUBLIC_IP>,localhost,127.0.0.1
```

**In EC2 `.env.docker`:**

```
ALLOWED_HOSTS=<EC2_PUBLIC_IP>,localhost,127.0.0.1
```

**In `docker-compose.yml`:**

```yaml
VITE_SERVER_BASE_URL: "http://<EC2_PUBLIC_IP>/api/v1"
```

**Push to GitHub:**

```bash
git add .
git commit -m "Allowed host & environments added"
git push origin main
```

---

## 🎯 Goal - Whenever I push code to GitHub, my EC2 server should automatically update.

But first...

**Manually pull the code from GitHub to EC2.**

While logged in to EC2:

```bash
git pull origin main
```

**Rebuild containers:**

```bash
docker compose down -v
docker compose up --build -d
```

**Rule Before Automation**

> ❗Never automate something you haven't done manually.

---

## Setup CI/CD (GitHub Actions)

In local project, create a new file:

`.github/workflows/automate.yml`

```yaml
name: Auto Deploy to AWS EC2

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd /opt/clickmart
            git pull origin main
            docker compose up --build -d
```

**Add GitHub Secrets:**

GitHub → Your Repository → Settings → Secrets and variables → Actions → New repository secret

| Secret Name | Value                                                                                |
| ----------- | ------------------------------------------------------------------------------------ |
| EC2_HOST    | `<EC2_PUBLIC_IP>`                                                                    |
| EC2_USER    | `ubuntu`                                                                             |
| EC2_SSH_KEY | Contents of your `.pem` file (open it with a text editor and paste the full content) |

**How to copy the SSH private key content:**

```bash
cat ~/.ssh/clickmart-aws.pem
```

Copy everything including `-----BEGIN OPENSSH PRIVATE KEY-----` and paste it as the secret value.

**Push automation file:**

```bash
git add .
git commit -m "CI/CD Setup"
git push origin main
```

Check the **GitHub Actions** tab.

Make a small frontend change and confirm auto-deploy.

✅ Auto deploy successful.

---

## Nginx Config

From local project, create file:

`nginx/default.conf`

```nginx
server {
    listen 80;

    # Frontend (React)
    location / {
        proxy_pass http://frontend:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Backend (Django)
    location /api/ {
        proxy_pass http://backend:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Django admin & static
    location /admin/ {
        proxy_pass http://backend:8000;
    }

    location /static/ {
        proxy_pass http://backend:8000;
    }

    location /media/ {
        proxy_pass http://backend:8000;
    }
}
```

**Docker Compose Changes**

- Add nginx service
- Remove ports from backend & frontend
- Update frontend API URL: `VITE_SERVER_BASE_URL="/api/v1"`

```yaml
nginx:
  image: nginx:alpine
  ports:
    - "80:80"
  volumes:
    - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
  depends_on:
    - frontend
    - backend
```

Push changes:

```bash
git add .
git commit -m "Nginx Setup"
git push origin main
```

---

## Update Firewall (Production)

Go to AWS Console → EC2 → Security Groups → Click your instance's security group → Edit inbound rules

**Keep:**

- 22 (SSH)
- 80 (HTTP)

**Remove:**

- 8000 (Backend)
- 5173 (Frontend)

---

## Final Test

`http://<EC2_PUBLIC_IP>/`

If you get an error: SSH into EC2 and manually add backend to allowed hosts in `.env.docker`.

**Restart docker:**

```bash
docker compose down -v
docker compose up --build -d
```

---

## Gunicorn Setup (Production WSGI Server)

**1. Add Gunicorn Dependency**

Add `gunicorn` inside `requirements.txt`

**Update Backend Dockerfile**

No special change is required other than ensuring `requirements.txt` is installed. Gunicorn will be installed automatically via dependencies.

**Update `docker-compose.yml`**

Replace the Django run command with Gunicorn:

```yaml
command: >
  gunicorn clickmart_main.wsgi:application --bind 0.0.0.0:8000 --workers 3
```

- `clickmart_main.wsgi:application` → Django entry point
- `--bind 0.0.0.0:8000` → Listen on all interfaces
- `--workers 3` → Run 3 Python worker processes

```bash
git add .
git commit -m "Deploy Gunicorn"
git push origin main
```

**Important Note**

✅ We did not change the application code.

✅ We only changed how Python code is executed in production.

**Verify Gunicorn Is Running**

SSH into the EC2 server:

```bash
ssh -i ~/.ssh/clickmart-aws.pem ubuntu@<EC2_PUBLIC_IP>
cd /opt/clickmart
docker compose logs backend
```

---

## Purchase a Domain

Purchase a domain from any provider (GoDaddy, Namecheap, etc.).

**Connect Domain to AWS EC2 (DNS)**

Add the following A records in your domain DNS:

| Type | Host | Value                  |
| ---- | ---- | ---------------------- |
| A    | @    | `<YOUR_EC2_PUBLIC_IP>` |
| A    | www  | `<YOUR_EC2_PUBLIC_IP>` |

Wait for DNS propagation (usually a few minutes to a few hours).

> ⚠️ AWS EC2 Free Tier IPs can change if you stop/start your instance. To get a permanent IP, assign an **Elastic IP** to your instance (free while the instance is running).

**How to assign Elastic IP:**

AWS Console → EC2 → Elastic IPs → Allocate Elastic IP address → Associate it with your instance.

Now your IP will never change even after restarts.

---

## Nginx Config as Server-Managed File

Certbot modifies the Nginx config directly on the server, so we must remove it from Git tracking.

```bash
git rm --cached nginx/default.conf
```

Removes the file from Git. Add to `.gitignore`:

```
nginx/default.conf
```

**Commit and Push**

```bash
git add .
git commit -m "Make nginx config server-managed"
git push origin main
```

**SSH into EC2 server**

```bash
ssh -i ~/.ssh/clickmart-aws.pem ubuntu@<EC2_PUBLIC_IP>
```

Create `nginx/default.conf` file and add domain to this file:

```nginx
server_name example.com www.example.com;
```

**Restart nginx:**

```bash
docker compose restart nginx
```

**Update Django ALLOWED_HOSTS**

Add your domain into `.env.docker`

**Restart backend:**

```bash
docker compose restart backend
```

**Test Domain (HTTP only)**

`http://example.com`

---

## Install SSL (Let's Encrypt)

In the server root directory, create folders:

```bash
mkdir -p certbot/www
mkdir -p certbot/conf
```

**Update `docker-compose.yml` (Nginx service)**

Edit `docker-compose.yml` locally (nginx service):

```yaml
volumes:
  - ./certbot/www:/var/www/certbot
  - ./certbot/conf:/etc/letsencrypt
```

Push to main branch.

**Update `nginx/default.conf`**

Add this block:

```nginx
location /.well-known/acme-challenge/ {
    root /var/www/certbot;
}
```

Restart Nginx container:

```bash
docker compose restart nginx
```

Make sure the site with HTTP still works at this point.

**Install Certbot**

```bash
sudo apt update
sudo apt install certbot -y
```

**Get SSL Certificate (WEBROOT METHOD)**

```bash
sudo certbot certonly \
  --webroot \
  -w /opt/clickmart/certbot/www \
  -d example.com \
  -d www.example.com
```

**Enable HTTPS in Nginx**

Edit `nginx/default.conf` again. Replace with FINAL CONFIG:

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        proxy_pass http://frontend:80;
    }

    location /api/ {
        proxy_pass http://backend:8000;
    }

    location /admin/ {
        proxy_pass http://backend:8000;
    }

    location /static/ {
        alias /static/;
    }
}
```

**Restart Nginx**

```bash
docker compose restart nginx
```

**Test HTTPS 🎉**

`https://example.com`

Congratulations 🎉 You did it.

---

## Fixing Media Files in Production (Docker + Nginx + Django)

This guide explains how to fix issues where media files (uploaded images) are not loading correctly in production.

**Step 1: Update Nginx Configuration (Server)**

Login to your production server.

Open the Nginx config file:

```bash
nano nginx/default.conf
```

Add the following block inside the HTTPS server block:

```nginx
location /media/ {
    alias /media/;
}
```

This tells Nginx to serve uploaded media files directly.

Restart nginx container:

```bash
docker compose restart nginx
```

**Step 2: Mount Media Folder in Docker (Local Project)**

Open `docker-compose.yml` in your local project. Inside the nginx service, add the media volume mapping:

```yaml
nginx:
  volumes:
    - ./backend-drf/media:/media
```

This allows the Nginx container to access uploaded media files created by Django.

Commit and push the changes:

```bash
git add .
git commit -m "Serve media files using nginx"
git push origin main
```

**Step 3: Verify Media Files**

Try opening a media file directly in the browser:

`https://your-domain.com/media/example.jpg`

If the image loads, media serving is working correctly.

**Step 4 (Fallback): Fix Serializer Image URL**

If media files load directly but still do not appear on the webpage, update the serializer to return a relative media path.

Open `products/serializers.py`. Update `ProductSerializer` - or whatever serializer the image is coming from. Refer to below code:

```python
from rest_framework import serializers

class ProductSerializer(serializers.ModelSerializer):
    image = serializers.SerializerMethodField()

    class Meta:
        model = Product
        fields = "__all__"

    def get_image(self, obj):
        return obj.image.url if obj.image else None
```

This ensures the API returns `/media/products/image.jpg` instead of Docker-internal URLs like `backend:8000`

Commit and push again:

```bash
git add .
git commit -m "Fix media image URL in serializer"
git push origin main
```

Test again.

---

## 💡 AWS Student Account Tips

- **t2.micro** is free tier eligible — always choose this instance type.
- **Elastic IP** is free as long as the instance is running. If you stop the instance without releasing the Elastic IP, AWS will charge a small amount. Either keep the instance running or release the Elastic IP when not using it.
- **AWS Sandbox / Academy accounts** may have region restrictions. If EC2 is not available, switch to `us-east-1`.
- Your sandbox credits are limited — avoid running heavy instance types or multiple instances at once.
- Set up a **billing alert** in AWS Console → Billing → Budgets to get notified if you're close to your credit limit.
