---
name: auth-flows
description: Use when implementing or reviewing authentication flows — signup, login, logout, password reset, email verification, magic links, OAuth/OIDC integration ("login with Google"), SSO, MFA/2FA, passkeys, refresh tokens, or auth for SPAs/mobile apps/APIs. Complements security-boundaries (authz, token mechanics, session storage).
---

# Auth Flows

The governing rule: **never design a novel auth flow.** Every flow in this file exists as a published standard or cheat sheet because each detail closes a named, actively exploited attack — a flow you invent has unnamed holes. The second rule: the strength of an account's protection is the strength of its **weakest enabled path** (login, reset, recovery, support override) — attackers pick the door, not you.

## Decision 1: Build vs buy

- **Standard email+password for your own app:** building it is fine *if* you use vetted primitives (framework auth library, argon2id — `security-boundaries` Decision 4) and follow Decisions 4–6 below. This is a well-mapped path.
- **OAuth *client* ("login with Google/GitHub"):** fine to build with a maintained OIDC client library — never hand-roll the token exchange or `state`/nonce handling.
- **Being an identity provider, enterprise SSO (SAML/OIDC federation), SCIM provisioning:** buy or adopt a dedicated system (Auth0/Cognito/Clerk/WorkOS-class, or self-hosted Keycloak/Ory). SAML in particular has a long CVE graveyard of XML-signature bypasses; it is not a weekend feature. The build trigger is B2B customers asking for SSO — treat that as a procurement decision, not a sprint task.

## Decision 2: Which OAuth grant

RFC 9700 (Best Current Practice for OAuth 2.0 Security, Jan 2025) settles this:

| Caller | Grant | Notes |
|---|---|---|
| Anything with a user in it (web, SPA, mobile, desktop, CLI) | **Authorization code + PKCE** | PKCE (S256) for *all* clients, public and confidential — it also defends confidential clients against code injection |
| Service ↔ service, no user | **Client credentials** | Machine identity; scope narrowly |
| TVs, consoles, input-constrained devices | **Device authorization grant** | RFC 8628 |
| ~~Implicit grant~~ | **Never** | Deprecated by RFC 9700 — tokens in URLs leak via history/referrer/logs |
| ~~Resource owner password (ROPC)~~ | **Never** | Deprecated — teaches users to type passwords into third parties, breaks MFA |

Always: exact-match redirect URI registration (no wildcards), `state` parameter (CSRF), and for OIDC validate the ID token's `iss`/`aud`/nonce (`security-boundaries` JWT rules). Refresh tokens in public clients must be **rotated on every use with reuse detection** — a replayed old refresh token means theft; revoke the whole family (RFC 9700 §4.14).

## Decision 3: Browser-app architecture — where do tokens live?

The IETF OAuth for Browser-Based Apps BCP ranks three patterns:

- **BFF (backend-for-frontend) — default for first-party SPAs.** A thin backend is the confidential OAuth client; tokens never reach the browser; the SPA gets only an `HttpOnly; Secure; SameSite` session cookie and the BFF attaches access tokens to proxied API calls. XSS can *use* the session while the tab is open but cannot *exfiltrate* long-lived credentials — that's the whole argument.
- **Token-mediating backend** — backend acquires tokens, hands the access token (not refresh) to the SPA. Less proxying, weaker: XSS can steal the access token for its lifetime.
- **Browser-held tokens — last resort** (no backend possible). Then: in-memory only (never `localStorage` — any XSS reads it), short access-token lifetimes, rotated refresh tokens with reuse detection.

If your SPA and API are same-site first-party, the honest simplification is often: skip OAuth internally and use a plain server-side session cookie (`security-boundaries` Decision 2) — OAuth is a *delegation* protocol; using it between your own frontend and your own backend adds moving parts without adding trust.

## Decision 4: Passwords — signup and login

NIST SP 800-63B rev 4 reversed the folk wisdom; follow it:

- **Length over composition:** minimum 15 characters when password-only (8 if paired with MFA), long maximum (64+), allow all printable characters and spaces. Composition rules (must-have-symbol etc.) **SHALL NOT** be imposed — they produce `P@ssw0rd1!`, not entropy.
- **No periodic rotation** — force a change only on evidence of compromise. Scheduled expiry produces incremented suffixes and helpdesk load, not security.
- **Screen against breach corpora** (Pwned Passwords k-anonymity API or equivalent) at signup and change.
- Login responses identical for "no such user" and "wrong password" — timing included; rate-limit per account *and* per IP (credential stuffing is the attack, not brute force on one account); hash with argon2id/bcrypt (`security-boundaries`).
- On password change: invalidate all other sessions and notify the user — a hijacked account that quietly keeps old sessions alive defeats the change.

