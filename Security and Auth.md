# Security & Auth - Notes

> Grouped by theme. Each section covers: **what it is → why it exists → how it works step-by-step → what to say in an interview**.

---

## 1. Transport & Certificate Security

### TLS / SSL

#### What it is
**SSL** (Secure Sockets Layer) is the old, now-deprecated protocol. **TLS** (Transport Layer Security) is its successor and what every modern website uses. When your browser shows a padlock, that's TLS at work. They're often still called "SSL" loosely in conversation.

TLS solves three problems at once:
- **Encryption** — nobody can read the data in transit.
- **Authentication** — you're really talking to the server you think you are (not an impostor).
- **Integrity** — data can't be tampered with mid-way.

#### How it works — the TLS Handshake

Before any application data flows, client and server do a handshake to agree on encryption:

```
Client                                      Server
  |                                            |
  |------ ClientHello ----------------------->|   (TLS version + cipher suites I support)
  |                                            |
  |<----- ServerHello + Certificate ----------|   (chosen cipher + server's public cert)
  |                                            |
  | [Client verifies cert against trusted CA] |
  |                                            |
  |------ Key Exchange (ECDHE) ------------->|   (ephemeral key pair negotiation)
  |<----- Key Exchange ----------------------|
  |                                            |
  | [Both sides independently derive the same symmetric session key]
  |                                            |
  |------ Finished (encrypted) ------------->|
  |<----- Finished (encrypted) --------------|
  |                                            |
  |===== Encrypted Application Data =========|
```

- **Asymmetric crypto** (RSA/ECDH) is used only during the handshake to safely exchange a key.
- **Symmetric crypto** (AES) is used for the actual data — much faster.
- In **TLS 1.3**, the handshake is 1-RTT (one round trip) vs 2-RTT in TLS 1.2, and weak ciphers are removed entirely.

#### Key interview points
- TLS does not store data — it only protects data **in transit**.
- HTTPS = HTTP over TLS.
- `HSTS` header tells browsers to always use HTTPS even if you type `http://`.
- mTLS (mutual TLS) = both client *and* server present certificates — used in service meshes.

---

### Certificate Authority (CA) & Certificate Signing

