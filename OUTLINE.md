# Why MCP and ChatGPT Apps Use Double Iframes (And What That Means for Your App)

> We'll explain why ChatGPT/MCP apps use double iframes, explore security policies, and reveal future "Policy allow" attributes for new features.

**Duration:** ~18 min + 2 min Q&A

**Key references:**
- [Reverse-engineering article by @infoxicator](https://www.dev.to/infoxicator/i-reverse-engineered-chatgpt-apps-iframe-sandbox-2ok3)
- [MCP Apps spec overview](https://apps.extensions.modelcontextprotocol.io/api/documents/Overview.html)
- [PR #158 — permissions negotiation (merged)](https://github.com/modelcontextprotocol/ext-apps/pull/158)
- [PR #355 — sandbox flags negotiation (open, v2)](https://github.com/modelcontextprotocol/ext-apps/pull/355)

---

## Part 1 — Context (3 slides, ~2 min)

### 1. Cover
- Title + speaker info

### 2. Quick recap — What are AI apps?
- Interactive UI rendered inside AI chats, built on MCP
- Available on ChatGPT, Claude, Goose, VSCode, Postman
- Your app's UI is a web page in an iframe — but not just any iframe

### 3. Zooming into the DOM
- Stylized DevTools view of the nested iframe structure on ChatGPT
- Host page (`chatgpt.com`) → outer iframe (`abc123.web-sandbox.oaiusercontent.com`, `sandbox="allow-scripts allow-same-origin"`) → inner iframe (`srcdoc`, `allow="camera; microphone"`) → Your App
- Hook: "Wait — why two iframes?"

---

## Part 2 — Why a Double Iframe? (3 slides, ~4 min)

### Single iFrame with srcdoc
- just the body and a single iframe with src to about:blank
- HTML content injected by host using `srcdoc` attribute
- Problem: If `srcdoc` is used without `sandbox` attribute, CSP will be inherited from the parent frame. ChatGPT's script-src is nonce-based, all scripts tags in your view will be blocked.

### Single iFrame with srcdoc and script-src relaxed
- just the body and a single iframe with src to about:blank and `srcdoc` attribute
- chatgpt.com CSP header on scripts is relaxed
- Problem: your iFramed view inherits chatgpt.com origin. Your scripts can access ChatGPT's cookies, local storage, DOM...

### Single iFrame with srcdoc and sandbox allow-scripts
- just the body and a single iframe with src to about:blank and `srcdoc` attribute
- new `sandbox` attribute is used to enforce a new CSP for the inner iframe with `allow-scripts`
- Problem: Browser assigns an opaque origin to the iframe. Can't use at all local storage, cookies, indexedDB, etc.

### Single iFrame with srcdoc and sandbox allow-scripts and allow-same-origin
- just the body and a single iframe with src to about:blank and `srcdoc` attribute
- updated `sandbox` attribute with added `allow-same-origin`
- Problem: your iFramed view inherits chatgpt.com origin. Your scripts can access ChatGPT's cookies, local storage, DOM... We need to use a different origin for the iframe

### Single iFrame with src 
- just the body and a single iframe with src to MCP server domain
- switching from srcdoc to src enables origin isolation
- using views exposed as SPA from MCP servers domains
- Problem: chatgpt.com CSP header on frame-src is restrictive. Opening up to * is too wide.

### Single iFrame with src on controlled domain
- just the body and a single iframe with src to a different origin
- switching from MCP server domain to a different origin controled by chatgpt.com. Adding *.oaiusercontent.com to frame-src CSP directive on chatgpt.com. Content is downloaded with resources/read request, cached, and served from an app-specific subdomain of oaiusercontent.com
- Problem: xxxx

### Double iFrame
- just the body and a single iframe with src to a different origin
- the same sandbox is served on all subdomains of *.web-sandbox.oaiusercontent.com
- the sandbox starts the view using iframe with srcdoc and sandbox allow-scripts allow-same-origin on the app-specific subdomain of web-sandbox.oaiusercontent.com

### 4. The problem: hosts have their own CSP
- CSP = HTTP header whitelisting which resources a page can load
- `frame-src` controls which domains can appear in iframes
- ChatGPT's actual `frame-src` shown (Stripe, Google, YouTube… and `*.web-sandbox.oaiusercontent.com`)
- MCP is an open ecosystem — can't whitelist every app domain
- **This is the root cause of the double iframe**
- Joke: "CSP is the new CORS — the config nightmare you'll hit on day one of building an AI app"

### 5. The solution: one trusted proxy domain
- Whitelist a single wildcard domain in `frame-src`
- ChatGPT: `*.web-sandbox.oaiusercontent.com` — Claude: `*.claudemcpcontent.com`
- Mermaid diagram: Host (strict CSP) → Outer iframe (sandbox proxy) → Inner iframe (per-app CSP) → Your App
- Proxy receives HTML via `postMessage`, creates inner iframe, applies per-app CSP
- **Not new — 15+ year old pattern.** Any marketplace rendering third-party UI on its own domain faces this exact problem:
  - Facebook Apps (~2008): `page_proxy.php` on FB's domain, nested iframes for cross-frame communication
  - OpenSocial / Yahoo / iGoogle (~2007): Apache Shindig proxy rendering gadgets in iframes
  - Shopify embedded apps (~2020): dynamic `frame-ancestors` CSP per-shop

### 6. Three security layers
- **Host CSP**: protects the host page. Nonce-based scripts, specific allowlists, Datadog violation reporting.
- **Sandbox proxy (outer)**: `sandbox="allow-scripts allow-same-origin"`. Different origin from host = no escape. Restricts capabilities.
- **Per-app CSP (inner)**: set by proxy from `_meta.ui.csp`. `connectDomains`, `resourceDomains`, `frameDomains`. Default = fully restrictive.

---

## Part 3 — What You Can Configure Today (2 slides, ~3 min)

### 7. CSP negotiation: unlocking resources
- Table of configurable inner CSP directives:
  - `connectDomains` → `connect-src` → fetch, XHR, WebSocket
  - `resourceDomains` → `script-src`, `style-src`, `font-src`, `img-src`, `media-src` → external scripts, styles, fonts, images
  - `frameDomains` → `frame-src` → nested iframes (maps, video)
  - `baseUriDomains` → `base-uri` → custom base URIs
- JSON manifest example
- Key point: **declaration is not authorization** — the app requests domains, the host decides what to grant. The host can review, restrict, or deny. Each app gets only its own CSP. That's why this isn't the same as `frame-src *`.

### 8. Permissions negotiation (PR #158, merged Jan 2026)
- Apps declare: `_meta.ui.permissions: { camera: {}, microphone: {}, geolocation: {}, clipboardWrite: {} }`
- Maps to `allow` attribute on inner iframe
- Hosts report granted permissions via `hostCapabilities.sandbox.permissions`
- VSCode and Goose adopted it the same day
- Already a live speech transcription example in the repo

---

## Part 4 — Host Readiness: Who Allows What? (2 slides, ~3 min)

### 9. Permissions Policy comparison across hosts
- **Claude**: camera, mic, geo, clipboard-write all granted to `*.claudemcpcontent.com`. **Ready.**
- **ChatGPT**: every permission `()` blocked. Sandbox proxy reportedly allows `microphone *; midi *`. **Not ready at host level.**
- **Postman, Goose, VSCode**: no Permissions-Policy header → permissive by default
- Two gates: host PP must allow + iframe `allow` must grant. Most restrictive wins.

### 10. Use cases & what's next
- Camera/mic → voice apps, barcode scanning, speech transcription
- Geolocation → store locators, delivery tracking
- Clipboard → copy generated code
- Payment → not yet in spec, no host grants it
- **Sandbox flags** coming in v2 (PR #355): `allow-forms`, `allow-popups`, `allow-modals`, `allow-downloads` become negotiable

---

## Part 5 — The Dangers (2 slides, ~3 min)

### 11. The classic sandbox escape
- `allow-scripts + allow-same-origin` = iframe can remove its own sandbox and reload unrestricted
- MCP spec uses both — safe because proxy is a **different origin** from host
- Origin isolation is the real security boundary, not sandbox alone

### 12. What goes wrong when hosts open too wide?
- `allow-top-navigation` → phishing redirects
- `connect-src *` → data exfiltration
- Broad `frame-src` → clickjacking
- Loose Permissions Policy → rogue camera/mic activation

---

## Part 6 — Wrap-up (2 slides, ~3 min)

### 13. Takeaways
1. **Double iframe solves a CSP whitelist problem** — one proxy domain (`*.web-sandbox.oaiusercontent.com` / `*.claudemcpcontent.com`) is the gateway for all apps
2. **Permissions negotiation is already in the spec** — Claude grants camera/mic/geo/clipboard at host level. ChatGPT doesn't yet.
3. **Sandbox flags negotiation is coming** (v2) — forms, popups, modals, downloads
4. **Respect the sandbox** — origin isolation is the real boundary. Request only what you need.

### 14. Thank you
- Questions?
- Links: skybridge.tech, alpic.ai
