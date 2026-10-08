# Intigriti Report — Password Hash Disclosure via Keycloak Account API

## Title
Bcrypt password hash exposed via Keycloak Account API to any authenticated user ([TARGET])

## Domain / URL
[TARGET]

## Severity
Medium

## VRT Category
Broken Authentication and Session Management > Sensitive Data Exposure

---

## Description

The Keycloak Account REST API at `[TARGET]` exposes the authenticated user's **bcrypt password hash** as a custom user attribute. Any user who creates an account (self-registration is open) can retrieve their own password hash with a single GET request.

The root cause is that the password hash is stored as a Keycloak **user attribute** (`hash`) instead of in Keycloak's internal credential storage. The Account API returns all user attributes by design, so the hash is included in the response.

Additionally, the API does not validate or sanitize user-controlled input: the `PUT /api/users/me` endpoint stores arbitrary HTML in `firstName`/`lastName` fields without sanitization. While the user portal's frontend HTML-encodes the output (preventing execution there), the raw HTML is stored server-side and returned unescaped by the API — the rendering behavior in other contexts (admin panel, email notifications) could not be verified.

---

## Steps to reproduce

### Prerequisites
- Browser (Firefox/Chrome)
- Burp Suite (or any HTTP intercepting proxy)

### Step 1 — Create an account
1. Navigate to `https://[TARGET]`
2. Click "Create account" and register with valid credentials
3. Verify email and log in

### Step 2 — Obtain Bearer token
4. With Burp intercepting, log in to the portal
5. Go to Burp → Proxy → WebSocket history
6. Find the `#authenticate` event — the `data` field contains the Bearer JWT

**[Screenshot: WebSocket #authenticate event showing the Bearer token]**

### Step 3 — Retrieve password hash
7. In Burp Repeater, send:

```http
GET /auth/realms/[REALM]/account HTTP/2
Host: [TARGET]
Authorization: Bearer <token_from_step_6>
Accept: application/json
```

8. The response (200 OK) includes the bcrypt hash in `attributes.hash`:

```json
{
  "id": "[ID]",
  "username": "USER@EXAMPLE.COM",
  "attributes": {
    "mfa": ["NONE"],
    "[vendor]_uuid": ["[UUID]"],
    "hash": ["$2a$10$[REDACTED]"]
  }
}
```

**[Screenshot: Burp Repeater showing the GET request and 200 response with the bcrypt hash highlighted in the attributes.hash field]**

That's it. The hash is exposed in one request.

---

## Additional finding — Missing input sanitization

The `PUT /api/users/me` endpoint accepts and stores arbitrary HTML in name fields without validation:

```http
PUT /api/users/me HTTP/2
Host: [TARGET]
Authorization: Bearer <token>
[CUSTOM-HEADER]: <company_uuid>
Content-Type: application/json

{
  "uniqueIdentifier":"<user_uuid>",
  "firstName":"Test<img src=x onerror=alert(1)>",
  "lastName":"User",
  "email":"<email>",
  "contact":{"firstName":"Test<img src=x onerror=alert(1)>","lastName":"User","email":"<email>"},
  "status":"ACTIVE","hasCurrentLicenseAgreed":true,"hasCurrentPrivacyPolicyAgreed":true,
  "country":{"uniqueIdentifier":"[COUNTRY-ID]","code":"[CC]"},
  "preferences":{"locale":"[LOCALE]","timeZone":"[TZ]","dateFormat":"dd/MM/yyyy","timeFormat":"HH:mm","defaultCompanyUUID":"<company_uuid>"},
  "privacySettings":{"allowContact":true},"mfa":"NONE","version":2
}
```

Response (200): the HTML is stored and returned raw:
```json
{
  "contact": {
    "firstName": "Test<img src=x onerror=alert(1)>",
    "fullName": "Test<img src=x onerror=alert(1)> User"
  }
}
```

**[Screenshot: Burp showing PUT request with HTML payload and 200 response confirming storage]**

**[Screenshot: Browser showing the welcome message with the raw HTML visible as text: "[WELCOME-MESSAGE], Test<img src=x onerror=alert(1)> User"]**

The user portal's Aurelia frontend HTML-encodes the output, preventing execution in the user-facing UI. I did not have access to the admin panel to verify its rendering behavior. The name was reverted to clean values immediately after testing.

**[Screenshot: Burp showing PUT with clean name values and 200 response confirming revert]**

The feedback endpoint (`POST /api/notifications/feedback/`) also accepts arbitrary HTML in all fields (message, replyTo, category) without validation, returning 201 Created for each.

---

## Impact

**Password hash disclosure (confirmed):**
- The bcrypt hash (cost factor 10) enables offline password cracking. Weak or common passwords would fall to GPU-based attacks.
- If users reuse passwords across services, a cracked hash enables account takeover elsewhere.
- The hash is exposed to the browser/client, meaning any client-side vulnerability (XSS in any context, browser extension, compromised device) could exfiltrate it.
- Self-registration is open — no barrier to exploitation.

**Missing input sanitization (confirmed server-side, client rendering varies):**
- The API stores and returns raw HTML without encoding.
- The user portal escapes it during rendering (observed).
- The admin panel, email templates, and other rendering contexts were not accessible for testing.

---

## Root cause

The password hash is stored as a Keycloak **user attribute** (`attributes.hash`) rather than in the Keycloak internal credential store. The Keycloak Account REST API is designed to return all user attributes — this is expected Keycloak behavior. The misconfiguration is storing the hash as an attribute.

---

## Remediation

1. **Remove the `hash` attribute** from Keycloak user profiles. If the hash is needed for application logic, move it to Keycloak's internal credential store or a backend database that is never exposed to client-facing APIs.
2. **Add server-side input validation** on name fields and feedback — reject or strip HTML tags before storage.
