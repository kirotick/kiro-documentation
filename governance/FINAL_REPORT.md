# KIRO GitHub governance — final report (2026-11-05)

Legend: **Implemented** · **Tested** (evidence below) · **Not yet implemented** · **Awaiting approval** · **Blocked (external)**. Nothing here has been independently reviewed by a second engineer (there is none yet, risk R-01).

## 1. Structure
No organisation could be found for `KIRO-TECHNOLOGIES`; on the owner's instruction all repositories are under the personal account **kirotick** (transferable to an organisation later, history and redirects intact). All public (D1), GitHub Free.

## 2. Repositories (all Implemented, v1.0.0 released)

| Repository | Purpose | From KIRO_POS (history kept) | Commits | CI jobs |
|---|---|---|---|---|
| kiro-web | Website, KIRO LUX, Netlify | web/site (minus app), tools, netlify.toml | 15 | secrets, browser-tests, codeql |
| kiro-pos | POS web app | web/site/app → app/ | 7 | secrets, browser-tests, codeql |
| kiro-mobile | Owner/manager mobile app | admin-app → root | 4 | secrets, tests, codeql |
| kiro-backend | Core API, AI relay, ERP validator | server/node (minus payments, migrations), ai-relay, erp-validator, sqlite reference | 5 | secrets, tests, codeql |
| kiro-payments | KIRO PAY (disabled) | payment handlers, signing, crypto, store, client, Python reference | 4 | secrets, tests, codeql |
| kiro-database | All schemas + migrations | server/node/migrations, migrate/db-info scripts | 4 | secrets, tests |
| kiro-infrastructure | Render blueprint, environments, backup | render.yaml | 5 | secrets |
| kiro-security | Threat model, risk register, access, incidents | new | 1 | secrets |
| kiro-documentation | Architecture, ADRs, governance, audit | docs, top-level notes, audit, inventory | 5 | secrets |
| kiro-qa-testing | All automated testing | qa, workflows | 12 | secrets, qa |
| KIRO_POS | **Archived** read-only (incl. desktop-saree, D2) | — | 25 | — |

## 3. Ownership and dependencies
See `architecture/OWNERSHIP_AND_DEPENDENCIES.md` and ADR-0001…0005. Versioned links: kiro-web `pos-version.txt` → kiro-pos tag; kiro-backend and kiro-payments `database-version.txt` → kiro-database tag.

## 4. Branch strategy (Implemented)
`main` protected in every repo; work in `feature/*`, `bugfix/*`, `security/*`, `release/*`, `hotfix/*` (CONTRIBUTING.md). No `develop` branch (single developer).

## 5. Access control (Implemented)
Ruleset "Protect main" in all 10 repos: no deletion, no force push, pull request required (0 approvals, D3 self-merge), required status checks per repo, review threads resolved. Matrix: kiro-security/ACCESS_CONTROL.md.

## 6. CI/CD (Implemented, Tested)
Per-repo `ci.yml`, read-only token, third-party actions pinned to commit SHAs, only GitHub-owned / verified / gitleaks actions allowed. Dependabot for npm and actions. GitHub environment `production` (owner approval) on kiro-web, kiro-backend, kiro-payments, kiro-infrastructure. Netlify deploys kiro-web `main`; rollback = previous deploy or revert.

## 7. Security configuration (Implemented)
Secret scanning + push protection, Dependabot alerts + security updates, private vulnerability reporting, CodeQL (JS) and gitleaks in CI, `.gitignore` for `.env*`, `*.pem`, `*.key` in every repo. Netlify headers: HSTS, nosniff, frame DENY, referrer policy, permissions policy.

## 8. Secrets
Runtime secrets only in Netlify / Render / Neon. Neon branches separate production / qa / dev. CI secret `TEST_DATABASE_URL` (qa branch) — **Not yet added** to the GitHub repos (owner action); until then CI runs database tests as skipped.

## 9. Database
kiro-database owns `migrations/core` and `migrations/payments`; idempotent; tested on Neon qa (apply, re-apply, groups). Production branch already migrated (core + payments, approved 2026-11-05). Destructive migrations require backup branch + plan (ADR-0003).

## 10. Testing results (Tested, this session)
| Check | Result |
|---|---|
| kiro-backend api (real PostgreSQL, HTTP, tenant isolation) | 54/54 |
| kiro-payments (incl. real PostgreSQL) | 16/16 |
| kiro-database migrations | 1/1 |
| kiro-mobile logic | 5/5 |
| AI relay / ERP validator | 9/9, 20/20 |
| Browser suite in split layout (Chromium) | 90 passed, 2 skipped |
| GitHub CI, all 10 repos | all jobs success |
| Live site after switching Netlify to kiro-web | 27/27; live /app/ byte-identical to kiro-pos v1.0.0 |
| Settings read back from GitHub API (rulesets, Dependabot, secret scanning, Actions, environments) | all as configured |
| Direct/force push to main refused | **Not executed** (blocked by the assistant's safety policy); ruleset read back as active |

## 11. Documentation status
README, CONTRIBUTING, SECURITY, CODEOWNERS, CHANGELOG, PR template (change record), Dependabot: all 10 repos. ADR 0001–0005. Threat model, risk register, access control, incident response, environments, backup and recovery.

## 12. Migration issues found and fixed
- CI YAML used `${{ }}` inside one-line maps → GitHub could not parse; fixed (block style), validated with a YAML parser.
- New repos lacked `.env` ignore rules (the monorepo root had them); found by the QA secret check before any push; fixed in all 10.
- Relative `KIRO_DATABASE_DIR` would resolve from the script folder; fixed to resolve from the working folder.
- Netlify API ignored `build_settings.repo_path`; relinked via the `repo` field.

## 13. Outstanding risks
See kiro-security/RISK_REGISTER.md (R-01 single reviewer, R-03 placeholder auth, R-04 duplicated bill math, R-05 auth contract copy, R-06 no RLS, R-07 no CSP, R-09 free tiers, R-10 Firefox POS).

## 14. Awaiting approval / external
- Second reviewer before live payments (R-01) — Awaiting approval.
- Enabling payments — Awaiting approval (ADR-0004).
- `TEST_DATABASE_URL` secret in kiro-backend, kiro-payments, kiro-database, kiro-qa-testing — owner action.
- Render deployment of kiro-api / kiro-pay — Not yet implemented.
- Organisation `KIRO-TECHNOLOGIES` — Blocked (not created).

## 15. Maintenance
Weekly Dependabot PRs; keep actions pinned (Dependabot updates SHAs); review RISK_REGISTER monthly; tag releases per repo; run `npm run qa` in kiro-qa-testing before every release; transfer to an organisation when a second developer joins.
