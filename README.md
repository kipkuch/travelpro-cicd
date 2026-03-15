# TravelPro CI/CD Infrastructure

This project contains the Docker Compose configuration and setup guides for the Abeona platform's CI/CD infrastructure running on the test server (`192.168.1.77`).

## Components

| Service | Port | Purpose |
|---------|------|---------|
| Forgejo | 3000 (web), 2222 (SSH) | Self-hosted Git platform |
| Woodpecker CI Server | 8000 | CI/CD pipeline server |
| Woodpecker CI Agent | — | Executes pipeline steps |
| Docker Registry | 5000 | Private container image registry |

## Setup Guides

1. [Forgejo Setup](docs/forgejo-setup.md) — do this first
2. [Woodpecker CI Setup](docs/woodpecker-setup.md) — requires Forgejo to be running

## Quick Start

```bash
# Copy .env.example to .env and fill in values
cp .env.example .env

# Start all infrastructure services
docker compose up -d

# Check status
docker compose ps

# View logs
docker compose logs -f
```

## Project Layout

```
docker-compose.yml      # All infrastructure services
.env.example            # Template for secrets
docs/
  forgejo-setup.md      # Forgejo installation and config guide
  woodpecker-setup.md   # Woodpecker CI setup and pipeline guide
```
