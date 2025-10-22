# Deployment Guide

## Overview
Complete production deployment guide for HIRESIGHT platform using Docker, Railway/Render for backend, and Vercel for frontend.

---

## Prerequisites

- Docker and Docker Compose installed
- GitHub account
- Railway/Render account (backend hosting)
- Vercel account (frontend hosting)
- PostgreSQL database (managed service)
- Redis instance
- AWS S3 bucket or compatible storage
- SendGrid account
- Domain name (optional)

---

## Environment Setup

### Backend Environment Variables

**File: `.env.production`**

```bash
# App
ENVIRONMENT=production
SECRET_KEY=your-super-secret-key-change-this
DEBUG=false

# Database
DATABASE_URL=postgresql://user:password@host:5432/hiresight
DATABASE_POOL_SIZE=20

# Redis
REDIS_HOST=redis-host.com
REDIS_PORT=6379
REDIS_PASSWORD=redis-password

# AWS S3
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
AWS_REGION=us-east-1
S3_BUCKET_NAME=hiresight-files

# Authentication
JWT_SECRET_KEY=another-secret-key
ACCESS_TOKEN_EXPIRE_MINUTES=60
REFRESH_TOKEN_EXPIRE_DAYS=30

# OAuth
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
GITHUB_CLIENT_ID=your-github-client-id
GITHUB_CLIENT_SECRET=your-github-client-secret
GITHUB_TOKEN=your-github-token

# Email
SENDGRID_API_KEY=your-sendgrid-api-key
FROM_EMAIL=noreply@hiresight.com

# Frontend URL
FRONTEND_URL=https://app.hiresight.com

# Monitoring
SENTRY_DSN=your-sentry-dsn
POSTHOG_API_KEY=your-posthog-key

# Stripe (if using billing)
STRIPE_SECRET_KEY=sk_live_...
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

### Frontend Environment Variables

**File: `.env.production`**

```bash
VITE_API_URL=https://api.hiresight.com/api/v1
VITE_GITHUB_CLIENT_ID=your-github-client-id
VITE_GOOGLE_CLIENT_ID=your-google-client-id
VITE_STRIPE_PUBLISHABLE_KEY=pk_live_...
VITE_SENTRY_DSN=your-sentry-dsn
VITE_POSTHOG_API_KEY=your-posthog-key
```

---

## Docker Configuration

### Backend Dockerfile

**File: `backend/Dockerfile`**

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Create non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Run migrations and start server
CMD ["sh", "-c", "alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port 8000"]
```

### Frontend Dockerfile

**File: `frontend/Dockerfile`**

```dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci

# Copy source and build
COPY . .
RUN npm run build

# Production image
FROM nginx:alpine

# Copy built files
COPY --from=builder /app/dist /usr/share/nginx/html

# Copy nginx config
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Docker Compose for Local Development

**File: `docker-compose.yml`**

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: hiresight
      POSTGRES_PASSWORD: password
      POSTGRES_DB: hiresight
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://hiresight:password@postgres:5432/hiresight
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      - postgres
      - redis
    volumes:
      - ./backend:/app

  celery-worker:
    build:
      context: ./backend
      dockerfile: Dockerfile
    command: celery -A app.workers.celery_app worker --loglevel=info
    environment:
      - DATABASE_URL=postgresql://hiresight:password@postgres:5432/hiresight
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      - postgres
      - redis
      - backend
    volumes:
      - ./backend:/app

  celery-beat:
    build:
      context: ./backend
      dockerfile: Dockerfile
    command: celery -A app.workers.celery_app beat --loglevel=info
    environment:
      - DATABASE_URL=postgresql://hiresight:password@postgres:5432/hiresight
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      - postgres
      - redis
      - backend

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    ports:
      - "5173:5173"
    volumes:
      - ./frontend:/app
      - /app/node_modules
    environment:
      - VITE_API_URL=http://localhost:8000/api/v1

volumes:
  postgres_data:
  redis_data:
```

---

## Database Setup

### Create Managed PostgreSQL Database

**Option 1: Render**
1. Go to Render Dashboard
2. Click "New +"  → "PostgreSQL"
3. Name: hiresight-db
4. Region: Choose closest to your users
5. Plan: Starter ($7/month)
6. Copy the connection string

**Option 2: Railway**
1. Go to Railway Dashboard
2. Click "New Project" → "Database" → "PostgreSQL"
3. Copy the connection string

**Option 3: AWS RDS**
```bash
# Create RDS instance
aws rds create-db-instance \
  --db-instance-identifier hiresight-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username hiresight \
  --master-user-password your-password \
  --allocated-storage 20
```

