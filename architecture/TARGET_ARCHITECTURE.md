# KIRO target architecture and phased plan — Stage 3 and 4 proposal

| Version | Owner | Created | Last review | Approval | Applies to |
|---|---|---|---|---|---|
| 0.1 | KIRO owner (CTO role) | 2026-11-05 | 2026-11-05 | **Awaiting owner review** | All KIRO systems |

Evidence base: kiro-security/ENTERPRISE_GAP_ASSESSMENT.md (findings G-01…G-22) and kiro-security/COMPLIANCE_REGISTER.md.

---

## Where I disagree with the directive (and why)
1. **Do not build the 10,000-restaurant platform first.** Design for it, but build in phases sized to real restaurants. A full AWS enterprise stack (WAF, Multi-AZ RDS, Fargate, GuardDuty, Security Hub, two regions) costs roughly ₹60,000–1,50,000 per month before the first paying customer (estimate, verify with the AWS Pricing Calculator). Phase it.
2. **Hyderabad (ap-south-2) as primary is good for latency and localisation, but check every service there first.** Newer regions sometimes lack services or instance types. Mumbai (ap-south-1) has the widest coverage. Recommendation: confirm each service in ap-south-2; if any critical one is missing, run primary in Mumbai and DR in Hyderabad (both satisfy India-only storage).
3. **Separate kiro-database repository adds coordination cost** (accepted owner decision D4; mitigated by version pinning, ADR-0003).
4. **"Every system connected" must not mean "one database".** Payments keep their own schema, role and service boundary; analytics/AI get a read-only, tenant-scoped replica, never the primary.

---

## 1. Objective
A secure, multi-tenant restaurant platform in India: verified UPI payments settled directly to each restaurant, offline-capable POS that never fabricates payments, and governance that passes a bank's vendor review and an independent VAPT.

## 2. Existing condition
See the gap assessment. In short: live POS (device-only data, cashier-confirmed UPI), website, built-but-undeployed core API and disabled payment service, Neon database in Singapore, strong CI and tests, no infrastructure, monitoring or device management.

## 3. Proposed architecture

```
                         Internet
                            │
            CloudFront + AWS WAF + Shield Standard (TLS 1.3 preferred, 1.2 min)
              │                         │                              │
   kiro-web (static, S3)      api.kiro… (ALB, private subnets)   pay.kiro… (ALB, separate target group)
                                        │                              │
                   ┌────────────────────┴──────────┐      ┌────────────┴─────────────┐
                   │ Core API  (ECS Fargate)        │      │ KIRO PAY (ECS Fargate)    │
                   │ auth, tenants, POS sync, menu, │─────▶│ QR, webhooks, ledger,     │
                   │ orders, KDS, inventory, reports│ mTLS │ status checks, refunds,   │
                   └──────┬─────────────┬──────────┘ /JWT │ reconciliation            │
                          │             │                 └───────┬──────────┬────────┘
                  RDS PostgreSQL     SQS queues (+DLQ)            │          │
                  (Multi-AZ, RLS,    workers: receipts,     RDS schema   Provider APIs
                   KMS, PITR)        exports, sync, recon   `payments`   (bank / PA)
                          │                                  (own role)       │
                  Read replica ──▶ Reporting / AI (read-only, tenant-scoped)  │
                                                                webhooks ─────┘ (signature,
   Devices: POS tablets / iPads (registered, revocable) ──HTTPS──▶ Core API      idempotency)
   Observability: CloudWatch logs (India, ≥180 d), CloudTrail, GuardDuty, Security Hub, alerts
   Secrets: Secrets Manager + KMS (separate key admins);  CI/CD: GitHub OIDC → deploy roles
   DR: cross-region snapshots + IaC to rebuild in the second Indian region
```

### Trust boundaries
1. Internet ↔ edge (WAF). 2. Edge ↔ core API. 3. Core API ↔ KIRO PAY (separate service, credentials and DB role). 4. KIRO PAY ↔ payment provider. 5. Services ↔ database (no public endpoint). 6. Devices ↔ API (device credentials). 7. Tenant ↔ tenant (RLS + tenant-scoped keys/caches/files). 8. AI ↔ data (read-only replica, aggregates, no tools that write). 9. CI/CD ↔ cloud (short-lived OIDC roles, production behind approval).

