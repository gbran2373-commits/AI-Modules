# AI Media Factory — Enterprise Infrastructure

Production-grade, self-hosted AI platform running n8n, PostgreSQL 17, Redis 8, MinIO, Qdrant, Ollama, and Nginx reverse proxy. Docker Compose based. Portable across GCP, AWS, Azure, Oracle Cloud, DigitalOcean, Hetzner, Contabo, and local Docker environments without architectural changes.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture Diagram](#architecture-diagram)
- [Services](#services)
- [Networks](#networks)
- [Volumes](#volumes)
- [Ports](#ports)
- [Installation](#installation)
- [Configuration](#configuration)
- [Environment Variables](#environment-variables)
- [Profiles](#profiles)
- [CPU Mode](#cpu-mode)
- [GPU Mode](#gpu-mode)
- [Updating](#updating)
- [Backup](#backup)
- [Restore](#restore)
- [Migration](#migration)
- [Scaling](#scaling)
- [Monitoring](#monitoring)
- [Logs](#logs)
- [Troubleshooting](#troubleshooting)
- [Security](#security)
- [Performance Tuning](#performance-tuning)
- [Best Practices](#best-practices)
- [Upgrade Guide](#upgrade-guide)
- [Disaster Recovery](#disaster-recovery)
- [Useful Docker Commands](#useful-docker-commands)
- [Useful Compose Commands](#useful-compose-commands)
- [Future Expansion](#future-expansion)
- [Versioning](#versioning)

---

## Project Overview

AI Media Factory is a comprehensive infrastructure platform designed to run AI-powered workflows at enterprise scale. It provides:

- **n8n** — Workflow automation engine for orchestrating AI pipelines, data transformations, and system integrations.
- **PostgreSQL 17** — Primary relational database for n8n workflow state, credentials, and execution records.
- **Redis 8** — Cache layer and Bull queue broker for n8n job distribution and ephemeral state management.
- **MinIO** — S3-compatible object storage for media files, model artifacts, and backup data.
- **Qdrant** — Vector database for embedding storage and similarity search in AI pipelines.
- **Ollama** — Local LLM inference runtime supporting CUDA and Vulkan GPU backends with Flash Attention.
- **Nginx** — High-performance reverse proxy with TLS termination, rate limiting, CORS, and security headers.
- **Prometheus** (optional) — Time-series metrics collection for infrastructure monitoring.
- **Grafana** (optional) — Visualization dashboard for infrastructure observability.

All services are containerized with Docker Compose and run on a single Docker host. The infrastructure is designed for horizontal resource scaling on any cloud or bare-metal provider.

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                         CLIENT / BROWSER                             │
│                          HTTPS :443                                  │
└──────────────────────────┬───────────────────────────────────────────┘
                           │
                     ┌─────▼─────┐
                     │  Nginx    │  Reverse Proxy (TLS termination,
                     │ :80/:443  │  rate limiting, CORS, security
                     │ (frontend │  headers, HSTS, OCSP stapling)
                     │  + backend│  health check endpoint)
                     └─────┬─────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
    ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────────┐
    │  n8n      │   │  Ollama   │   │  Qdrant        │
    │ :5678     │   │ :11434    │   │ :6333 / :6334  │
    │ (backend) │   │ (AI GPU)  │   │ (AI vectors)   │
    └─────┬─────┘   └─────┬─────┘   └──────┬─────────┘
          │                │                 │
    ┌─────▼────────────────▼─────────────────▼──────────┐
    │                  Backend Network                    │
    └────────────────────┬──────────────────────────────┘
                         │
    ┌────────────────────┼───────────────────────────────┐
    │  ┌───────────┐  ┌──▼──────────┐  ┌──────────────┐ │
    │  │PostgreSQL  │  │  Redis 8    │  │  MinIO       │ │
    │  │ :5432     │  │ :6379       │  │ :9000/:9001  │ │
    │  │ (database)│  │ (cache/     │  │ (storage/    │ │
    │  └───────────┘  │  queue)     │  │  console)    │ │
    │                  └─────────────┘  └──────────────┘ │
    │                                                     │
    │         ┌──────────────────────────────┐           │
    │         │      Database Network        │           │
    │         └──────────────────────────────┘           │
    │                  ┌──────────────────┐               │
    │                  │    Storage       │               │
    │                  │    Network       │               │
    │                  └──────────────────┘               │
    └─────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────┐
    │              Management Network (Optional)               │
    │  ┌──────────────┐  ┌──────────────┐                    │
    │  │ Prometheus   │  │  Grafana     │                    │
    │  │ :9090        │  │ :3000        │                    │
    │  └──────────────┘  └──────────────┘                    │
    │  (Activated with --profile monitoring)                   │
    └─────────────────────────────────────────────────────────┘
```

**Network Isolation:**
- `frontend` — Inbound traffic from clients through Nginx.
- `backend` — Internal communication between application services (n8n, Ollama, Qdrant, Nginx).
- `database` — Isolated network for PostgreSQL and Redis only. No application traffic.
- `storage` — Isolated network for MinIO only.
- `ai` — Purpose-built network for AI services (Qdrant, Ollama) with high-throughput requirements.
- `management` — Optional network for Prometheus and Grafana (profile-gated).

---

## Services

### Nginx Reverse Proxy

| Attribute | Value |
|-----------|-------|
| Image | `nginxinc/nginx-unprivileged:1.26-alpine` |
| Container | `ai-media-factory-nginx` |
| Ports | `80`, `443` |
| Mode | Reverse proxy with TLS termination |
| Health Check | `GET /health` on internal port 8080 |
| CPU Limit | 2.0 cores |
| Memory Limit | 512 MB |
| Security | `read_only`, `cap_drop ALL`, `no-new-privileges` |
| Features | HTTP/2, OCSP stapling, HSTS, CORS, rate limiting, Brotli (optional), TLS session resumption |

### n8n — Workflow Automation Engine

| Attribute | Value |
|-----------|-------|
| Image | `n8nio/n8n:latest` |
| Container | `ai-media-factory-n8n` |
| Port | `5678` (internal) |
| Mode | Web application with Bull queue backed by Redis |
| Health Check | `GET /healthz` on port 5678 |
| CPU Limit | 4.0 cores |
| Memory Limit | 2 GB |
| Features | Basic auth, encrypted credentials, webhooks, execution timeout (3600s), max function calls (1000), runners enabled, external function modules allowed |
| Storage | `n8n_data` (workflows, credentials), `n8n_logs` |

### PostgreSQL 17

| Attribute | Value |
|-----------|-------|
| Image | `postgres:17-bookworm` |
| Container | `ai-media-factory-postgres` |
| Port | `5432` (internal only) |
| Mode | Primary standalone database |
| Health Check | `pg_isready` on connection |
| CPU Limit | 4.0 cores |
| Memory Limit | 4 GB |
| Features | WAL at replica level, autovacuum enabled, max 200 connections, 256MB shared buffers, structured JSON logging |
| Storage | `pg_data` (data directory), `pg_config`, `pg_init` |

### Redis 8

| Attribute | Value |
|-----------|-------|
| Image | `redis:8-bookworm` |
| Container | `ai-media-factory-redis` |
| Port | `6379` (internal only) |
| Mode | Cache + persistent message broker |
| Health Check | `redis-cli ping` with authentication |
| CPU Limit | 2.0 cores |
| Memory Limit | 2 GB |
| Features | AOF persistence (`everysec`), RDB snapshots (900/1/300/10/60/10000), maxmemory 2GB with `allkeys-lru` eviction, `overcommit_memory=1` |
| Storage | `redis_data`, `redis_config`, `redis_logs` |

### MinIO — S3-Compatible Object Storage

| Attribute | Value |
|-----------|-------|
| Image | `minio/minio:latest` |
| Container | `ai-media-factory-minio` |
| Ports | `9000` (API), `9001` (Console) |
| Health Check | `GET /minio/health/live` on port 9000 |
| CPU Limit | 4.0 cores |
| Memory Limit | 4 GB |
| Features | Browser console enabled, cache drives (10GB), compression, audit logging in JSON, lifecycle expiry (365 days), S3 API compliance |
| Storage | `minio_data`, `minio_config`, `minio_logs` |

### Qdrant — Vector Database

| Attribute | Value |
|-----------|-------|
| Image | `qdrant/qdrant:latest` |
| Container | `ai-media-factory-qdrant` |
| Ports | `6333` (HTTP), `6334` (gRPC) |
| Health Check | `GET /healthz` on port 6333 |
| CPU Limit | 4.0 cores |
| Memory Limit | 8 GB |
| Features | jemalloc allocator, OpenVisor enabled (non-telemetry), CORS enabled, JSON structured logging, WAL and snapshot paths configured, max 1000 collections, memlock unlimited |
| Storage | `qdrant_data`, `qdrant_snapshots`, `qdrant_config`, `qdrant_logs` |

### Ollama — Local LLM Runtime

| Attribute | Value |
|-----------|-------|
| Image | `ollama/ollama:latest` |
| Container | `ai-media-factory-ollama` |
| Port | `11434` (internal + published for host access) |
| Health Check | `GET /api/tags` on port 11434 |
| CPU Limit | 8.0 cores |
| Memory Limit | 32 GB |
| GPU | NVIDIA (required). Vulkan fallback enabled for AMD/Intel. Flash Attention enabled. CUDA graphs enabled. |
| Features | 5-minute model keep-alive, 1 parallel inference, 2 max loaded models, Vulkan backend, API key authentication |
| Security | `privileged: true` (required for GPU memory locking). `no-new-privileges` disabled for GPU operations. |
| Storage | `ollama_models`, `ollama_data`, `ollama_cache`, `ollama_logs` |

---

## Networks

| Network | Driver | Subnet | Gateway | Purpose |
|---------|--------|--------|---------|---------|
| `ai-media-factory-frontend` | bridge | 172.20.0.0/16 | 172.20.0.1 | Inbound client traffic via Nginx |
| `ai-media-factory-backend` | bridge | 172.21.0.0/16 | 172.21.0.1 | Application-to-application communication |
| `ai-media-factory-database` | bridge | 172.22.0.0/16 | 172.22.0.1 | PostgreSQL and Redis isolation |
| `ai-media-factory-storage` | bridge | 172.23.0.0/16 | 172.23.0.1 | MinIO client access isolation |
| `ai-media-factory-ai` | bridge | 172.24.0.0/16 | 172.24.0.1 | AI services (Qdrant, Ollama) high-throughput |
| `ai-media-factory-management` | bridge | 172.25.0.0/16 | 172.25.0.1 | Monitoring stack (Prometheus, Grafana) |

All networks have ICC enabled and IP masquerading enabled for inter-container and outbound communication.

---

## Volumes

All volumes use bind mounts pointing to `AI_MEDIA_FACTORY_VOLUME_BASE` (default: `/opt/ai-media-factory`). The `driver: local` with `type: none` and `o: bind` options ensure direct host filesystem access.

| Volume Name | Mount Path | Host Path | Purpose | Size |
|-------------|-----------|-----------|---------|------|
| `ai-media-factory-n8n_data` | `/home/node/.n8n` | `/opt/ai-media-factory/n8n/data` | n8n workflows, credentials | Unlimited (DB growth) |
| `ai-media-factory-n8n_logs` | `/home/node/.n8n/logs` | `/opt/ai-media-factory/n8n/logs` | n8n execution logs | Unlimited |
| `ai-media-factory-pg_data` | `/var/lib/postgresql/data` | `/opt/ai-media-factory/postgres/data` | PostgreSQL data files | Unlimited (DB growth) |
| `ai-media-factory-pg_config` | `/etc/postgresql` | — | PostgreSQL config (read-only) | ~10 MB |
| `ai-media-factory-pg_init` | `/docker-entrypoint-initdb.d` | — | Init scripts (read-only) | Configurable |
| `ai-media-factory-redis_data` | `/data` | `/opt/ai-media-factory/redis/data` | Redis AOF + RDB files | Unlimited |
| `ai-media-factory-redis_config` | `/usr/local/etc/redis` | — | Redis config (read-only) | ~10 MB |
| `ai-media-factory-redis_logs` | `/var/log/redis` | — | Redis logs | Unlimited |
| `ai-media-factory-minio_data` | `/data` | `/opt/ai-media-factory/minio/data` | Object storage data | Unlimited (capacity bound by disk) |
| `ai-media-factory-minio_config` | `/root/.minio` | — | MinIO server config | ~50 MB |
| `ai-media-factory-minio_logs` | `/var/log/minio` | — | MinIO audit + server logs | Unlimited |
| `ai-media-factory-qdrant_data` | `/qdrant/storage` | `/opt/ai-media-factory/qdrant/data` | Vector collections and indices | Unlimited |
| `ai-media-factory-qdrant_snapshots` | `/qdrant/snapshots` | `/opt/ai-media-factory/qdrant/snapshots` | Point-in-time recovery snapshots | Unlimited |
| `ai-media-factory-qdrant_config` | `/qdrant/config` | — | Qdrant config (read-only) | ~10 MB |
| `ai-media-factory-qdrant_logs` | `/var/log/qdrant` | — | Qdrant logs | Unlimited |
| `ai-media-factory-ollama_models` | `/root/.ollama/models` | `/opt/ai-media-factory/ollama/models` | Downloaded LLM models (hundreds of GB) | Unlimited (disk bound) |
| `ai-media-factory-ollama_data` | `/root/.ollama` | — | Ollama runtime state | ~1 GB |
| `ai-media-factory-ollama_cache` | `/root/.cache/ollama` | — | Ollama model cache | Unlimited |
| `ai-media-factory-ollama_logs` | `/var/log/ollama` | — | Ollama logs | Unlimited |
| `ai-media-factory-nginx_conf` | `/etc/nginx/conf.d` | — | Nginx virtual host configs (read-only) | ~1 MB |
| `ai-media-factory-nginx_certs` | `/etc/nginx/certs` | — | TLS certificates (read-only) | Small |
| `ai-media-factory-nginx_ssl` | `/etc/nginx/ssl` | — | SSL keys and DH params (read-only) | Small |
| `ai-media-factory-nginx_logs` | `/var/log/nginx` | — | Access and error logs | Unlimited |
| `ai-media-factory-nginx_static` | `/usr/share/nginx/html` | — | Static assets (read-only) | Small |
| `ai-media-factory-prometheus_config` | `/etc/prometheus` | — | Prometheus config and rules (read-only) | Small |
| `ai-media-factory-prometheus_data` | `/prometheus` | `/opt/ai-media-factory/prometheus/data` | TSDB data files | 10 GB (config retention) |
| `ai-media-factory-prometheus_rules` | `/etc/prometheus/rules` | — | Alerting rules (read-only) | Small |
| `ai-media-factory-grafana_data` | `/var/lib/grafana` | `/opt/ai-media-factory/grafana/data` | Dashboards, datasources, users | Unlimited (DB growth) |
| `ai-media-factory-grafana_config` | `/etc/grafana` | — | Grafana config (read-only) | Small |
| `ai-media-factory-grafana_plugins` | `/var/lib/grafana/plugins` | — | Grafana plugin files | ~1 GB |
| `ai-media-factory-grafana_logs` | `/var/log/grafana` | — | Grafana logs | Unlimited |

---

## Ports

These ports are published to the Docker host. Adjust `published` values or remove entries if running behind an external load balancer.

| Host Port | Container Port | Protocol | Service | Description |
|-----------|---------------|----------|---------|-------------|
| 80 | 8080 | TCP | Nginx | HTTP (redirected to HTTPS when `NGINX_REDIRECT_HTTP_TO_HTTPS=true`) |
| 443 | 443 | TCP | Nginx | HTTPS |
| 11434 | 11434 | TCP | Ollama | LLM API (host-accessible for direct inference calls) |
| 3000 | 3000 | TCP | Grafana | Monitoring dashboard (profile only) |
| 9090 | 9090 | TCP | Prometheus | Metrics endpoint (profile only) |
| 9000 | 9000 | TCP | MinIO | S3 API (internal only, not published) |
| 9001 | 9001 | TCP | MinIO | Console UI (internal only, not published) |
| 6333 | 6333 | TCP | Qdrant | HTTP API (internal only) |
| 6334 | 6334 | TCP | Qdrant | gRPC API (internal only) |
| 5678 | 5678 | TCP | n8n | Webhook receiver (internal only) |
| 5432 | 5432 | TCP | PostgreSQL | (internal only) |
| 6379 | 6379 | TCP | Redis | (internal only) |

Internal-only services are not published to the Docker host. Access must go through the Nginx reverse proxy or the internal Docker network.

---

## Installation

### Prerequisites

| Requirement | Minimum Version | Notes |
|-------------|----------------|-------|
| Docker Engine | 24.0+ | Required for Compose Spec v3.8 support |
| Docker Compose | 2.23+ | Standalone plugin, not `docker-compose` Python package |
| NVIDIA Container Toolkit | 1.16+ | Required for Ollama GPU support; optional for CPU-only |
| Vulkan SDK | 1.3+ | Required for AMD GPU Ollama support; optional for NVIDIA |
| Operating System | Linux (kernel 5.15+) | Required on all cloud providers and bare metal |
| Kernel modules | `bridge`, `overlay`, `nf_conntrack` | Loaded by Docker; verify with `lsmod` |
| Disk space | 200 GB minimum | 500 GB+ recommended for Ollama models and MinIO data |
| RAM | 16 GB minimum | 64 GB+ recommended with GPU-accelerated Ollama models |
| CPU | 8 cores minimum | 16+ cores recommended for concurrent workflow execution |
| Swap | Disabled or ≥32 GB | PostgreSQL and Ollama perform poorly with swap |

### Basic Installation

```bash
# 1. Clone the repository (if applicable)
git clone https://github.com/<org>/ai-media-factory.git
cd ai-media-factory

# 2. Copy the environment file template and edit it
cp .env.example .env
# Edit .env with your domain names, passwords, and configuration values

# 3. Create the volume base directory with secure permissions
sudo mkdir -p /opt/ai-media-factory
sudo chown -R $(id -u):$(id -g) /opt/ai-media-factory
sudo chmod 750 /opt/ai-media-factory

# 4. Validate the compose configuration
docker compose config

# 5. Start all services
docker compose up -d

# 6. Verify all services are healthy
docker compose ps
```

### Cloud Provider Installation Notes

| Provider | Additional Steps |
|----------|-----------------|
| **Google Cloud** | Attach a Persistent Disk (SSD) for `/opt/ai-media-factory`. Configure firewall rules for ports 80/443. Use a reserved internal IP for the Docker host. |
| **AWS** | Attach an EBS gp3 volume (≥500 GB) for data. Use an ECS or EC2 instance with EBS-optimized networking. Configure Security Groups for port 80/443 ingress. |
| **Azure** | Attach a Managed Disk (Premium SSD) for data. Configure NSG rules for ports 80/443. Use an accelerated networking-enabled VM. |
| **Oracle Cloud** | Attach a PV EBS volume. Use an Arm-based VM instance for cost efficiency. Configure security list rules for ports 80/443. |
| **DigitalOcean** | Attach a Block Storage volume. Create a firewall with ports 80/443 open. Use a dedicated CPU droplet. |
| **Hetzner** | Attach a Hetzner Storage Box or local NVMe. Use the Hetzner Cloud Firewall for port filtering. Use CX or CC series for GPU support. |
| **Contabo** | Attach a RAID1 storage pool using LVM for `/opt/ai-media-factory`. Use a dedicated server with NVIDIA GPU for Ollama GPU workloads. |
| **Local Docker** | No cloud-specific steps required. Docker volumes are stored on the host filesystem by default. |

---

## Configuration

### Domain Setup

Before starting the platform, ensure all service domains resolve to the Nginx reverse proxy's public IP address or load balancer:

| Domain | Service | DNS Record Type |
|--------|---------|-----------------|
| `ai-media-factory.local` | Nginx, platform root | A |
| `n8n.ai-media-factory.local` | n8n web interface and webhooks | A |
| `minio.ai-media-factory.local` | MinIO S3 API and console (if exposed) | A |
| `grafana.ai-media-factory.local` | Grafana dashboard (optional) | A |
| `prometheus.ai-media-factory.local` | Prometheus metrics endpoint (optional) | A |

### TLS Certificates

Place TLS certificate files and keys in the Nginx volume directories or use Docker secrets:

```bash
# Using Docker secrets (recommended for production)
cp server.crt /run/secrets/ai-media-factory-tls-crt
cp server.key /run/secrets/ai-media-factory-tls-key
cp ca-bundle.crt /run/secrets/ai-media-factory-tls-ca

# Using mount bind paths (alternative)
# Place certificates at the paths configured in .env:
# TLS_CERT_PATH=/etc/nginx/certs/ai-media-factory.crt
# TLS_KEY_PATH=/etc/nginx/certs/ai-media-factory.key
```

### Initial n8n Setup

1. After `docker compose up -d`, navigate to `https://n8n.ai-media-factory.local`.
2. Log in with the `N8N_USER` and `N8N_PASSWORD` credentials from `.env`.
3. Create a new workflow and test the Ollama connection at `http://ollama:11434` (internal network).
4. Create MinIO credentials and configure S3 nodes using `MINIO_ROOT_USER`/`MINIO_ROOT_PASSWORD`.
5. Create a Qdrant API key and configure the Qdrant node using the value from `.env`.

---

## Environment Variables

All environment variables are documented in the `.env` file at the repository root. Variables are grouped by category. The table below lists every variable referenced by `docker-compose.yml` with its source file and a description.

### General

| Variable | Source | Description |
|----------|--------|-------------|
| `COMPOSE_PROJECT_NAME` | `.env` | Docker Compose project name prefix for container and network names |
| `AI_MEDIA_FACTORY_VOLUME_BASE` | `.env`, `docker-compose.yml` | Root directory for persistent volume bind-mounts |
| `AI_MEDIA_FACTORY_SECRETS_DIR` | `.env`, `docker-compose.yml` | Directory containing Docker secrets files |
| `AI_MEDIA_FACTORY_CONFIGS_DIR` | `.env`, `docker-compose.yml` | Directory containing Docker config files |
| `TZ` | `.env` | Timezone for all containers |
| `AI_MEDIA_FACTORY_IMAGE_CHANNEL` | `.env` | Container image release channel |

### Domains

| Variable | Source | Description |
|----------|--------|-------------|
| `NGINX_HOST` | `.env`, `docker-compose.yml` | Primary domain for the AI Media Factory platform |
| `N8N_HOST` | `.env`, `docker-compose.yml` | Subdomain for n8n workflow engine |
| `MINIO_DOMAIN` | `.env`, `docker-compose.yml` | Subdomain for MinIO S3 console and API |
| `GRAFANA_HOST` | `.env` | Subdomain for Grafana dashboard |
| `PROMETHEUS_HOST` | `.env` | Subdomain for Prometheus metrics endpoint |
| `ALLOWED_ORIGINS` | `.env` | Comma-separated list of allowed CORS origins |

### Ports

| Variable | Source | Description |
|----------|--------|-------------|
| `NGINX_PORT_HTTP` | `.env` | HTTP port for Nginx reverse proxy |
| `NGINX_PORT_HTTPS` | `.env` | HTTPS port for Nginx reverse proxy |
| `NGINX_HEALTH_PORT` | `.env` | Internal health check port for Nginx |
| `N8N_PORT` | `.env` | Internal n8n API port |
| `POSTGRES_PORT` | `.env` | Internal PostgreSQL client port |
| `REDIS_PORT` | `.env` | Internal Redis server port |
| `MINIO_API_PORT` | `.env` | Internal MinIO API port |
| `MINIO_CONSOLE_PORT` | `.env` | Internal MinIO Console port |
| `QDRANT_HTTP_PORT` | `.env` | Internal Qdrant HTTP API port |
| `QDRANT_GRPC_PORT` | `.env` | Internal Qdrant gRPC port |
| `OLLAMA_PORT` | `.env` | Internal Ollama API port |
| `PROMETHEUS_PORT` | `.env` | Internal Prometheus port |
| `GRAFANA_PORT` | `.env` | Internal Grafana port |

### SSL / TLS

| Variable | Source | Description |
|----------|--------|-------------|
| `TLS_CERT_PATH` | `.env` | Host path to TLS certificate file |
| `TLS_KEY_PATH` | `.env` | Host path to TLS private key file |
| `TLS_CA_BUNDLE_PATH` | `.env` | Host path to CA certificate bundle |
| `TLS_MIN_VERSION` | `.env` | Minimum TLS protocol version for Nginx |
| `TLS_CIPHERS` | `.env` | TLS cipher suite ordering |
| `HSTS_ENABLED` | `.env` | Enable HTTP Strict Transport Security |
| `HSTS_MAX_AGE` | `.env` | HSTS max-age in seconds |
| `HSTS_INCLUDE_SUBDOMAINS` | `.env` | Include `includeSubDomains` HSTS directive |
| `HSTS_PRELOAD` | `.env` | Enable HSTS `preload` directive |
| `NGINX_REDIRECT_HTTP_TO_HTTPS` | `.env` | Redirect HTTP to HTTPS |
| `SSL_OCSP_STAPLING` | `.env` | Enable OCSP stapling in Nginx |
| `SSL_SESSION_TIMEOUT` | `.env` | TLS session timeout |
| `SSL_SESSION_CACHE_SIZE` | `.env` | SSL session cache size |

### PostgreSQL

| Variable | Source | Description |
|----------|--------|-------------|
| `POSTGRES_USER` | `.env` | PostgreSQL superuser account name |
| `POSTGRES_PASSWORD` | `.env` | PostgreSQL superuser password (≥32 chars) |
| `POSTGRES_DB` | `.env` | Default database name |
| `PG_MAX_CONNECTIONS` | `.env` | Maximum concurrent client connections |
| `PG_SHARED_BUFFERS` | `.env` | PostgreSQL shared buffer cache size |
| `PG_EFFECTIVE_CACHE_SIZE` | `.env` | Planner effective cache size |
| `PG_WORK_MEM` | `.env` | Memory for sort/hash operations |
| `PG_MAINTENANCE_WORK_MEM` | `.env` | Memory for maintenance operations |
| `PG_WAL_LEVEL` | `.env` | WAL level (minimal or replica) |
| `PG_MAX_WAL_SIZE` | `.env` | Maximum WAL file size |
| `PG_MIN_WAL_SIZE` | `.env` | Minimum WAL file size |
| `PG_CHECKPOINT_COMPLETION_TARGET` | `.env` | Checkpoint I/O spreading fraction |
| `PG_LOG_STATEMENT` | `.env` | SQL statement logging level |
| `PG_LOG_LINE_PREFIX` | `.env` | Log line prefix format |
| `PG_LOG_ROTATE_SIZE` | `.env` | PostgreSQL log rotation size |
| `PG_LOG_RETENTION_COUNT` | `.env` | PostgreSQL log retention count |
| `PG_AUTO_VACUUM` | `.env` | Enable autovacuum |
| `PG_AUTOVACUUM_MAX_WORKERS` | `.env` | Autovacuum worker count |
| `PG_AUTOVACUUM_NAPILIMIT` | `.env` | Autovacuum nap limit |
| `DB_POSTGRESDB_HOST` | Internal | PostgreSQL host (derived: `postgres`) |
| `DB_POSTGRESDB_PORT` | Internal | PostgreSQL port (derived: 5432) |
| `DB_POSTGRESDB_DATABASE` | Internal | n8n database name |
| `DB_POSTGRESDB_USER` | Internal | n8n database user |
| `DB_POSTGRESDB_PASSWORD` | Internal | n8n database password |
| `DB_POSTGRESDB_SSL` | Internal | PostgreSQL SSL mode |
| `DB_POSTGRESDB_POOL_SIZE` | `.env` | n8n PostgreSQL connection pool size |

### Redis

| Variable | Source | Description |
|----------|--------|-------------|
| `REDIS_PASSWORD` | `.env` | Redis authentication password |
| `REDIS_MAXMEMORY` | `.env` | Maximum Redis memory before eviction |
| `REDIS_MAXMEMORY_POLICY` | Internal | Redis eviction policy |
| `REDIS_APPENDONLY` | Internal | Enable AOF persistence |
| `REDIS_APPENDFSYNC` | Internal | AOF fsync policy |
| `REDIS_APPENDFILENAME` | Internal | AOF filename |
| `REDIS_SAVE_900_1` | Internal | RDB snapshot: 1 change in 900s |
| `REDIS_SAVE_300_10` | Internal | RDB snapshot: 10 changes in 300s |
| `REDIS_SAVE_60_10000` | Internal | RDB snapshot: 10000 changes in 60s |
| `REDIS_MAXCLIENTS` | `.env` | Maximum Redis client connections |
| `REDIS_SLOWLOG_LOGSLOWERTHAN` | `.env` | Slow log threshold (microseconds) |
| `REDIS_SLOWLOG_MAXLEN` | `.env` | Slow log maximum length |
| `REDIS_VM_OVERCOMMIT_MEMORY` | Internal | Overcommit memory for Redis fork |
| `REDIS_ACTIVEDEFRAG` | Internal | Enable active defragmentation |

### MinIO

| Variable | Source | Description |
|----------|--------|-------------|
| `MINIO_ROOT_USER` | `.env` | MinIO root access key |
| `MINIO_ROOT_PASSWORD` | `.env` | MinIO root secret key (≥32 chars) |
| `MINIO_DOMAIN` | `.env`, `docker-compose.yml` | MinIO virtual-hosted-style domain |
| `MINIO_STORAGE_CLASS` | `.env` | Default storage class for buckets |
| `MINIO_CACHE_DRIVES` | `.env` | MinIO cache drive path |
| `MINIO_CACHE_SIZE` | `.env` | MinIO cache size |
| `MINIO_CACHE_EXCLUDE` | `.env` | File patterns excluded from cache |
| `MINIO_COMPUTE_NODES` | `.env` | Number of MinIO compute nodes |
| `MINIO_UPDATE` | `.env` | MinIO auto-update policy |
| `MINIO_AUDIT_LOG_FORMAT` | `.env` | MinIO audit log format |
| `MINIO_AUDIT_LOG_PATH` | Internal | Path for MinIO audit logs |
| `MINIO_COMPRESSION` | `.env` | Enable MinIO compression |
| `MINIO_LIFECYCLE_EXPIRY_DAYS` | `.env` | Default bucket lifecycle expiry |
| `MINIO_MAX_BUCKETS` | `.env` | Maximum number of buckets |

### Qdrant

| Variable | Source | Description |
|----------|--------|-------------|
| `QDRANT_API_KEY` | `.env` | Qdrant API authentication key |
| `QDRANT_HTTP_PORT` | `.env` | Qdrant HTTP API port |
| `QDRANT_GRPC_PORT` | `.env` | Qdrant gRPC port |
| `QDRANT_ALLOW_CORS` | `.env` | Enable CORS for Qdrant API |
| `QDRANT_WAL_PATH` | Internal | Qdrant WAL storage path |
| `QDRANT_SNAPSHOT_PATH` | Internal | Qdrant snapshot storage path |
| `QDRANT_STORAGE_PATH` | Internal | Qdrant collection data path |
| `QDRANT_MAX_WAL_SIZE` | `.env` | Maximum WAL file size |
| `QDRANT_WAL_MMAP_THRESHOLD` | `.env` | WAL memory-mapping threshold |
| `QDRANT_SNAPSHOT_INTERVAL` | `.env` | Qdrant snapshot interval (seconds) |
| `QDRANT_PERFORMANCE_WARNING_THRESHOLD` | `.env` | Performance warning point threshold |
| `QDRANT_MAX_COLLECTIONS` | `.env` | Maximum number of collections |
| `QDRANT_OPENVISOR_ENABLED` | `.env` | Enable OpenVisor observability |
| `QDRANT_LOG_LEVEL` | `.env` | Qdrant log verbosity level |
| `QDRANT_LOG_JSON` | `.env` | Emit Qdrant logs in JSON format |
| `QDRANT_LOG_FILE_ENABLE` | `.env` | Write Qdrant logs to file |
| `QDRANT_LOG_FILE_PATH` | Internal | Qdrant log file path |
| `QDRANT_TELEMETRY_ENABLED` | `.env` | Enable Qdrant telemetry |
| `QDRANT_CLUSTER_ENABLED` | `.env` | Enable Qdrant horizontal clustering |
| `QDRANT_MEMORY_ALLOCATOR` | `.env` | Memory allocator (jemalloc) |
| `QDRANT_PARALLEL_THREADS` | `.env` | Qdrant parallel search threads |

### Ollama

| Variable | Source | Description |
|----------|--------|-------------|
| `OLLAMA_HOST` | Internal | Host Ollama binds to |
| `OLLAMA_ORIGINS` | `.env` | Allowed CORS origins for Ollama API |
| `OLLAMA_MODELS` | Internal | Ollama models directory |
| `OLLAMA_KEEP_ALIVE` | `.env` | Model keep-alive duration |
| `OLLAMA_NUM_PARALLEL` | `.env` | Parallel inference requests |
| `OLLAMA_MAX_LOADED_MODELS` | `.env` | Maximum loaded models in GPU memory |
| `OLLAMA_GPU_LAYERS` | `.env` | GPU layers for model offloading |
| `OLLAMA_VULKAN` | `.env` | Enable Vulkan GPU backend |
| `OLLAMA_FLASH_ATTENTION` | `.env` | Enable Flash Attention optimization |
| `OLLAMA_API_KEY` | `.env` | Ollama HTTP API bearer token |
| `OLLAMA_TIMEOUT` | `.env` | Ollama inference request timeout |
| `OLLAMA_CONTEXT_LENGTH` | `.env` | Maximum context window tokens |

### Workers

| Variable | Source | Description |
|----------|--------|-------------|
| `N8N_RUNNERS` | `.env` | Number of n8n worker processes |
| `N8N_QUEUE_BULL_BOARD_MAX_BULK_SIZE` | `.env` | Max messages per Bull queue poll |
| `N8N_QUEUE_BULL_BOARD_REDIS_BULK_DELAY` | `.env` | Bull queue poll interval (ms) |
| `N8N_WORKERS` | `.env` | n8n background worker threads |
| `N8N_MAX_WEBHOOK_QUEUE_SIZE` | `.env` | Max simultaneous webhook executions |
| `N8N_MAX_RETRIES` | `.env` | Max retries for failed n8n executions |
| `N8N_RETRY_DELAY` | `.env` | Delay between n8n retry attempts (ms) |
| `N8N_EXECUTIONS_TTL` | `.env` | TTL for completed execution records (seconds) |

### Execution

| Variable | Source | Description |
|----------|--------|-------------|
| `EXECUTIONS_TIMEOUT` | `.env` | Max execution duration (seconds) |
| `MAX_FUNCTION_CALLS` | `.env` | Max function calls per execution |
| `HTTP_MAX_RESPONSE_SIZE` | `.env` | Max HTTP response size (bytes) |
| `HTTP_MAX_REQUEST_SIZE` | `.env` | Max HTTP request size (bytes) |
| `HTTP_REQUEST_TIMEOUT` | `.env` | HTTP request timeout (seconds) |
| `N8N_MAX_CONCURRENT_EXECUTIONS` | `.env` | Global max concurrent executions |
| `N8N_QUEUE_TYPE` | `.env` | Queue backend type (redis) |
| `N8N_QUEUE_POLL_INTERVAL` | `.env` | Queue poll interval (ms) |

### Uploads

| Variable | Source | Description |
|----------|--------|-------------|
| `N8N_MAX_UPLOAD_SIZE` | `.env` | Max n8n webhook upload size (bytes) |
| `MINIO_MAX_UPLOAD_SIZE` | `.env` | Max MinIO S3 upload size (bytes) |
| `UPLOAD_TEMP_DIR` | `.env` | Temporary upload directory |
| `UPLOAD_TEMP_MAX_AGE` | `.env` | Max age for temp upload files (seconds) |
| `N8N_ALLOWED_MIME_TYPES` | `.env` | Allowed MIME types for n8n uploads |
| `MINIO_UPLOAD_MAX_DIRS` | `.env` | Max MinIO upload directories |
| `MINIO_MULTIPART_UPLOAD` | `.env` | Enable multipart upload in MinIO |
| `MINIO_MULTIPART_PART_SIZE` | `.env` | MinIO multipart part size (bytes) |

### Storage

| Variable | Source | Description |
|----------|--------|-------------|
| `AI_MEDIA_FACTORY_VOLUME_BASE` | `.env` | Host directory for all bind mounts |
| `N8N_VOLUME_SUBDIR` | `.env` | n8n data subdirectory |
| `PG_VOLUME_SUBDIR` | `.env` | PostgreSQL data subdirectory |
| `REDIS_VOLUME_SUBDIR` | `.env` | Redis data subdirectory |
| `MINIO_VOLUME_SUBDIR` | `.env` | MinIO data subdirectory |
| `QDRANT_VOLUME_SUBDIR` | `.env` | Qdrant data subdirectory |
| `OLLAMA_VOLUME_SUBDIR` | `.env` | Ollama models subdirectory |
| `NGINX_CONF_SUBDIR` | `.env` | Nginx config subdirectory |
| `NGINX_CERTS_SUBDIR` | `.env` | SSL cert subdirectory |
| `NGINX_LOG_SUBDIR` | `.env` | Nginx log subdirectory |
| `VOLUME_FILESYSTEM` | `.env` | Filesystem type for volumes |
| `VOLUME_QUOTAS_ENABLED` | `.env` | Enable filesystem quotas |
| `MINIO_CACHE_VOLUME_SIZE` | `.env` | MinIO cache volume size |
| `VOLUME_TRIM_ENABLED` | `.env` | Enable TRIM on volumes |

### Logging

| Variable | Source | Description |
|----------|--------|-------------|
| `DOCKER_LOG_DRIVER` | `.env` | Host Docker daemon log driver |
| `DOCKER_LOG_MAX_SIZE` | `.env` | Host Docker log max size |
| `DOCKER_LOG_MAX_FILES` | `.env` | Host Docker log retention count |
| `DOCKER_LOG_COMPRESS` | `.env` | Compress rotated Docker logs |
| `N8N_LOG_LEVEL` | `.env` | n8n application log level |
| `PG_LOG_LEVEL` | `.env` | PostgreSQL log level |
| `REDIS_LOG_LEVEL` | `.env` | Redis log level |
| `MINIO_LOG_LEVEL` | `.env` | MinIO log level |
| `QDRANT_LOG_LEVEL` | `.env` | Qdrant log verbosity level |
| `OLLAMA_LOG_LEVEL` | `.env` | Ollama log level |
| `NGINX_LOG_LEVEL` | `.env` | Nginx log level |

### Performance

| Variable | Source | Description |
|----------|--------|-------------|
| `NET_CORE_SOMAXCONN` | `.env`, `sysctls` | TCP socket backlog |
| `NET_IPV4_TCP_MAX_SYN_BACKLOG` | `.env`, `sysctls` | Maximum SYN queue length |
| `NET_IPV4_TCP_TW_REUSE` | `.env`, `sysctls` | Enable TIME-WAIT socket reuse |
| `FS_FILE_MAX` | `.env`, `sysctls` | System file descriptor limit |
| `VM_OVERCOMMIT_MEMORY` | `.env`, `sysctls` | Enable memory overcommit |
| `NR_OPEN` | `.env`, `ulimits` | Max open file descriptors |
| `VM_SWAPPINESS` | `.env` | Kernel swappiness (0–100) |
| `VM_DIRTY_WRITEBACK_CENTISECONDS` | `.env` | Dirty page writeback interval |
| `VM_DIRTY_RATIO` | `.env` | % memory at which dirty writeback begins |
| `VM_DIRTY_EXPIRATION_CENTISECONDS` | `.env` | % memory at which dirty pages are forced out |
| `N8N_NOFILE_LIMIT` | `.env`, `ulimits` | n8n file descriptor limit |
| `QDRANT_NUM_THREADS` | `.env` | Qdrant search threads |

### Backups

| Variable | Source | Description |
|----------|--------|-------------|
| `BACKUP_PG_ENABLED` | `.env` | Enable PostgreSQL automated backups |
| `BACKUP_PG_CRON_SCHEDULE` | `.env` | PostgreSQL backup cron schedule |
| `BACKUP_PG_RETENTION_DAYS` | `.env` | PostgreSQL backup retention period |
| `BACKUP_PG_MAX_SIZE_MB` | `.env` | Max PostgreSQL backup file size |
| `BACKUP_MINIO_ENABLED` | `.env` | Enable MinIO lifecycle policies for backups |
| `BACKUP_MINIO_CRON_SCHEDULE` | `.env` | MinIO lifecycle policy schedule |
| `BACKUP_MINIO_NONCURRENT_VERSION_EXPIRY` | `.env` | Days before non-current version expiry |
| `BACKUP_MINIO_DELETE_MARKER_EXPIRY` | `.env` | Days before delete marker expiry |
| `BACKUP_MINIO_TRANSITION_STORAGE_CLASS` | `.env` | MinIO storage class for lifecycle transitions |
| `BACKUP_QDRANT_ENABLED` | `.env` | Enable Qdrant snapshot backups |
| `BACKUP_QDRANT_CRON_SCHEDULE` | `.env` | Qdrant snapshot cron schedule |
| `BACKUP_QDRANT_RETENTION` | `.env` | Qdrant snapshot retention count |
| `BACKUP_OLLAMA_ENABLED` | `.env` | Enable Ollama model backup |
| `BACKUP_HEALTH_CHECK_INTERVAL` | `.env` | Backup integrity check interval |
| `BACKUP_ALERT_EMAIL` | `.env` | Email for backup failure notifications |
| `BACKUP_OFFSITE_ENDPOINT` | `.env` | Offsite backup endpoint (S3-compatible) |
| `BACKUP_OFFSITE_ACCESS_KEY` | `.env` | Offsite backup access key |
| `BACKUP_OFFSITE_SECRET_KEY` | `.env` | Offsite backup secret key |
| `BACKUP_OFFSITE_BUCKET` | `.env` | Offsite backup bucket name |
| `BACKUP_OFFSITE_REGION` | `.env` | Offsite backup region |

### Monitoring

| Variable | Source | Description |
|----------|--------|-------------|
| `MONITORING_ENABLED` | `.env` | Enable Prometheus + Grafana monitoring stack |
| `PROMETHEUS_SCRAPE_PORT` | `.env` | Prometheus scrape port |
| `PROMETHEUS_SCRAPE_INTERVAL` | `.env` | Prometheus scrape interval |
| `PROMETHEUS_EVALUATION_INTERVAL` | `.env` | Prometheus rule evaluation interval |
| `PROMETHEUS_SCRAPE_TIMEOUT` | `.env` | Prometheus scrape timeout |
| `GRAFANA_URL` | `.env` | External Grafana URL |
| `GRAFANA_SESSION_TIMEOUT` | `.env` | Grafana session timeout |
| `GRAFANA_ANONYMOUS_ACCESS` | `.env` | Grafana anonymous access |
| `GRAFANA_ANONYMOUS_ROLE` | `.env` | Grafana anonymous role |
| `PROMETHEUS_RETENTION_TIME` | `.env` | Prometheus data retention period |
| `PROMETHEUS_RETENTION_SIZE` | `.env` | Prometheus TSDB size limit |
| `PROMETHEUS_MAX_SAMPLES` | `.env` | Max samples per Prometheus query |
| `PROMETHEUS_QUERY_TIMEOUT` | `.env` | Prometheus query timeout |
| `PROMETHEUS_READ_TIMEOUT` | `.env` | Prometheus client read timeout |
| `PROMETHEUS_SCRAPE_CONCURRENCY` | `.env` | Prometheus concurrent scrape limit |
| `PROMETHEUS_SCRAPE_TARGETS` | `.env` | Semicolon-separated Prometheus targets |

### Security

| Variable | Source | Description |
|----------|--------|-------------|
| `AI_MEDIA_FACTORY_SECRET_KEY` | `.env` | Root shared secret for crypto signing across services |
| `N8N_ENCRYPTION_KEY` | `.env` | n8n credential encryption key (AES-256-GCM) |
| `QDRANT_HMAC_SIGNING_KEY` | `.env` | Qdrant HMAC webhook signing key |
| `MINIO_API_KEY` | `.env` | MinIO S3 API authentication key |
| `QDRANT_API_KEY` | `.env` | Qdrant API endpoint authentication key |
| `OLLAMA_API_KEY` | `.env` | Ollama HTTP API bearer token |
| `PROMETHEUS_API_KEY` | `.env` | Prometheus API key for external tools |
| `GRAFANA_API_KEY` | `.env` | Grafana API key for external tools |
| `NGINX_ADMIN_API_KEY` | `.env` | Nginx management API key |
| `GRAFANA_SECRET_KEY` | `.env` | Grafana session signing key |
| `GRAFANA_ADMIN_PASSWORD` | `.env` | Grafana admin password (≥32 chars) |
| `N8N_PASSWORD` | `.env` | n8n basic auth password |
| `N8N_DEFAULT_USER_ROLE` | `.env` | Default n8n user role |
| `REDIS_PASSWORD` | `.env` | Redis authentication password |
| `N8N_POSTGRES_PASSWORD` | `.env` | n8n PostgreSQL service user password |
| `QDRANT_POSTGRES_PASSWORD` | `.env` | Qdrant PostgreSQL service user password |
| `PROMETHEUS_POSTGRES_PASSWORD` | `.env` | Prometheus PostgreSQL exporter password |
| `GRAFANA_POSTGRES_PASSWORD` | `.env` | Grafana PostgreSQL datasource password |
| `N8N_MTLS_ENABLED` | `.env` | Enable mTLS for n8n API endpoints |
| `N8N_MTLS_CLIENT_CERT_PATH` | `.env` | mTLS client certificate path |
| `N8N_MTLS_CLIENT_KEY_PATH` | `.env` | mTLS client private key path |
| `NGINX_RATELIMIT_ENABLED` | `.env` | Enable Nginx rate limiting |
| `NGINX_RATELIMIT_RPM` | `.env` | Rate limit requests per minute |
| `NGINX_RATELIMIT_CONCURRENT` | `.env` | Rate limit concurrent connections |
| `NGINX_RATELIMIT_BURST` | `.env` | Rate limit burst multiplier |
| `NGINX_CLIENT_MAX_BODY_SIZE` | `.env` | Nginx maximum request body size |
| `NGINX_TLS_EARLY_DATA` | `.env` | Enable TLS 1.3 early data (0-RTT) |

### Encryption

| Variable | Source | Description |
|----------|--------|-------------|
| `N8N_ENCRYPTION_KEY` | `.env` | n8n credential encryption master key |
| `N8N_ENCRYPTION_CIPHER` | `.env` | n8n encryption cipher algorithm |
| `N8N_ENCRYPTION_KDF` | `.env` | Key derivation function |
| `N8N_ENCRYPTION_ITERATIONS` | `.env` | PBKDF2 key derivation iterations |
| `N8N_HMAC_KEY` | `.env` | n8n workflow integrity HMAC key |
| `QDRANT_PAYLOAD_ENCRYPTION` | `.env` | Qdrant payload encryption algorithm |
| `MINIO_TLS_ENABLED` | `.env` | Enable MinIO TLS encryption |
| `MINIO_TLS_CERT_PATH` | `.env` | MinIO TLS certificate path |
| `MINIO_TLS_KEY_PATH` | `.env` | MinIO TLS private key path |

### GPU

| Variable | Source | Description |
|----------|--------|-------------|
| `OLLAMA_GPU_DEVICES` | `.env` | GPU devices exposed to Ollama |
| `GPU_DRIVER` | `.env` | GPU driver type (nvidia, rocm) |
| `OLLAMA_GPU_VRAM_LIMIT` | `.env` | Maximum VRAM allocation for Ollama |
| `OLLAMA_GPU_LOCK_MEMORY` | `.env` | Lock Ollama GPU memory to prevent swapping |
| `OLLAMA_GPU_MAX_CONCURRENT_REQUESTS` | `.env` | Max concurrent Ollama GPU inference requests |
| `OLLAMA_CUDA_GRAPHS_ENABLED` | `.env` | Enable CUDA graphs for Ollama |
| `OLLAMA_CUDA_MIN_COMPUTE_CAPABILITY` | `.env` | Minimum CUDA compute capability |
| `OLLAMA_TENSOR_PARALLELISM_ENABLED` | `.env` | Enable Ollama tensor parallelism |
| `OLLAMA_TENSOR_PARALLELISM_DEVICES` | `.env` | GPUs for tensor parallelism |
| `OLLAMA_FLASH_ATTENTION` | `.env` | Enable Ollama Flash Attention |
| `OLLAMA_VULKAN` | `.env` | Enable Vulkan compute backend |
| `OLLAMA_VULKAN_NUM_QUEUES` | `.env` | Vulkan compute queues for Ollama |
| `OLLAMA_VULKAN_MEMORY_ALLOCATOR` | `.env` | Vulkan memory allocator type |

### CPU

| Variable | Source | Description |
|----------|--------|-------------|
| `N8N_CPU_LIMIT` | `.env`, `deploy.resources.limits.cpus` | n8n maximum CPU allocation |
| `N8N_CPU_RESERVATION` | `.env`, `deploy.resources.reservations.cpus` | n8n minimum CPU reservation |
| `PG_CPU_LIMIT` | `.env`, `deploy.resources.limits.cpus` | PostgreSQL maximum CPU allocation |
| `PG_CPU_RESERVATION` | `.env`, `deploy.resources.reservations.cpus` | PostgreSQL minimum CPU reservation |
| `REDIS_CPU_LIMIT` | `.env`, `deploy.resources.limits.cpus` | Redis maximum CPU allocation |
| `REDIS_CPU_RESERVATION` | `.env`, `deploy.resources.reservations.cpus` | Redis minimum CPU reservation |
| `MINIO_CPU_LIMIT` | `.env`, `deploy.resources.limits.cpus` | MinIO maximum CPU allocation |
| `MINIO_CPU_RESERVATION` | `.env`, `deploy.resources.reservations.cpus` | MinIO minimum CPU reservation |
| `QDRANT_CPU_LIMIT` | `.env`, `deploy.resources.limits.cpus` | Qdrant maximum CPU allocation |
| `QDRANT_CPU_RESERVATION` | `.env`, `deploy.resources.reservations.cpus` | Qdrant minimum CPU reservation |
| `OLLAMA_CPU_LIMIT` | `.env`, `deploy.resources.limits.cpus` | Ollama maximum CPU allocation |
| `OLLAMA_CPU_RESERVATION` | `.env`, `deploy.resources.reservations.cpus` | Ollama minimum CPU reservation |
| `NGINX_CPU_LIMIT` | `.env`, `deploy.resources.limits.cpus` | Nginx maximum CPU allocation |
| `NGINX_CPU_RESERVATION` | `.env`, `deploy.resources.reservations.cpus` | Nginx minimum CPU reservation |
| `PROMETHEUS_CPU_LIMIT` | `.env`, `deploy.resources.limits.cpus` | Prometheus maximum CPU allocation |
| `PROMETHEUS_CPU_RESERVATION` | `.env`, `deploy.resources.reservations.cpus` | Prometheus minimum CPU reservation |
| `GRAFANA_CPU_LIMIT` | `.env`, `deploy.resources.limits.cpus` | Grafana maximum CPU allocation |
| `GRAFANA_CPU_RESERVATION` | `.env`, `deploy.resources.reservations.cpus` | Grafana minimum CPU reservation |
| `OLLAMA_CPU_INFERENCE_CORES` | `.env` | CPU cores for Ollama CPU-only inference |
| `HOST_CPU_RESERVED_CORES` | `.env` | CPU cores reserved for host OS and Docker daemon |
| `DOCKER_CPU_SCHEDULER` | `.env` | Docker CPU scheduling mode |

### Docker

| Variable | Source | Description |
|----------|--------|-------------|
| `DOCKER_COMPOSE_PROJECT_NAME` | `.env` | Compose project name |
| `DOCKER_CGROUP_DRIVER` | `.env` | Docker daemon cgroup driver |
| `DOCKER_PULL_CONCURRENCY` | `.env` | Max concurrent image pulls |
| `DOCKER_AUTO_REMOVE` | `.env` | Auto-remove stopped containers |
| `DOCKER_LEAN_IMAGES` | `.env` | Disable intermediate layer storage |
| `DOCKER_UMASK` | `.env` | Default umask for container files |
| `DOCKER_DAEMON_LOG_DRIVER` | `.env` | Host Docker daemon log driver |
| `DOCKER_DAEMON_LOG_MAX_SIZE` | `.env` | Host Docker daemon log max size |
| `DOCKER_DAEMON_LOG_MAX_FILES` | `.env` | Host Docker daemon log retention |
| `DOCKER_LIVE_RESTORE` | `.env` | Enable live container restore after daemon restart |
| `DOCKER_DNS1` / `DOCKER_DNS2` | `.env` | Container DNS servers |
| `DOCKER_DEFAULT_BRIDGE` | `.env` | Docker bridge network CIDR |
| `DOCKER_MTU` | `.env` | Docker container MTU |
| `DOCKER_IPTABLES` | `.env` | Docker iptables masquerading |
| `DOCKER_IPTABLES_FORWARD` | `.env` | Docker iptables port forwarding |
| `DOCKER_IMAGE_PULL_POLICY` | `.env` | Docker image pull policy |
| `DOCKER_LABELS_ENABLED` | `.env` | Expose Docker container labels to API |
| `DOCKER_IMAGE_CACHE_MAX_SIZE` | `.env` | Maximum Docker image cache size |
| `DOCKER_SOCKET_PATH` | `.env` | Docker socket path for container management |

### Reverse Proxy

| Variable | Source | Description |
|----------|--------|-------------|
| `NGINX_STRIP_FORWARDED_PROTO` | `.env` | Strip X-Forwarded-Proto header |
| `NGINX_REAL_IP_ENABLED` | `.env` | Enable X-Real-IP header |
| `NGINX_REAL_IP_HEADER` | `.env` | HTTP header for client IP detection |
| `NGINX_TRUSTED_PROXY_RANGES` | `.env` | Trusted proxy IP ranges |
| `NGINX_CLIENT_MAX_BODY_SIZE` | `.env` | Maximum request body size |
| `NGINX_WORKER_PROCESSES` | `.env` | Nginx worker process count |
| `NGINX_WORKER_CONNECTIONS` | `.env` | Nginx worker connection count |
| `NGINX_SENDFILE` | `.env` | Enable kernel-level file transfer optimization |
| `NGINX_TCP_NOPUSH` | `.env` | Enable TCP Nagle optimization |
| `NGINX_TCP_KEEPALIVE_IDLE` | `.env` | TCP keepalive idle time |
| `NGINX_TCP_KEEPALIVE_PROBES` | `.env` | TCP keepalive probe count |
| `NGINX_TCP_KEEPALIVE_INTERVAL` | `.env` | TCP keepalive probe interval |
| `NGINX_GZIP_ENABLED` | `.env` | Enable gzip compression |
| `NGINX_GZIP_MIN_LENGTH` | `.env` | Minimum response size for gzip |
| `NGINX_GZIP_COMP_LEVEL` | `.env` | gzip compression level (1–9) |
| `NGINX_GZIP_TYPES` | `.env` | MIME types for gzip compression |
| `NGINX_CLIENT_BODY_BUFFER_SIZE` | `.env` | Nginx request body buffer size |
| `NGINX_PROXY_BUFFER_SIZE` | `.env` | Nginx upstream response header buffer |
| `NGINX_PROXY_BUFFERS` | `.env` | Nginx upstream response body buffers |
| `NGINX_BUSY_BUFFERS_SIZE` | `.env` | Nginx temporary response buffer size |
| `NGINX_CLIENT_MAX_TEMP_FILE_SIZE` | `.env` | Max temp file size for requests |
| `NGINX_PROXY_CONNECT_TIMEOUT` | `.env` | Nginx upstream connection timeout |
| `NGINX_PROXY_SEND_TIMEOUT` | `.env` | Nginx upstream send timeout |
| `NGINX_PROXY_READ_TIMEOUT` | `.env` | Nginx upstream read timeout |
| `NGINX_PROXY_NEXT_UPSTREAM_TIMEOUT` | `.env` | Nginx next upstream retry timeout |
| `NGINX_HTTP2_ENABLED` | `.env` | Enable HTTP/2 on HTTPS listener |
| `NGINX_HTTP3_ENABLED` | `.env` | Enable HTTP/3 (QUIC) |
| `NGINX_BROTLI_ENABLED` | `.env` | Enable Brotli compression |
| `NGINX_OCSP_STAPLING` | `.env` | Enable OCSP stapling |
| `NGINX_TLS_SESSION_TICKETS` | `.env` | Enable TLS session tickets |
| `NGINX_TLS_TICKET_KEY_ROTATION` | `.env` | TLS ticket key rotation interval |
| `NGINX_SERVER_TOKENS` | `.env` | Suppress Nginx version in headers |
| `NGINX_PROXY_HIDE_HEADERS_SERVER` | `.env` | Hide upstream server header |
| `NGINX_PROXY_PROTOCOL_ENABLED` | `.env` | Accept proxy protocol v2 headers |
| `NGINX_PROXY_PROTOCOL_TIMEOUT` | `.env` | Proxy protocol header timeout |
| `NGINX_CSP_ENABLED` | `.env` | Enable Content Security Policy headers |
| `NGINX_CSP_DIRECTIVES` | `.env` | CSP directives string |
| `NGINX_X_CONTENT_TYPE_OPTIONS` | `.env` | X-Content-Type-Options header |
| `NGINX_X_FRAME_OPTIONS` | `.env` | X-Frame-Options header |
| `NGINX_REFERRER_POLICY` | `.env` | Referrer-Policy header |
| `NGINX_PERMISSIONS_POLICY` | `.env` | Permissions-Policy header |
| `NGINX_CORS_ENABLED` | `.env` | Enable CORS headers |
| `NGINX_CORS_ALLOW_ORIGIN` | `.env` | CORS Allow-Origin values |
| `NGINX_CORS_ALLOW_METHODS` | `.env` | CORS Allow-Methods values |
| `NGINX_CORS_ALLOW_HEADERS` | `.env` | CORS Allow-Headers values |
| `NGINX_CORS_ALLOW_CREDENTIALS` | `.env` | CORS Access-Control-Allow-Credentials |
| `NGINX_CORS_MAX_AGE` | `.env` | CORS Access-Control-Max-Age |
| `NGINX_COEP` | `.env` | Cross-Origin Opener Policy |
| `NGINX_CORP` | `.env` | Cross-Origin Resource Policy |
| `NGINX_DNS_PREFETCH_CONTROL` | `.env` | DNS Prefetch Control header |

---

## Profiles

Docker Compose profiles allow selective service activation. Use `--profile <name>` to include additional services not activated by default.

| Profile | Activation | Services | Usage |
|---------|-----------|----------|-------|
| `base` | `docker compose up -d` | n8n, PostgreSQL, Redis, MinIO, Qdrant, Ollama, Nginx | Default production stack |
| `production` | `docker compose --profile production up -d` | Same as `base` | Explicit production mode with all production-hardening features active |
| `monitoring` | `docker compose --profile monitoring up -d` | Adds Prometheus, Grafana | Activates the observability stack |

### Profile Combinations

```bash
# Base production stack
docker compose up -d

# Production stack with monitoring
docker compose --profile monitoring up -d

# Full stack with monitoring
docker compose --profile production --profile monitoring up -d
```

---

## CPU Mode

The AI Media Factory infrastructure runs on CPU by default. Ollama in CPU-only mode handles inference using the host CPU.

### CPU Mode Configuration

1. Set `OLLAMA_GPU_DEVICES` to `""` (empty) in `.env` to disable GPU offloading.
2. Set `OLLAMA_CPU_INFERENCE_CORES` in `.env` to limit the CPU cores used for inference (default: 4).
3. Set `OLLAMA_NUM_PARALLEL` to 1 for CPU-only inference (higher values cause CPU thrashing).
4. Set `OLLAMA_MAX_LOADED_MODELS` to 1 for CPU-only inference (each model consumes significant RAM).
5. Ensure `OLLAMA_VULKAN=0` if Vulkan is not needed.

### Recommended CPU-Only Resource Values

| Service | CPU Limit | Memory Limit |
|---------|-----------|-------------|
| n8n | 2.0 | 1G |
| PostgreSQL | 2.0 | 2G |
| Redis | 1.0 | 1G |
| MinIO | 2.0 | 2G |
| Qdrant | 2.0 | 4G |
| Ollama (CPU) | 4.0 | 8G |
| Nginx | 1.0 | 256M |

---

## GPU Mode

Ollama supports GPU-accelerated inference on NVIDIA GPUs via the NVIDIA Container Toolkit. AMD GPUs are supported via the Vulkan backend.

### NVIDIA GPU Setup

1. **Install NVIDIA Container Toolkit** on the host:
   ```bash
   # Ubuntu/Debian
   curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
   curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
     sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
     sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
   sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
   sudo nvidia-ctk runtime configure --runtime=docker
   sudo systemctl restart docker
   ```

2. **Verify GPU access** from a container:
   ```bash
   docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
   ```

3. **Set `.env` values for Ollama GPU mode:**
   | Variable | Value |
   |----------|-------|
   | `OLLAMA_GPU_DEVICES` | `all` |
   | `OLLAMA_GPU_VRAM_LIMIT` | Set to available VRAM in bytes (e.g., `16384000000` for 16 GB) |
   | `OLLAMA_GPU_LOCK_MEMORY` | `true` |
   | `OLLAMA_CUDA_GRAPHS_ENABLED` | `true` |
   | `OLLAMA_FLASH_ATTENTION` | `true` |

4. **Set Ollama resource limits in `.env`:**
   | Variable | Recommended Value |
   |----------|-------------------|
   | `OLLAMA_CPU_LIMIT` | 8.0 |
   | `OLLAMA_MEMORY_LIMIT` | 32G |
   | `OLLAMA_GPU_LIMIT` | 1 (GPU device count) |

### AMD GPU (Vulkan) Setup

1. **Install Vulkan SDK** on the host:
   ```bash
   sudo apt-get install -y vulkan-tools mesa-vulkan-drivers vulkan-utils
   ```

2. **Set `.env` values for Vulkan mode:**
   | Variable | Value |
   |----------|-------|
   | `OLLAMA_VULKAN` | `1` |
   | `OLLAMA_VULKAN_MEMORY_ALLOCATOR` | `unified` |
   | `GPU_DRIVER` | `vulkan` |

3. **Verify Vulkan access:**
   ```bash
   docker run --rm vulkan/vulkan-samples:v1.3.275 vulkaninfo --summary
   ```

---

## Updating

### Rolling Update Process

1. Review the latest release notes for any breaking changes in the new version.
2. Update the `docker-compose.yml` `image:` tags for services that require updates.
3. Update `.env` variables if new required variables have been added.
4. Run `docker compose config` to validate the updated configuration.
5. Pull updated images:
   ```bash
   docker compose pull
   ```
6. Perform a rolling restart with zero-downtime:
   ```bash
   docker compose up -d --no-deps ollama minio qdrant n8n postgres redis nginx
   ```
7. Verify service health after each restart:
   ```bash
   docker compose ps
   docker compose exec n8n n8n healthcheck --url http://localhost:5678/healthz
   ```
8. Monitor logs for errors during the rolling update:
   ```bash
   docker compose logs --follow --tail 50 ollama
   ```

### Update Checklist

- [ ] Backup configuration and data before updating.
- [ ] Review changelog for breaking changes.
- [ ] Validate `docker compose config` with the updated docker-compose.yml.
- [ ] Pull new images.
- [ ] Restart services in dependency order.
- [ ] Verify health checks pass for all services.
- [ ] Run the n8n test suite if available.
- [ ] Monitor Grafana dashboards for anomalies post-update.
- [ ] Update the Project Registry with the new version.

---

## Backup

### Automated Backup Procedure

Backups are automated via cron-scheduled tasks referenced in the `.env` configuration. The following data types are backed up:

1. **PostgreSQL** — Full logical backup via `pg_dump` on the schedule defined by `BACKUP_PG_CRON_SCHEDULE`.
2. **MinIO** — Bucket lifecycle policies handle data tiering and eventual offsite replication.
3. **Qdrant** — Snapshot-based backup on the schedule defined by `BACKUP_QDRANT_CRON_SCHEDULE`.
4. **Redis** — RDB snapshot files and AOF log files are persisted in `redis_data` and `redis_logs` volumes.
5. **n8n** — Workflow JSON, credentials, and execution records are stored in the `n8n_data` volume and backed up via the PostgreSQL backup.
6. **Ollama** — Model files are stored in `ollama_models` volume. Automated Ollama backup is disabled by default (`BACKUP_OLLAMA_ENABLED=false`). Enable if model files require backup.
7. **Nginx** — TLS certificates (`nginx_certs` volume) and configuration (`nginx_conf` volume). Manual backup recommended.

### Manual Backup Commands

```bash
# Backup PostgreSQL database
docker compose exec postgres pg_dump -U ${POSTGRES_USER} -d ${POSTGRES_DB} > backup_$(date +%Y%m%d_%H%M%S).sql

# Backup n8n data volume
docker run --rm -v ai-media-factory-n8n_data:/data -v $(pwd)/backups:/backup alpine tar czf /backup/n8n_data_$(date +%Y%m%d).tar.gz -C /data .

# Backup MinIO data volume
docker run --rm -v ai-media-factory-minio_data:/data -v $(pwd)/backups:/backup alpine tar czf /backup/minio_data_$(date +%Y%m%d).tar.gz -C /data .

# Backup Qdrant snapshots
# Qdrant snapshots are stored in qdrant_snapshots volume. Copy manually:
docker run --rm -v ai-media-factory-qdrant_snapshots:/snapshots -v $(pwd)/backups:/backup alpine cp -r /snapshots /backup/qdrant_snapshots_$(date +%Y%m%d)

# Backup Ollama models
docker run --rm -v ai-media-factory-ollama_models:/models -v $(pwd)/backups:/backup alpine tar czf /backup/ollama_models_$(date +%Y%m%d).tar.gz -C /models .

# Backup configuration files (n8n, redis, minio, qdrant configs)
# These are stored in the respective config volumes and should be backed up alongside data backups.
```

### Backup Verification

The `BACKUP_HEALTH_CHECK_INTERVAL` in `.env` controls how frequently automated health checks validate backup integrity. Failed backup verification triggers an alert to the email address specified by `BACKUP_ALERT_EMAIL`.

---

## Restore

### PostgreSQL Restore

```bash
# Stop n8n and Qdrant to prevent data mutations during restore
docker compose stop n8n qdrant

# Restore from a dump file
docker compose exec -T postgres bash -c "psql -U ${POSTGRES_USER} -d ${POSTGRES_DB} < /backup/backup_FILE.sql"

# Start services in order
docker compose start postgres redis minio ollama qdrant n8n nginx
```

### MinIO Restore

```bash
# Restore MinIO data volume from backup
docker compose stop minio
docker run --rm -v ai-media-factory-minio_data:/data -v $(pwd)/backups:/backup alpine sh -c "rm -rf /data/* && tar xzf /backup/minio_data_BACKUP.tar.gz -C /data"
docker compose start minio
```

### Qdrant Restore

```bash
# Copy snapshot files to Qdrant snapshots volume
docker compose stop qdrant
docker run --rm -v ai-media-factory-qdrant_snapshots:/snapshots -v $(pwd)/backups/qdrant_snapshots_RESTORE:/restore alpine sh -c "cp -r /restore/* /snapshots/"
docker compose start qdrant

# Trigger snapshot restore via Qdrant API (requires QDRANT_API_KEY from .env)
curl -X POST "http://localhost:6333/collections/COLLECTION_NAME/cluster" \
  -H "api-key: ${QDRANT_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"restore_snapshot": "/qdrant/snapshots/SNAPSHOT_FILE"}'
```

### Ollama Model Restore

```bash
# Stop Ollama
docker compose stop ollama

# Restore model files
docker run --rm -v ai-media-factory-ollama_models:/models -v $(pwd)/backups:/backup alpine sh -c "rm -rf /models/* && tar xzf /backup/ollama_models_BACKUP.tar.gz -C /models"

# Start Ollama — models auto-load if configured
docker compose start ollama
```

### Nginx Certificate Restore

```bash
# Copy TLS certificates back to the secrets directory
cp server.crt /run/secrets/ai-media-factory-tls-crt
cp server.key /run/secrets/ai-media-factory-tls-key
cp ca-bundle.crt /run/secrets/ai-media-factory-tls-ca

# Restart Nginx to pick up restored certificates
docker compose restart nginx
```

---

## Migration

### Major Version Migration

Major version migrations (e.g., PostgreSQL 16 → 17, Redis 7 → 8) require careful planning:

1. **Pre-migration audit** — Document current data sizes, connection counts, and active workflows.
2. **Snapshot** — Take a full backup of all volumes and database dumps.
3. **Test migration** — Run the migration on a staging host with production data snapshots before applying to production.
4. **Update compose file** — Change image tags in `docker-compose.yml` and `.env`.
5. **Update configuration** — Review `.env` for any new or deprecated variables for the target version.
6. **Rolling restart** — Restart services in dependency order with health check verification between each step.
7. **Post-migration validation** — Run connection tests, query benchmarks, and workflow execution tests.

### Cloud Provider Migration

The infrastructure is designed to be portable between cloud providers without architectural changes. To migrate:

1. Stop all containers: `docker compose down`
2. Create the new cloud provider's block storage and attach it at the same host path (`/opt/ai-media-factory`).
3. Copy all volume data to the new storage:
   ```bash
   rsync -avz /opt/ai-media-factory/ NEW_STORAGE_MOUNT/
   ```
4. Update DNS records for the new provider's IP address.
5. Start containers on the new host: `docker compose up -d`
6. Verify all health checks pass.
7. Update the Project Registry with the new provider details.

---

## Scaling

### Horizontal Scaling

The platform supports horizontal scaling for stateless services and specific stateful services:

| Service | Scaling Method | Notes |
|---------|---------------|-------|
| **n8n** | Increase `N8N_RUNNERS` in `.env` | Additional runner containers can be deployed behind a load balancer |
| **Nginx** | Horizontal scaling behind an external load balancer | Nginx is stateless for request routing; session affinity required for WebSocket-heavy workflows |
| **Redis** | Redis Sentinel or Redis Cluster for HA | Currently runs as a single instance; cluster setup requires changes to the compose file |
| **MinIO** | MinIO distributed mode (multi-node) | Requires setting `MINIO_COMPUTE_NODES` > 1 and deploying multiple MinIO containers with shared storage |
| **Qdrant** | Qdrant cluster mode | Enable `QDRANT_CLUSTER_ENABLED` and configure peer nodes |
| **PostgreSQL** | Streaming replication + read replicas | Currently runs as a single instance; read replicas require separate compose configuration |
| **Ollama** | Multiple Ollama models on a single GPU; tensor parallelism across GPUs | Enable `OLLAMA_TENSOR_PARALLELISM_ENABLED` for multi-GPU inference |

### Vertical Scaling

Scale a single service by adjusting its resource limits:

Edit the limits directly in `docker-compose.yml` and `.env`, then run `docker compose up -d` to apply changes.

### Auto-Scaling Considerations

Docker Compose does not include native auto-scaling. For production auto-scaling:

- Use Docker Swarm with `--replicas` for stateless services.
- Use Kubernetes with HPA (Horizontal Pod Autoscaler) for container orchestration.
- Use cloud provider load balancers with health check integration.

---

## Monitoring

### Built-In Monitoring (Optional Profile)

Activate `PROMETHEUS_SCRAPE_PORT=9090` in `.env` and enable the `--profile monitoring` flag to activate the Prometheus + Grafana observability stack.

**Prometheus scrapes every service** according to the interval defined by `PROMETHEUS_SCRAPE_INTERVAL`. Targets are defined in the `PROMETHEUS_SCRAPE_TARGETS` variable.

**Grafana dashboards** can be imported using standard Prometheus + container-exporter dashboard JSON files. The default admin dashboard includes panels for:

- Container CPU and memory usage per service
- Request rate and latency per Nginx upstream
- PostgreSQL connection count, active queries, and replication lag
- Redis memory usage, hit rate, and command statistics
- MinIO bucket count, object count, and storage usage
- Qdrant collection count and index size
- Ollama loaded models, GPU utilization, and inference latency
- n8n execution count, success rate, and queue depth

### Health Check Endpoints

| Service | Endpoint | Port | Auth |
|---------|----------|------|------|
| Nginx | `/health` | 8080 | None |
| n8n | `/healthz` | 5678 | Basic auth (if enabled) |
| PostgreSQL | `pg_isready` | 5432 | None (socket) |
| Redis | `redis-cli ping` | 6379 | Password |
| MinIO | `/minio/health/live` | 9000 | None |
| Qdrant | `/healthz` | 6333 | API key |
| Ollama | `/api/tags` | 11434 | Bearer token |

### Custom Alerts

Configure alerts in the `prometheus_rules` volume directory. Prometheus Rule Files (YAML format) can be placed at the path configured by `AI_MEDIA_FACTORY_CONFIGS_DIR/prometheus/rules/`.

---

## Logs

### Accessing Logs

```bash
# View all logs for a specific service
docker compose logs ollama

# View real-time logs with follow
docker compose logs --follow ollama

# View logs from the last 100 lines
docker compose logs --tail 100 ollama

# View logs with timestamps
docker compose logs --timestamps ollama

# View logs for multiple services
docker compose logs n8n ollama qdrant

# Filter logs by level (requires JSON log driver)
docker compose logs ollama | grep '"level":"error"'
```

### Log Storage and Rotation

Each service has Docker-level log rotation configured via the `logging` block in `docker-compose.yml`:

| Service | Max Size | Max Files | Compress |
|---------|----------|-----------|----------|
| All services | 10m – 20m (varies) | 3 – 5 (varies) | Yes |

Log files are stored on the host at the Docker daemon's log directory. Default path: `/var/lib/docker/containers/<container-id>/<container-id>-json.log`.

### Structured Logging

Qdrant, Prometheus, and Nginx emit structured JSON logs for compatibility with centralized log aggregators (e.g., Loki, ELK, Datadog). Enable JSON logging for all services by setting the appropriate log-level variables.

### Log Levels Reference

| Service | Production | Debug |
|---------|-----------|-------|
| n8n | `info` | `debug` |
| PostgreSQL | `log` | `debug5` |
| Redis | `notice` | `verbose` |
| MinIO | `info` | `debug` |
| Qdrant | `info` | `debug` |
| Ollama | `warn` | `debug` |
| Nginx | `notice` | `debug` |

---

## Troubleshooting

### Common Issues and Resolutions

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| Docker daemon refuses to start | Incorrect cgroup driver configuration | Verify `DOCKER_CGROUP_DRIVER` matches `/etc/docker/daemon.json`. Set to `systemd` on modern systems. |
| PostgreSQL fails to start | Incorrect `POSTGRES_PASSWORD` or disk full | Check `docker compose logs postgres` for authentication errors. Verify `pg_data` volume has disk space. |
| Redis authentication failure | Mismatched `REDIS_PASSWORD` between .env and redis.conf | Ensure `REDIS_PASSWORD` in `.env` matches the `--requirepass` argument in the redis entrypoint. |
| Ollama model loads slowly or fails | Insufficient VRAM or RAM for model | Check `OLLAMA_GPU_VRAM_LIMIT` against model requirements. Reduce `OLLAMA_MAX_LOADED_MODELS`. |
| n8n webhooks not reachable | Nginx upstream misconfiguration or port collision | Verify `N8N_HOST` in `.env` resolves to the server IP. Check `docker compose port n8n 5678`. |
| MinIO console returns 403 | Authentication or CORS misconfiguration | Verify `MINIO_ROOT_USER`/`MINIO_ROOT_PASSWORD` in `.env`. Check `MINIO_DOMAIN` DNS resolution. |
| Qdrant search returns empty results | No vectors loaded or vector dimension mismatch | Verify vectors are indexed. Check dimension consistency between insert and search operations. |
| Container OOM killed | Memory limit too low for the workload | Increase `OLLAMA_MEMORY_LIMIT`, `PG_MEMORY_LIMIT`, or relevant service memory in the compose file. |
| TLS handshake fails | Certificate mismatch or weak cipher configuration | Verify TLS_CERT_PATH and TLS_KEY_PATH point to correct files. Check `TLS_MIN_VERSION` compatibility. |
| Rate limiting causes 429 errors | Nginx `NGINX_RATELIMIT_RPM` set too low for legitimate traffic | Increase `NGINX_RATELIMIT_RPM` or adjust `NGINX_RATELIMIT_BURST`. |

### Debug Mode

Temporarily increase log verbosity for a service to diagnose issues:

```bash
# Edit .env to set log level to debug for the affected service
# Example: QDRANT_LOG_LEVEL=debug
# Then restart the service
docker compose restart qdrant

# View debug output in real time
docker compose logs --follow qdrant
```

### Resource Exhaustion

If the host runs low on resources:

```bash
# Check container resource usage
docker stats

# Check host disk usage of volumes
docker system df
df -h /opt/ai-media-factory

# Check host memory
free -h

# Check host swap usage (should be near zero for database workloads)
swapon --show

# Prune unused Docker resources (use with caution)
docker system prune -a --volumes
```

---

## Security

### Security Hardening Applied

The Docker Compose configuration applies the following security hardening to every container by default:

| Hardening Measure | Applies To | Description |
|-------------------|-----------|-------------|
| **`read_only: true`** | n8n, PostgreSQL, Redis, MinIO, Qdrant | Mounts filesystem as read-only; only designated writable paths are accessible via tmpfs or volumes. |
| **`cap_drop: ALL`** | All services | Drops all Linux capabilities. Only explicitly added capabilities (`cap_add`) are granted. |
| **`security_opt: no-new-privileges`** | All services (except Ollama) | Prevents processes from gaining additional privileges beyond their initial user namespace. |
| **`tmpfs` mounts** | All `read_only` services | Provides ephemeral writable storage for temporary files without persisting to disk. |
| **`privileged: false`** | All services except Ollama | Restricts containers from accessing host devices and kernel modules. |
| **Non-root user** | Nginx (`nginx-unprivileged`), MinIO, Qdrant | Services run as non-root users inside the container where available. |
| **`no-new-privileges`** | All containers | Prevents setuid/setgid binaries from escalating privileges. |
| **Resource limits** | All services | CPU and memory limits prevent a single service from starving other services of resources (denial of service). |
| **TLS everywhere** | Nginx → upstream services | All external traffic terminates TLS at Nginx. Internal traffic on the Docker network is unencrypted; use TLS internally if the Docker network is on a shared host. |
| **mTLS (optional)** | n8n API | Enable `N8N_MTLS_ENABLED` to require mutual TLS authentication for n8n API endpoints. |
| **Rate limiting** | Nginx reverse proxy | Prevents brute-force and denial-of-service attacks at the network edge. |

### Network Security

The six network isolation tiers ensure minimum blast radius:

1. **frontend** — Clients can only reach Nginx. No service is directly accessible from the frontend network.
2. **backend** — Application services communicate with each other and with databases/storage.
3. **database** — Only PostgreSQL and Redis are accessible. No application traffic on this network.
4. **storage** — Only MinIO is accessible. No application traffic on this network.
5. **ai** — Qdrant and Ollama have isolated high-throughput communication. No client traffic from the frontend.
6. **management** — Monitoring tools are isolated from all application networks. Only reachable from the backend (for scraping) and the host (for administrative access).

### Secret Management

In production deployments, all secrets defined in `.env` should be migrated to Docker Secrets or an external secrets manager (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault):

```bash
# Example: Create Docker secrets from .env values
echo -n "${POSTGRES_PASSWORD}" | docker secret create ai-media-factory-postgres_password -
echo -n "${REDIS_PASSWORD}" | docker secret create ai-media-factory-redis_password -
# ... repeat for each secret
```

Update `docker-compose.yml` to reference secrets instead of `environment` variables for all sensitive values. See the `secrets` block in the compose file for the defined secret names.

---

## Performance Tuning

### PostgreSQL Tuning

| Parameter | Production Value | Tuning Guidance |
|-----------|-----------------|----------------|
| `shared_buffers` | 25% of available RAM | Never exceed 40% of total system RAM. |
| `effective_cache_size` | ~3× shared_buffers | Helps the planner estimate index scan vs sequential scan cost. |
| `work_mem` | 16MB → 64MB for complex queries | Increase for OLAP workloads; decrease for OLTP to prevent OOM. |
| `maintenance_work_mem` | 128MB → 512MB for large tables | Increase for large VACUUM and CREATE INDEX operations. |
| `max_parallel_workers_per_gather` | 2–4 | Increase for large analytical queries on multi-core hosts. |
| `random_page_cost` | 1.1 for SSD, 4.0 for HDD | Set based on storage type to influence planner index usage. |
| `effective_io_concurrency` | 200 for SSD, 2 for HDD | Adjust based on storage subsystem capability. |

### Redis Tuning

| Parameter | Production Value | Tuning Guidance |
|-----------|-----------------|----------------|
| `maxmemory` | 50–70% of available RAM | Leave headroom for the host OS and the Redis fork operations. |
| `maxmemory-policy` | `allkeys-lru` | Evicts least-recently-used keys. Use `volatile-lru` if only expired keys should be evicted. |
| `appendfsync` | `everysec` | Balances durability and performance. `always` is safer but adds 1ms latency per write. |
| `hz` | 10 (default) | Higher hz values check more keys for expiration but consume CPU. |
| `tcp-backlog` | 511 → 4096 | Increase for high-connection workloads if `net.core.somaxconn` is also increased. |

### Nginx Tuning

| Parameter | Production Value | Tuning Guidance |
|-----------|-----------------|----------------|
| `worker_processes` | auto (matches CPU cores) | Increase to 2× CPU cores on high-core-count hosts for I/O-bound workloads. |
| `worker_connections` | 4096 | Max concurrent connections per worker. `worker_processes × worker_connections` = theoretical max. |
| `keepalive_timeout` | 65s (default) | Reduce for high-concurrency APIs; increase for long-lived WebSocket connections. |
| `gzip_comp_level` | 6 | Increase to 9 for bandwidth-critical environments at the cost of CPU. |
| `gzip_min_length` | 256 bytes | Increase to 1024 to avoid compressing very small responses where overhead exceeds benefit. |
| `proxy_buffer_size` | 4k → 8k for large headers | Increase if proxied services return large response headers. |
| `proxy_buffers` | 4 × 32k → 8 × 64k | Increase for services returning large response bodies (e.g., MinIO file downloads). |

### Ollama Tuning

| Parameter | Production Value | Tuning Guidance |
|-----------|-----------------|----------------|
| `OLLAMA_NUM_PARALLEL` | 1–2 per GPU | Increase for GPU-only inference on high-VRAM GPUs. Keep at 1 for CPU-only. |
| `OLLAMA_MAX_LOADED_MODELS` | 1–2 | Limit to available VRAM. Each model consumes VRAM equal to its size. |
| `OLLAMA_GPU_LAYERS` | Total model layer count | Set to maximum to offload all layers to GPU. Reduce if VRAM is insufficient. |
| `OLLAMA_CONTEXT_LENGTH` | 4096 → 8192+ | Increase for long-context workloads at the cost of VRAM (approximately 2MB per 1k tokens). |
| `OLLAMA_KEEP_ALIVE` | 5m | Reduce for memory-constrained environments; increase for frequently used models. |

---

## Best Practices

### Operational Best Practices

1. **Secrets rotation** — Rotate all passwords and API keys every 90 days. Use `.env.production-YYYYMMDD` snapshots to track credential history.
2. **Immutable tags** — Pin production images to specific version tags (e.g., `ollama/ollama:0.3.4`) rather than `latest`. Update tags deliberately through a change management process.
3. **Immutable infrastructure** — Treat containers as immutable. Never exec into a running container to make configuration changes. Rebuild and redeploy instead.
4. **Backup before every update** — Run manual backups before updating any `docker-compose.yml` or `.env` changes.
5. **Log rotation monitoring** — Monitor Docker log file sizes. If container log rotation triggers frequent rotations, investigate the application for excessive log spam.
6. **Health check validation** — Run `docker compose ps` at least once daily to verify service health. Configure out-of-band monitoring (e.g., PagerDuty) for health check alerting.
7. **Network segmentation** — Never publish all ports to the host. Use Nginx as the sole ingress point and keep database/storage/AI ports internal.
8. **Volume naming** — All named volumes follow the `ai-media-factory-<service>-<component>` naming convention for uniqueness and clarity.
9. **DNS consistency** — All service domains must resolve to the Nginx reverse proxy's IP. Do not expose individual service ports directly to the internet.
10. **Document deviations** — When a `.env` value deviates from the secure default, document the reason in a `CHANGES.md` file at the repository root.

### Security Best Practices

1. **Never commit `.env`** — Add `.env` to `.gitignore` immediately. Use `.env.example` with placeholder values instead.
2. **Least privilege** — Each service runs with the minimum capabilities required. Never run containers as root unless explicitly required (Ollama) and aware of the risks.
3. **TLS everywhere** — Even internal service communication should use TLS if the Docker network is on a shared host.
4. **Firewall rules** — Only ports 80 and 443 should be publicly accessible. All other ports (3000, 9000, 5432, 6379, etc.) must be firewalled.
5. **Certificate management** — Use Let's Encrypt with certbot or a managed certificate service for production TLS. Never use self-signed certificates in production.
6. **Audit logging** — MinIO audit logs and PostgreSQL query logs provide full transaction traceability. Ship these to a centralized SIEM.
7. **Container image scanning** — Scan all container images for CVEs before deploying to production. Use tools like `trivy`, `grype`, or Docker Scout.
8. **Watchtower usage** — The `com.centurylinklabs.watchtower.enable=true` label on all images enables automatic image updates. Ensure Watchtower runs on a schedule that matches your update policy.

---

## Upgrade Guide

### Pre-Upgrade Checklist

- [ ] Read the release notes for all services being upgraded.
- [ ] Take a full backup (database dump + all volume snapshots).
- [ ] Test the upgrade on a staging environment.
- [ ] Verify DNS and TLS certificates are valid and not expiring within 30 days.
- [ ] Check service dependencies between compose file version changes (e.g., Compose Spec updates).
- [ ] Review `.env` for any new required or deprecated variables.
- [ ] Notify stakeholders of the maintenance window.

### Upgrade Procedure

```bash
# 1. Enter maintenance mode (drain traffic)
# Redirect Nginx to a maintenance page or set a 503 status

# 2. Stop all non-database services
docker compose stop n8n qdrant ollama minio redis nginx

# 3. Pull new images
docker compose pull

# 4. Upgrade PostgreSQL if major version changed (see PostgreSQL migration below)
# This is the only service where in-place upgrades carry risk for data integrity.
# For minor/patch upgrades, proceed to step 5.

# 5. Start database first
docker compose start postgres

# 6. Start services in dependency order (database → cache → storage → AI → application → proxy)
docker compose start redis minio qdrant ollama n8n nginx

# 7. Verify health
docker compose ps
# Manually verify each health check endpoint

# 8. Exit maintenance mode
# Remove maintenance redirect or restore normal routing
```

### PostgreSQL Major Version Upgrade

PostgreSQL major version upgrades (e.g., 16 → 17) require special handling:

```bash
# 1. Take a full physical backup (pg_basebackup or pg_dumpall)
# 2. Create a new volume for the upgraded database:
#    docker volume create ai-media-factory-pg_data_new
# 3. Update docker-compose.yml image tag from postgres:16-bookworm to postgres:17-bookworm
# 4. Start the new PostgreSQL container (it will initialize the new data directory)
# 5. Run pg_upgrade or pg_dump/pg_restore to migrate data
# 6. Verify data integrity
# 7. Remove the old data volume (after confirming successful migration)
# 8. Restart all services
```

### Compose Specification Version Upgrade

When the Compose specification version changes (e.g., v3.7 → v3.8):

1. Update the compose file format version or remove the version field (Compose Spec is always implied by the Docker CLI version).
2. Run `docker compose config` to validate compatibility.
3. Review all deprecated or changed features against the Docker Compose changelog.

---

## Disaster Recovery

### Recovery Time Objectives (RTO) and Recovery Point Objectives (RPO)

| Service | RTO | RPO | Backup Method |
|---------|-----|-----|---------------|
| PostgreSQL | <5 min | <1 hour | `pg_dump` on schedule; point-in-time recovery from WAL |
| Redis | <2 min | <5 minutes | RDB snapshot + AOF file restore; Redis persistence on volume |
| MinIO | <10 min | <1 hour | MinIO data volume snapshot; lifecycle replication to offsite endpoint |
| Qdrant | <5 min | <5 minutes | Qdrant snapshot restore from `qdrant_snapshots` volume |
| Ollama | <15 min | <1 day | `ollama_models` volume snapshot; model download from registry |
| n8n | <5 min | <1 hour | `n8n_data` volume snapshot; PostgreSQL backup covers n8n state |
| Nginx | <1 min | None | Stateless; configuration in `nginx_conf` volume |

### Recovery Procedures

#### Full Infrastructure Recovery

1. Ensure the host OS and Docker Engine are functional.
2. Mount or reattach all persistent volumes to the correct paths under `/opt/ai-media-factory`.
3. Restore `.env` from the last known-good snapshot.
4. Restore `docker-compose.yml` from version control.
5. Run `docker compose config` to validate.
6. Run `docker compose up -d` to start all services.
7. Verify all health checks pass.
8. Verify n8n workflows and credentials are intact.
9. Verify Qdrant collections exist and contain expected vectors.
10. Verify MinIO buckets contain expected objects.

#### Single Service Recovery

```bash
# 1. Stop the failed service
docker compose stop <service-name>

# 2. Check logs for root cause
docker compose logs <service-name> | tail -100

# 3. Remove the failed container (if necessary)
docker rm ai-media-factory-<service-name>

# 4. Restore from backup if volume is corrupted
# Example for PostgreSQL:
docker volume rm ai-media-factory-pg_data
# Restore from backup to the volume mount path

# 5. Restart the service
docker compose up -d <service-name>

# 6. Verify health
docker compose ps
```

#### Data Corruption Recovery

```bash
# PostgreSQL point-in-time recovery (requires WAL archiving)
# 1. Stop PostgreSQL
docker compose stop postgres
# 2. Restore base backup + replay WAL files to target point in time
# 3. Start PostgreSQL (it will enter recovery mode automatically)
docker compose up -d postgres
# 4. Verify recovery status
docker compose exec postgres pg_isready -U ${POSTGRES_USER}
```

### Disaster Recovery Testing

Conduct DR drills at least quarterly:

1. Schedule a maintenance window.
2. Simulate a full infrastructure failure (stop all containers, corrupt a volume, restore from backup).
3. Execute the full infrastructure recovery procedure.
4. Measure RTO and RPO achievement.
5. Document findings and update this guide with any discrepancies discovered.

---

## Useful Docker Commands

```bash
# Check system-wide Docker resources
docker system df                                    # Disk usage summary
docker system events --since 1h                     # Events from the last hour
docker system info                                  # Docker system information
docker system prune -a --volumes                     # Remove all unused images, containers, networks, volumes (WARNING: aggressive)

# Container management
docker ps                                           # Running containers
docker ps -a                                        # All containers (including stopped)
docker stats                                        # Real-time resource usage
docker inspect <container-name>                     # Detailed container configuration
docker logs -f --tail 100 <container-name>      # Follow logs for a container
docker exec -it <container-name> sh                 # Open a shell inside a running container

# Network management
docker network ls                                   # List all Docker networks
docker network inspect ai-media-factory-backend     # Inspect a specific network
docker network prune                                # Remove unused networks

# Volume management
docker volume ls                                    # List all volumes
docker volume inspect ai-media-factory-pg_data      # Inspect a specific volume
docker volume prune                                 # Remove unused volumes (WARNING: destroys data)

# Image management
docker images                                       # List Docker images
docker pull <image>:<tag>                           # Pull a specific image
docker image prune -a                               # Remove unused images

# Swarm-specific (if using Docker Swarm)
docker node ls                                      # List swarm nodes
docker service ls                                   # List swarm services
docker stack deploy -c docker-compose.yml ai-media-factory  # Deploy as a stack
```

---

## Useful Compose Commands

```bash
# Start all services (base profile)
docker compose up -d

# Start all services including monitoring profile
docker compose --profile monitoring up -d

# Start a specific service and its dependencies
docker compose up -d n8n

# Stop all services (containers remain; volumes persist)
docker compose stop

# Stop and remove all containers, networks, and volumes (destroys data!)
docker compose down --volumes --remove-orphans

# Gracefully stop all containers (allows processes to finish)
docker compose stop --timeout 30

# Restart a specific service
docker compose restart n8n

# Recreate containers after configuration changes
docker compose up -d --no-deps n8n

# Rebuild an image and recreate its containers
docker compose build --no-cache n8n
docker compose up -d --force-recreate

# Validate the compose file for syntax and configuration errors
docker compose config
docker compose config --schema   # Validate against the Compose specification schema

# Display the resolved configuration (including defaults and computed values)
docker compose config --resolve-image-digests

# Pull all images used in the compose file
docker compose pull

# Run a command inside a container without starting it
docker compose run --rm postgres psql -U ${POSTGRES_USER} -d ${POSTGRES_DB}

# Scale a service (horizontal scaling)
docker compose up -d --scale n8n=3

# View compose project services in a tree format
docker compose ls

# Show the full configuration including environment variables and resolved values
docker compose config --export

# Watch for configuration changes and restart containers (compose watch)
docker compose watch

# Generate an events stream for the compose project
docker compose events
```

---

## Future Expansion

### Planned Enhancements

The AI Media Factory infrastructure is designed to accommodate the following future expansions without major re-architecture:

- **GPU Auto-Scaling** — Integration with NVIDIA GPU operator or cloud-provider GPU autoscalers to dynamically allocate GPU resources based on Ollama inference demand.
- **Multi-Node MinIO Erasure Coding** — Distributed MinIO across multiple hosts for higher durability and read throughput. Requires `MINIO_COMPUTE_NODES` > 1 and shared network storage between nodes.
- **Qdrant Distributed Clustering** — Enable `QDRANT_CLUSTER_ENABLED` and configure peer nodes for horizontal vector search scaling.
- **PostgreSQL Read Replicas** — Deploy read replicas of PostgreSQL for n8n read-heavy workloads (reporting, analytics). Changes the `DB_POSTGRESDB_HOST` to use a load balancer or read-write split proxy.
- **Redis Sentinel / Cluster** — Upgrade Redis to a clustered or sentinel configuration for high availability of the n8n Bull queue.
- **Vault Integration** — Replace `.env` file secrets with HashiCorp Vault dynamic secrets using the Docker secrets API or Consul Template.
- **CI/CD Pipeline** — GitHub Actions or GitLab CI pipeline for automated image building, vulnerability scanning, and canary deployments.
- **TLS Certificate Automation** — Replace manual TLS certificate management with Let's Encrypt and cert-manager (automatic renewal and deployment).
- **Prometheus AlertManager** — Add AlertManager to the monitoring stack for automated alert routing to PagerDuty, Slack, or email.
- **Loki Logging** — Replace Docker's json-file log driver with Loki for centralized log aggregation and querying via Grafana's log analytics panel.
- **Traefik Reverse Proxy** — Optional swap of Nginx for Traefik to enable automatic Let's Encrypt certificate provisioning and Docker-native configuration discovery.
- **Terraform Infrastructure** — Terraform modules for provisioning the cloud infrastructure (compute, disks, networks, firewall rules) required to host this stack.

---

## Versioning

This project follows [Semantic Versioning 2.0.0](https://semver.org/).

| Segment | Meaning | Change Criteria |
|---------|---------|----------------|
| **MAJOR** | `X.y.z` | Breaking changes to infrastructure, compose file structure, or network topology that require manual intervention during upgrade. |
| **MINOR** | `x.Y.z` | New services, profiles, or features added in a backward-compatible manner. Existing services continue to function without changes. |
| **PATCH** | `x.y.Z` | Bug fixes, configuration corrections, dependency updates (same major/minor version), and security patches. |

### Current Version

| Document | Version |
|----------|---------|
| docker-compose.yml | 2.0.0 |
| .env | 2.0.0 |
| README.md | 2.0.0 |

### Changelog Location

Breaking changes, minor feature additions, and patch-level fixes are tracked in the `CHANGELOG.md` file at the repository root (created separately).

### Version Bumping Process

1. Update the relevant file(s) with the new version.
2. Commit with a conventional commit message:
   - `feat(infra): add Prometheus monitoring profile` → MINOR bump
   - `fix(config): correct Qdrant snapshot path` → PATCH bump
   - `feat(infra): switch compose spec to v3.8` → MAJOR bump if migration required
3. Tag the release: `git tag docker-compose-v2.1.0`
4. Push the tag: `git push origin --tags`