#### What it is
A **CA** is a trusted organization (DigiCert, Let's Encrypt, GlobalSign) that acts like a passport office — it verifies who you are and issues you a certificate that others can trust.

Without CAs, anyone could create a certificate claiming to be `google.com`. CAs solve this by being a trusted middleman whose root certificates are pre-installed in every OS and browser.

#### How the signing process works

**Step 1 — You generate a key pair**
```
openssl genrsa -out private.key 2048          # private key (never share)
openssl rsa -in private.key -pubout           # public key (embedded in cert)
```

**Step 2 — Create a CSR (Certificate Signing Request)**
- Contains: your domain name, organisation name, public key.
- Signed with your private key to prove you own it.
```
openssl req -new -key private.key -out request.csr
```

**Step 3 — CA verifies and signs**
- You submit the CSR to a CA (e.g. Let's Encrypt via ACME protocol).
- CA verifies domain ownership (puts a file on your server, or DNS record).
- CA signs your certificate using *its* private key.
- Result: a `.crt` file — your signed certificate.

**Step 4 — Certificate is trusted by browsers**
- Browser checks cert → finds it's signed by Intermediate CA → signed by Root CA → Root CA is in its trust store → ✅ valid.

#### Chain of trust
```
Root CA  (self-signed, stored in OS/browser)
  └── Intermediate CA  (signed by Root CA)
        └── Your server cert  (signed by Intermediate CA)
```

Browsers don't rely on a single root — intermediates limit the blast radius if a CA is compromised.

#### Certificate types
| Type | Verification level | Visual |
|---|---|---|
| **DV** (Domain Validated) | Only domain ownership | Padlock |
| **OV** (Organisation Validated) | Domain + company identity | Padlock |
| **EV** (Extended Validation) | Thorough business vetting | Padlock (green bar in old browsers) |

#### Key interview points
- Self-signed certs are fine for internal/dev use, but browsers will warn users.
- Let's Encrypt automates issuance and renewal via the **ACME protocol** (free, 90-day certs).
- A **wildcard cert** (`*.example.com`) covers all subdomains.

---

### X.509 Certificates vs Secrets Vaults

These are often confused — they solve different problems and work best together.

#### X.509 Certificates
The **standard format** for public-key certificates (what TLS uses, what CAs sign). Think of it as a digital ID card that contains:
- Your public key
- Your identity (domain, organisation)
- Who signed it (CA name)
- Expiry date
- Allowed usages (server auth, client auth, code signing)

Used for: TLS on websites, mutual TLS between services, signing code/emails.

#### Secrets Vault (e.g. HashiCorp Vault, AWS Secrets Manager)
A **centralised secrets management system** — a secure store where apps fetch their credentials at runtime instead of baking them into config files or environment variables.

**How Vault works:**
1. Your app authenticates to Vault (via AWS IAM role, Kubernetes service account, etc.).
2. Vault returns a short-lived secret — e.g., DB username/password that expires in 1 hour.
3. Vault auto-rotates credentials. If leaked, damage is limited.
4. Every secret access is **audited**.

| | X.509 Certificates | Secrets Vault |
|---|---|---|
| **Purpose** | Identity & encrypted transport | Storing and distributing secrets |
| **Stores** | Public key + CA signature | Passwords, API keys, DB creds, dynamic secrets |
| **Auth via** | PKI — trust the cert chain | Token, AppRole, cloud IAM |
| **Best for** | TLS, mTLS, service identity | App secrets, DB creds, short-lived tokens |
| **Rotation** | Manual or ACME automation | Built-in, automatic |

> They complement each other: use **certs for identity and encryption**, use **vaults for secret distribution**.

---

## 2. Identity & Authentication Protocols

### OAuth 2.0

#### What it is
OAuth 2.0 is an **authorisation framework** — it lets a user grant a third-party app access to their resources *without giving away their password*.

Real-world analogy: You give a valet a special "valet key" that only lets them drive the car — not open the boot. OAuth gives apps a limited-access token, not your password.

> **Critical distinction:** OAuth 2.0 is about *authorisation* (what you can do), not *authentication* (who you are). OIDC adds authentication on top.

#### Key roles
| Role | Meaning | Example |
|---|---|---|
| **Resource Owner** | The user | You |
| **Client** | The app requesting access | A todo app |
| **Authorisation Server** | Issues tokens | Google, Okta, Auth0 |
| **Resource Server** | The API holding your data | Google Contacts API |

#### How it works — Authorization Code Flow (most secure)

This is the standard flow for web apps with a backend:

```
User            Client App          Auth Server          Resource Server (API)
  |                  |                    |                        |
  |-- clicks login ->|                    |                        |
  |                  |-- redirect to AS ->|                        |
  |                  |  (client_id,       |                        |
  |                  |   redirect_uri,    |                        |
  |                  |   scope, state)    |                        |
  |                  |                    |                        |
  |<------------ login page from AS ------|                        |
  |                                       |                        |
  |-- enters credentials + consents ----->|                        |
  |                                       |                        |
  |<-- redirect to app with auth CODE ----|                        |
  |                  |                    |                        |
  |                  |-- POST /token ---->|                        |
  |                  |  (code,            |                        |
  |                  |   client_secret)   |                        |
  |                  |                    |                        |
  |                  |<-- access_token ---|                        |
  |                  |    refresh_token   |                        |
  |                  |                    |                        |
  |                  |------- GET /data with Bearer token -------->|
  |                  |<------ protected data ----------------------|
```

- The **authorization code** is short-lived (seconds) and single-use.
- The **access token** is what hits the API — usually valid 5–60 minutes.
- The **refresh token** (long-lived) is used to silently get new access tokens when they expire.
- The `state` parameter prevents CSRF on the redirect.

#### Other OAuth flows
| Flow | When to use |
|---|---|
| **Authorization Code + PKCE** | SPAs and mobile apps (no backend to hold client_secret) |
| **Client Credentials** | Server-to-server (no user involved) |
| **Device Code** | TVs, CLIs — device shows a code, user authenticates on phone |
| ~~Implicit~~ | Deprecated — tokens exposed in URL |

#### Scopes
Scopes define *what* access the token grants: `openid`, `email`, `read:contacts`, `write:calendar`. The user sees and consents to these on the consent screen.

---

### PKCE (Proof Key for Code Exchange)

#### What it is
PKCE (pronounced "pixie") is a security extension to OAuth 2.0 that protects the **authorization code flow for public clients** — apps like SPAs and mobile apps that cannot safely store a `client_secret` (anyone can view a mobile app's source or a browser's JS files).

Without PKCE, if a malicious app intercepts the authorization code (via a redirect URI attack on mobile), it can exchange it for a token. PKCE stops this.

#### How it works

```
App generates:
  code_verifier  = random 43-128 char string  (e.g. "abc123xyz...")
  code_challenge = BASE64URL(SHA256(code_verifier))  (a hash)

Step 1 — Auth request (sends the HASH, not the verifier):
  GET /authorize?
    client_id=...
    &code_challenge=<hash>
    &code_challenge_method=S256
    &redirect_uri=...

Step 2 — Auth server stores the challenge, user logs in, returns auth code.

Step 3 — Token exchange (sends the ORIGINAL verifier):
  POST /token
    code=<auth_code>
    code_verifier=<original_random_string>

Step 4 — Auth server hashes the verifier, compares with stored challenge → match → issues token.
```

**Why it's secure:** An attacker who steals the auth code doesn't have the `code_verifier` (it was never transmitted). They can't exchange the code.

#### Key interview points
- PKCE replaces `client_secret` for public clients.
- Even confidential clients (server apps) should use PKCE now — it adds defence in depth.
- The verifier is created fresh per auth request, so it can't be pre-computed.

---

### OIDC (OpenID Connect)

#### What it is
OIDC is a thin **identity layer built on top of OAuth 2.0**. OAuth tells you *what* a user can access; OIDC tells you *who the user is*.

It adds one key thing: an **ID Token** — a JWT returned alongside the access token that contains verified identity information.

#### How it differs from plain OAuth 2.0

| | OAuth 2.0 | OIDC |
|---|---|---|
| Purpose | Authorisation | Authentication + Authorisation |
| Token returned | Access Token | Access Token + **ID Token** |
| Who are you? | ❌ Not defined | ✅ ID Token has `sub`, `email`, `name` |
| Scope needed | Any custom scope | Must include `openid` scope |

#### How it works

The flow is the same as OAuth Authorization Code — with one addition: include `scope=openid` in the request. The token endpoint then returns both an access token and an ID Token.

**ID Token structure (it's a JWT):**
```json
{
  "iss": "https://accounts.google.com",
  "sub": "1234567890",
  "aud": "your-client-id",
  "email": "user@example.com",
  "name": "Jane Doe",
  "iat": 1715689200,
  "exp": 1715692800
}
```

| Claim | Meaning |
|---|---|
| `iss` | Issuer — who issued the token |
| `sub` | Subject — unique user ID |
| `aud` | Audience — your app's client_id |
| `iat` | Issued at (Unix timestamp) |
| `exp` | Expiry (Unix timestamp) |

#### OIDC extras
- **Discovery endpoint:** `/.well-known/openid-configuration` — your app fetches this to find all OIDC endpoints automatically.
- **`/userinfo` endpoint:** Call with the access token to get fresh user attributes.
- OIDC is the foundation of modern SSO. When you "Login with Google", that's OIDC.

---

### SAML (Security Assertion Markup Language)

#### What it is
SAML is an **XML-based open standard for exchanging authentication and authorisation data** — mainly used for **enterprise SSO**. It's older (2005) but widely adopted in corporate environments (think Okta, Microsoft ADFS connecting to internal apps).

#### Key roles
- **Identity Provider (IdP):** Authenticates users and issues assertions. Example: Okta, Azure AD, ADFS.
- **Service Provider (SP):** Your application that trusts the IdP. Example: Salesforce, your internal HR app.

#### How it works — SP-Initiated Flow

```
User               Your App (SP)               Okta (IdP)
  |                     |                           |
  |-- access /dashboard->|                           |
  |                     |                           |
  |     (no session)    |                           |
  |                     |-- SAML AuthnRequest ----->|
  |<-- redirect to Okta--|   (XML, signed)           |
  |                     |                           |
  |-- logs in at Okta -------------------------------->|
  |                     |                           |
  |<-- browser posts SAML Response to /acs ----------|
  |     (contains signed XML Assertion)              |
  |                     |                           |
  |-- POST /acs ------->|                           |
  |                     | (SP validates signature)  |
  |<-- logged in -------|                           |
```

- `/acs` = Assertion Consumer Service — the endpoint on your app that receives SAML responses.
- The assertion is **XML signed** with the IdP's private key. Your app verifies it with the IdP's public key.

#### What a SAML Assertion contains
- User identity (email, name, employee ID)
- Roles/groups
- Authentication timestamp
- Expiry time

#### IdP-Initiated Flow
User logs into the IdP portal first (e.g. Okta dashboard), clicks on your app tile → IdP sends assertion directly. No redirect back to SP first.

#### SAML vs OIDC
| | SAML | OIDC |
|---|---|---|
| Format | XML | JSON / JWT |
| Age | 2005 | 2014 |
| Complexity | High | Low |
| Best for | Enterprise legacy apps | Web apps, mobile, APIs |
| Transport | Browser POST | HTTP redirect + REST |

---

### SSO (Single Sign-On)

#### What it is
SSO lets a user **log in once and access multiple applications** without being asked to log in again for each one.

Real-world analogy: A hotel key card — you check in once at the front desk, and that same key opens your room, the gym, and the parking garage. You don't re-authenticate at each door.

#### How it works

The key mechanism is a **centralised session at the IdP**:

1. User goes to App A → no session → redirected to IdP (Okta, Azure AD).
2. User logs in at IdP → IdP creates a **session cookie** on its own domain.
3. IdP redirects back to App A with a token/assertion → App A creates its own session.
4. User goes to App B → no session → redirected to IdP.
5. IdP already has a session cookie → **silently authenticates** → redirects back to App B with a new token.
6. User never sees a second login prompt.

```
App A ──redirects──> IdP ──issues token──> App A (session created)
                      |
App B ──redirects──> IdP ──silent check──> App B (session created, no login prompt)
```

#### SSO implementations
| Protocol | Used by |
|---|---|
| SAML 2.0 | Corporate apps (Salesforce, Workday) |
| OIDC | Consumer apps (Google, GitHub login) |
| Kerberos | Windows domain networks (on-premise) |

#### Federated SSO
SSO *across* organisations. Example: A partner company's employee logs into your app using *their* company's IdP. Your app trusts their IdP via a pre-established federation agreement.

#### Key interview points
- SSO improves UX and security (fewer passwords = fewer breach points).
- **Single logout (SLO):** Logging out of one app should log you out of all. Tricky to implement correctly.
- If the IdP goes down, all apps relying on it go down → IdP is a critical availability dependency.

---

### MFA (Multi-Factor Authentication) with OAuth

#### What it is
MFA requires users to prove identity with **two or more factors from different categories**:
- **Something you know** — password, PIN
- **Something you have** — TOTP app (Google Authenticator), hardware key (YubiKey), SMS OTP
- **Something you are** — fingerprint, face scan

The idea: Even if your password is stolen, the attacker still can't log in without the second factor.

#### How TOTP (Time-Based OTP) works
TOTP is the most common MFA method (the 6-digit code in your authenticator app):

1. During setup, IdP generates a **shared secret** (random bytes) and shows it as a QR code.
2. Your authenticator app stores that secret.
3. Every 30 seconds: `OTP = HMAC-SHA1(secret, current_time_window)` — both your app and the server compute the same value.
4. You type the 6-digit code → server verifies it matches.

No internet needed — the code is derived from a shared secret + time.

#### How MFA integrates with OAuth
OAuth/OIDC doesn't mandate how you authenticate — that's entirely up to the IdP. The flow is:

```
User → enters username + password at IdP
IdP → prompts for second factor (TOTP code, push notification, etc.)
User → completes MFA
IdP → issues access token + ID token (as normal)
```

The **`acr` claim** (Authentication Context Class Reference) in the ID Token can tell the app *how strong* the authentication was:
- `acr: "urn:mace:incommon:iap:silver"` = password only
- `acr: "urn:mace:incommon:iap:gold"` = MFA completed

**Step-up authentication:** An app can force re-authentication with MFA mid-session by including `acr_values` in a new OAuth request — e.g. when a user tries to access a sensitive page.

---

### Passkeys

#### What it is
Passkeys are a **passwordless authentication standard** based on **FIDO2 / WebAuthn**. Instead of a password, your device holds a private key and uses it to prove your identity — your biometric (fingerprint, FaceID) unlocks the key locally, it never goes to the server.

Think of it like this: Instead of showing someone a password (which can be written down and stolen), you prove identity by signing something only you can sign — cryptographically.

#### How it works

**Phase 1 — Registration (one time)**
```
Browser/App                    Your Device                    Server
     |                              |                            |
     |-- register passkey --------->|                            |
     |                              |                            |
     |                   [biometric prompt]                      |
     |                              |                            |
     |              generates key pair:                          |
     |              private_key (stored securely on device)      |
     |              public_key  (sent to server)                 |
     |                              |                            |
     |<--- public_key + credential_id --------------------------->|
     |                              |         (server stores it) |
```

**Phase 2 — Authentication (every login)**
```
Browser/App                    Your Device                    Server
     |                              |                            |
     |<-- challenge (random nonce) --------------------------------|
     |-- forward challenge -------->|                            |
     |                              |                            |
     |                   [biometric prompt]                      |
     |                              |                            |
     |              signs(challenge, private_key) = signature    |
     |                              |                            |
     |<-- signature + credential_id--|                            |
     |                              |                            |
     |-- send signature ----------------------------------------->|
     |                    [server verifies signature             |
     |                     with stored public_key] ✅            |
```

#### Why it's more secure than passwords
- **No password to steal** — there's nothing on the server to breach.
- **Phishing-resistant** — the private key is domain-scoped. A fake `g00gle.com` cannot request a key made for `google.com`.
- **No shared secret** — the server only stores the public key; it's useless to an attacker.
- Passkeys **sync across devices** via iCloud Keychain or Google Password Manager.

---

## 3. API & Web Security

### API Authentication Methods

#### Overview comparison
| Method | How it works | Best for | Watch out for |
|---|---|---|---|
| **API Key** | Static string in header/query param | Simple server-to-server | No user identity; rotate often; easy to leak |
| **Bearer Token (JWT)** | Token in `Authorization: Bearer <token>` header | Stateless APIs | Validate signature + claims; keep short-lived |
| **OAuth 2.0** | Delegated token from auth server | User-facing APIs (3rd party access) | Use Authorization Code + PKCE |
| **mTLS** | Both client and server present X.509 certs | High-security microservices, banking APIs | Infrastructure-heavy to set up |
| **HMAC Signature** | Client signs request body + timestamp with shared secret | Webhooks, AWS-style APIs | Replay attacks — include timestamp + nonce |
| **Session Cookie** | Server stores session; client sends cookie | Traditional web apps | Susceptible to CSRF; use SameSite + CSRF token |

#### API Key — how it works
```
Client → GET /data
         Header: X-API-Key: abc123secret
Server → looks up abc123secret in DB → checks permissions → responds
```
Simple but no user context. Anyone who gets the key has full access.

#### mTLS — how it works
Normal TLS only authenticates the *server*. mTLS adds client authentication:
```
Client presents its cert → Server verifies client cert against trusted CA
Server presents its cert → Client verifies server cert
Both sides are authenticated before any data flows
```
Used in service meshes (Istio, Linkerd) to ensure only authorised services communicate.

#### HMAC Signature — how it works
```
Client:
  signature = HMAC-SHA256(secret_key, method + url + body + timestamp)
  sends: request + signature + timestamp in header

Server:
  recomputes signature with the same secret
  compares → if match, request is genuine
  checks timestamp → reject if > 5 min old (prevents replay attacks)
```
AWS uses this (SigV4). Stripe uses it for webhooks.

---

### JWT (JSON Web Token)

#### What it is
JWT is a **compact, self-contained token format** for securely transmitting information between parties as a JSON object that is **digitally signed**. The server can verify the token's authenticity without a database lookup — making APIs stateless and scalable.

#### Structure
A JWT looks like this: `xxxxx.yyyyy.zzzzz` — three Base64URL-encoded parts separated by dots.

**Part 1 — Header**
```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

**Part 2 — Payload (claims)**
```json
{
  "sub": "user_123",
  "iss": "https://auth.example.com",
  "aud": "https://api.example.com",
  "exp": 1715692800,
  "iat": 1715689200,
  "email": "user@example.com",
  "roles": ["admin", "editor"]
}
```

| Claim | Meaning |
|---|---|
| `sub` | Subject — who the token is about |
| `iss` | Issuer — who created the token |
| `aud` | Audience — who should accept it |
| `exp` | Expiry timestamp |
| `iat` | Issued at timestamp |

**Part 3 — Signature**
```
RSASHA256(
  base64url(header) + "." + base64url(payload),
  private_key
)
```

The server verifies by re-checking the signature using the **public key**. If anyone tampers with the payload, the signature check fails.

#### How a server validates a JWT
1. Split token into 3 parts.
2. Verify the **signature** using the auth server's public key.
3. Check `exp` — is it expired?
4. Check `iss` — is it from the expected issuer?
5. Check `aud` — is it intended for this API?
6. Read claims — extract user identity and roles.

#### Common pitfalls and attacks
- **`alg: none` attack:** Some buggy libraries accept a token with `"alg": "none"` and skip verification. Always enforce the algorithm server-side.
- **Storing in localStorage:** Vulnerable to XSS. Prefer `HttpOnly` cookies for sensitive tokens.
- **Long expiry:** Tokens can't be revoked before expiry. Keep access tokens short (5–15 min) and use refresh tokens.
- **Sensitive data in payload:** JWT payload is Base64-encoded, not encrypted — anyone can decode it. Never put passwords or PII. Use **JWE** (JSON Web Encryption) if needed.

---

### CORS (Cross-Origin Resource Sharing)

#### What it is
By default, browsers enforce the **Same-Origin Policy** — a script on `https://myapp.com` cannot make requests to `https://api.otherdomain.com`. CORS is the mechanism that lets servers *relax* this restriction in a controlled way.

An "origin" = scheme + hostname + port. So `http://example.com` and `https://example.com` are *different* origins.

#### Why it exists
Without SOP, a malicious site could make authenticated requests to your bank's API using your stored cookies. CORS is the browser-enforced security boundary.

#### How it works

**Simple requests** (GET, HEAD, POST with basic content types):
```
Browser:
  GET https://api.example.com/data
  Origin: https://myapp.com

Server:
  Access-Control-Allow-Origin: https://myapp.com   ← allows this origin
```
If the server doesn't return this header, the browser blocks the response (request went through, but JS can't see the response).

**Preflight (complex requests)** — for requests with custom headers, non-simple methods (PUT/DELETE), or with credentials:
```
Browser first sends OPTIONS request:
  OPTIONS /data
  Origin: https://myapp.com
  Access-Control-Request-Method: DELETE
  Access-Control-Request-Headers: Authorization

Server responds:
  Access-Control-Allow-Origin: https://myapp.com
  Access-Control-Allow-Methods: GET, POST, DELETE
  Access-Control-Allow-Headers: Authorization
  Access-Control-Max-Age: 86400   ← cache preflight for 1 day

Then browser sends the actual DELETE request.
```

#### Common CORS headers
| Header | Set by | Purpose |
|---|---|---|
| `Access-Control-Allow-Origin` | Server | Which origin is allowed |
| `Access-Control-Allow-Methods` | Server | Which HTTP methods |
| `Access-Control-Allow-Headers` | Server | Which request headers |
| `Access-Control-Allow-Credentials` | Server | Allow cookies/auth headers |
| `Access-Control-Max-Age` | Server | Cache preflight duration |
| `Origin` | Browser | Browser sends this automatically |

#### Key points
- CORS is **browser-enforced only** — curl and server-to-server calls ignore it.
- `Access-Control-Allow-Origin: *` + `Access-Control-Allow-Credentials: true` is **invalid** — you cannot allow all origins *and* send credentials.
- CORS doesn't prevent all attacks — it's not a substitute for auth.

---

### CSRF (Cross-Site Request Forgery)

#### What it is
CSRF tricks a logged-in user's browser into **sending an unintended request to a site they're authenticated on**. The browser automatically attaches session cookies, so the server sees a legitimate-looking request.

#### Attack scenario — step by step
```
1. User logs into bank.com → browser stores session cookie.
2. User visits evil.com (attacker's site).
3. evil.com has a hidden form:
   <form action="https://bank.com/transfer" method="POST">
     <input name="amount" value="10000">
     <input name="to" value="attacker_account">
   </form>
   <script>document.forms[0].submit()</script>
4. Browser submits the form → attaches bank.com cookie automatically.
5. Bank server sees a valid session → transfers money.
```
The user never clicked anything — the page auto-submitted.

#### Mitigations

**1. CSRF Tokens (most reliable)**
- Server generates a unique, random token per session.
- Embeds it in every form as a hidden field and reads it from a header on state-changing requests.
- On form submit, server checks token matches → attacker's site can't read or forge it (SOP prevents cross-origin reads).

**2. SameSite Cookie Attribute**
```
Set-Cookie: session=abc123; SameSite=Strict; HttpOnly; Secure
```
- `SameSite=Strict` — cookie never sent on cross-site requests.
- `SameSite=Lax` — cookie sent on top-level navigations (link clicks) but not from subresources/forms on other sites.
- Modern browsers default to `Lax`, which prevents most CSRF.

**3. Custom Request Headers**
REST APIs that require `Content-Type: application/json` or a custom `X-Requested-With` header are safe from CSRF — HTML forms can't set these, and a cross-origin XHR that sets them triggers a CORS preflight.

#### CSRF vs CORS
- **CORS** prevents cross-origin *reads* (JS reading the response).
- **CSRF** abuses the fact that browsers *send* requests cross-origin automatically.
- They defend against different things and you need both.

---

### HTTP Security Headers

These are response headers your server sets to tell the browser how to behave securely.

#### Content-Security-Policy (CSP)

**Problem it solves:** XSS (Cross-Site Scripting) — attacker injects malicious script into your page. CSP tells the browser *which sources are allowed* to execute scripts, load images, etc.

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com; img-src *; style-src 'self' 'unsafe-inline'
```

| Directive | Meaning |
|---|---|
| `default-src 'self'` | Only load resources from same origin by default |
| `script-src` | Where scripts can come from |
| `img-src *` | Images from anywhere |
| `'unsafe-inline'` | Allows inline scripts/styles (weakens CSP — avoid if possible) |
| `'nonce-abc123'` | Allow specific inline script with this nonce |

**CSP in report-only mode:** `Content-Security-Policy-Report-Only` — logs violations without blocking. Use this to test a new policy.

#### Strict-Transport-Security (HSTS)
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```
- Tells the browser: "Always use HTTPS for this domain, for the next year."
- `includeSubDomains` — applies to all subdomains.
- `preload` — submit to browser preload lists so even the first visit is HTTPS.
- Prevents SSL-stripping attacks.

#### X-Frame-Options
```
X-Frame-Options: DENY
```
Prevents your page being embedded in an `<iframe>` — stops **clickjacking** (attacker overlays your page in an invisible iframe and tricks users into clicking buttons).
- `DENY` — never allow.
- `SAMEORIGIN` — only allow same origin.
- Replaced by `Content-Security-Policy: frame-ancestors 'none'` in modern usage.

#### X-Content-Type-Options
```
X-Content-Type-Options: nosniff
```
Prevents the browser from **MIME-sniffing** — guessing a file's type and executing it as a script even if the server says it's text/plain. Always set this.

#### Referrer-Policy
```
Referrer-Policy: strict-origin-when-cross-origin
```
Controls how much URL info is included in the `Referer` header when users navigate. Prevents leaking sensitive URL parameters (e.g. tokens in query strings) to third-party sites.

#### Permissions-Policy
```
Permissions-Policy: camera=(), microphone=(), geolocation=(self)
```
Restricts what browser APIs (camera, mic, location) your page and embedded iframes can access. Replaces the old `Feature-Policy` header.

---

## 4. Authorization & Access Control

### RBAC (Role-Based Access Control)

#### What it is
RBAC is an access control model where **permissions are assigned to roles, and users are assigned to roles** — not directly to permissions. This is the most common authorisation model in real-world applications.

```
User  →  Role  →  Permissions  →  Resources
John  →  Admin →  read, write,    /users, /settings
                  delete
Mary  →  Viewer → read            /reports
```

#### How it works in practice

**Database model:**
```
users     roles       user_roles     role_permissions    permissions
------    ------      ----------     ----------------    -----------
id        id          user_id        role_id             id
email     name        role_id        permission_id       name (e.g. "delete:user")
```

**In JWTs:** Roles are embedded as claims so APIs don't need a DB call on every request:
```json
{
  "sub": "user_123",
  "roles": ["admin", "editor"],
  "permissions": ["read:reports", "write:posts"]
}
```

**Middleware check example (pseudo-code):**
```python
@require_role("admin")
def delete_user(user_id):
    ...
```

#### RBAC vs ABAC
- **RBAC** — access based on *role* (who you are). Simple, easy to reason about.
- **ABAC** (Attribute-Based Access Control) — access based on *attributes* (user department, resource owner, time of day). More flexible, more complex. Used in AWS IAM policies.

#### Key interview points
- **Principle of least privilege** — assign the minimum role needed.
- **Role explosion** — if you end up with too many fine-grained roles, consider ABAC.
- Watch for **privilege escalation** — a user shouldn't be able to assign themselves a higher role.

---

### Zero Trust

#### What it is
Zero Trust is a **security philosophy and architecture model**: *never trust, always verify*. No user, device, or service is automatically trusted just because it's on the corporate network.

This contrasts with the old **castle-and-moat** model: once inside the network (VPN or office), you were trusted everywhere. That model is broken — insider threats and lateral movement after a breach are too easy.

#### Core principles

**1. Verify explicitly**
- Authenticate and authorise every request using all available signals: user identity, device health, location, time.
- Use MFA, device compliance checks, and short-lived credentials.

**2. Use least-privilege access**
- Just-in-time access — grant access only when needed, revoke immediately after.
- Scope tokens tightly. Segment networks so a breach in one area can't spread.

**3. Assume breach**
- Design as if attackers are already inside.
- Log everything. Monitor for anomalies.
- Micro-segment the network — services can't talk to each other unless explicitly allowed.

#### How it's implemented
```
Old model:
  [Internet] → [Firewall] → [Trusted internal network — everything OK]

Zero Trust model:
  [Internet] → [Identity-aware proxy] → [continuous auth check] → [specific resource]
                                        ↑
                 every request evaluated: Who? What device? From where? Normal behaviour?
```

- **Identity-aware proxies** (Google BeyondCorp, Cloudflare Access) — sit in front of internal apps, enforce identity verification.
- **mTLS between services** — services prove identity via certificates, not network position.
- **SIEM + anomaly detection** — flag unusual access patterns in real time.

---

### Bastion Host

#### What it is
A **bastion host** (also called a jump server) is a specially hardened server that acts as the **only allowed entry point into a private network**. Engineers SSH into the bastion first, then connect to internal servers from there.

```
                    Internet
                       |
             [SSH] → Bastion Host (public IP, hardened)
                       |
               Private Network (no public IPs)
               ├── App Server
               ├── Database
               └── Internal Services
```

#### Why it's necessary
Without a bastion, you'd have to expose your DB or app servers directly to the internet (public IP) just so engineers can SSH in — massive attack surface. With a bastion, *only one machine* is publicly exposed, and that machine is heavily locked down.

#### How it's hardened
- Only port 22 (SSH) allowed inbound. All other ports blocked.
- Only specific IPs whitelisted (engineer IPs or VPN exit IPs).
- No other software running on it — minimal attack surface.
- All SSH sessions logged and audited.
- SSH key-based auth only — password auth disabled.
- MFA required to log in.

#### Modern alternatives
| Tool | How |
|---|---|
| **AWS SSM Session Manager** | Tunnel through AWS agent — no port 22 open at all, no public IP needed |
| **Google IAP TCP Forwarding** | Identity-aware proxy for SSH — requires Google identity |
| **Teleport** | Access gateway with audit logs, session recording, RBAC |

> Bastions are still common but are being replaced by identity-aware tunnels that don't require any public port.

---

## 5. Identity Provisioning

### SCIM (System for Cross-domain Identity Management)

#### What it is
SCIM is a **standardised REST API and JSON schema for automating user lifecycle management** (provisioning and deprovisioning) across SaaS applications.

Without SCIM: When a new employee joins, an IT admin manually creates accounts in Slack, Jira, GitHub, Salesforce, and 10 other tools. When they leave, same in reverse — and if any gets missed, you have a security gap.

With SCIM: The IdP (Okta, Azure AD) automatically creates, updates, and deactivates accounts in all connected apps the moment HR updates the employee record.

#### How it works

SCIM defines a standard REST API that apps must implement:

```
New employee joins:
  IdP → POST /scim/v2/Users
  {
    "userName": "john.doe@company.com",
    "name": { "givenName": "John", "familyName": "Doe" },
    "emails": [{ "value": "john.doe@company.com" }],
    "active": true,
    "groups": ["engineering", "github-team"]
  }
  App creates the account automatically.

Employee leaves:
  IdP → PATCH /scim/v2/Users/{id}
  { "active": false }
  App deactivates the account immediately.

Employee changes team:
  IdP → PATCH /scim/v2/Users/{id}
  { "groups": ["finance"] }
  App updates group memberships.
```

#### Standard SCIM endpoints
| Endpoint | Purpose |
|---|---|
| `GET /Users` | List users |
| `POST /Users` | Create user |
| `GET /Users/{id}` | Get user by ID |
| `PUT /Users/{id}` | Replace user |
| `PATCH /Users/{id}` | Partial update |
| `DELETE /Users/{id}` | Delete user |
| `GET /Groups` | List groups |
| `POST /Groups` | Create group |

#### Key interview points
- SCIM handles **provisioning** (creating accounts). SSO (SAML/OIDC) handles **authentication** (logging in). They work together.
- SCIM is push-based — the IdP pushes changes to apps.
- Without SCIM, departing employee accounts often linger → security risk.

---

## 6. Credential & Password Security

### How Databases Store Passwords

#### What NOT to do

| Approach | Problem |
|---|---|
| Plain text | Single DB breach = all passwords exposed |
| Reversible encryption | If key is stolen, all passwords decryptable |
| MD5 / SHA-256 (unsalted) | Rainbow table attacks — precomputed hashes of common passwords crack them instantly |
| SHA-256 with salt | Better, but SHA-256 is too *fast* — billions of guesses per second on GPU |

#### The correct approach: Slow hashing + per-user salt

**Why salting?**
A **salt** is a random string generated uniquely for each user. It's appended to the password before hashing. This means:
- Two users with the same password get different hashes.
- Attackers can't use precomputed rainbow tables — they'd need a separate table per user.

**Why slow hashing (bcrypt / Argon2 / scrypt)?**
Fast hash functions (SHA-256) can run **billions of times per second** on modern GPUs. Bcrypt is designed to run at a cost you configure — e.g. ~100ms per hash. That's fine for one login, but makes brute-forcing billions of guesses impractical.

#### Step-by-step: how a password is stored

**Registration:**
```
user enters: "mypassword123"

1. Generate random salt:    "a7f3k2m9"
2. Compute hash:            bcrypt("mypassword123" + "a7f3k2m9", cost=12)
                            → "$2b$12$a7f3k2m9...hashedvalue..."
3. Store in DB:             { user_id: 42, password_hash: "$2b$12$..." }
   (salt is embedded in the bcrypt hash string — no separate column needed)
```

**Login:**
```
user enters: "mypassword123"

1. Fetch stored hash from DB for that user.
2. Extract the salt from the hash string.
3. Compute: bcrypt("mypassword123" + extracted_salt, same_cost)
4. Compare result to stored hash.
   → Match ✅ → log in
   → No match ❌ → reject
```

**The server never decrypts** — it re-hashes and compares. There is no way to "reverse" a bcrypt hash.

#### Argon2 — the modern recommendation
- Winner of the 2015 Password Hashing Competition.
- Three variants: **Argon2id** (recommended) — resists both GPU and side-channel attacks.
- Configurable: memory cost, iteration count, parallelism — all increase cracking difficulty.
- Preferred over bcrypt for new systems.

#### Key interview points
- bcrypt output already includes the salt, so you only store one column.
- Cost factor (bcrypt `rounds`, Argon2 `iterations`) should be tuned so hashing takes ~100–300ms.
- **Pepper:** An additional server-side secret mixed into the hash before storing — even if DB is leaked, attacker needs the pepper too. Stored in application config / vault, not in DB.

---

## Quick Reference: Which Protocol for What?

| Need | Use |
|---|---|
| Third-party app access (delegated) | OAuth 2.0 + PKCE |
| User identity (who logged in) | OIDC (ID Token) |
| Enterprise SSO (legacy/corporate) | SAML 2.0 |
| Modern SSO for web/mobile | OIDC |
| Modern passwordless login | Passkeys (FIDO2/WebAuthn) |
| Auto-provision users in SaaS apps | SCIM |
| Secure secrets & DB credentials | HashiCorp Vault / AWS Secrets Manager |
| Service-to-service identity/encryption | mTLS + X.509 |
| Transport encryption | TLS 1.3 |
| Access control rules in your app | RBAC (or ABAC for fine-grained) |
| Secure internal infrastructure access | Bastion / SSM Session Manager |
| Network security with no implicit trust | Zero Trust architecture |

---

## One-liners to Memorize

- **TLS** = encrypted transport; asymmetric for handshake, symmetric for data.
- **CA** = trusted signer of certificates; chain of trust from Root → Intermediate → Leaf.
- **X.509** = certificate format for identity/TLS; **Vault** = secrets at rest; both are needed.
- **OAuth 2.0** = authorisation (access), not authentication (identity).
- **OIDC** = OAuth 2.0 + identity (adds ID Token with `sub`, `email`, `iss`, `exp`).
- **PKCE** = OAuth for public clients; `code_verifier` prevents stolen code from being used.
- **SAML** = XML-based enterprise SSO; IdP signs assertions; SP validates them.
- **SSO** = one login at IdP; all apps get tokens silently on subsequent visits.
- **JWT** = signed self-contained token; verify signature + `exp` + `iss` + `aud`.
- **MFA** = handled at IdP; `acr` claim says what factors were used; step-up via `acr_values`.
- **Passkeys** = FIDO2 passwordless; private key on device, public key on server; phishing-resistant.
- **CORS** = browser policy; server opts-in via response headers; preflight for complex requests.
- **CSRF** = browser auto-sends cookies cross-site; mitigate with CSRF token + SameSite cookie.
- **CSP** = whitelist content sources; primary XSS defence in-depth layer.
- **SCIM** = REST API for automated user provisioning; SCIM ≠ SSO — they complement each other.
- **Zero Trust** = verify every request; assume breach; least privilege; no implicit network trust.
- **Bastion** = hardened jump server; single public entry point; modern alt = SSM/IAP.
- **RBAC** = user → role → permissions; least privilege; roles in JWT claims.
- **Passwords in DB** = bcrypt/Argon2id with per-user salt; never plain, MD5, or fast hash.

