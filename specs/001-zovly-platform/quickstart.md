# Quickstart Guide: Zovly Hyperlocal eCommerce Platform

## Overview

This guide provides a step-by-step approach to setting up and developing the Zovly Hyperlocal eCommerce Platform based on the implementation plan and technical specifications.

## Prerequisites

1. Python 3.11+
2. PostgreSQL 15 with PostGIS extension
3. Redis 7
4. Node.js (for npm packages)
5. Docker and Docker Compose
6. Firebase account with Admin SDK enabled
7. Cloudinary account
8. Anthropic API key

## Project Structure

The project follows this structure:

```
backend/
├── app/
│   ├── __init__.py
│   ├── main.py                  # FastAPI app factory, middleware, CORS, router registration
│   ├── core/
│   │   ├── config.py            # Pydantic Settings from .env
│   │   ├── database.py          # Async SQLAlchemy engine + session factory
│   │   ├── redis.py             # Redis connection pool
│   │   ├── security.py          # JWT encode/decode, password hash
│   │   └── dependencies.py      # FastAPI Depends: get_db, get_current_buyer, get_current_seller
│   ├── api/
│   │   └── v1/
│   │       ├── router.py        # Aggregates all v1 routes
│   │       ├── auth.py
│   │       ├── shops.py
│   │       ├── products.py
│   │       ├── discovery.py
│   │       ├── offers.py
│   │       ├── chat.py
│   │       ├── vouchers.py
│   │       └── admin.py
│   ├── models/                  # SQLAlchemy models (one file per entity)
│   ├── schemas/                 # Pydantic schemas (one file per entity)
│   ├── services/                # Business logic (one file per domain)
│   ├── websockets/
│   │   └── chat_handler.py
│   └── workers/
│       └── tasks.py             # ARQ background tasks
├── migrations/
├── tests/
├── .env.example
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

## Setup Instructions

### 1. Environment Configuration

1. Copy `.env.example` to `.env`:
   ```bash
   cp .env.example .env
   ```

2. Fill in all required environment variables:
   - `APP_ENV=development`
   - `SECRET_KEY=` (generate a secure key)
   - `ALLOWED_ORIGINS=http://localhost:3000`
   - `DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/Zovly`
   - `DATABASE_URL_SYNC=postgresql://user:pass@localhost:5432/Zovly`
   - `REDIS_URL=redis://localhost:6379/0`
   - `FIREBASE_PROJECT_ID=`
   - `FIREBASE_SERVICE_ACCOUNT_JSON=`
   - `CLOUDINARY_CLOUD_NAME=`
   - `CLOUDINARY_API_KEY=`
   - `CLOUDINARY_API_SECRET=`
   - `ANTHROPIC_API_KEY=`
   - `ADMIN_SECRET_KEY=`
   - `ADMIN_ALLOWED_IPS=127.0.0.1`

### 2. Database Setup

1. Install PostgreSQL 15 with PostGIS extension:
   ```bash
   # Ubuntu/Debian
   sudo apt-get install postgresql-15 postgis

   # macOS
   brew install postgresql postgis
   ```

2. Create the database:
   ```sql
   CREATE DATABASE Zovly;
   \c Zovly;
   CREATE EXTENSION IF NOT EXISTS postgis;
   ```

### 3. Install Dependencies

1. Create virtual environment:
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

2. Install Python dependencies:
   ```bash
   pip install -r requirements.txt
   ```

### 4. Database Migration

1. Initialize Alembic:
   ```bash
   alembic init migrations
   ```

2. Create initial migration:
   ```bash
   alembic revision --autogenerate -m "Initial migration"
   ```

3. Apply migrations:
   ```bash
   alembic upgrade head
   ```

### 5. Running the Application

1. Start Redis:
   ```bash
   redis-server
   ```

2. Start the FastAPI application:
   ```bash
   uvicorn app.main:app --reload
   ```

3. Start the ARQ worker:
   ```bash
   arq workers.tasks
   ```

4. Run tests:
   ```bash
   pytest tests/
   ```

## Development Workflow

### 1. Testing

Each module should include tests in the `tests/` directory:
- `tests/test_{module}.py` with pytest-asyncio
- Happy path test + auth failure test + validation failure test
- Use `httpx.AsyncClient` with `ASGITransport` for API tests
- Use `pytest-mock` to mock external services

### 2. API Development

Follow these conventions for API development:
- All routes versioned: `/api/v1/`
- JWT Bearer token in Authorization header for all protected routes
- Buyer token ≠ Seller token (different scopes in JWT payload)
- All geo endpoints accept: `lat`, `lng`, `radius_km` query params
- Pagination: cursor-based (not offset) for product/shop lists
- All responses follow envelope: `{ success, data, error, meta }`

### 3. Security

Implement security middleware in this order:
1. `TrustedHostMiddleware` — only allow configured hosts
2. `CORSMiddleware` — ALLOWED_ORIGINS from env
3. Custom `RateLimitMiddleware` — Redis-backed, per IP, per endpoint override
4. Custom `AdminIPWhitelistMiddleware` — block `/api/v1/admin/*` for non-whitelisted IPs
5. `GZipMiddleware` — compress responses > 1000 bytes

## Key Implementation Details

### 1. Geo-Spatial Queries

Use PostGIS for all location-based queries:
- Store shop locations as `Geometry(POINT, 4326)`
- Use `ST_DWithin()` for radius filtering
- Create spatial indexes for performance

### 2. Anonymous Chat

- Implement WebSocket endpoints for real-time communication
- Use Redis pub/sub for message broadcasting
- Generate buyer aliases (`Buyer#XXXX`) for anonymity
- Sanitize messages before storage

### 3. AI Integration

- All Claude API calls are async background tasks
- Queue tasks using ARQ worker system
- Implement error handling for API failures

### 4. Seller Verification

- Implement multi-step verification workflow
- Store CNIC as SHA-256 hash in database
- Use Cloudinary for private photo storage
- Admin dashboard for verification management

## Monitoring and Debugging

1. Enable logging in `config.py`
2. Monitor Redis connections
3. Watch database query performance
4. Track ARQ worker task execution
5. Monitor API response times

## Deployment

Deploy to Railway.app with:
1. PostgreSQL plugin (enable PostGIS via init SQL)
2. Redis plugin
3. Dockerfile for backend service
4. Separate worker service for ARQ tasks
5. Environment variables configured in dashboard
6. Custom domain with SSL