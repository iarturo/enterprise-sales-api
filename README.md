<p align="center">
  <img src="https://img.shields.io/badge/n8n-Workflow-FF6D5A?style=for-the-badge&logo=n8n&logoColor=white" alt="n8n" />
  <img src="https://img.shields.io/badge/Supabase-Auth-3ECF8E?style=for-the-badge&logo=supabase&logoColor=white" alt="Supabase" />
  <img src="https://img.shields.io/badge/PostgreSQL-Database-4169E1?style=for-the-badge&logo=postgresql&logoColor=white" alt="PostgreSQL" />
  <img src="https://img.shields.io/badge/Redis-Cache-DC382D?style=for-the-badge&logo=redis&logoColor=white" alt="Redis" />
  <img src="https://img.shields.io/badge/Datadog-Monitoring-632CA6?style=for-the-badge&logo=datadog&logoColor=white" alt="Datadog" />
  <img src="https://img.shields.io/badge/Version-3.0.1-blue?style=for-the-badge" alt="Version" />
  <img src="https://img.shields.io/badge/License-Proprietary-red?style=for-the-badge" alt="License" />
</p>

<h1 align="center">🏢 Enterprise Sales</h1>

<p align="center">
  <strong>A production-grade, security-first Sales Management API built entirely on n8n workflows.</strong>
  <br />
  Full authentication, transactional sales processing, real-time inventory management, automated reporting, and observability — without writing a single line of backend server code.
</p>

<p align="center">
  <a href="#-quick-start">Quick Start</a> •
  <a href="#-architecture">Architecture</a> •
  <a href="#-api-reference">API Reference</a> •
  <a href="#-security">Security</a> •
  <a href="#-deployment">Deployment</a> •
  <a href="#-license">License</a>
</p>

---

## 🎯 Overview

**Enterprise Sales** transforms [n8n](https://n8n.io) from a simple automation tool into a fully featured, enterprise-ready REST API backend. It exposes secure webhook endpoints that handle the complete lifecycle of a sales operation — from user authentication to invoice generation — while enforcing rate limiting, idempotency, input sanitization, and atomic database transactions.

### Why Enterprise Sales?

| Traditional Approach | Enterprise Sales |
|---|---|
| Months building a custom backend | Import a single JSON workflow |
| Managing Express/FastAPI servers | n8n handles HTTP routing & orchestration |
| Writing auth middleware from scratch | Supabase Auth with JWT validation out of the box |
| Manual rate limiting implementation | Redis-backed rate limiting per IP + User-Agent |
| Complex deployment pipelines | One-click deploy on n8n Cloud or self-host |

---

## ✨ Key Features

### 🔐 Authentication & Authorization
- **Supabase Auth** integration with JWT-based access & refresh tokens
- Role-based access control (`admin`, `seller`, custom roles)
- Multi-tenant support via `company_id` scoping
- CORS configuration with environment-driven allowed origins

### 🛡️ Enterprise Security
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
- Support for up to 100 line items per sale, quantities up to 10,000 units

### 📊 Reporting & Export
- Paginated sales queries with date range filtering (up to 10,000 records)
- Excel/PDF export via **CloudConvert** API integration
- Async report generation with execution job tracking

### 🔄 Operational Excellence
- **Health check endpoint** — monitors PostgreSQL + Redis connectivity
- **Automated daily backups** — scheduled at 2:00 AM UTC with `pg_dump`
- **Backup audit logging** — every backup is tracked in a `backup_logs` table
- **Centralized error logging** — all failures are forwarded to **Datadog**

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
| **Daily Backup** | Cron `0 2 * * *` | Execute pg_dump → Log Backup | Automated database backup |

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
git clone https://github.com/iarturo/Enterprise-Sales-API.git
cd Enterprise-Sales-API
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
-- This handles inventory locking, stock validation, and sale insertion atomically
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
3. Select `EnterpriseSales3.0.1.json`
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
  },
  "version": "1.0.0"
}
```

---

## 🔒 Security

Enterprise Sales implements defense-in-depth across multiple layers:

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
| **Error Logging** | Datadog integration | All errors are shipped to centralized logging |
| **CORS** | Environment-driven | `Access-Control-Allow-Origin` set via `FRONTEND_URL` |

> For the full changelog of security improvements, see [SECURITY_IMPROVEMENTS.md](SECURITY_IMPROVEMENTS.md).

---

## 🚢 Deployment

### Option 1 — Lightweight VPS (Recommended for small teams)

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

For 1,000+ concurrent users. Deploy PostgreSQL, Redis, and n8n as separate services behind an ingress controller with horizontal pod autoscaling.

### Scaling Guide

| Users | Auth | Cache | Workflow |
|---|---|---|---|
| 10–100 | Supabase Free | Redis single | n8n single |
| 100–1,000 | Supabase Pro | Redis cluster | n8n + load balancer |
| 1,000+ | Supabase Enterprise | Redis Enterprise | n8n on Kubernetes |

---

## 📁 Project Structure

```
Enterprise-Sales-API/
├── EnterpriseSales3.0.1.json          # ✅ Current production workflow (v3.0.1)
├── EnterpriseSales2.4.json            # Previous stable release
├── EnterpriseSales2.3.json            # Archive
├── EnterpriseSales2.2.json            # Archive
├── EnterpriseSales2.1.json            # Archive
├── EnterpriseSales2.0.json            # Archive
├── Function.txt                       # PL/pgSQL stored function (required)
├── GUIA_CONFIGURACION_FUNCIONAL.md    # Step-by-step setup guide (Spanish)
├── SECURITY_IMPROVEMENTS.md           # Security changelog & audit notes
├── LICENSE.md                         # Proprietary license
└── README.md                          # ← You are here
```

---

## 🗺️ Roadmap

- [ ] Two-factor authentication (TOTP via Supabase)
- [ ] Webhook notifications on sale creation (Slack / WhatsApp Business)
- [ ] PDF invoice generation with branded templates
- [ ] Multi-currency support
- [ ] Advanced analytics dashboard endpoint
- [ ] OpenAPI / Swagger specification
- [ ] Automated integration test suite

---

## 🤝 Contributing

This is a proprietary project. Contributions are accepted by invitation only. If you have been granted access for evaluation or collaboration purposes, please coordinate with the project owner before submitting changes.

---

## 📄 License

**Copyright © 2025 Arturo Ortega Salinas — All Rights Reserved.**

This software is proprietary and protected by copyright law. No part of this software may be used, copied, modified, distributed, sublicensed, or sold without prior written permission from the copyright holder. Access to this repository is granted solely for authorized evaluation and collaboration purposes.

See [LICENSE.md](LICENSE.md) for the full license text.

---

<p align="center">
  Built with ❤️ using <a href="https://n8n.io">n8n</a> · Powered by <a href="https://supabase.com">Supabase</a> & <a href="https://www.postgresql.org/">PostgreSQL</a>
</p>
