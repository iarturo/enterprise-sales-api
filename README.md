<p align="center">
  <img src="https://img.shields.io/badge/n8n-Workflow-FF6D5A?style=for-the-badge&logo=n8n&logoColor=white" alt="n8n" />
  <img src="https://img.shields.io/badge/Supabase-Auth-3ECF8E?style=for-the-badge&logo=supabase&logoColor=white" alt="Supabase" />
  <img src="https://img.shields.io/badge/PostgreSQL-Database-4169E1?style=for-the-badge&logo=postgresql&logoColor=white" alt="PostgreSQL" />
  <img src="https://img.shields.io/badge/Redis-Cache-DC382D?style=for-the-badge&logo=redis&logoColor=white" alt="Redis" />
  <img src="https://img.shields.io/badge/Status-Reference%20Implementation-blue?style=for-the-badge" alt="Status" />
  <img src="https://img.shields.io/badge/License-MIT-green?style=for-the-badge" alt="License" />
</p>

<h1 align="center">🏢 Enterprise Sales — n8n Backend Reference</h1>

<p align="center">
  <strong>A reference implementation of an enterprise-style Sales Management API, built entirely on n8n workflows.</strong>
  <br />
  Demonstrates production patterns: authentication, rate limiting, idempotency, atomic transactions, and observability — without writing a single line of backend server code.
</p>

<p align="center">
  <a href="#-about-this-project">About</a> •
  <a href="#-quick-start">Quick Start</a> •
  <a href="#-architecture">Architecture</a> •
  <a href="#-api-reference">API Reference</a> •
  <a href="#-security-patterns">Security Patterns</a> •
  <a href="#-deployment">Deployment</a>
</p>

---

## 📖 About This Project

This repository is a **portfolio piece** demonstrating how far n8n can be pushed as a backend platform. It is a fully functional reference implementation — every endpoint works, every validation runs, every workflow is wired end-to-end — but it has not been deployed to serve real users in production.

It exists to showcase a specific set of skills:

- Designing REST APIs with proper auth, rate limiting, and idempotency
- Modeling atomic database transactions for inventory + sales flows
- Composing observability and operational tooling (health checks, logging, backups)
- Architecting layered security (defense in depth)
- Documenting an API to a level where a third party could integrate against it

If you are evaluating this repo for hiring or collaboration purposes, the code, workflow, and documentation here represent how I approach backend design problems. Happy to walk through any design decision in detail.

---

## 🎯 Overview