### Run Migrations

```bash
# Set DATABASE_URL
export DATABASE_URL=postgresql://user:pass@host:5432/hiresight

# Run migrations
cd backend
alembic upgrade head

# Seed initial data (optional)
python scripts/seed_data.py
```

---

## Backend Deployment

### Option 1: Railway

1. **Install Railway CLI**
```bash
npm install -g @railway/cli
railway login
```

2. **Create Project**
```bash
cd backend
railway init
```

3. **Add Environment Variables**
```bash
railway variables set DATABASE_URL=postgresql://...
railway variables set REDIS_HOST=...
# Add all other variables
```

4. **Deploy**
```bash
railway up
```

5. **Custom Domain** (optional)
```bash
railway domain add api.hiresight.com
```

### Option 2: Render

1. **Create Blueprint**

**File: `render.yaml`**
```yaml
services:
  - type: web
    name: hiresight-api
    env: python
    buildCommand: "pip install -r requirements.txt && alembic upgrade head"
    startCommand: "uvicorn app.main:app --host 0.0.0.0 --port $PORT"
    envVars:
      - key: DATABASE_URL
        fromDatabase:
          name: hiresight-db
          property: connectionString
      - key: REDIS_HOST
        fromService:
          name: hiresight-redis
          property: host
      - key: SECRET_KEY
        generateValue: true

  - type: worker
    name: hiresight-worker
    env: python
    buildCommand: "pip install -r requirements.txt"
    startCommand: "celery -A app.workers.celery_app worker"
    envVars:
      - key: DATABASE_URL
        fromDatabase:
          name: hiresight-db
          property: connectionString

databases:
  - name: hiresight-db
    databaseName: hiresight
    plan: starter

  - name: hiresight-redis
    plan: starter
```

2. **Deploy**
- Push to GitHub
- Connect repository to Render
- Click "Apply"

### Option 3: AWS ECS (Advanced)

**File: `task-definition.json`**
```json
{
  "family": "hiresight-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "your-ecr-repo/hiresight-api:latest",
      "portMappings": [
        {
          "containerPort": 8000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "ENVIRONMENT", "value": "production"}
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:region:account:secret:db-url"
        }
      ]
    }
  ]
}
```

---

## Frontend Deployment

### Vercel (Recommended)

1. **Install Vercel CLI**
```bash
npm install -g vercel
```

2. **Configure Project**

**File: `vercel.json`**
```json
{
  "buildCommand": "npm run build",
  "outputDirectory": "dist",
  "framework": "vite",
  "env": {
    "VITE_API_URL": "https://api.hiresight.com/api/v1"
  }
}
```

3. **Deploy**
```bash
cd frontend
vercel --prod
```

4. **Custom Domain**
- Go to Vercel Dashboard → Project → Settings → Domains
- Add `app.hiresight.com`
- Configure DNS

### Netlify (Alternative)

**File: `netlify.toml`**
```toml
[build]
  command = "npm run build"
  publish = "dist"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200

[build.environment]
  VITE_API_URL = "https://api.hiresight.com/api/v1"
```

---

## SSL/HTTPS Setup

### Using Cloudflare (Recommended)

1. Add your domain to Cloudflare
2. Update nameservers at your registrar
3. Enable "Full (strict)" SSL mode
4. Enable "Always Use HTTPS"
5. Set up firewall rules
6. Enable DDoS protection

### Using Let's Encrypt (Self-hosted)

```bash
# Install certbot
sudo apt-get install certbot python3-certbot-nginx

# Get certificate
sudo certbot --nginx -d api.hiresight.com -d app.hiresight.com

# Auto-renewal
sudo certbot renew --dry-run
```

---

## Monitoring & Logging

### Sentry Setup

**Backend:**
```python
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration

sentry_sdk.init(
    dsn=settings.SENTRY_DSN,
    environment="production",
    integrations=[FastApiIntegration()],
    traces_sample_rate=0.1,
)
```

**Frontend:**
```typescript
import * as Sentry from "@sentry/react";

Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN,
  environment: "production",
  tracesSampleRate: 0.1,
});
```

### Logging

**File: `backend/app/core/logging.py`**
```python
import logging
import sys
from loguru import logger

# Configure loguru
logger.remove()
logger.add(
    sys.stdout,
    format="<green>{time:YYYY-MM-DD HH:mm:ss}</green> | <level>{level: <8}</level> | <cyan>{name}</cyan>:<cyan>{function}</cyan> - <level>{message}</level>",
    level="INFO"
)

# File logging
logger.add(
    "logs/app.log",
    rotation="500 MB",
    retention="10 days",
    level="INFO"
)
```

