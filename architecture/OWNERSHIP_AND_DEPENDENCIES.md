# KIRO ownership and dependency map

One owner per component. Other repositories use a component by **version** (git tag) or through its **API**, never by copying it.

## Repositories

| Repository | Owns | Deployed to | Owner |
|---|---|---|---|
| kiro-web | Website (8 pages), KIRO LUX shop, Netlify config, icons | Netlify site `kiro-pos` → kiro-pos.netlify.app | @kirotick |
| kiro-pos | POS web app (billing, orders, tables, kitchen, cashier, offline) | Inside kiro-web at `/app/` (tag in `pos-version.txt`) | @kirotick |
| kiro-mobile | Owner/manager mobile app | Not deployed yet | @kirotick |
| kiro-backend | Core API: auth/JWT, roles, tenants, staff, items, stock, sales, reports, AI, nightly report; AI relay; ERP validator | Render (not deployed yet) | @kirotick |
| kiro-payments | KIRO PAY: PhonePe QR, webhooks, idempotency, bill status (**disabled**) | Render (not deployed; disabled) | @kirotick |
| kiro-database | All schemas and migrations (`core`, `payments`), migration runner | Neon project `kiro` | @kirotick |
| kiro-infrastructure | Render blueprint, environment map, backup/recovery | — | @kirotick |
| kiro-security | Threat model, risk register, access control, incident response | — | @kirotick |
| kiro-documentation | Architecture, ADRs, governance, audit, guides | — | @kirotick |
| kiro-qa-testing | All browser, unit-runner, security, performance and load testing; reports | GitHub Actions | @kirotick |

## Who owns what

| Thing | Owner | Used by |
|---|---|---|
| JWT issuing, roles (`perms.js`) | kiro-backend | kiro-payments verifies tokens (contract copy, ADR-0002); kiro-mobile sends them |
| Core tables (businesses, users, items, stock_lots, sales, sale_lines, sale_line_lots) | kiro-database `migrations/core` | kiro-backend only |
| Payment tables (merchant_payment_gateways, bills, phonepe_webhook_events) | kiro-database `migrations/payments` | kiro-payments only |
| Core API endpoints (`/api/auth/*`, `/api/me`, `/api/staff`, `/api/items`, `/api/sales`, `/api/reports/*`, `/api/ask`, `/api/ai/*`) | kiro-backend | kiro-mobile, kiro-pos (future sync) |
| Payment endpoints (`/phonepe-webhook`, `/api/gateway`, `/api/bills/*`) | kiro-payments | kiro-backend / POS (future), PhonePe |
| Bill arithmetic (GST split, round-off) | kiro-backend (server) — kiro-pos keeps an offline copy until sync exists; shared test vectors to be added (risk R-04) | — |
| POS app release | kiro-pos (tag) | kiro-web (`pos-version.txt`) |
| Database version used in tests/deploys | kiro-database (tag) | `database-version.txt` in kiro-backend/api and kiro-payments/service/node |

## Dependency diagram

```
kiro-web ──(downloads tag in pos-version.txt at build)──▶ kiro-pos
kiro-mobile ──(HTTPS API)──▶ kiro-backend ──(core migrations @ tag)──▶ kiro-database
kiro-backend (issues JWT) ··· verified by ···▶ kiro-payments ──(payments migrations @ tag)──▶ kiro-database
kiro-payments ──(signed webhooks; disabled)──▶ PhonePe
kiro-infrastructure ──(Render blueprint)──▶ kiro-backend, kiro-payments
kiro-qa-testing ──(checks out and tests)──▶ every repository
```
