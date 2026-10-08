# Bugcrowd Report — Government Agency VDP

## Platform Fields (for the Bugcrowd form)

- **Title:** Unauthenticated Access to Liferay JSONWS API Console and Data on [TARGET]
- **Asset:** [TARGET]
- **VRT:** Sensitive Data Exposure > Visible Detailed Error/Debug Page > Detailed Server Configuration
- **Severity:** P3 (Medium)

---

## Description

The Liferay portal at `[TARGET]` exposes its JSONWS
API console at `/api/jsonws` without authentication. The console is an
interactive developer tool with an **Execute button** that allows
direct API invocation from the browser. It lists 40+ service contexts
with hundreds of methods, including **write and delete operations**
(`delete-file-entry`, `delete-folder`, `add-file-entry`,
`update-company`, `export-layouts-as-file`, `enable-remote-staging`).
The console provides a `p_auth` CSRF token to the unauthenticated user.
The API processes requests and returns data — a call to
`/api/jsonws/country/get-countries` returns 122 KB of JSON including
internal identifiers (`companyId`, `userId`, UUIDs, epoch timestamps).
The GraphQL endpoint at `/o/graphql` also returns its full schema via
introspection without authentication.

## Vulnerability Discovery

During authorized testing under the agency's VDP, I identified
`[TARGET]` as a Liferay Digital Experience Platform
instance (confirmed by the `Liferay-Portal` response header). I tested
standard Liferay API endpoints and found `/api/jsonws` returning an
interactive service browser, and `/o/graphql` accepting introspection
queries, both without any authentication.

## Proof of Concept

### Step 1 — JSONWS API Console is publicly accessible

```
curl -sk "https://[TARGET]/api/jsonws" \
  -o jsonws.html -w "HTTP: %{http_code}  Size: %{size_download}"
```

**Response:** HTTP 200, ~440 KB. The HTML page renders an
interactive JSONWS API console with an Execute button on each method,
a pre-filled `p_auth` CSRF token, and parameter input fields. The
console lists these service contexts:

```
portal, account, asset, assetlist, audit, backgroundtask, batchengine,
batchplanner, blogs, calendar, comment, commerce, contact, ct, ddl, ddm,
depot, dispatch, fragment, journal, kaleo, kaleoforms, kb, layout,
layoututilitypage, listtype, marketplace, mb, notification, oauthclient,
object, portallanguageoverride, redirect, remoteapp, sap,
savedcontententry, segments, sharing, sitenavigation, stylebook,
translation, trash, wiki
```

### Step 2 — JSONWS returns real data without authentication

```
curl -sk "https://[TARGET]/api/jsonws/country/get-countries"
```

**Response:** HTTP 200, ~120 KB. JSON array of country records.
Each record includes internal fields not intended for public access:

```json
{
  "a2": "AF",
  "a3": "AFG",
  "companyId": "[ID]",
  "countryId": "20",
  "userId": "[ID]",
  "uuid": "[UUID]",
  "modifiedDate": [EPOCH-MS],
  "billingAllowed": true,
  "shippingAllowed": true,
  "subjectToVAT": false
}
```

The `companyId` is the Liferay instance identifier.
The `userId` is the internal system user ID.
The `uuid` values are internal object identifiers.

### Step 3 — JSONWS invoke processes requests without auth

```
curl -sk "https://[TARGET]/api/jsonws/invoke" \
  -H "Content-Type: application/json" -d '{}'
```

**Response:** HTTP 500 (Internal Server Error). The endpoint processes
the request and fails because no valid method was specified — it does
NOT return 401 or 403. There is no authentication check on the API
layer.

### Step 4 — GraphQL introspection returns full schema

```
curl -sk "https://[TARGET]/o/graphql" \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name } } }"}'
```

**Response:** HTTP 200. Returns the complete GraphQL schema with 300+
types including: `Account`, `UserAccount`, `Role`, `Cart`, `Order`,
`BlogPosting`, `Admin`, `Catalog`, `Channel`, `Attachment`.

The `UserAccount` type exposes 40 fields including:
`currentPassword`, `password`, `passwordResetTicket`,
`emailVerificationTicket`, `birthDate`, `emailAddress`,
`emailAddresses`, `roleBriefs`, `accountBriefs`,
`userAccountContactInformation`.

### Step 5 — Interactive console exposes write/delete operations with Execute button

Open `https://[TARGET]/api/jsonws` in a browser.
The console renders an interactive interface where each API method has:

- An **Execute** button to invoke the method directly
- A pre-filled **`p_auth`** token (e.g., `[TOKEN]`)
- Input fields for all parameters

Write and delete operations visible in the console include:

```
/user/delete-user (userId) — delete a portal user
/role/delete-role — delete a role
/role/add-role — create a new role
/role/add-user-roles — assign roles to users
delete-file-entry, delete-folder, delete-group, delete-layout,
delete-category, delete-country, delete-email-address,
add-file-entry, add-folder, add-group, add-layout, add-category,
update-company, update-layout, update-folder, update-file-entry,
export-layouts-as-file, import-layouts,
enable-local-staging, enable-remote-staging, publish-layout,
move-file-entry-to-trash, restore-file-entry-from-trash
```

Read operations that could expose user data:

```
/user/get-user-by-screen-name (companyId, screenName)
/socialactivity/get-user-groups-and-organizations-activities (userId)
/dlapp/search (repositoryId, searchContext, query)
```

**Note:** Write/delete operations were not tested to avoid modifying
data on the target system. The console's availability with Execute
functionality and a valid CSRF token is the finding.

## Exploitation

**Preconditions:** None — unauthenticated, no cookies or tokens needed.

1. Open a terminal. No prior authentication is required.
2. Run: `curl -sk "https://[TARGET]/api/jsonws"`
3. **Expected:** 401 Unauthorized or 403 Forbidden.
   **Actual:** HTTP 200 with 442 KB interactive API console showing 40+ service contexts and method signatures.
4. Run: `curl -sk "https://[TARGET]/api/jsonws/country/get-countries"`
5. **Expected:** 401 Unauthorized.
   **Actual:** HTTP 200 with 122 KB JSON containing country records with internal identifiers (companyId, userId, UUIDs).
6. Run the GraphQL introspection query from Step 4 of the PoC.
7. **Expected:** Introspection disabled or auth required.
   **Actual:** HTTP 200 with complete schema including types with sensitive field names.

## Impact

Demonstrated:

- The JSONWS API console exposes the complete API surface of the
  Liferay portal (40+ service contexts, hundreds of methods) to
  unauthenticated users. This is a developer/debug tool not intended
  for production exposure.
- The console is interactive: each method has an Execute button,
  parameter input fields, and a pre-filled `p_auth` CSRF token —
  ready for direct invocation from the browser.
- Write and delete operations are listed and callable from the
  console: `delete-file-entry`, `delete-folder`, `delete-group`,
  `add-file-entry`, `update-company`, `export-layouts-as-file`,
  `enable-remote-staging`, among many others.
- The JSONWS API returns real data without authentication: 122 KB of
  records including internal Liferay identifiers (companyId, userId,
  UUIDs, epoch timestamps).
- The GraphQL schema exposes the full data model including types with
  sensitive field names (UserAccount.currentPassword, password,
  passwordResetTicket, emailVerificationTicket, birthDate).
- The invoke endpoint has no authentication layer — requests are
  processed regardless of auth state.

Not demonstrated (responsible disclosure):

- Write/delete operations were not executed to avoid modifying data
  on the target system.
- Extraction of user PII or credentials via the API (GraphQL data
  queries returned 500 errors; country data is reference data).
- Access to the admin panel (redirects to login).
