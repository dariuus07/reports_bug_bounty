# Subdomain Takeover via Dangling CNAME to Expiring Third-Party Domain ([TARGET-DOMAIN])

## Title (for YesWeHack)
Subdomain Takeover Risk: 3 [TARGET-DOMAIN] subdomains point to unresolvable third-party domain expiring within about two months

## Severity
High

## Asset
[TARGET-DOMAIN] — a government agency (government-domain scope)

## Vulnerability Type
CWE-672 — Operation on a Resource after Expiration or Release
Subdomain Takeover

---

## Summary

Three subdomains of **[TARGET-DOMAIN]** have DNS CNAME records pointing to subdomains of the third-party domain **[THIRD-PARTY-DOMAIN]** that no longer resolve. The CNAME targets have no A records, making the chain broken.

**The critical factor:** `[THIRD-PARTY-DOMAIN]` is registered through a commercial registrar and **expires roughly two months after the discovery date**. If the current registrant does not renew it, anyone can register this domain for ~$12 and immediately serve arbitrary content on three government subdomains.

---

## Affected Assets

| Government Subdomain | CNAME Target | Resolves? |
|---|---|---|
| `[APP].[TARGET-DOMAIN]` | `[ORG]-frontend-live.[THIRD-PARTY-DOMAIN]` | No |
| `admin-[APP].[TARGET-DOMAIN]` | `[ORG]-backend-live.[THIRD-PARTY-DOMAIN]` | No |
| `api-[APP].[TARGET-DOMAIN]` | `[ORG]-api-live.[THIRD-PARTY-DOMAIN]` | No |

---

## Steps to Reproduce

**Step 1** — Verify the CNAME records exist:

```bash
dig CNAME [APP].[TARGET-DOMAIN] +short
# Output: [ORG]-frontend-live.[THIRD-PARTY-DOMAIN].

dig CNAME admin-[APP].[TARGET-DOMAIN] +short
# Output: [ORG]-backend-live.[THIRD-PARTY-DOMAIN].

dig CNAME api-[APP].[TARGET-DOMAIN] +short
# Output: [ORG]-api-live.[THIRD-PARTY-DOMAIN].
```

**Step 2** — Verify the CNAME targets do NOT resolve (no A record):

```bash
dig A [ORG]-frontend-live.[THIRD-PARTY-DOMAIN] +short
# Output: (empty — no A record)

dig A [ORG]-backend-live.[THIRD-PARTY-DOMAIN] +short
# Output: (empty — no A record)

dig A [ORG]-api-live.[THIRD-PARTY-DOMAIN] +short
# Output: (empty — no A record)
```

**Step 3** — Verify the parent domain resolves but the subdomains do not:

```bash
dig A [THIRD-PARTY-DOMAIN] +short
# Output: [IP] / [IP] / [IP]
# (cloud-hosted: the domain is alive, but the agency's subdomains on it are gone)
```

**Step 4** — Verify the domain expiration date:

```bash
whois [THIRD-PARTY-DOMAIN] | grep -i expir
# Output: Registry Expiry Date: [DATE, roughly two months after discovery]
```

The domain is registered through a commercial registrar and uses a cloud provider's DNS name servers.

---

## Attack Scenario

### Immediate path (if [THIRD-PARTY-DOMAIN] allows subdomain registration)
If `[THIRD-PARTY-DOMAIN]` is a hosting platform that allows customers to create subdomains, an attacker could sign up, create `[ORG]-frontend-live`, `[ORG]-backend-live`, and `[ORG]-api-live`, and the government CNAME records would immediately route traffic to the attacker's servers.

### Near-term path (domain expiration — about two months)
1. Wait for `[THIRD-PARTY-DOMAIN]` to expire
2. Register the domain (~$12 on any registrar)
3. Set up DNS with A records for the three target subdomains
4. Serve arbitrary content on `[APP].[TARGET-DOMAIN]`, `admin-[APP].[TARGET-DOMAIN]`, and `api-[APP].[TARGET-DOMAIN]`

No interaction with the agency's infrastructure is needed. The attacker controls the DNS resolution through the third-party domain.

---

## Impact

**Phishing under government-domain trust:**
An attacker could host a convincing government login page at `[APP].[TARGET-DOMAIN]`. The government TLD carries strong implicit trust — browsers show no warning, corporate firewalls whitelist it, and users have no reason to suspect the content is malicious. This could be used to harvest credentials for the national digital identity, the agency's services, or any government portal.

**Cookie exposure:**
If `[TARGET-DOMAIN]` sets cookies scoped to the parent domain (`Domain=.[TARGET-DOMAIN]`) or if any application shares cookies across subdomains, the attacker-controlled subdomain could read those cookies, potentially hijacking authenticated sessions.

**Malware distribution:**
Hosting malware downloads under a government domain bypasses most URL reputation filters, email security gateways, and corporate web proxies that whitelist government domains.

**Supply chain risk:**
The subdomain names (`admin-[APP]`, `api-[APP]`, `[APP]`) suggest these were application components (frontend, backend, API). If any internal system still references these hostnames (monitoring, CI/CD, scripts), the attacker could intercept that traffic.

---

## Evidence Summary

| Item | Value |
|------|-------|
| Affected subdomains | `[APP].[TARGET-DOMAIN]`, `admin-[APP].[TARGET-DOMAIN]`, `api-[APP].[TARGET-DOMAIN]` |
| CNAME targets | `[ORG]-{frontend,backend,api}-live.[THIRD-PARTY-DOMAIN]` |
| Target A records | **None** — broken CNAME chain |
| Third-party domain registrar | commercial registrar |
| Third-party domain expiry | roughly two months after discovery |
| Third-party DNS | cloud provider DNS |
| Third-party hosting | cloud provider |
| Discovery method | Passive DNS enumeration + CNAME resolution verification |
| Date discovered | September 2026 |

---

## Remediation

**Immediate (< 1 day):** Remove the three stale CNAME records from the `[TARGET-DOMAIN]` DNS zone. Since the target service is decommissioned, these records serve no purpose and only create risk.

**Short-term:** Audit all of the agency's government DNS zones for other CNAME records pointing to third-party domains, and verify that each target is still active and under government control.

**Long-term:** Implement a DNS hygiene process that includes removing CNAME records when decommissioning services hosted on third-party platforms. Consider monitoring CNAME targets for expiration dates.

---

## Notes

- I did **not** attempt to register the domain or any subdomains. This report is based purely on passive DNS analysis.
- All evidence is independently verifiable using standard DNS tools (`dig`, `whois`).
- The subdomain naming pattern (`frontend-live`, `backend-live`, `api-live`) suggests a decommissioned web application that was previously hosted on `[THIRD-PARTY-DOMAIN]`.
