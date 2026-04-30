# Content Security Policy (CSP)

## What

Browser security mechanism controlling what resources can be loaded/executed.
Model: **default deny -> explicit allow**. Missing directives fall back to `default-src`.

---

## Core Directives

| Directive | Controls | Secure Default |
|-----------|----------|----------------|
| `default-src` | Fallback for all unspecified directives | `'self'` |
| `script-src` | JS execution (fallback for elem/attr) | `'self' 'nonce-XYZ' 'strict-dynamic'` |
| `script-src-elem` | `<script>` tags and external scripts | `'self' 'nonce-XYZ'` |
| `script-src-attr` | Inline event handlers (`onclick`, `onerror`, etc.) | `'none'` |
| `style-src` | CSS | `'self'` |
| `img-src` | Images | `'self'` |
| `connect-src` | fetch / XHR / WebSocket | `'self'` |
| `font-src` | Fonts | `'self'` |
| `object-src` | `<object>`, `<embed>`, `<applet>` | `'none'` |
| `frame-src` | iframes | `'none'` or trusted origin |
| `frame-ancestors` | Who can embed this page | `'none'` (anti-clickjack) |
| `base-uri` | `<base>` tag injection | `'self'` |
| `form-action` | Form submission targets | `'self'` |

---

## script-src Values

| Value | Risk | Notes |
|-------|------|-------|
| `'unsafe-inline'` | **Critical** | Allows inline JS |
| `'unsafe-eval'` | High | Allows `eval()` |
| `'nonce-XYZ'` | Safe | Only scripts with matching nonce execute |
| `'sha256-...'` | Safe | Locks script to exact content hash |
| `'strict-dynamic'` | Safe (with nonce) | Trusts scripts loaded by trusted scripts |
| `*` | **Critical** | Wildcard - defeats CSP |
| `https:` | High | Any HTTPS origin - trivially bypassed |

---

## Nonce-Based CSP (Best Practice)

Header:
```http
Content-Security-Policy: script-src 'self' 'nonce-abc123' 'strict-dynamic';
```

HTML:
```html
<script nonce="abc123">/* allowed */</script>
```

Nonce must be: cryptographically random, unique per request, never reused.

---

## Hash-Based CSP

```http
Content-Security-Policy: script-src 'sha256-base64encodedHash=';
```

Use when script content is static (inline snippet that never changes).

---

## Weak / Broken Configs (Attacker View)

```http
script-src * 'unsafe-inline' 'unsafe-eval';   /* fully defeated */
default-src *;                                 /* fully defeated */
script-src https: 'unsafe-inline';             /* fully defeated */
script-src 'self' https://cdn.jsdelivr.net;    /* JSONP bypass possible */
```

---

## Bypass Vectors

| Vector | Condition |
|--------|-----------|
| Inline event handlers (`onerror`, `onclick`) | `'unsafe-inline'` present |
| JSONP endpoint on allowed origin | Origin in `script-src` allowlist |
| Trusted CDN compromise / open redirect | CDN in allowlist |
| DOM XSS sinks (`innerHTML`, `eval`) | CSP does not block DOM XSS |
| SVG + `data:` URI | `img-src data:` allowed |
| `strict-dynamic` misuse | Loads attacker script via trusted script |
| `<base>` tag injection | `base-uri` missing |
| Form exfil | `form-action` missing |

---

## Deployment Modes

```http
/* Enforce (blocks violations) */
Content-Security-Policy: default-src 'self';

/* Report-only - test before enforcing */
Content-Security-Policy-Report-Only: default-src 'self'; report-uri /csp-report;
```

---

## Secure CSP Template

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-RANDOM' 'strict-dynamic';
  style-src 'self' https://fonts.googleapis.com;
  img-src 'self' data:;
  font-src https://fonts.gstatic.com;
  connect-src 'self' https://api.example.com;
  object-src 'none';
  base-uri 'self';
  frame-ancestors 'none';
  form-action 'self';
```

---

## Testing Steps (AI Agent)

1. Extract `Content-Security-Policy` header from target response.
2. Check `script-src` for `'unsafe-inline'`, `'unsafe-eval'`, `*`, `https:`, or JSONP-capable CDNs.
3. Check `object-src` - if missing or not `'none'`, `<object>` can execute JS.
4. Check `base-uri` - if missing, inject `<base href=//attacker.com>` to rewrite relative URLs.
5. Check `form-action` - if missing, form exfil to attacker origin is possible.
6. Check `frame-ancestors` - if missing, clickjacking is possible.
7. If nonce used: verify it is not static or guessable, not leaked elsewhere in the page.
8. If `strict-dynamic` used: trace all trusted script loads for attacker-influenceable sources.

---

## Key Limitations

- CSP does **not** prevent DOM XSS (JS sinks like `innerHTML`, `document.write`)
- `strict-dynamic` ignores host allowlists at runtime - misconfiguration broadens trust scope
- Report-only mode never blocks - must switch to enforce before it has effect
