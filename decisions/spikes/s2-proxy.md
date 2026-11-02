# Spike S2: Proxy and Grafana

**Date:** 2026-11  
**Status:** Partial: desk research done; the hands-on checks wait for the BIOS update  
**Plan:** [§9.1](https://github.com/raisga/p4n4/blob/main/RUST_REWRITE_PLAN.md#91-spikes-p0)  
**Code:** none yet; `spikes/s2-proxy/` in the superproject once the hands-on part starts

## Questions

From the plan's S2 row:

- Port `default.conf.template` to the axum proxy rule by rule.
- Check each upstream through it: the Node-RED editor's WebSocket, n8n's editor and push connection, Grafana under `/grafana/`, MQTTX Web under `/svc/mqtt-ui/` connecting through `/mqtt`, MJPEG and go2rtc camera pages, and `/demo/` video with range requests.
- Check whether Grafana's auth proxy can take the role from a header.

It decides the proxy implementation, the settings each upstream needs, and Grafana role mapping.

## Setup

This part was desk research, done in 2026-11. No code was compiled or run, and no container was touched, so every check is "not run" until the hands-on run. The sources:
- the v0.4.2 baseline at the superproject's pins: dashboard v1.5.0 (`a2629a75`), stacks/iot (`d80ea1b0`), stacks/ai (`cb23a02e`) and tools/templates (`30ace573`);
- each upstream's source and docs at the tag its image resolves to;
- Docker Hub's tags API (`https://hub.docker.com/v2/repositories/<repo>/tags/<tag>`), which resolves floating tags without Docker;
- the crates.io API, and crate sources unpacked from `static.crates.io`.

| Upstream | Pinned at | Tag | Resolves to (last push) |
|---|---|---|---|
| Grafana | `stacks/iot/docker-compose.yml:177` | `grafana/grafana:13.0` | 13.0.12 (2026-10-27) |
| Node-RED | `stacks/iot/docker-compose.yml:120` | `nodered/node-red:5.0` | 5.0.9 (2026-10-30) |
| Mosquitto | `stacks/iot/docker-compose.yml:28` | `eclipse-mosquitto:2.0` | 2.0.22 (2026-10-23) |
| MQTTX Web | `stacks/iot/docker-compose.yml:309` | `emqx/mqttx-web:v1.13.1` | itself |
| n8n | `stacks/ai/docker-compose.yml:78` | `n8nio/n8n:2.41.6` | itself |
| go2rtc | `tools/templates/projects/mqtt-influx-grafana-ollama-go2rtc/extras/docker-compose.yml:16` | `alexxit/go2rtc:1.9.14` | itself |

The multi-platform index digests, for pinning the hands-on run:

```text
grafana/grafana:13.0.12@sha256:28476aa8096cd8d0f658a05a443b605fffb450e57f444335216175d604cfd23d
nodered/node-red:5.0.9@sha256:2ccd38b4066648bd5e17430c50a2f8102c566a82fd4f2fc5c342e61664d6896f
eclipse-mosquitto:2.0.22@sha256:09b62ff32c2e7b64d4b431b9a89236073289b68db273de2b089edeb9b0d92b5c
emqx/mqttx-web:v1.13.1@sha256:e777a2221a4e92ea0b87b3170a96ae09d4e557d2dc80c694175669762f47cfd0
n8nio/n8n:2.41.6@sha256:87e0bab2c93192e8dd885ff7b0697c22a1bd97489568a8c67cc140fd7dbb342d
alexxit/go2rtc:1.9.14@sha256:675c318b23c06fd862a61d262240c9a63436b4050d177ffc68a32710d9e05bae
```

- **Other images.** `stacks/extras` pins only OBD-II images. go2rtc, from the template above, is the baseline's only MJPEG server, and may run on another machine "at the site" (the template's compose file, L4–9). The dashboard's nginx serves `/demo/` itself.
- **Today's proxy.** `nginxinc/nginx-unprivileged:1.29-alpine` (`dashboard/Dockerfile:45`), behind Caddy `caddy:2-alpine` for TLS (`dashboard/docker-compose.tls.yml:11`).
- **Crates read.** The current axum 0.8.9, hyper 1.12.0, hyper-util 0.1.21, tower-http 0.7.1 and tokio-tungstenite 0.30.0, and the proxy candidates [below](#the-proxy-implementation).

## Results

### Q1: the nginx configuration, rule by rule

The files:
- **T**: `default.conf.template`;
- **C**: `proxy_common.conf`;
- **E**: `embed_proxy.conf`;
- **S**: `security_headers.conf`;
- **D**: `dashboard/Dockerfile`.

The first four are in `dashboard/docker/nginx/`. The axum routes are from the [§6.4](https://github.com/raisga/p4n4/blob/main/RUST_REWRITE_PLAN.md#64-server-p4n4-server) Routing table.

| # | nginx rule | What it does | axum route | What the port needs | Result |
|---|---|---|---|---|---|
| 1 | D:57–70: rendering the template | envsubst fills in the variables that are set, at start. The API upstream defaults to `http://host.docker.internal:8000`, the others to empty | none | Routes built at start from `p4n4.toml` and the services' `ui` entries | not run |
| 2 | `18-upstream-hosts.envsh` | Swaps `extra_hosts` names in upstream URLs for IPs, which Docker's DNS doesn't serve | none | Nothing: upstreams are `p4n4-net` names | not run |
| 3 | T:78–79, `19-basic-auth.envsh` | Optional site-wide basic auth from `DASHBOARD_BASIC_AUTH` | none: the p4n4 login replaces it (§12.2) | The embed-cookie check on `/svc/`, `/grafana/`, `/demo/` and `/mqtt` | not run |
| 4 | T:14–21 (maps), C:8–9, E:12–13 | Drops Basic credentials; restores the API token from `X-Upstream-Authorization` | none (Appendix A: no longer needed) | The per-mount credential policy (finding 1) | not run |
| 5 | T:29–33 (map), T:135 | Node-RED requests with a bearer token skip basic auth, except its login, `auth/token` | none | Rate-limit `/svc/node-red/auth/token` | not run |
| 6 | T:37–44 (maps), E:26–27 | Adds `nosniff` and `Referrer-Policy: same-origin` to embedded pages that lack them | none | `SetResponseHeaderLayer::if_not_present` per mount | not run |
| 7 | T:47–50 (map), E:15–16 | WebSocket upgrades: forwards `Upgrade`, and `Connection: upgrade` only when the client asked | `/svc/<id>/`, `/grafana/`, `/mqtt` | HTTP/1.1 upgrades on both sides; `Sec-WebSocket-*` passed both ways | not run |
| 8 | T:53–56 (map), E:8 | `X-Forwarded-Proto` from Caddy's header, else the scheme; a client can forge it | none (§6.4 Proxies sets `X-Forwarded-*`) | XFP from the server's scheme or a `trusted_proxies` peer; client `Forwarded` and `X-Forwarded-*` dropped | not run |
| 9 | T:62–64 (map), `19-frame-src.envsh`, S:13 | The dashboard CSP's `frame-src`: itself, the direct origins of Grafana, Node-RED, InfluxDB, n8n and the API, and `DASHBOARD_FRAME_ORIGINS` | none; §12.2 maps `DASHBOARD_FRAME_ORIGINS` to `frame_origins`, which §5.2 makes frame-ancestors | Separate `frame_src` and `frame_ancestors` lists | not run |
| 10 | T:66–69 (`server`) | `listen 8080` for any Host, rooted at the Flutter build; published on 8088 | `[server] port = 8088` | Accept any Host | not run |
| 11 | T:73 (`resolver`) | Resolves upstreams per request (cached 10 s), so nginx starts while stacks are down | none | Resolve per connection; a stopped upstream gives 502 | not run |
| 12 | T:75 and S, server level | `nosniff`, `Referrer-Policy: same-origin` and the dashboard CSP (`default-src 'self'`, `script-src 'self' 'wasm-unsafe-eval'`, `frame-ancestors 'self'`, …). By `add_header` inheritance, only `/healthz`, `/api/` and `/health` get them from here | none | The same headers on `/`, `/config.json`, `/demo/` and the API; never on proxied mounts | not run |
| 13 | T:81–86 (`= /healthz`) | 200 `ok`, without auth or logging; the image's healthcheck | `/health` | Alias it, or drop it | not run |
| 14 | T:90–95 (`= /config.json`) | The app's defaults (`host`, `apiBase`, `grafanaBase`, `serviceUrls`), inserted unescaped; `no-store` and S | `/config.json` | Built with serde; `no-store` and S | not run |
| 15 | T:98–105 (`/api/`), with C | Proxies p4n4-api with its own Host, XFF, XFP, XFH and a 5 s connect timeout. No buffering and a 600 s read timeout, so assistant replies stream | `/api/v2/…`, in-process | No proxy; streaming handlers keep 600 s, exempt from request timeouts | not run |
| 16 | T:106–110 (`= /health`) | Proxied to p4n4-api, with C | `/health` | In-process | not run |
| 17 | T:116–123 (`/grafana/`), with E | 404 while `GRAFANA_UPSTREAM` is empty (L118–120); the path passes through | `/grafana/` | Pass through to `grafana:3000`, keeping the Host; auth-proxy headers on every request and handshake; `Authorization`, `Cookie` and `X-WEBAUTH-*` stripped | not run |
| 18 | T:134–142 (`/svc/node-red/`), with E | Row 5's realm; 404 while unset; the path passes through | `/svc/<id>/` | Pass through; upgrades at `/svc/node-red/comms`; the bearer token forwarded; a body limit of at least 5 MiB | not run |
| 19 | T:143–151 (`/svc/n8n/`), with E | 404 while unset; `rewrite ^/svc/n8n/(.*)$ /$1 break` strips the prefix | `/svc/<id>/` | Strip the prefix; push upgrades that pass n8n's Origin check; `n8n-auth` forwarded; a 16 MiB body limit | not run |
| 20 | T:155–164 (`/swagger-ui`, `= /openapi.json`), with E | Proxied to p4n4-api with E's headers, so the Services tab can frame Swagger UI | `/api/v2/docs`, `/api/v2/openapi.json` | In-process (utoipa), with its own CSP | not run |
| 21 | T:169–176 (`/demo/`) | A nested, case-sensitive regex (L170) serves only names of `[A-Za-z0-9_-]+` with the extension mp4, m4v, webm, ogv or mov, from `DASHBOARD_DEMO_DIR`, with `no-cache` and S; anything else is 404. nginx answers range requests | `/demo/` | `ServeDir` behind the embed cookie, plus the name filter; never compressed | not run |
| 22 | T:179–184 (entry points) | `index.html`, `flutter_bootstrap.js`, `flutter_service_worker.js`, `manifest.json` and `version.json`, matched case-insensitively: `no-cache`, S, and a fallback to `index.html` | `/` | `no-cache`, with ETag and Last-Modified | not run |
| 23 | T:186–193 (`/`) | Everything else: `no-cache`, because Flutter's file names aren't hashed (T:186–188); S; SPA fallback (`try_files $uri $uri/ /index.html`) | `/` ("Hashed assets are served as immutable") | Keep `no-cache`. No SPA fallback under `/api/`, `/svc/`, `/grafana/`, `/demo/` or `/mqtt`: today an unknown `/svc/x/` gets `index.html` | not run |
| 24 | T:195–196 (`gzip`) | Compresses JavaScript, JSON, wasm, CSS, SVG and HTML, proxied responses included | none (Appendix C lists no compression) | tower-http compression for the same types only (finding 4) | not run |
| 25 | E:4–9 | Embedded upstreams get the browser's Host ("Grafana and n8n check Origin against Host", E:5), XFF, row 8's XFP, and XFH | `/svc/<id>/`, `/grafana/` | Preserve the Host; set XFH and XFP after dropping client values | not run |
| 26 | E:17 | `proxy_read_timeout 300s`: a page or WebSocket closes after 300 s without upstream data | none (Hardening: "Timeouts") | A 300 s idle timeout per mount, in both directions once upgraded | not run |
| 27 | E:21–23 | Replaces upstream `X-Frame-Options` with `SAMEORIGIN`; adds CSP `frame-ancestors 'self'` beside any upstream policy | `/svc/<id>/`, `/grafana/` (Proxies: Framing) | The same, plus `[server] frame_origins` | not run |
| 28 | nginx defaults | 1 MiB body limit; buffering on, except under `/api/`; 60 s timeouts. No `Location` rewriting ([`proxy_redirect default`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_redirect) can't apply to a variable `proxy_pass`), cookie-path rewriting, `sub_filter` or `X-Forwarded-Prefix` | Hardening: per-route limits and timeouts | A body limit (1 MiB by default) and timeouts per mount, 5 s to connect. No rewriting while each upstream knows its sub-path | not run |
| 29 | `dashboard/docker/caddy/Caddyfile` L6–13 | TLS from Caddy's own CA (`internal`) or Let's Encrypt, `encode gzip`, a year of HSTS, `flush_interval -1`; HTTP/2 to browsers | §6.4 TLS | rustls with ALPN `h2` and `http/1.1`; extended CONNECT off (finding 5); an HSTS policy | not run |

**nginx behaviour with no counterpart in the plan:**
- the dashboard's own CSP, `nosniff` and `Referrer-Policy` (rows 12, 14 and 21–23); the plan's Framing paragraph covers proxied pages only;
- `no-store` on `/config.json`, and `no-cache` on every static file;
- headers added when absent (row 6), and Host preservation (row 25);
- the 600 s, 300 s and 5 s timeouts, and the 1 MiB body default;
- the compressed types (row 24), `/healthz`, the `frame-src` list (row 9), and the basic-auth lock on Node-RED's login (row 5);
- HSTS and HTTP/2, from Caddy.

**Plan routes with no nginx rule:**
- `/mqtt`: browsers reach Mosquitto's port 9001 directly today;
- `/svc/mqtt-ui/`: MQTTX Web runs on its own port, 8081;
- `/svc/camera-<id>/` and the vision endpoints: cameras load cross-origin today, which `img-src *` and `media-src http: https:` allow;
- `/ready`, `/api/v2/version` and `/metrics`;
- the embed-cookie check, and Grafana's auth-proxy headers.

**Findings that change the plan** (the proposed edits are under [Follow-ups](#follow-ups)):
1. **Credentials.** §6.4 Proxies never forwards `Authorization` or `Cookie`, but Node-RED's editor needs its bearer token, and n8n's its `n8n-auth` cookie. Proposed, per mount: always strip p4n4's own tokens (`p4a_`, `p4r_`, `p4e_`, `p4d_`) and embed cookies; strip every credential for Grafana; pass the rest to Node-RED and n8n.
2. **Host.** Grafana Live, n8n push and go2rtc check the Origin against the Host or the forwarded host, so the Host must be preserved.
3. **`/mqtt` gets no embed cookie:** the cookies' paths are `/svc/`, `/grafana/` and `/demo/` (§6.4 "Embed cookie").
4. **Compression.** tower-http's default predicate excludes only images, gRPC and SSE ([predicate.rs, L117–128](https://docs.rs/crate/tower-http/0.7.1/source/src/compression/predicate.rs)), so it would compress MJPEG and video; use an allowlist.
5. **Extended CONNECT.** If the server advertises RFC 8441 on HTTP/2, browsers may open WebSockets as CONNECT requests, which the leading crate answers with 501; keep it off.

**Left for the hands-on run:**
- One prototype mount per upstream with the settings in Q2. Check each row: headers both ways, upgrades, idle timeouts, body limits, a 404 for a disabled service and a 502 for a stopped one.
- `curl -sD-` on every route, against nginx and the prototype, to diff the headers.
- Browsers over HTTP/1.1, then over TLS with HTTP/2.

### Q2: what each upstream needs under its sub-path

#### Node-RED 5.0.9

- **Path.** `httpAdminRoot` comes from `NODE_RED_ADMIN_ROOT` (`stacks/iot/config/node-red/settings.js:44`), which defaults to `/` (`stacks/iot/docker-compose.yml:141`). Set to `/svc/node-red/`, it makes Node-RED serve under the prefix, so the path passes through.
- **Editor WebSocket.** `httpAdminRoot` + `comms`, matched exactly ([comms.js, L218–249](https://github.com/node-red/node-red/blob/5.0.9/packages/node_modules/@node-red/editor-api/lib/editor/comms.js#L218-L249)). The client builds the URL from `location` ([client comms.js, L49–56](https://github.com/node-red/node-red/blob/5.0.9/packages/node_modules/@node-red/editor-client/src/js/comms.js#L49-L56)). The server pings every 15 s and checks no Origin, and the client sends its token as the first message.
- **Credentials.** The admin API takes `Authorization: Bearer`, issued by `POST auth/token`. There's no `tokenHeader` unless `adminAuth.tokens` is set ([users.js, L110–117](https://github.com/node-red/node-red/blob/5.0.9/packages/node_modules/@node-red/editor-api/lib/auth/users.js#L110-L117)).
- **Body.** `apiMaxLength` is 5 MB ([editor-api index.js, L58–60](https://github.com/node-red/node-red/blob/5.0.9/packages/node_modules/@node-red/editor-api/lib/index.js#L58-L60)). Today nginx's 1 MiB default caps the size of a deploy.
- **`httpNodeRoot` is `/`** (settings.js:45), so HTTP-in endpoints and node UIs stay outside the mount, reachable only on 1880, which §12.2 unpublishes. FlowFuse Dashboard at `/dashboard`, the compose file's example package (L147–148), is one.

**Needs:** the path passed through, with `NODE_RED_ADMIN_ROOT=/svc/node-red/`; the WebSocket; `Authorization` forwarded; a body limit of at least 5 MiB.

#### n8n 2.41.6

- **Sub-path.** Supported through `N8N_PATH`. At start, n8n writes the path into the editor's files ([start.ts, L161–186](https://github.com/n8n-io/n8n/blob/n8n@2.41.6/packages/cli/src/commands/start.ts#L161-L186)) but still serves from its root, so the proxy strips `/svc/n8n/` (T:148). The docs warn that `N8N_PATH` with a reverse proxy "can cause folder navigation issues", and suggest a subdomain ([deployment variables](https://docs.n8n.io/deploy/host-n8n/configure-n8n/basic-configuration/use-environment-variables/deployment); the docs aren't versioned). p4n4 has no subdomain to offer, so folders need a hands-on check.
- **URLs.** `N8N_EDITOR_BASE_URL` is `<public URL>/svc/n8n/`, and `N8N_WEBHOOK_URL` is the "successor to the deprecated `WEBHOOK_URL`" ([config index.ts, L281–287](https://github.com/n8n-io/n8n/blob/n8n@2.41.6/packages/@n8n/config/src/index.ts#L281-L287)). The ai stack still sets `WEBHOOK_URL=http://localhost:5678/` (`stacks/ai/docker-compose.yml:95`).
- **Push.** `N8N_PUSH_BACKEND` is `websocket` (the default) or `sse`; only the WebSocket enables collaboration ([server.ts, L241–245](https://github.com/n8n-io/n8n/blob/n8n@2.41.6/packages/cli/src/server.ts#L241-L245)). The path is `/rest/push?pushRef=…` ([push/index.ts, L70–97](https://github.com/n8n-io/n8n/blob/n8n@2.41.6/packages/cli/src/push/index.ts#L70-L97)), so `/svc/n8n/rest/push` in the browser.
- **Origin check.** In production, which the image sets ([Dockerfile, L54](https://github.com/n8n-io/n8n/blob/n8n@2.41.6/docker/images/n8n/Dockerfile#L54); [push/index.ts, L122–143](https://github.com/n8n-io/n8n/blob/n8n@2.41.6/packages/cli/src/push/index.ts#L122-L143)), the handshake's Origin must match the host from the first `Forwarded` element, else `X-Forwarded-Host` (with `X-Forwarded-Proto`), else `Host`; a missing Origin fails ([origin-validator.ts, L43–101](https://github.com/n8n-io/n8n/blob/n8n@2.41.6/packages/cli/src/push/origin-validator.ts#L43-L101)). A client-sent `Forwarded: host=…` would win, so the proxy must drop it.
- **Cookie.** `n8n-auth` has `Path=/` ([auth.service.ts, L290–296](https://github.com/n8n-io/n8n/blob/n8n@2.41.6/packages/cli/src/auth/auth.service.ts#L290-L296)) and is `Secure` by default ([auth.config.ts, L12–17](https://github.com/n8n-io/n8n/blob/n8n@2.41.6/packages/@n8n/config/src/configs/auth.config.ts#L12-L17)). Over plain HTTP on any host but localhost, the editor replaces itself with a warning ([settings.store.ts, L318–326](https://github.com/n8n-io/n8n/blob/n8n@2.41.6/packages/frontend/@n8n/stores/src/settings.store.ts#L318-L326)), as it does today, so with `tls = "off"` p4n4 must set `N8N_SECURE_COOKIE=false`.
- **Limits.** `N8N_PAYLOAD_SIZE_MAX` is 16 MiB ([endpoints.config.ts, L150–152](https://github.com/n8n-io/n8n/blob/n8n@2.41.6/packages/@n8n/config/src/configs/endpoints.config.ts#L150-L152)). `N8N_PROXY_HOPS=1` makes n8n read the client address from `X-Forwarded-For` ([index.ts, L266–268](https://github.com/n8n-io/n8n/blob/n8n@2.41.6/packages/@n8n/config/src/index.ts#L266-L268)).
- **Webhooks.** Telegram, WhatsApp and the web chat call n8n's webhooks (`stacks/ai/.env.example` L85–92). With 5678 unpublished (§12.2) and `/svc/n8n/` behind the embed cookie, they can't reach it.

**Needs:** the prefix stripped, with `N8N_PATH=/svc/n8n/`; `N8N_EDITOR_BASE_URL` and `N8N_WEBHOOK_URL` from a public URL; push over WebSocket, with Host, XFH, XFP and `Forwarded` matching the browser's origin; `n8n-auth` forwarded; a 16 MiB body limit; `N8N_SECURE_COOKIE=false` without TLS.

#### Grafana 13.0.12: the sub-path (the auth proxy is Q3)

- **Path.** With `GF_SERVER_SERVE_FROM_SUB_PATH=true` and a `GF_SERVER_ROOT_URL` ending in `/grafana/` ([defaults.ini, L63–66](https://github.com/grafana/grafana/blob/v13.0.12/conf/defaults.ini#L63-L66)), Grafana serves under the prefix, so the path passes through. The iot stack builds `root_url` on `localhost:3000` (compose L191–192), which breaks links in notifications; build it from a public URL.
- **Live.** `/grafana/api/live/ws` requires a signed-in user ([live.go, L387–389](https://github.com/grafana/grafana/blob/v13.0.12/pkg/services/live/live.go#L387-L389)), so the handshake needs the auth headers too. Its Origin check passes when the Origin's host equals the `Host` or `root_url`'s host, or matches `[live] allowed_origins` ([L535–585](https://github.com/grafana/grafana/blob/v13.0.12/pkg/services/live/live.go#L535-L585)). `[live] max_connections` is 100 ([defaults.ini, L2055](https://github.com/grafana/grafana/blob/v13.0.12/conf/defaults.ini#L2055)).
- **Framing.** With `allow_embedding=false` ([L434](https://github.com/grafana/grafana/blob/v13.0.12/conf/defaults.ini#L434)), Grafana sends `X-Frame-Options: deny` ([middleware.go, L60–65](https://github.com/grafana/grafana/blob/v13.0.12/pkg/middleware/middleware.go#L60-L65)). The proxy replaces it, so embedding can stay off. `cookie_samesite` (`lax`, [L431](https://github.com/grafana/grafana/blob/v13.0.12/conf/defaults.ini#L431)) needs no change: the frame is same-origin, and the auth proxy sets no Grafana session cookie.
- **CSRF.** Without that cookie (`enable_login_token=false`), Grafana skips its CSRF check ([csrf.go, L77–81](https://github.com/grafana/grafana/blob/v13.0.12/pkg/middleware/csrf/csrf.go#L77-L81)) unless `csrf_always_check` is set ([defaults.ini, L482](https://github.com/grafana/grafana/blob/v13.0.12/conf/defaults.ini#L482)). p4n4's SameSite=Strict embed cookie and the proxy's Origin check cover the gap.
- **Compression.** Grafana gzips its own responses ([L75](https://github.com/grafana/grafana/blob/v13.0.12/conf/defaults.ini#L75)); tower-http leaves encoded responses alone.

**Needs:** the path passed through, with `GF_SERVER_ROOT_URL=<public URL>/grafana/` and `GF_SERVER_SERVE_FROM_SUB_PATH=true`; the Host preserved; the auth headers on every request and handshake; `X-Frame-Options` replaced.

#### MQTTX Web v1.13.1, and Mosquitto 2.0.22 through `/mqtt`

- **MQTTX Web can't move to `/svc/mqtt-ui/` as built.** The image serves a prebuilt `dist` with `http-server` ([web/Dockerfile](https://github.com/emqx/MQTTX/blob/v1.13.1/web/Dockerfile)), built with `BASE_URL=/` ([.env, L6](https://github.com/emqx/MQTTX/blob/v1.13.1/web/.env#L6)) as webpack's `publicPath` ([vue.config.js, L6](https://github.com/emqx/MQTTX/blob/v1.13.1/web/vue.config.js#L6)), so its assets load from `/js/…` and `/css/…`. The router uses the URL hash ([router, L9](https://github.com/emqx/MQTTX/blob/v1.13.1/web/src/router/index.ts#L9)), so the base path is the only problem. The options: a p4n4 build with `BASE_URL=/svc/mqtt-ui/`; the server serving the built files itself; rewriting on the fly, which is fragile because webpack compiles the path in; or keeping port 8081, against §12.2. Saved connections live in the browser, per origin (compose L306–307), so the move loses them once.
- **Connecting.** MQTTX builds `ws(s)://host:port/path` ([mqttUtils.ts, L119–127](https://github.com/emqx/MQTTX/blob/v1.13.1/web/src/utils/mqttUtils.ts#L119-L127)), with the path `/mqtt` and a 60 s keepalive by default ([L167–183](https://github.com/emqx/MQTTX/blob/v1.13.1/web/src/utils/mqttUtils.ts#L167-L183)). Users enter the dashboard's host and port, and `wss` when TLS is on.
- **Mosquitto.** p4n4 writes `listener 9001` with `protocol websockets`, and the broker asks for a device login (`stacks/iot/config/mosquitto/bridge.sh:155–157`). The listener's subprotocols, `mqtt` and `mqttv3.1` (plus `http-only`), are chosen by `Sec-WebSocket-Protocol` ([websockets.c, L70–100](https://github.com/eclipse-mosquitto/mosquitto/blob/v2.0.22/src/websockets.c#L70-L100)), so that header must pass both ways. Mosquitto probably ignores the URL path (medium confidence).

**Needs:** an MQTTX build for the sub-path; `/mqtt` proxied to `mqtt:9001`, with `Sec-WebSocket-Protocol` passed both ways; an idle timeout above the keepalive; the embed cookie on `/mqtt`.

#### go2rtc 1.9.14

- **Where it runs.** The `mqtt-influx-grafana-ollama-go2rtc` template runs it as `video` on port 1984, possibly on another machine. Its `.p4n4.json` points the dashboard at `/api/stream.mjpeg?src=road`, and `src=plates` for the second camera (L27–34). The plan's camera entries have a `port` but no host (§5.2), so they can't point at another machine.
- **Path.** `api.base_path` prefixes every route ([api.go, L156–166](https://github.com/AlexxIT/go2rtc/blob/v1.9.14/internal/api/api.go#L156-L166)), but isn't needed: `stream.html` loads `./video-stream.js` and opens `new URL('api/ws?src=…', location.href)` ([stream.html, L31–60](https://github.com/AlexxIT/go2rtc/blob/v1.9.14/www/stream.html#L31-L60)). Both are relative, so stripping `/svc/camera-<id>/` works.
- **`/api/ws`.** Without an `origin` setting, it accepts a missing Origin, or one whose host equals the `Host`, port aside ([ws.go, L66–100](https://github.com/AlexxIT/go2rtc/blob/v1.9.14/internal/api/ws/ws.go#L66-L100)), so the Host must be preserved.
- **Modes.** The player tries `webrtc,mse,hls,mjpeg` ([video-rtc.js, L39](https://github.com/AlexxIT/go2rtc/blob/v1.9.14/www/video-rtc.js#L39)), and `stream.html` takes `mode=` from the query. MSE (over `/api/ws`), HLS and MJPEG go through the proxy. WebRTC media needs port 8555 over UDP or TCP, plus STUN ([webrtc README](https://github.com/AlexxIT/go2rtc/blob/v1.9.14/internal/webrtc/README.md)), which no HTTP proxy carries, so camera paths should add `&mode=mse`.
- **Security.** go2rtc also registers `api/config`, `api/restart`, `api/exit` and `api/log` ([api.go, L56–70](https://github.com/AlexxIT/go2rtc/blob/v1.9.14/internal/api/api.go#L56-L70)), with no login unless a username is set, and the template sets none (`video.sh` L59–62). A wholesale mount would let any viewer read the camera URLs, passwords included, or stop the service. go2rtc's `allow_paths` limits what it registers ([L22–37](https://github.com/AlexxIT/go2rtc/blob/v1.9.14/internal/api/api.go#L22-L37); medium confidence on the static files), and the proxy should allow only the player, `/api/ws`, `/api/stream.mjpeg` and `/api/frame.jpeg`.

**Needs:** the prefix stripped, with the Host kept; `/api/ws` upgrades; a path allowlist; `mode=mse`, or 8555 published; a host field for cameras.

#### MJPEG

go2rtc sends `multipart/x-mixed-replace; boundary=frame` and flushes each frame ([writer.go, L11–35](https://github.com/AlexxIT/go2rtc/blob/v1.9.14/pkg/mjpeg/writer.go#L11-L35)), with `Cache-Control: no-cache` ([mjpeg.go, L155](https://github.com/AlexxIT/go2rtc/blob/v1.9.14/internal/mjpeg/mjpeg.go#L155)). The web dashboard shows it in an `<img>` (`dashboard/lib/platform/html_view_web.dart:88–133`); the native app parses it (`dashboard/lib/widgets/mjpeg_view.dart`). Through the proxy:
- **Buffering.** hyper streams it frame by frame; only compression could buffer it (finding 4).
- **Timeouts.** Only an idle timeout may apply. axum-reverse-proxy's `with_upstream_timeout` covers only the response head.
- **Connections.** Over HTTP/1.1 a browser holds at most six connections per host ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)), and each MJPEG `<img>` keeps one. Cameras use their own origin (`:1984`) today; on the dashboard's origin, five or six would starve the API and SSE. HTTP/2 (the TLS modes), snapshots (`/api/frame.jpeg`, which `MjpegView` can poll) or MSE avoid this, and Hardening's per-user stream cap must leave room for a camera grid.

#### `/demo/` video

nginx serves `DASHBOARD_DEMO_DIR`, mounted at `/usr/share/nginx/demo` (`dashboard/docker-compose.yml:36`), with row 21's name filter. The web app plays the files in a `<video>` (`html_view_web.dart:138–212`), which sends `Range` requests. tower-http 0.7.1's `ServeDir` ([serve_dir/future.rs](https://docs.rs/crate/tower-http/0.7.1/source/src/services/fs/serve_dir/future.rs)) sends `Accept-Ranges`, `Last-Modified` and an ETag (L233–256) and answers one range with 206 (L258–283), but several with 416 where nginx sends `multipart/byteranges` (L285–297). It ignores `If-Range` and has no name filter. Compression drops `Accept-Ranges` and `Content-Length` ([compression/future.rs, L139–140](https://docs.rs/crate/tower-http/0.7.1/source/src/compression/future.rs)), so video stays out of the allowlist.

**Left for the hands-on run:**
- **Node-RED:** log in, deploy a flow over 1 MiB, and leave `comms` idle for 10 minutes.
- **n8n:** the editor; push through the Origin check, with the proxy's `Forwarded`; folders under `/svc/n8n/`; `N8N_SECURE_COOKIE` over HTTP.
- **Grafana:** Live and framing.
- **MQTTX:** a sub-path build, once compiling is allowed. Then `/mqtt` with `Sec-WebSocket-Protocol: mqtt`, and whether Mosquitto ignores the path.
- **go2rtc:** `stream.html` with `mode=mse`, the Origin check, and `allow_paths`.
- **MJPEG:** six cameras over HTTP/1.1, then over HTTP/2; a stalled camera against the idle timeout.
- **`/demo/`:** seeking in Chrome, Firefox and Safari; 206 and 304 answers.

### Q3: Grafana's auth proxy at 13.0.12

**Yes, the auth proxy can take the role from a header, and has since Grafana 8.1.0.** The `headers` setting accepted `Name`, `Email`, `Login` and `Groups` in 8.0.0 ([v8.0.0, L48](https://github.com/grafana/grafana/blob/v8.0.0/pkg/services/contexthandler/authproxy/authproxy.go#L48)), and 8.1.0 added `Role` ([v8.1.0, L48](https://github.com/grafana/grafana/blob/v8.1.0/pkg/services/contexthandler/authproxy/authproxy.go#L48)). In 13.0.12 the client is [proxy.go](https://github.com/grafana/grafana/blob/v13.0.12/pkg/services/authn/clients/proxy.go), and [`AuthenticateProxy` in grafana.go, L38–97](https://github.com/grafana/grafana/blob/v13.0.12/pkg/services/authn/clients/grafana.go#L38-L97) builds the identity.

**Settings.** From [defaults.ini, L976–985](https://github.com/grafana/grafana/blob/v13.0.12/conf/defaults.ini#L976-L985), [setting_auth_proxy.go, L22–45](https://github.com/grafana/grafana/blob/v13.0.12/pkg/setting/setting_auth_proxy.go#L22-L45) and the [v13.0 docs](https://grafana.com/docs/grafana/v13.0/setup-grafana/configure-access/configure-authentication/auth-proxy/):

| Setting | Default | Proposed | Notes |
|---|---|---|---|
| `GF_AUTH_PROXY_ENABLED` | `false` | `true` | |
| `GF_AUTH_PROXY_HEADER_NAME` | `X-WEBAUTH-USER` | the default | A request without it skips the auth proxy ([proxy.go, L147–148](https://github.com/grafana/grafana/blob/v13.0.12/pkg/services/authn/clients/proxy.go#L147-L148)) |
| `GF_AUTH_PROXY_HEADER_PROPERTY` | `username` | the default | The value is the login, and also the email if it parses as one |
| `GF_AUTH_PROXY_AUTO_SIGN_UP` | `true` | the default | Creates the Grafana user on the first visit |
| `GF_AUTH_PROXY_HEADERS` | empty | `Role:X-WEBAUTH-ROLE Name:X-WEBAUTH-NAME` | `Field:Header` pairs, split on spaces or commas. The fields are `Name`, `Email`, `Login`, `Role` and `Groups`. Empty headers are ignored |
| `GF_AUTH_PROXY_WHITELIST` | empty: any address | `172.20.0.2`, the `[network] server_ip` | IPs or CIDRs, checked against the TCP peer, not `X-Forwarded-For` ([proxy.go, L199–257](https://github.com/grafana/grafana/blob/v13.0.12/pkg/services/authn/clients/proxy.go#L199-L257)) |
| `GF_AUTH_PROXY_ENABLE_LOGIN_TOKEN` | `false` | the default | `true` issues a session cookie that outlives the p4n4 session |
| `GF_AUTH_PROXY_SYNC_TTL` | `15` (minutes) | the default | See Timing |
| `GF_AUTH_PROXY_HEADERS_ENCODED` | `false` | the default | Quoted-printable values, for non-ASCII text |

**Role values.** `None`, `Viewer`, `Editor` and `Admin`, matched case-sensitively ([role_type.go, L11–36](https://github.com/grafana/grafana/blob/v13.0.12/pkg/apimachinery/identity/role_type.go#L11-L36)); any other value, `viewer` included, counts as no role. **`GrafanaAdmin` isn't a role** but the separate server-admin flag, which the proxy never sets or clears: grafana.go passes no value for it (L81–87), and user sync skips a missing one ([user_sync.go, L525–530](https://github.com/grafana/grafana/blob/v13.0.12/pkg/services/authn/authnimpl/sync/user_sync.go#L525-L530)).

**Organisations.** The role applies to `DefaultOrgID()`: `auto_assign_org_id` when `auto_assign_org` is on, else 1 ([setting.go, L2423–2428](https://github.com/grafana/grafana/blob/v13.0.12/pkg/setting/setting.go#L2423-L2428)). Each sync removes the user from every other organisation unless they're its last admin ([org_sync.go, L67–122](https://github.com/grafana/grafana/blob/v13.0.12/pkg/services/authn/authnimpl/sync/org_sync.go#L67-L122)), so in effect there's one organisation.

**Without the role header,** `getRoles` returns nothing ([utils.go, L12–26](https://github.com/grafana/grafana/blob/v13.0.12/pkg/services/authn/clients/utils.go#L12-L26)). An existing user keeps their roles ([org_sync.go, L55–58](https://github.com/grafana/grafana/blob/v13.0.12/pkg/services/authn/authnimpl/sync/org_sync.go#L55-L58)). A new user gets `auto_assign_org_role` ([legacy_user.go, L184–197](https://github.com/grafana/grafana/blob/v13.0.12/pkg/services/user/userimpl/legacy_user.go#L184-L197)): `Viewer` by default ([defaults.ini, L579](https://github.com/grafana/grafana/blob/v13.0.12/conf/defaults.ini#L579)), and `Viewer` for any value but `None`, `Viewer`, `Editor` or `Admin` ([setting.go, L2034–2040](https://github.com/grafana/grafana/blob/v13.0.12/pkg/setting/setting.go#L2034-L2040)).

**Timing.** A role change applies on the next request. The cache key includes the extra headers' values, so a new role misses the cache and triggers a full sync ([proxy.go, L155–197](https://github.com/grafana/grafana/blob/v13.0.12/pkg/services/authn/clients/proxy.go#L155-L197); the comment at L172–177 says so). `sync_ttl` still bounds how long a role change made in Grafana's UI lasts ([org_users.go, L428](https://github.com/grafana/grafana/blob/v13.0.12/pkg/api/org_users.go#L428)), and how long a user deleted in Grafana stays locked out ([docs source, L236](https://github.com/grafana/grafana/blob/v13.0.12/docs/sources/setup-grafana/configure-access/configure-authentication/auth-proxy/index.md#L236)).

**`skip_org_role_sync`** doesn't exist for the auth proxy; only the OAuth providers, JWT and LDAP have it (defaults.ini, L762–1020; [configuration docs, L1128–1139](https://github.com/grafana/grafana/blob/v13.0.12/docs/sources/setup-grafana/configure-grafana/_index.md#L1128-L1139)). To let Grafana manage roles instead, leave `Role` out of `headers`.

**Caveats:**
- **The `admin` collision.** p4n4's bootstrap user is `admin` (§6.4 "Bootstrap"), and so is Grafana's built-in server admin (`GF_SECURITY_ADMIN_USER`, `stacks/iot/docker-compose.yml:185`). `X-WEBAUTH-USER: admin` would sign in as Grafana's server admin, and demoting the p4n4 user wouldn't clear the flag. Send `p4n4-<user id>`, which also survives renames, and put the username in `X-WEBAUTH-NAME`.
- **No server admin through the proxy.** The built-in admin stays for provisioning, the smoke tests (basic auth on port 3000, `tools/templates/projects/mqtt-influx-grafana/tests/smoke.sh:169,180`) and break-glass access inside `p4n4-net`.
- **Loose ends.** Deleting a p4n4 user leaves its Grafana user behind. Team Sync, from `Groups`, is Enterprise only ([docs source, L239–248](https://github.com/grafana/grafana/blob/v13.0.12/docs/sources/setup-grafana/configure-access/configure-authentication/auth-proxy/index.md#L239-L248)). Sign-out can't end a proxied session, so set `GF_AUTH_DISABLE_SIGNOUT_MENU=true` ([L661](https://github.com/grafana/grafana/blob/v13.0.12/conf/defaults.ini#L661)).
- **The whitelist** relies on the server's fixed IP; a second network on its container could change the source address.

**Proposed mapping,** from the p4n4 roles in §6.4 "Roles":

| p4n4 role | `X-WEBAUTH-ROLE` | In Grafana |
|---|---|---|
| `viewer` | `Viewer` | Dashboards, read-only |
| `operator` | `Editor` (leaning) or `Viewer`; P1 decides | Also edits dashboards and annotations, and uses Explore and alert silences |
| `admin` | `Admin` | Organisation admin: data sources, users and plugins. Never server admin |
| `device` | none | No embed cookie (`POST /auth/embed` requires `viewer`), so `/grafana/` answers 401 |

The environment p4n4 would set:

```yaml
environment:
  GF_SERVER_ROOT_URL: "<public URL>/grafana/"
  GF_SERVER_SERVE_FROM_SUB_PATH: "true"
  GF_AUTH_PROXY_ENABLED: "true"
  GF_AUTH_PROXY_HEADER_NAME: "X-WEBAUTH-USER"
  GF_AUTH_PROXY_HEADER_PROPERTY: "username"
  GF_AUTH_PROXY_AUTO_SIGN_UP: "true"
  GF_AUTH_PROXY_HEADERS: "Role:X-WEBAUTH-ROLE Name:X-WEBAUTH-NAME"
  GF_AUTH_PROXY_WHITELIST: "172.20.0.2"   # [network] server_ip
  GF_AUTH_PROXY_ENABLE_LOGIN_TOKEN: "false"
  GF_AUTH_PROXY_SYNC_TTL: "15"
  GF_USERS_AUTO_ASSIGN_ORG_ROLE: "None"    # fail closed
  GF_AUTH_DISABLE_SIGNOUT_MENU: "true"
  GF_SECURITY_ALLOW_EMBEDDING: "false"
```

**Left for the hands-on run:**
- Sign in as each p4n4 role and check the role Grafana shows. Then change the role and reload.
- The headers on the Live WebSocket. Does an open Live connection keep its old role? If so, closing tunnels at revocation covers it.
- Forged headers from another container, or through a published port 3000: the whitelist must refuse them.
- `X-WEBAUTH-USER: admin`, to confirm the collision.
- `GF_USERS_AUTO_ASSIGN_ORG_ROLE=None`, and `GF_SECURITY_CSRF_ALWAYS_CHECK=true` as defence in depth.

### The proxy implementation

The candidates on 2026-11-01, from the crates.io API, with sources from `static.crates.io`. Downloads are all-time, then the last 90 days; open issues, pull requests excluded, are from GitHub. All of them stream bodies.

| Crate | Releases | Licence | Downloads | Open issues | hyper; axum | WebSocket | Hooks |
|---|---|---|---|---|---|---|---|
| [`axum-reverse-proxy`](https://crates.io/crates/axum-reverse-proxy) | 2.4.0, 2026-11-01. 26 since 2024-12, in bursts; 2.0 in 2026-08 | MIT | 267,640; 60,399 | 0 | 1; 0.8 | HTTP/1.1 upgrades, copied raw since 2.3.0. RFC 8441 gets 501 | `ProxyPolicy`, a custom client, tower layers, tunnel guards |
| [`tower-proxy`](https://crates.io/crates/tower-proxy) | 0.10.1, 2026-09-22. 6 since 2026-04 | MIT OR Apache-2.0 | 25,322; 15,078 | 5 | 1; 0.8 | none | Path rewriters only |
| [`hyper-reverse-proxy`](https://crates.io/crates/hyper-reverse-proxy) | 0.5.1, 2022-03-12. 8 in 2017–22 | Apache-2.0 | 89,419; 4,590 | 10 | 0.14; none | none | Few |
| [`reverse-proxy-service`](https://crates.io/crates/reverse-proxy-service) | 0.2.1, 2023-09-11. 5 in 2022–23 | MIT OR Apache-2.0 | 13,722; 654 | 1 | 0.14; 0.6 | none | Path rewriters |
| [`pingora-proxy`](https://crates.io/crates/pingora-proxy) | 0.9.0, 2026-09-09. 11 since 2024-02 | Apache-2.0 | 10.1 M; 1.2 M | 199, all of pingora | Its own server | yes | Many, but not tower |
| Hand-written | none | none | none | none | hyper-util 0.1 | `hyper::upgrade::on` with `copy_bidirectional` | Anything |

**`axum-reverse-proxy` 2.4.0.** One maintainer, 39 stars; edition 2024, MSRV 1.88, about 4,300 lines. GitHub has tags only up to v2.2.0, so the links are docs.rs's copy of the published crate.
- **Routing.** `ReverseProxy::new(path, target)` strips `path` and prepends the target's own path, keeping the query ([proxy.rs, L368–417](https://docs.rs/crate/axum-reverse-proxy/2.4.0/source/src/proxy.rs)). So `("/grafana", "http://grafana:3000/grafana")` passes the path through, and `("/svc/n8n", "http://n8n:5678")` strips it. `new_with_client` (L284) takes p4n4's own client; both constructors panic on an invalid target (L292–297).
- **`ProxyPolicy`** (L45–151): `HostBehaviour::Preserve` (the default is Replace); `XForwardedFor::Append` (the default; it needs `into_make_service_with_connect_info`); `with_public_scheme`; `with_upstream_timeout`, which covers only the response head, then answers 504; `with_max_request_body_bytes` (413); a 5 s WebSocket connect timeout; and `with_via`, which answers 508 on loops.
- **Forwarded headers** ([forward.rs, L238–333](https://docs.rs/crate/axum-reverse-proxy/2.4.0/source/src/forward.rs)). It appends to `X-Forwarded-For` and `Forwarded`, adding `host=` and `proto=` to `Forwarded` only when it starts the header, and sets `X-Forwarded-Host` and `X-Forwarded-Proto` only when they're absent, so p4n4 must drop client values first. CONNECT gets 501 (L384–391).
- **WebSocket.** "HTTP/1.1 upgrades are supported; RFC 8441 extended CONNECT is not", and streams are "copied byte-for-byte, preserving negotiated subprotocols and extensions" ([lib.rs, L252–258](https://docs.rs/crate/axum-reverse-proxy/2.4.0/source/src/lib.rs)). 2.2.0 re-framed messages through tungstenite instead ([CHANGELOG](https://docs.rs/crate/axum-reverse-proxy/2.4.0/source/CHANGELOG.md)), so require 2.3.0 or later.
- **Lifecycle, new in 2.4.0** ([lifecycle.rs, L12–95](https://docs.rs/crate/axum-reverse-proxy/2.4.0/source/src/lifecycle.rs)). A `TunnelGuard` in the request's extensions holds any value, such as a semaphore permit, until the response body ends or the upgraded tunnel closes, which is what the per-user stream cap needs. A `TunnelCancellation` token drops upgraded tunnels, with no close frame, for revocation or shutdown; it leaves ordinary responses such as MJPEG alone.
- **Missing.** Idle timeouts on upgraded connections and bodies; `Location`, cookie and HTML rewriting, none of which p4n4 needs today.
- **Dependencies** ([Cargo.toml](https://docs.rs/crate/axum-reverse-proxy/2.4.0/source/Cargo.toml.orig)). `default = ["tls"]` pulls in hyper-rustls; there's also a `native-tls` feature, which p4n4's cargo-deny bans. tokio-tungstenite ^0.28 is always on, used only for `derive_accept_key` (`websocket.rs:32`), while axum 0.8.9's `ws` uses ^0.29, so cargo-deny will flag a duplicate.
- **Pros:** built for axum 0.8 and hyper 1; active; WebSockets and streaming work; the policy covers rows 7, 8, 25 and 28, and the guard the stream cap. **Cons:** one maintainer; 2.0 dates from 2026-08-29, and the byte-copying tunnels and guards p4n4 relies on from 2026-10-31 and 2026-11-01, so they're barely tried; the duplicate tungstenite; no idle timeouts.

**`tower-proxy` 0.10.1** is the newest find. It continues `axum-proxy`, whose repository is archived and whose 0.8.0 README says it's "no longer maintained", which in turn forked `reverse-proxy-service`. It streams and rewrites paths, but handles no upgrades, hop-by-hop headers or `X-Forwarded-*`; p4n4 would write all of that.

**Rejected:** `hyper-reverse-proxy` and `reverse-proxy-service` are stale, still on hyper 0.14, with no upgrades. `pingora-proxy` is a whole server framework, not a tower service, and would need its own listener. `salvo-proxy` 1.0.1, `warp-reverse-proxy` 2.0.0 and `rama` 0.4.0 belong to other frameworks.

**Hand-written.** hyper-util's legacy `Client` sends the requests. For upgrades, `hyper::upgrade::on` runs on the request and on the upstream's 101, and `tokio::io::copy_bidirectional` joins the two. It's a few hundred lines, with full control of timeouts and headers, and it could translate RFC 8441 CONNECT into an HTTP/1.1 upgrade. But p4n4 would own the hop-by-hop headers, the headers `Connection` lists, `TE`, trailers, and their tests.

**Needed whatever the choice.** None of the candidates does p4n4's own checks. Tower middleware on each mount adds the embed cookie (with a minimum role, if one is chosen), Grafana's headers, `X-Frame-Options: SAMEORIGIN` and `frame-ancestors`, the per-user stream cap, and the §6.4 Proxies additions listed under [Follow-ups](#follow-ups).

**HTTP/2.** axum 0.8.9's `axum::serve` calls `enable_connect_protocol()` on HTTP/2 connections ([serve/mod.rs, L394](https://github.com/tokio-rs/axum/blob/axum-v0.8.9/axum/src/serve/mod.rs#L394)). Browsers that see it may open WebSockets as CONNECT requests, which axum-reverse-proxy rejects (medium confidence on browsers' current defaults). In the TLS modes, p4n4's accept loop should use a hyper-util `auto::Builder` without it, so browsers use HTTP/1.1 for WebSockets.

**Left for the hands-on run:**
- `spikes/s2-proxy/`, using axum-reverse-proxy 2.4.0 with `default-features = false` and p4n4's middleware, against each upstream; the stream cap with `TunnelGuard`.
- The same mounts hand-written, if gaps show up: idle timeouts, headers, the tungstenite duplicate.
- `cargo tree -d` and `cargo deny check`.
- WebSockets over TLS with HTTP/2, with extended CONNECT on and off.

## Decision

Not decided yet.

**Leaning: the implementation.** `axum-reverse-proxy` 2.4.0 or later, with `default-features = false`, built with `new_with_client` on p4n4's hyper-util client (5 s connect timeout). Its policy: `HostBehaviour::Preserve`, `with_public_scheme`, and per mount an upstream timeout and a body limit (1 MiB by default, at least 5 MiB for Node-RED, 16 MiB for n8n). A `TunnelGuard` holds each stream's permit, and a `TunnelCancellation` per session closes its WebSockets when the session is revoked, by a role change too. p4n4's tower middleware does the rest, and extended CONNECT stays off. If the hands-on run finds gaps, idle timeouts above all, fall back to a hand-written proxy on hyper-util.

**Leaning: Grafana.** The auth proxy, with `X-WEBAUTH-USER: p4n4-<user id>`, `X-WEBAUTH-NAME: <username>` and `X-WEBAUTH-ROLE` from the mapping (`viewer` → `Viewer`, `operator` → `Editor`, `admin` → `Admin`, no access for `device`); `GF_USERS_AUTO_ASSIGN_ORG_ROLE=None`; the server's IP as the whitelist; and the rest of the settings in Q3.

## Follow-ups

**Proposed plan changes.** This note doesn't edit the plan.
- **§6.4 Routing.**
  - `/`: replace "Hashed assets are served as immutable" with "Every file is served with `no-cache` and validators, because Flutter web's file names aren't hashed".
  - `/svc/<id>/`: say which mounts strip their prefix (n8n, MQTTX Web, cameras) and which pass it through (Node-RED, like `/grafana/`). Unknown ids get 404, not the SPA.
  - `/mqtt`: `Sec-WebSocket-Protocol` passes both ways.
  - An n8n webhook route, if one is chosen (open questions).
- **§6.4 Embed cookie, and Appendix B's `POST /auth/embed` row:** add `Path=/mqtt`.
- **§6.4 Grafana paragraph:** "Grafana's auth proxy takes the role from a header, as it has since 8.1 (S2). On every request and WebSocket handshake, the server sets `X-WEBAUTH-USER: p4n4-<user id>`, `X-WEBAUTH-NAME` and `X-WEBAUTH-ROLE`, mapping `viewer`, `operator` and `admin` to `Viewer`, `Editor` and `Admin`. Grafana trusts only `[network] server_ip`, and new users default to `None`." Add a note on the `admin` collision.
- **§6.4 Proxies.** Replace "`Authorization` and `Cookie` are never forwarded upstream" with the per-mount credential policy (finding 1). Add: the browser's Host is preserved; client-sent `Forwarded`, `X-Forwarded-*` and `X-Real-IP` are dropped unless the peer is a trusted proxy; an Origin or `Sec-Fetch-Site` check on unsafe methods and upgrades; compression for an allowlist of types only; a 300 s idle timeout, an upstream timeout and a body limit per mount; `nosniff` and `Referrer-Policy` when absent; camera mounts limited to the player, `/api/ws`, `/api/stream.mjpeg` and `/api/frame.jpeg`. Name the crate once S2 decides.
- **§6.4 Hardening:** the per-user stream cap leaves room for a camera grid; streams are exempt from request timeouts; revoking a session (§6.4 "Revocation") also closes its proxied WebSockets, which otherwise outlive it; `/svc/node-red/auth/token` is rate-limited.
- **§6.4 TLS:** extended CONNECT stays off; add an HSTS policy (Caddy sends one year today), perhaps only with `acme`.
- **§5.4 `ui`:** add `strip_prefix`, `body_limit`, `credentials` (`none`, `bearer` or `cookie`) and the upstream port; set `websocket = true` on Grafana, for Live.
- **§5.2:** split `frame_origins` into `frame_ancestors` and `frame_src`, which takes today's `DASHBOARD_FRAME_ORIGINS`; add `[server] public_url`, for n8n's URLs and Grafana's `root_url`; give cameras a `host`, and add `mode=mse` to go2rtc paths.
- **§12.2:** MQTTX Web's saved connections are lost with the change of origin; p4n4 sets `N8N_SECURE_COOKIE=false` when `tls = "off"`; `N8N_WEBHOOK_URL` replaces `WEBHOOK_URL`.
- **Appendix C:** name the proxy crate. `tokio-tungstenite` isn't needed for proxying, since upgrades are copied as bytes and axum's `ws` covers p4n4's own WebSockets: drop it, or align its version. Add compression to tower-http's uses.

**Open questions:**
- **n8n webhooks.** A public `/hooks/n8n/` route without the embed cookie, relying on n8n's own webhook authentication, or 5678 published again?
- **Same-origin trust.** Proxied UIs share the dashboard's origin, so a script injected into Node-RED, n8n or Grafana could call `POST /api/v2/auth/refresh` (the refresh cookie and `X-P4N4-Refresh` both work same-origin) and take over the p4n4 session. Basic auth has the same exposure today. Accept it, or serve proxied UIs from a second origin?
- **Who may open which UI?** A minimum role per mount, such as `operator` or `admin` for Node-RED and n8n?
- **The native app** doesn't share the browser's cookies. How do `/demo/` and the camera mounts get the embed token?
- **Standalone tabs** lose access 10 minutes after the dashboard closes. Should failed navigations redirect to the login, with a return path?
- **Node-RED's `httpNodeRoot`** (FlowFuse Dashboard, HTTP-in endpoints): a second mount, or out of scope?
- **MQTTX Web:** a p4n4 build, or files served by the server?

**P1 work:**
- the proxy module and its middleware, with integration tests per upstream that record handshakes and headers;
- the MQTTX build, if that option is chosen;
- the Grafana environment in `stacks/iot`, and the decision on `operator`;
- `allow_paths` and a login in the template's `video.sh`.
