# Production Deployment Plan — Netcup VPS

Deploys four surfaces (Catalog API, Booking API, Backoffice UI, Catalog UI) on the Netcup VPS.

| Detail | Value |
|--------|-------|
| VPS IP | `159.195.37.110` |
| Default hostname | `v2202603344923441589.happysrv.de` |
| Target domain | `abeonatravel.africa` (when DNS is configured) |
| SSH access | `ssh deployguy@159.195.37.110` |
| Docker Hub repo | `kipkuch/tourpro` |

---

## Step 0: Point Your Domain at the VPS

> **Do this first** if you want HTTPS from the start. If you'd rather get everything running on HTTP first and add TLS later, skip to Step 1 and come back to this.

### Option A: Use your own domain (`abeonatravel.africa`)

1. Log into your domain registrar (wherever you bought `abeonatravel.africa`)
2. Go to **DNS Management**
3. Add an **A record**:

| Type | Name | Value | TTL |
|------|------|-------|-----|
| A | `@` | `159.195.37.110` | 300 (or Auto) |

4. If you also want `www.abeonatravel.africa`:

| Type | Name | Value | TTL |
|------|------|-------|-----|
| CNAME | `www` | `abeonatravel.africa` | 300 |

5. Wait for DNS propagation (usually 5–30 minutes, can take up to 48 hours):

```bash
# Check from your local machine
nslookup abeonatravel.africa
# Should return 159.195.37.110
```

### Option B: Use the default Netcup hostname

The hostname `v2202603344923441589.happysrv.de` already points to your VPS IP. You can use this immediately — no DNS changes needed. However, Let's Encrypt may refuse to issue a certificate for a `*.happysrv.de` subdomain you don't own. If that happens, you'll need to use Option A.

### Which to choose?

- **Setting up the domain first** means you get HTTPS right away and don't have to reconfigure URLs later
- **Skipping the domain for now** means you can test everything over HTTP immediately

Either way, all the steps below use `<DOMAIN>` as a placeholder — substitute your chosen hostname.

---

## Step 1: Server Prep

SSH in as root and run:

```bash
# Update system
apt update && apt upgrade -y

# Install essentials
apt install -y curl git ufw

# Create deploy user (skip if deployguy already exists)
adduser deployguy
usermod -aG sudo deployguy

# Install Docker
curl -fsSL https://get.docker.com | sh
usermod -aG docker deployguy

# Install Nginx
apt install -y nginx

# Install Certbot (for TLS later)
apt install -y certbot python3-certbot-nginx

# Firewall
ufw allow OpenSSH
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable

# Switch to deploy user for the rest
su - deployguy
```

### Log into Docker Hub

The images are on Docker Hub, so the deploy user needs to authenticate:

```bash
docker login -u kipkuch
# Enter your Docker Hub access token when prompted
```

---

## Step 2: Create Directory Structure

```bash
# Catalog API surface
mkdir -p ~/tourpro_catalog/{db-data,images}

# Booking API surface
mkdir -p ~/tourpro_booking/db-data

# Backoffice UI surface
mkdir -p ~/abeona/backoffice-ui

# Catalog UI surface
mkdir -p ~/abeona/catalog-ui
```

---

## Step 3: Deploy Catalog API Surface

```bash
cd ~/tourpro_catalog
```

Create `docker-compose.yml`:

```yaml
services:
  catalog-db:
    container_name: catalog-db
    image: postgres:17-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: tourpro_catalog
      POSTGRES_USER: tourpro_catalog_user
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - ./db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U tourpro_catalog_user -d tourpro_catalog"]
      interval: 5s
      timeout: 5s
      retries: 5

  catalog-api:
    container_name: catalog-api
    image: kipkuch/tourpro:catalog-api-latest
    restart: unless-stopped
    depends_on:
      catalog-db:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: prod
      SPRING_DATASOURCE_URL: jdbc:postgresql://catalog-db:5432/tourpro_catalog
      SPRING_DATASOURCE_USERNAME: tourpro_catalog_user
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
      IMAGES_BASE_URL: ${IMAGES_BASE_URL}
    volumes:
      - ./images:/tourpro-images
    ports:
      - "127.0.0.1:8080:8080"
```

Create `.env`:

```env
DB_PASSWORD=<strong-password>
IMAGES_BASE_URL=https://<DOMAIN>/images
```

