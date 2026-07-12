# Authorization and access control

## Contents
1. [General architecture](#general-architecture)
2. [JWT tokens](#jwt-tokens)
3. [Password authentication](#password-authentication)
4. [OAuth/OIDC](#oauthoidc)
5. [LDAP](#ldap)
6. [API keys](#api-keys)
7. [Trusted Headers (proxy authorization)](#trusted-headers)
8. [SCIM provisioning](#scim-provisioning)
9. [Roles and permissions](#roles-and-permissions)
10. [Groups and Access Grants](#groups-and-access-grants)
11. [Rate Limiting](#rate-limiting)
12. [Security: production checklist](#security)

---

## General architecture

Open WebUI supports 5 authentication methods, which can be combined:

```
User → [Password | OAuth | LDAP | API key | Trusted Header] → JWT → API
```

All methods ultimately issue a JWT token, which is used to authorize API requests.

Key file: `backend/open_webui/routers/auths.py`

---

## JWT tokens

### How they're structured

- **Algorithm**: HS256
- **Secret**: `WEBUI_SECRET_KEY` (CRITICAL: set this in production!)
- **Payload structure**: `{id, exp, jti}`
  - `id` — user UUID
  - `exp` — expiration time
  - `jti` — unique token identifier (for revocation)
- **Lifetime**: configured via `JWT_EXPIRES_IN` (format: `"30m"`, `"24h"`, `"7d"`)

### Cookie storage

```env
WEBUI_SESSION_COOKIE_SAME_SITE=lax    # lax | strict | none
WEBUI_AUTH_COOKIE_SECURE=true          # true for HTTPS
WEBUI_AUTH_COOKIE_HTTP_ONLY=true       # protection against XSS
```

### Token revocation

If Redis is configured, Open WebUI supports token revocation via a JTI blacklist:
- On logout, the token is added to the Redis blacklist
- On each request, it checks whether the JTI has been revoked

Without Redis, token revocation doesn't work — the token remains valid until `exp`.

---

## Password authentication

### Endpoints

```
POST /api/v1/auths/signin     — sign in
POST /api/v1/auths/signup     — sign up
```

### Password hashing

- **bcrypt** is used (limitation: only the first 72 bytes of the password)
- Optional password strength validation:

```env
ENABLE_PASSWORD_VALIDATION=true
PASSWORD_VALIDATION_REGEX_PATTERN="^(?=.*[A-Z])(?=.*\d).{8,}$"
```

### Registration management

```env
ENABLE_SIGNUP=true                    # allow registration
ENABLE_PASSWORD_AUTH=true             # allow password login
ENABLE_INITIAL_ADMIN_SIGNUP=true      # first user = admin
DEFAULT_USER_ROLE=pending             # default role: pending | user
```

If `DEFAULT_USER_ROLE=pending`, new users won't be able to do anything until an administrator approves them and changes their role to `user`.

---

## OAuth/OIDC

### Supported providers

Open WebUI supports out of the box:
- **Google**
- **Microsoft**
- **GitHub**
- **Feishu (Lark)**
- **Any OIDC-compatible provider** (Keycloak, Auth0, Okta, etc.)

### Setup

Each provider is configured via environment variables. Example for Google:

```env
ENABLE_OAUTH_SIGNUP=true
OAUTH_MERGE_ACCOUNTS_BY_EMAIL=true

# Google
GOOGLE_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=<your-client-secret>
GOOGLE_OAUTH_SCOPE="openid email profile"
GOOGLE_REDIRECT_URI=https://your-domain.com/oauth/google/callback
```

### Example for a generic OIDC provider (Keycloak)

```env
OAUTH_CLIENT_ID=open-webui
OAUTH_CLIENT_SECRET=<your-client-secret>
OAUTH_PROVIDER_NAME=Keycloak
OAUTH_OPENID_CONFIG_URL=https://keycloak.example.com/realms/master/.well-known/openid-configuration
OAUTH_SCOPES="openid email profile"
OAUTH_REDIRECT_URI=https://your-domain.com/oauth/oidc/callback
```

### Important flags

```env
OAUTH_MERGE_ACCOUNTS_BY_EMAIL=true    # merge accounts by email
OAUTH_MAX_SESSIONS_PER_USER=10        # active session limit
ENABLE_OAUTH_TOKEN_EXCHANGE=false     # OAuth token exchange
ENABLE_OAUTH_ID_TOKEN_COOKIE=false    # ID token in cookie
```

### Pitfall: account merging

If `OAUTH_MERGE_ACCOUNTS_BY_EMAIL=true`, a user with the same email from different providers will be treated as a single account. This is convenient, but creates a risk: if an attacker controls the email at another provider, they gain access to the existing account.

---

## LDAP

### Setup

```env
ENABLE_LDAP=true
LDAP_SERVER_HOST=ldap.example.com
LDAP_SERVER_PORT=389                  # 636 for LDAPS
LDAP_USE_TLS=true
LDAP_BIND_DN="cn=admin,dc=example,dc=com"
LDAP_BIND_PASSWORD=<your-password>
LDAP_SEARCH_BASE="ou=users,dc=example,dc=com"
LDAP_SEARCH_FILTERS="(&(objectClass=person)(uid={username}))"
LDAP_ATTRIBUTE_FOR_MAIL=mail
LDAP_ATTRIBUTE_FOR_USERNAME=uid
```

### Endpoint

```
POST /api/v1/auths/ldap    — sign in via LDAP
```

### Group mapping

LDAP groups can be mapped to Open WebUI roles. Configured via additional LDAP attributes.

---

## API keys

### Generation

```
POST /api/v1/auths/api-key    — create a key (requires authorization)
```

### Format

```
sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Format: `sk-` prefix + 32 hex characters.

### Usage

```bash
curl -H "Authorization: Bearer <your-api-key>" \
     https://your-domain.com/api/v1/chats
```

### Notes

- Keys are stored in the `api_key` table with `expires_at` and `last_used_at` fields
- Key rotation is supported
- An expiration time can be set
- An administrator can revoke other users' keys

---

## Trusted Headers

Used when running behind a reverse proxy (nginx, Traefik, Cloudflare Access, etc.) that has already performed authentication.

```env
WEBUI_AUTH_TRUSTED_EMAIL_HEADER=X-Forwarded-Email
WEBUI_AUTH_TRUSTED_NAME_HEADER=X-Forwarded-Name
WEBUI_AUTH_TRUSTED_GROUPS_HEADER=X-Forwarded-Groups
```

**Warning**: make sure these headers cannot be spoofed by the client! Configure the proxy so it overwrites these headers.

---

## SCIM provisioning

SCIM 2.0 allows automatically creating/updating/deleting users from corporate IdPs (Azure AD, Okta).

```env
ENABLE_SCIM=true
SCIM_AUTH_HEADER=Authorization
SCIM_AUTH_TOKEN=<your-token>
```

Endpoints: `backend/open_webui/routers/scim.py`

---

## Roles and permissions

### Hierarchy

```
admin > user > pending
```

### What each role can do

| Action | admin | user | pending |
|----------|-------|------|---------|
| Chat with models | + | + | - |
| Upload files | + | + | - |
| Create functions | + | - (configurable) | - |
| Manage models | + | - | - |
| Manage users | + | - | - |
| System settings | + | - | - |
| Create knowledge bases | + | + | - |

### Model access

An administrator can restrict access to specific models:
- By user
- By group
- `BYPASS_MODEL_ACCESS_CONTROL=true` — disable model access checks

### Groups

Groups let you combine users and assign permissions in bulk. Created via API or the admin panel.

---

## Groups and Access Grants

### Access Grants

A fine-grained access control system. Allows sharing resources:

```
resource_type: "knowledge" | "model" | "chat" | ...
target_type: "user" | "group"
access_level: "read" | "write" | "admin"
```

This allows, for example, sharing a knowledge base with a particular group as read-only.

---

## Rate Limiting

Protection against password brute-forcing:
- **Limit**: 5 login attempts per 3 minutes (per IP)
- Configured in `auths.py` code

---

## Security

### Production checklist

1. **REQUIRED**: set `WEBUI_SECRET_KEY` — without it, the JWT secret is generated randomly on every restart, invalidating all sessions
2. Enable `WEBUI_AUTH_COOKIE_SECURE=true` if using HTTPS
3. Set `WEBUI_SESSION_COOKIE_SAME_SITE=strict` for maximum protection
4. If using Trusted Headers — make sure the proxy overwrites them
5. Set `DEFAULT_USER_ROLE=pending` so new users require approval
6. Enable `ENABLE_PASSWORD_VALIDATION=true` to require strong passwords
7. Configure Redis to support token revocation
8. Use `ENABLE_AUDIT_LOGS_FILE=true` for auditing
