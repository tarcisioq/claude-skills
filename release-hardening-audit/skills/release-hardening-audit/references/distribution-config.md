# Distribution Config

Audit of headers, CSP, CORS, cookies, and Express/Fastify/Koa middleware that shapes the public surface at request time.

## Decision matrix — which checks apply

| Surface | Run these checks |
|---|---|
| SPA on CDN, no own backend | CSP, SRI on third-party `<script>`, security headers (set by host: Cloudflare / S3 + CloudFront / Vercel) |
| SPA + own Node API | Above + helmet on API + CORS allowlist + cookies + body limit + trust proxy |
| Pure Node API (no SPA) | helmet + CORS + cookies + body limit + trust proxy + JWT verification |
| npm package (browser) | CSP-safe code only (no `eval`, no `new Function`); no SRI / header concerns at this layer |

## Security headers — what to require

Audit on the live deployment with `curl -sI`. These are the headers that matter for a typical web app:

| Header | Required value (or pattern) | Severity if missing |
|---|---|---|
| `Strict-Transport-Security` | `max-age=15552000; includeSubDomains` minimum | HARDEN |
| `X-Content-Type-Options` | `nosniff` | HARDEN |
| `X-Frame-Options` | `DENY` or `SAMEORIGIN` (or use `frame-ancestors` in CSP) | HARDEN |
| `Referrer-Policy` | `no-referrer` / `strict-origin-when-cross-origin` | HARDEN |
| `Content-Security-Policy` | non-default (see CSP section below) | HARDEN (BLOCK if SPA loads third-party scripts without it) |
| `Permissions-Policy` | restrict at minimum: `camera=(), microphone=(), geolocation=(), payment=()` | NICE-TO-HAVE |
| `Cross-Origin-Opener-Policy` | `same-origin` for sites doing `postMessage` to popups | NICE-TO-HAVE |
| `Cross-Origin-Resource-Policy` | `same-origin` or `same-site` | NICE-TO-HAVE |

**Audit command:**
```bash
curl -sI https://app.example.com/ \
  | grep -iE '(strict-transport|x-content-type|x-frame|referrer-policy|content-security|permissions-policy|cross-origin-(opener|resource))'
```

## helmet for Express — defaults check

```js
import helmet from 'helmet';
app.use(helmet());        // ✅ defaults set the 7 essentials above (except CSP — see below)
```

### What helmet defaults DO and DO NOT cover

| What helmet defaults | Sets |
|---|---|
| ✅ | `Strict-Transport-Security`, `X-Content-Type-Options`, `X-DNS-Prefetch-Control`, `X-Download-Options`, `X-Frame-Options`, `X-Permitted-Cross-Domain-Policies`, `X-XSS-Protection: 0`, `Referrer-Policy`, `Cross-Origin-Embedder-Policy`, `Cross-Origin-Opener-Policy`, `Cross-Origin-Resource-Policy`, `Origin-Agent-Cluster` |
| ⚠️ Sets a **default CSP** since v6 | `default-src 'self'`; this often breaks SPAs without further config |
| ❌ | Does not handle CORS — use `cors` package separately |

### Common misconfig — CSP disabled wholesale

```js
// ❌ — disables CSP entirely because "it broke the SPA"
app.use(helmet({ contentSecurityPolicy: false }));
```

**Fix:** configure CSP per the SPA's actual needs:

```js
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'", "'nonce-{NONCE}'"],
    styleSrc: ["'self'", "'unsafe-inline'"],   // for SPAs with CSS-in-JS; lift to nonces if possible
    imgSrc: ["'self'", "data:", "https:"],
    connectSrc: ["'self'", "https://api.example.com"],
    frameAncestors: ["'none'"],
  },
}));
```

## CSP recipes

### SPA with inline styles (typical Vite/Next/Astro)

```
default-src 'self';
script-src 'self' 'strict-dynamic' 'nonce-{per-request-nonce}';
style-src 'self' 'unsafe-inline';
img-src 'self' data: https:;
connect-src 'self' https://api.example.com;
frame-ancestors 'none';
base-uri 'self';
form-action 'self';
```

`'strict-dynamic'` lets your nonced bootstrapper load child scripts without listing every CDN. Drops the need for hash allowlists.

### Static SPA on S3 + CloudFront (no per-request server)

Use hash-based CSP (Vite/Webpack plugins emit hashes):

```
default-src 'self';
script-src 'self' 'sha256-AbCd...' 'sha256-EfGh...';
style-src 'self' 'sha256-...';
```

### Tools to validate

```bash
# Mozilla Observatory
curl -s "https://observatory.mozilla.org/api/v2/scan?host=app.example.com" | jq .

# CSP Evaluator (Google) — paste header, get warnings
# https://csp-evaluator.withgoogle.com/
```

## SRI on third-party scripts

```html
<!-- ❌ no integrity → CDN compromise = your app compromised -->
<script src="https://cdn.example.com/lib.js"></script>

<!-- ✅ integrity hash + crossorigin -->
<script src="https://cdn.example.com/lib.js"
        integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
        crossorigin="anonymous"></script>
```

### Audit

