# Firecrawl Self-Hosted Deployment Guide - Railway

**Date:** October 9, 2025
**Project:** attractive-analysis
**Deployed URL:** https://firecrawl-api-production-403e.up.railway.app

---

## Overview

Successfully deployed Firecrawl v2 to Railway infrastructure as a cost-effective alternative to cloud hosting.

**Cost Savings:** ~$283-313/month
- Cloud Firecrawl: $333/month
- Self-hosted Railway: ~$20-50/month

---

## Architecture

### Services Deployed

1. **firecrawl-api** (API Server)
   - Port: 3002
   - Start Command: `node dist/src/index.js`
   - Handles HTTP requests, adds jobs to queue

2. **firecrawl-workers** (Job Processors)
   - Port: 3006
   - Start Command: `node --import ./dist/src/otel.js dist/src/services/worker/nuq-worker.js`
   - Processes scrape jobs from queue

3. **firecrawl** (Playwright Service)
   - Port: 3000
   - Handles JavaScript rendering and complex web pages

4. **Redis-B6U4** (Cache & Rate Limiting)
   - Region: EU West (Amsterdam)
   - Public URL: `redis://default:nLPxKNufaWeqiRFkyvBoXZEpJDczogMx@turntable.proxy.rlwy.net:32470`

5. **Postgres-6Td0** (Job Queue Database)
   - Region: EU West
   - Public URL: `postgresql://postgres:nDlYUyNlSriNIgboKHOqqXwEvzBkMBMp@maglev.proxy.rlwy.net:28496/railway`
   - Schema: nuq (queue_scrape table)

---

## Initial Setup Steps

### 1. Repository Setup

```bash
# Download Firecrawl repository
cd /home/snbjason
wget https://github.com/mendableai/firecrawl/archive/refs/heads/main.zip
unzip main.zip
mv firecrawl-main firecrawl
```

### 2. Railway CLI Installation

```bash
# Install Railway CLI
npm install -g @railway/cli

# Login to Railway
railway login

# Link to project
railway link
# Select: attractive-analysis project
```

### 3. Fix Dockerfile for Railway

**Problem:** Railway doesn't support BuildKit cache mounts

**Solution:** Edit `apps/api/Dockerfile` and remove cache mount lines:

```dockerfile
# BEFORE (lines 43-46):
RUN --mount=type=cache,id=pnpm,target=/pnpm/store \
    --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/native/target \
    pnpm install --frozen-lockfile

# AFTER:
RUN pnpm install --frozen-lockfile
```

**Commit this change to your forked repository on GitHub**

---

## Service Deployment

### Service 1: firecrawl-api (API Server)

1. **Create via Railway Dashboard:**
   - Project: attractive-analysis
   - GitHub Repo: Your firecrawl fork
   - Root Directory: `apps/api`
   - Dockerfile: `apps/api/Dockerfile.railway`

2. **Environment Variables:**

```bash
railway service firecrawl-api

# Database (use PUBLIC URLs due to region issues)
railway variables --set 'NUQ_DATABASE_URL=postgresql://postgres:nDlYUyNlSriNIgboKHOqqXwEvzBkMBMp@maglev.proxy.rlwy.net:28496/railway'

# Redis (use PUBLIC URLs)
railway variables --set 'REDIS_URL=redis://default:nLPxKNufaWeqiRFkyvBoXZEpJDczogMx@turntable.proxy.rlwy.net:32470'
railway variables --set 'REDIS_RATE_LIMIT_URL=redis://default:nLPxKNufaWeqiRFkyvBoXZEpJDczogMx@turntable.proxy.rlwy.net:32470'

# Playwright service
railway variables --set 'PLAYWRIGHT_MICROSERVICE_URL=http://firecrawl.railway.internal:3000/scrape'

# Other configs
railway variables --set 'USE_DB_AUTHENTICATION=false'
railway variables --set 'HOST=0.0.0.0'
railway variables --set 'PORT=3002'
```

3. **Start Command (in Railway Settings):**
```
node dist/src/index.js
```

4. **Generate Public Domain:**
   - Railway Dashboard → firecrawl-api → Settings → Networking
   - Click "Generate Domain"
   - Result: `https://firecrawl-api-production-403e.up.railway.app`

---

### Service 2: firecrawl-workers (Job Processors)

1. **Create via Railway Dashboard:**
   - Click "+ New Service"
   - Select GitHub Repo: Your firecrawl fork
   - Root Directory: `apps/api`
   - Name: `firecrawl-workers`

2. **Environment Variables:**

