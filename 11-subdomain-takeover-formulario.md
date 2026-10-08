# Formulario de un VDP gubernamental — Contenido para cada campo

Copia y pega cada sección en el campo correspondiente del formulario.

---

## 1. Report title (250 char max)

```
Dangling CNAME on [APP].[TARGET-DOMAIN], admin-[APP].[TARGET-DOMAIN], api-[APP].[TARGET-DOMAIN] to expiring third-party domain enables subdomain takeover
```

## 2. Product/website concerned (250 char max)

```
[TARGET-DOMAIN]
```

## 3. Bug type

Selecciona: **Misconfiguration** (o "Subdomain Takeover" si aparece en el dropdown)

## 4. Endpoint

```
[APP].[TARGET-DOMAIN], admin-[APP].[TARGET-DOMAIN], api-[APP].[TARGET-DOMAIN]
```

## 5. Vulnerable part

Selecciona: **Infrastructure** o **DNS** (lo que esté disponible)

## 6. Parameter Name

```
CNAME DNS record
```

## 7. Payload/POC

```
dig CNAME [APP].[TARGET-DOMAIN] +short
dig CNAME admin-[APP].[TARGET-DOMAIN] +short
dig CNAME api-[APP].[TARGET-DOMAIN] +short
dig A [ORG]-frontend-live.[THIRD-PARTY-DOMAIN] +short
whois [THIRD-PARTY-DOMAIN] | grep -i expir
```

## 8. Technical environment (250 char max)

```
Linux, dig 9.18, whois CLI, passive DNS enumeration (subfinder, assetfinder)
```

## 9. CVE

Dejar vacío (no aplica).

## 10. Impact

Selecciona: **Medium** o **High** (lo que esté disponible; recomiendo Medium)

---

## CVSS 3.1 — Selecciona estos valores en cada dropdown:

| Campo | Valor | Justificación |
|---|---|---|
| 11. Attack Vector (AV) | **Network (N)** | DNS es accesible por red |
| 12. Attack Complexity (AC) | **High (H)** | Requiere que el dominio expire y el atacante lo registre |
| 13. Privileges Required (PR) | **None (N)** | Cualquiera puede registrar un dominio expirado |
| 14. User Interaction (UI) | **None (N)** | El takeover no requiere interacción de usuario |
| 15. Scope (S) | **Changed (C)** | El componente vulnerable es el DNS del organismo; el impacto es en los usuarios que visitan los subdominios |
| 16. Confidentiality (C) | **Low (L)** | Posible exposición de cookies si están scoped al dominio padre |
| 17. Integrity (I) | **Low (L)** | El atacante puede servir contenido arbitrario en 3 subdominios |
| 18. Availability (A) | **None (N)** | Sin impacto en disponibilidad demostrado |

## 19. CVSS Score

```
6.1
```

(Vector: CVSS:3.1/AV:N/AC:H/PR:N/UI:N/S:C/C:L/I:L/A:N)

## 20. Vulnerability Severity

Selecciona: **Medium**

---

## 21. Technical description (Part 1) — máx 10000 chars

```
VULNERABILITY DISCOVERY

During passive reconnaissance of [TARGET-DOMAIN] subdomains using subfinder and assetfinder, I identified 175 subdomains. While verifying DNS resolution with dig, I noticed three subdomains had CNAME records pointing to subdomains of "[THIRD-PARTY-DOMAIN]" that returned no A records — a broken CNAME chain. I checked the domain's WHOIS data and found it expires roughly two months from now.

AFFECTED SUBDOMAINS

- [APP].[TARGET-DOMAIN] → CNAME → [ORG]-frontend-live.[THIRD-PARTY-DOMAIN] (no A record)
- admin-[APP].[TARGET-DOMAIN] → CNAME → [ORG]-backend-live.[THIRD-PARTY-DOMAIN] (no A record)
- api-[APP].[TARGET-DOMAIN] → CNAME → [ORG]-api-live.[THIRD-PARTY-DOMAIN] (no A record)

STEPS TO REPRODUCE

Step 1 — Verify the CNAME records exist:

  dig CNAME [APP].[TARGET-DOMAIN] +short
  Output: [ORG]-frontend-live.[THIRD-PARTY-DOMAIN].

  dig CNAME admin-[APP].[TARGET-DOMAIN] +short
  Output: [ORG]-backend-live.[THIRD-PARTY-DOMAIN].

  dig CNAME api-[APP].[TARGET-DOMAIN] +short
  Output: [ORG]-api-live.[THIRD-PARTY-DOMAIN].

Step 2 — Verify the CNAME targets do NOT resolve:

  dig A [ORG]-frontend-live.[THIRD-PARTY-DOMAIN] +short
  Output: (empty — no A record)

  dig A [ORG]-backend-live.[THIRD-PARTY-DOMAIN] +short
  Output: (empty — no A record)

  dig A [ORG]-api-live.[THIRD-PARTY-DOMAIN] +short
  Output: (empty — no A record)

Step 3 — Verify the parent domain is still registered but the agency subdomains are gone:

  dig A [THIRD-PARTY-DOMAIN] +short
  Output: [IP] / [IP] / [IP]

Step 4 — Verify the domain expiration date:

  whois [THIRD-PARTY-DOMAIN] | grep -i expir
  Output: Registry Expiry Date: [DATE, roughly two months after discovery]

  Registrar: [REGISTRAR]. Name servers: cloud provider DNS.

IMPACT

Three [TARGET-DOMAIN] subdomains have CNAME records pointing to a third-party domain ([THIRD-PARTY-DOMAIN]) where the target subdomains no longer exist. The third-party domain expires in roughly two months. After expiry, anyone can register [THIRD-PARTY-DOMAIN] for approximately $12, configure DNS for the three target subdomains, and serve arbitrary content under the government domain's trust.

This is a subdomain takeover condition. The naming pattern (frontend-live, backend-live, api-live) indicates a decommissioned web application whose DNS records were not cleaned up.

REMEDIATION

Remove the three stale CNAME records from the [TARGET-DOMAIN] DNS zone. The target service is decommissioned and these records serve no purpose.

NOTE

I did not attempt to register the domain or any subdomains. This finding is based on passive DNS analysis only. All evidence is independently verifiable using dig and whois.
```

## 22-23. Technical description Part 2 y Part 3

Dejar vacío — todo cabe en Part 1.

## 24. Attachments

Opcional: puedes adjuntar capturas de pantalla de los outputs de `dig` y `whois` para dar más peso visual.

---

## 25-27. Reporter Information

- 25. Tu nombre
- 26. Tu nacionalidad  
- 27. Tu email (recibirás un OTP para verificar)

## 28. Vulnerability Disclosure Policy

Marca la casilla de aceptación.
