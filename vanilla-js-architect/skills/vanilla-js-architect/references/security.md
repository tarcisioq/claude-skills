# Security

SDKs run in the consumer's environment with the consumer's privileges. A vulnerability in your SDK is a vulnerability in *every app that depends on it*. This file is the security baseline — non-negotiable.

## Threat model — what we defend against

1. **CSP-hostile code** — your SDK breaks consumers running with strict Content-Security-Policy
2. **Prototype pollution** — malicious payloads mutate `Object.prototype` via your SDK's option-merging
3. **Cross-origin message confusion** — your SDK exposes `postMessage` and trusts unverified senders
4. **Supply chain compromise** — your published package gets replaced by a malicious version
5. **Secret exposure** — credentials end up in logs, errors, or `localStorage` accidentally
6. **DOM-based injection** — your SDK writes consumer-supplied data into `innerHTML`
7. **Dependency rot** — transitive deps with CVEs

## CSP-safe code

A growing number of consumers run with strict CSP. Your SDK must work without `'unsafe-eval'` and without `'unsafe-inline'`.

### Banned constructs

```javascript
// ❌ All of these violate `script-src 'self'` strict CSP
eval("...");                                    // hard ban
new Function("...");                            // hard ban
setTimeout("doStuff()", 100);                   // hard ban (string form)
setInterval("doStuff()", 100);                  // hard ban
document.write("...");                          // banned by `script-src` if creates scripts
element.innerHTML = userControlledString;       // banned by Trusted Types
element.outerHTML = "...";                      // same
element.insertAdjacentHTML("beforeend", "...");// same
element.setAttribute("onclick", "...");         // banned by inline script CSP
```

### Safe replacements

```javascript
// ✅ Replace `eval` of JSON
const obj = JSON.parse(jsonString); // never eval JSON

// ✅ Replace `new Function` with closures or static dispatch
const handlers = { add, sub, mul };
const result = handlers[opName]?.(a, b);

// ✅ DOM construction without HTML strings
const div = document.createElement("div");
div.textContent = userText;        // text — safe
div.classList.add("foo");
parent.append(div);

// ✅ Trusted Types support (when CSP requires it)
let policy = null;
if (window.trustedTypes?.createPolicy) {
  policy = window.trustedTypes.createPolicy("my-sdk", {
    createHTML: (s) => sanitize(s),
  });
}
element.innerHTML = policy ? policy.createHTML(html) : sanitize(html);
```

### Lint config that catches violations

```json
// .eslintrc.json
{
  "rules": {
    "no-eval": "error",
    "no-implied-eval": "error",
    "no-new-func": "error",
    "no-script-url": "error"
  }
}
```

Run as part of CI on every PR. Build pipelines should also fail on `eval`/`Function` in built output.

## Prototype pollution — the option-merge trap

Naïve recursive merge of consumer-supplied options is the most common SDK vulnerability.

```javascript
// ❌ — `merge({}, { __proto__: { isAdmin: true } })` pollutes Object.prototype
function merge(target, source) {
  for (const key in source) {
    if (typeof source[key] === "object") {
      target[key] = merge(target[key] ?? {}, source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}
```

**Fixes:**

```javascript
// ✅ Filter dangerous keys
const FORBIDDEN_KEYS = new Set(["__proto__", "constructor", "prototype"]);

function safeMerge(target, source) {
  for (const key of Object.keys(source)) { // Object.keys ignores prototype keys
    if (FORBIDDEN_KEYS.has(key)) continue;
    const v = source[key];
    if (v && typeof v === "object" && !Array.isArray(v)) {
      target[key] = safeMerge(target[key] ?? Object.create(null), v);
    } else {
      target[key] = v;
    }
  }
  return target;
}

// ✅ Even better — use null-prototype objects internally
const config = Object.assign(Object.create(null), defaults, userOptions);

// ✅ Or use structuredClone for deep copy that ignores `__proto__`
const safe = structuredClone(userOptions); // ignores __proto__ as own property
```

### Test for prototype pollution

