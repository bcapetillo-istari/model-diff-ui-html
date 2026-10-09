# Iframe Capability Findings — 2026-10-08

Goal: determine what an HTML app running inside an iframe can do, to inform
building interactive applications in an iframe environment (Istari platform).

Everything below was verified with the tools in this repo
(`iframe-capability-tester.html`, `server.js`, Cloudflare quick tunnel) unless
marked **pending**. Local verification is done; the same tests still need to be
run inside the actual deployed platform iframe, since the host's sandbox
attributes and CSP can only be measured from in there.

## Confirmed capabilities

### Rendering & execution
- **JavaScript executes** inside the iframe (assuming the host doesn't set
  `sandbox` without `allow-scripts` — the tester's banner detects this case).
- **CSS renders** normally (verified via computed-style probe, not just
  stylesheet presence).

### Network: requests vs. readability (the key distinction)
- **Outbound requests generally go out.** `fetch`/XHR/image loads from an
  iframe reach external domains unless the host's CSP (`connect-src`,
  `img-src`) or network policy blocks them.
- **Reading the response is gated by CORS, not by the iframe.** JS can only
  read a cross-origin response if the target server opts in with
  `Access-Control-Allow-Origin`. This is standard browser behavior on every
  page — google.com is reachable but unreadable from any site's JS.
  - CORS-enabled APIs (most JSON APIs): fully readable, no workaround needed.
  - Non-CORS sites (e.g. google.com): request succeeds, response is opaque.
- A "CORS error" in the console therefore does NOT mean the iframe blocked the
  request. The tester's "Can I reach this URL?" button distinguishes
  reachable-with-full-access / reachable-but-unreadable / blocked.

### Reading non-CORS sites: server-side proxy (verified working)
- The browser cannot bypass CORS client-side. The solution is to make the
  request **outside the browser**: iframe JS calls our proxy with the target
  URL as a parameter; the proxy fetches the target server-to-server (CORS does
  not apply there) and returns the body with CORS headers friendly to our
  domain. The browser only ever sees a request to the proxy.
- Verified end-to-end: `server.js` proxy + `cloudflared` quick tunnel returned
  google.com's full HTML (84 KB) readable by JS, from the public internet.
- Public third-party proxies are not dependable (tested 2026-10-08:
  corsproxy.io now requires an API key, allorigins was down; jina reader
  `r.jina.ai` worked but returns markdown and is rate-limited). Use our own
  proxy.

### Credential forwarding (verified mechanics; design decision pending)
- **Explicit headers forward cleanly.** An `Authorization: Bearer <token>`
  header set by iframe JS passes through the proxy to the target — verified
  end-to-end through the tunnel (httpbin echo showed the header intact).
  `server.js` forwards all request headers except hop-by-hop headers,
  tunnel/CDN noise, and cookies.
- **Browser-managed credentials (session cookies) can NOT be forwarded, ever.**
  Cookies are scoped to the domain a request is sent to; a request to the
  proxy carries the proxy's cookies, never the target site's. No proxy design
  fixes this. Token-based auth is the viable path.
- The Istari platform's `Authorization: Bearer` token (observed in DevTools on
  host-page requests) is the forwardable kind. BUT: headers on the host page's
  requests are set by host-app JS and are **not inherited** by the iframe's
  requests. The iframe must be handed the token explicitly.

## Update 2026-10-09 — verified from inside the platform iframe

- **The Istari API allows CORS for the iframe's origin.** A direct (no proxy)
  fetch from the tester running in the deployed platform to
  `api.demo.istari.app` returned a readable HTTP 403 in ~120ms — the CORS
  check passed and JS could read the response body. **No proxy is needed for
  Istari API calls**; the proxy remains relevant only for third-party sites
  without CORS.
