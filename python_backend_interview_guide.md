# Python Backend Developer Interview Guide (4-5 Years Experience)

## Table of Contents

- [Part 1: Scenario-Based Questions (1-100)](#part-1-scenario-based-questions-questions-1-100)
  - [Section 1: API Performance & Optimization (1-20)](#section-1-api-performance--optimization-questions-1-20)
  - [Section 2: Handling Large Files & Data (21-40)](#section-2-handling-large-files--data-questions-21-40)
  - [Section 3: FastAPI Specific Scenarios (41-60)](#section-3-fastapi-specific-scenarios-questions-41-60)
  - [Section 4: Database & Data Handling Scenarios (61-80)](#section-4-database--data-handling-scenarios-questions-61-80)
  - [Section 5: System Design & Architecture Scenarios (81-100)](#section-5-system-design--architecture-scenarios-questions-81-100)
- [Part 2: Core Python & Backend Concepts (101-200)](#part-2-core-python--backend-concepts-questions-101-200)
  - [Section 6: Python Core Concepts (101-125)](#section-6-python-core-concepts-questions-101-125)
  - [Section 7: File Handling (126-140)](#section-7-file-handling-questions-126-140)
  - [Section 8: FastAPI Framework (141-165)](#section-8-fastapi-framework-questions-141-165)
  - [Section 9: Database & ORM (166-185)](#section-9-database--orm-questions-166-185)
  - [Section 10: API Design & Best Practices (186-200)](#section-10-api-design--best-practices-questions-186-200)

---

## Part 1: Scenario-Based Questions (Questions 1-100)

### Section 1: API Performance & Optimization (Questions 1-20)

### Question 1: Optimizing Slow API Endpoints

**Question:** Your FastAPI endpoint that fetches user dashboards is taking 3-4 seconds to respond. Users are complaining. Walk me through your systematic approach to diagnose and fix the performance issue.

**Answer:**
Profile first (never guess), then apply targeted fixes: database query optimization, caching, async I/O, and connection pooling.

**Detailed Explanation:**
Performance optimization should be data-driven. Start with profiling tools like `cProfile`, `py-spy`, or middleware-based timing. Common culprits: N+1 queries, synchronous I/O in async context, missing indexes, no caching, serialization overhead. Fix in order of impact: DB queries usually give the biggest win, then caching, then async refactoring.

**Code Example:**
```python
import time
import cProfile
from fastapi import FastAPI, Request
from functools import wraps

app = FastAPI()

# Step 1: Add timing middleware to identify slow endpoints
@app.middleware("http")
async def timing_middleware(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    duration = time.perf_counter() - start
    response.headers["X-Process-Time"] = f"{duration:.4f}s"
    if duration > 1.0:  # Log slow requests
        print(f"SLOW REQUEST: {request.url.path} took {duration:.4f}s")
    return response

# Step 2: Profile the actual function
import cProfile, pstats, io

def profile_endpoint(func):
    @wraps(func)
    async def wrapper(*args, **kwargs):
        pr = cProfile.Profile()
        pr.enable()
        result = await func(*args, **kwargs)
        pr.disable()
        s = io.StringIO()
        ps = pstats.Stats(pr, stream=s).sort_stats("cumulative")
        ps.print_stats(20)
        print(s.getvalue())
        return result
    return wrapper

# Step 3: Fix - use select_related to avoid N+1
from sqlalchemy.orm import selectinload
from sqlalchemy.ext.asyncio import AsyncSession

async def get_dashboard_optimized(user_id: int, db: AsyncSession):
    # BAD: N+1 problem
    # user = await db.get(User, user_id)
    # orders = await db.execute(select(Order).where(Order.user_id == user_id))

    # GOOD: Single query with eager loading
    result = await db.execute(
        select(User)
        .options(selectinload(User.orders).selectinload(Order.items))
        .where(User.id == user_id)
    )
    return result.scalar_one_or_none()

# Step 4: Add Redis caching
import redis.asyncio as aioredis
import json

redis_client = aioredis.from_url("redis://localhost")

async def get_cached_dashboard(user_id: int):
    cache_key = f"dashboard:{user_id}"
    cached = await redis_client.get(cache_key)
    if cached:
        return json.loads(cached)
    
    data = await fetch_dashboard_from_db(user_id)
    await redis_client.setex(cache_key, 300, json.dumps(data))  # 5 min TTL
    return data
```

**Key Takeaways:**
- Always profile before optimizing — measure, don't guess
- Database N+1 queries are the #1 cause of slow APIs; use eager loading
- Redis caching with appropriate TTL can reduce DB load by 80%+
- Timing middleware gives real-world performance data per endpoint

---

### Question 2: Redis Caching Strategy for REST APIs

**Question:** Design a caching strategy using Redis for a product catalog API that serves millions of requests daily. How do you handle cache invalidation, cache stampede, and partial cache updates?

**Answer:**
Use a layered cache strategy: cache-aside pattern with TTL, implement cache warming, use Redis locks to prevent stampede, and use pub/sub or versioned keys for invalidation.

**Detailed Explanation:**
Cache-aside (lazy loading) is the most common pattern: check cache first, on miss fetch from DB and populate cache. Cache stampede (thundering herd) occurs when many requests miss simultaneously — solve with probabilistic early expiration or a distributed lock. For invalidation, use event-driven invalidation or versioned cache keys to avoid stale data.

**Code Example:**
```python
import redis.asyncio as aioredis
import json
import asyncio
from typing import Optional, Any

redis = aioredis.from_url("redis://localhost", decode_responses=True)

class CacheService:
    LOCK_TIMEOUT = 10  # seconds
    
    @staticmethod
    async def get_or_set(
        key: str, 
        fetch_fn, 
        ttl: int = 300,
        lock_ttl: int = 10
    ) -> Any:
        """Cache-aside with stampede protection using Redis lock."""
        # Try cache first
        cached = await redis.get(key)
        if cached:
            return json.loads(cached)
        
        # Acquire lock to prevent stampede
        lock_key = f"lock:{key}"
        acquired = await redis.set(lock_key, "1", nx=True, ex=lock_ttl)
        
        if acquired:
            try:
                data = await fetch_fn()
                await redis.setex(key, ttl, json.dumps(data))
                return data
            finally:
                await redis.delete(lock_key)
        else:
            # Wait for the lock holder to populate cache
            for _ in range(20):
                await asyncio.sleep(0.5)
                cached = await redis.get(key)
                if cached:
                    return json.loads(cached)
            # Fallback: fetch directly
            return await fetch_fn()
    
    @staticmethod
    async def invalidate_pattern(pattern: str):
        """Invalidate all keys matching a pattern."""
        async for key in redis.scan_iter(match=pattern):
            await redis.delete(key)
    
    @staticmethod
    async def warm_cache(product_ids: list[int]):
        """Pre-warm cache for popular products."""
        pipe = redis.pipeline()
        for pid in product_ids:
            data = await fetch_product_from_db(pid)
            pipe.setex(f"product:{pid}", 600, json.dumps(data))
        await pipe.execute()

# Probabilistic early expiration (avoid stampede without locks)
import math, random

async def get_with_early_expire(key: str, fetch_fn, ttl: int = 300, beta: float = 1.0):
    """XFetch algorithm for probabilistic cache refresh."""
    cached_raw = await redis.get(key)
    if cached_raw:
        data = json.loads(cached_raw)
        remaining_ttl = await redis.ttl(key)
        # Probabilistically refresh before expiry
        if remaining_ttl - beta * math.log(random.random()) < 0:
            data = await fetch_fn()
            await redis.setex(key, ttl, json.dumps(data))
        return data
    
    data = await fetch_fn()
    await redis.setex(key, ttl, json.dumps(data))
    return data

# Cache invalidation via versioned keys
async def get_product_v(product_id: int, version: int = None):
    if version is None:
        version = await redis.get(f"product_version:{product_id}") or 1
    key = f"product:{product_id}:v{version}"
    return await CacheService.get_or_set(key, lambda: fetch_product_from_db(product_id))

async def invalidate_product(product_id: int):
    """Bump version to invalidate without deleting."""
    await redis.incr(f"product_version:{product_id}")
```

**Key Takeaways:**
- Cache-aside pattern: check cache → DB miss → populate cache
- Use Redis SET NX (atomic lock) to prevent thundering herd on cache misses
- Versioned cache keys allow instant invalidation without scanning
- Pipeline Redis commands for bulk operations to reduce round-trips

---

### Question 3: Database Query Optimization for High-Traffic APIs

**Question:** Your user search API returns results in 800ms. It queries a PostgreSQL table with 10 million users. The query does a LIKE search on email and filters by status and created_at. How do you optimize it?

**Answer:**
Add composite indexes, switch LIKE to full-text search or trigram indexes (pg_trgm), use query explains, and consider materialized views for complex aggregations.

**Detailed Explanation:**
PostgreSQL query optimization hierarchy: 1) Use EXPLAIN ANALYZE to understand the query plan, 2) Add appropriate indexes (B-tree for equality/range, GIN/GiST for text search), 3) Rewrite the query to use indexes effectively (avoid leading wildcards in LIKE), 4) Use connection pooling (PgBouncer), 5) Consider read replicas for heavy read loads.

**Code Example:**
```python
from sqlalchemy import text, Index, create_engine
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import String, DateTime, Enum as SAEnum
from datetime import datetime

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255))
    status: Mapped[str] = mapped_column(String(20))
    created_at: Mapped[datetime] = mapped_column(DateTime)
    
    # Composite index for common query pattern
    __table_args__ = (
        Index("ix_users_status_created", "status", "created_at"),
        Index("ix_users_email_gin", "email", postgresql_using="gin",
              postgresql_ops={"email": "gin_trgm_ops"}),  # trigram for LIKE
    )

# Migration to add pg_trgm extension and index
MIGRATION_SQL = """
-- Enable trigram extension
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Trigram index for fast LIKE/ILIKE on email  
CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_users_email_trgm 
    ON users USING gin (email gin_trgm_ops);

-- Composite index for status + date range queries
CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_users_status_created 
    ON users (status, created_at DESC);

-- Partial index for active users only (most queries filter by status='active')
CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_users_active_created
    ON users (created_at DESC)
    WHERE status = 'active';
"""

# Optimized query with SQLAlchemy
from sqlalchemy import select, and_

async def search_users_optimized(
    db: AsyncSession,
    email_query: str,
    status: str,
    page: int = 1,
    page_size: int = 20
):
    # Use % wrapping for trigram similarity search
    stmt = (
        select(User)
        .where(
            and_(
                User.email.ilike(f"%{email_query}%"),  # uses trigram index
                User.status == status,
            )
        )
        .order_by(User.created_at.desc())
        .limit(page_size)
        .offset((page - 1) * page_size)
    )
    
    result = await db.execute(stmt)
    return result.scalars().all()

# Always check query plan in development
async def explain_query(db: AsyncSession, email_query: str):
    explain = await db.execute(
        text("""
        EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
        SELECT * FROM users
        WHERE email ILIKE :email AND status = 'active'
        ORDER BY created_at DESC
        LIMIT 20
        """),
        {"email": f"%{email_query}%"}
    )
    return explain.scalar()
```

**Key Takeaways:**
- Always run EXPLAIN ANALYZE before and after optimization to verify index usage
- pg_trgm enables fast LIKE/ILIKE with GIN indexes — critical for text search
- Partial indexes on commonly filtered subsets dramatically reduce index size
- CONCURRENTLY creates indexes without locking the table in production

---

### Question 4: Connection Pooling for Database Performance

**Question:** Your API has 200 concurrent users hitting a PostgreSQL database. You're seeing 'too many connections' errors and connection timeouts. How do you implement and tune connection pooling?

**Answer:**
Use SQLAlchemy's built-in pool with appropriate settings, deploy PgBouncer as an external pooler for production, and configure pool_size/max_overflow based on your PostgreSQL max_connections.

**Detailed Explanation:**
PostgreSQL has a hard limit on connections (default 100). Each connection uses ~5-10MB RAM. SQLAlchemy's pool manages connections at the application level. For production with multiple worker processes, PgBouncer (transaction-mode pooling) is essential — it multiplexes thousands of app connections to a few dozen DB connections.

**Code Example:**
```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.pool import AsyncAdaptedQueuePool
import asyncio

# Application-level connection pool configuration
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    pool_size=20,          # Base connections kept open
    max_overflow=10,       # Extra connections when pool is full (total: 30)
    pool_timeout=30,       # Seconds to wait for a connection
    pool_recycle=1800,     # Recycle connections after 30 min (avoids stale connections)
    pool_pre_ping=True,    # Test connection health before use
    echo_pool="debug",     # Log pool events (disable in production)
)

AsyncSessionLocal = async_sessionmaker(
    engine, expire_on_commit=False, class_=AsyncSession
)

# Dependency for FastAPI
from fastapi import FastAPI, Depends

app = FastAPI()

async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

# Monitor pool health
async def get_pool_status():
    pool = engine.pool
    return {
        "size": pool.size(),
        "checked_in": pool.checkedin(),
        "checked_out": pool.checkedout(),
        "overflow": pool.overflow(),
        "invalid": pool.invalid(),
    }

@app.get("/health/db")
async def db_health():
    status = await get_pool_status()
    return status

# PgBouncer configuration (pgbouncer.ini)
PGBOUNCER_CONFIG = """
[databases]
mydb = host=postgres-server port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction          ; Best for async apps
max_client_conn = 1000           ; App connections to PgBouncer
default_pool_size = 25           ; PgBouncer -> PostgreSQL connections
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3
server_idle_timeout = 600
"""

# With PgBouncer, use smaller SQLAlchemy pool
engine_with_pgbouncer = create_async_engine(
    "postgresql+asyncpg://user:pass@pgbouncer-host:6432/mydb",
    pool_size=5,       # PgBouncer handles the real pooling
    max_overflow=0,    # No overflow needed
    pool_timeout=10,
)
```

**Key Takeaways:**
- pool_size + max_overflow = max simultaneous DB connections per worker process
- pool_pre_ping=True prevents errors from stale connections (e.g., after network blip)
- PgBouncer transaction-mode pooling is ideal for async apps — connections returned immediately after each transaction
- With 4 Gunicorn workers each having pool_size=5, you use at most 20 DB connections

---

### Question 5: Implementing Async Processing for Better Throughput

**Question:** Your API endpoint processes images synchronously: resize, compress, upload to S3. This takes 5-10 seconds and blocks the response. How do you redesign this for non-blocking performance?

**Answer:**
Return immediately with a job ID, process asynchronously using Celery + Redis/RabbitMQ as broker, and provide a status polling endpoint or WebSocket for completion notification.

**Detailed Explanation:**
Async processing decouples request acceptance from work completion. The API returns a 202 Accepted with a job ID. Celery workers process the image in background. Client polls /jobs/{id}/status or receives a webhook/WebSocket notification when done. This pattern handles spikes and prevents timeouts.

**Code Example:**
```python
from fastapi import FastAPI, BackgroundTasks, UploadFile, HTTPException
from celery import Celery
import uuid
import redis

app = FastAPI()
redis_client = redis.Redis(host="localhost", decode_responses=True)

# Celery configuration
celery_app = Celery(
    "tasks",
    broker="redis://localhost:6379/0",
    backend="redis://localhost:6379/1",
)
celery_app.conf.update(
    task_serializer="json",
    result_expires=3600,
    worker_prefetch_multiplier=1,  # One task at a time per worker
    task_acks_late=True,           # Ack only after completion (safer)
)

# Celery task for image processing
@celery_app.task(bind=True, max_retries=3, default_retry_delay=60)
def process_image_task(self, job_id: str, file_path: str, user_id: int):
    try:
        redis_client.hset(f"job:{job_id}", mapping={"status": "processing", "progress": "0"})
        
        # Step 1: Resize
        resized_path = resize_image(file_path)
        redis_client.hset(f"job:{job_id}", "progress", "33")
        
        # Step 2: Compress
        compressed_path = compress_image(resized_path)
        redis_client.hset(f"job:{job_id}", "progress", "66")
        
        # Step 3: Upload to S3
        s3_url = upload_to_s3(compressed_path, f"users/{user_id}/images/")
        redis_client.hset(f"job:{job_id}", mapping={
            "status": "completed",
            "progress": "100",
            "result_url": s3_url,
        })
        redis_client.expire(f"job:{job_id}", 3600)
        return {"status": "completed", "url": s3_url}
    
    except Exception as exc:
        redis_client.hset(f"job:{job_id}", "status", "failed")
        raise self.retry(exc=exc)

# FastAPI endpoint
@app.post("/upload/image", status_code=202)
async def upload_image(file: UploadFile, user_id: int):
    job_id = str(uuid.uuid4())
    
    # Save file temporarily
    file_path = f"/tmp/{job_id}_{file.filename}"
    with open(file_path, "wb") as f:
        content = await file.read()
        f.write(content)
    
    # Queue async task
    redis_client.hset(f"job:{job_id}", mapping={"status": "queued", "progress": "0"})
    process_image_task.delay(job_id, file_path, user_id)
    
    return {"job_id": job_id, "status_url": f"/jobs/{job_id}/status"}

@app.get("/jobs/{job_id}/status")
async def get_job_status(job_id: str):
    job = redis_client.hgetall(f"job:{job_id}")
    if not job:
        raise HTTPException(404, "Job not found")
    return job
```

**Key Takeaways:**
- 202 Accepted + job ID pattern decouples submission from completion
- Celery task_acks_late=True prevents task loss if worker crashes mid-execution
- Store job progress in Redis for real-time status polling
- Set result TTL to avoid Redis memory bloat from completed jobs

---

### Question 6: Rate Limiting and Throttling API Endpoints

**Question:** You need to implement rate limiting for your public API: 100 requests/minute per IP for unauthenticated users, 1000/minute per API key for authenticated users. How do you implement this without a single point of failure?

**Answer:**
Use Redis sliding window or token bucket algorithm via middleware. Use slowapi library for FastAPI. For distributed deployments, use Redis atomic Lua scripts to ensure accuracy across instances.

**Detailed Explanation:**
Rate limiting strategies: Fixed window (simple but bursting possible), Sliding window log (accurate), Token bucket (allows bursts), Leaky bucket (smooth output). For production, sliding window with Redis is the sweet spot. Use Lua scripts for atomicity since INCR+EXPIRE isn't atomic.

**Code Example:**
```python
import time
import redis.asyncio as aioredis
from fastapi import FastAPI, Request, HTTPException, Depends
from fastapi.responses import JSONResponse

app = FastAPI()
redis = aioredis.from_url("redis://localhost")

# Sliding window rate limiter using Redis sorted sets
async def sliding_window_limit(
    key: str, 
    limit: int, 
    window: int = 60
) -> tuple[bool, dict]:
    """Returns (is_allowed, rate_limit_info)"""
    now = time.time()
    window_start = now - window
    
    pipe = redis.pipeline()
    # Remove old entries outside the window
    pipe.zremrangebyscore(key, 0, window_start)
    # Count remaining entries
    pipe.zcard(key)
    # Add current request timestamp
    pipe.zadd(key, {str(now): now})
    # Set expiry
    pipe.expire(key, window + 1)
    results = await pipe.execute()
    
    current_count = results[1]
    allowed = current_count < limit
    
    return allowed, {
        "limit": limit,
        "remaining": max(0, limit - current_count - 1),
        "reset": int(now + window),
    }

# FastAPI middleware for rate limiting
@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    # Determine rate limit based on auth
    api_key = request.headers.get("X-API-Key")
    
    if api_key:
        key = f"ratelimit:key:{api_key}"
        limit = 1000
    else:
        ip = request.client.host
        key = f"ratelimit:ip:{ip}"
        limit = 100
    
    allowed, info = await sliding_window_limit(key, limit, window=60)
    
    if not allowed:
        return JSONResponse(
            status_code=429,
            content={"error": "Rate limit exceeded", "retry_after": 60},
            headers={
                "X-RateLimit-Limit": str(info["limit"]),
                "X-RateLimit-Remaining": "0",
                "X-RateLimit-Reset": str(info["reset"]),
                "Retry-After": "60",
            }
        )
    
    response = await call_next(request)
    response.headers["X-RateLimit-Limit"] = str(info["limit"])
    response.headers["X-RateLimit-Remaining"] = str(info["remaining"])
    return response

# Using slowapi library (recommended for FastAPI)
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address, storage_uri="redis://localhost")
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.get("/api/data")
@limiter.limit("100/minute")
async def get_data(request: Request):
    return {"data": "..."}
```

**Key Takeaways:**
- Sliding window prevents burst exploitation at window boundaries (unlike fixed window)
- Pipeline Redis commands for atomicity and reduced round-trips
- Return X-RateLimit headers so clients can self-throttle
- Use different limits for authenticated vs anonymous users

---

### Question 7: Load Testing and Profiling API Performance

**Question:** Before going live with a new API, how do you load test it? Walk through your toolchain, what metrics you measure, and how you interpret the results to make optimization decisions.

**Answer:**
Use locust or k6 for load testing, py-spy/cProfile for profiling, Prometheus + Grafana for metrics. Measure p50/p95/p99 latency, throughput (RPS), error rate, and resource utilization.

**Detailed Explanation:**
Load testing validates performance under realistic conditions. Key phases: smoke test (1-5 users), load test (expected peak), stress test (2-3x peak), spike test (sudden surge). p95/p99 latency matters more than average — the 99th percentile is what your worst 1% of users experience. Profile under load to find real bottlenecks, not synthetic ones.

**Code Example:**
```python
# locustfile.py - Load testing with Locust
from locust import HttpUser, task, between, events
import random

class APIUser(HttpUser):
    wait_time = between(0.5, 2)  # Simulate think time
    
    def on_start(self):
        """Called once per user — authenticate."""
        response = self.client.post("/auth/token", json={
            "username": f"user{random.randint(1, 1000)}",
            "password": "testpass"
        })
        self.token = response.json()["access_token"]
        self.headers = {"Authorization": f"Bearer {self.token}"}
    
    @task(3)  # Weight: 3x more likely than other tasks
    def get_user_profile(self):
        user_id = random.randint(1, 10000)
        self.client.get(f"/users/{user_id}", headers=self.headers,
                       name="/users/[id]")  # Group in reports
    
    @task(1)
    def search_products(self):
        query = random.choice(["laptop", "phone", "tablet"])
        self.client.get(f"/products?q={query}&page=1", headers=self.headers)
    
    @task(1)
    def create_order(self):
        with self.client.post("/orders", json={
            "product_id": random.randint(1, 100),
            "quantity": random.randint(1, 5)
        }, headers=self.headers, catch_response=True) as response:
            if response.status_code == 201:
                response.success()
            else:
                response.failure(f"Unexpected status: {response.status_code}")

# Run: locust -f locustfile.py --host=http://localhost:8000 --users=200 --spawn-rate=10

# Profiling with py-spy (attach to running process)
# py-spy record -o profile.svg --pid <PID> --duration 30

# FastAPI with built-in metrics using Prometheus
from prometheus_fastapi_instrumentator import Instrumentator
from fastapi import FastAPI

app = FastAPI()
Instrumentator().instrument(app).expose(app)

# Custom histogram for business metrics
from prometheus_client import Histogram, Counter
REQUEST_DURATION = Histogram("api_request_duration_seconds", 
                              "Request duration", 
                              ["endpoint", "method", "status"])
DB_QUERY_DURATION = Histogram("db_query_duration_seconds", "DB query time")

import time
from contextlib import contextmanager

@contextmanager
def track_db_time():
    start = time.perf_counter()
    try:
        yield
    finally:
        DB_QUERY_DURATION.observe(time.perf_counter() - start)
```

**Key Takeaways:**
- Use p99 latency as your SLA target — averages hide the worst user experiences
- Profile under actual load (py-spy) — hotspots change under concurrency
- Ramp up load gradually to find the knee in the throughput/latency curve
- Test the full stack (DB, cache, external services) not just the API in isolation

---

### Question 8: HTTP Caching Headers for API Performance

**Question:** Explain how you would implement HTTP caching headers in your FastAPI API to reduce server load. When would you use ETag vs Cache-Control vs Last-Modified, and what are the tradeoffs?

**Answer:**
Use Cache-Control for time-based caching, ETag for content-based validation, and Last-Modified for time-based validation. Combine them for optimal browser and CDN caching behavior.

**Detailed Explanation:**
HTTP caching works at multiple levels: browser cache, CDN, reverse proxy. Cache-Control: max-age tells clients how long to cache. ETag is a hash of the content; clients send If-None-Match to check freshness without downloading full response (304 Not Modified). Last-Modified + If-Modified-Since works similarly but uses timestamps. Use ETags for dynamic content, Cache-Control for static/semi-static data.

**Code Example:**
```python
import hashlib
import json
from datetime import datetime, timezone
from fastapi import FastAPI, Request, Response
from fastapi.responses import JSONResponse

app = FastAPI()

def generate_etag(data: dict) -> str:
    """Generate ETag from response content."""
    content = json.dumps(data, sort_keys=True, default=str)
    return hashlib.sha256(content.encode()).hexdigest()[:16]

@app.get("/products/{product_id}")
async def get_product(product_id: int, request: Request, response: Response):
    product = await fetch_product(product_id)
    
    if not product:
        return JSONResponse(status_code=404, content={"error": "Not found"})
    
    # Generate ETag
    etag = f'"{generate_etag(product)}"'
    last_modified = product["updated_at"].strftime("%a, %d %b %Y %H:%M:%S GMT")
    
    # Check If-None-Match (ETag validation)
    if request.headers.get("If-None-Match") == etag:
        return Response(status_code=304, headers={"ETag": etag})
    
    # Check If-Modified-Since
    if_modified_since = request.headers.get("If-Modified-Since")
    if if_modified_since:
        client_time = datetime.strptime(if_modified_since, "%a, %d %b %Y %H:%M:%S GMT")
        if product["updated_at"].replace(tzinfo=None) <= client_time:
            return Response(status_code=304, headers={"Last-Modified": last_modified})
    
    # Set caching headers
    response.headers["Cache-Control"] = "public, max-age=300, stale-while-revalidate=60"
    response.headers["ETag"] = etag
    response.headers["Last-Modified"] = last_modified
    response.headers["Vary"] = "Accept-Encoding, Accept-Language"
    
    return product

# Different cache policies per endpoint type
class CachePolicy:
    STATIC = "public, max-age=86400, immutable"      # 24h for static
    DYNAMIC = "private, max-age=0, must-revalidate"   # No cache for user data
    SEMI_STATIC = "public, max-age=300, stale-while-revalidate=60"  # 5min
    NO_CACHE = "no-store"  # Sensitive data

@app.get("/config")
async def get_config(response: Response):
    """Rarely changes — long cache."""
    response.headers["Cache-Control"] = CachePolicy.STATIC
    return {"version": "1.0", "features": [...]}

@app.get("/user/profile")
async def get_user_profile(response: Response):
    """User-specific — no public cache."""
    response.headers["Cache-Control"] = CachePolicy.DYNAMIC
    return {"user": "..."}

@app.get("/products")
async def list_products(response: Response):
    """Changes occasionally — short cache."""
    response.headers["Cache-Control"] = CachePolicy.SEMI_STATIC
    return {"products": [...]}
```

**Key Takeaways:**
- ETag + If-None-Match: server sends 304 instead of full response if unchanged (saves bandwidth)
- stale-while-revalidate allows serving stale content while refreshing in background
- private vs public: private caches only in browser, public allows CDN caching
- Always set Vary header when response depends on request headers (e.g., Accept-Encoding)

---

### Question 9: Efficient Pagination and Filtering for Large Datasets

**Question:** Your product listing API uses OFFSET pagination and is slow for high page numbers (page 500+). The table has 5 million rows. How do you fix the pagination and implement efficient filtering?

**Answer:**
Replace OFFSET with cursor-based (keyset) pagination using the last seen ID/timestamp. Add composite indexes to match filter + sort patterns. Use search-specific tools (Elasticsearch) for complex filtering.

**Detailed Explanation:**
OFFSET pagination requires scanning and discarding all preceding rows — OFFSET 10000 LIMIT 20 reads 10,020 rows. Cursor pagination uses WHERE id > last_seen_id LIMIT 20, which uses the index directly. The tradeoff: cursor pagination can't jump to arbitrary pages, but APIs rarely need that.

**Code Example:**
```python
from fastapi import FastAPI, Query
from sqlalchemy import select, and_
from sqlalchemy.ext.asyncio import AsyncSession
from typing import Optional
from datetime import datetime
import base64
import json

app = FastAPI()

# BAD: Offset pagination (slow at high offsets)
async def get_products_offset(db: AsyncSession, page: int, page_size: int = 20):
    offset = (page - 1) * page_size
    # SLOW: PostgreSQL scans and discards `offset` rows
    result = await db.execute(
        select(Product).order_by(Product.id).offset(offset).limit(page_size)
    )
    return result.scalars().all()

# GOOD: Cursor-based pagination
def encode_cursor(product_id: int, created_at: datetime) -> str:
    """Encode cursor as base64 JSON for opacity."""
    data = {"id": product_id, "ts": created_at.isoformat()}
    return base64.urlsafe_b64encode(json.dumps(data).encode()).decode()

def decode_cursor(cursor: str) -> dict:
    return json.loads(base64.urlsafe_b64decode(cursor.encode()))

async def get_products_cursor(
    db: AsyncSession,
    cursor: Optional[str] = None,
    page_size: int = 20,
    category: Optional[str] = None,
    min_price: Optional[float] = None,
    max_price: Optional[float] = None,
):
    conditions = []
    
    # Filtering conditions
    if category:
        conditions.append(Product.category == category)
    if min_price is not None:
        conditions.append(Product.price >= min_price)
    if max_price is not None:
        conditions.append(Product.price <= max_price)
    
    # Cursor condition (keyset pagination)
    if cursor:
        cursor_data = decode_cursor(cursor)
        conditions.append(
            # Uses composite index on (created_at, id)
            (Product.created_at < cursor_data["ts"]) |
            (
                (Product.created_at == cursor_data["ts"]) &
                (Product.id < cursor_data["id"])
            )
        )
    
    stmt = (
        select(Product)
        .where(and_(*conditions))
        .order_by(Product.created_at.desc(), Product.id.desc())
        .limit(page_size + 1)  # Fetch one extra to determine has_next
    )
    
    result = await db.execute(stmt)
    products = result.scalars().all()
    
    has_next = len(products) > page_size
    if has_next:
        products = products[:page_size]
    
    next_cursor = None
    if has_next and products:
        last = products[-1]
        next_cursor = encode_cursor(last.id, last.created_at)
    
    return {"items": products, "next_cursor": next_cursor, "has_next": has_next}

@app.get("/products")
async def list_products(
    cursor: Optional[str] = Query(None),
    page_size: int = Query(20, ge=1, le=100),
    category: Optional[str] = None,
    db: AsyncSession = Depends(get_db)
):
    return await get_products_cursor(db, cursor, page_size, category)
```

**Key Takeaways:**
- Cursor pagination is O(log n) via index; OFFSET pagination is O(n) — critical at scale
- Encode cursors as opaque base64 to prevent clients from manipulating them
- Fetch page_size + 1 rows to determine has_next_page without a COUNT query
- Create composite indexes matching your filter + sort column combination

---

### Question 10: Response Compression with Gzip and Brotli

**Question:** Your API responses are large JSON payloads (500KB+). How do you implement compression to reduce bandwidth and improve response times? Compare gzip vs brotli and when to use each.

**Answer:**
Add GZipMiddleware in FastAPI for gzip compression. Use Brotli for modern clients (better compression ratio). Only compress responses >1KB — small responses cost more CPU than bandwidth saved.

**Detailed Explanation:**
Brotli achieves 15-30% better compression than gzip but requires more CPU. Gzip is universally supported. Most CDNs handle compression at the edge, so you may not need it at the app layer. The Content-Encoding negotiation via Accept-Encoding header determines which algorithm to use.

**Code Example:**
```python
from fastapi import FastAPI
from fastapi.middleware.gzip import GZipMiddleware
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response
import brotli
import gzip

app = FastAPI()

# Built-in GZip middleware (minimum_size in bytes)
app.add_middleware(GZipMiddleware, minimum_size=1000)

# Custom Brotli middleware with fallback to gzip
class CompressionMiddleware(BaseHTTPMiddleware):
    MINIMUM_SIZE = 1000  # bytes
    
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        # Only compress text-based content
        content_type = response.headers.get("content-type", "")
        if not any(t in content_type for t in ["json", "text", "xml", "javascript"]):
            return response
        
        accept_encoding = request.headers.get("accept-encoding", "")
        body = b"".join([chunk async for chunk in response.body_iterator])
        
        if len(body) < self.MINIMUM_SIZE:
            return Response(content=body, status_code=response.status_code,
                          headers=dict(response.headers), media_type=response.media_type)
        
        if "br" in accept_encoding:
            compressed = brotli.compress(body, quality=4)  # quality 0-11
            encoding = "br"
        elif "gzip" in accept_encoding:
            compressed = gzip.compress(body, compresslevel=6)
            encoding = "gzip"
        else:
            return Response(content=body, status_code=response.status_code,
                          headers=dict(response.headers), media_type=response.media_type)
        
        headers = dict(response.headers)
        headers["Content-Encoding"] = encoding
        headers["Content-Length"] = str(len(compressed))
        headers["Vary"] = "Accept-Encoding"
        
        return Response(
            content=compressed,
            status_code=response.status_code,
            headers=headers,
            media_type=response.media_type,
        )

app.add_middleware(CompressionMiddleware)

# Example: Compressing a large response
@app.get("/reports/large")
async def large_report():
    # 500KB+ JSON response
    data = {"records": [{"id": i, "data": "x" * 100} for i in range(5000)]}
    return data  # Automatically compressed by middleware

# Benchmark compression savings
def benchmark_compression(data: bytes):
    gzip_compressed = gzip.compress(data, compresslevel=6)
    brotli_compressed = brotli.compress(data, quality=6)
    
    print(f"Original:  {len(data):,} bytes")
    print(f"Gzip:      {len(gzip_compressed):,} bytes ({100*(1-len(gzip_compressed)/len(data)):.1f}% smaller)")
    print(f"Brotli:    {len(brotli_compressed):,} bytes ({100*(1-len(brotli_compressed)/len(data)):.1f}% smaller)")
```

**Key Takeaways:**
- Brotli provides 15-30% better compression than gzip but only for modern browsers/clients
- Set minimum_size to ~1KB — compressing tiny responses wastes CPU
- Add Vary: Accept-Encoding so CDNs cache both compressed and uncompressed versions separately
- Let your CDN/nginx handle compression to offload CPU from the application server

---

### Question 11: API Response Time SLAs and Monitoring

**Question:** How do you set up monitoring to ensure your API meets response time SLAs (e.g., p95 < 200ms)? What tools, metrics, and alerting would you implement?

**Answer:**
Use Prometheus for metrics collection, Grafana for dashboards, and AlertManager for alerting. Track p50/p95/p99 latency histograms, error rates, and throughput. Set alerts on SLA breaches.

**Detailed Explanation:**
The four golden signals for monitoring: Latency, Traffic (throughput), Errors, Saturation (resource usage). Histograms are preferred over averages because they capture the distribution. Apdex score (Application Performance Index) quantifies user satisfaction as a single number.

**Code Example:**
```python
from prometheus_client import Histogram, Counter, Gauge
from fastapi import FastAPI, Request
import time
import asyncio

app = FastAPI()

# Define metrics
REQUEST_DURATION = Histogram(
    "http_request_duration_seconds",
    "HTTP request duration",
    ["method", "endpoint", "status_code"],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
)

REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status_code"],
)

ACTIVE_REQUESTS = Gauge(
    "http_active_requests",
    "Currently active requests",
)

DB_QUERY_DURATION = Histogram(
    "db_query_duration_seconds",
    "Database query duration",
    ["query_type", "table"],
    buckets=[0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0],
)

@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    ACTIVE_REQUESTS.inc()
    start = time.perf_counter()
    
    try:
        response = await call_next(request)
        duration = time.perf_counter() - start
        
        # Normalize path (replace IDs with placeholders)
        path = request.url.path
        for part in path.split("/"):
            if part.isdigit():
                path = path.replace(part, "{id}", 1)
        
        REQUEST_DURATION.labels(
            method=request.method,
            endpoint=path,
            status_code=response.status_code,
        ).observe(duration)
        
        REQUEST_COUNT.labels(
            method=request.method,
            endpoint=path,
            status_code=response.status_code,
        ).inc()
        
        return response
    finally:
        ACTIVE_REQUESTS.dec()

# Prometheus alerting rules (prometheus.yml)
ALERT_RULES = """
groups:
  - name: api_sla
    rules:
      - alert: HighP95Latency
        expr: |
          histogram_quantile(0.95, 
            rate(http_request_duration_seconds_bucket[5m])
          ) > 0.2
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "p95 latency > 200ms"
          
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{status_code=~"5.."}[5m]) /
          rate(http_requests_total[5m]) > 0.01
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Error rate > 1%"
"""

# Apdex score calculation
def calculate_apdex(satisfied_threshold: float = 0.1, 
                   tolerating_threshold: float = 0.4):
    """Apdex = (Satisfied + Tolerating/2) / Total"""
    # satisfied: response < T, tolerating: T <= response < 4T, frustrated: >= 4T
    pass  # Implemented via Prometheus recording rules
```

**Key Takeaways:**
- Track histograms not averages — p95/p99 expose tail latency SLA violations
- Normalize path parameters (/users/123 → /users/{id}) to avoid metric cardinality explosion
- Set alerts with for: 2m to avoid flapping on transient spikes
- The four golden signals (Latency, Traffic, Errors, Saturation) cover all key API health aspects

---

### Question 12: Async vs Sync Endpoint Performance in FastAPI

**Question:** When should you use async def vs def for FastAPI route handlers? What happens if you use async def with a blocking operation like synchronous SQLAlchemy?

**Answer:**
Use async def for I/O-bound operations (database, HTTP calls, file I/O) with async libraries. Use def (sync) for CPU-bound work — FastAPI runs it in a thread pool automatically. Never use blocking I/O inside async def — it blocks the event loop.

**Detailed Explanation:**
FastAPI runs on uvicorn's asyncio event loop. async def handlers run on the event loop directly. def handlers run in a thread pool executor, so they don't block the event loop. A blocking call (time.sleep, synchronous psycopg2) inside async def freezes all other requests because it blocks the single event loop thread.

**Code Example:**
```python
import asyncio
import time
from fastapi import FastAPI
import httpx

app = FastAPI()

# BAD: Blocking call in async function (freezes event loop)
@app.get("/bad-async")
async def bad_async_endpoint():
    time.sleep(2)  # BLOCKS THE EVENT LOOP!
    # While sleeping, NO other request can be served
    return {"result": "done"}

# GOOD: Non-blocking async call
@app.get("/good-async")  
async def good_async_endpoint():
    await asyncio.sleep(2)  # Yields control to event loop
    # Other requests can be served during sleep
    return {"result": "done"}

# GOOD: Sync function (FastAPI runs in thread pool)
@app.get("/sync-endpoint")
def sync_endpoint():
    time.sleep(2)  # OK: runs in thread pool, doesn't block event loop
    return {"result": "done"}

# BAD: Synchronous DB in async endpoint
from sqlalchemy import create_engine  # sync engine
sync_engine = create_engine("postgresql://user:pass@localhost/db")

@app.get("/bad-db")
async def bad_db_query():
    # Synchronous DB call blocks the event loop!
    with sync_engine.connect() as conn:
        result = conn.execute("SELECT * FROM users")  # BLOCKING!
    return result.fetchall()

# GOOD: Async DB operations
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy import select

async_engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")

@app.get("/good-db/{user_id}")
async def good_db_query(user_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.id == user_id))
    return result.scalar_one_or_none()

# If you must use sync library in async context, use run_in_executor
@app.get("/sync-lib-in-async")
async def use_sync_lib():
    loop = asyncio.get_event_loop()
    # Run blocking call in thread pool — doesn't block event loop
    result = await loop.run_in_executor(None, blocking_operation)
    return result

# Async HTTP client (not requests library)
@app.get("/external-api")
async def call_external():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
    return response.json()
```

**Key Takeaways:**
- async def + blocking I/O = event loop deadlock; use await with async libraries
- def endpoints run in uvicorn's thread pool — safe for sync/blocking code
- Use asyncpg/aiosqlite for async DB, httpx for async HTTP (not psycopg2/requests)
- run_in_executor bridges sync libraries to async context without blocking event loop

---

### Question 13: Microservices Inter-Service HTTP Communication

**Question:** Your Python microservice needs to call 5 other internal services to build a response. The sequential calls take 500ms total. How do you optimize this inter-service communication?

**Answer:**
Use asyncio.gather() to make concurrent HTTP calls with httpx.AsyncClient. Implement circuit breakers, timeouts, retries, and connection reuse via persistent clients.

**Detailed Explanation:**
Sequential calls: 100ms x 5 = 500ms. Concurrent calls: max(100ms each) ≈ 100ms. httpx.AsyncClient should be reused (not created per request) for connection pooling. Add retries for transient failures, circuit breakers to prevent cascade failures, and fallbacks for degraded service behavior.

**Code Example:**
```python
import asyncio
import httpx
from fastapi import FastAPI
from contextlib import asynccontextmanager
from typing import Optional

app = FastAPI()

# Module-level client for connection reuse
http_client: Optional[httpx.AsyncClient] = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global http_client
    # Startup: create persistent HTTP client
    http_client = httpx.AsyncClient(
        timeout=httpx.Timeout(connect=2.0, read=5.0, write=2.0, pool=10.0),
        limits=httpx.Limits(max_connections=100, max_keepalive_connections=20),
    )
    yield
    # Shutdown: close client
    await http_client.aclose()

app = FastAPI(lifespan=lifespan)

# BAD: Sequential calls (slow)
async def build_dashboard_sequential(user_id: int):
    user = await http_client.get(f"http://user-service/users/{user_id}")
    orders = await http_client.get(f"http://order-service/orders?user_id={user_id}")
    products = await http_client.get(f"http://product-service/featured")
    notifications = await http_client.get(f"http://notif-service/unread/{user_id}")
    analytics = await http_client.get(f"http://analytics-service/summary/{user_id}")
    # Total: 100ms * 5 = 500ms
    return {...}

# GOOD: Concurrent calls
async def build_dashboard_concurrent(user_id: int):
    async def safe_get(url: str, fallback=None):
        try:
            response = await http_client.get(url)
            response.raise_for_status()
            return response.json()
        except (httpx.HTTPError, httpx.TimeoutException) as e:
            print(f"Service call failed: {url} - {e}")
            return fallback  # Graceful degradation
    
    # All calls start simultaneously
    user, orders, products, notifications, analytics = await asyncio.gather(
        safe_get(f"http://user-service/users/{user_id}"),
        safe_get(f"http://order-service/orders?user_id={user_id}", fallback=[]),
        safe_get("http://product-service/featured", fallback=[]),
        safe_get(f"http://notif-service/unread/{user_id}", fallback={"count": 0}),
        safe_get(f"http://analytics-service/summary/{user_id}", fallback={}),
    )
    # Total: max(100ms each) ≈ 100ms
    
    return {
        "user": user,
        "orders": orders,
        "featured_products": products,
        "notifications": notifications,
        "analytics": analytics,
    }

# Circuit breaker pattern
class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30):
        self.failures = 0
        self.threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.last_failure_time = None
        self.state = "closed"  # closed, open, half-open
    
    async def call(self, coro):
        if self.state == "open":
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = "half-open"
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = await coro
            if self.state == "half-open":
                self.state = "closed"
                self.failures = 0
            return result
        except Exception:
            self.failures += 1
            self.last_failure_time = time.time()
            if self.failures >= self.threshold:
                self.state = "open"
            raise
```

**Key Takeaways:**
- asyncio.gather() runs all coroutines concurrently — n*latency becomes max(latency)
- Reuse httpx.AsyncClient across requests for HTTP/2 multiplexing and connection pooling
- Return fallback values from failed services for graceful degradation
- Circuit breakers prevent cascade failures when a downstream service is down

---

### Question 14: API Versioning Strategies

**Question:** Your API has been live for 2 years with clients depending on it. You need to make breaking changes (rename fields, change data types). How do you implement API versioning without breaking existing clients?

**Answer:**
Use URL path versioning (/v1/, /v2/) for clarity. Maintain both versions with shared business logic but separate request/response models. Use deprecation headers to communicate sunset timelines.

**Detailed Explanation:**
Versioning strategies: URL path (/v1/users), query param (?version=1), header (API-Version: 1), content negotiation (Accept: application/vnd.myapi.v1+json). URL path is most explicit and cache-friendly. Never remove versions without adequate deprecation notice (6-12 months minimum). Keep business logic shared, only vary the serialization layer.

**Code Example:**
```python
from fastapi import FastAPI, APIRouter, Request, Response
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

# V1 response model (legacy)
class UserResponseV1(BaseModel):
    id: int
    full_name: str          # V1 uses full_name
    email: str
    created: str            # V1 uses string timestamp

# V2 response model (new)
class UserResponseV2(BaseModel):
    id: int
    first_name: str         # V2 splits into first/last
    last_name: str
    email: str
    created_at: str         # V2 uses ISO format
    avatar_url: Optional[str] = None  # V2 adds new fields

# Shared business logic
async def get_user_from_db(user_id: int) -> dict:
    """Returns canonical internal representation."""
    # ... fetch from DB
    return {
        "id": user_id,
        "first_name": "John",
        "last_name": "Doe",
        "email": "john@example.com",
        "created_at": "2024-01-15T10:30:00Z",
        "avatar_url": "https://cdn.example.com/avatars/1.jpg",
    }

# Version-specific serialization
def serialize_user_v1(user: dict) -> UserResponseV1:
    return UserResponseV1(
        id=user["id"],
        full_name=f"{user['first_name']} {user['last_name']}",
        email=user["email"],
        created=user["created_at"][:10],  # Date only for V1
    )

def serialize_user_v2(user: dict) -> UserResponseV2:
    return UserResponseV2(**user)

# Separate routers per version
v1_router = APIRouter(prefix="/v1", tags=["V1 (Deprecated)"])
v2_router = APIRouter(prefix="/v2", tags=["V2 (Current)"])

@v1_router.get("/users/{user_id}", deprecated=True)
async def get_user_v1(user_id: int, response: Response):
    response.headers["Deprecation"] = "true"
    response.headers["Sunset"] = "Sat, 01 Jan 2026 00:00:00 GMT"
    response.headers["Link"] = '</v2/users/{user_id}>; rel="successor-version"'
    
    user = await get_user_from_db(user_id)
    return serialize_user_v1(user)

@v2_router.get("/users/{user_id}")
async def get_user_v2(user_id: int):
    user = await get_user_from_db(user_id)
    return serialize_user_v2(user)

app.include_router(v1_router)
app.include_router(v2_router)

# Header-based versioning alternative
@app.get("/users/{user_id}")
async def get_user_versioned(user_id: int, request: Request):
    version = request.headers.get("API-Version", "2")
    user = await get_user_from_db(user_id)
    
    if version == "1":
        return serialize_user_v1(user)
    return serialize_user_v2(user)
```

**Key Takeaways:**
- URL path versioning (/v1/) is explicit, cache-friendly, and easiest to route in load balancers
- Share business logic across versions — only vary serialization models
- Use Deprecation + Sunset headers (RFC 8594) to formally announce version retirement
- Keep v1 running for at least 6 months after v2 launch before deprecating

---

### Question 15: Handling API Timeouts and Retries

**Question:** Your API calls an external payment service that sometimes takes 10+ seconds or times out. How do you handle timeouts gracefully, implement retries with backoff, and prevent cascading failures?

**Answer:**
Set explicit timeouts on all external calls, use exponential backoff with jitter for retries, implement circuit breakers, and return graceful degraded responses when the service is unavailable.

**Detailed Explanation:**
Retrying without backoff can overwhelm a struggling service (retry storm). Exponential backoff with jitter spreads retry attempts. The circuit breaker prevents retrying a known-down service. Idempotency keys are essential for payment retries to prevent double charges.

**Code Example:**
```python
import asyncio
import httpx
import random
import time
from typing import Optional, Callable, Any
from functools import wraps

# Retry decorator with exponential backoff and jitter
def retry_with_backoff(
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    exponential_base: float = 2.0,
    jitter: bool = True,
    retryable_exceptions: tuple = (httpx.TimeoutException, httpx.ConnectError),
):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            last_exception = None
            
            for attempt in range(max_retries + 1):
                try:
                    return await func(*args, **kwargs)
                except retryable_exceptions as e:
                    last_exception = e
                    if attempt == max_retries:
                        break
                    
                    delay = min(base_delay * (exponential_base ** attempt), max_delay)
                    if jitter:
                        delay *= (0.5 + random.random())  # ±50% jitter
                    
                    print(f"Attempt {attempt+1} failed: {e}. Retrying in {delay:.2f}s")
                    await asyncio.sleep(delay)
            
            raise last_exception
        return wrapper
    return decorator

# Payment service client with full resilience
class PaymentServiceClient:
    def __init__(self):
        self.client = httpx.AsyncClient(
            base_url="https://payments.example.com",
            timeout=httpx.Timeout(connect=3.0, read=8.0, write=3.0),
        )
        self._failure_count = 0
        self._last_failure_time: Optional[float] = None
        self._circuit_open = False
        self.FAILURE_THRESHOLD = 5
        self.RECOVERY_TIMEOUT = 30.0
    
    def _check_circuit(self):
        if self._circuit_open:
            if time.time() - self._last_failure_time > self.RECOVERY_TIMEOUT:
                self._circuit_open = False
                self._failure_count = 0
            else:
                raise Exception("Circuit breaker OPEN: payment service unavailable")
    
    @retry_with_backoff(max_retries=3, base_delay=1.0, jitter=True)
    async def charge(self, amount: float, card_token: str, idempotency_key: str):
        self._check_circuit()
        try:
            response = await self.client.post(
                "/charge",
                json={"amount": amount, "token": card_token},
                headers={"Idempotency-Key": idempotency_key},  # Prevents double charge
            )
            response.raise_for_status()
            self._failure_count = 0  # Reset on success
            return response.json()
        except (httpx.TimeoutException, httpx.HTTPStatusError) as e:
            self._failure_count += 1
            self._last_failure_time = time.time()
            if self._failure_count >= self.FAILURE_THRESHOLD:
                self._circuit_open = True
            raise

payment_client = PaymentServiceClient()

from fastapi import FastAPI, HTTPException
app = FastAPI()

@app.post("/payments")
async def process_payment(amount: float, card_token: str):
    import uuid
    idempotency_key = str(uuid.uuid4())
    
    try:
        result = await payment_client.charge(amount, card_token, idempotency_key)
        return {"status": "success", "transaction_id": result["id"]}
    except Exception as e:
        raise HTTPException(503, f"Payment service unavailable: {str(e)}")
```

**Key Takeaways:**
- Exponential backoff with jitter prevents retry storms from overwhelming a recovering service
- Always use idempotency keys for payment/mutation retries to prevent duplicate operations
- Circuit breakers fail fast (no wait) when a service is known-down, improving user experience
- Set separate connect and read timeouts — a fast connection to a slow response is different

---

### Question 16: Content Delivery Network (CDN) Integration

**Question:** How do you architect your FastAPI API to work effectively with a CDN like CloudFront or Cloudflare? What can be CDN-cached vs what must hit your origin server?

**Answer:**
Cache static and semi-static API responses at CDN edges via appropriate Cache-Control headers. Use cache keys with query params. Purge CDN cache on data updates. Route dynamic/authenticated requests to origin, static public data to CDN.

**Detailed Explanation:**
CDN caches by URL + cache key. Public GET endpoints with Cache-Control: public can be cached globally. POST/PUT/DELETE and authenticated requests must reach origin. CDN edge caching reduces latency from ~100ms (cross-region) to ~5ms (nearby edge node) and protects origin from traffic spikes.

**Code Example:**
```python
from fastapi import FastAPI, Request, Response
from functools import lru_cache

app = FastAPI()

# CDN-cacheable endpoint: public product catalog
@app.get("/products/{product_id}")
async def get_product(product_id: int, response: Response):
    product = await fetch_product(product_id)
    
    # Allow CDN to cache this for 5 minutes
    response.headers["Cache-Control"] = "public, max-age=300, s-maxage=300"
    response.headers["CDN-Cache-Control"] = "max-age=3600"  # CloudFront: 1 hour
    response.headers["Surrogate-Key"] = f"product-{product_id}"  # For targeted purge
    return product

# CDN-cacheable with vary for locale
@app.get("/products")
async def list_products(response: Response, lang: str = "en"):
    products = await fetch_products_by_lang(lang)
    
    response.headers["Cache-Control"] = "public, max-age=120"
    response.headers["Vary"] = "Accept-Language, Accept-Encoding"
    return products

# NOT CDN-cacheable: user-specific data
@app.get("/user/cart")
async def get_cart(response: Response, user=Depends(get_current_user)):
    response.headers["Cache-Control"] = "private, no-store"
    return await fetch_user_cart(user.id)

# NOT CDN-cacheable: mutations
@app.post("/products/{product_id}/purchase")
async def purchase(product_id: int):
    # Bypass CDN entirely — always hits origin
    return await process_purchase(product_id)

# CDN cache invalidation after product update
async def update_product(product_id: int, data: dict):
    await update_product_in_db(product_id, data)
    
    # Purge CDN cache via API
    await purge_cloudfront_cache(
        distribution_id="E1234567890",
        paths=[f"/products/{product_id}", "/products*"]  # Wildcard for listing
    )

import httpx

async def purge_cloudfront_cache(distribution_id: str, paths: list[str]):
    """Invalidate CloudFront cache paths."""
    # Uses boto3 in practice
    import boto3
    cf = boto3.client("cloudfront")
    cf.create_invalidation(
        DistributionId=distribution_id,
        InvalidationBatch={
            "Paths": {"Quantity": len(paths), "Items": paths},
            "CallerReference": str(time.time()),
        }
    )

# CDN bypass header for debugging
@app.middleware("http")  
async def cdn_debug_middleware(request: Request, call_next):
    response = await call_next(request)
    # Add header to trace cache hit/miss at origin
    response.headers["X-Origin-Response"] = "true"
    return response
```

**Key Takeaways:**
- s-maxage overrides max-age for shared caches (CDN) — different TTL for browser vs CDN
- Surrogate-Key headers enable surgical cache purge by content type rather than URL-by-URL
- Always set Cache-Control: private, no-store for authenticated/user-specific responses
- CDN edge caching cuts cross-region latency from 100ms+ to single-digit milliseconds

---

### Question 17: GraphQL vs REST for Different Use Cases

**Question:** A client team wants you to add a GraphQL endpoint alongside your REST API. When would you choose GraphQL over REST, and how do you implement it in a FastAPI service while avoiding N+1 query problems?

**Answer:**
Use GraphQL when clients need flexible field selection and you have many client types with different data needs. Use Strawberry or Ariadne for FastAPI integration. Solve N+1 with DataLoader (batching + caching).

**Detailed Explanation:**
GraphQL shines when: multiple client types (mobile/web/3rd party) need different field subsets, you want to eliminate over-fetching, or schema evolution is needed without versioning. N+1 is the classic GraphQL pitfall — each list item resolver triggers a separate DB query. DataLoader batches these into a single query per batch cycle.

**Code Example:**
```python
import strawberry
from strawberry.fastapi import GraphQLRouter
from strawberry.dataloader import DataLoader
from typing import List, Optional
from fastapi import FastAPI

# Models
@strawberry.type
class Product:
    id: int
    name: str
    price: float
    category_id: int
    category: Optional["Category"] = None

@strawberry.type
class Category:
    id: int
    name: str
    products: List[Product] = strawberry.field(default_factory=list)

# DataLoader to solve N+1 problem
async def load_categories(category_ids: List[int]) -> List[Category]:
    """Batch load categories — one DB query for many IDs."""
    categories = await db.execute(
        select(CategoryModel).where(CategoryModel.id.in_(category_ids))
    )
    cat_map = {c.id: Category(id=c.id, name=c.name) for c in categories}
    return [cat_map.get(cid) for cid in category_ids]

# Context with DataLoaders (created per request)
class Context:
    def __init__(self):
        self.category_loader = DataLoader(load_fn=load_categories)

async def get_context():
    return Context()

# Resolvers
@strawberry.type
class Query:
    @strawberry.field
    async def products(self, info) -> List[Product]:
        raw_products = await db.execute(select(ProductModel))
        products = []
        for p in raw_products:
            product = Product(id=p.id, name=p.name, price=p.price, category_id=p.category_id)
            # DataLoader batches these N calls into 1 DB query
            product.category = await info.context.category_loader.load(p.category_id)
            products.append(product)
        return products
    
    @strawberry.field
    async def product(self, id: int) -> Optional[Product]:
        p = await db.get(ProductModel, id)
        if not p:
            return None
        return Product(id=p.id, name=p.name, price=p.price, category_id=p.category_id)

@strawberry.type
class Mutation:
    @strawberry.mutation
    async def create_product(self, name: str, price: float, category_id: int) -> Product:
        product = ProductModel(name=name, price=price, category_id=category_id)
        db.add(product)
        await db.commit()
        return Product(id=product.id, name=product.name, price=product.price, category_id=category_id)

schema = strawberry.Schema(query=Query, mutation=Mutation)

# Mount GraphQL alongside REST in FastAPI
graphql_app = GraphQLRouter(schema, context_getter=get_context)
app = FastAPI()
app.include_router(graphql_app, prefix="/graphql")

# Example GraphQL query
EXAMPLE_QUERY = """
query GetProducts {
  products {
    id
    name
    price
    category {
      name
    }
  }
}
"""
```

**Key Takeaways:**
- DataLoader is mandatory for GraphQL — without it, fetching N products triggers N+1 DB queries
- GraphQL is ideal for multiple client types that need different data shapes from the same endpoint
- DataLoader batches all loads within a single async tick into one SQL IN query
- REST is better for simple CRUD, cacheable responses, and teams unfamiliar with GraphQL

---

### Question 18: API Response Envelope and Error Standardization

**Question:** Your team has inconsistent API responses — some endpoints return data directly, others wrap it, errors vary by endpoint. How do you standardize this across a FastAPI application?

**Answer:**
Define a standard envelope schema with data/error/meta fields. Use custom exception handlers and a response wrapper utility. Implement RFC 7807 Problem Details for error responses.

**Detailed Explanation:**
Consistent API responses reduce client integration complexity. The envelope pattern adds metadata (pagination, version, request_id) alongside data. RFC 7807 (Problem Details) is the HTTP standard for error responses — use it instead of inventing your own error format.

**Code Example:**
```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from pydantic import BaseModel
from typing import Any, Optional, Generic, TypeVar
import uuid

T = TypeVar("T")

# Standard envelope models
class Meta(BaseModel):
    request_id: str
    version: str = "2.0"
    
class PaginationMeta(Meta):
    page: int
    page_size: int
    total: Optional[int] = None
    next_cursor: Optional[str] = None

class ApiResponse(BaseModel, Generic[T]):
    success: bool
    data: Optional[T] = None
    error: Optional[dict] = None
    meta: Optional[dict] = None

# RFC 7807 Problem Details
class ProblemDetail(BaseModel):
    type: str           # URI identifying the problem type
    title: str          # Human-readable summary
    status: int         # HTTP status code
    detail: str         # Human-readable explanation
    instance: str       # URI of the specific occurrence
    # Extension fields
    errors: Optional[list] = None   # Validation errors
    code: Optional[str] = None      # Application error code

app = FastAPI()

# Add request ID to all requests
@app.middleware("http")
async def add_request_id(request: Request, call_next):
    request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
    request.state.request_id = request_id
    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    return response

# Custom HTTP exception handler (RFC 7807)
@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    request_id = getattr(request.state, "request_id", str(uuid.uuid4()))
    return JSONResponse(
        status_code=exc.status_code,
        content=ProblemDetail(
            type=f"https://api.example.com/errors/{exc.status_code}",
            title=exc.detail,
            status=exc.status_code,
            detail=str(exc.detail),
            instance=str(request.url),
        ).model_dump(),
        headers={"Content-Type": "application/problem+json"},
    )

# Validation error handler
@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    errors = [{"field": ".".join(str(x) for x in e["loc"]), "msg": e["msg"]} 
              for e in exc.errors()]
    return JSONResponse(
        status_code=422,
        content=ProblemDetail(
            type="https://api.example.com/errors/validation",
            title="Validation Error",
            status=422,
            detail="Request validation failed",
            instance=str(request.url),
            errors=errors,
        ).model_dump(),
        headers={"Content-Type": "application/problem+json"},
    )

# Utility function for consistent success responses
def success_response(data: Any, meta: dict = None, status_code: int = 200):
    return JSONResponse(
        status_code=status_code,
        content={"success": True, "data": data, "meta": meta}
    )

@app.get("/users/{user_id}")
async def get_user(user_id: int, request: Request):
    user = await fetch_user(user_id)
    if not user:
        raise HTTPException(404, "User not found")
    return success_response(
        data=user,
        meta={"request_id": request.state.request_id}
    )
```

**Key Takeaways:**
- RFC 7807 Problem Details uses Content-Type: application/problem+json — the HTTP standard for errors
- X-Request-ID enables distributed tracing — correlate logs, errors, and client reports
- Generic ApiResponse[T] provides type-safe envelope with IDE autocomplete
- Handle RequestValidationError separately with field-level error details for 422 responses

---

### Question 19: Optimizing JSON Serialization Performance

**Question:** Profiling shows your API spends 40% of response time on JSON serialization of large Pydantic models. How do you speed up serialization in FastAPI?

**Answer:**
Use orjson instead of the standard json library (3-10x faster), use model.model_dump() selectively, avoid extra Pydantic validation on responses, and use response_model_exclude_unset=True.

**Detailed Explanation:**
Python's json module is implemented in Python (slow). orjson is implemented in Rust and handles datetime, UUID, numpy arrays natively without custom encoders. Pydantic v2's model.model_dump() + orjson.dumps() is the fastest combination for large payloads.

**Code Example:**
```python
import orjson
from fastapi import FastAPI
from fastapi.responses import ORJSONResponse, Response
from pydantic import BaseModel
from typing import Any

# Use ORJSON as the default response class (3-10x faster than default json)
app = FastAPI(default_response_class=ORJSONResponse)

class ProductResponse(BaseModel):
    id: int
    name: str
    price: float
    description: str
    tags: list[str]

# Method 1: ORJSONResponse for fast serialization
@app.get("/products/{id}", response_class=ORJSONResponse)
async def get_product(id: int):
    product = await fetch_product(id)
    return product  # ORJSONResponse serializes automatically

# Method 2: Manual orjson for maximum control
@app.get("/products/bulk")
async def get_products_bulk():
    products = await fetch_all_products()
    
    # model_dump() then orjson — fastest for Pydantic models
    data = [p.model_dump() for p in products]
    return Response(
        content=orjson.dumps(data, option=orjson.OPT_SERIALIZE_NUMPY),
        media_type="application/json"
    )

# Method 3: Streaming large responses
from fastapi.responses import StreamingResponse
import io

async def generate_large_json(items):
    """Stream large JSON arrays without loading all into memory."""
    yield b'{"items": ['
    first = True
    async for item in items:
        if not first:
            yield b","
        yield orjson.dumps(item.model_dump())
        first = False
    yield b"]}"

@app.get("/products/stream")
async def stream_products():
    items = await fetch_products_generator()
    return StreamingResponse(
        generate_large_json(items),
        media_type="application/json"
    )

# Exclude unset fields (reduces response size)
@app.get("/products/{id}/sparse", response_model=ProductResponse, 
         response_model_exclude_unset=True)
async def get_product_sparse(id: int):
    return await fetch_product(id)

# Benchmark comparison
import timeit
import json

def benchmark():
    data = [{"id": i, "name": f"Product {i}", "price": 9.99, "tags": ["a", "b"]} 
            for i in range(10000)]
    
    std_time = timeit.timeit(lambda: json.dumps(data), number=100)
    orjson_time = timeit.timeit(lambda: orjson.dumps(data), number=100)
    
    print(f"json.dumps:   {std_time:.3f}s")
    print(f"orjson.dumps: {orjson_time:.3f}s ({std_time/orjson_time:.1f}x faster)")
```

**Key Takeaways:**
- orjson is 3-10x faster than Python's json module and handles datetime/UUID natively
- Set ORJSONResponse as default_response_class to apply globally to all endpoints
- response_model_exclude_unset=True reduces payload size by omitting optional None fields
- Streaming JSON responses prevent memory spikes for large datasets

---

### Question 20: API Gateway Pattern Implementation

**Question:** Your company has 10 microservices. Clients are hitting each service directly. How do you implement an API gateway using Python/FastAPI as a BFF (Backend for Frontend)?

**Answer:**
Implement a FastAPI BFF that aggregates multiple microservice calls, handles auth centrally, transforms responses for each client type (mobile/web), and adds cross-cutting concerns.

**Detailed Explanation:**
The API Gateway/BFF pattern provides a single entry point that: authenticates requests, routes to appropriate services, aggregates multiple service responses into one, and tailors responses for specific client types. The BFF pattern creates separate gateways per client type (mobile BFF, web BFF) for optimized responses.

**Code Example:**
```python
import asyncio
import httpx
from fastapi import FastAPI, Request, HTTPException, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt

app = FastAPI(title="API Gateway / BFF")

# Service registry
SERVICES = {
    "users": "http://user-service:8001",
    "products": "http://product-service:8002",
    "orders": "http://order-service:8003",
    "inventory": "http://inventory-service:8004",
    "payments": "http://payment-service:8005",
}

# Centralized HTTP client with service-specific configs
class ServiceClient:
    def __init__(self):
        self.client = httpx.AsyncClient(
            timeout=httpx.Timeout(5.0),
            limits=httpx.Limits(max_connections=200),
        )
    
    async def call(self, service: str, method: str, path: str, 
                   headers: dict = None, **kwargs):
        base_url = SERVICES[service]
        url = f"{base_url}{path}"
        
        try:
            response = await self.client.request(method, url, headers=headers, **kwargs)
            response.raise_for_status()
            return response.json()
        except httpx.TimeoutException:
            raise HTTPException(504, f"{service} service timeout")
        except httpx.HTTPStatusError as e:
            raise HTTPException(e.response.status_code, f"{service} service error")

service_client = ServiceClient()

# Centralized JWT verification
security = HTTPBearer()

async def verify_token(credentials: HTTPAuthorizationCredentials = Depends(security)):
    try:
        payload = jwt.decode(credentials.credentials, "SECRET", algorithms=["HS256"])
        return payload
    except jwt.JWTError:
        raise HTTPException(401, "Invalid token")

# Mobile BFF: Aggregated dashboard endpoint
@app.get("/mobile/dashboard")
async def mobile_dashboard(user_data = Depends(verify_token)):
    user_id = user_data["sub"]
    
    # Parallel service calls
    user, orders, recommendations = await asyncio.gather(
        service_client.call("users", "GET", f"/users/{user_id}"),
        service_client.call("orders", "GET", f"/orders?user_id={user_id}&limit=5"),
        service_client.call("products", "GET", f"/recommendations?user_id={user_id}&limit=10"),
        return_exceptions=True  # Don't fail if one service is down
    )
    
    # Handle partial failures gracefully
    return {
        "user": user if not isinstance(user, Exception) else None,
        "recent_orders": orders if not isinstance(orders, Exception) else [],
        "recommendations": recommendations if not isinstance(recommendations, Exception) else [],
    }

# Request forwarding with auth header injection
@app.api_route("/{service}/{path:path}", methods=["GET", "POST", "PUT", "DELETE"])
async def proxy_request(service: str, path: str, request: Request,
                        user_data = Depends(verify_token)):
    if service not in SERVICES:
        raise HTTPException(404, f"Service '{service}' not found")
    
    # Inject user identity as internal header
    headers = {
        "X-User-ID": str(user_data["sub"]),
        "X-User-Role": user_data.get("role", "user"),
        "Content-Type": request.headers.get("content-type", "application/json"),
    }
    
    body = await request.body()
    return await service_client.call(
        service, request.method, f"/{path}",
        headers=headers, content=body
    )
```

**Key Takeaways:**
- BFF centralizes auth, rate limiting, and logging — microservices focus on business logic only
- Use return_exceptions=True in asyncio.gather() to handle partial service failures gracefully
- Inject user context as internal headers (X-User-ID) so downstream services don't need auth
- BFF pattern per client type (mobile/web) allows independent optimization

---

