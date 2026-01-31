# 🚀 CI/CD Deployment Guide (SSH + Docker Compose)

This guide shows how to deploy this project using **GitHub Actions**, **SSH**, and **Docker Compose**.  

We separate authentication into two layers:

- **GitHub Actions → Server** 🔑 (SSH key or password)  
- **Server → GitHub** 🖥️ (Deploy key)

---

## 🛠 Architecture Overview

```
GitHub Actions
      |
      |  (SSH_PRIVATE_KEY)
      v
Remote Server
      |
      |  (Server Deploy Key)
      v
GitHub Repository
```

---

## 1️⃣ Passwordless SSH Key (Recommended) 🔑

On the deployment server, create a dedicated SSH key:

```bash
ssh-keygen -t ed25519 -C "deploy-key"
```

- Press **Enter** for the default path  
- Leave **passphrase empty**

View the public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

Add this **public key** to your repository as a **Deploy Key**:

```
Repository → Settings → Deploy keys → Add deploy key
```

Test access from the server:

```bash
ssh -T git@github.com
git pull
```

---

## 2️⃣ SSH with Password (Fallback) 🔒

If you cannot use keys, you can connect with a password.  

> ⚠️ Warning: storing passwords is insecure; use SSH keys whenever possible.

Install `sshpass` on the server:

```bash
sudo apt install -y sshpass
```

Example usage in GitHub Actions:

```bash
sshpass -p "SERVER_PASSWORD" ssh SERVER_USER@SERVER_HOST "
    cd /PATH/TO/PROJECT &&
    git pull origin main &&
    sudo docker compose up -d --build &&
    sudo docker compose logs --tail=50
"
```

---

## 3️⃣ Server → GitHub (Deploy Key) 🖥️

On the server, generate a key specifically for GitHub access:

```bash
ssh-keygen -t ed25519 -C "server-deploy-key"
```

Add the **public key** to the repository’s **Deploy Keys** section.  

Verify:

```bash
ssh -T git@github.com
git pull
```

---

## 4️⃣ GitHub Actions Workflow Example ⚡

```yaml
name: Deploy

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Deploy via SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan -H ${{ secrets.SERVER_HOST }} >> ~/.ssh/known_hosts

          ssh ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }} "
            cd /PATH/TO/PROJECT &&
            git pull origin main &&
            sudo docker compose up -d --build &&
            sleep 5 &&
            sudo docker compose logs --tail=50
          "
```

> For password SSH, replace the `ssh` command with `sshpass -p "${{ secrets.SERVER_PASSWORD }}" ssh ...`

---

## 5️⃣ Docker Sudo on Server 🐳

Allow the deployment user to run Docker without a password:

```bash
sudo visudo
```

Add:

```
SERVER_USER ALL=(ALL) NOPASSWD: /usr/bin/docker, /usr/bin/docker compose
```

---

## 6️⃣ Logging & Verification 📊

After deployment:

```bash
docker compose ps
docker compose logs -f
```

---

## 7️⃣ Security Notes 🔐

- Always prefer **SSH keys** over passwords  
- Use a **dedicated deploy key** per server  
- Rotate keys periodically  
- Avoid storing passwords in plaintext  
- Limit write access unless necessary

---

## 8️⃣ Summary 📝

| Direction | Authentication |
|------------|----------------|
| GitHub Actions → Server | 🔑 SSH key (passwordless) or 🔒 password fallback |
| Server → GitHub | 🖥️ Repository Deploy Key |
| Server → Docker | 🐳 Passwordless sudo |

This setup provides a **secure, repeatable, and easy-to-use deployment workflow**.
