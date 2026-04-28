# XSS Automation Agent — Protocol

## Files

| File | Purpose |
|------|---------|
| `cheatsheet/tags.txt` | One HTML tag per line — use to generate payloads |
| `cheatsheet/events.txt` | One event handler per line — combine with tags |
| `common-xss_cases.md` | Common injection contexts: HTML body, attribute, script, SVG, etc. |
| `dom-xss.md` | DOM-specific sources, sinks, and payload patterns |

---

## Agent Workflow

### Step 1 — Identify the injection context

Probe the target with a canary string (e.g. `xsscanary1234`) and observe:
- Where does it appear in the response / DOM?
- Is it inside HTML, inside an attribute, inside a `<script>` block, or in a URL?

### Step 2 — Try common cases first

Read `common-xss_cases.md` → **Quick Reference** section at the top.
Run payloads in order. Stop and report as soon as one triggers.

### Step 3 — Build tag × event payloads from cheatsheet

If Step 2 finds no hit, generate payloads by combining `tags.txt` × `events.txt`:

```
<{tag} {event}=alert(1)>
```

Example loop (bash):
```bash
while IFS= read -r tag; do
  while IFS= read -r evt; do
    echo "<${tag} ${evt}=alert(1)>"
  done < cheatsheet/events.txt
done < cheatsheet/tags.txt
```

Feed each generated payload to the source parameter/field.

### Step 4 — Escalate to DOM XSS

If no hit after Steps 2–3, read `dom-xss.md` → **Quick Reference** section at the top.

Check:
1. Are there client-side sources? (`location.hash`, `location.search`, `postMessage`, etc.)
2. Do those sources reach a dangerous sink? (`innerHTML`, `eval`, `document.write`, etc.)
3. Inject canary via source → confirm it lands in a sink → send XSS payload.

### Step 5 — Report or Escalate

- **Hit found** → record: context, payload, parameter, evidence.
- **No hit** → mark as not vulnerable via these vectors; escalate to stored/blind XSS checks.

---

## Decision Tree

```
Inject canary
  └─ appears in HTML body?        → try common HTML injection payloads
  └─ appears inside attribute?    → try attribute break + event payloads
  └─ appears inside <script>?     → try JS string/statement break payloads
  └─ appears in URL/href?         → try javascript: protocol payloads
  └─ not reflected in response?
        └─ check DOM sources      → run DOM XSS checks (dom-xss.md)
        └─ still nothing          → mark clean for reflected XSS
```

---

## Notes

- Always start from `common-xss_cases.md`; only escalate if you exhaust it.
- `textContent` is safe; `innerHTML` / `eval` / `document.write` are sinks.
- Validate `e.origin` on every `postMessage` listener before treating it as a source.