> Replace `<DOMAIN>` with `abeonatravel.africa` or `v2202603344923441589.happysrv.de`.
> Use `http://` instead of `https://` if you haven't set up TLS yet.

---

## Step 4: Deploy Booking API Surface

```bash
cd ~/tourpro_booking
```

Create `docker-compose.yml`:

```yaml
services:
  booking-db:
    container_name: booking-db
    image: postgres:17-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: tourpro_booking
      POSTGRES_USER: tourpro_booking_user
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - ./db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U tourpro_booking_user -d tourpro_booking"]
      interval: 5s
      timeout: 5s
      retries: 5

  booking-api:
    container_name: booking-api
    image: kipkuch/tourpro:booking-api-latest
    restart: unless-stopped
    depends_on:
      booking-db:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: prod
      SPRING_DATASOURCE_URL: jdbc:postgresql://booking-db:5432/tourpro_booking
      SPRING_DATASOURCE_USERNAME: tourpro_booking_user
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
    ports:
      - "127.0.0.1:8081:8080"
```

Create `.env`:

```env
DB_PASSWORD=<strong-password>
```

> Use a different password than the catalog API database.

---

## Step 5: Deploy Backoffice UI Surface

```bash
cd ~/abeona/backoffice-ui
```

Create `docker-compose.yml`:

```yaml
services:
  backoffice:
    container_name: backoffice
    image: kipkuch/tourpro:backoffice-ui-latest
    restart: unless-stopped
    ports:
      - "127.0.0.1:4002:3001"
    environment:
      - NODE_ENV=production
      - API_BACKEND_URL=${API_BACKEND_URL}
      - API_PATH_PREFIX=${API_PATH_PREFIX:-/api/catalog}
      - AUTH_USERNAME=${AUTH_USERNAME}
      - AUTH_PASSWORD=${AUTH_PASSWORD}
```

Create `.env`:

```env
AUTH_USERNAME=admin
AUTH_PASSWORD=<strong-password>
API_BACKEND_URL=https://<DOMAIN>
API_PATH_PREFIX=/api/catalog
```

---

## Step 6: Deploy Catalog UI Surface

```bash
cd ~/abeona/catalog-ui
```

Create `docker-compose.yml`:

```yaml
services:
  catalog-ui:
    container_name: catalog-ui
    image: kipkuch/tourpro:catalog-ui-latest
    restart: unless-stopped
    ports:
      - "127.0.0.1:4000:4000"
    environment:
      - NODE_ENV=production
```

---

## Step 7: Configure Host Nginx

Create the site config:

```bash
sudo nano /etc/nginx/sites-available/abeona
```

Paste the following (replace `<DOMAIN>` with your hostname):

```nginx
server {
    listen 80;
    server_name <DOMAIN>;

    # Serve static images
    location /images/ {
        alias /home/deployguy/tourpro_catalog/images/;
        autoindex off;
        expires 30d;
        add_header Cache-Control "public, immutable";

        limit_except GET HEAD {
            deny all;
        }
    }

    # Proxy API requests (clean URLs)
    location /api/catalog/ {
        proxy_pass http://localhost:8080/api/v1/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Booking API
    location /api/booking/ {
        proxy_pass http://localhost:8081/api/v1/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # --- Swagger UI (optional, remove to disable) ---
    location /swagger-ui/ {
        proxy_pass http://localhost:8080/swagger-ui/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /v3/api-docs {
        proxy_pass http://localhost:8080/v3/api-docs;
        proxy_set_header Host $host;
    }

    location /swagger/api/ {
        proxy_pass http://localhost:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    # --- End Swagger UI ---

    # Backoffice UI (Next.js, basePath=/backoffice)
    location /backoffice {
        proxy_pass http://localhost:4002;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        client_max_body_size 20M;
    }

    # Health check
    location /actuator/health {
        proxy_pass http://localhost:8080/actuator/health;
    }

    # Catalog UI - catch-all (must be last)
    location / {
        proxy_pass http://localhost:4000;
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

Enable the site and test:

```bash
sudo ln -sf /etc/nginx/sites-available/abeona /etc/nginx/sites-enabled/abeona
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

---

## Step 8: File Permissions

Nginx (`www-data`) needs read access to serve images:

```bash
sudo chmod 755 /home/deployguy
chmod 755 /home/deployguy/tourpro_catalog
chmod 755 /home/deployguy/tourpro_catalog/images
```

For any uploaded images, ensure files are `644` and directories are `755`:

