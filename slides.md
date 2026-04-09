---
theme: ./theme
title: "Why MCP and ChatGPT Apps Use Double Iframes"
info: |
  By Frederic Barthelet, co-founder at Alpic
fonts:
  sans: Fira Sans
  mono: Roboto Mono
  local: Mozilla Text Variable
transition: slide-left
mdc: true
duration: 18min
layout: cover
---

# Why MCP & ChatGPT Apps Use [Double Iframes]{.accent}

### And What That Means for Your App

<div class="speaker-info mt-8">
  <img src="/fred.jpg" alt="Frederic" />
  <div>
    <p class="font-semibold text-base">Frédéric Barthelet</p>
    <p class="text-sm opacity-50">Co-founder @ Alpic</p>
  </div>
</div>

<!--
Hi everyone! I'm Frederic, co-founder at Alpic — we build infrastructure for AI apps. Today we're going deep on something every AI app developer bumps into: the double iframe architecture. Why it exists, what it protects, and what new features it's about to unlock.
-->

---

# What are [AI apps]{.accent}?

<div class="mt-6 flex items-center gap-12">
  <div>
    <ul class="space-y-3 text-lg">
      <li><span class="accent font-semibold">Discoverability</span> in-chat and in stores on ChatGPT and Claude</li>
      <li><span class="accent font-semibold">Interactive UI</span> inside AI chats</li>
      <li>
        <ul class="ml-6 space-y-2">
          <li>Liad and Ido experimenting with <span class="highlight font-semibold">MCP-UI</span></li>
          <li>Initially released on ChatGPT as <span class="highlight font-semibold">OpenAI's Apps SDK</span></li>
          <li>Standardized as <span class="highlight font-semibold">MCP</span> first official extension</li>
        </ul>
      </li>
    </ul>
  </div>
  <div>
    <!-- TODO: placeholder — screenshot of a ChatGPT app rendering inside the chat (e.g. a weather widget or map) -->
    <div class="card flex items-center justify-center" style="width: 320px; height: 200px;">
      <p class="text-xs opacity-30 text-center">📷 Screenshot: AI app rendering inside ChatGPT chat</p>
    </div>
  </div>
</div>

<!--
Quick recap. AI apps are interactive UIs rendered directly inside AI chats. Built on MCP, available on ChatGPT, Claude, Goose, VSCode, Postman, and more. Your app's UI is a web page inside an iframe — but a very specific kind of iframe.
-->

---

# How it [works]{.accent}

<div class="grid grid-cols-[1fr_1.6fr] gap-8 mt-4">
<div class="flex flex-col justify-center space-y-6">
  <div>
    <p class="text-base font-semibold highlight">Views render in response to tool calls</p>
  </div>
  <div>
    <p class="text-base font-semibold highlight">HTML documents live in resources</p>
  </div>
  <div>
    <p class="text-base font-semibold highlight">Views can be cached ahead of time</p>
  </div>
</div>
<div>

```mermaid {scale: 0.72, theme: 'base', themeVariables: {actorTextColor: '#ffffff', actorLineColor: '#ffffff', signalColor: '#ffffff', signalTextColor: '#ffffff', labelTextColor: '#ffffff', noteBkgColor: '#333333', noteTextColor: '#ffffff', sequenceNumberColor: '#ffffff', actorBorder: '#ffffff', actorBkg: '#1a1a2e'}}
sequenceDiagram
  participant H as 🌐 Host
  participant S as ⚙️ MCP Server

  H->>S: tools/list
  S-->>H: tools with _meta.ui.resourceUri
  loop
    H->>+S: tools/call
    H->>S: resources/read
    S-->>H: View HTML with _meta.ui { csp permissions }
    Note over H: Create view in the chat<br/> with iframe
    S-->>-H: result { content + _meta.ui.url }
    Note over H: Hydrate view with tool response
  end
```

</div>
</div>

<!--
Here's how the UI flow works. First, the host discovers your tools via tools/list. The response already includes _meta.ui metadata — CSP directives, permissions — so the host knows before it even invokes the tool that it will render a view. It can prepare the security policies upfront. Then when the host calls the tool, the server returns the result with a _meta.ui.url pointing to the app's HTML. The host creates a sandboxed iframe with the right policies already in place and renders the UI directly in the chat. That iframe setup is where all the security architecture we're about to explore comes into play.

