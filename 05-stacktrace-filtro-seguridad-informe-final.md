# Intigriti Report — Visitor-Registration API Information Disclosure

## Title
Broken security filter leaks security internals and K8s infrastructure on a visitor-registration API (acceptance environment)

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

The visitor-registration API at `[TARGET]` has a misconfigured security filter that causes every API endpoint (except one) to return HTTP 500 with full Java stack traces. No authentication is required.

The stack traces expose the security filter's class name, source file line numbers, annotation-based authorization mechanism, and the internal package structure of the vendor framework. The one working endpoint (`/account/api/auth/none`) returns a 24KB JSON response containing the Kubernetes pod hostname, environment identifier, and an internal URL — without authentication.

Additionally, the related ForgeRock AM at `[TARGET-AUTH]` sets session cookies for `[INTERNAL-DOMAIN]` — an internal network domain — in every authentication response, visible to any external client.

This is not verbose error output or banner grabbing: the stack traces reveal the **security authorization mechanism itself** — the filter class, the annotations it checks, and the exact lines where it fails. Combined with the K8s pod hostname and internal domain name, this provides concrete infrastructure targeting information.

---

## Steps to reproduce

### Prerequisites
- curl
- No authentication required

### PoC 1 — Stack trace with security filter internals

```bash
curl -s https://[TARGET]/account/api/countries?language=EN
```

Response (HTTP 500):
```json
{
  "message": "expected @RestSecurityData or @LoggedInUserRestSecurityData for resource ICountryResource, but was not found",
  "stacktrace": "java.lang.IllegalStateException: expected @RestSecurityData or @LoggedInUserRestSecurityData for resource ICountryResource, but was not found\n\tat com.[VENDOR].[PLATFORM].managers.rest.impl.security.NewSecurityFilter.getSecurityDataFromResource(NewSecurityFilter.java:142)\n\tat com.[VENDOR].[PLATFORM].managers.rest.impl.security.NewSecurityFilter.getSecurityData(NewSecurityFilter.java:115)\n\tat com.[VENDOR].[PLATFORM].managers.rest.impl.security.NewSecurityFilter.filter(NewSecurityFilter.java:66)\n\tat org.jboss.resteasy.core.ResourceMethodInvoker.invokeOnTarget(ResourceMethodInvoker.java:263)\n\t[...33 frames]"
}
```

The same pattern reproduces on every API endpoint, each naming a different internal interface:

| Command | Interface leaked |
|---|---|
| `curl -s https://[TARGET]/account/api/registernewusers/policy` | `IRegisterNewUserResource` |
| `curl -s -X POST -H 'Content-Type: application/json' -d '{"email":"test@example.com"}' https://[TARGET]/account/api/lostpassword` | `ILostPasswordResource` |
| `curl -s -X POST https://[TARGET]/account/api/resetpassword` | `IResetPasswordResource` |

**[Screenshot 1: Terminal showing the 4 curl commands with 500 responses, each naming a different resource interface]**

### PoC 2 — K8s hostname and environment via /auth/none

```bash
curl -s https://[TARGET]/account/api/auth/none | python3 -m json.tool
```

Response (HTTP 200, 24KB). Relevant extract:
```json
{
  "server": {
    "environment": "[ENV]",
    "hostname": "[APP]-api-[REPLICASET-HASH]-[POD-SUFFIX]"
  },
  "module": {
    "code": "[MODULE]",
    "[module]_url": "https://[TARGET-PORTAL]/profile/"
  }
}
```

**[Screenshot 2: Terminal output with server.hostname highlighted]**

### Supplementary — Internal domain leak on the authentication host

The ForgeRock AM on the same registration infrastructure leaks an internal network domain via Set-Cookie headers:

```bash
curl -sD- -o /dev/null -X POST \
  -H "Content-Type: application/json" \
  -H "X-OpenAM-Username: test" \
  -H "X-OpenAM-Password: test" \
  https://[TARGET-AUTH]/[AM-CONTEXT]/json/authenticate
```

Response headers:
```
Set-Cookie: amlbcookie=06; Path=/; Domain=[TARGET-AUTH]; Secure; HttpOnly
Set-Cookie: amlbcookie=06; Path=/; Domain=[INTERNAL-DOMAIN]; Secure; HttpOnly
```

Confirmed on a second authentication host (`[TARGET-AUTH-2]`) as well (same `[INTERNAL-DOMAIN]` domain).

**[Screenshot 3: Terminal showing both Set-Cookie lines with the internal domain highlighted]**

---

## Impact

- The stack traces reveal the security filter class (`NewSecurityFilter`), its source file line numbers (66, 115, 142), and the annotation-based authorization mechanism (`@RestSecurityData` / `@LoggedInUserRestSecurityData`). This is the internal security model, not a version string.

- The `/auth/none` endpoint exposes a Kubernetes pod hostname (`[APP]-api-[REPLICASET-HASH]-[POD-SUFFIX]`), the environment tag (`[ENV]`), and an internal portal URL. This is concrete infrastructure information.

- The ForgeRock AM leaks the internal AD domain `[INTERNAL-DOMAIN]` via Set-Cookie headers on the authentication endpoint, revealing the internal network naming convention.

- All of the above is accessible with zero authentication from the public internet.

**Not claimed:**
- Whether the internal domain resolves from outside the network
- Impact on production (out of scope for testing)
