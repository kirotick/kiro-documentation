# ADR-0005: kiro-web downloads the POS app at a pinned tag

**Context.** The POS app is its own repository but must stay available at `kiro-pos.netlify.app/app/`.

**Decision.** kiro-web's Netlify build runs `tools/fetch-pos.mjs`, which downloads `app/index.html` and `app/sw.js` from kiro-pos at the tag in `pos-version.txt` and checks the file is the POS app. Releasing a POS version = tag kiro-pos, then a reviewed pull request in kiro-web that changes `pos-version.txt`.

**Consequences.** The live POS version is always a known tag. Rollback = set `pos-version.txt` back to the previous tag. If GitHub is unreachable during a build, the build fails and the previous deploy stays live.
