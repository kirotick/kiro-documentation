# Architecture decision records

| ADR | Decision | Status | Date |
|---|---|---|---|
| [0001](0001-repository-structure.md) | Split `KIRO_POS` into 10 repositories under the `kirotick` account, history preserved | Accepted | 2026-11-05 |
| [0002](0002-auth-contract-copy.md) | kiro-payments verifies backend JWTs with a contract copy of `auth.js` | Accepted (temporary) | 2026-11-05 |
| [0003](0003-database-owns-migrations.md) | kiro-database owns every schema; services pin it by tag | Accepted | 2026-11-05 |
| [0004](0004-payments-disabled.md) | Payments disabled by default until approved | Accepted | 2026-11-05 |
| [0005](0005-pos-served-by-web.md) | kiro-web downloads the POS app at a pinned tag at build time | Accepted | 2026-11-05 |

Template: Context · Decision · Consequences · Status. A decision is changed by a new ADR that supersedes it, never by editing history.
