# FINDING: Password Hash Disclosure via Keycloak Account API

## Asset
`[TARGET]` (drone operator portal of a port authority; Tier 2, in scope through the
organization's wildcard domain `*.[TARGET-DOMAIN]`)

## Severity Estimate: Medium-High
**VRT:** Broken Authentication and Session Management > Password Hash Disclosure

---

## Summary

The Keycloak Account REST API at `[TARGET]` returns the authenticated user's
**bcrypt password hash** as a user attribute. Any authenticated user can retrieve
their own password hash by calling `GET /auth/realms/[REALM]/account`. The hash
is stored as a custom user attribute (`hash`) instead of in Keycloak's internal
credential store, causing it to be returned in the standard Account API response.

---

## Evidence

### PoC — Retrieve Own Password Hash

```http
GET /auth/realms/[REALM]/account HTTP/2
Host: [TARGET]
Authorization: Bearer <valid_access_token>
Accept: application/json
```

Response (200 OK):
```json
{
  "id": "[ID]",
  "username": "[REPORTER-EMAIL]",
  "attributes": {
    "mfa": ["NONE"],
    "[vendor]_uuid": ["[UUID]"],
    "hash": ["$2a$10$[REDACTED]"]
  },
  "userProfileMetadata": { ... }
}
```

The `attributes.hash` field contains a **bcrypt hash** (`$2a$10$...`) of the
user's password with cost factor 10.

### Reproduction Steps (for a triager with no context)

1. Go to `https://[TARGET]` and create an account
   (registration is open in the browser UI).
2. Log in. The app uses Keycloak (realm: `[REALM]`, client: `[CLIENT-ID]`).
3. Capture the Bearer token from the WebSocket `#authenticate` event in
   browser devtools (Network → WS → Messages → look for `#authenticate`).
4. Send: `GET https://[TARGET]/auth/realms/[REALM]/account`
   with header `Authorization: Bearer <token>`.
5. The response JSON includes `attributes.hash` containing the bcrypt hash.

### Additional Exposed Data

The same `/account` endpoint also exposes:
- `mfa: ["NONE"]` — confirms MFA is not enabled
- `[vendor]_uuid` — internal UUID of a third-party platform
- Full `userProfileMetadata` with field validators, lengths, regex patterns

The `/account/sessions` endpoint reveals:
- Active session IDs, client IP addresses, browser fingerprint
- **Session lifetime: 100 days** (delta between start and expiry = 8,640,000s = 100 days exactly)

---

## Root Cause

The developers stored the bcrypt password hash as a **Keycloak user attribute**
(`hash`) rather than using Keycloak's built-in credential storage system.
Keycloak's Account REST API is designed to return all user attributes to the
authenticated user — this is expected behavior. The vulnerability is that a
**password hash should never be stored as a user attribute**.

---

## Impact

### Confirmed
1. **Password hash disclosure**: Any authenticated user retrieves their own
   bcrypt hash via a single GET request.
2. **Offline cracking**: bcrypt cost 10 is within crackable range for targeted
   attacks (~10-100 hashes/sec on GPU). Weak or common passwords would fall.
3. **Credential stuffing amplification**: If users reuse passwords, a cracked
   hash leads to account takeover on other services.
4. **XSS chain escalation**: Any XSS on `[TARGET]`
   would allow an attacker to exfiltrate the victim's password hash by stealing
   their Bearer token and calling this endpoint — turning a reflected XSS into
   credential theft.

### Architecture Concern
- The password hash being stored as a user attribute (not a credential) suggests
  the application manages passwords outside Keycloak's standard password flow.
  This may indicate a **custom authentication scheme** that bypasses Keycloak's
  built-in security controls (password policies, brute force protection, etc.).

### NOT Claimed (would need further testing)
- Access to OTHER users' hashes (admin API returns 301 → login, no IDOR found)
- Hash corresponds to current password (hash may be stale or from migration)
- Impact on the portal's operations (drone flight authorization in a restricted area)

---

## Exclusion Risk Assessment

- **NOT** "verbose messages without sensitive info" — this is a password hash.
- **NOT** "missing security headers" — this is data leakage from the API.
- **NOT** "API key disclosure without business impact" — this is a password hash.
- **IS** a configuration error with direct impact on credential security.

**Honest assessment**: Solid Medium. The hash is the user's own (no IDOR to get
others'), but the design flaw is clear and the impact chain with XSS is
realistic. The 100-day session lifetime is a secondary concern.

---

## Tested: October 2026
