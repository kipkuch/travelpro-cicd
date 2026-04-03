# CI/CD Setup Guide

This guide covers setting up Forgejo, Woodpecker CI, the private Docker registry, and the promote-to-production pipeline on the test server.

## Architecture

```
┌─────────────┐   webhook      ┌──────────────────┐   gRPC         ┌──────────────────┐
│   Forgejo    │ ─────────────> │ Woodpecker Server│ ─────────────> │ Woodpecker Agent │
│  :3000       │                │  :8000           │                │ (runs pipelines) │
└─────────────┘                └──────────────────┘                └────────┬─────────┘
                                       │                                    │
                                       │ manual trigger              build & push
                                       v                                    v
                                ┌──────────────┐                   ┌──────────────────┐
                                │ Promote      │                   │ Private Registry │
                                │ Pipeline     │                   │  :5000           │
                                └──────┬───────┘                   └──────────────────┘
                                       │ docker push
                                       v
                                ┌──────────────────┐
                                │ Docker Hub       │
                                │ kipkuch/tourpro  │
                                └────────┬─────────┘
                                         │ docker pull
                                         v
                                ┌──────────────────┐
                                │ Netcup VPS       │
                                │ (production)     │
                                └──────────────────┘
```

---

## 1. Forgejo

### 1.1 Start Forgejo

```bash
ssh <DEPLOY_USER>@<SERVER_IP>
mkdir -p ~/travelpro-cicd && cd ~/travelpro-cicd
cp .env.example .env
docker compose up -d forgejo
```

### 1.2 Initial Configuration

Open **http://\<SERVER_IP\>:3000** and complete the installation wizard:

| Setting | Value |
|---------|-------|
| Database Type | SQLite3 |
| Site Title | Abeona Platform |
| SSH Server Domain | \<SERVER_IP\> |
| SSH Server Port | 2222 |
| Forgejo Base URL | http://\<SERVER_IP\>:3000/ |

Create an admin account, then click **Install Forgejo**.

### 1.3 Add Your SSH Key

**Settings → SSH / GPG Keys → Add Key** — paste your public key.

### 1.4 Create Repositories and Push Code

```bash
# From your local machine
git remote add forgejo ssh://git@<SERVER_IP>:2222/<username>/travelpro_catalog.git
git push forgejo --all
```

---

## 2. Woodpecker CI

### 2.1 Create OAuth2 App in Forgejo

**Site Administration → Applications → Create a new OAuth2 Application**

| Field | Value |
|-------|-------|
| Application Name | Woodpecker CI |
| Redirect URI | http://\<SERVER_IP\>:8000/authorize |

Copy the **Client ID** and **Client Secret**.

### 2.2 Configure .env

```bash
cd ~/travelpro-cicd && nano .env
```

```env
WOODPECKER_AGENT_SECRET=<openssl rand -hex 32>
WOODPECKER_FORGEJO_CLIENT=<Client ID>
WOODPECKER_FORGEJO_SECRET=<Client Secret>
WOODPECKER_ADMIN=<your-forgejo-username>
```

### 2.3 Start All Services

```bash
docker compose up -d
docker compose ps
# Expected: forgejo, woodpecker-server, woodpecker-agent, docker-registry — all Up
```

### 2.4 Log into Woodpecker

Open **http://\<SERVER_IP\>:8000**, click **Login**, and approve the OAuth2 request in Forgejo.

### 2.5 Activate Repositories

In Woodpecker UI: **+ Add repository** → find and activate each repo.

**Fix webhook URL** — in Forgejo, go to each repo → **Settings → Webhooks** → edit the Woodpecker webhook:
- Change `http://<SERVER_IP>:8000/api/hook?...` → `http://woodpecker-server:8000/api/hook?...`
- Save and **Test Delivery**

### 2.6 Configure Insecure Registry

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
  "insecure-registries": ["<SERVER_IP>:5000"]
}
```

```bash
sudo systemctl restart docker
cd ~/travelpro-cicd && docker compose up -d
```

### 2.7 Add Deploy SSH Key Secret

In Woodpecker UI → repo → **Settings → Secrets**:

| Name | Value | Events |
|------|-------|--------|
| `deploy_ssh_key` | Contents of private SSH key (`cat ~/.ssh/id_ed25519`) | ☑ Push |

The corresponding public key must be in `<DEPLOY_USER>`'s `~/.ssh/authorized_keys` on the server.

### 2.8 CI Pipeline — `.woodpecker/build.yml`

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
      repo: <SERVER_IP>:5000/tourpro/catalog-api
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
        - cd /home/<DEPLOY_USER>/tourpro_catalog
        - docker compose pull catalog-api
        - docker compose up -d catalog-api
```

### 2.9 Update Catalog docker-compose for Registry Images

Once the pipeline is pushing images, update the server's `tourpro_catalog/docker-compose.yml`:

```yaml
catalog-api:
  image: <SERVER_IP>:5000/tourpro/catalog-api:latest  # replaces "build: ."
```

### 2.10 Test the Pipeline

```bash
git add . && git commit -m "Add CI/CD pipeline" && git push forgejo main
```

Watch at **http://\<SERVER_IP\>:8000** — all stages should pass: clone → test → build-image → deploy.

---

## 3. Promote Pipeline (Test → Docker Hub)

### 3.1 Docker Hub Tagging Convention

