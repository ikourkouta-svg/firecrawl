# Firecrawl Self-Hosted Deployment Guide - Railway

**Version:** v2.8.0 (updated March 2026)
**Project:** attractive-analysis
**Deployed URL:** https://firecrawl-api-production-403e.up.railway.app

---

## Overview

Self-hosted Firecrawl v2.8.0 on Railway as a shared scraping service for all projects (ExportIQ, lead gen, etc).

**Cost:** ~$20-50/month (vs $333/month cloud)

---

## Architecture

### Services Required (5 services in Railway)

1. **firecrawl-api** (API + Workers — unified harness)
   - Port: 3002
   - Start Command: `node dist/src/harness.js --start-docker`
   - Handles HTTP requests + runs workers in-process
   - Builds from `apps/api/Dockerfile.railway`

2. **Playwright Service** (JS rendering)
   - Image: `ghcr.io/firecrawl/playwright-service:latest`
   - Port: 3000
   - Handles JavaScript rendering and complex web pages

3. **Redis** (Cache, Rate Limiting, Job Queues)
   - Image: `redis:alpine`
   - Railway managed or Docker service

4. **RabbitMQ** (Message Queue — NEW in v2.8.0)
   - Image: `rabbitmq:3-management`
   - Port: 5672 (AMQP), 15672 (management UI)
   - Required for nuq workers

5. **PostgreSQL** (Queue State — nuq)
   - Image: Railway managed Postgres or `apps/nuq-postgres`
   - Schema: nuq (queue_scrape table)

---

## Environment Variables (firecrawl-api service)

### Required
```
PORT=3002
HOST=0.0.0.0
USE_DB_AUTHENTICATION=false
REDIS_URL=redis://<railway-redis-internal-url>:6379
REDIS_RATE_LIMIT_URL=redis://<railway-redis-internal-url>:6379
PLAYWRIGHT_MICROSERVICE_URL=http://<playwright-service-internal-url>:3000/scrape
NUQ_RABBITMQ_URL=amqp://<rabbitmq-internal-url>:5672
POSTGRES_HOST=<postgres-internal-host>
POSTGRES_PORT=5432
POSTGRES_DB=postgres
POSTGRES_USER=postgres
POSTGRES_PASSWORD=<your-password>
BULL_AUTH_KEY=<strong-random-key>
```

### Optional (AI features)
```
OPENAI_API_KEY=<key>
OPENAI_BASE_URL=<url>
```

---

## Setup Steps

### 1. Add services to Railway project

In the Railway dashboard for `attractive-analysis`:

- **Redis**: Add → Database → Redis
- **RabbitMQ**: Add → Docker Image → `rabbitmq:3-management`
  - Set PORT=5672
- **PostgreSQL**: Add → Database → PostgreSQL (if not existing)
- **Playwright**: Add → Docker Image → `ghcr.io/firecrawl/playwright-service:latest`
  - Set PORT=3000
- **firecrawl-api**: Deploy from GitHub repo `ikourkouta-svg/firecrawl`
  - Uses `apps/api/Dockerfile.railway` via `railway.toml`

### 2. Connect services via env vars

Use Railway's internal networking (e.g., `redis.railway.internal:6379`).

### 3. Deploy

```bash
cd /path/to/firecrawl
railway link  # select attractive-analysis → firecrawl-api
railway up
```

### 4. Verify

```bash
curl https://firecrawl-api-production-403e.up.railway.app/health
curl -X POST https://firecrawl-api-production-403e.up.railway.app/v1/scrape \
  -H 'Content-Type: application/json' \
  -d '{"url": "https://example.com"}'
```

---

## Using as a Shared Service

Any project can call Firecrawl via HTTP:

```typescript
const FIRECRAWL_API_URL = process.env.FIRECRAWL_API_URL // Railway internal or public URL
const FIRECRAWL_API_KEY = process.env.FIRECRAWL_API_KEY // 'fc-self-hosted' for self-hosted

const response = await fetch(`${FIRECRAWL_API_URL}/v1/scrape`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${FIRECRAWL_API_KEY}`,
  },
  body: JSON.stringify({ url: 'https://example.com', formats: ['markdown'] }),
})
```

### Projects using this service
- **ExportIQ trade-show-scraper** — scrapes EventsEye for trade shows
- (Add more as needed)

---

## Troubleshooting

### ECONNREFUSED to Firecrawl
- Check if firecrawl-api service is running in Railway dashboard
- Verify `FIRECRAWL_API_URL` env var uses correct Railway internal URL
- Check that Redis, RabbitMQ, and Postgres are all healthy

### All scrapes failing
- Check Playwright service is running
- Check `PLAYWRIGHT_MICROSERVICE_URL` points to correct internal URL
- Check RabbitMQ is healthy (management UI on port 15672)

### Build fails on Railway
- The Dockerfile uses no Docker cache mounts (Railway doesn't support them)
- Build requires ~4GB RAM and ~10min for Go + Rust compilation
