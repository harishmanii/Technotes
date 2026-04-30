# Same-Origin Policy (SOP) — Agent Knowledge Module

## DEFINITION

SOP is a browser security mechanism that restricts how documents or scripts from one origin interact with resources from another origin.

**Origin** = `scheme + host + port`

| URL | Same-origin as `https://example.com`? |
|-----|--------------------------------------|
| `https://example.com/page` | YES |
| `https://api.example.com` | NO (different host) |
| `http://example.com` | NO (different scheme) |
| `https://example.com:8080` | NO (different port) |

---

## CORE RULE

> SOP allows sending cross-origin requests but **blocks reading responses**.

---

## ALLOWED vs BLOCKED

### ALWAYS ALLOWED
- Sending HTTP requests (`fetch`, `XHR`, `form`, `img`, `script`)
- Navigation (`window.location`)
- Embedding (`<iframe>`, `<img>`, `<script>`)
- Executing cross-origin scripts (runs in current origin's context)

### BLOCKED
- Reading response body from cross-origin fetch/XHR
- Accessing cross-origin DOM (`iframe.contentDocument`)
- Accessing cross-origin storage (`localStorage`, `sessionStorage`, `IndexedDB`)
- Reading cookies of another origin

---

## SPECIAL CASE: `<script>` Tag

```html
<script src="https://cdn.example.com/lib.js"></script>
```

- Script **loads** cross-origin
- Script **executes** in the **current page's origin**
- Has **full access** to current page DOM, cookies, localStorage

> Treat any cross-origin script inclusion as **full trust execution**.

---

## PARTIAL ACCESS (Cross-Origin Window)

| Property / Method | Read | Write |
|---|---|---|
| `window.location` | No | Yes |
| `window.closed` | Yes | No |
| `window.length` | Yes | No |
| `window.close()` | — | Yes (call) |
| `window.focus()` | — | Yes (call) |
| `window.postMessage()` | — | Yes (call) |

---

## COOKIE SCOPE

- Cookies are **domain-scoped**, not strict-origin scoped.
- `Domain=example.com` is accessible from `app.example.com` and `admin.example.com`.
- Subdomain compromise leads to potential session/cookie compromise across subdomains.

---

## RELAXATION MECHANISMS

### 1. CORS
- Server sets `Access-Control-Allow-Origin` to explicitly permit cross-origin reads.
- Misconfiguration bypasses the SOP read restriction.

### 2. document.domain
```javascript
document.domain = "example.com"; // both pages must set this
```
- Allows same parent-domain subdomains to become same-origin.
- Cannot be set to an unrelated domain or TLD.
- Security risk: a compromised subdomain gains access to all others that set it.

### 3. postMessage
```javascript
window.postMessage(data, targetOrigin);
```
- Controlled cross-origin communication channel.
- Abuse vector if origin validation is missing or uses `"*"`.

---

## SOP DOES NOT PREVENT

| Threat | Why SOP does not help |
|---|---|
| CSRF | Requests are sent with cookies; the response is irrelevant |
| Script inclusion attacks | Execution is allowed by design |
| Supply chain attacks | External script tags are fully trusted |

---

## ATTACK MAPPING

| Attack | SOP behavior exploited |
|---|---|
| CSRF | Allowed: request sending with cookies |
| CORS exploitation | Bypasses: response read restriction via misconfigured headers |
| JSONP / script injection | Allowed: cross-origin script execution |
| postMessage abuse | Allowed: cross-origin communication with weak origin check |
| Subdomain takeover + document.domain | Exploits: SOP relaxation |
| Supply chain (CDN compromise) | Exploits: script execution in current origin |

---

## AGENT DECISION RULES

**Rule 1 — Verify SOP is working**

IF cross-origin request succeeds BUT response is unreadable → SOP intact

**Rule 2 — Detect SOP bypass**

IF cross-origin response is readable → CHECK: CORS headers, document.domain, postMessage

**Rule 3 — Script inclusion**

IF external script is loaded → treat as FULL TRUST, check CDN integrity (SRI)

**Rule 4 — Subdomain interaction**

IF subdomains interact → check document.domain usage and cookie Domain scope

**Rule 5 — Form-based action**

IF attack uses form submission → SOP irrelevant → test CSRF

---

## AGENT TEST WORKFLOW

```
1. Is the request cross-origin?
   YES: continue
   NO:  SOP does not apply

2. Can the response be read by the attacker?
   YES: SOP bypassed → investigate CORS misconfiguration
   NO:  SOP working → pivot to CSRF / indirect abuse

3. Is a script loaded from an external origin?
   YES: verify SRI integrity hash; if missing → supply chain risk

4. Are subdomains involved?
   YES: test document.domain; check cookie Domain attribute

5. Is the action achievable via a form?
   YES: test CSRF; SOP reading restriction is irrelevant
```

---

## RED FLAGS

- `Access-Control-Allow-Origin: *` combined with `Access-Control-Allow-Credentials: true`
- `document.domain` set in any page
- `postMessage` handlers with no `event.origin` check
- `<script>` tags loading from third-party domains without `integrity` attribute
- Cookies without `SameSite` or `Secure`, or scoped to a parent domain

---

## COMMON MISCONCEPTIONS

| Wrong | Correct |
|---|---|
| "SOP blocks cross-origin requests" | SOP blocks **reading** cross-origin responses |
| "CORS disables SOP" | CORS **selectively relaxes** the read restriction |
| "fetch is blocked cross-origin" | fetch sends fine; only the **read** is blocked |

---

## MENTAL MODEL

SOP = "You can interact, but you cannot see"

Attacks exploit what is ALLOWED, not what is blocked.
