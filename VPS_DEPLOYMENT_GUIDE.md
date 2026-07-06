# VPS Deployment Guide: Ubuntu, Nginx, Gunicorn, GitHub Actions & Let's Encrypt

> A complete production deployment guide for Python web applications on an Ubuntu VPS using Gunicorn, Nginx, GitHub Actions CI/CD, and Let's Encrypt SSL.

---

## Overview

This guide documents the complete deployment process for Python web applications running on an Ubuntu VPS.

It covers:

- Initial Ubuntu server setup
- Python virtual environments
- Environment variable configuration
- Gunicorn service setup
- Nginx reverse proxy configuration
- DNS configuration
- HTTPS with Let's Encrypt (Certbot)
- GitHub Actions automated deployment
- SSH deployment keys
- Deployment verification
- Troubleshooting
- Useful administration commands

Although this guide was written primarily for Flask applications, the same deployment workflow can be adapted for FastAPI, Django, and other Python web frameworks.

---

# Table of Contents

1. Initial Server Setup
2. Create Application Directory
3. Deploy the Application
4. Create Python Virtual Environment
5. Configure Environment Variables
6. Test the Application
7. Configure Gunicorn
8. Verify Gunicorn
9. Configure Nginx
10. Configure DNS
11. Verify HTTP
12. Install SSL
13. Configure GitHub Actions Deployment
14. Deployment Verification
15. Useful Commands
16. Troubleshooting
17. Deployment Checklist

---

# 1. Initial Server Setup (One Time Per VPS)

Update the server:

```bash
sudo apt update
sudo apt upgrade -y
```

Install required packages:

```bash
sudo apt install -y \
git \
nginx \
python3 \
python3-pip \
python3-venv \
certbot \
python3-certbot-nginx \
libreoffice-writer \
fonts-liberation
```

Enable Nginx:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

---

# 2. Create Application Directory

Example:

```bash
sudo mkdir -p /var/www/my-app
sudo chown $USER:$USER /var/www/my-app
```

---

# 3. Deploy the Application

Clone the repository:

```bash
cd /var/www

git clone https://github.com/USERNAME/REPOSITORY.git my-app

cd my-app
```

Or deploy automatically using GitHub Actions.

---

# 4. Create Python Virtual Environment

```bash
cd /var/www/my-app

python3 -m venv venv

source venv/bin/activate

pip install -r requirements.txt
```

---

# 5. Configure Environment Variables

Create the `.env` file:

```bash
nano /var/www/my-app/.env
```

Example:

```env
SMTP_USER=...
SMTP_PASSWORD=...
SMTP_FROM_NAME=Concept Engineers

PUBLIC_BASE_URL=https://example.com

SECRET_KEY=xxxxxxxxxxxxxxxx

DATABASE_URL=...

...
```

Save:

```
Ctrl + O
Enter
Ctrl + X
```

---

# 6. Test the Application

Run locally:

```bash
python app.py
```

Verify:

```
http://127.0.0.1:5000
```

Stop:

```
Ctrl + C
```

---

# 7. Configure Gunicorn

Create the systemd service:

```bash
sudo nano /etc/systemd/system/my-app.service
```

Example:

```ini
[Unit]
Description=My App
After=network.target

[Service]
User=root
WorkingDirectory=/var/www/my-app

EnvironmentFile=/var/www/my-app/.env

ExecStart=/var/www/my-app/venv/bin/gunicorn \
    --workers 3 \
    --bind 127.0.0.1:5000 \
    app:app

Restart=always

[Install]
WantedBy=multi-user.target
```

Reload systemd:

```bash
sudo systemctl daemon-reload
```

Enable:

```bash
sudo systemctl enable my-app
```

Start:

```bash
sudo systemctl start my-app
```

Check status:

```bash
sudo systemctl status my-app
```

---

# 8. Verify Gunicorn

Check listening ports:

```bash
ss -tlnp
```

Expected:

```
127.0.0.1:5000
```

Test:

```bash
curl http://127.0.0.1:5000
```

You should receive HTML.

---

# 9. Configure Nginx

Create a new site:

```bash
sudo nano /etc/nginx/sites-available/my-app
```

Example:

```nginx
server {

    listen 80;

    server_name app.example.com;

    client_max_body_size 25M;

    location / {

        proxy_pass http://127.0.0.1:5000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

    }

}
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/my-app /etc/nginx/sites-enabled/
```

Test configuration:

```bash
sudo nginx -t
```

Reload Nginx:

```bash
sudo systemctl reload nginx
```

---

# 10. Configure DNS

Create an **A Record**:

| Field | Value |
|-------|-------|
| Host | app |
| Type | A |
| Value | YOUR_VPS_PUBLIC_IP |

Verify:

```bash
dig +short app.example.com
```

Check your server IP:

