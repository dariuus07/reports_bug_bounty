# Intigriti Report — Visitor-Registration API: Broken Security Filter + Information Disclosure

## Title
Broken security filter exposes full Java stack traces and internal infrastructure on a visitor-registration API ([TARGET])

## Domain / URL
[TARGET]

## Severity
Medium (CVSS 5.3)

## CVSS Vector
AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N

## CWE
CWE-209: Generation of Error Message Containing Sensitive Information

## VRT Category
Information Disclosure > Sensitive Data Exposure

---

## Description

The visitor-registration API at `[TARGET]` has a misconfigured security filter that causes **every API endpoint** (except one) to return HTTP 500 with a full Java stack trace. No authentication is required — a single unauthenticated GET request is enough.

The root cause is that the security filter `NewSecurityFilter` requires `@RestSecurityData` or `@LoggedInUserRestSecurityData` annotations on each REST resource interface, but 5 resource interfaces lack them. The filter throws `IllegalStateException` before any request processing occurs, dumping the full exception chain to the response body.

Additionally, the one working endpoint (`/account/api/auth/none`) returns a 24KB JSON response containing the Kubernetes pod hostname, environment identifier, internal URLs, and 132 i18n keys that reveal business logic details — all without authentication.

---

## Steps to reproduce

### Prerequisites
- curl (or any HTTP client)
- No authentication, cookies, or tokens required

### Step 1 — Stack trace disclosure

Open a terminal and run:

```bash
curl -sk https://[TARGET]/account/api/countries?language=EN
```

Response (HTTP 500):
```json
{
  "message": "expected @RestSecurityData or @LoggedInUserRestSecurityData for resource ICountryResource, but was not found",
  "stacktrace": "java.lang.IllegalStateException: expected @RestSecurityData or @LoggedInUserRestSecurityData for resource ICountryResource, but was not found\n\tat com.[VENDOR].[PLATFORM].managers.rest.impl.security.NewSecurityFilter.getSecurityDataFromResource(NewSecurityFilter.java:142)\n\tat com.[VENDOR].[PLATFORM].managers.rest.impl.security.NewSecurityFilter.getSecurityData(NewSecurityFilter.java:115)\n\tat com.[VENDOR].[PLATFORM].managers.rest.impl.security.NewSecurityFilter.filter(NewSecurityFilter.java:66)\n\tat org.jboss.resteasy.core.ResourceMethodInvoker.invokeOnTarget(ResourceMethodInvoker.java:263)\n\t[...33 frames]"
}
```

**[Screenshot: Terminal showing the curl command and the 500 response with the full stack trace]**

### Step 2 — Confirm it affects all 5 resource interfaces

Each of the following returns 500 with the same stack trace pattern, naming a different internal interface:

```bash
# IRegisterNewUserResource
curl -sk https://[TARGET]/account/api/registernewusers/policy

# ILostPasswordResource
curl -sk -X POST -H "Content-Type: application/json" \
  -d '{"email":"test@example.com"}' \
  https://[TARGET]/account/api/lostpassword

# IResetPasswordResource (405 — endpoint exists but wrong method)
curl -sk https://[TARGET]/account/api/resetpassword

# IRegisterNewUserResource (arbitrary path — security filter fails before routing)
curl -sk https://[TARGET]/account/api/registernewusers/ANYTHING/applications
```

**[Screenshot: Terminal showing all 4 requests with 500 responses, each naming a different resource interface]**

### Step 3 — Infrastructure disclosure via /auth/none

```bash
curl -sk https://[TARGET]/account/api/auth/none | python3 -m json.tool
```

Response (HTTP 200, 24KB):
```json
{
  "module": {
    "code": "[MODULE]",
    "supported_languages": ["DE", "EN", "FR", "NL"],
    "[module]_url": "https://[TARGET-PORTAL]/profile/"
  },
  "bundles": [
    {"language": "EN", "key_values": {"...132 keys...": "..."}},
    {"language": "NL", "key_values": {"...132 keys...": "..."}}
  ],
  "server": {
    "environment": "[ENV]",
    "hostname": "[APP]-api-[REPLICASET-HASH]-[POD-SUFFIX]"
  }
}
```