```bash
grep -rE "<script[^>]*src=['\"]https?://" dist/ public/ \
  | grep -v 'integrity='
```

Each hit is HARDEN (BLOCK if the script is from a host you do not control AND the script runs in an authenticated context).

## CORS — origin allowlist patterns

```js
// ❌ — wildcard with credentials = CSRF cross-site enabled
cors({ origin: '*', credentials: true });
cors({ origin: true, credentials: true });
cors({ origin: (req, cb) => cb(null, true), credentials: true });

// ✅ — explicit allowlist
const allowed = ['https://app.example.com', 'https://admin.example.com'];
cors({
  origin: (origin, cb) => allowed.includes(origin) ? cb(null, true) : cb(new Error('CORS: origin not allowed')),
  credentials: true,
});

// ✅ — public API, no credentials, fine to wildcard
cors({ origin: '*' });   // only when credentials: false (default)
```

### Subtle — wildcard subdomain

`cors` does not accept `*.example.com` patterns. Implement via function:

```js
const isAllowed = (origin) => {
  if (!origin) return false;
  try {
    const { hostname } = new URL(origin);
    return hostname === 'example.com' || hostname.endsWith('.example.com');
  } catch { return false; }
};
cors({ origin: (origin, cb) => isAllowed(origin) ? cb(null, true) : cb(new Error('CORS')), credentials: true });
```

## Cookie flag matrix

```js
res.cookie('session', token, {
  secure: true,                     // HTTPS only
  httpOnly: true,                   // not readable from JS (XSS mitigation)
  sameSite: 'lax',                  // CSRF mitigation; 'strict' if you can afford it
  maxAge: 1000 * 60 * 60 * 24,      // bounded lifetime
  path: '/',
  // domain: '.example.com',        // omit for first-party-only — enables __Host- prefix
});
```

| Cookie kind | Required flags | Optional |
|---|---|---|
| Auth / session | `secure`, `httpOnly`, `sameSite=lax` (or `strict`) | `__Host-` prefix |
| CSRF token | `secure`, `sameSite=strict` (NOT `httpOnly` — JS reads it) | — |
| Theme / preference | `secure`, `sameSite=lax` | — |

### `__Host-` prefix

Forces `secure`, `path=/`, no `domain` → the browser refuses cookies that violate. Strongest first-party-only guarantee.

```js
res.cookie('__Host-session', token, { secure: true, httpOnly: true, sameSite: 'strict', path: '/' });
```

## Body parsing — limits

```js
// ❌ unbounded
app.use(express.json());
app.use(express.urlencoded());

// ✅ explicit
app.use(express.json({ limit: '100kb' }));
app.use(express.urlencoded({ limit: '100kb', extended: true }));
app.use('/api/upload', express.raw({ limit: '5mb', type: 'application/octet-stream' }));
```

| Endpoint type | Recommended `limit` |
|---|---|
| Form / JSON API | `100kb` |
| File upload | `5mb` (or whatever your storage tier supports) |
| Webhook | `1mb` (typical webhook payload size) |
| GraphQL | `1mb` (queries are small; payloads grow with mutation variables) |

## `trust proxy` configuration

Behind a reverse proxy (Cloudflare, nginx, ALB), Express needs to know how many hops to trust for `X-Forwarded-For` and `X-Forwarded-Proto`.

```js
// ❌ — trusts socket address (localhost in containerized deploy) for req.ip
const app = express();

// ❌ — trusts ANY value of X-Forwarded-For = spoofable
app.set('trust proxy', true);

// ✅ — trust exactly N proxies (count from client side inward)
app.set('trust proxy', 1);   // single Cloudflare layer
app.set('trust proxy', 2);   // Cloudflare → ALB

// ✅ — subnet-based
app.set('trust proxy', 'loopback, 192.168.0.0/16, 10.0.0.0/8');
```

### Verify

After deploy, hit the app and assert `req.ip` equals the actual client IP (use a debug endpoint that echoes `req.ip` and compare with what `curl ifconfig.me` returns from the client).

## Mixed content audit

```bash
# Find absolute http:// URLs (not localhost) in dist
grep -rE "http://(?!localhost|127\\.0\\.0\\.1)" dist/ | head
```

Each match is HARDEN — modern browsers block mixed content silently in many contexts.

## Verification commands

```bash
# Headers on live deployment
curl -sI https://app.example.com/ | grep -iE '(strict-transport|x-content|x-frame|referrer-policy|content-security|permissions-policy)'

# CORS behavior
curl -sI -H "Origin: https://attacker.example.com" https://app.example.com/api/me \
  | grep -iE '^access-control-allow-(origin|credentials)'
# Should NOT echo back attacker origin with credentials: true

# Cookie flags
curl -sI -X POST https://app.example.com/login -d 'user=x&pass=y' \
  | grep -iE '^set-cookie' \
  | grep -E 'Secure.*HttpOnly.*SameSite'
# Each cookie line should have all three flags

# Body limit
curl -X POST https://app.example.com/api/data \
  -H 'Content-Type: application/json' \
  --data-binary @<(head -c 1048576 /dev/urandom | base64)
# Should return 413 Payload Too Large
```
