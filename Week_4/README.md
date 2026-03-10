# Week 4 — Fullstack local development with Docker
Run Multi-Tier Applications with Docker Compose

## Architecture
```
┌─────────────────┐
│  React+nginx    │  ecommerce_frontend (Port 3000)
│   (Tier 1)      │
└────────┬────────┘
         │ /api/ proxy
┌────────▼────────┐
│  Node.js+Express│  ecommerce_backend (Port 5000)
│   (Tier 2)      │
└────────┬────────┘
         │ SQL
┌────────▼────────┐
│   PostgreSQL    │  ecommerce_db (Port 5432)
│   (Tier 3)      │
└─────────────────┘
```

## Tech Stack
- Frontend: React 18
- Backend: Node.js 20, Express.js
- Database: PostgreSQL 16
- Orchestration: Docker Compose


## What I Did

### 1. Wrote Backend Dockerfile
I Used a multi-stage build with node:20-alpine to keep the image small
- Stage 1 installs only production dependencies
- Stage 2 copies node_modules and source code into a fresh image

### 2. Wrote Frontend Dockerfile
Used multi-stage build:
- Stage 1 (node:20-alpine) - installs dependencies, builds React app
- Stage 2 (nginx:alpine) - copies the compiled build folder and serves it

### 3. Wrote nginx config file
I wrote a default nginx conf file that handles `/api/` requests to the backend internally so only port 3000 is exposed publicly

### 4. Wrote docker-compose.yml
Configured environment variables and connected all three services with shared networking and volumes

### 5. Ran Everything
```bash
docker compose up --build
```
Builds all images, creates the network and volume, starts containers and outputs all logs to the terminal.

## Key Concepts Learned

### Alpine Images
Alpine reduces image sizes.

### Docker Networking
Containers communicate by service name, not localhost.
Backend reaches database at `postgresql:5432` not `localhost:5432`.

### nginx as Static Server
nginx is more efficient than node + serve for
serving compiled React apps in production.

### Volumes
Without volumes, database data is lost when containers stop.
Named volumes persist data between restarts.

## Important Docker Commands
```bash
docker compose up --build        # build and start all services
docker compose up -d --build     # run in background
docker compose down              # stop and remove containers
docker compose ps                # check container status
docker compose logs -f           # live logs all services
docker compose logs backend -f   # live logs specific service
docker images                    # list all images and sizes
docker system prune              # clean up unused resources
```

## Errors Encountered & Fixed
| Error | Cause | Fix |
|---|---|---|
| Products not loading | App.js had localhost hardcoded | Updated to use empty string with nginx proxy |