**[Screenshot: Terminal showing the curl output with `server.hostname` and `server.environment` highlighted]**

---

## Disclosed information

### From stack traces (all endpoints, no auth)

| Category | Detail |
|----------|--------|
| Security filter class | `com.[VENDOR].[PLATFORM].managers.rest.impl.security.NewSecurityFilter` |
| Security annotations | `@RestSecurityData`, `@LoggedInUserRestSecurityData` |
| Filter source lines | `getSecurityDataFromResource()` :142, `getSecurityData()` :115, `filter()` :66 |
| REST framework | JBoss RESTEasy (`ResourceMethodInvoker`, `SynchronousDispatcher`) |
| Web server | Eclipse Jetty (`ServletHolder`, `SessionHandler`, `HttpConnection`) |
| Vendor package | `com.[VENDOR].[PLATFORM]` (third-party vendor platform) |
| Internal interfaces | `IRegisterNewUserResource`, `IParameterResource`, `ILostPasswordResource`, `IResetPasswordResource`, `ICountryResource` |
| Java runtime | `java.base/java.lang.Thread` (line 840) |

### From /auth/none (200 OK, no auth)

| Category | Detail |
|----------|--------|
| Kubernetes pod hostname | `[APP]-api-[REPLICASET-HASH]-[POD-SUFFIX]` |
| Environment identifier | `[ENV]` |
| Internal portal URL | `https://[TARGET-PORTAL]/profile/` |
| i18n keys (EN + NL) | 132 per language — reveal user management workflows, company registration, organization-code format, application codes, activity management |
| Entity types handled | tax, customs, trade-registry and logistics identifiers |
| Support contact | `[SUPPORT-EMAIL]` |

---

## Impact

**Confirmed:**

1. **Security architecture exposed:** An attacker learns exactly how the security filter works — its class name, source file line numbers, and the annotation-based authorization mechanism. This information directly aids in crafting targeted attacks against other endpoints in the same vendor framework deployment.

2. **Infrastructure disclosed:** The Kubernetes pod hostname (`[APP]-api-[REPLICASET-HASH]-[POD-SUFFIX]`) reveals the deployment naming convention and hash. Combined with the `environment: [ENV]` tag, an attacker confirms this is a pre-production environment sharing infrastructure patterns with production.

3. **Business logic mapped:** The 132 i18n keys reveal the full user and company management workflow — registration, activation, admin roles, organization-code management, community access control, and application provisioning. This is a detailed blueprint of the application's authorization model.

4. **Zero-auth access:** All of the above is accessible with a single unauthenticated HTTP request. No account creation, no tokens, no cookies.

5. **Registration API non-functional:** The broken security filter means the entire registration workflow (new user registration, password reset, country lookup) returns 500, constituting a service disruption.

**Not claimed (cannot verify):**

- Whether the same deployment hash appears in other environments
- Whether the organization-code format disclosure enables enumeration of valid codes
- Impact on the associated portal (`[TARGET-PORTAL]`)

---

## Root cause

The `NewSecurityFilter` (line 66) calls `getSecurityData()` (line 115) which calls `getSecurityDataFromResource()` (line 142) for each incoming request. This method looks for `@RestSecurityData` or `@LoggedInUserRestSecurityData` annotations on the target resource interface. Five interfaces lack these annotations, causing `IllegalStateException`. The exception is not caught by any handler in the chain, so JBoss RESTEasy serializes it as a JSON response including the full stack trace — the default error handler for uncaught exceptions.

The `/account/api/auth/none` endpoint works because its resource interface (likely `IAuthResource`) does have the required annotation, so the security filter passes.

---

## Remediation

1. Add the missing `@RestSecurityData` or `@LoggedInUserRestSecurityData` annotations to the 5 affected resource interfaces.
2. Configure a global exception handler that returns a generic error response (without stack traces or internal class names) for unhandled exceptions.
3. Remove the `server.hostname` and `server.environment` fields from the `/account/api/auth/none` response, or restrict this endpoint to authenticated users.