### Key design decisions (alternatives considered)
| Decision | Choice | Alternatives | Why |
|---|---|---|---|
| Compute | ECS Fargate | EKS; Lambda | Least operations for a small team; Lambda awkward for long-lived POS sync and webhooks bursts; EKS overkill |
| Database | RDS PostgreSQL Multi-AZ | Aurora PostgreSQL; Neon | Aurora costs more at small scale; Neon region is Singapore today (G-02). Revisit Aurora at scale |
| Tenancy | Shared schema + `tenant_id` + RLS | Schema-per-tenant; DB-per-tenant | Scales to 10k tenants with one migration path; RLS gives DB-level guarantee; big chains can get dedicated DBs later |
| Queues | SQS + DLQ | Kafka; Redis streams | Durable, managed, cheap; ordering per tenant via message groups where needed |
| Auth | Own service: short access tokens, rotating refresh, EdDSA keys in KMS, TOTP/passkeys for owners and staff admins | Cognito; Auth0 | Cognito is an option (data in India region); decide in Phase 1 after a spike; either way no shared HS256 secret |
| POS offline | Local queue of signed orders, sync with idempotency keys; **UPI never confirmed offline** | Online-only | Restaurants need offline billing; offline UPI shows "pending verification" until synced and verified |

### Payment architecture (KIRO PAY)
- Per-merchant provider credentials stored encrypted (KMS envelope); KIRO never holds funds or settlement accounts.
- Flow: POS creates bill → KIRO PAY `Created` → requests QR from provider (`Awaiting QR`) → shows QR on customer display (`Pending`, expiry) → provider webhook (signature, merchant, amount, order, txn ref, replay window) **or** server status check → `Successful` → receipt event → POS bill marked "Paid (verified)".
- State machine enforced in DB; append-only `payment_events` ledger; refunds need an authorised role + reason + second confirmation; reconciliation job compares bills ↔ payments ↔ provider reports ↔ settlement files; mismatches go to an exception queue, never auto-corrected.

### Data classification
Public (website) · Internal (product metrics) · Confidential (menus, sales, staff) · **Restricted financial** (payment events, merchant credentials, settlement data — India only, KMS, strict roles) · **Sensitive personal** (customer phones, staff IDs — minimised, retention limits). Never store UPI PINs, card data or bank passwords.

## 4. Security analysis (summary)
STRIDE per boundary is in kiro-security/THREAT_MODEL.md (to be extended per phase). Highest risks: payment fraud (G-01), data residency (G-02), token theft (G-04), cross-tenant bugs (G-05), lost devices (G-07), single admin (G-10).

## 5. Attack scenarios to design against
Fake payment screenshot · replayed/forged webhook · two customers paying one bill · webhook ×3 · stale QR scanned · cashier refund abuse · tenant-ID tampering in API/exports/cache keys · stolen tablet · compromised owner account · leaked CI credentials · malicious npm dependency · prompt injection via menu item names into AI · provider outage during dinner rush.

## 6. Regulatory implications
See COMPLIANCE_REGISTER.md: DPDP (core duties 13 May 2027), CERT-In 6-hour reporting and 180-day India logs, RBI payment-data localisation (via provider), RBI PA Directions 2025 (KIRO must not handle funds), GST invoice correctness, consumer protection for KIRO LUX.

