# CLAUDE.md - Jussi Project

## Project Overview

Jussi is a high-performance JSON-RPC 2.0 reverse proxy for the Hive blockchain ecosystem. It provides intelligent routing, multi-tier caching (memory + Redis), and request aggregation for blockchain API calls.

Key features:
- Routes JSON-RPC requests to appropriate upstream servers based on method namespaces
- Two-tier caching: in-memory LRU + Redis with configurable TTLs
- Blockchain-aware caching (irreversible block detection)
- Batch request support with concurrent upstream dispatch
- WebSocket connection pooling
- Account blacklisting and rate limiting

## Tech Stack

- **Language**: Python 3.6
- **Web Framework**: Sanic 0.8.3 (async)
- **Event Loop**: uvloop
- **Caching**: aredis (async Redis client), in-memory LRU
- **HTTP Client**: aiohttp
- **JSON**: ujson, python-rapidjson
- **Logging**: structlog (JSON output)
- **Testing**: pytest, pytest-asyncio, pytest-docker, pytest-sanic
- **Containerization**: Docker (phusion/baseimage), nginx, runit

## Directory Structure

```
jussi/                      # Main Python package
├── serve.py               # Entry point, CLI args, Sanic app setup
├── handlers.py            # Request handlers (jsonrpc, healthcheck)
├── upstream.py            # Upstream routing with trie-based config
├── urn.py                 # URN parsing for JSON-RPC methods
├── validators.py          # Request validation (limits, block ranges)
├── request/               # Request handling
│   ├── http.py           # HTTP request wrapper
│   └── jsonrpc.py        # JSON-RPC parsing and validation
├── middlewares/           # Request/response pipeline
│   ├── jussi.py          # Main processing middleware
│   ├── caching.py        # Cache read/write
│   ├── limits.py         # Rate limiting, blacklisting
│   └── statsd.py         # Metrics collection
├── cache/                 # Caching subsystem
│   ├── ttl.py            # TTL determination
│   ├── cache_group.py    # Multi-level cache orchestration
│   └── backends/         # Memory and Redis implementations
└── ws/                    # WebSocket connection pooling

tests/                      # Test suite
├── conftest.py            # Fixtures and configuration
├── test_*.py              # Unit and integration tests
└── data/                  # Test data (JSON-RPC samples, schemas)

service/                    # Deployment service definitions
├── jussi/                 # Jussi runit service
└── nginx/                 # Nginx reverse proxy config
```

## Development Commands

```bash
# Setup
pipenv install --dev
pipenv run pre-commit install

# Run locally
make run-local                    # Runs on port 9000
# Or directly:
pipenv run python3.6 -m jussi.serve --server_workers=1 --upstream_config_file DEV_config.json

# Testing
make test                         # Run all tests
make test-with-docker            # Tests requiring Docker services
pipenv run pytest -vv tests/     # Direct pytest

# Code quality
make lint                         # pylint
make fmt                          # autopep8 formatting
make fix-imports                  # Remove unused + sort imports
make pre-commit                   # Run pre-commit hooks
make mypy                         # Type checking

# Combined workflows
make prepare                      # fix-imports + lint + fmt + pre-commit + test
make prepare-without-test         # Same without tests

# Docker
make build                        # Build Docker image (runs full test suite)
make run                          # Run container (ports 8080, 7777)
```

## Key Files

| File | Purpose |
|------|---------|
| `jussi/serve.py` | Entry point, CLI parsing, Sanic app initialization |
| `jussi/handlers.py` | JSON-RPC routing, healthcheck endpoints |
| `jussi/upstream.py` | Upstream config, trie-based URL/TTL/timeout lookups |
| `jussi/urn.py` | Parses JSON-RPC methods into namespace/api/method URNs |
| `DEV_config.json` | Development upstream configuration |
| `upstreams_schema.json` | JSON Schema for config validation |
| `Dockerfile` | Multi-stage build with test suite |
| `Makefile` | 30+ build/test/quality targets |

## Configuration

**Environment Variables:**
- `JUSSI_UPSTREAM_CONFIG_FILE` - Upstream config JSON path
- `JUSSI_SERVER_PORT` - Listen port (default: 9000)
- `JUSSI_SERVER_WORKERS` - Worker count (default: CPU count)
- `JUSSI_REDIS_URL` - Redis connection (redis://host:port)
- `JUSSI_JSONRPC_BATCH_SIZE_LIMIT` - Max batch size (default: 50)
- `JUSSI_STATSD_URL` - StatsD endpoint
- `LOG_LEVEL` - Logging level (INFO, WARNING, etc.)

**TTL Values:**
- `0` - Cache indefinitely
- `-1` - Don't cache
- `-2` - Cache indefinitely if block is irreversible
- Positive - Cache for N seconds

## Coding Conventions

- **Style**: PEP 8 with 100-char line limit
- **Formatting**: autopep8 (`make fmt`)
- **Imports**: Single-line, sorted with isort
- **Linting**: pylint (see `.pylintrc`)
- **Async**: All I/O uses async/await
- **Type hints**: Used throughout
- **Logging**: structlog with JSON output

**Module pattern:**
```python
# -*- coding: utf-8 -*-
[imports]
logger = structlog.get_logger(__name__)
[functions/classes]
```

## CI/CD Notes

**GitHub Actions** (`.github/workflows/codeql-analysis.yml`):
- Runs on push to `develop` branch
- CodeQL security analysis for Python 3.6 and 3.8

**CircleCI** (`circle.yml`):
- Builds Docker image with full test suite

**Docker Build:**
- Base: phusion/baseimage:0.9.19
- Installs Python 3.6.5, nginx, runit
- Runs `pipenv install --dev` and test suite during build
- Exposes: 8080 (nginx), 9000 (jussi), 7777 (monitoring)

**Pre-commit Hooks:**
- AST checking, trailing whitespace, JSON/YAML validation
- autopep8, tab detection, bash syntax checking

## Monitoring Endpoints

- `/health` - Basic healthcheck (GET/HEAD)
- `/.well-known/healthcheck.json` - Standard healthcheck
- Port 7777 - Monitoring interface

## Testing

```bash
# Unit tests
pipenv run pytest tests/test_urn.py tests/test_validators.py

# Integration tests (requires Docker)
pipenv run pytest --rundocker tests/

# Live tests against production
pipenv run pytest -vv tests/test_responses.py --jussiurl http://localhost:9000
```

Key test areas: URN parsing, upstream routing, caching, validators, error handling, full request/response flows.
