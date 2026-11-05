# ADR-0001: Ten repositories under kirotick, history preserved

**Context.** All KIRO code lived in one public repository (`kirotick/KIRO_POS`) with no branch protection, mixed products and payments inside the core API. The governance directive asks for separate, owned repositories.

**Decision.** Split into kiro-web, kiro-pos, kiro-mobile, kiro-backend, kiro-payments, kiro-database, kiro-infrastructure, kiro-security, kiro-documentation and kiro-qa-testing under the personal account `kirotick` (owner decision; no organisation yet). Each file keeps its history (`git filter-branch` keeping only owned paths, then `git mv` to the final layout). Repositories are public on GitHub Free (owner decision D1), which makes rulesets, secret scanning, push protection, Dependabot and CodeQL free. `desktop-saree` is archived inside `KIRO_POS` (D2). `KIRO_POS` is archived read-only, never deleted.

**Consequences.** Every change goes through a pull request with checks in the owning repo. Cross-repo changes need coordinated tags. The repositories can be transferred to an organisation later with history and redirects intact.
