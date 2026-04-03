# TourPro CI/CD Infrastructure

This project contains the Docker Compose configuration and setup guides for the Abeona platform's CI/CD infrastructure running on the test server (`<SERVER_IP>`).

## Components

| Service | Port | Purpose |
|---------|------|---------|
| Forgejo | 3000 (web), 2222 (SSH) | Self-hosted Git platform |
| Woodpecker CI Server | 8000 | CI/CD pipeline server |
| Woodpecker CI Agent | — | Executes pipeline steps |
| Docker Registry | 5000 | Private container image registry |

## Setup Guides

1. [CI/CD Setup](docs/cicd-setup.md) — Forgejo, Woodpecker CI, and promote pipeline
2. [Deployment Model](docs/deployment-model.md) — application surfaces and routing
3. [Netcup VPS Deployment](docs/netcup-deployment-plan.md) — production server setup

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
docker-compose.yml          # All infrastructure services
.env.example                # Template for secrets
docs/
  cicd-setup.md             # Forgejo, Woodpecker CI, and promote pipeline guide
  deployment-model.md       # Application surfaces and routing
  netcup-deployment-plan.md # Production VPS setup and deployment
```