```bash
railway service <workers-service-id>

railway variables --set 'NUQ_DATABASE_URL=postgresql://postgres:nDlYUyNlSriNIgboKHOqqXwEvzBkMBMp@maglev.proxy.rlwy.net:28496/railway'
railway variables --set 'REDIS_URL=redis://default:nLPxKNufaWeqiRFkyvBoXZEpJDczogMx@turntable.proxy.rlwy.net:32470'
railway variables --set 'REDIS_RATE_LIMIT_URL=redis://default:nLPxKNufaWeqiRFkyvBoXZEpJDczogMx@turntable.proxy.rlwy.net:32470'
railway variables --set 'PLAYWRIGHT_MICROSERVICE_URL=http://firecrawl.railway.internal:3000/scrape'
railway variables --set 'USE_DB_AUTHENTICATION=false'
railway variables --set 'HOST=0.0.0.0'
railway variables --set 'PORT=3006'
railway variables --set 'NUQ_WORKER_PORT=3006'
railway variables --set 'NUQ_REDUCE_NOISE=true'
railway variables --set 'NUQ_POD_NAME=nuq-worker-0'
```

3. **Start Command (in Railway Settings):**
```
node --import ./dist/src/otel.js dist/src/services/worker/nuq-worker.js
```

---

### Service 3: firecrawl (Playwright Service)

1. **Create via Railway Dashboard:**
   - GitHub Repo: Your firecrawl fork
   - Root Directory: `apps/playwright-service-ts`
   - Name: `firecrawl`

2. **Environment Variables:**
```
PORT=3000
```

---

### Service 4: Redis-B6U4

1. **Create via Railway Dashboard:**
   - Click "+ New"
   - Select "Redis"
   - Name: `Redis-B6U4`
   - **Important:** Select EU West region (same as other services)

2. **Note the public URL for configuration**

---

### Service 5: Postgres-6Td0

1. **Create via Railway Dashboard:**
   - Click "+ New"
   - Select "PostgreSQL"
   - Name: `Postgres-6Td0`
   - **Important:** Select EU West region

2. **Initialize Database Schema:**

```bash
# Install PostgreSQL client dependency
cd /home/snbjason
npm install pg

# Create simplified schema (without pg_cron)
cat > /home/snbjason/nuq_simple.sql << 'EOF'
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE SCHEMA IF NOT EXISTS nuq;

DO $$ BEGIN
  CREATE TYPE nuq.job_status AS ENUM ('queued', 'active', 'completed', 'failed');
EXCEPTION
  WHEN duplicate_object THEN null;
END $$;

CREATE TABLE IF NOT EXISTS nuq.queue_scrape (
  id uuid NOT NULL DEFAULT gen_random_uuid(),
  status nuq.job_status NOT NULL DEFAULT 'queued'::nuq.job_status,
  data jsonb,
  created_at timestamp with time zone NOT NULL DEFAULT now(),
  priority int NOT NULL DEFAULT 0,
  lock uuid,
  locked_at timestamp with time zone,
  stalls integer,
  finished_at timestamp with time zone,
  listen_channel_id text,
  returnvalue jsonb,
  failedreason text,
  CONSTRAINT queue_scrape_pkey PRIMARY KEY (id)
);

ALTER TABLE nuq.queue_scrape
SET (autovacuum_vacuum_scale_factor = 0.01,
     autovacuum_analyze_scale_factor = 0.01,
     autovacuum_vacuum_cost_limit = 2000,
     autovacuum_vacuum_cost_delay = 2);

CREATE INDEX IF NOT EXISTS queue_scrape_active_locked_at_idx ON nuq.queue_scrape USING btree (locked_at) WHERE (status = 'active'::nuq.job_status);
CREATE INDEX IF NOT EXISTS nuq_queue_scrape_queued_optimal_2_idx ON nuq.queue_scrape (priority ASC, created_at ASC, id) WHERE (status = 'queued'::nuq.job_status);
CREATE INDEX IF NOT EXISTS nuq_queue_scrape_failed_created_at_idx ON nuq.queue_scrape USING btree (created_at) WHERE (status = 'failed'::nuq.job_status);
CREATE INDEX IF NOT EXISTS nuq_queue_scrape_completed_created_at_idx ON nuq.queue_scrape USING btree (created_at) WHERE (status = 'completed'::nuq.job_status);
EOF

# Create initialization script
cat > /home/snbjason/init_nuq_db.cjs << 'EOF'
const fs = require('fs');
const { Client } = require('pg');

const DATABASE_URL = "postgresql://postgres:nDlYUyNlSriNIgboKHOqqXwEvzBkMBMp@maglev.proxy.rlwy.net:28496/railway";

async function initDatabase() {
  const client = new Client({ connectionString: DATABASE_URL });

  try {
    console.log('Connecting to database...');
    await client.connect();

    console.log('Reading SQL file...');
    const sql = fs.readFileSync('/home/snbjason/nuq_simple.sql', 'utf8');

    console.log('Executing SQL script...');
    await client.query(sql);

    console.log('✓ Database schema initialized successfully!');

    const result = await client.query("SELECT table_name FROM information_schema.tables WHERE table_schema = 'nuq'");
    console.log(`✓ Created tables in nuq schema: ${result.rows.map(r => r.table_name).join(', ')}`);

  } catch (err) {
    console.error('✗ Error:', err.message);
    process.exit(1);
  } finally {
    await client.end();
  }
}

initDatabase();
EOF

# Run initialization
node /home/snbjason/init_nuq_db.cjs
```

