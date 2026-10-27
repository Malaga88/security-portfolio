# E-Commerce Platform Security Assessment
### Black-Box Penetration Testing Report (Redacted for Portfolio)

> **Note:** This is a sanitized version of a confidential assessment conducted under written authorization for a live e-commerce platform. Company name, domain, and identifying details have been removed or replaced. Findings, methodology, and technical content are unaltered.

**Tested by:** Hosea Habila Yakubu ("Darkshade")
**Testing period:** November 2025
**Scope:** Public-facing e-commerce API endpoints
**Methodology:** Black-box penetration testing with limited gray-box analysis
**Authorization:** Conducted under explicit written authorization from the platform owner

---

## Executive Summary

A black-box penetration test of a production e-commerce platform identified **four vulnerabilities**, including one **critical** NoSQL injection flaw, exposing customer data, internal business data, and user identifiers to unauthorized access. All findings were reported to the platform owner with a 90-day responsible disclosure window.

| Vulnerability | Severity | CVSS | Priority |
|---|---|---|---|
| NoSQL Injection | Critical | 9.1 | P0 – Immediate |
| Mass Information Disclosure | High | 7.5 | P1 – Urgent |
| User ID Exposure (IDOR risk) | Medium | 6.5 | P2 – Important |
| Insufficient Input Validation | Medium | 5.5 | P2 – Important |

---

## Finding 1: NoSQL Injection (Critical, CVSS 9.1)

**Endpoint:** `/api/store/cart`

The cart API accepted raw MongoDB query operators in request bodies without sanitization, allowing injection of operators directly into database queries.

**Operators confirmed to be processed by the backend:**
```
{"$gt": 0}          — greater-than
{"$gte": 0}         — greater-than-or-equal
{"$ne": null}       — not-equal
{"$regex": ".*"}    — regex matching
{"$exists": true}   — existence check
{"$where": "this.quantity > 0"} — arbitrary JS evaluation
```

**Proof of concept:**
```bash
curl 'https://[REDACTED]/api/store/cart' \
  -X PUT \
  -H 'Content-Type: application/json' \
  -H 'Cookie: [SESSION_COOKIE]' \
  --data-raw '{"productId":{"$ne":null},"quantity":{"$gte":0}}'
```

The application returned a response consistent with query execution against the database, confirming the injected operators were processed rather than rejected.

**Impact:** unauthorized access to other users' cart data, potential authentication bypass, database structure enumeration, and a path toward privilege escalation.

---

## Finding 2: Mass Information Disclosure (High, CVSS 7.5)

**Endpoint:** `/api/store/products`

The endpoint returned the entire product database with no pagination, authentication, or field filtering — exposing internal MongoDB IDs, exact stock levels, internal organizational IDs, approval workflow status, and full pricing data.

**Example response:**
```json
{
  "_id": "68cce2185106f1395fb54d07",
  "name": "[Product Name]",
  "stock": 8,
  "chapter": "68cce2185106f1395fb54cef",
  "approved": true
}
```

**Impact:** competitive intelligence leakage, automated stock monitoring/scalping, exposure of internal organizational structure.

---

## Finding 3: User ID Exposure (Medium, CVSS 6.5)

**Endpoint:** `/api/store/cart`

Cart responses exposed internal MongoDB user `_id` values, usable for IDOR attacks, user enumeration, and session correlation.

```json
{
  "user": "690f37218ab8fb0d8c649162",
  "_id": "690f37f3397e4cb7f6d2c346"
}
```

---

## Finding 4: Insufficient Input Validation (Medium, CVSS 5.5)

**Endpoint:** `/api/store/cart`

The endpoint accepted arrays, booleans, string-typed numbers, scientific notation, decimals, and complex objects for fields expecting integers — validating only against explicit negative numbers. This opens the door to type-confusion attacks and business-logic bypass.

---

## Testing Methodology

- **Tools:** Nmap 7.95, curl, custom bash fuzzing scripts, manual logic testing
- **Test cases:** 16 NoSQL injection variants, authentication bypass attempts, IDOR testing, information disclosure checks, input validation fuzzing, business logic testing
- **Approach:** Black-box, with limited gray-box analysis once initial access was established

---

## Remediation Highlights

Full remediation guidance (including Joi schema validation, response sanitization, rate limiting, security headers, and logging/monitoring recommendations) was delivered to the platform owner with working code examples for each fix, split into:

- **Immediate actions (24–48 hrs):** strict input validation, MongoDB query sanitization, access controls + pagination on the products endpoint, response field stripping
- **Short-term (1 week):** schema validation middleware, security logging/alerting on injection attempts, security headers, rate limiting
- **Long-term (1 month):** full third-party audit, ODM migration (Mongoose), developer security training, incident response planning

---

## Responsible Disclosure

This test was conducted under written authorization, with findings reported privately to the platform owner and a 90-day disclosure window before any public discussion of the engagement (in this redacted, non-identifying form only).

---

*This report is a redacted portfolio sample. Original findings, endpoints, and company identity remain confidential per the terms of the engagement.*
