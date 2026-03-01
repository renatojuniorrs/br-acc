# Backend — How It Works

Language: **English** | [Português (Brasil)](pt-BR/backend.md)

The BR/ACC backend is a [FastAPI](https://fastapi.tiangolo.com/) application written in Python 3.12+.  
It exposes a REST API backed by a [Neo4j 5](https://neo4j.com/) graph database and runs as an async ASGI service.

## Directory Layout

```
api/
├── src/bracc/
│   ├── main.py          # Application factory and middleware wiring
│   ├── config.py        # Settings loaded from environment variables
│   ├── constants.py     # Shared domain constants (e.g. PEP roles)
│   ├── dependencies.py  # FastAPI dependency injection (DB driver, auth)
│   ├── middleware/      # Security headers, rate limiting, CPF masking
│   ├── models/          # Pydantic request/response schemas
│   ├── queries/         # Named Cypher query files (.cypher)
│   ├── routers/         # Route handlers grouped by domain
│   ├── services/        # Business logic and Neo4j helpers
│   ├── i18n/            # Internationalisation strings
│   └── templates/       # Jinja2 templates (PDF export)
├── tests/               # Pytest test suite
├── pyproject.toml       # Dependencies and tool configuration
└── Dockerfile           # Production container image
```

## Application Startup

`main.py` creates the FastAPI application and wires everything together:

1. **Lifespan hook** — on startup, a Neo4j async driver is created via `init_driver()` and stored on `app.state`.  
   `ensure_schema()` runs `schema_init.cypher` so all indexes and constraints exist before the first request.
2. **Middleware stack** (outermost → innermost):
   | Middleware | Purpose |
   |---|---|
   | `SlowAPIMiddleware` | Enforces per-client rate limits |
   | `CORSMiddleware` | Sets allowed origins from `CORS_ORIGINS` env var |
   | `SecurityHeadersMiddleware` | Adds `X-Frame-Options`, `CSP`, `HSTS` (prod), etc. |
   | `CPFMaskingMiddleware` | Masks Brazilian CPF tax IDs in JSON responses |
3. **Routers** are registered for each domain (see [Routers](#routers) below).

## Configuration

All settings are loaded by `config.py` using [pydantic-settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/).  
Environment variables map 1-to-1 to setting names (no prefix required).  
Key settings:

| Variable | Default | Description |
|---|---|---|
| `NEO4J_URI` | `bolt://localhost:7687` | Neo4j connection URI |
| `NEO4J_USER` | `neo4j` | Database user |
| `NEO4J_PASSWORD` | `changeme` | Database password — **must be changed** |
| `JWT_SECRET_KEY` | `change-me-in-production` | HMAC secret for JWT tokens — **must be ≥ 32 chars** |
| `JWT_EXPIRE_MINUTES` | `1440` | Token lifetime (24 h) |
| `RATE_LIMIT_ANON` | `60/minute` | Default rate limit for unauthenticated requests |
| `RATE_LIMIT_AUTH` | `300/minute` | Rate limit for authenticated requests |
| `PRODUCT_TIER` | `community` | Feature tier |
| `PUBLIC_MODE` | `false` | Enables public-safe API restrictions |
| `PUBLIC_ALLOW_PERSON` | `false` | Allow person-level entity lookups in public mode |
| `PUBLIC_ALLOW_ENTITY_LOOKUP` | `false` | Allow entity-lookup endpoints in public mode |
| `PUBLIC_ALLOW_INVESTIGATIONS` | `false` | Allow investigation endpoints in public mode |
| `PATTERNS_ENABLED` | `false` | Enable pattern engine |
| `CORS_ORIGINS` | `http://localhost:3000` | Comma-separated allowed origins |

## Database Access

All database interactions go through `services/neo4j_service.py`:

- **`CypherLoader`** reads `.cypher` files from `queries/` and caches their text in memory.
- **`execute_query(session, query_name, params)`** runs a named query and returns all records.
- **`execute_query_single(session, query_name, params)`** returns a single record (or `None`).
- **`sanitize_props(props)`** converts Neo4j temporal types, lists, and dicts into JSON-safe scalars.
- **`ensure_schema(driver)`** runs `schema_init.cypher` at startup to create constraints and indexes idempotently.

No raw Cypher strings are embedded in Python code; every query lives in a dedicated `.cypher` file.

## Routers

| Router module | Prefix | Auth required | Description |
|---|---|---|---|
| `meta.py` | `/api/v1/meta` | No | Database health check, aggregate stats, source registry |
| `public.py` | `/api/v1/public` | No | Public company subgraph, patterns (503 when disabled), meta |
| `auth.py` | `/api/v1/auth` | No (register/login) | User registration (invite-gated), login, `/me` |
| `entity.py` | `/api/v1/entity` | Yes | Entity lookup by CPF/CNPJ, timeline, connections, exposure |
| `search.py` | `/api/v1` | No | Full-text entity search (`/search?q=…`) |
| `graph.py` | `/api/v1/graph` | Yes | Ego-graph expansion with APOC label filtering |
| `patterns.py` | `/api/v1/patterns` | Yes | Pattern engine endpoints (disabled by default) |
| `baseline.py` | `/api/v1/baseline` | Yes | Regional and sector baseline statistics |
| `investigation.py` | `/api/v1/investigations` | Yes | CRUD for investigations, annotations, tags; PDF/JSON export; share links |

A single unauthenticated `GET /health` endpoint exists directly on the root app for container health checks.

## Authentication

The `auth` service (`services/auth_service.py`) implements email + password authentication:

1. **Registration** — passwords are hashed with `bcrypt`. An optional invite code is validated against `INVITE_CODE` env var. The user node is stored in Neo4j via `user_create.cypher`.
2. **Login** — credentials are verified with `bcrypt.checkpw`. A signed JWT (`HS256`) is returned.
3. **Token validation** — `decode_access_token()` verifies the JWT signature and expiry. `get_current_user()` (injected via `CurrentUser` dependency) loads the user from Neo4j on every authenticated request.
4. **Rate limiting** — login and register are capped at 10 requests/minute per IP regardless of global limits.

## Middleware Details

### CPF Masking (`middleware/cpf_masking.py`)

After a response is generated, the middleware reads the full JSON body and replaces CPF patterns:

- Formatted CPF (`123.456.789-00`) → `***.***.789-00`
- Raw 11-digit CPF (`12345678900`) → `*******8900`

CPFs belonging to **Politically Exposed Persons (PEPs)** are left unmasked.  
The middleware walks the JSON tree to collect PEP CPFs (identified by `is_pep: true` or political keywords in `role`/`cargo` fields) before applying masks.

### Rate Limiting (`middleware/rate_limit.py`)

[SlowAPI](https://slowapi.readthedocs.io/) (a Starlette wrapper for limits/rate-limit) is used.  
The key function prefers the authenticated user ID (extracted from the `Authorization: Bearer …` header) over the client IP address, so authenticated users get a higher quota.

### Security Headers (`middleware/security_headers.py`)

Every HTTP response receives:

- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `Referrer-Policy: no-referrer`
- `Permissions-Policy` — disables browser APIs (camera, geolocation, etc.)
- `Content-Security-Policy` — strict `default-src 'none'` for API paths
- `Strict-Transport-Security` (production + HTTPS only)

## Public Mode and Access Control

When `PUBLIC_MODE=true`, `services/public_guard.py` applies tiered access controls:

| Guard function | What it enforces |
|---|---|
| `enforce_entity_lookup_enabled()` | Blocks `/entity/*` unless `PUBLIC_ALLOW_ENTITY_LOOKUP=true` |
| `enforce_entity_lookup_policy(identifier)` | Blocks CPF lookups unless `PUBLIC_ALLOW_PERSON=true` |
| `enforce_person_access_policy(labels)` | Returns 403 if a node has Person/Partner labels and person access is disabled |
| `ensure_investigations_enabled()` | Blocks `/investigations/*` unless `PUBLIC_ALLOW_INVESTIGATIONS=true` |
| `sanitize_public_properties(props)` | Strips CPF and sensitive fields from node properties in public mode |
| `infer_exposure_tier(labels)` | Returns `public_safe`, `restricted`, or `internal_only` |

## Running Locally

```bash
cd api
uv sync --dev

# start Neo4j first (see docker-compose in infra/)
uvicorn bracc.main:app --reload
```

- API: `http://localhost:8000`
- Interactive docs: `http://localhost:8000/docs`

## Testing

```bash
cd api
uv run pytest
```

The test suite uses `pytest-asyncio`. Tests marked `integration` require a running Neo4j instance and are excluded by default (`-m 'not integration'`).
