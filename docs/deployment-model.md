# Abeona – Simplified Four‑Surface Deployment Model

This document defines the streamlined deployment architecture for the Abeona platform. The goal is to package the system into four independent deployable surfaces, each isolated, versionable, and easy to manage via Docker and CI/CD.

---

## Overview

Abeona is deployed as **four separate surfaces**, each with its own Docker Compose stack:

1. **Catalog API Surface**  
2. **Booking API Surface**  
3. **Backoffice UI Surface**  
4. **Catalog UI Surface**

Each surface is self‑contained and can be deployed, rolled back, or updated independently.

---

## 1. Catalog API Surface

**Purpose**
- Exposes all catalog data (tours, itineraries, pricing, metadata).
- Owns image metadata and writes image files to a host directory.
- Provides content to both UIs.

**Components**
- Spring Boot container  
- Postgres container (dedicated DB)  
- Image volume (bind mount served by host Nginx)

**Directory Layout**
```
/home/deployguy/tourpro_catalog/
  docker-compose.yml
  .env
  db-data/
  images/
```


**`.env` Variables**
- `DB_PASSWORD` — Postgres password
- `CATALOG_API_IMAGE` — Docker image reference (e.g. `192.168.1.77:5000/tourpro/catalog-api:latest`)
- `IMAGES_BASE_URL` — Public image URL (e.g. `http://192.168.1.77/images`)

**Notes**
- Images are written by the API into `/home/deployguy/tourpro_catalog/images`.
- Host Nginx serves this directory directly as static content.
- Container port bound to `127.0.0.1:8080` — all external access goes through host Nginx.
- Deployed via Woodpecker CI from private registry at `192.168.1.77:5000`.

---

## 2. Booking API Surface

**Purpose**
- Handles reservations, customer details, and booking workflows.
- Independent from catalog logic.

**Components**
- Spring Boot container
- Postgres container (dedicated DB)

**Directory Layout**
```
/srv/abeona/booking-api/
  docker-compose.yml
  .env
  db-data/
```


**Notes**
- No image storage required.
- Clean separation from catalog domain.

---

## 3. Backoffice UI Surface

**Purpose**
- Internal administrative interface.
- Used for managing catalog content, pricing, images, and availability.

**Components**
- Nginx container serving Angular build

**Directory Layout**
```
/srv/abeona/backoffice-ui/
  docker-compose.yml
  .env
```


**Notes**
- Served behind host Nginx.
- Requires authentication and stricter routing rules.

---

## 4. Catalog UI Surface

**Purpose**
- Public‑facing customer interface.
- Displays tours, images, pricing, and booking flows.

**Components**
- Nginx container serving Angular build

**Directory Layout**
```
/srv/abeona/catalog-ui/
  docker-compose.yml
  .env
```


**Notes**
- Routed through host Nginx.
- Loads images from the Catalog API’s image volume via static Nginx routes.

---

## Host Nginx as the Single Entry Point

The host machine runs a single Nginx instance that handles:

- TLS termination  
- Routing to each surface  
- Static image delivery  
- Security headers  
- Public domain mapping  

**Example Routing (Conceptual)**
```
https://abeonatravel.africa/              → Catalog UI
https://abeonatravel.africa/backoffice/   → Backoffice UI
https://abeonatravel.africa/api/catalog/  → Catalog API (/api/catalog/ → localhost:8080/api/v1/)
https://abeonatravel.africa/api/booking/  → Booking API
https://abeonatravel.africa/images/       → Host Nginx serving /home/deployguy/tourpro_catalog/images
```

**Swagger UI (optional, removable)**
```
/swagger-ui/   → localhost:8080/swagger-ui/
/v3/api-docs   → localhost:8080/v3/api-docs
/swagger/api/  → localhost:8080/  (passthrough proxy for Swagger Execute)
```
Swagger UI can be disabled by removing these three Nginx location blocks.

This keeps containers private and the server secure.

---

## Why This Model Works

**Independent deployments**  
Each surface can be updated without affecting the others.

**Clean CI/CD integration**  
Woodpecker can deploy each surface separately.

**Predictable persistence**  
Each API has its own DB volume.  
Catalog API has its own image volume.

**Easy rollback**  
Each surface uses versioned Docker images.

**Scalable**  
If you ever split across multiple servers, each surface is already isolated.

---

## Summary

Abeona is deployed as four isolated Docker surfaces:

- **Catalog API**: Spring Boot + Postgres, DB volume + image volume, served by host Nginx  
- **Booking API**: Spring Boot + Postgres, DB volume, served by host Nginx  
- **Backoffice UI**: Angular + Nginx, served by host Nginx  
- **Catalog UI**: Angular + Nginx, served by host Nginx  

This model is simple, maintainable, and production‑ready.
