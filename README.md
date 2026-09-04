# 🛒 Scalable E-Commerce Microservices

**A high-throughput, event-driven e-commerce engine built with Node.js and Docker.**

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Build Status](https://img.shields.io/badge/build-passing-brightgreen.svg)](#)
[![Version](https://img.shields.io/badge/version-1.4.0-orange.svg)](#)
[![Node](https://img.shields.io/badge/node-%3E%3D20.x-339933.svg)](#)
[![Docker](https://img.shields.io/badge/docker-ready-2496ED.svg)](#)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-ff69b4.svg)](#contributing)

> A decoupled, domain-driven backend for e-commerce workloads — designed to survive traffic spikes, avoid distributed monoliths, and stay debuggable in production.

---

## 📖 Table of Contents

- [Architecture](#-architecture--high-level-design)
- [Key Features](#-key-features--domain-modules)
- [Architectural & Security Decisions](#-architectural--security-decisions)
- [Tech Stack](#-tech-stack)
- [Local Setup & Quickstart](#-local-setup--quickstart)
- [Testing & QA](#-testing--qa)
- [Roadmap](#-development-roadmap)
- [Contributing](#-contributing)
- [License](#-license)

---

## 🏗 Architecture / High-Level Design

The system is composed of independently deployable services communicating over **REST (synchronous)** for client-facing queries and **RabbitMQ (asynchronous)** for cross-service side effects (inventory decrement, email receipts, analytics events). Each service owns its own data — there is no shared database.

```
                                   ┌────────────────────┐
                                   │   API Gateway /     │
                                   │   Nginx (Edge)       │
                                   └──────────┬──────────┘
                                              │
                     ┌────────────────────────┼────────────────────────┐
                     │                        │                        │
             ┌───────▼───────┐       ┌────────▼────────┐      ┌────────▼────────┐
             │  Auth Service  │       │  Catalog Service │      │  Cart Service    │
             │  (Node/Express)│       │  (Node/Express)  │      │  (Node/Express)  │
             └───────┬───────┘       └────────┬────────┘      └────────┬────────┘
                     │                        │                        │
             ┌───────▼───────┐       ┌────────▼────────┐      ┌────────▼────────┐
             │  PostgreSQL    │       │  PostgreSQL      │      │  Redis           │
             │  (users)       │       │  (products)      │      │  (cart sessions) │
             └────────────────┘       └──────────────────┘      └──────────────────┘

                     ┌─────────────────────────────────────────────────┐
                     │                RabbitMQ (Event Bus)               │
                     │  order.created · payment.confirmed · stock.low   │
                     └───────┬─────────────────────────────┬────────────┘
                             │                             │
                     ┌───────▼───────┐             ┌───────▼───────┐
                     │ Order Service  │             │ Notification   │
                     │ (Node/Express) │             │ Service        │
                     └───────┬───────┘             └───────┬───────┘
                             │                             │
                     ┌───────▼───────┐             ┌───────▼───────┐
                     │  PostgreSQL    │             │  SMTP / Push    │
                     │  (orders)      │             │  Provider       │
                     └────────────────┘             └────────────────┘

                     ┌─────────────────────────────────────────────────┐
                     │        Consul (Service Discovery & Health)        │
                     │        ELK Stack (Centralized Logging/Metrics)   │
                     └─────────────────────────────────────────────────┘
```

**Request flow example — placing an order:**

1. Client → API Gateway → `Order Service` (`POST /orders`)
2. `Order Service` validates cart contents against `Cart Service` (sync HTTP)
3. `Order Service` persists the order and publishes `order.created` to RabbitMQ
4. `Catalog Service` consumes `order.created` → decrements stock
5. `Notification Service` consumes `order.created` → sends confirmation email
6. All services emit structured logs to the ELK stack for tracing and alerting

---

## ✨ Key Features & Domain Modules

| Module | Description |
|---|---|
| 🔐 **Auth & Identity** | JWT-based auth with refresh token rotation, role-based access control (RBAC) |
| 📦 **Product Catalog** | Full-text search, faceted filtering, category trees, stock-aware pricing |
| 🛒 **Cart Service** | Redis-backed ephemeral carts with TTL, guest-to-user cart merging |
| 💳 **Checkout & Orders** | Idempotent order creation, saga-based payment orchestration |
| 🔔 **Notifications** | Async, queue-driven email/push notifications decoupled from the request path |
| 📊 **Observability** | Centralized structured logging, request tracing, and health dashboards |
| ⚙️ **Service Discovery** | Consul-based dynamic registration — no hardcoded service URLs |
| 🚦 **Rate Limiting** | Per-user and per-IP throttling at the gateway layer |

---

## 🧠 Architectural & Security Decisions

Notable, non-obvious engineering choices and the reasoning behind them:

- **UUIDv7 over auto-increment IDs.** All public-facing resource identifiers (orders, users, products) use UUIDv7 rather than sequential integers. This prevents [IDOR](https://owasp.org/www-project-top-ten/2017/A5_2017-Broken_Access_Control) enumeration attacks (`/orders/1043` → `/orders/1044`) while retaining timestamp-sortable ordering for efficient indexing, unlike random UUIDv4.
- **Event-driven decoupling over synchronous chaining.** Side effects that don't need to block the response (stock updates, emails, analytics) are published to RabbitMQ instead of being called via HTTP. This keeps the checkout critical path fast and prevents cascading failures — if the Notification Service is down, orders still succeed.
- **Cache-aside strategy for the Catalog Service.** Product reads hit Redis first; on a miss, Postgres is queried and the result is written back with a short TTL. Writes invalidate the specific cache key rather than flushing broad prefixes, minimizing thundering-herd risk.
- **No shared database across services.** Each service owns its schema exclusively, accessed only through its own API. This trades some query convenience for genuine deployability independence and blast-radius containment.
- **Idempotency keys on write endpoints.** `POST /orders` and `POST /payments` require an `Idempotency-Key` header, stored with a short TTL, so retried requests (client timeouts, network blips) never create duplicate orders or double-charge a customer.
- **Secrets never baked into images.** All credentials are injected via environment variables at runtime (see `.env.example`) and are never committed or baked into Docker layers.

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| **Runtime** | Node.js 20.x, Express |
| **Database** | PostgreSQL 16 (per-service schemas) |
| **Cache / Sessions** | Redis 7 |
| **Messaging / Events** | RabbitMQ |
| **Service Discovery** | Consul |
| **Monitoring / Logging** | ELK Stack (Elasticsearch, Logstash, Kibana) |
| **Containerization** | Docker, Docker Compose |
| **Auth** | JWT, bcrypt |
| **Testing** | Jest, Supertest, Testcontainers |
| **CI/CD** | GitHub Actions |

---

## 🚀 Local Setup & Quickstart

### Prerequisites

- Node.js `>= 20.x`
- Docker & Docker Compose
- `make` (optional, for shortcut commands)

### 1. Clone the repository

```bash
git clone https://github.com/your-org/scalable-ecommerce-microservices.git
cd scalable-ecommerce-microservices
```

### 2. Configure environment variables

```bash
cp .env.example .env
```

Edit `.env` and set the required values:

```env
# --- Core ---
NODE_ENV=development
PORT=3000

# --- Database ---
POSTGRES_USER=ecommerce
POSTGRES_PASSWORD=changeme
POSTGRES_DB=ecommerce_dev

# --- Cache ---
REDIS_URL=redis://redis:6379

# --- Messaging ---
RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672

# --- Auth ---
JWT_SECRET=replace-with-a-long-random-string
JWT_REFRESH_SECRET=replace-with-another-long-random-string

# --- Service Discovery ---
CONSUL_HOST=consul
CONSUL_PORT=8500
```

### 3. Start the stack

```bash
docker compose up --build
```

This spins up: `api-gateway`, `auth-service`, `catalog-service`, `cart-service`, `order-service`, `notification-service`, `postgres`, `redis`, `rabbitmq`, `consul`, and the `elk` stack.

### 4. Run database migrations

```bash
docker compose exec order-service npm run migrate
docker compose exec catalog-service npm run migrate
```

### 5. Verify it's alive

```bash
curl http://localhost:3000/health
# → {"status":"ok","services":["auth","catalog","cart","order","notification"]}
```

| Service | Local URL |
|---|---|
| API Gateway | http://localhost:3000 |
| Kibana (logs) | http://localhost:5601 |
| Consul UI | http://localhost:8500 |
| RabbitMQ Management | http://localhost:15672 |

---

## 🧪 Testing & QA

```bash
# Run all unit tests across every service
npm run test:unit

# Run integration tests (spins up Testcontainers for Postgres/Redis/RabbitMQ)
npm run test:integration

# Run end-to-end tests against a running docker compose stack
docker compose -f docker-compose.test.yml up -d
npm run test:e2e

# Full CI-equivalent run (lint + unit + integration)
npm run test:ci

# Generate a coverage report
npm run test:coverage
```

Coverage reports are output to `/coverage` per service and aggregated in CI via `codecov`.

---

## 🗺 Development Roadmap

### ✅ MVP (Shipped)

- [x] Auth service with JWT + refresh token rotation
- [x] Product catalog with search & filtering
- [x] Redis-backed cart with guest merging
- [x] Order creation with idempotency keys
- [x] Event-driven stock decrement via RabbitMQ
- [x] Centralized logging via ELK
- [x] Dockerized local development environment

### 🚧 In Progress

- [ ] Payment service with Stripe webhook reconciliation
- [ ] Saga-based distributed transaction rollback for failed payments
- [ ] Per-service rate limiting via Redis token buckets

### 🔭 Planned

- [ ] GraphQL gateway as an alternative to REST aggregation
- [ ] Kubernetes Helm charts for production deployment
- [ ] Multi-region read replicas for the Catalog Service
- [ ] Admin dashboard (React) for order & inventory management
- [ ] OpenTelemetry distributed tracing across all services

---

## 🤝 Contributing

Contributions are welcome! Please:

1. Fork the repo and create a feature branch (`git checkout -b feature/my-feature`)
2. Follow the existing code style (`npm run lint` before committing)
3. Write or update tests for any behavioral change
4. Open a PR with a clear description of the change and its motivation

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for full guidelines and the [Code of Conduct](CODE_OF_CONDUCT.md).

---

## 📄 License

Distributed under the **MIT License**. See [`LICENSE`](LICENSE) for details.

---

<p align="center">Built with ☕ and a healthy fear of distributed systems.</p>