All projects share a single Docker Hub repo: **`kipkuch/tourpro`**.

| Project | Private Registry Image | Docker Hub Tags |
|---------|----------------------|-----------------|
| Catalog API | `192.168.1.77:5000/tourpro/catalog-api` | `catalog-api-latest`, `catalog-api-{sha}` |
| Booking API | `192.168.1.77:5000/tourpro/booking-api` | `booking-api-latest`, `booking-api-{sha}` |
| Catalog UI | `192.168.1.77:5000/abeona-travel` | `catalog-ui-latest`, `catalog-ui-{sha}` |
| Backoffice UI | `192.168.1.77:5000/abeona/backoffice-ui` | `backoffice-ui-latest`, `backoffice-ui-{sha}` |

### 3.2 Add Docker Hub Secrets

In Woodpecker UI → org or repo → **Settings → Secrets**:

| Name | Value | Events |
|------|-------|--------|
| `dockerhub_username` | `kipkuch` | ☑ Manual |
| `dockerhub_password` | Docker Hub access token | ☑ Manual |

### 3.3 Promote Pipeline — `.woodpecker/promote.yml`

Each repo gets a `promote.yml` — only the `REGISTRY_IMAGE` and tag prefix differ.

**Catalog API:**

```yaml
when:
  - event: manual

steps:
  - name: promote-to-dockerhub
    image: docker:27
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      DOCKERHUB_USERNAME:
        from_secret: dockerhub_username
      DOCKERHUB_PASSWORD:
        from_secret: dockerhub_password
    commands:
      - COMMIT_SHA=$(echo $CI_COMMIT_SHA | cut -c1-8)
      - REGISTRY_IMAGE="192.168.1.77:5000/tourpro/catalog-api"
      - DOCKERHUB_IMAGE="kipkuch/tourpro"
      - echo "Promoting $REGISTRY_IMAGE:latest → $DOCKERHUB_IMAGE:catalog-api-latest + catalog-api-$COMMIT_SHA"
      - docker pull $REGISTRY_IMAGE:latest
      - docker tag $REGISTRY_IMAGE:latest $DOCKERHUB_IMAGE:catalog-api-latest
      - docker tag $REGISTRY_IMAGE:latest $DOCKERHUB_IMAGE:catalog-api-$COMMIT_SHA
      - echo "$DOCKERHUB_PASSWORD" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin
      - docker push $DOCKERHUB_IMAGE:catalog-api-latest
      - docker push $DOCKERHUB_IMAGE:catalog-api-$COMMIT_SHA
      - docker logout
      - echo "Done — promoted as catalog-api-latest and catalog-api-$COMMIT_SHA"
```

**Booking API:** Same structure — change `REGISTRY_IMAGE` to `192.168.1.77:5000/tourpro/booking-api` and tag prefix to `booking-api`.

**Catalog UI:** Same structure — change `REGISTRY_IMAGE` to `192.168.1.77:5000/abeona-travel` and tag prefix to `catalog-ui`.

**Backoffice UI:** Same structure — change `REGISTRY_IMAGE` to `192.168.1.77:5000/abeona/backoffice-ui` and tag prefix to `backoffice-ui`.

> Repos must have **Trusted** status in Woodpecker for volume mounts to work (admin enables this in repo settings).

### 3.4 Trigger a Promote

1. Woodpecker UI → select repo → click **+** (New pipeline)
2. Select branch (`main`) → click **Run**

### 3.5 Deploy on Production (Netcup VPS)

```bash
ssh deployguy@159.195.37.110

# Pull and restart whichever surface was promoted
cd ~/tourpro_catalog && docker compose pull && docker compose up -d
cd ~/tourpro_booking && docker compose pull && docker compose up -d
cd ~/abeona/catalog-ui && docker compose pull && docker compose up -d
cd ~/abeona/backoffice-ui && docker compose pull && docker compose up -d
```

---

## Useful Commands

```bash
# Forgejo logs
docker compose logs -f forgejo

# Woodpecker logs
docker compose logs -f woodpecker-server woodpecker-agent

# List images in private registry
curl http://<SERVER_IP>:5000/v2/_catalog

# List tags for an image
curl http://<SERVER_IP>:5000/v2/tourpro/catalog-api/tags/list

# Restart all CI/CD services
cd ~/travelpro-cicd && docker compose restart
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Woodpecker shows no repos | Click "Add repository" and sync — ensure OAuth2 redirect URI is correct |
| `pipeline definition not found` | Ensure pipeline file is in `.woodpecker/` directory, not at repo root |
| `http: server gave HTTP response to HTTPS client` | Add `insecure: true` to the `build-image` step settings |
| Webhook delivery fails | Change webhook URL to use `woodpecker-server:8000` instead of external IP |
| Webhook blocked by Forgejo | Verify `FORGEJO__webhook__ALLOWED_HOST_LIST: "*"` is set on Forgejo service |
| `secret not found` | Add the secret in Woodpecker UI → repo → Settings → Secrets |
| Secret not allowed for event | Edit the secret and enable the correct event checkbox (e.g. Manual) |
| Pipeline fails at deploy | Verify `deploy_ssh_key` secret is set and public key is in `authorized_keys` |
| Agent not picking up jobs | Check `WOODPECKER_AGENT_SECRET` matches between server and agent |