## Decision 5: Password reset — the flow attackers try first

The reset flow is an alternate login that skips the password; build it to the OWASP Forgot Password Cheat Sheet:

1. Request: uniform response regardless of account existence ("if that address exists, we sent a link"), uniform response *time*, rate-limited per account.
2. Token: CSPRNG-generated, single-use, expires in minutes-to-an-hour, **stored hashed** (a leaked DB must not yield live reset tokens), bound to the account, delivered by side channel (email link).
3. Redemption: invalidate the token on use *and* on any newer token issuance; require the new password to pass Decision 4 rules.
4. After: invalidate all existing sessions, notify the user ("your password was changed — wasn't you?"), and **do not auto-login** from the reset link — it turns every emailed link into a session-minting endpoint and complicates the session code (OWASP's explicit advice).

**Magic links are this same flow used as the login** — every rule above applies (single-use, short expiry, hashed at rest, uniform responses). **Email verification** likewise: single-use expiring token; and never merge an unverified email/password account with a later OAuth login for the same address without re-verification — that merge is a documented account-takeover class.

## Decision 6: MFA

Ranked by what actually resists current attacks — real-time phishing proxies (AitM kits) defeat any factor the user can be tricked into relaying:

1. **Passkeys / WebAuthn (FIDO2)** — phishing-resistant *by construction*: the credential is origin-bound, so a spoofed domain can't complete the ceremony. Default for new builds; offer as primary or step-up.
2. **TOTP (authenticator app)** — solid against password reuse and offline attacks, **not** phishing-resistant (a proxy relays the 6 digits in real time). Fine baseline.
3. **SMS** — weakest: SIM-swap and SS7 attacks, plus phishable. Acceptable only as a transitional/legacy option, never the ceiling for high-value accounts.

Mechanics: recovery codes generated at enrollment (shown once, stored hashed); **the fallback sets the real security level** — passkey + SMS-fallback is SMS-grade against a targeted attacker, so deliberately decide what an attacker can downgrade to; require **step-up re-authentication** for sensitive operations (email/password change, MFA changes, payouts, deletions) rather than trusting session age; rate-limit code verification (6 digits brute-force in seconds unthrottled).

## Decision 7: Sessions per client type

- **First-party web:** server-side session cookie — `HttpOnly; Secure; SameSite=Lax` + CSRF protection (`security-boundaries` Decision 2). Absolute *and* idle timeouts sized to the app's risk, not "30 days because convenient."
- **Mobile apps:** authorization code + PKCE (system browser, not embedded webview — webviews enable credential harvesting and break SSO); tokens in Keychain/Keystore; rotated refresh tokens.
- **Third-party API consumers:** API keys (hashed, prefixed, scoped, rotatable — `security-boundaries`) or client-credentials OAuth when you need scopes/expiry/delegation.
- Logout means server-side invalidation, not deleting the cookie client-side; "log out everywhere" requires enumerable sessions per user — design the session table for it from day one.

## Review checklist

- Grant type per RFC 9700? Any implicit/ROPC anywhere (including old docs/SDK samples)?
- PKCE + exact redirect URIs + `state`? Refresh rotation with reuse detection?
- SPA: tokens out of the browser (BFF), or at minimum out of `localStorage`?
- Password policy: ≥15 chars (or 8+MFA), no composition rules, no scheduled expiry, breach screening?
- Enumeration: signup, login, and reset all uniform in message *and* timing?
- Reset tokens: hashed at rest, single-use, short expiry? Sessions invalidated + user notified on password change?
- MFA: recovery path as strong as the front door? Step-up on sensitive ops? Verification rate-limited?
- Mobile: system browser, PKCE, secure storage?

## Anti-patterns

- Implicit flow or ROPC in anything new; tokens in URLs or `localStorage`
- Hand-rolled OAuth token exchange / JWT parsing instead of a maintained library
- "User not found" vs "wrong password" (or fast-path timing differences)
- Composition rules + 90-day rotation (NIST explicitly retired both)
- Reset tokens stored in plaintext, multi-use, or valid for days; auto-login from reset links
- Silent account merge on OAuth email match with an unverified local account
- SMS as the only or fallback factor on high-value accounts — the fallback is the front door
- MFA without recovery codes (lockout support tickets become the social-engineering vector)
- Embedded webviews for OAuth on mobile
- Logout that only clears the cookie; no way to enumerate/revoke a user's sessions
- Building SAML SSO in-house on a deadline