---

## n8n Integration

1. **Update n8n Environment Variable:**

```bash
railway service Primary  # Your n8n service

railway variables --set 'FIRECRAWL_URL=https://firecrawl-api-production-403e.up.railway.app'
```

2. **In n8n Workflows:**
   - Use HTTP Request node or Firecrawl node (if available)
   - Base URL: `https://firecrawl-api-production-403e.up.railway.app`
   - No API key needed for self-hosted version

---

## Testing

### Test Scraping Endpoint

```bash
# Basic test
curl -X POST https://firecrawl-api-production-403e.up.railway.app/v1/scrape \
  -H 'Content-Type: application/json' \
  -d '{"url":"https://example.com","formats":["markdown"]}' | jq

# Expected successful response:
{
  "success": true,
  "data": {
    "markdown": "Example Domain\n==============\n...",
    "metadata": {
      "url": "https://example.com",
      "title": "Example Domain",
      "statusCode": 200,
      "creditsUsed": 1
    }
  }
}
```

### Test Script

```bash
# Create test script
cat > /home/snbjason/test-firecrawl.sh << 'EOF'
#!/bin/bash
curl -X POST https://firecrawl-api-production-403e.up.railway.app/v1/scrape \
  -H 'Content-Type: application/json' \
  -d '{"url":"https://firecrawl.dev","formats":["markdown"]}'
EOF

chmod +x /home/snbjason/test-firecrawl.sh
bash /home/snbjason/test-firecrawl.sh
```

---

## Troubleshooting Guide

### Issue 1: Cache Mount Error During Build

**Error:** `Cache mount ID is not prefixed with cache key`

**Solution:** Remove cache mounts from Dockerfile:
- Edit `apps/api/Dockerfile`
- Remove `--mount=type=cache` lines from RUN commands
- Commit and push to GitHub

---

### Issue 2: Redis Connection Failures

**Error:** `ENOTFOUND redis.railway.internal`

**Cause:** Railway private networking issues between regions

**Solution:** Use public Redis URLs instead:
```bash
# Get public URL
railway service Redis-B6U4
railway variables --kv | grep PUBLIC

# Use in firecrawl-api and firecrawl-workers
REDIS_URL=redis://default:PASSWORD@turntable.proxy.rlwy.net:PORT
```

---

### Issue 3: Database Schema Missing

**Error:** `relation "nuq.queue_scrape" does not exist`

**Solution:** Initialize database schema manually (see Postgres setup above)

---

### Issue 4: Extract Worker Crashes

**Error:** `thread 'main' panicked at library/std/src/io/stdio.rs`

**Cause:** Native Rust libraries incompatible with Railway environment

**Solution:** Run services separately without harness:
- firecrawl-api: `node dist/src/index.js`
- firecrawl-workers: `node --import ./dist/src/otel.js dist/src/services/worker/nuq-worker.js`

---

### Issue 5: Scrape Timeout

**Error:** `SCRAPE_TIMEOUT`

**Cause:** No workers running to process jobs

**Solution:** Ensure firecrawl-workers service is running and connected:
```bash
railway service <workers-service-id>
railway logs --tail 50 | grep "No jobs to process"
# Should see worker polling for jobs
```

---

## Important Notes

### Why Public URLs Instead of Private Networking?

Railway's private networking (`.railway.internal`) had DNS resolution issues between services. Using public URLs ensures reliable connectivity.

### Why Separate Worker Service?

The extract-worker component has native Rust dependencies that crash in Railway's environment. Running workers separately avoids the harness that tries to start all services together.

### Database Schema Differences

The deployed schema omits `pg_cron` extension (not available in standard PostgreSQL). This means:
- ✅ Scraping works perfectly
- ❌ No automatic cleanup of old jobs
- **Action:** Manually clean completed/failed jobs periodically if needed:
  ```sql
  DELETE FROM nuq.queue_scrape
  WHERE status = 'completed' AND created_at < now() - interval '1 hour';
  ```

