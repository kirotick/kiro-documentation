# ADR-0003: kiro-database owns every schema; services pin it by tag

**Context.** Owner decision D4 created a separate database repository. Splitting migrations from the code that uses them can let schema and code drift apart.

**Decision.** kiro-database owns `migrations/core` and `migrations/payments` and the runner. kiro-backend and kiro-payments record the database version they are tested against in `database-version.txt`; their CI checks out kiro-database at that tag, and their `npm run migrate` applies only their own group. A schema change is: pull request in kiro-database → tag → pull request in the service that bumps `database-version.txt` and adapts the code.

**Consequences.** Migrations stay additive and idempotent (safe to re-run). Destructive production migrations need an approved backup (Neon branch from production), a plan and a recovery path.