If ui.resourceUri is present and host supports MCP Apps, host renders tool results using the specified UI resource. If host does not support MCP Apps, tool behaves as standard tool (text-only fallback)


-->

---

# Zooming into the [View]{.accent}

<div class="card mt-6 font-['Roboto_Mono'] text-[0.7em] leading-[1.8] w-max mx-auto">
  <p><span class="opacity-40">&lt;body&gt;</span> <span class="opacity-50 highlight">chatgpt.com</span></p>
  <p class="ml-4"><span class="accent">&lt;iframe</span> <span class="opacity-50">src="<span class="highlight">abc123.web-sandbox.oaiusercontent.com</span>"&gt;</span></p>
  <p class="ml-8"><span class="accent">&lt;iframe</span> <span class="opacity-50">src="about:blank"&gt;</span></p>
  <p class="ml-12"><span class="opacity-60">&lt;div&gt; Your App &lt;/div&gt;</span></p>
  <p class="ml-8"><span class="accent">&lt;/iframe&gt;</span></p>
  <p class="ml-4"><span class="accent">&lt;/iframe&gt;</span></p>
  <p><span class="opacity-40">&lt;/body&gt;</span></p>
</div>

<p class="text-2xl mt-8 text-center font-semibold">Why <span class="accent">two</span> iframes?</p>

<!--
Open DevTools on any ChatGPT app and here's what you'll see. The host page at chatgpt.com. Inside, an outer iframe pointing to web-sandbox.oaiusercontent.com — with sandbox attributes. Inside that, an inner iframe with your actual app HTML and an allow attribute for permissions. Two nested iframes. Why? Let's find out.
-->

---

# Before AI Apps — [the host page]{.accent}

<div class="grid grid-cols-[1fr_1.4fr] gap-6 mt-2">
<div>

<div class="card py-2 px-3 font-['Roboto_Mono'] text-[0.7em] w-max leading-[1.5]">
  <p><span class="text-[1.1em] accent font-semibold mb-1">chatgpt.com</span> Content-Security-Policy</p>
  <p>default-src <span class="accent">'self'</span></p>
  <p>script-src <span class="accent">'nonce-…' https://*.chatgpt.com</span></p>
  <p>style-src <span class="accent">'self' 'unsafe-inline'</span></p>
  <p>connect-src <span class="accent">'self' *.openai.com wss://*.chatgpt.com</span></p>
  <p>img-src <span class="accent">'self' * blob: data: https:</span></p>
  <p>frame-src <span class="accent">'self' *.stripe.com *.youtube.com …</span></p>
</div>

</div>
<div>

| Directive   | Allowed values                                                            |
| ----------- | ------------------------------------------------------------------------- |
| default-src | `'none'` `'self'` `https:`                                                |
| script-src  | `'none'` `'nonce-…'` `'strict-dynamic'` `'unsafe-inline'` `'unsafe-eval'` |
| style-src   | `'self'` `'unsafe-inline'` `'nonce-…'`                                    |
| connect-src | `'self'` `https:` `wss:`                                                  |
| img-src     | `'self'` `data:` `blob:`                                                  |
| frame-src   | `'self'` `'none'` `blob:`                                                 |

</div>
</div>

<!--
Before we look at iframes, let's look at what ChatGPT sends with every HTTP response. A Content-Security-Policy header — a set of directives that tell the browser exactly what this page is allowed to load. default-src is the fallback: if a specific directive isn't set, default-src applies. script-src controls which scripts can run — ChatGPT uses a nonce, a per-request token, so only scripts with the matching token execute. style-src for stylesheets. connect-src controls where fetch, XHR, and WebSocket calls can go. img-src for images. And frame-src — which URLs are allowed inside iframes. This last one is about to become very important.
-->

---

# What if — [srcdoc]{.accent}?

<div class="card mt-2 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p class="opacity-40">chatgpt.com Content-Security-Policy</p>
  <p class="opacity-40">script-src 'nonce-…' https://*.chatgpt.com</p>
  <p class="opacity-40">frame-src 'self' *.stripe.com *.youtube.com …</p>
</div>
<br/>
<div class="card mt-3 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p><span class="opacity-40">&lt;body&gt;</span></p>
  <p class="ml-4"><span class="accent">&lt;iframe</span> srcdoc="…"&gt;</p>
  <p class="ml-8"><span class="opacity-40">&lt;div&gt; Your App &lt;/div&gt;</span></p>
  <p class="ml-4"><span class="accent">&lt;/iframe&gt;</span></p>
  <p><span class="opacity-40">&lt;/body&gt;</span></p>
