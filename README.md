# nginx-api-gateway

Nginx reverse proxy / API gateway for two Python backends (`user-backend`, `order-backend`), orchestrated with Docker Compose.

## Architecture Overview

The stack runs via Docker Compose and consists of an Nginx reverse proxy sitting in front of two lightweight Python HTTP backend services:

* **Nginx Gateway (`nginx:stable-alpine-slim`)**: Listens on ports `80` (HTTP) and `443` (HTTPS). Redirects all HTTP traffic to HTTPS, terminates TLS, and routes API requests to upstream services.
* **Users Service (`users-app-1`, `users-app-2`)**: Two identical Python 3.14 backend instances listening internally on port `8080`, load-balanced by Nginx under `/api/users/`.
* **Orders Service (`orders-app`)**: A single Python 3.14 backend instance listening internally on port `8080`, routed under `/api/orders/`.

### Key Gateway Features

* **TLS Termination & HSTS**: Enforces HTTPS (`301` redirect from port `80`), uses TLSv1.2/TLSv1.3 with strong ciphers, and sets `Strict-Transport-Security` headers.
* **Load Balancing & Passive Health Checks**: Distributes `/api/users/` requests across `users-app-1` and `users-app-2` (`max_fails=2`, `fail_timeout=10s`).
* **Proxy Caching**: Caches `200 OK` responses from `/api/users/` for 1 minute (`100MB` max cache size) and exposes cache status via the `X-Cache-Status` response header (`HIT`, `MISS`, `EXPIRED`).
* **Rate Limiting**: Protects `/api/orders/` with a `1r/s` limit per client IP (`burst=5 nodelay`), returning HTTP `429 Too Many Requests` when exceeded.
* **Extended Upstream Logging**: Defines a `main_ext` log format capturing `$request_time`, `$upstream_connect_time`, `$upstream_header_time`, and `$upstream_response_time`.

---

## Project Structure

```text
.
├── certs/
│   ├── nginx-selfsigned.crt     # Self-signed TLS certificate (gitignored)
│   └── nginx-selfsigned.key     # Private key (gitignored)
├── nginx/
│   ├── cache/                   # Mounted volume for Nginx proxy cache
│   ├── conf.d/
│   │   ├── 00-log-format.conf   # Custom upstream timing log format
│   │   └── api-gateway.conf     # Server blocks, upstreams, caching & rate limits
│   └── snippets/
│       ├── proxy-headers.conf   # Standard X-Forwarded-* & Host headers
│       └── tls-params.conf      # TLS protocols, ciphers & HSTS settings
├── order-backend/
│   ├── Dockerfile               # Python 3.14 Alpine HTTP server
│   └── index.html               # Mock order service response
├── user-backend/
│   ├── Dockerfile               # Python 3.14 Alpine HTTP server
│   └── index.html               # Mock user service response
├── compose.yaml                 # Multi-container orchestration
└── prep-certs.sh                # Script to generate self-signed TLS certs
```

---

## Prerequisites

* [Docker](https://docs.docker.com/get-docker/) & [Docker Compose](https://docs.docker.com/compose/)
* `openssl` (for generating local self-signed certificates)
* `curl` (for testing endpoints)

---

## Getting Started

### 1. Generate Self-Signed Certificates

Because `certs/` is excluded from version control, generate a local RSA 2048-bit certificate and private key before starting the stack:

```bash
chmod +x prep-certs.sh
./prep-certs.sh

```

### 2. Start the Services

Build and launch the containers in detached mode:

```bash
docker compose up -d --build

```

### 3. Stop the Services

```bash
docker compose down

```

---

## Testing the Endpoints

Use `curl -k` (or `--insecure`) to bypass self-signed certificate warnings when testing locally.

### HTTP to HTTPS Redirect

```bash
curl -I http://localhost/api/users/
# Expected: HTTP/1.1 301 Moved Permanently -> Location: https://localhost/api/users/

```

### Users Endpoint (Load Balancing & Caching)

Send two consecutive requests to observe caching behavior via the `x-cache-status` header:

```bash
# First request (Cache MISS)
curl -ik https://localhost/api/users/

# Second request within 60 seconds (Cache HIT)
curl -ik https://localhost/api/users/

```

### Orders Endpoint (Rate Limiting)

Send a rapid burst of requests to trigger the `1r/s` (`burst=5`) rate limit:

```bash
for i in {1..10}; do
  curl -sk -o /dev/null -w "Request $i: %{http_code}\n" https://localhost/api/orders/
done
# Expected: First 6 requests return 200, subsequent rapid requests return 429

```
