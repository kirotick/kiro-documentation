# ADR-0004: Payments disabled by default

**Context.** No payment provider, merchant onboarding or security testing has been approved for production.

**Decision.** kiro-payments starts with payments **disabled** unless `PAYMENTS_ENABLED=true`; while disabled, every payment endpoint returns 503 and no QR is created and no webhook is processed. No environment sets the flag. A frontend response or screenshot is never proof of payment; only a verified webhook (signature, merchant and exact amount match, idempotent state change) marks a bill paid.

**Consequences.** Enabling payments needs: approved provider, merchant onboarding, security test of kiro-payments, a second reviewer (D3), and the owner's written approval in a kiro-security record.
