https://github.com/Arya-Mhaske/mern-student-management
https://github.com/Arya-Mhaske/mern-task-management
https://github.com/Arya-Mhaske/mern-ecommerce
https://github.com/Arya-Mhaske/mern-event-registration
https://github.com/Arya-Mhaske/mern-online-blog

# 🚀 MERN Stack Web App Deployment on AWS EC2

> **Stack:** MongoDB Atlas · Express.js · React (Vite) · Node.js  
> **Platform:** AWS EC2 (Ubuntu 24.04 LTS)  
> **Repo used:** [mern-online-blog](https://github.com/Arya-Mhaske/mern-online-blog)

---

## 📋 Table of Contents

1. [AWS EC2 Instance Setup](#1-aws-ec2-instance-setup)
2. [Connect to EC2 via SSH](#2-connect-to-ec2-via-ssh)
3. [Update System & Install NVM + Node.js](#3-update-system--install-nvm--nodejs)
4. [Install Git & Clone Repository](#4-install-git--clone-repository)
5. [MongoDB Atlas Setup](#5-mongodb-atlas-setup)
6. [Backend Setup](#6-backend-setup)
7. [Install PM2 & Start Backend](#7-install-pm2--start-backend)
8. [Frontend Setup](#8-frontend-setup)
9. [Start Frontend with PM2](#9-start-frontend-with-pm2)
10. [Verify Deployment](#10-verify-deployment)
11. [Useful Commands](#11-useful-commands)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. AWS EC2 Instance Setup

### Launch an EC2 Instance
1. Log in to [AWS Console](https://console.aws.amazon.com) → **EC2** → **Launch Instance**
2. Configure:
   - **Name:** `mern-blog-server`
   - **AMI:** Ubuntu Server 24.04 LTS (Free Tier eligible)
   - **Instance type:** `t3.small` (recommended) or `t2.micro` (free tier)
   - **Key pair:** Create new → name `mern-blog-key` → download `.pem` file

### Security Group (Firewall) Rules

| Type       | Protocol | Port | Source            | Purpose         |
|------------|----------|------|-------------------|-----------------|
| SSH        | TCP      | 22   | My IP             | Secure SSH access |
| HTTP       | TCP      | 80   | 0.0.0.0/0         | Web traffic     |
| HTTPS      | TCP      | 443  | 0.0.0.0/0         | Secure web traffic |
| Custom TCP | TCP      | 5000 | 0.0.0.0/0         | Backend API     |
| Custom TCP | TCP      | 5173 | 0.0.0.0/0         | Frontend (Vite) |

3. **Storage:** 8 GiB gp3 (default is fine)
4. Click **Launch Instance** and wait for all 3 status checks to pass ✅

---

## 2. Connect to EC2 via SSH

### On Windows (PowerShell) / Mac / Linux

```bash
# Navigate to the folder containing your .pem key
cd Downloads

# On Mac/Linux: fix key permissions first
chmod 400 mern-blog-key.pem

# SSH into the instance (replace with your actual Public IPv4)
ssh -i mern-blog-key.pem ubuntu@<YOUR_PUBLIC_IP>
```

> 💡 Find your **Public IPv4 address** in EC2 → Instances → click your instance → Details tab.

When prompted:
```
Are you sure you want to continue connecting (yes/no)? yes
```

✅ You're in when you see: `ubuntu@ip-172-xx-xx-xx:~$`

---

## 3. Update System & Install NVM + Node.js

```bash
# Update and upgrade system packages
sudo apt update && sudo apt upgrade -y

# Install NVM (Node Version Manager)
# ⚠️ Do NOT use: sudo apt install nodejs (installs old version incompatible with Vite)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Reload shell to activate NVM
source ~/.bashrc

# Install Node.js version 22
nvm install 22
nvm use 22

# Verify installation
node -v    # Should show v22.x.x
npm -v     # Should show 10.x.x
```

---

## 4. Install Git & Clone Repository

```bash
# Install Git
sudo apt install git -y

# Verify Git installation
git --version

# Clone the repository
git clone https://github.com/Arya-Mhaske/mern-online-blog.git

# Navigate into the project folder
cd mern-online-blog

# Verify project structure
ls
# Expected output: client  server  Dockerfile  docker-compose.yml  README.md
```

---

## 5. MongoDB Atlas Setup

1. Go to [https://cloud.mongodb.com](https://cloud.mongodb.com) → Sign up / Log in
2. Create a **Free M0 cluster** (choose nearest region)
3. **Security → Database Access** → Add New Database User:
   - Username: `bloguser`
   - Password: `YourStrongPassword`
   - Role: `Read and write to any database`
4. **Security → Network Access** → Add IP Address → **Allow Access from Anywhere** (`0.0.0.0/0`)
5. **Clusters** → Click **Connect** → **Drivers** → Select `Node.js`
6. Copy the connection string:
   ```
   mongodb+srv://bloguser:YourStrongPassword@cluster0.xxxxx.mongodb.net/blogdb?retryWrites=true&w=majority&appName=Cluster0
   ```

---

## 6. Backend Setup

```bash
# Navigate to the server folder
cd ~/mern-online-blog/server

# Clean install dependencies (recommended)
rm -rf node_modules package-lock.json
npm install

# Create the environment variables file
nano .env
```

Paste the following into the `.env` file:

```env
PORT=5000
MONGO_URI=mongodb+srv://bloguser:YourStrongPassword@cluster0.xxxxx.mongodb.net/blogdb?retryWrites=true&w=majority&appName=Cluster0
```

Save and exit nano:
- `Ctrl+O` → `Enter` (save)
- `Ctrl+X` (exit)

```bash
# Verify .env file contents
cat .env
```

---

## 7. Install PM2 & Start Backend

> PM2 is a process manager that keeps your app running after you close the SSH session.

```bash
# Install PM2 globally
npm install -g pm2

# Start the backend server
pm2 start index.js --name "blog-backend"

# Save PM2 process list (auto-restart on reboot)
pm2 save

# Configure PM2 to start on system boot
pm2 startup
# ⚠️ Copy and run the command that PM2 outputs after this

# Check backend is running
pm2 status

# Test backend API response
curl http://localhost:5000
```

✅ You should see the backend responding at `http://localhost:5000`

---

## 8. Frontend Setup

```bash
# Navigate to the client folder
cd ~/mern-online-blog/client

# Edit Vite config to point to your EC2 public IP
nano vite.config.js
```

Update the `target` in the proxy section to your EC2 **Public IPv4 address**:

```js
export default {
  server: {
    proxy: {
      '/api': {
        target: 'http://<YOUR_EC2_PUBLIC_IP>:5000',
        changeOrigin: true,
      }
    },
    host: '0.0.0.0',   // Allow external access
    port: 5173
  }
}
```

Save and exit: `Ctrl+O` → `Enter` → `Ctrl+X`

```bash
# Clean install frontend dependencies
rm -rf node_modules package-lock.json
npm install
```

---

## 9. Start Frontend with PM2

```bash
# Start frontend via PM2
pm2 start "npm run dev" --name "blog-frontend"

# Save updated PM2 process list
pm2 save

# Verify both services are running
pm2 status
```

Expected output:
```
┌────┬──────────────────┬─────────┬──────┬───────────┬──────────┐
│ id │ name             │ mode    │ pid  │ status    │ cpu      │
├────┼──────────────────┼─────────┼──────┼───────────┼──────────┤
│ 0  │ blog-backend     │ fork    │ xxxx │ online    │ 0%       │
│ 1  │ blog-frontend    │ fork    │ xxxx │ online    │ 0%       │
└────┴──────────────────┴─────────┴──────┴───────────┴──────────┘
```

---

## 10. Verify Deployment

Open a browser and navigate to:

```
http://<YOUR_EC2_PUBLIC_IP>:5173
```

You should see the **MERN Blog App** running! ✅

- ✅ Create new blog posts
- ✅ View all blog posts
- ✅ Delete blog posts

**Backend API** is accessible at:
```
http://<YOUR_EC2_PUBLIC_IP>:5000
```

---

## 11. Useful Commands

```bash
# ── PM2 Process Management ──────────────────────────────────────

# Check status of all processes
pm2 status

# View backend logs (live)
pm2 logs blog-backend

# View frontend logs (live)
pm2 logs blog-frontend

# Restart a specific service
pm2 restart blog-backend
pm2 restart blog-frontend

# Restart all services
pm2 restart all

# Stop all services
pm2 stop all

# Delete a process from PM2
pm2 delete blog-backend

# ── SSH ─────────────────────────────────────────────────────────

# Reconnect to EC2
ssh -i mern-blog-key.pem ubuntu@<YOUR_EC2_PUBLIC_IP>

# ── File Management ─────────────────────────────────────────────

# View .env contents
cat ~/mern-online-blog/server/.env

# Edit a file
nano <filename>

# ── Node / npm ──────────────────────────────────────────────────

# Check Node version
node -v

# Check npm version
npm -v

# Switch Node version (if needed)
nvm use 22

# ── System ──────────────────────────────────────────────────────

# Check disk usage
df -h

# Check memory usage
free -m

# Check running processes
htop
```

---

## 12. Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `Permission denied (publickey)` | Wrong .pem file or path | Check file path; on Mac/Linux run `chmod 400 mern-blog-key.pem` |
| `Connection timed out` on SSH | Security group missing port 22 | Add SSH inbound rule in EC2 Security Group |
| Can't access app in browser | Ports 5000/5173 not open | Add Custom TCP rules for 5000 and 5173 in Security Group |
| `nvm: command not found` | Shell not reloaded | Run `source ~/.bashrc` |
| Frontend can't reach backend | Wrong IP in vite.config.js | Update `target` to your EC2 Public IPv4 |
| MongoDB connection error | Wrong URI or network access | Check Atlas Network Access whitelist & `.env` URI |
| App stops after SSH closes | Not using PM2 | Use `pm2 start` instead of `node index.js` directly |
| SSH disconnected mid-task | Idle timeout | Reconnect with same SSH command and resume |
| Dependency errors | Stale node_modules | Run `rm -rf node_modules package-lock.json && npm install` |
| `EACCES` permission error on npm | Global install permission | Use `sudo npm install -g` or fix npm permissions |

---

## 🏗️ Architecture Overview

```
Internet (Users)
       │
       ▼
  EC2 Instance (Ubuntu 24.04) — Public IP: 13.x.x.x
       │
       ├── Port 5173 ──► React/Vite Frontend (PM2: blog-frontend)
       │                        │
       │                        └── API calls proxy to ▼
       └── Port 5000 ──► Express Backend (PM2: blog-backend)
                                 │
                                 └── MongoDB Atlas (Cloud, Mumbai)
```

---

## 📝 Notes

- **t3.small** is recommended over t2.micro for running both frontend and backend simultaneously without memory issues.
- **SSH source** should be set to **My IP** (not Anywhere) for security. Update it if your IP changes.
- **MongoDB Atlas M0** (free tier) has a 512 MB storage limit — sufficient for development and demo purposes.
- If your EC2 instance is stopped and restarted, the **Public IP will change** unless you assign an **Elastic IP**.

---

*Guide prepared for deploying: [Arya-Mhaske/mern-online-blog](https://github.com/Arya-Mhaske/mern-online-blog)*

# 🚀 MERN Stack Deployment on AWS EC2 — Complete Guide

## Prerequisites
- AWS Account
- GitHub Account
- MongoDB Atlas Account (free forever at mongodb.com/atlas)
- Your MERN project with `client/` (React) and `server/` (Express) folders

---

## STEP 1 — Launch EC2 Instance

1. Go to **AWS Console → EC2 → Launch Instance**
2. Fill in:
   - **Name:** `mern-app-server`
   - **AMI:** Ubuntu Server 22.04 LTS or 26.04 LTS (Free tier eligible)
   - **Instance type:** `t2.micro`
3. **Key pair:** Click "Create new key pair"
   - Name: `mern-key`
   - Type: RSA, Format: `.pem`
   - ⚠️ Download and save the `.pem` file safely — you can't download it again!
4. **Network settings → Edit → Add rules:**
   | Type | Port | Source |
   |------|------|--------|
   | SSH | 22 | Anywhere (0.0.0.0/0) |
   | HTTP | 80 | Anywhere |
   | HTTPS | 443 | Anywhere |
   | Custom TCP | 5000 | Anywhere |
5. **Storage:** 8 GiB (default) or up to 30 GiB (free tier limit)
6. Click **"Launch Instance"**
7. Wait for **Status checks → 2/2 checks passed**

---

## STEP 2 — Connect to EC2 via SSH

### Windows (PowerShell):
```powershell
cd ~/Downloads

# Fix key permissions
icacls "mern-key.pem" /inheritance:r /grant:r "$($env:USERNAME):(R)"

# Connect
ssh -i "mern-key.pem" ubuntu@<YOUR-PUBLIC-IP>
```

### Mac/Linux (Terminal):
```bash
cd ~/Downloads
chmod 400 mern-key.pem
ssh -i "mern-key.pem" ubuntu@<YOUR-PUBLIC-IP>
```

> Type `yes` when asked about fingerprint.
> You'll see: `ubuntu@ip-xxx-xxx-xxx-xxx:~$`

---

## STEP 3 — Install Dependencies on EC2

```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Install Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs -y

# Verify
node -v && npm -v

# Install PM2 (keeps app running)
sudo npm install -g pm2

# Install Nginx (reverse proxy)
sudo apt install nginx -y
```

---

## STEP 4 — Set Up MongoDB Atlas

1. Go to [mongodb.com/atlas](https://mongodb.com/atlas) → Sign up (free)
2. Create a **Free M0 cluster**
   - Provider: AWS
   - Region: Mumbai (ap-south-1) or closest to you
3. **Database Access** → Add Database User
   - Username & Password (save these!)
4. **Network Access** → Add IP Address → Allow Access from Anywhere (`0.0.0.0/0`)
5. **Connect → Drivers → Node.js** → Copy connection string:
   ```
   mongodb+srv://<username>:<password>@cluster0.xxxxx.mongodb.net/<dbname>?appName=Cluster0
   ```

---

## STEP 5 — Push Code to GitHub

On your **local PC**, run in PowerShell/Terminal:

```bash
cd path/to/your/project

git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/YOUR-REPO.git
git push -u origin main
```

> If remote already exists: `git remote set-url origin https://github.com/YOUR-USERNAME/YOUR-REPO.git`

---

## STEP 6 — Clone Project on EC2

Back in your **SSH terminal**:

```bash
cd ~
git clone https://github.com/YOUR-USERNAME/YOUR-REPO.git
cd YOUR-REPO
ls  # verify client/ and server/ folders exist
```

---

## STEP 7 — Setup & Start Backend

```bash
cd ~/YOUR-REPO/server

# Install dependencies
npm install

# Create environment file
nano .env
```

Paste inside `.env`:
```
PORT=5000
MONGO_URI=mongodb+srv://<username>:<password>@cluster0.xxxxx.mongodb.net/mydb?appName=Cluster0
NODE_ENV=production
```
Save: **Ctrl+X → Y → Enter**

```bash
# Verify .env content
cat .env

# Start backend with PM2
pm2 start index.js --name "mern-backend"

# Save and enable auto-start on reboot
pm2 save
pm2 startup
# Copy and run the command it outputs, e.g.:
# sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu --hp /home/ubuntu
```

---

## STEP 8 — Build Frontend

```bash
cd ~/YOUR-REPO/client

# Install dependencies
npm install

# If build fails due to Node version, reinstall node_modules:
rm -rf node_modules package-lock.json
npm install

# Build for production
npm run build
# Output will be in dist/ folder
```

> ⚠️ **Important:** Update your API base URL in the React code before building.
> Change `http://localhost:5000/api/...` to `http://<YOUR-PUBLIC-IP>/api/...`
>
> Quick fix with sed:
> ```bash
> sed -i 's|http://localhost:5000|http://<YOUR-PUBLIC-IP>|g' src/App.jsx
> ```
> Then rebuild: `npm run build`

---

## STEP 9 — Configure Nginx

```bash
sudo nano /etc/nginx/sites-available/mern-app
```

Paste this config:
```nginx
server {
    listen 80;
    server_name <YOUR-PUBLIC-IP>;

    # Serve React frontend
    location / {
        root /home/ubuntu/YOUR-REPO/client/dist;
        index index.html;
        try_files $uri /index.html;
    }

    # Proxy API calls to Express backend
    location /api/ {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Save: **Ctrl+X → Y → Enter**

```bash
# Enable the config
sudo ln -s /etc/nginx/sites-available/mern-app /etc/nginx/sites-enabled/

# Fix permissions
sudo chmod 755 /home/ubuntu

# Test and restart Nginx
sudo nginx -t
sudo systemctl restart nginx
```

---

## STEP 10 — Access Your App 🎉

Open browser and go to:
```
http://<YOUR-PUBLIC-IP>
```

Your MERN app is live!

---

## Useful Commands

```bash
# Check backend status
pm2 status

# View backend logs
pm2 logs mern-backend --lines 30

# View error logs only
pm2 logs mern-backend --err --lines 30

# Restart backend
pm2 restart mern-backend

# Test backend API
curl http://localhost:5000/api/<your-route>

# Restart Nginx
sudo systemctl restart nginx

# Check Nginx status
sudo systemctl status nginx
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| SSH connection timed out | Update EC2 Security Group → SSH rule → set source to `0.0.0.0/0` |
| 500 Internal Server Error | Run `sudo chmod 755 /home/ubuntu` then restart Nginx |
| Failed to load data | Check API URL in React — must match your EC2 public IP |
| MongoDB connection failed | Add `0.0.0.0/0` to MongoDB Atlas → Network Access |
| Build failed (Vite/Node version) | Upgrade to Node 20: `curl -fsSL https://deb.nodesource.com/setup_20.x \| sudo -E bash - && sudo apt install nodejs -y` |
| Port 5000 not responding | Check PM2 logs: `pm2 logs mern-backend --err --lines 30` |

---

## Cost Management

| Resource | Cost |
|----------|------|
| t2.micro (running 24/7) | ~$8.5/month |
| 8 GB gp3 storage | ~$0.64/month |
| MongoDB Atlas M0 | Free forever |

> 💡 **Stop your EC2 instance** when not in use to save money:
> AWS Console → EC2 → Instances → Select instance → Instance State → Stop

---

## Architecture

```
User Browser
     ↓ HTTP (port 80)
  Nginx (reverse proxy)
     ↓                    ↓
React Frontend      /api/* → Express Backend (port 5000)
(dist/ folder)              ↓
                     MongoDB Atlas (cloud DB)
```