```bash
curl -4 ifconfig.me
```

Both should match.

---

# 11. Verify HTTP

```bash
curl -I http://app.example.com
```

Expected:

```
HTTP/1.1 200 OK
```

---

# 12. Install SSL (Let's Encrypt)

Install the SSL certificate:

```bash
sudo certbot --nginx -d app.example.com
```

Verify HTTPS:

```
https://app.example.com
```

Test automatic renewal:

```bash
sudo certbot renew --dry-run
```

---

# 13. Configure GitHub Actions Deployment

## Repository Secrets

Configure the following GitHub Secrets:

```
VPS_HOST
VPS_USER
VPS_SSH_KEY
```

---

## Generate a Deployment SSH Key

Create a new key pair:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/deploy_key
```

Public key:

```
~/.ssh/deploy_key.pub
```

Append it to the server:

```
~/.ssh/authorized_keys
```

Private key:

Base64 encode:

```bash
base64 ~/.ssh/deploy_key
```

Store the output as:

```
VPS_SSH_KEY
```

Verify:

```bash
ssh USER@SERVER
```

---

# 14. Deployment Verification

After every deployment verify:

## GitHub Actions

- GitHub workflow completed successfully

## Gunicorn

```bash
sudo systemctl status my-app
```

## Nginx

```bash
sudo systemctl status nginx
```

## Application

```bash
curl http://127.0.0.1:5000
```

## Website

```
https://app.example.com
```

## SSL

Verify the browser shows a secure HTTPS connection.

## Logs

```bash
journalctl -u my-app -n 100
```

---

# 15. Useful Commands

## Restart Application

```bash
sudo systemctl restart my-app
```

## Stop Application

```bash
sudo systemctl stop my-app
```

## View Live Logs

```bash
journalctl -u my-app -f
```

## Reload Nginx

```bash
sudo systemctl reload nginx
```

## Restart Nginx

```bash
sudo systemctl restart nginx
```

## Test Nginx Configuration

```bash
sudo nginx -t
```

## Check Listening Ports

```bash
ss -tlnp
```

## Check DNS

```bash
dig +short app.example.com
```

## Check Public IPv4 Address

```bash
curl -4 ifconfig.me
```

## Check Public IPv6 Address

```bash
curl ifconfig.me
```

---

# 16. Troubleshooting

## Permission denied (publickey)

**Cause**

GitHub Actions cannot authenticate to the VPS.

**Check**

- VPS_SSH_KEY
- VPS_USER
- VPS_HOST
- ~/.ssh/authorized_keys
- SSH key permissions

---

## Load key "...": error in libcrypto

**Cause**

The SSH private key stored in GitHub Secrets is invalid or not Base64 encoded correctly.

**Solution**

Generate a new deployment key.

Base64 encode the private key.

Update the GitHub Secret.

---

## 502 Bad Gateway

**Cause**

Gunicorn is not running.

Check:

```bash
sudo systemctl status my-app
```

Restart:

```bash
sudo systemctl restart my-app
```

---

## Nginx Configuration Error

Validate configuration:

```bash
sudo nginx -t
```

Reload:

```bash
sudo systemctl reload nginx
```

---

## SSL Certificate Failed

Verify DNS:

```bash
dig +short app.example.com
```

Verify HTTP:

```bash
curl -I http://app.example.com
```

Retry:

```bash
sudo certbot --nginx -d app.example.com
```

---

## DNS Not Updating

Check current DNS:

```bash
dig +short app.example.com
```

Remember that DNS propagation can take several minutes or hours depending on your DNS provider.

---

# 17. Deployment Checklist

## Server

- [ ] Ubuntu updated
- [ ] Python installed
- [ ] Nginx installed
- [ ] Certbot installed
- [ ] LibreOffice installed
- [ ] Fonts installed

## Application

- [ ] Repository deployed
- [ ] Virtual environment created
- [ ] Requirements installed
- [ ] .env configured
- [ ] Gunicorn running

## Systemd

- [ ] Service created
- [ ] Service enabled
- [ ] Service running

## Nginx

- [ ] Configuration created
- [ ] Site enabled
- [ ] Configuration tested
- [ ] Nginx reloaded

## Domain

- [ ] DNS A record configured
- [ ] Domain resolves to VPS
- [ ] HTTP working

## SSL

- [ ] Certificate installed
- [ ] HTTPS working
- [ ] Auto-renew tested

## GitHub Actions

- [ ] VPS_HOST configured
- [ ] VPS_USER configured
- [ ] VPS_SSH_KEY configured
- [ ] Deployment successful

## Final Verification

- [ ] Website loads successfully
- [ ] HTTPS enabled
- [ ] Application logs are clean
- [ ] Emails working
- [ ] PDF generation working
- [ ] Server survives reboot
