# API Design Standards

## API Documentation

Dokumentasi lengkap setiap endpoint ada di direktori `api-documentation/`:

| File | Isi |
|---|---|
| `api-documentation/index.yaml` | Entry point — overview semua API |
| `api-documentation/auth.yaml` | Auth endpoints (register, login, refresh, dll) |
| `api-documentation/transactions.yaml` | Transaction endpoints + request/response schema |
| `api-documentation/accounts.yaml` | Account endpoints |
| `api-documentation/categories.yaml` | Category endpoints |
| `api-documentation/budgets.yaml` | Budget endpoints |
| `api-documentation/dashboard.yaml` | Dashboard home endpoint |

**Sebelum implement atau call endpoint apapun — baca file YAML yang relevan di `api-documentation/` untuk memastikan request body, response shape, dan field names yang benar.**

---

## Base URL

```
Production: https://api.spendos.elora.work/v1
```

## Available Endpoints

```
POST  /auth/register
POST  /auth/login
POST  /auth/google
POST  /auth/refresh
POST  /auth/logout
POST  /auth/verify-email
POST  /auth/resend-verification
POST  /auth/forgot-password
POST  /auth/reset-password

GET   /dashboard/home

GET   /transactions
POST  /transactions
GET   /transactions/:id
PUT   /transactions/:id
DELETE /transactions/:id         # soft delete only
POST  /transactions/:id/review
POST  /transactions/:id/split

GET   /accounts
POST  /accounts
PUT   /accounts/:id
POST  /accounts/:id/set-default
POST  /accounts/:id/archive
POST  /accounts/:id/restore

GET   /categories
POST  /categories
PUT   /categories/:id
POST  /categories/:id/hide
POST  /categories/:id/show

GET   /budgets
POST  /budgets
PUT   /budgets/:id
DELETE /budgets/:id
POST  /budgets/copy
```

## API Rules

- Every authenticated request must include: `Authorization: Bearer {jwt}`
- All timestamps in **ISO 8601 UTC**
- All IDs are **UUID v4**
- Success response: `{ "data": {...} }`
- Error response: `{ "error": { "code": "...", "message": "..." } }`
- Amount is always **integer** (IDR, no decimals) — format for display on the client
- Batch push capped at **500 records** per request
- DELETE on synced entities is always **soft delete** — never physical


## Update Api Documentation
- update api documentation secara berskala setiap session baru dengan membaca kembali folder `api-documentation`