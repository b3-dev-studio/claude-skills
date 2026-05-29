---
artifact: blueprint
name: oauth-integration
version: 1
tags: [oauth, auth, authentication, sso, identity, provider]
---

# Blueprint: OAuth Integration

## When to Use

Apply this blueprint when building a feature that:
- Adds authentication via an external OAuth 2.0 provider (Google, GitHub, Slack, Microsoft, etc.)
- Requires an authorization code flow with a callback URL
- Needs to map an external identity to a local user record

Do not apply when: building an OAuth *server* (this blueprint covers the client/consumer side only), the feature is API key or password authentication, or the provider uses a non-standard auth flow.

---

## Canonical Component Graph

```
C1 — OAuth Controller
  │  Owns: /auth/{provider} initiation route,
  │        /auth/{provider}/callback route,
  │        redirecting to provider, handling callback response
  │  Does not own: token exchange, user mapping, session creation
  │
  ▼
C2 — OAuth Service
  │  Owns: state/PKCE parameter generation and validation,
  │        authorization URL construction,
  │        authorization code → token exchange,
  │        fetching user profile from provider
  │  Does not own: HTTP routing, local user management, session
  │
  ▼
C3 — Identity Mapper
     Owns: mapping the OAuth profile to a local user record
           (find existing user by provider ID, or create new user)
     Does not own: OAuth mechanics, session creation
```

Session or JWT issuance happens after C3 returns a local user — in C1, using the existing auth session mechanism. C3 must not issue sessions.

---

## Required Interfaces (abstract)

**C1 — OAuth Controller**
- Initiation route: generates state parameter via C2, stores it, redirects to provider authorization URL
- Callback route: validates state parameter, exchanges code via C2, maps identity via C3, issues session
- Handles provider errors returned in callback query params (error, error_description)
- Never exposes raw OAuth tokens in responses or redirects

**C2 — OAuth Service**
- `getAuthorizationUrl(provider, state)` → redirect URL string
- `exchangeCode(provider, code, state)` → typed token response
- `getUserProfile(provider, accessToken)` → normalised provider profile object
- Validates state parameter before code exchange (CSRF protection)
- Stateless — does not store tokens or sessions

**C3 — Identity Mapper**
- `findOrCreate(providerProfile)` → local User object
- Looks up existing user by provider ID and provider name
- Creates new user if none found, using profile data for initial fields
- Returns the local User — never the provider profile directly

---

## Adaptation Questions

The bootstrap generator must answer these by surveying the codebase.

1. Is there an existing OAuth integration or auth flow to trace as a representative example?
2. What OAuth library is in use, if any? (passport, oauth4webapi, custom, provider SDK)
3. How and where is the OAuth state parameter stored between initiation and callback? (session, Redis, signed cookie)
4. How is the existing session or JWT issued after successful login? Trace an existing auth success path.
5. What does the local User model look like? How are OAuth provider IDs stored? (separate table, columns on user, JSON field)
6. Where are OAuth client credentials (client_id, client_secret) stored and accessed?
7. What is the redirect URL registration requirement for the provider?
8. How are auth errors currently surfaced to the end user? (error page, redirect with query param, JSON response)
9. Are OAuth access/refresh tokens stored after login? If so, where and how?
10. Is PKCE required or used? Does the existing auth flow use it?

---

## Known Anti-Patterns

These must appear in the bootstrap's Constraints section, confirmed against the actual codebase.

- **Missing state validation:** not verifying the state parameter on callback allows CSRF attacks. State must be validated before code exchange, every time.
- **Token exposure:** redirecting with tokens in URL fragments or returning them in JSON responses. Tokens must never appear in URLs or client-facing responses.
- **Plaintext token storage:** storing access or refresh tokens without encryption. If tokens must be stored, they must be encrypted at rest.
- **C3 issuing sessions:** the identity mapper creates or modifies sessions directly. Session issuance belongs in C1 after C3 returns a local user.
- **Hardcoded provider config:** client_id, client_secret, or redirect URIs embedded in code. All provider config must come from environment or a secrets store.
- **Single callback route for all providers:** one route with conditional branching on provider name. Each provider should have its own route or the dispatcher should be explicit.
- **No error handling for provider failures:** assuming the callback always contains a valid code. Providers return error params on denial or failure — these must be handled gracefully.