</div>

<p class="text-lg mt-6 text-center font-semibold"><span class="accent">✗</span> inherits parent CSP without script nonce, <span class="accent">all view scripts blocked</span></p>

<!--
-->

---

# What if — [relax the CSP]{.accent}?

<div class="card mt-2 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p class="opacity-40">chatgpt.com Content-Security-Policy</p>
  <p>script-src <span class="accent">'unsafe-inline' 'unsafe-eval'</span> </p>
  <p class="opacity-40">frame-src 'self' *.stripe.com *.youtube.com …</p>
</div>
<br/>
<div class="card mt-2 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p><span class="opacity-40">&lt;body&gt;</span></p>
  <p class="ml-4"><span class="accent opacity-40">&lt;iframe</span> <span class="opacity-40">srcdoc="…"&gt;</span></p>
  <p class="ml-8"><span class="opacity-40">&lt;div&gt; Your App &lt;/div&gt;</span></p>
  <p class="ml-4"><span class="accent opacity-40">&lt;/iframe&gt;</span></p>
  <p><span class="opacity-40">&lt;/body&gt;</span></p>
</div>

<p class="text-lg mt-6 text-center font-semibold"><span class="accent">✗</span> iframe inherits parent's origin — view scripts can access ChatGPT's <span class="accent">cookies, localStorage, DOM</span></p>

<!--
-->

---

# What if — [sandbox]{.accent}?

<div class="card mt-2 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p class="opacity-40">chatgpt.com Content-Security-Policy</p>
  <p>script-src <span class="accent">'nonce-…' https://*.chatgpt.com</span></p>
  <p class="opacity-40">frame-src 'self' *.stripe.com *.youtube.com …</p>
</div>
<br/>
<div class="card mt-2 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p><span class="opacity-40">&lt;body&gt;</span></p>
  <p class="ml-4"><span class="accent opacity-40">&lt;iframe</span> <span class="opacity-40">srcdoc="…"</span> sandbox="allow-scripts"<span class="opacity-40">&gt;</span></p>
  <p class="ml-8"><span class="opacity-40">&lt;div&gt; Your App &lt;/div&gt;</span></p>
  <p class="ml-4"><span class="accent opacity-40">&lt;/iframe&gt;</span></p>
  <p><span class="opacity-40">&lt;/body&gt;</span></p>
</div>

<p class="text-lg mt-6 text-center font-semibold"><span class="accent">✗</span> Browser assigns <span class="accent">opaque origin</span> — no localStorage, no cookies, no IndexedDB</p>

<!--
Sandboxing generates an opaque origin
srcdoc takes precedence over src, no point trying to set origin on the iframe
-->

---

# What if — [allow‑same‑origin]{.accent}?

<div class="card mt-2 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p class="opacity-40">chatgpt.com Content-Security-Policy</p>
  <p class="opacity-40">script-src 'nonce-…' https://*.chatgpt.com</p>
  <p class="opacity-40">frame-src 'self' *.stripe.com *.youtube.com …</p>
</div>
<br/>
<div class="card mt-2 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p><span class="opacity-40">&lt;body&gt;</span></p>
  <p class="ml-4"><span class="accent opacity-40">&lt;iframe</span> <span class="opacity-40">srcdoc="…" sandbox="allow-scripts</span> allow-same-origin<span class="opacity-40">"&gt;</span></p>
  <p class="ml-8"><span class="opacity-40">&lt;div&gt; Your App &lt;/div&gt;</span></p>
  <p class="ml-4"><span class="accent opacity-40">&lt;/iframe&gt;</span></p>
  <p><span class="opacity-40">&lt;/body&gt;</span></p>
</div>

<p class="text-lg mt-6 text-center font-semibold"><span class="accent">✗</span> <code>srcdoc</code> inherits <code class="highlight">chatgpt.com</code> origin — <span class="accent">sandbox escape</span> possible</p>

<!--
can't set an origin when iframe has srcdoc
only way to remove the opaque origin is to use allow-same-origin
falling back into the same original problem
-->

---

# What if — [src]{.accent}?

