# FINDING: Visitor-Registration API Stack Trace Disclosure + Broken Security Filter

## Asset
`[TARGET]` (acceptance environment of a port authority's visitor-registration service; Tier 2, in scope)
Also reachable via redirect from `[TARGET-ALT-DOMAIN]`.

## Severity Estimate: Low-Medium
**VRT:** Information Disclosure > Stack Trace / Security Misconfiguration

---

## Summary

The visitor-registration API at `[TARGET]` exposes **full Java stack traces** on
every API endpoint except `/account/api/auth/none`. The error reveals a
misconfigured security filter (`NewSecurityFilter`) that is missing required
`@RestSecurityData` annotations on 5 distinct REST resource interfaces.

The **entire registration workflow** (create account, reset password, list countries, view
policies, check applications) returns HTTP 500 with the full stack trace.

---

## Evidence

### Temporal Context
- **Early in the session**: the browser received **200 OK** responses from
  these same endpoints during active registration
- **About 13 minutes later**: all endpoints began returning **500** with stack traces
- Same Kubernetes deployment hash (`[REPLICASET-HASH]`) on both pods observed
  (`[POD-SUFFIX-1]` and `[POD-SUFFIX-2]`)

### Affected Resource Interfaces (5 discovered)

| Resource Interface | Endpoint Pattern | Method |
|---|---|---|
| `IRegisterNewUserResource` | `/account/api/registernewusers/*` | GET/POST |
| `IParameterResource` | `/account/api/parameters/*` | GET |
| `ILostPasswordResource` | `/account/api/lostpassword` | POST |
| `IResetPasswordResource` | `/account/api/resetpassword/*` | GET |
| `ICountryResource` | `/account/api/countries` | GET |

### Working Endpoint (control)

| Resource | Endpoint | Status |
|---|---|---|
| (Unknown - likely IAuthResource) | `/account/api/auth/none` | **200 OK** (24KB) |

### PoC — Registration Policy Endpoint

```
GET https://[TARGET]/account/api/registernewusers/policy
Accept: application/json
```

Response (500):
```json
{
  "message": "expected @RestSecurityData or @LoggedInUserRestSecurityData for resource IRegisterNewUserResource, but was not found",
  "stacktrace": "java.lang.IllegalStateException: expected @RestSecurityData or @LoggedInUserRestSecurityData for resource IRegisterNewUserResource, but was not found\n\tat com.[VENDOR].[PLATFORM].managers.rest.impl.security.NewSecurityFilter.getSecurityDataFromResource(NewSecurityFilter.java:142)\n\tat com.[VENDOR].[PLATFORM].managers.rest.impl.security.NewSecurityFilter.getSecurityData(NewSecurityFilter.java:115)\n\tat com.[VENDOR].[PLATFORM].managers.rest.impl.security.NewSecurityFilter.filter(NewSecurityFilter.java:66)\n\tat org.jboss.resteasy.core.ResourceMethodInvoker.invokeOnTarget(ResourceMethodInvoker.java:263)\n\t[...33 frames total]"
}
```

### PoC — Country List Endpoint (different resource class)

```
GET https://[TARGET]/account/api/countries?language=EN
Accept: application/json
```

Response (500):
```json
{
  "message": "expected @RestSecurityData or @LoggedInUserRestSecurityData for resource ICountryResource, but was not found",
  "stacktrace": "java.lang.IllegalStateException: [same pattern, different resource name]"
}
```

### PoC — Lost Password (POST, different resource class)

```
POST https://[TARGET]/account/api/lostpassword
Content-Type: application/json

{"email":"test@example.com"}
```

Response (500): Same stack trace, `ILostPasswordResource`.

### PoC — Arbitrary Token (no token validation reached)

```
GET https://[TARGET]/account/api/registernewusers/AAAAAAAAAAAAAAAA/applications
```

Response (500): Same stack trace. The security filter crashes **before** token validation,
meaning ANY request to these endpoints triggers the disclosure regardless of input.

---

## Information Disclosed

### Security Architecture
- **Security filter class**: `com.[VENDOR].[PLATFORM].managers.rest.impl.security.NewSecurityFilter`
- **Security annotations**: `@RestSecurityData`, `@LoggedInUserRestSecurityData`
- **Filter entry points**: `getSecurityDataFromResource()` (line 142), `getSecurityData()` (line 115), `filter()` (line 66)
- **Vendor/framework**: third-party vendor platform (`com.[VENDOR].[PLATFORM]`)

### Application Stack
- **REST framework**: JBoss RESTEasy
- **Web server**: Eclipse Jetty
- **Servlet container**: `org.eclipse.jetty.servlet.ServletHandler`
- **Runtime**: Java (Thread.java:840)

### Internal Interface Names
1. `IRegisterNewUserResource`
2. `IParameterResource`
3. `ILostPasswordResource`
4. `IResetPasswordResource`
5. `ICountryResource`

### Infrastructure (from /auth/none — the one working endpoint)
- **Kubernetes pod**: `[APP]-api-[REPLICASET-HASH]-[POD-SUFFIX-1]` (and `[POD-SUFFIX-2]`)
- **Environment**: `[ENV]` (acceptance)
- **Module**: `[MODULE]`
- **Portal URL**: `https://[TARGET-PORTAL]/profile/`

---

## Impact

### Confirmed
1. **Stack trace disclosure**: Full 33-frame Java stack traces with internal class names,
   package structure, security filter implementation, and line numbers
2. **Security architecture exposure**: Attacker learns the exact security annotation
   mechanism and filter class, useful for crafting targeted attacks
3. **5 internal resource interfaces enumerated**: Complete REST API internal naming revealed
4. **Registration workflow broken**: All API calls return 500 — new registrations, password
   resets, and existing registration lookups are non-functional

### Reportability Assessment
- **NOT** "verbose messages without sensitive info" — reveals security filter internals
- **NOT** "banner grabbing" — goes far beyond version strings
- **IS** a security misconfiguration with information disclosure
- **Weakened** by being acceptance environment (not production)
- **Strengthened** by the scope of info leaked (security architecture, not just versions)

---

## Exclusion Risk

The program excludes verbose messages, files and directory listings without sensitive information.
The argument for reportability is that this DOES contain sensitive information:
the security filter's internal implementation, annotation names, and class structure.
A triager might disagree and classify this as verbose error messages.

**Honest assessment**: Borderline Low. Worth reporting if framed around the security
architecture exposure, not just "stack trace shown".