**Enterprise Sales** transforms [n8n](https://n8n.io) from a simple automation tool into a structured REST API backend. It exposes webhook endpoints that handle the full lifecycle of a sales operation — from user authentication to invoice generation — while enforcing rate limiting, idempotency, input sanitization, and atomic database transactions.

### What this project demonstrates

| Conventional Approach | This Reference Implementation |
|---|---|
| Months building a custom Express/FastAPI backend | A single n8n workflow handles HTTP routing & orchestration |
| Writing auth middleware from scratch | Supabase Auth with JWT validation, wired through n8n nodes |
| Manual rate limiting implementation | Redis-backed rate limiting per IP + User-Agent |
| Custom idempotency layer | Redis `SETNX` with 24h TTL |
| Boilerplate transaction handling | PostgreSQL stored function for atomic sale + inventory update |

---

## ✨ Features Implemented

### 🔐 Authentication & Authorization
- **Supabase Auth** integration with JWT-based access & refresh tokens
- Role-based access control (`admin`, `seller`, custom roles)
- Multi-tenant support via `company_id` scoping
- CORS configuration with environment-driven allowed origins

### 🛡️ Security Patterns
- **Redis-powered rate limiting** — 5 requests/minute per IP + User-Agent fingerprint
- **Idempotency keys** — prevent duplicate sale creation with Redis `SETNX` + 24h TTL
- **Input sanitization** — HTML/SQL injection protection on all user-supplied strings
- **Parameterized SQL** — zero raw string interpolation in database queries
- **Request validation** — strict type checking, range enforcement, format validation
- **HTTP timeouts** — 10s timeouts on all external service calls

### 💰 Sales Processing
- Atomic sale creation with PostgreSQL stored function (`create_sale_with_inventory_update`)
- Real-time inventory verification with row-level locking (`SELECT ... FOR UPDATE`)
- Race condition detection with HTTP 409 responses
- Automatic tax calculation with configurable tax rates
- Auto-generated invoice numbers (`F-YYYY-XXXXXX-XXX`)
- Validation for up to 100 line items per sale, quantities up to 10,000 units

### 📊 Reporting & Export
- Paginated sales queries with date range filtering (up to 10,000 records)
- Excel/PDF export via **CloudConvert** API integration
- Async report generation with execution job tracking

### 🔄 Operational Patterns
- **Health check endpoint** — monitors PostgreSQL + Redis connectivity
- **Scheduled backup workflow** — `pg_dump` on cron, with audit logging to `backup_logs` table
- **Centralized error logging** — failures forwarded to **Datadog** for observability

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CLIENT APPLICATION                          │
│                    (Web App / Mobile / cURL)                        │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ HTTPS
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         n8n WORKFLOW ENGINE                         │
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐  ┌──────────────┐ │
│  │  /api/auth   │  │  /api/sales  │  │ /api/    │  │  /api/health │ │
│  │   /login     │  │  (POST)      │  │ reports/ │  │  (GET)       │ │
│  │  (POST)      │  │              │  │  sales   │  │              │ │
│  └──────┬───────┘  └──────┬───────┘  └────┬─────┘  └──────┬───────┘ │
│         │                 │               │               │         │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌────▼─────┐  ┌──────▼───────┐│
│  │   Validate   │  │  Validate    │  │ Validate │  │    Check     ││
│  │    Input     │  │   Token      │  │  Token   │  │  Services    ││
│  └──────┬───────┘  └──────┬───────┘  └────┬─────┘  └──────────────┘│
│         │                 │               │                         │
│  ┌──────▼───────┐  ┌──────▼───────┐       │                        │
│  │ Rate Limit   │  │ Idempotency  │       │                        │
│  │  (Redis)     │  │   (Redis)    │       │                        │
│  └──────┬───────┘  └──────┬───────┘       │                        │
│         │                 │               │                        │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌────▼─────┐                 │
│  │  Supabase    │  │  Inventory   │  │  Query   │                 │
│  │    Auth      │  │ Verification │  │  + Export │                 │
│  └──────────────┘  └──────┬───────┘  └──────────┘                 │
│                           │                                        │
│                    ┌──────▼───────┐                                │
│                    │   Atomic     │                                │
│                    │ Transaction  │                                │
│                    │ (PL/pgSQL)   │                                │
│                    └──────────────┘                                │
└──────────────────────────────┬─────────────────────────────────────┘
                               │
            ┌──────────────────┼──────────────────┐
            │                  │                  │
      ┌─────▼──────┐   ┌──────▼──────┐   ┌───────▼──────┐
      │ PostgreSQL  │   │    Redis    │   │   Supabase   │
      │ (Data)      │   │ (Cache/     │   │   (Auth)     │
      │             │   │  Rate Limit)│   │              │
      └─────────────┘   └─────────────┘   └──────────────┘
```

### Workflow Pipelines

| Pipeline | Trigger | Key Nodes | Purpose |
|---|---|---|---|
| **Auth Login** | `POST /api/auth/login` | Validate Input → Redis Rate Limit → Supabase Auth → Process Auth | Authenticate users, return JWT |
| **Create Sale** | `POST /api/sales` | Validate Token → Supabase Verify → Redis Idempotency → Verify Inventory → Process Sale → Atomic Transaction | Create a new sale with inventory update |
| **Sales Report** | `GET /api/reports/sales` | Validate Token → Supabase Verify → PostgreSQL Query → CloudConvert Export | Generate & export sales reports |
| **Health Check** | `GET /api/health` | PostgreSQL Ping → Redis Ping → Aggregate Status | Monitor infrastructure health |
| **Scheduled Backup** | Cron `0 2 * * *` | Execute pg_dump → Log Backup | Database backup workflow |

---

## 🚀 Quick Start

### Prerequisites

| Service | Minimum Version | Purpose |
|---|---|---|
| [n8n](https://n8n.io) | v1.0+ | Workflow engine |
| [PostgreSQL](https://www.postgresql.org/) | 14+ | Primary data store |
| [Redis](https://redis.io/) | 6+ | Rate limiting & idempotency cache |
| [Supabase](https://supabase.com/) | Free tier | Authentication provider |

### 1. Clone the Repository

```bash
git clone https://github.com/iarturo/enterprise-sales-api.git
cd enterprise-sales-api
```

### 2. Set Up the Database

```sql
-- Create the database
CREATE DATABASE ventas_enterprise;
CREATE USER ventas_app WITH PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE ventas_enterprise TO ventas_app;
```

Then run the stored function required for atomic sale transactions:

```sql
-- File: Function.txt
-- Creates the `create_sale_with_inventory_update` PL/pgSQL function
-- Handles inventory locking, stock validation, and sale insertion atomically
\i Function.txt
```

### 3. Configure Environment Variables

Set the following environment variables in your n8n instance:

```bash
# ── Supabase ──────────────────────────────────────
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=your-anon-key

# ── Database ──────────────────────────────────────
DB_HOST=localhost
DB_USER=postgres
DB_NAME=ventas_enterprise
DB_PASSWORD=your-db-password

# ── Monitoring ────────────────────────────────────
DATADOG_API_KEY=your-datadog-api-key

# ── Business Logic ────────────────────────────────
TAX_RATE=0.16
PAYMENT_METHODS=cash,card,transfer,check
FRONTEND_URL=https://app.yourdomain.com

# ── Reporting (Optional) ─────────────────────────
CLOUDCONVERT_API_KEY=your-cloudconvert-key
```

### 4. Import the Workflow

1. Open your n8n instance
2. Click **Import from file**
3. Select `enterprise-sales.json`
4. Assign credentials to each node group:
   - **PostgreSQL nodes** → your PostgreSQL credential
   - **Redis nodes** → your Redis credential
   - **Supabase HTTP nodes** → Header Auth credential
   - **CloudConvert nodes** → Header Auth credential
   - **Datadog nodes** → Header Auth credential
5. Toggle the workflow to **Active** ✅

### 5. Create Your First User

```sql
-- In the Supabase SQL Editor:
INSERT INTO auth.users (email, encrypted_password, email_confirmed_at, created_at, updated_at)
VALUES ('admin@yourcompany.com', crypt('strong_password_here', gen_salt('bf')), NOW(), NOW(), NOW());

UPDATE auth.users
SET raw_user_meta_data = '{"role": "admin", "company_id": 1, "profile": {"name": "Admin User"}}'
WHERE email = 'admin@yourcompany.com';
```

---

## 📡 API Reference

### Authentication

#### `POST /api/auth/login`

Authenticate a user and receive JWT tokens.

**Request:**
```bash
curl -X POST https://your-n8n.com/webhook/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@yourcompany.com",
    "password": "strong_password_here"
  }'