```bash
find ~/tourpro_catalog/images -type d -exec chmod 755 {} \;
find ~/tourpro_catalog/images -type f -exec chmod 644 {} \;
```

---

## Step 9: Start Everything

```bash
cd ~/tourpro_catalog && docker compose up -d
cd ~/tourpro_booking && docker compose up -d
cd ~/abeona/backoffice-ui && docker compose up -d
cd ~/abeona/catalog-ui && docker compose up -d
```

Verify all containers are running:

```bash
docker ps
```

Test from outside (replace `<DOMAIN>` with your hostname):

```
http://<DOMAIN>/                         → Catalog UI
http://<DOMAIN>/backoffice               → Backoffice UI
http://<DOMAIN>/api/catalog/             → Catalog API
http://<DOMAIN>/api/booking/             → Booking API
http://<DOMAIN>/images/<pkg>/hero.jpg    → Static images
http://<DOMAIN>/swagger-ui/index.html    → Swagger UI (optional)
http://<DOMAIN>/actuator/health          → Health check
```

---

## Step 10: Enable HTTPS with Certbot

> **Prerequisite:** Your domain must resolve to `159.195.37.110` before running Certbot. Verify with `nslookup <DOMAIN>`.

```bash
sudo certbot --nginx -d <DOMAIN>
```

Certbot will:
1. Verify you own the domain (HTTP-01 challenge via port 80)
2. Obtain a Let's Encrypt certificate
3. Modify the Nginx config to add `listen 443 ssl` with the cert paths
4. Add an HTTP → HTTPS redirect on port 80
5. Set up automatic renewal (via a systemd timer)

### After Certbot runs

Certbot modifies the Nginx config automatically. Verify it looks correct:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Test HTTPS:

```
https://<DOMAIN>/actuator/health
https://<DOMAIN>/
```

### Update .env files for HTTPS

After TLS is working, make sure all `.env` files use `https://`:

```bash
# ~/tourpro_catalog/.env
IMAGES_BASE_URL=https://<DOMAIN>/images

# ~/abeona/backoffice-ui/.env
API_BACKEND_URL=https://<DOMAIN>
```

Then restart the affected containers:

```bash
cd ~/tourpro_catalog && docker compose up -d
cd ~/abeona/backoffice-ui && docker compose up -d
```

### Verify auto-renewal

```bash
sudo certbot renew --dry-run
```

---

## Step 11: Updating Deployments

After running the promote pipeline in Woodpecker (which pushes new images to Docker Hub), deploy the updates on the VPS:

```bash
# Deploy a single surface
cd ~/tourpro_catalog && docker compose pull && docker compose up -d

# Or all four
cd ~/tourpro_catalog && docker compose pull && docker compose up -d
cd ~/tourpro_booking && docker compose pull && docker compose up -d
cd ~/abeona/backoffice-ui && docker compose pull && docker compose up -d
cd ~/abeona/catalog-ui && docker compose pull && docker compose up -d
```

---

## Checklist

- [ ] Domain DNS A record points to `159.195.37.110` (or using default hostname)
- [ ] Server updated and essentials installed
- [ ] `deployguy` user created with Docker access
- [ ] Docker and Nginx installed
- [ ] `docker login -u kipkuch` completed
- [ ] Firewall configured (SSH, 80, 443)
- [ ] Directory structure created
- [ ] Catalog API `docker-compose.yml` and `.env` created
- [ ] Booking API `docker-compose.yml` and `.env` created
- [ ] Backoffice UI `docker-compose.yml` and `.env` created
- [ ] Catalog UI `docker-compose.yml` created
- [ ] Nginx config created and enabled
- [ ] File permissions set for Nginx image serving
- [ ] All four surfaces running (`docker ps`)
- [ ] Catalog UI accessible at `http://<DOMAIN>/`
- [ ] Backoffice UI accessible at `http://<DOMAIN>/backoffice`
- [ ] Catalog API accessible at `http://<DOMAIN>/api/catalog/tourpackages/summary`
- [ ] Booking API accessible at `http://<DOMAIN>/api/booking/`
- [ ] Images served at `http://<DOMAIN>/images/`
- [ ] Health check at `http://<DOMAIN>/actuator/health`
- [ ] Certbot TLS configured (`https://<DOMAIN>/`)
- [ ] `.env` files updated with `https://` URLs
- [ ] `certbot renew --dry-run` passes