---

## Performance Optimization

### Backend

1. **Enable Gzip Compression**
```python
from fastapi.middleware.gzip import GZipMiddleware

app.add_middleware(GZipMiddleware, minimum_size=1000)
```

2. **Database Connection Pooling**
```python
engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,
    max_overflow=40,
    pool_pre_ping=True
)
```

3. **Redis Caching**
```python
from functools import lru_cache
from redis import Redis

redis_client = Redis(host=REDIS_HOST, decode_responses=True)

@lru_cache(maxsize=1000)
def get_cached_data(key: str):
    return redis_client.get(key)
```

### Frontend

1. **Code Splitting**
```typescript
const CandidatesPage = lazy(() => import('./pages/CandidatesPage'));
const AnalyticsPage = lazy(() => import('./pages/AnalyticsPage'));
```

2. **Image Optimization**
```typescript
// Use WebP format with fallback
<picture>
  <source srcSet="image.webp" type="image/webp" />
  <img src="image.jpg" alt="..." />
</picture>
```

3. **CDN for Static Assets**
- Use Cloudflare CDN
- Enable browser caching
- Compress images

---

## Backup Strategy

### Database Backups

**Automated Backups (Render/Railway)**
- Automatic daily backups enabled
- Point-in-time recovery available

**Manual Backups**
```bash
# Backup
pg_dump $DATABASE_URL > backup_$(date +%Y%m%d).sql

# Restore
psql $DATABASE_URL < backup_20250101.sql
```

### File Storage Backups

**S3 Versioning**
```bash
aws s3api put-bucket-versioning \
  --bucket hiresight-files \
  --versioning-configuration Status=Enabled
```

---

## CI/CD Pipeline

### GitHub Actions

**File: `.github/workflows/deploy.yml`**
```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Deploy to Railway
        run: |
          npm install -g @railway/cli
          railway up
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}

  deploy-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Deploy to Vercel
        run: |
          npm install -g vercel
          vercel --prod --token ${{ secrets.VERCEL_TOKEN }}
```

---

## Health Checks

**File: `backend/app/api/v1/health.py`**
```python
@router.get("/health")
async def health_check(db: Session = Depends(get_db)):
    """Health check endpoint"""

    health = {
        "status": "healthy",
        "version": "1.0.0",
        "timestamp": datetime.utcnow().isoformat()
    }

    # Check database
    try:
        db.execute("SELECT 1")
        health["database"] = "connected"
    except:
        health["database"] = "error"
        health["status"] = "unhealthy"

    # Check Redis
    try:
        redis_client.ping()
        health["redis"] = "connected"
    except:
        health["redis"] = "error"
        health["status"] = "unhealthy"

    return health
```

---

## Post-Deployment Checklist

- [ ] Database migrations applied
- [ ] Environment variables configured
- [ ] SSL certificates installed
- [ ] DNS configured correctly
- [ ] Health checks passing
- [ ] Monitoring setup (Sentry, Posthog)
- [ ] Backups configured
- [ ] Rate limiting enabled
- [ ] CORS configured
- [ ] Email sending working
- [ ] File uploads working
- [ ] OAuth working (Google, GitHub)
- [ ] Webhooks working
- [ ] Background jobs running
- [ ] Performance tested
- [ ] Security audit completed

---

## Troubleshooting

### Common Issues

1. **Database Connection Errors**
   - Check DATABASE_URL format
   - Verify firewall rules
   - Check connection pooling settings

2. **Redis Connection Errors**
   - Verify REDIS_HOST and PORT
   - Check Redis password
   - Ensure Redis is running

3. **CORS Errors**
   - Add frontend URL to CORS origins
   - Check Access-Control headers

4. **File Upload Errors**
   - Verify S3 credentials
   - Check bucket permissions
   - Verify file size limits

5. **Slow Performance**
   - Enable database query logging
   - Check slow queries
   - Increase connection pool size
   - Add database indexes

---

## Scaling

### Horizontal Scaling

1. **Backend**: Add more API instances
2. **Workers**: Add more Celery workers
3. **Database**: Read replicas for read-heavy operations
4. **Redis**: Redis Cluster for high availability

### Vertical Scaling

1. Upgrade instance size
2. Increase database resources
3. Optimize queries

---

This deployment guide provides a complete path to production for the HIRESIGHT platform with scalability and reliability in mind.