```

**Response `200 OK`:**
```json
{
  "success": true,
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "admin@yourcompany.com",
    "role": "admin",
    "company_id": 1,
    "profile": { "name": "Admin User" }
  },
  "tokens": {
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "v1.MjQ1NjM4..."
  },
  "expiresIn": 3600
}
```

**Error Responses:**

| Status | Code | Description |
|---|---|---|
| `400` | Bad Request | Missing or invalid email/password format |
| `401` | Unauthorized | Invalid credentials |
| `429` | Too Many Requests | Rate limit exceeded (5 req/min) |

---

### Sales

#### `POST /api/sales`

Create a new sale with automatic inventory management.

**Headers:**
| Header | Required | Description |
|---|---|---|
| `Authorization` | ✅ | `Bearer <access_token>` |
| `Idempotency-Key` | ✅ | Unique UUID to prevent duplicate submissions |
| `Content-Type` | ✅ | `application/json` |

**Request:**
```bash
curl -X POST https://your-n8n.com/webhook/api/sales \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -H "Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000" \
  -d '{
    "customer_info": {
      "name": "Jane Doe",
      "email": "jane@example.com",
      "phone": "+1-555-123-4567"
    },
    "items": [
      { "product_id": 1, "quantity": 2, "unit_price": 49.99 },
      { "product_id": 5, "quantity": 1, "unit_price": 129.00 }
    ],
    "payment_method": "card"
  }'
```

**Response `200 OK`:**
```json
{
  "success": true,
  "invoiceNumber": "F-2026-847293-042",
  "totalAmount": 228.98,
  "message": "Sale registered successfully"
}
```

**Validation Constraints:**

| Field | Rule |
|---|---|
| `items` | 1–100 items per sale |
| `items[].quantity` | 1–10,000 units |
| `items[].unit_price` | $0.01–$1,000,000 |
| `customer_info.name` | Required, max 255 chars, sanitized |
| `customer_info.email` | Optional, validated format |
| `payment_method` | One of: `cash`, `card`, `transfer`, `check` (configurable) |

**Error Responses:**

| Status | Code | Description |
|---|---|---|
| `400` | Bad Request | Validation error or insufficient inventory |
| `401` | Unauthorized | Missing or invalid token |
| `409` | Conflict | Idempotency key already processed / Race condition detected |

---

### Reports

#### `GET /api/reports/sales`

Retrieve paginated sales data with optional date filtering.

**Query Parameters:**

| Parameter | Default | Description |
|---|---|---|
| `company_id` | `1` | Company scope |
| `start_date` | 30 days ago | ISO date `YYYY-MM-DD` |
| `end_date` | Today | ISO date `YYYY-MM-DD` |
| `limit` | `1000` | Max `10000` |
| `offset` | `0` | Pagination offset |

**Request:**
```bash
curl "https://your-n8n.com/webhook/api/reports/sales?start_date=2026-01-01&end_date=2026-05-21&limit=50" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

