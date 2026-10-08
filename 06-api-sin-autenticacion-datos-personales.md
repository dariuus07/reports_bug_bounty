# Intigriti Report — Unauthenticated API exposes PII of 158 university users

**Program:** [UNIVERSITY] VDP
**Asset:** [TARGET] ([IP])
**VRT:** Broken Access Control > IDOR / Broken Object Level Authorization
**Severity:** High

---

## Title

Unauthenticated REST API on [TARGET] exposes personal data (names + emails) of 158 university users

---

## Description

The application (a university course auction/selection tool) at `[TARGET]` exposes a Django REST Framework API at `/api/`. Several endpoints return sensitive data without requiring any authentication:

- `/api/persons` returns **all 158 user records** including full names, institutional email addresses, and user roles
- `/api/auctionItems` returns 477 course items with 49 student email addresses embedded in vote records

The authentication is inconsistently applied: `/api/votes` returns 403 (requires auth), while `/api/persons` and `/api/auctionItems` return full data to any unauthenticated request.

---

## Steps to Reproduce

### PoC 1 — Full user list without authentication

1. Open a browser and navigate to:
   ```
   https://[TARGET]/api/persons
   ```
2. The page returns a JSON array with **158 user records**, each containing:
   - `first_name`, `last_name` (full real names)
   - `email` (institutional addresses: staff and student domains of the university)
   - `type` (admin/student/normal)
   - `created_at`, `modified_at`, `created_by`, `modified_by`
   - Individual auction settings and vote data

3. The response includes 14 admin/staff accounts with staff-domain emails and 121 student accounts with student-domain emails.

### PoC 2 — API root enumeration

1. Navigate to:
   ```
   https://[TARGET]/api/
   ```
2. The Django REST Framework browsable API lists all available endpoints:
   ```json
   {
     "persons": "https://[TARGET]/api/persons",
     "auctions": "https://[TARGET]/api/auctions",
     "auctionItems": "https://[TARGET]/api/auctionItems",
     "votes": "https://[TARGET]/api/votes"
   }
   ```

### PoC 3 — Auction items with student emails

1. Navigate to:
   ```
   https://[TARGET]/api/auctionItems
   ```
2. Returns 477 auction items. Vote records embedded in items contain student email addresses (49 unique student emails).

### Inconsistent authorization

| Endpoint | Auth required? | Status |
|----------|---------------|--------|
| `/api/persons` | No | 200 — returns all 158 users |
| `/api/auctionItems` | No | 200 — returns 477 items + votes |
| `/api/votes` | Yes | 403 Forbidden |
| `/api/auctions` | N/A | 500 Server Error |

---

## Impact

### Confirmed

- **Personal data exposure**: An unauthenticated attacker obtains the full names and institutional email addresses of 158 university users (staff and students). This is personal data under the applicable data protection law and GDPR.
- **Role enumeration**: The `type` field distinguishes admin accounts from student accounts, identifying high-value targets.
- **Student course preferences**: Vote records in `/api/auctionItems` reveal which students voted for which courses, which could be considered sensitive academic data.

### Potential (not verified)

- The exposed emails could be used for targeted phishing campaigns against university staff and students.
- Admin accounts identified by role could be targeted for credential attacks on other university services.

---

## Remediation

1. Add authentication to all API endpoints (`/api/persons`, `/api/auctionItems`) — the application already enforces auth on `/api/votes`, so the mechanism exists.
2. If public API access is needed for the course auction functionality, restrict the response fields to only what's necessary (exclude email, type, timestamps).
3. Consider whether this application is still actively used — if not, take it offline.

---

## Note on prior testing

During the review of the exposed data, I noticed test accounts created by other researchers (e.g., accounts with non-university email domains). This vulnerability may have been previously reported. However, I am submitting this report because the PII remains exposed and accessible.