- The 403 means the request arrived unauthenticated. Remaining work is
  attaching credentials, two candidate paths (tester now supports both):
  1. **Bearer header** (likely correct — the platform's own requests use
     `Authorization: Bearer`): paste the token into the "Extra request
     headers" box and re-run CORS fetch.
  2. **Cookies**: "Send cookies (credentials: include)" checkbox. Only works
     if the API replies with `Access-Control-Allow-Credentials: true` plus a
     specific origin (not `*`), and the session cookie is `SameSite=None` or
     the iframe is same-site with the API (both under `istari.app` would
     qualify).

## Update 2026-10-09 (later) — auth path confirmed from inside the platform

- **Bearer header works: HTTP 200.** Direct fetch from the platform iframe to
  the Istari API with `Authorization: Bearer <token>` (token copied from
  DevTools) succeeded. End-to-end capability confirmed: the iframe can call
  the Istari API directly, no proxy, with full response readability.
- **Cookies do not work (403 with `credentials: include`).** The session
  cookie doesn't flow from the iframe context to `api.demo.istari.app`
  (cookie scope / SameSite / third-party blocking). Cookie auth is a dead end;
  token hand-off is the design.

### Eliminating manual token copy-paste: postMessage token broker

The iframe can't read the host's token itself — the host page must hand it
over. Recommended pattern (iframe side already implemented in the tester:
"Request auth token from host" button auto-fills the Authorization header):

The platform host page needs a listener like:

```js
// On the host (demo.istari.app) page:
window.addEventListener('message', (e) => {
  // 1. Only answer our own embedded apps:
  if (!ALLOWED_IFRAME_ORIGINS.includes(e.origin)) return;
  if (e.data?.type === 'capability-tester-request-token') {
    e.source.postMessage(
      { type: 'capability-tester-token', token: currentApiToken },
      e.origin  // 2. specific targetOrigin — never '*' with a real token
    );
  }
});
```

Security requirements for the real implementation:
- Host must allowlist which iframe origins may request tokens (`e.origin`).
- Host must post back with a specific `targetOrigin`, never `'*'`.
- Iframe should verify `e.origin` is the expected platform origin before
  trusting a received token.
- Tokens are short-lived JWTs — the host should re-send on refresh, or the
  iframe should re-request when it gets a 401.
- Consider scoping: ideally the platform mints a reduced-scope token for
  embedded apps rather than handing over the user's full session token.

Alternatives if postMessage brokering isn't adopted: token as a query param
on the iframe `src` (leaks into logs/history), or platform templates the
token into the served iframe HTML (only if the platform serves the HTML).

## Update 2026-10-09 (capstone) — the platform's actual iframe sandbox

Observed attributes on the platform's embedding iframe:

```html
<iframe sandbox="allow-scripts allow-forms allow-popups allow-modals"
        title="HTML Content Viewer"
        srcdoc="<!DOCTYPE html>...our uploaded HTML...">
```

Two details decide the whole auth story:

1. **`srcdoc` (not `src`)** — our HTML is injected inline by the platform as a
   string. There is no URL to append a token to, so "token as a query param on
   src" is impossible here.
2. **`sandbox` omits `allow-same-origin`** → the iframe runs in an **opaque
   origin** (`origin === "null"`). Therefore `localStorage`, `sessionStorage`,
   cookies, and IndexedDB all throw — **no client-side storage at all**, so a
   token cannot be stashed and reused across loads. (This is the correct secure
   config: `allow-scripts` + `allow-same-origin` together would defeat the
   sandbox, so the platform rightly drops `allow-same-origin`.)

**Consequence: only two channels can get a token into this iframe, and both
require the platform** —
- inject it into the `srcdoc` the platform already generates, or
- `postMessage` from the parent (works through the opaque origin; parent sees
  `event.origin === "null"`).

There is no client-only workaround; the sandbox is specifically built to deny
the iframe any self-service path to a credential. The CORS Bearer call itself
works from the opaque origin (empirically got a 200), so the only missing piece
is credential delivery.

