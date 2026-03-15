# Woodpecker CI Setup

Woodpecker CI is a lightweight CI/CD engine that integrates natively with Forgejo. It runs pipelines defined in `.woodpecker/` directory in your repositories.

## Prerequisites

- Forgejo running and accessible at `http://<SERVER_IP>:3000` (see [Forgejo Setup](forgejo-setup.md))
- Docker and Docker Compose on the test server

## Architecture

```
┌─────────────┐     webhook      ┌──────────────────┐     gRPC      ┌──────────────────┐
│   Forgejo    │ ───────────────> │ Woodpecker Server│ ────────────> │ Woodpecker Agent │
│  :3000       │                  │  :8000           │               │ (runs pipelines) │
└─────────────┘                  └──────────────────┘               └──────────────────┘
                                                                            │
                                                                            v
                                                                    ┌──────────────────┐
                                                                    │ Docker Registry  │
                                                                    │  :5000           │
                                                                    └──────────────────┘
```

- **Woodpecker Server** — receives webhooks from Forgejo, schedules pipelines
- **Woodpecker Agent** — picks up jobs, runs pipeline steps as Docker containers
- **Docker Registry** — stores built images locally (no Docker Hub needed)

## Step 1: Create an OAuth2 Application in Forgejo

Woodpecker authenticates with Forgejo via OAuth2.

1. Log into Forgejo at **http://<SERVER_IP>:3000**
2. Go to **Site Administration → Applications** (or **User Settings → Applications** if not admin)
3. Click **Create a new OAuth2 Application**

| Field | Value |
|-------|-------|
| Application Name | Woodpecker CI |
| Redirect URI | http://<SERVER_IP>:8000/authorize |

4. Click **Create Application**
5. Copy the **Client ID** and **Client Secret** — you'll need these next

## Step 2: Configure the .env File

On the test server, edit the `.env` file in the `travelpro-cicd` directory:

```bash
cd ~/travelpro-cicd
nano .env
```

Fill in the values:

```env
WOODPECKER_AGENT_SECRET=<generate with: openssl rand -hex 32>
WOODPECKER_FORGEJO_CLIENT=<Client ID from Step 1>
WOODPECKER_FORGEJO_SECRET=<Client Secret from Step 1>
```

Generate the agent secret:

```bash
openssl rand -hex 32
```

## Step 3: Start All Services

```bash
docker compose up -d
```

Verify all containers are running:

```bash
docker compose ps
```

You should see: `forgejo`, `woodpecker-server`, `woodpecker-agent`, `docker-registry` — all `Up`.

## Step 4: Log into Woodpecker

1. Open **http://<SERVER_IP>:8000**
2. Click **Login** — you'll be redirected to Forgejo to authorise
3. Approve the OAuth2 request
4. You're now logged into Woodpecker

## Step 5: Activate Your Repository

1. In the Woodpecker UI, click **+ Add repository** (or go to Repositories)
2. Find `travelpro_catalog` and click **Activate**
3. Woodpecker will automatically create a webhook in Forgejo

### Fix the Webhook URL

The auto-created webhook uses the external IP (`<SERVER_IP>`), but Forgejo runs inside Docker and needs to reach Woodpecker via the Docker service name.

1. Go to Forgejo → repo → **Settings → Webhooks**
2. Click the pencil icon on the Woodpecker webhook
3. Change the Target URL from `http://<SERVER_IP>:8000/api/hook?...` to `http://woodpecker-server:8000/api/hook?...`
4. Save and click **Test Delivery** to confirm it works

### Enable Webhook Delivery to Private Hosts

Forgejo blocks webhooks to private/internal addresses by default. The `docker-compose.yml` includes `FORGEJO__webhook__ALLOWED_HOST_LIST: "*"` to allow this. If webhooks fail with a delivery error, verify this environment variable is set on the Forgejo service.

## Step 6: Configure the Docker Registry as Insecure (Local Only)

Since the registry runs on HTTP (not HTTPS), Docker needs to trust it. On the test server:

```bash
sudo nano /etc/docker/daemon.json
```

Add:

```json
{
  "insecure-registries": ["<SERVER_IP>:5000"]
}
```

Restart Docker:

```bash
sudo systemctl restart docker

# Restart all CI/CD services after Docker restart
cd ~/travelpro-cicd
docker compose up -d
```

