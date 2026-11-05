# ADR-0002: kiro-payments verifies backend tokens with a contract copy of auth.js

**Context.** kiro-backend issues HS256 JWTs. kiro-payments must verify them but must not depend on backend internals at runtime, and there is no shared package registry yet.

**Decision.** `kiro-payments/service/node/src/auth.js` is a marked, byte-for-byte copy of `kiro-backend/api/src/auth.js` (the token contract, ~50 lines). Both repositories run the same JWT test vectors (valid, wrong secret, `alg: none`, expired, missing `exp`). kiro-payments does not use `requireLiveUser` (it must not read the core `users` table); tokens expire after 12 hours.

**Consequences.** A change to the token format must change both files in linked pull requests. Replace with a versioned package (`@kiro/auth-contract`) when a second shared component appears. Tracked as risk R-05.