```javascript
import { describe, it, expect } from "vitest";

describe("prototype pollution defense", () => {
  it("does not pollute Object.prototype via __proto__", () => {
    new Client({ "__proto__": { polluted: true } });
    expect({}.polluted).toBeUndefined();
  });

  it("does not pollute via constructor.prototype", () => {
    new Client({ constructor: { prototype: { polluted: true } } });
    expect({}.polluted).toBeUndefined();
  });
});
```

## DOM injection (if your SDK touches the DOM)

```javascript
// ❌ — XSS vector
element.innerHTML = `<div>${userText}</div>`;

// ✅ — text-only insertion
const div = document.createElement("div");
div.textContent = userText;
element.append(div);

// ✅ — when you genuinely need HTML, sanitize with DOMPurify
import DOMPurify from "dompurify";
element.innerHTML = DOMPurify.sanitize(htmlFromUntrustedSource);
```

If your SDK writes consumer data into the DOM, document the assumed trust level of inputs. Default position: assume untrusted, sanitize.

## `postMessage` and cross-origin

```javascript
// ❌ — accepts message from any origin, trusts payload
window.addEventListener("message", (e) => {
  handle(e.data);
});

// ✅ — origin allowlist + structured message validation
const ALLOWED_ORIGINS = new Set(["https://app.example.com", "https://staging.example.com"]);

window.addEventListener("message", (e) => {
  if (!ALLOWED_ORIGINS.has(e.origin)) return;
  if (e.data?.protocol !== "my-sdk" || typeof e.data.type !== "string") return;
  handle(e.data);
});

// ❌ — leaks data to any window
otherWindow.postMessage(secret, "*");

// ✅ — explicit target origin
otherWindow.postMessage(secret, "https://app.example.com");
```

**Anti-patterns:**
- Origin check via `includes` or `startsWith` — `https://evil.example.com.attacker.com`.startsWith("https://evil.example.com") would be true with naïve checks. Use exact match.
- Trusting `event.source.location` — you can't read it cross-origin.
- Forgetting `event.isTrusted` — irrelevant for `MessageEvent`; only set by user gestures on UI events.

## Subresource Integrity (SRI) for CDN consumers

Already covered in `distribution.md`. Quick reminder: consumers using `<script>` from a CDN should add `integrity=` so a CDN compromise can't inject malicious code into apps that pinned a specific version.

```html
<script
  src="https://cdn.jsdelivr.net/npm/my-sdk@1.2.3/dist/index.iife.js"
  integrity="sha384-..."
  crossorigin="anonymous"
></script>
```

Generate at release time. Surface in README + CHANGELOG.

## Supply chain hygiene

### Publishing

```bash
# 1. Always provenance
npm publish --provenance --access public

# 2. Lockfile committed
git add package-lock.json && git commit -m "chore: update lockfile"

# 3. Audit before publish
npm audit --omit=dev --audit-level=high
```

### Dependencies

- **Minimize dependencies.** Every dep is a potential supply-chain vector. A 50-line utility is cheaper to copy than to import.
- **Pin major + minor in `dependencies`.** Use `^` only when you trust the upstream's semver discipline.
- **`peerDependencies` for things consumers should provide.** Don't bundle things that risk version mismatch.
- **Run `npm audit` weekly** in CI. Fail builds on `--audit-level=high`.

### `package.json` hardening

```json
{
  "name": "my-sdk",
  "version": "1.0.0",
  "type": "module",
  "files": ["dist", "README.md", "LICENSE"],   // never ship more than needed
  "publishConfig": {
    "access": "public",
    "provenance": true
  },
  "engines": {
    "node": ">=18"
  },
  "scripts": {
    "preinstall": "echo 'no preinstall scripts'",  // hint: avoid lifecycle scripts
    "prepublishOnly": "npm run build && npm test"
  }
}
```

**Avoid `postinstall` scripts.** They're a known supply-chain attack vector and many security policies block packages that have them. If you must, document loudly.

## Secret handling

### What counts as a secret

- API keys, OAuth tokens, JWTs, refresh tokens
- Anything passed in `Authorization` header
- Encryption keys, signing keys
- DB connection strings, deeply held config

### Rules