### Recommended platform ask: inject a short-lived token into `srcdoc`

Cleanest option given the constraints. When the platform renders the wrapper,
have it define a JS variable holding a **short-lived access token** (minted
server-side — the server does the ES384 signing/exchange with the private key;
the browser never sees the key). Convention the uploaded HTML expects:

```html
<!-- platform injects this line into the srcdoc, before our script -->
<script>window.__ISTARI_ACCESS_TOKEN__ = "<short-lived-token>";</script>
```

The uploaded HTML then reads `window.__ISTARI_ACCESS_TOKEN__` and attaches it
as `Authorization: Bearer <token>`. Terminology note: the iframe is *presenting*
a pre-signed bearer token, not signing — the signing happened server-side when
the token was minted. Pair with the postMessage broker for refresh on 401,
since an opaque-origin iframe can't refresh a token on its own.

Interim stopgap (no platform change): hardcode a SHORT-LIVED token as that
variable in the uploaded HTML. It persists a credential in stored platform
content — use the shortest lifetime available, never the access key, never a
long-lived PAT if avoidable, and rotate/revoke after testing.

## Open questions / next steps

1. ~~Run the tester inside the actual platform iframe~~ — **done for network;**
   still worth capturing the tester's sandbox/storage/permissions-policy
   sections from in there.
2. **Token hand-off design for Istari SDK calls** — how does the iframe get
   the Bearer token? Candidates:
   - Host page `postMessage`s the token to the iframe after load (tester's
     postMessage test shows whether the host talks to iframes today).
   - Token passed as a query parameter on the iframe `src`.
   - Platform templates the token into the served HTML.
   For testing now: copy the token from DevTools into the tester's
   "Extra request headers" box.
3. ~~Does the Istari API allow CORS?~~ — **Yes, verified 2026-10-09** (see
   update above). Direct calls work; confirm the preflight also allows the
   `Authorization` header by re-running CORS fetch with the token attached.
4. **Production proxy** — the laptop + quick tunnel setup is test-only (URL
   rotates on restart, open relay while up, no uptime). If a proxy is needed
   in production, deploy a small service (e.g. Cloudflare Worker) locked down
   to allowlisted target domains and our iframe's origin.

## Security notes
- Bearer tokens are usually short-lived JWTs — a 401 mid-testing likely means
  expiry, not a new block.
- Kill the `cloudflared` tunnel when not testing; while up, the proxy is an
  open relay that forwards whatever auth headers it receives.
- Never paste real tokens into third-party public proxies; if one transits
  them, rotate it.

## Repo tooling (see README.md for usage)
- `iframe-capability-tester.html` — self-contained probe page; deploy into the
  target iframe. Tests rendering, storage, network reachability, proxy fetch
  with custom headers, postMessage, popups/modals/clipboard/fullscreen, device
  permissions, sandbox detection, permissions-policy introspection.
- `host.html` — local harness with toggleable `sandbox`/`allow` attributes.
- `server.js` — static server + CORS proxy with header forwarding
  (localhost:8080, expose via `cloudflared tunnel --url http://localhost:8080`).


# Next Steps

See if I can call a job from the HTML. Example authorization-gated API call: 

https://api.demo.istari.app/registry/api/v2/models/48ffd05e-4f85-41c4-846e-d6eb0ab255bf/jobs?function_name=%40istari%3Aextract&tool_name=open_pdf&tool_version=1.0.0&operating_system=Ubuntu%2022.04

## Results (10/09/2026)

Was able to call job w/200 manually copy/pasting the Authorization: Bearer <token> from my session (from the network tab), or by adding a PAT in place of <token>. This succeeded with a 200: *Jobs are able to be called from the Iframe and responses are able to be read w/out proxy server.* We still have to manually pass tokens into the HTML (acceptable for now), with the option of working with Cyber/platform teams to inject the current session's scoped token into the srcdoc so it is accessible.