## 7. Implementation phases (small, verifiable, each ends in review)
| Phase | Scope | Closes | Est. effort* |
|---|---|---|---|
| **0 — Immediate safety (now)** | POS: label UPI as "confirmed by cashier, not verified", require UTR, day-end unverified-UPI report; GST test vectors in both repos; security.txt | G-01 (interim), G-16, G-22 | 3–5 days |
| **1 — Identity** | Auth redesign (short tokens, refresh rotation, revocation, MFA, sessions), Argon2id, Redis-backed limits | G-04, G-11, G-13 | 3–4 weeks |
| **2 — India infrastructure** | AWS account structure (Organizations: prod / staging / dev / security / log-archive), Terraform, VPC, RDS in India, KMS, Secrets Manager, logging ≥180 d, GuardDuty, Security Hub, backups + restore drill, CI OIDC deploys with approval | G-02, G-08, G-09, G-17 (partly) | 4–6 weeks |
| **3 — Tenant hardening** | RLS on all tenant tables, tenant-scoped caches/files/exports, automatic cross-tenant test generation per endpoint, API v1 + schema validation + correlation IDs | G-05, G-18 | 2–3 weeks |
| **4 — POS sync and devices** | Device registration/revocation, encrypted local store, offline queue with idempotent sync, server as source of truth for bills | G-07, G-16 | 4–6 weeks |
| **5 — KIRO PAY** (only after 0–4 + provider approval) | Full lifecycle, ledger, status checks, refunds, reconciliation + exception queue, customer display QR, VAPT, then controlled pilot with 1–3 restaurants | G-01 (final), G-03, G-06 | 6–8 weeks + external |
| **6 — Privacy and ISMS** | DPDP data map, notices, rights workflow, retention jobs, ISO 27001 ISMS documents, supplier register | G-15, G-14 | 4 weeks, ongoing |
| Later | Customer display, QR table ordering, online orders, KDS, inventory/procurement, recipes, attendance, multi-branch and central admin, SOC 2 | — | per module, each with its own design + threat model |

*Effort estimates for one experienced full-time engineer; they are planning estimates, not commitments.

## 8. Testing requirements
Every phase adds tests to kiro-qa-testing and the owning repo: unit, integration (real PostgreSQL on Neon qa / RDS staging), API contract, cross-tenant (automatic), payment lifecycle (all Part 13 scenarios with expected vs actual), load (k6: peak-hour model, see below), failure injection (provider down, DB failover, queue stop), backup restore drill, accessibility and browser matrix (existing).

### Capacity model (starting assumptions — to validate)
10,000 restaurants × avg 2 active POS devices; peak-hour bills ≈ 10,000 × 40 bills/h = 400,000 bills/h ≈ 110 bills/s average in the peak hour, design for 3× spikes ≈ 350 bills/s; each bill ≈ 5–10 API writes; payments ≈ 60% UPI ≈ 200 payment events/s at spike. Storage ≈ 10,000 × 300 bills/day × ~3 KB ≈ 9 GB/day before compression. These numbers drive DB sizing, connection pooling (RDS Proxy) and queue throughput tests.

### Proposed service levels (targets, not achieved)
App 99.9% · KIRO PAY 99.95% · payment-data RPO ≤ 5 min · payment RTO ≤ 30 min · zero silent transaction loss · zero cross-tenant leakage. Measure for at least one quarter before publishing.

## 9. Cost and resources (rough estimates — verify)
| Stage | Monthly infra (₹) | People |
|---|---|---|
| Pilot (≤50 restaurants): single-AZ RDS small, 2 Fargate tasks, CloudWatch, WAF basic | ~15,000–30,000 | 1–2 engineers |
| Production (≤1,000): Multi-AZ RDS, 4–6 tasks, GuardDuty, Security Hub, DR snapshots | ~60,000–1,50,000 | 3–5 engineers + part-time security + DevOps |
| Scale (10,000): larger RDS / Aurora, replicas, RDS Proxy, more tasks, 24×7 on-call | ~4–10 lakh | dedicated SRE, security, payments teams |
External one-off: legal opinion (payments + DPDP), CERT-In-empanelled VAPT, ISO 27001 audit — quotes required.

## 10. Outstanding risks
Single developer (G-10) blocks real separation of duties; provider choice and approval pending (G-03); budget for AWS and external audits not yet set; DPDP role classification unconfirmed.

## 11. Completion evidence (per phase)
Merged pull requests with green CI, test reports in kiro-qa-testing, Terraform plans and state in kiro-infrastructure, restore-drill and load-test reports, VAPT report and retest, signed provider approval, updated risk register entries closed with evidence links.

## 12. Approval requirements
| Item | Approver |
|---|---|
| Each phase plan and merge to main | KIRO owner (+ second engineer once hired) |
| Production infrastructure changes | Owner via GitHub environment `production` |
| Payment model and go-live | Owner + legal counsel + bank/PA written approval + VAPT sign-off |
| Privacy notices and DPDP role | Legal counsel |
| Security risk acceptance | Owner (as CISO until appointed), recorded in RISK_REGISTER |