<div class="card mt-2 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p class="opacity-40">chatgpt.com Content-Security-Policy</p>
  <p class="opacity-40">script-src 'nonce-…' https://*.chatgpt.com</p>
  <p class="opacity-40">frame-src 'self' *.stripe.com *.youtube.com …</p>
</div>
<br/>
<div class="card mt-2 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p><span class="opacity-40">&lt;body&gt;</span></p>
  <p class="ml-4"><span class="accent opacity-40">&lt;iframe</span> src="https://my-app.com/view.html"<span class="opacity-40">&gt;</span></p>
  <p class="ml-8"><span class="opacity-40">&lt;div&gt; Your App &lt;/div&gt;</span></p>
  <p class="ml-4"><span class="accent opacity-40">&lt;/iframe&gt;</span></p>
  <p><span class="opacity-40">&lt;/body&gt;</span></p>
</div>

<p class="text-lg mt-4 text-center font-semibold"><span class="accent">✗</span> <code>frame-src</code> is a <span class="accent">finite allowlist</span> — can't whitelist every MCP App domain</p>

<!--
Relaxing frame-src to * is not an option
requires serving view from MCP server domain instead of resources
-->

---

# What if — [controlled domain]{.accent}?

<div class="card mt-2 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p class="opacity-40">chatgpt.com Content-Security-Policy</p>
  <p class="opacity-40">script-src 'nonce-…' https://*.chatgpt.com</p>
  <p>frame-src <span class="accent">'self' *.stripe.com *.youtube.com … *.oaiusercontent.com</span></p>
</div>
<br/>
<div class="card mt-2 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p><span class="opacity-40">&lt;body&gt;</span></p>
  <p class="ml-4"><span class="accent opacity-40">&lt;iframe</span> src="https://abc123.oaiusercontent.com"<span class="opacity-40">&gt;</span></p>
  <p class="ml-8"><span class="opacity-40">&lt;div&gt; Your App &lt;/div&gt;</span></p>
  <p class="ml-4"><span class="accent opacity-40">&lt;/iframe&gt;</span></p>
  <p><span class="opacity-40">&lt;/body&gt;</span></p>
</div>

<p class="text-lg mt-6 text-center font-semibold"><span class="accent">✗</span> Per-app hosting — <span class="accent">needs heavy dynamic server infrastructure</span></p>

