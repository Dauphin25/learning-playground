# Lesson 08 — Auth: JWT, Refresh Rotation & API Security

**Track:** FastAPI (Surwill) · **Time:** ~2.5h · **Prerequisites:** lessons 04–06
**Objectives:** own the full token lifecycle (access + rotating refresh), password and reset flows, Google Sign-In verification, rate limiting, SSRF/headers hardening — and the trade-offs behind each.

---

## Why this matters

Auth is the subsystem where a mistake is a breach. It's also the interview
section with the most standard questions ("access vs refresh?", "where do you
store tokens?", "how does logout work with JWTs?"). Surwill implements the
full modern stack — short-lived HMAC access tokens, rotating single-use
refresh tokens stored hashed, bcrypt passwords, verified Google ID tokens,
Redis rate limiting — so you can answer from a system you can show.

## Theory: the two-token model

**Access token** — a JWT: header.payload.signature, HMAC-SHA256-signed with
`SECRET_KEY`. Stateless: any API instance verifies it with one hash operation,
no DB. Therefore *unrevocable* until expiry → keep it short (15 min here).

**Refresh token** — an opaque random string, *stored server-side (hashed)*.
Long-lived (30 days), exchanged for new tokens when the access token dies.
Because it's in the DB it **is** revocable — logout, password reset, and theft
response all operate on refresh tokens. **Rotation**: every use revokes the old
one and issues a new one, so a stolen-and-used refresh token collides with the
legitimate one and the anomaly is detectable.

The pairing gives you both properties: cheap stateless request auth + a
revocable session layer.

## The code, explained

### Primitives (`app/shared/security.py`)

```python
ALGORITHM = "HS256"

def hash_password(password: str) -> str:
    return bcrypt.hashpw(password.encode(), bcrypt.gensalt()).decode()

def hash_token(token: str) -> str:
    return hashlib.sha256(token.encode()).hexdigest()

def generate_refresh_token() -> str:
    return secrets.token_urlsafe(64)

def create_access_token(subject: str, extra: dict | None = None) -> str:
    expire = datetime.now(UTC) + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
    payload = {"sub": subject, "exp": expire, "type": "access"}
    ...
    return jwt.encode(payload, settings.SECRET_KEY, algorithm=ALGORITHM)

def decode_access_token(token: str) -> dict | None:
    try:
        payload = jwt.decode(token, settings.SECRET_KEY, algorithms=[ALGORITHM])
        if payload.get("type") != "access":
            return None
        return payload
    except JWTError:
        return None
```

Every line is an interview answer:

- **bcrypt for passwords** — deliberately slow, per-password salt; GPU-hostile.
  (Never SHA-256 for passwords.)
- **SHA-256 for tokens** — refresh tokens are 512-bit random strings, not
  guessable; hashing them fast is fine and means a DB dump leaks nothing usable.
  Knowing *why the two cases differ* (low-entropy vs high-entropy secrets) is
  the senior beat.
- **`secrets.token_urlsafe`**, never `random` — CSPRNG for anything
  security-bearing.
- **`algorithms=[ALGORITHM]` pinned on decode** — closes the classic `alg:none`
  / algorithm-confusion attacks.
- **`type` claim** — an access token can't be replayed where a different token
  kind is expected.

### Issue & rotate (`app/auth/services.py`)

```python
async def _issue_tokens(user_id, db) -> tuple[str, str]:
    access_token = create_access_token(str(user_id))
    raw_refresh = generate_refresh_token()
    token = RefreshToken(user_id=user_id, token_hash=hash_token(raw_refresh),
                         expires_at=now + timedelta(days=30))
    db.add(token); await db.commit()
    return access_token, raw_refresh          # raw goes to client, hash to DB

async def refresh_tokens(raw_token, db) -> tuple[str, str]:
    token = ...  # look up by hash, not revoked, not expired
    if token is None:
        raise UnprocessableError("Invalid or expired refresh token")
    token.revoked = True                       # ← rotation: single-use
    await db.flush()
    return await _issue_tokens(token.user_id, db)
```

The client (Surwill's Next.js `lib/api.ts`) holds up its side: on a 401 it
calls `/auth/refresh` **single-flight** (concurrent 401s share one refresh
promise) because rotation makes the refresh token single-use — two parallel
refresh calls with the same token means the second one fails and logs the user
out. That frontend-backend contract is a great systems-thinking story.

### Password reset done right (`request_password_reset` / `confirm_password_reset`)

- Response is identical whether the email exists or not (**no user
  enumeration**); the raw token is only echoed in non-production for dev flow.
- Token: same CSPRNG + hash-at-rest pattern, 60-minute expiry, **single use**,
  previous unused tokens invalidated on re-request.
- On confirm: password re-hashed, and — the detail interviewers reward —
  **every active refresh token for the user is revoked**:

```python
await db.execute(
    update(RefreshToken)
    .where(RefreshToken.user_id == user.id, RefreshToken.revoked == False)
    .values(revoked=True)
)
```

If the reset was because the account was compromised, the thief's sessions die
with the old password.

### Google Sign-In — verify, don't trust

`complete_google_login` receives a Google ID token (a JWT signed by Google) and
verifies it **properly**: fetch Google's JWKS (cached 6h, with a one-shot
refetch on unknown `kid` to survive key rotation), verify RS256 signature,
check `aud` == our client ID and `iss` ∈ Google's issuers, require
`email_verified`. Then: match by `google_id`, else **link by verified email**
(an existing password account gains Google login), else create a passwordless
account. The linking-only-on-verified-email detail prevents an account-takeover
vector (attacker registers a Google account with your unverified email spelling).

### Rate limiting & the rest of the armor

- Login: 10 attempts / 15 min per **IP+email** pair (Redis INCR+EXPIRE window;
  `_client_ip` honors `X-Forwarded-For` because uvicorn runs `--proxy-headers`).
  Reset requests: 3/hour per email — the mail pipe can't be weaponized.
- OTP-era lesson kept: limits enforced only in production so dev/test stay
  frictionless — a pragmatic pattern with a caveat (staging won't exercise it).
- Boot-time guard refuses production with dev `SECRET_KEY` (lesson 04).
- Security headers middleware (nosniff, X-Frame-Options, HSTS) — lesson 02.
- SSRF defense for user-supplied URLs — lesson 10's `assert_safe_url`.

**Storage trade-off to own:** the web client keeps tokens in `localStorage` —
simple, but readable by any XSS. The hardened alternative is httpOnly cookies
(+ CSRF defense). Surwill documents the migration as planned work; in an
interview, volunteering this trade-off unprompted reads as maturity, not
weakness.

## How this is used in production

This exact stack (short JWT + rotating refresh + bcrypt/argon2 + provider
ID-token verification) is the modern default whether hand-rolled or bought
(Auth0, Cognito, Firebase). At enterprise scale the deltas are: asymmetric
signing (RS256/EdDSA) so services verify with a public key and only the auth
service holds the private one; a JWKS endpoint of your own; central sessions
service for instant revocation; and WebAuthn/passkeys layering on top. Rate
limiting graduates from app-level Redis counters to the edge (Cloudflare,
API gateway) with the app layer kept as defense in depth.

## Advanced corner

- **Revocation lag** is the JWT tax: a disabled user's access token works for
  ≤15 min. Fixes if required: short TTL (chosen), a Redis denylist checked per
  request (adds the lookup JWTs were avoiding), or version claims checked
  against user records. Name the spectrum before the interviewer does.
- **HS256 vs RS256:** symmetric is fine while one service both signs and
  verifies. The moment a second service must verify, sharing the HMAC secret
  means anyone who can verify can *mint* — that's the cue for asymmetric.
- **bcrypt's 72-byte truncation** — why the schema caps passwords at 128 chars
  and why "passphrase + pepper" designs must mind the limit. Argon2id is the
  contemporary pick if starting fresh.
- **Timing side channels:** the login error is identical for "no such user"
  and "wrong password", but bcrypt only runs when a user exists — a
  measurable-in-theory oracle. Constant-work mitigation: hash a dummy on miss.
  Depth-of-knowledge point, rarely worth the code.

## Best practices

- Access short + stateless; refresh long + hashed + rotating + revocable.
- CSPRNG for all secrets; pin JWT algorithms; type-claim your tokens.
- Enumeration-proof flows (login errors, reset responses).
- Revoke all sessions on password change; single-flight refresh on the client.

## Common mistakes & gotchas

- Storing refresh tokens (or worse, passwords) un-hashed — a DB read becomes
  full account takeover.
- Verifying Google tokens by calling `tokeninfo` without checking `aud` — any
  app's token logs into yours.
- JWT decode without pinned algorithms / expiry check.
- Rate limiting by IP alone (breaks behind NAT, useless against botnets) —
  key on IP+identity like this codebase does.

## Where AI helps (and hurts)

AI knows these patterns cold and generates them well — *when asked for the
pattern by name* ("rotating refresh tokens, hashed at rest"). Asked vaguely
("add login"), it produces the naive version: long-lived JWT, plaintext refresh
rows, no rotation. You must know the names of the right things to ask for.

## Learn independently

- FastAPI security tutorial: https://fastapi.tiangolo.com/tutorial/security/
- RFC 6749 (OAuth2) §1.5 refresh tokens; RFC 7519 (JWT); RFC 8725 (JWT Best
  Current Practices — short, essential).
- OWASP Authentication & Session Management cheat sheets:
  https://cheatsheetseries.owasp.org/
- **Architecture Patterns with Python** ch. 13 for where auth wiring belongs.

## Interview Q&A

**Q: Why both an access and a refresh token?**
A: "They optimize opposite properties. The access token is a stateless JWT —
any instance authenticates a request with one HMAC check, no DB — but that
statelessness makes it unrevocable, so it lives 15 minutes. The refresh token
is the opposite: a random 512-bit secret stored hashed in Postgres, 30-day
life, fully revocable — logout and password reset just flip its row. And it
rotates: each use revokes it and issues a successor, so a stolen refresh token
that gets used collides with the real user's next refresh and surfaces."

**Q: How does 'logout everywhere after password reset' work?**
A: "Sessions *are* refresh-token rows, so it's one UPDATE setting
revoked=true for the user. Their access tokens ride out at most 15 minutes —
that's the deliberate revocation lag of stateless auth, and if the product
needed instant cutoff I'd add a denylist check and pay the per-request lookup."

**Q: You accept Google login. What exactly do you verify?**
A: "The ID token's RS256 signature against Google's published JWKS — cached,
with a refetch on unknown key ID to survive rotation — plus audience equals my
client ID, issuer is Google, and email_verified is true before I'll link it to
an existing account. Trusting an unverified email there is an account-takeover
vector."

## Exercises against the codebase

- **Easy:** Decode one of your access tokens on jwt.io; identify `sub`, `exp`,
  `type`, and explain why pasting the token there is safe-ish but the
  SECRET_KEY never would be.
- **Medium:** Use a refresh token twice via curl; observe rotation (second call
  422s). Then implement *reuse detection*: on refresh of an already-revoked
  token, revoke the user's whole family and log a security event.
- **Hard:** Sketch (or implement behind a flag) the httpOnly-cookie migration:
  what changes in `lib/api.ts`, what CSRF defense you add, and how refresh
  works cross-subdomain. Write the trade-off table you'd show a reviewer.

## Key takeaways

- 15-min HMAC JWTs + 30-day rotating hashed refresh tokens = cheap requests, revocable sessions, theft detection.
- bcrypt (slow) for passwords vs SHA-256 (fast) for high-entropy tokens — know why both are right.
- Reset flow: enumeration-proof, single-use, revokes all sessions.
- Verify provider tokens fully (signature/aud/iss/email_verified); rate-limit on IP+identity; own the localStorage trade-off.
