---
name: security-boundaries
description: Use when designing or reviewing authentication, authorization, sessions, JWTs, API keys, multi-tenant data isolation, secrets handling, password storage, file uploads, or any endpoint that touches user data — and when reviewing diffs that add endpoints, queries, or user-controlled input. Not a substitute for a professional audit on high-stakes systems.
---

# Security Boundaries

Three principles generate almost every rule here: **validate at the trust boundary, trust inside** (know where the boundary is); **least privilege** (a compromised component can only do what it was allowed to); **defense in depth** (one broken check ≠ breach). The empirical priority is set by OWASP Top 10 2021: **Broken Access Control is #1**, found in 94% of tested applications — authorization mistakes, not exotic crypto, are where real systems bleed.

## Decision 1: Authorization — the highest-yield section

- **AuthN ≠ AuthZ.** Authentication says who; authorization says whether *this* user may do *this* to *this object*. Most breaches are authz, not authn.
- **Object-level checks on every access (the IDOR rule).** "User is logged in" plus "record exists" is not authorization — verify the requester owns or may access *that specific object* (OWASP: IDOR). The classic failure: `GET /invoices/12345` returning another tenant's invoice because only login was checked.
- **Deny by default.** Access requires an explicit grant; a route with no policy attached fails closed, not open.
- **Centralize enforcement** in middleware/guards/a policy layer — per-handler ad-hoc checks guarantee the one forgotten endpoint, and access control "requires correct logic at every endpoint; one missed check is a vulnerability."
- **Server-side only.** UI hiding is UX, not security; every check the client performs is re-performed on the server. Never accept identity or role fields from the request body — derive them from the authenticated session.
- Whether unauthorized access to an *existing* resource returns 403 or 404 (existence-leak trade-off) is a deliberate per-resource decision — see `api-contracts` Decision 1.

## Decision 2: Session mechanism

**Options:**
- **Server-side sessions (default for first-party web apps)** — opaque cookie ID, state on the server. Instantly revocable, small, no crypto to misconfigure. Cookies get `HttpOnly; Secure; SameSite` + CSRF protection for state-changing requests.
- **JWTs** — choose for stateless service-to-service auth or third-party API access, accepting the cost: **a JWT cannot be revoked before expiry**. That forces short access-token lifetimes (minutes) + refresh tokens (revocable, stored server-side), which is a session store with extra steps — so don't pick JWT to "avoid state."

**JWT hard rules** (each maps to a real attack class — RFC 8725: JWT Best Current Practices):
- **Pin the accepted algorithms server-side; never read `alg` from the token.** The `alg: none` and RS256→HS256 confusion attacks both exploit trusting the attacker-controlled header.
- Validate `iss`, `aud`, and `exp` on every verification — a token minted for one service must not authenticate to another (use `typ`/explicit typing against cross-JWT confusion).
- Signing keys are secrets (Decision 4); asymmetric signing when verifiers shouldn't be able to mint.
- Claims are authenticated *at issue time* — long-lived role claims go stale; re-check privileged roles server-side.

**API keys** (machine callers): store hashed like passwords, prefix for identification (`sk_live_…`), scope narrowly, support rotation with overlapping validity from day one.

## Decision 3: Multi-tenant isolation

The most expensive class of bug a B2B SaaS can ship is tenant data bleed, and it's an access-control bug at scale.
- **Enforce tenancy at the data layer, not per-handler memory.** Every tenant-owned query filters by the *session-derived* tenant ID — via a scoped repository/query layer that makes unscoped queries impossible to write, with database row-level security as the backstop where available. Handler-by-handler `WHERE tenant_id = ?` discipline fails exactly once, which is enough.
- Tenant ID comes from the authenticated context, **never** from a request parameter.
- Schema requirements (tenant_id on every table, leading composite indexes, isolation-level choice) are in `data-modeling` Decision 8; the isolation model is a one-way door — design-doc it (`arch-design-doc`).
- Background jobs and admin tooling are part of the boundary: a cron that iterates "all rows" or an admin panel without tenant scoping is a bleed vector too.

## Decision 4: Secrets and credentials

- **Passwords:** slow, salted, adaptive hash — argon2id or bcrypt. Never MD5/SHA-family (built to be fast = built for brute force), never reversible encryption, never plaintext. Add a pepper only if you can manage it as a real secret.
- **Secrets live in a secret manager or injected env, never in code, git history, logs, error messages, or LLM prompts** (prompts persist in event history — see `llm-app-architecture`). One leaked-into-git credential means rotation, not deletion — assume copied.
- Design rotation in from the start (two valid keys during overlap); log secret *usage*, never secret *values*; redact tokens in request logging.

## Decision 5: Input and output boundaries

- **Trust boundaries are where validation lives** — user input, external APIs, file uploads, webhook payloads, message queues fed by other systems. Inside the boundary, don't re-validate what the type system and your own code guarantee (validation theater hides the real edges).
- **Injection is solved by construction, not sanitization:** parameterized queries always (string-built SQL is a finding regardless of escaping); same for shell commands (no string interpolation into `sh -c`), LDAP, XPath.
- **Output encoding for the destination context** — HTML-escape user content rendered to pages (XSS), including content that round-tripped through your DB or an LLM (model output is untrusted — `llm-app-architecture`).
- **SSRF:** any user-supplied URL your server fetches gets scheme/host allowlisting and blocked private-range resolution (OWASP A10).
- **File uploads:** validate type by content not extension, cap size, store outside the web root under generated names, serve with a content-disposition that prevents execution.
- Rate-limit authentication endpoints (credential stuffing) and expensive operations; respond identically for "wrong password" and "no such user."

## Review checklist (any diff adding endpoints or queries)

- Every new endpoint: authn required, authz check against the *specific object*, deny-by-default policy attached?
- Any ID from the client used to fetch data — ownership verified?
- Tenant scoping via the scoped layer, session-derived?
- New secrets: in the manager, rotatable, absent from logs?
- Any SQL/shell/HTML built by string concatenation from tainted data?
- Any server-side fetch of a user-supplied URL?
- Tokens: algorithm pinned, `iss`/`aud`/`exp` validated, lifetime justified?
- Auth endpoints rate-limited?

## Anti-patterns

- Authorization = "is logged in"; role checks without object checks
- Roles/user IDs read from the request body
- Per-handler tenant filtering with no structural enforcement
- Long-lived JWTs as sessions "to avoid a session store"; `alg` read from the token
- Fast hashes for passwords; secrets in git or log lines
- Frontend-only access control; security by obscurity of IDs (see `data-modeling` on non-enumerable public IDs — helpful, but not authorization)
- Sanitizing/escaping SQL instead of parameterizing
- Rolling your own crypto, session tokens, or password reset flows when vetted primitives exist
