# Forgejo Setup

Forgejo is a lightweight, self-hosted Git platform (fork of Gitea). It runs as a Docker container on the test server.

## Prerequisites

- Docker and Docker Compose installed on `192.168.1.77`
- Port 3000 (web) and 2222 (SSH) available

## Step 1: Start Forgejo

```bash
ssh deployguy@192.168.1.77

# Copy this project to the server
mkdir -p ~/travelpro-cicd && cd ~/travelpro-cicd

# Copy .env.example to .env (fill in later when setting up Woodpecker)
cp .env.example .env

# Start Forgejo only for now
docker compose up -d forgejo
```

## Step 2: Initial Configuration

Open **http://192.168.1.77:3000** in your browser. You'll see the installation wizard.

### Database Settings

| Setting | Value |
|---------|-------|
| Database Type | SQLite3 |

### General Settings

| Setting | Value |
|---------|-------|
| Site Title | Abeona Platform |
| Repository Root Path | (leave default) |
| SSH Server Domain | 192.168.1.77 |
| SSH Server Port | 2222 |
| Forgejo Base URL | http://192.168.1.77:3000/ |

### Admin Account

Create an admin account — this will be used to manage repos and configure Woodpecker integration.

Click **Install Forgejo**.

## Step 3: Add Your SSH Key

1. Go to **Settings → SSH / GPG Keys** (top-right avatar → Settings)
2. Click **Add Key**
3. Paste your public key (from your Windows machine: `cat ~/.ssh/id_ed25519.pub` or `cat ~/.ssh/id_rsa.pub`)

## Step 4: Create the Catalog Repository

1. Click **+** → **New Repository**
2. Repository name: `travelpro_catalog`
3. Visibility: Private
4. Do NOT initialise with README (you'll push existing code)
5. Click **Create Repository**

## Step 5: Push Your Code

From your **Windows machine**:

```bash
cd C:\Users\Admin\Documents\repo\travelpro_catalog

# Add Forgejo as a remote
git remote add forgejo ssh://git@192.168.1.77:2222/<your-username>/travelpro_catalog.git

# Push all branches
git push forgejo --all
```

Or using HTTP:

```bash
git remote add forgejo http://192.168.1.77:3000/<your-username>/travelpro_catalog.git
git push forgejo --all
```

## Step 6: Verify

Browse to **http://192.168.1.77:3000/\<your-username\>/travelpro_catalog** — you should see your code, commits, and branches.

## Useful Commands

```bash
# View Forgejo logs
docker compose logs -f forgejo

# Restart Forgejo
docker compose restart forgejo

# Stop Forgejo
docker compose stop forgejo

# Backup Forgejo data
docker run --rm -v travelpro-cicd_forgejo-data:/data -v $(pwd):/backup alpine tar czf /backup/forgejo-backup.tar.gz /data
```

## Important: Webhook Configuration

When Woodpecker CI is set up later, it creates webhooks in Forgejo. Two things to be aware of:

1. **Allowed hosts** — Forgejo blocks webhooks to private/internal addresses by default. The `docker-compose.yml` includes `FORGEJO__webhook__ALLOWED_HOST_LIST: "*"` to allow this.

2. **Webhook URL** — Woodpecker auto-creates webhooks using the external IP, but Forgejo needs to reach Woodpecker via the Docker service name (`woodpecker-server:8000`). You'll need to manually edit the webhook URL in Forgejo after activating a repo in Woodpecker. See [Woodpecker Setup — Step 5](woodpecker-setup.md#step-5-activate-your-repository) for details.

## What's Next

Once Forgejo is running and your code is pushed, proceed to [Woodpecker CI Setup](woodpecker-setup.md).
