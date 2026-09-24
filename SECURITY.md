# Security architecture (HPL AI study)

## The one rule

**No API key, secret, or service credential is ever placed in `index.html`, any other client-side file, or any committed file in this repo. Ever.**

The OpenAI key lives only as a Vercel environment variable, accessible only to the serverless functions in `/api`. The browser never sees it.

Firebase Realtime Database is private by default. The only browser-readable Firebase path is a single child under `/email_hashes/{hash}`, used for fast duplicate-email checks. The browser cannot read `/email_hashes` as a list and cannot write to Firebase.

## Architecture

```
Participant browser
  │
  │  fetch('/api/token')        ← issues an HMAC-signed session token (5-hour TTL)
  │  fetch('/api/chat',   ...)  ← gpt-5.4
  │  fetch('/api/router', ...)  ← gpt-5.4-mini
  │  fetch('/api/embed',  ...)  ← text-embedding-3-small
  │  fetch(Firebase /email_hashes/{hash}.json) ← direct duplicate-email read only
  │
  ▼
Vercel serverless functions (api/*.js)
  ├─ verify HMAC token (lib/token.js, lib/auth.js)
  ├─ enforce per-session rate limit + min-interval
  ├─ enforce model allowlist (no pivot to expensive models)
  ├─ enforce input-size caps
  └─ forward to OpenAI with the env-injected OPENAI_API_KEY
  │
  ▼
api.openai.com
```

Token issuance is rate-limited per IP (default 15/hour) and CORS-locked to the deployment origin.

Required Realtime Database rules:

```json
{
  "rules": {
    ".read": false,
    ".write": false,
    "email_hashes": {
      "$hash": {
        ".read": true,
        ".write": false
      }
    }
  }
}
```

## Required environment variables (set in Vercel dashboard, not in code)

| Variable             | Purpose                                                  |
|----------------------|----------------------------------------------------------|
| `OPENAI_API_KEY`     | The actual OpenAI key. Restricted scope: only the 3 models in the allowlist. |
| `TOKEN_SECRET`       | HMAC secret for signing study session tokens. >=32 random chars. |
| `ALLOWED_ORIGINS`    | (optional) Comma-separated CROSS-origin sites allowed to call the API. Same-origin requests are always allowed, so this is only needed if the API is called from a different domain. e.g. `https://hpl-study.vercel.app` |
| `FIREBASE_DATABASE_URL` | Realtime Database URL, e.g. `https://vera-f94da-default-rtdb.firebaseio.com`. Required for the backend Firebase route. |
| `FIREBASE_SERVICE_ACCOUNT_BASE64` | Base64 of the Firebase service-account JSON (Firebase console → Project Settings → Service Accounts → Generate new private key, then `base64 -i serviceAccount.json`). Without it the admin SDK falls back to `applicationDefault()` and HANGS on Vercel (30s timeouts). Powers condition rotation, sequential participant IDs, email-hash writes, and the cross-participant answer cache. |
| `PER_IP_TOKEN_LIMIT` | (optional) Tokens issuable per IP per hour. Default `15`. |
| `PER_TOKEN_CALL_LIMIT` | (optional) Calls allowed per session token. Default `500`. |

To set them:

```bash
vercel env add OPENAI_API_KEY
vercel env add TOKEN_SECRET
vercel env add ALLOWED_ORIGINS
```

Or via the dashboard: Project → Settings → Environment Variables.

## Layered defenses (what stops each kind of abuse)

| Threat                                      | What stops it                                      |
|---------------------------------------------|----------------------------------------------------|
| Key extracted from HTML                     | There is no key in the HTML. Nothing to extract.   |
| Function URL discovered by scraper          | CORS lock + token requirement → 403 / 401          |
| Bot scripts minting many tokens             | Per-IP issuance rate limit on `/api/token`         |
| Single legit token abused for high volume   | Per-token call cap + min interval                  |
| Compromised request pivots to costly model  | Per-endpoint model allowlist (rejects others)      |
| Long-running abuse with rotating tokens     | OpenAI project-level hard spend cap (set in OpenAI dashboard) |
| Key accidentally committed in the future    | `.git/hooks/pre-commit` blocks `sk-*` patterns     |

## Pre-commit hook

`.git/hooks/pre-commit` scans staged diffs for known secret patterns (OpenAI, AWS, Google, GitHub, Slack, PEM blocks). If any match shows up in a `+` diff line, the commit is refused.

The hook lives in `.git/hooks/` which is not tracked. If a teammate clones the repo, they must re-install it. To keep things simple, a copy is preserved at `scripts/install-hooks.sh` (TODO: add if a teammate ever joins).

To run a one-off scan on the whole tree:

```bash
git ls-files | xargs grep -EnH "sk-(proj|ant|or)-[A-Za-z0-9_-]{20,}" 2>/dev/null
```

## Past incident (2026-06-27)

The repo was public from 2026-05-28 to 2026-06-27 with three OpenAI keys committed in plaintext on `index.html`. Scraper bots harvested at least one key and burned ~$95 of unauthorized OpenAI spend before the spend alert tripped. Mitigations applied: keys revoked, repo flipped to private, password rotated, this proxy architecture built. The old keys remain in git history but are revoked and inert.

**If this repo is ever made public again**, scrub history first with `git-filter-repo` or BFG, even though the old keys are revoked — visible plaintext keys in a Harvard study repo are a bad look.

## Going forward — golden rules

1. **Never** `git add` a file containing a `sk-*` string. The pre-commit hook will catch obvious cases; you are the second layer.
2. **Never** paste a real key into source, even temporarily. Use env vars from minute one.
3. **Never** paste a real key into a chat with an AI assistant, a screenshot of your editor, a slack message, or a help-desk ticket body. Refer to it indirectly ("the key in Vercel env").
4. Set a **hard** OpenAI project spend cap, not just an alert. Alerts notify; caps stop.
5. Treat repo visibility as a deliberate decision, not a default. Default to private.