## Step 7: Create a Pipeline in Your Catalog Repo

Create a `.woodpecker/` directory in the repo root and add a pipeline file (e.g. `.woodpecker/travelpro_catalog.yml`).

> **Note:** Woodpecker v2+ looks for pipelines in the `.woodpecker/` directory, not a `.woodpecker.yml` file at the root.

```yaml
when:
  branch: main
  event: push

steps:
  - name: test
    image: maven:3.9-eclipse-temurin-25
    commands:
      - mvn clean test -q

  - name: build-image
    image: plugins/docker
    settings:
      repo: <SERVER_IP>:5000/travelpro/catalog-api
      registry: <SERVER_IP>:5000
      insecure: true
      tags:
        - latest
        - "${CI_COMMIT_SHA:0:8}"

  - name: deploy
    image: appleboy/drone-ssh
    settings:
      host: <SERVER_IP>
      username: <DEPLOY_USER>
      key:
        from_secret: deploy_ssh_key
      script:
        - cd /home/<DEPLOY_USER>/travelpro_catalog
        - docker compose pull catalog-api
        - docker compose up -d catalog-api
```

> **Important:** The `insecure: true` flag on `build-image` is required because the `plugins/docker` step runs its own Docker daemon (Docker-in-Docker) which doesn't inherit the host's `insecure-registries` config.

### Pipeline Explanation

| Step | What it does |
|------|-------------|
| **test** | Runs `mvn clean test` — fails the pipeline if tests fail |
| **build-image** | Builds the Docker image and pushes to the private registry |
| **deploy** | SSHes into the server, pulls the new image, and restarts the API container |

## Step 8: Add the Deploy SSH Key as a Secret

The deploy step needs SSH access to the server. In Woodpecker:

1. Go to your repository settings in Woodpecker UI
2. Click **Secrets**
3. Add a new secret:

| Field | Value |
|-------|-------|
| Name | `deploy_ssh_key` |
| Value | Contents of the private SSH key (see below) |
| Events | ☑ Push |

The private key must correspond to a public key in `<DEPLOY_USER>`'s `~/.ssh/authorized_keys` on the server. To get the key contents:

```bash
# On the machine where the key lives (e.g. your local machine)
cat ~/.ssh/id_ed25519
```

Paste the entire output including `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----`.

## Step 9: Update the Catalog docker-compose.yml for Registry Images

Once the pipeline is pushing images, update `travelpro_catalog/docker-compose.yml` on the server to pull from the registry instead of building locally:

```yaml
  catalog-api:
    container_name: catalog-api
    image: <SERVER_IP>:5000/travelpro/catalog-api:latest  # <-- replaces "build: ."
    restart: unless-stopped
    ...
```

## Step 10: Test the Pipeline

Push a commit to `main`:

```bash
cd C:\Users\Admin\Documents\repo\travelpro_catalog
git add .
git commit -m "Add CI/CD pipeline"
git push forgejo main
```

Watch the pipeline run at **http://<SERVER_IP>:8000**. All four stages should pass: clone → test → build-image → deploy.

## Useful Commands

```bash
# View Woodpecker logs
docker compose logs -f woodpecker-server woodpecker-agent

# List images in the registry
curl http://<SERVER_IP>:5000/v2/_catalog

# List tags for an image
curl http://<SERVER_IP>:5000/v2/travelpro/catalog-api/tags/list

# Restart all CI/CD services
docker compose restart
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Woodpecker shows no repos | Click "Add repository" and sync — ensure OAuth2 redirect URI is correct |
| `pipeline definition not found` | Ensure pipeline file is in `.woodpecker/` directory, not at repo root |
| `http: server gave HTTP response to HTTPS client` | Add `insecure: true` to the `build-image` step settings |
| Webhook delivery fails (red dot) | Change webhook URL to use `woodpecker-server:8000` instead of the external IP |
| Webhook blocked by Forgejo | Add `FORGEJO__webhook__ALLOWED_HOST_LIST: "*"` to Forgejo environment |
| `secret not found` | Add the secret in Woodpecker UI → repo → Settings → Secrets |
| Pipeline fails at deploy | Verify the `deploy_ssh_key` secret is set and the public key is in `<DEPLOY_USER>`'s `authorized_keys` |
| Agent not picking up jobs | Check `WOODPECKER_AGENT_SECRET` matches between server and agent |