```javascript
// ❌ — secret in localStorage (XSS reads it)
localStorage.setItem("apiKey", apiKey);

// ✅ — keep secrets in memory only when possible
class Client {
  #apiKey;  // private field, not enumerable
  constructor({ apiKey }) { this.#apiKey = apiKey; }
}

// ❌ — secret in error message / logs
throw new Error(`Auth failed for token ${apiKey}`);
console.log("Request:", { url, headers });  // headers contain Authorization

// ✅ — redact in errors and logs
throw new SDKError("auth failed", "AUTH_FAILED", {
  details: { url, /* no headers */ },
});

function redactHeaders(h) {
  const copy = { ...h };
  if (copy.Authorization) copy.Authorization = "[REDACTED]";
  if (copy["X-Api-Key"])  copy["X-Api-Key"]  = "[REDACTED]";
  return copy;
}

// ❌ — secret in URL (logged everywhere)
fetch(`https://api.example.com/x?api_key=${apiKey}`);

// ✅ — secret in header
fetch("https://api.example.com/x", { headers: { Authorization: `Bearer ${apiKey}` } });
```

### Browser SDK reality check

In a public browser SDK, the API key IS publicly visible in the bundle. That's fine for *intended* public keys (like Stripe's publishable key, segment.io's write key). It is NOT fine for keys that grant elevated privileges. Document loudly: "this SDK's `apiKey` is a public-safe key; never use a server-side key here."

## Random and crypto

```javascript
// ❌ — Math.random() is NOT cryptographically secure
const sessionId = Math.random().toString(36);

// ✅ — Use Web Crypto
const sessionId = crypto.randomUUID();
const tokenBytes = crypto.getRandomValues(new Uint8Array(32));

// ✅ — Hashing
const buf = new TextEncoder().encode(input);
const hash = await crypto.subtle.digest("SHA-256", buf);

// ✅ — HMAC signature
const key = await crypto.subtle.importKey(
  "raw",
  new TextEncoder().encode(secret),
  { name: "HMAC", hash: "SHA-256" },
  false,
  ["sign"]
);
const sig = await crypto.subtle.sign("HMAC", key, new TextEncoder().encode(message));
```

## Regular expressions — ReDoS

```javascript
// ❌ — catastrophic backtracking on certain inputs
const EMAIL = /^([a-zA-Z0-9._-]+)+@example\.com$/;
EMAIL.test("aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa!"); // hangs

// ✅ — avoid nested quantifiers; bound input lengths
function isEmail(input) {
  if (input.length > 254) return false;
  return /^[a-zA-Z0-9._-]+@example\.com$/.test(input); // no nested +
}
```

For untrusted input, length-bound first, then regex. Or skip regex and parse explicitly.

## Time and randomness as side channels

If your SDK does anything crypto-adjacent (signing, comparing tokens):

```javascript
// ❌ — early-exit comparison leaks length and prefix info
function tokensEqual(a, b) {
  if (a.length !== b.length) return false;
  for (let i = 0; i < a.length; i++) if (a[i] !== b[i]) return false;
  return true;
}

// ✅ — constant-time comparison
function constantTimeEqual(a, b) {
  if (typeof a !== "string" || typeof b !== "string" || a.length !== b.length) return false;
  let diff = 0;
  for (let i = 0; i < a.length; i++) diff |= a.charCodeAt(i) ^ b.charCodeAt(i);
  return diff === 0;
}
```

## Quick Reference

| Concern | Rule |
|---------|------|
| `eval` / `new Function` | Never. CI lint rule mandatory |
| Recursive option merge | Filter `__proto__`/`constructor`/`prototype`; or `Object.create(null)` |
| `innerHTML` | Avoid; use `textContent` or sanitize via DOMPurify |
| `postMessage` recv | Origin allowlist + protocol/version check |
| `postMessage` send | Specify target origin, never `*` |
| CDN ship | Generate SRI hashes per release |
| `npm publish` | Always `--provenance --access public` |
| Lifecycle scripts | Avoid `postinstall`; document if unavoidable |
| Secrets in storage | Memory-only or HttpOnly cookies (server) |
| Secrets in errors/logs | Redact Authorization, X-Api-Key |
| Secrets in URL | Always headers, never query string |
| Random for security | `crypto.getRandomValues` / `crypto.randomUUID` |
| Token comparison | Constant-time |
| Regex on untrusted input | Length-bound first; avoid nested quantifiers |