---

### Health

#### `GET /api/health`

Check the status of all infrastructure dependencies.

**Request:**
```bash
curl https://your-n8n.com/webhook/api/health
```

**Response `200 OK`:**
```json
{
  "status": "healthy",
  "timestamp": "2026-05-21T18:43:00.000Z",
  "services": {
    "database": { "status": "healthy" },
    "redis": { "status": "healthy" }
  }
}
```

---

## 🔒 Security Patterns

This implementation applies defense-in-depth across multiple layers:

```
Request ──▶ Rate Limiting ──▶ Input Validation ──▶ JWT Verification ──▶ Idempotency ──▶ Business Logic
              (Redis)          (Regex + Type)       (Supabase)           (Redis SETNX)    (Parameterized SQL)
```

| Layer | Mechanism | Details |
|---|---|---|
| **Rate Limiting** | Redis INCR + EXPIRE | 5 req/min per IP + User-Agent fingerprint |
| **Input Validation** | Regex + type assertions | Email format, numeric ranges, string sanitization |
| **Authentication** | Supabase Auth | JWT token validation via Supabase `/auth/v1/user` |
| **Idempotency** | Redis SETNX | Atomic set-if-not-exists with 24h TTL |
| **SQL Injection** | Parameterized queries | All queries use `$1, $2, ...` placeholders |
| **XSS Prevention** | Input sanitization | Strip `< > ' " \` from user-supplied strings |
| **Race Conditions** | `SELECT ... FOR UPDATE` | Row-level locks during inventory checks |
| **Timeouts** | 10s on all HTTP calls | Prevents hanging connections |
| **Error Logging** | Datadog integration | All errors forwarded to centralized logging |
| **CORS** | Environment-driven | `Access-Control-Allow-Origin` set via `FRONTEND_URL` |

---

## 🚢 Deployment

If you wanted to take this from reference implementation to actual production, here is how I'd approach scaling it.

### Option 1 — Lightweight VPS (small teams)

| Component | Service | Cost |
|---|---|---|
| Compute | DigitalOcean Droplet | ~$6/mo |
| Database | Supabase (PostgreSQL + Auth) | Free |
| Cache | Redis (self-hosted) | Included |
| Workflow | n8n (self-hosted) | Free |

### Option 2 — Fully Managed

| Component | Service | Cost |
|---|---|---|
| Auth + Database | Supabase Pro | ~$25/mo |
| Cache | Redis Cloud | Free–$7/mo |
| Workflow | n8n Cloud | ~$20/mo |

### Option 3 — Kubernetes (Enterprise scale)

For 1,000+ concurrent users. PostgreSQL, Redis, and n8n would deploy as separate services behind an ingress controller with horizontal pod autoscaling.

### Scaling Considerations

| Users | Auth | Cache | Workflow |
|---|---|---|---|
| 10–100 | Supabase Free | Redis single | n8n single |
| 100–1,000 | Supabase Pro | Redis cluster | n8n + load balancer |
| 1,000+ | Supabase Enterprise | Redis Enterprise | n8n on Kubernetes |

---

## 📁 Project Structure

```
enterprise-sales-api/
├── enterprise-sales.json              # n8n workflow export
├── Function.txt                       # PL/pgSQL stored function
├── GUIA_CONFIGURACION_FUNCIONAL.md    # Setup guide (Spanish)
├── LICENSE                            # MIT License
└── README.md                          # ← You are here
```

---

## 🗺️ Possible Extensions

If I were to take this further, the natural next steps would be:

- [ ] Two-factor authentication (TOTP via Supabase)
- [ ] Webhook notifications on sale creation (Slack / WhatsApp Business)
- [ ] PDF invoice generation with branded templates
- [ ] Multi-currency support
- [ ] Advanced analytics dashboard endpoint
- [ ] OpenAPI / Swagger specification
- [ ] Automated integration test suite

---

## 💬 Feedback Welcome

This is a personal portfolio project. If you're reviewing it for hiring, collaboration, or just curiosity, I'm happy to walk through any architectural decision, trade-off, or extension idea. Open an issue or reach out directly.

📧 arturo66@gmail.com
🔗 [LinkedIn](https://www.linkedin.com/in/arturo-ortega-90761a406/)

---

## 📄 License

Released under the **MIT License**. See [LICENSE](LICENSE) for details.

Built by **Arturo Ortega Salinas** — Mexico City.