<!--
Switch back to view as resources
requires the infrastructure to host all MCP servers views with a dynamic server router that uses subdomain to resources/read request for content and CSP. Content served as payload and CSP as header (can't change CSP after iframe is created)
-->

---

# What if - the [double iframe]{.accent}

<div class="card mt-2 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p class="opacity-40">chatgpt.com Content-Security-Policy</p>
  <p class="opacity-40">script-src 'nonce-…' https://*.chatgpt.com</p>
  <p>frame-src <span class="accent">'self' *.stripe.com *.youtube.com … web-sandbox.oaiusercontent.com</span></p>
</div>
<br/>
<div class="card mt-2 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p><span class="opacity-40">&lt;body&gt;</span></p>
  <p class="ml-4"><span class="accent">&lt;iframe</span> src="https://web-sandbox.oaiusercontent.com"&gt;</p>
  <p class="ml-8">&lt;script&gt;initializeApp()&lt;/script&gt;</p>
    <p class="ml-8"><span class="accent">&lt;iframe</span> srcdoc="…" sandbox="allow-scripts allow-same-origin"&gt;</p>
  <p class="ml-12"><span class="opacity-40">&lt;div&gt; Your App &lt;/div&gt;</span></p>
  <p class="ml-8"><span class="accent">&lt;/iframe&gt;</span></p>
  <p class="ml-4"><span class="accent">&lt;/iframe&gt;</span></p>
  <p><span class="opacity-40">&lt;/body&gt;</span></p>
</div>

<!--
one sandbox page on web-sandbox.oaiusercontent.com used to load resource and create inner iframe with srcdoc and sandbox allow-scripts allow-same-origin
-->

---

# What if - double iframe on [app-specific subdomain]{.accent}

<div class="card mt-2 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p class="opacity-40">chatgpt.com Content-Security-Policy</p>
  <p class="opacity-40">script-src 'nonce-…' https://*.chatgpt.com</p>
  <p>frame-src <span class="accent">'self' *.stripe.com *.youtube.com … *.web-sandbox.oaiusercontent.com</span></p>
</div>
<br/>
<div class="card mt-2 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p><span class="opacity-40">&lt;body&gt;</span></p>
  <p class="ml-4"><span class="accent">&lt;iframe</span> src="https://abc123.web-sandbox.oaiusercontent.com"&gt;</p>
  <p class="ml-8 opacity-40">&lt;script&gt;initializeApp()&lt;/script&gt;</p>
    <p class="ml-8 opacity-40"><span class="accent">&lt;iframe</span> srcdoc="…" sandbox="allow-scripts allow-same-origin"&gt;</p>
  <p class="ml-12"><span class="opacity-40">&lt;div&gt; Your App &lt;/div&gt;</span></p>
  <p class="ml-8"><span class="accent opacity-40">&lt;/iframe&gt;</span></p>
  <p class="ml-4"><span class="accent opacity-40">&lt;/iframe&gt;</span></p>
  <p><span class="opacity-40">&lt;/body&gt;</span></p>
</div>

<!--
same sandbox loaded on all subdomains of web-sandbox.oaiusercontent.com
still use subdomain even if same content for app localStorage isolation
-->

---

# The solution - double iframe on app-specific subdomain [with CSP]{.accent}

<div class="card mt-2 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p class="opacity-40">chatgpt.com Content-Security-Policy</p>
  <p class="opacity-40">script-src 'nonce-…' https://*.chatgpt.com</p>
  <p class="opacity-40">frame-src 'self' *.stripe.com *.youtube.com … *.web-sandbox.oaiusercontent.com</p>
</div>
<br/>
<div class="card mt-2 font-['Roboto_Mono'] text-[0.65em] leading-[1.8] w-max mx-auto">
  <p><span class="opacity-40">&lt;body&gt;</span></p>
  <p class="ml-4 opacity-40"><span class="accent">&lt;iframe</span> src="https://abc123.web-sandbox.oaiusercontent.com"&gt;</p>
  <p class="ml-8"><span class="highlight">&lt;meta</span> http-equiv="Content-Security-Policy" content="connect-src 'my-service.com'"&gt;</p>
  <p class="ml-8 opacity-40">&lt;script&gt;initializeApp()&lt;/script&gt;</p>
    <p class="ml-8 opacity-40"><span class="accent">&lt;iframe</span> srcdoc="…" sandbox="allow-scripts allow-same-origin"&gt;</p>
  <p class="ml-12"><span class="opacity-40">&lt;div&gt; Your App &lt;/div&gt;</span></p>
  <p class="ml-8"><span class="accent opacity-40">&lt;/iframe&gt;</span></p>
  <p class="ml-4"><span class="accent opacity-40">&lt;/iframe&gt;</span></p>
  <p><span class="opacity-40">&lt;/body&gt;</span></p>
</div>

<!--
enforcing strict CSP by default
allow configuration via _meta.ui.csp
inspectable via resources/read request
-->

---

# You must [declare all domains]{.accent} your app [depends on]{.accent}

| _meta.ui.csp field    | CSP directive(s)                                          | Unlocks                                 |
| ----------------- | --------------------------------------------------------- | --------------------------------------- |
| `connectDomains`  | `connect-src`                                             | fetch, XHR, WebSocket                   |
| `resourceDomains` | `script-src` `style-src` `font-src` `img-src` `media-src` | External scripts, styles, fonts, images |
| `frameDomains`    | `frame-src`                                               | Nested iframes (maps, video)            |
| `baseUriDomains`  | `base-uri`                                                | Custom base URIs                        |

<!--
declaration used in app validation
-->

---

# [CSP]{.accent} is the new [CORS]{.accent}

<img src="/Gemini_Generated_Image_cz7ll3cz7ll3cz7l.png" class="rounded-lg mx-auto mt-4" style="max-height: 380px;" />

---

# Improving builder [DevX]{.accent}

<div class="flex items-center gap-8 mt-4">
  <img src="/openai-csp-in-dev.png" class="rounded-lg" style="max-height: 340px;" />
  <div class="space-y-4">
    <p class="text-base">ChatGPT now <span class="highlight">relaxes CSP in developer mode</span> by default</p>
    <p class="text-sm opacity-50">Dev apps get unrestricted network access — no need to declare domains while iterating</p>
    <p class="text-sm opacity-50">Toggle on "Enforce CSP" to test production behavior before shipping</p>
  </div>
</div>

---

# Improving builder DevX with [Skybridge]{.accent}

<div class="flex items-center gap-6 mt-6">
  <img src="/logos/skybridge/icon-only-white.svg" class="h-16" />
  <p class="text-lg font-mono opacity-60">skybridge.tech</p>
</div>

<div class="mt-6 grid grid-cols-3 gap-6">
  <div class="card text-center">
    <p class="text-sm font-semibold accent mb-2">CSP Inspector</p>
    <p class="text-xs opacity-50">See the exact policy applied to your app per host — at runtime</p>
  </div>
  <div class="card text-center">
    <p class="text-sm font-semibold accent mb-2">CSP Validation</p>
    <p class="text-xs opacity-50">Catch missing domains at build time before your fetch silently fails</p>
  </div>
  <div class="card text-center">
    <p class="text-sm font-semibold accent mb-2">Cross-host testing</p>
    <p class="text-xs opacity-50">Verify CSP behavior across ChatGPT, Claude & Cursor from one codebase</p>
  </div>
</div>

<!--
Skybridge ships with built-in CSP devtools. The inspector shows the exact policy each host applies to your app at runtime. The validator catches missing domains at build time — so you don't discover a blocked fetch in production. And you can test across all hosts from a single codebase.
-->

---

## layout: section

# [Demo]{.accent} — CSP Inspector

<!--
Now let me show you this in action. Skybridge ships with devtools that let you inspect the actual CSP policies applied to your app at runtime — across every host. Let me switch to the browser and walk you through the CSP inspector.
-->

---

# Build MCP Apps with [Skybridge]{.accent}

<div class="flex items-center gap-16 mt-4">
  <div class="flex flex-col items-center gap-6">
    <img src="/qr-skybridge.svg" class="w-56 h-56" alt="QR code to Skybridge GitHub" />
    <p class="text-xs font-mono opacity-70">github.com/alpic-ai/skybridge</p>
  </div>
  <div class="space-y-5">
    <div class="flex items-center gap-4 mb-6">
      <img src="/logos/skybridge/logo-white.svg" class="h-10" alt="Skybridge" />
      <span class="text-lg opacity-40">—</span>
      <p class="text-lg font-mono opacity-60">skybridge.tech</p>
    </div>
    <p class="text-xl font-semibold"><span class="highlight">Open-source framework</span> for building MCP Apps</p>
    <ul class="text-base opacity-60 space-y-1 my-1">
      <li>End-to-end typesafety</li>
      <li>Modern development environment</li>
      <li>Unified APIs for host cross-compatibility</li>
    </ul>
    <div class="card mt-4 flex items-center gap-4">
      <p class="text-lg font-semibold"><span class="accent">⭐</span> Star the repo</p>
      <span class="opacity-30">→</span>
      <p class="text-base"><span class="highlight font-semibold">enter the lottery</span> to win a mask <span class="opacity-60">🎭</span></p>
    </div>
  </div>
</div>

<!--
Everything I just demoed — the CSP inspector, permissions devtools, cross-host testing — it all ships with Skybridge. It's our open-source framework for building MCP Apps. Works on ChatGPT, Claude, VSCode, Goose, Postman — anywhere MCP runs. Scan the QR code, check out the repo, try it on your next project. And if you star it, you're automatically entered in our lottery to win a mask. Takes 5 seconds — go ahead, I'll wait.
-->

---

## layout: end

# [Thank you!]{.accent}

Questions?

<div class="grid grid-cols-2 gap-12 mt-10">
  <div class="flex flex-col items-center gap-2">
    <img src="/logos/skybridge/logo-white.svg" class="h-8" alt="Skybridge" />
    <p class="text-sm font-mono opacity-50">skybridge.tech</p>
  </div>
  <div class="flex flex-col items-center gap-2">
    <img src="/logos/alpic/logo-white.svg" class="h-8" alt="Alpic" />
    <p class="text-sm font-mono opacity-50">alpic.ai</p>
  </div>
</div>

<div class="speaker-info mt-10 justify-center">
  <img src="/fred.jpg" alt="Frederic" />
  <div class="text-left">
    <p class="font-semibold">Frederic</p>
    <p class="text-sm opacity-60">Co-founder @ Alpic</p>
  </div>
</div>

<!--
Thank you! Happy to answer questions about double iframes, security policies, or anything else about building AI apps.
-->