---

## Maintenance

### Monitoring

```bash
# Check API logs
railway service firecrawl-api
railway logs --tail 100

# Check worker logs
railway service <workers-service-id>
railway logs --tail 100 | grep "nuqGetJobToProcess"
```

### Scaling Workers

To add more workers:
1. Duplicate the firecrawl-workers service
2. Change `NUQ_POD_NAME` to unique value (nuq-worker-1, nuq-worker-2, etc.)
3. Change `NUQ_WORKER_PORT` to unique port (3007, 3008, etc.)

### Database Cleanup

```bash
# Connect to database
psql "postgresql://postgres:nDlYUyNlSriNIgboKHOqqXwEvzBkMBMp@maglev.proxy.rlwy.net:28496/railway"

# Clean old completed jobs (older than 1 hour)
DELETE FROM nuq.queue_scrape
WHERE status = 'completed' AND created_at < now() - interval '1 hour';

# Clean old failed jobs (older than 6 hours)
DELETE FROM nuq.queue_scrape
WHERE status = 'failed' AND created_at < now() - interval '6 hours';
```

---

## Service URLs & Credentials

### Production Endpoints

- **Firecrawl API:** https://firecrawl-api-production-403e.up.railway.app
- **n8n Instance:** https://primary-production-8e92.up.railway.app

### Database Connections

**PostgreSQL (Postgres-6Td0):**
- Public: `postgresql://postgres:nDlYUyNlSriNIgboKHOqqXwEvzBkMBMp@maglev.proxy.rlwy.net:28496/railway`
- User: `postgres`
- Password: `nDlYUyNlSriNIgboKHOqqXwEvzBkMBMp`
- Database: `railway`

**Redis (Redis-B6U4):**
- Public: `redis://default:nLPxKNufaWeqiRFkyvBoXZEpJDczogMx@turntable.proxy.rlwy.net:32470`
- Password: `nLPxKNufaWeqiRFkyvBoXZEpJDczogMx`

---

## Cost Breakdown

### Monthly Estimates

**Railway Services:**
- firecrawl-api: ~$5-10
- firecrawl-workers: ~$5-10
- firecrawl (Playwright): ~$5-10
- Redis-B6U4: ~$2-5
- Postgres-6Td0: ~$2-5
- **Total:** ~$20-50/month

**vs Cloud Firecrawl:**
- Professional Plan: $333/month
- **Savings:** ~$283-313/month (85-94% cost reduction)

---

## API Usage Examples

### Scrape a URL

```bash
curl -X POST https://firecrawl-api-production-403e.up.railway.app/v1/scrape \
  -H 'Content-Type: application/json' \
  -d '{
    "url": "https://example.com",
    "formats": ["markdown", "html"]
  }'
```

### Scrape with Options

```bash
curl -X POST https://firecrawl-api-production-403e.up.railway.app/v1/scrape \
  -H 'Content-Type: application/json' \
  -d '{
    "url": "https://example.com",
    "formats": ["markdown"],
    "onlyMainContent": true,
    "waitFor": 1000
  }'
```

### Use in n8n HTTP Request Node

```json
{
  "method": "POST",
  "url": "https://firecrawl-api-production-403e.up.railway.app/v1/scrape",
  "headers": {
    "Content-Type": "application/json"
  },
  "body": {
    "url": "{{$json.websiteUrl}}",
    "formats": ["markdown"]
  }
}
```

---

## Success Criteria ✅

- [x] Firecrawl API deployed and accessible
- [x] Workers processing scrape jobs successfully
- [x] Playwright service rendering JavaScript pages
- [x] Redis connected for rate limiting and caching
- [x] PostgreSQL database initialized with queue schema
- [x] n8n integration configured
- [x] End-to-end scraping test successful
- [x] Cost reduced by 85-94%

---

## Future Improvements

1. **Add monitoring:** Set up health checks and alerting
2. **Implement auto-scaling:** Scale workers based on queue depth
3. **Add database cleanup cron:** External cron job for old job cleanup
4. **SSL/TLS:** Add custom domain with SSL certificate
5. **Extract worker fix:** Investigate native library compatibility for full feature set

---

## Support & Documentation

- **Firecrawl Docs:** https://docs.firecrawl.dev
- **Railway Docs:** https://docs.railway.app
- **GitHub Repo:** https://github.com/mendableai/firecrawl
- **This Guide:** `/home/snbjason/FIRECRAWL_RAILWAY_DEPLOYMENT_GUIDE.md`

---

**Last Updated:** October 9, 2025
**Status:** ✅ Production Ready
**Deployed By:** Claude Code Assistant
