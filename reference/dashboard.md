# Dashboard

`p4n4-dashboard` (`dashboard/`) is a Flutter app for operating a p4n4 deployment.
It runs on Linux, macOS, Windows, Android and iOS, and can be white-labelled per client.

## Tabs

| Tab | What it does | Talks to |
|-----|--------------|----------|
| **Services** | Launcher for every p4n4 service, with live status | [p4n4-api](api.md) `GET /api/v1/stacks`, or a direct probe of each port when the API is unreachable |
| **Edge** | CPU, memory, SoC temperature and inference latency, with 2 minutes of history | A metrics URL, by default `GET /api/v1/edge/metrics` ([contract](#edge-metrics-contract)); has a demo mode |
| **Agent** | Chat with a local model or a stateful agent | Ollama `/api/chat` (streaming) or Letta `/v1/agents/{id}/messages` |
| **Grafana** | Embedded Grafana, in kiosk mode by default and in the app's light or dark theme | `http://<host>:3000` |
| **Video** | Live camera feeds from the edge device, one at a time or in a grid | Any MJPEG stream or JPEG snapshot URL |

## Admin, power and normie views

The view comes from the signed-in account and is kept until you sign out.

| | Admin | Power | Normie |
|---|---|---|---|
| Tabs | Every brand tab, plus **Clients** | The tabs an admin enables (default: all) | **Home**, plus the tabs an admin enables (default: Agent, Grafana, Video) |
| Home | — | — | One large health card, edge readings and large shortcuts, without service names, hosts, ports or URLs |
| Clients | Client deployments as connection profiles (name, host, optional API URL) with live status; **Connect** switches the dashboard to one | — | — |
| Details | Hosts, URLs and raw errors; camera, agent and Grafana config | Same as admin | Plain-language messages only |
| Settings | Everything, plus **Views** (each view's tabs, and a preview), **Users** and **Diagnostics** | Appearance, connection (switch between saved deployments), endpoints, account and about | Appearance, account and about |

> **Sign-in uses p4n4-api accounts** (`p4n4-api users add`, or Settings → Users). `admin`
> accounts get the admin view, `operator` accounts the power view and `normie` accounts the
> normie view. Sign-in is per deployment: the refresh token is
> kept in secure storage and the access token is refreshed automatically. If the API runs
> with `P4N4_API_AUTH=off` or can't be reached, the screen offers a role picker instead,
> which is dropped as soon as the API requires sign-in.

Connection settings (host, API, metrics, agent, Grafana and cameras) belong to the
connected deployment and switch together on **Connect**. The Letta password is kept in the
platform's secure storage (Keychain, Android Keystore, Windows Credential Manager or the
Linux Secret Service). Use host `10.0.2.2` to reach your machine from the Android emulator.

## Project settings

When p4n4-api answers `GET /api/v1/project`, the dashboard tailors itself to the connected
project's `.p4n4.json`. It reloads when you **Connect** to another deployment, and retries
every 30 s while the API is unreachable:

| Manifest | Effect |
|---|---|
| `layers` | Services and Home show only those stacks (plus the API) |
| `dashboard.tabs` | Brand tabs outside the list are hidden, in both views |
| `dashboard.grafana_path` | The Grafana tab opens this page, unless the deployment has its own path. Project defaults come before brand defaults |
| `dashboard.cameras` | The Video tab's cameras, until the deployment saves its own; they come before a brand `videoUrl`. Each is `{id, name}` plus an absolute `url`, or a `port` and `path` on the connected host |

Without the API (or before signing in), nothing changes: every
brand tab and stack is shown. See the [greenhouse use case](https://github.com/raisga/p4n4-templates/blob/main/docs/use-cases/greenhouse-telemetry.md).

## Run as a service

The dashboard ships as a web container (`p4n4-dashboard`, port **8088**): nginx serves the
Flutter web build and proxies p4n4-api (`/api/`), Ollama (`/ollama/`) and Letta (`/letta/`),
so the browser talks to one origin and the services need no CORS.

```bash
cd dashboard
cp .env.example .env
docker compose up -d          # or `make up-local` to build from source
```

p4n4-api v0.1 runs on the host. Start it on the Docker bridge address
(`P4N4_API_HOST=172.17.0.1`) or `0.0.0.0` so the container can reach it. The dashboard signs in
with p4n4-api accounts through the proxy. Grafana needs
`GF_SECURITY_ALLOW_EMBEDDING=true`. `make image THEME=<project>` builds an image with a
project's white-label theme, and only that theme. Keep the API's auth on (otherwise the
sign-in screen offers a role picker), and bind the container to a LAN address (`DASHBOARD_BIND`).

## Develop

```bash
cd dashboard
flutter pub get
flutter run -d linux        # or macos, windows, android, ios
flutter run -d chrome       # web
flutter test
```

Release builds: `flutter build apk --split-per-abi | appbundle | ios | macos | windows | linux`,
and `flutter build web --release --no-web-resources-cdn` for the browser (CanvasKit bundled,
so it works offline; serve `build/web` from any static server).

On web, the browser enforces CORS. Allow the dashboard's origin on p4n4-api
(`P4N4_API_CORS_ORIGINS`) and Ollama (`OLLAMA_ORIGINS`), and set
`GF_SECURITY_ALLOW_EMBEDDING=true` on Grafana. The planned container (`p4n4-dashboard`) will
put them behind one origin instead. With no configuration, the web build uses the host that
served the page. An optional `config.json` next to the app sets `host`, `apiBase`, `ollamaBase`
and `lettaBase`.
Linux builds need `libsecret-1-dev`.

## White-label

The dashboard commits only its default `p4n4` brand. A client's brand is a **theme** in
the client's p4n4 project (`theme/` with `brand.json`, icon and fonts, named by
`.p4n4.json` `dashboard.theme`). Install it and apply it before building:

```bash
dart run tool/brand.dart install ~/projects/greenhouse --apply   # theme → brands/<id>/ (gitignored), patch native projects, regenerate icons
dart run tool/brand.dart apply p4n4                              # back to the default
dart run tool/brand.dart remove <id>                             # delete an installed theme
```

A brand sets the name, wordmark or logo, colors per theme, fonts, visible tabs, links,
first-run defaults, app IDs and launcher icons. The tool checks every text color for WCAG AA
contrast. Only the applied brand is bundled, so one client's build never contains another's
branding. Templates can ship a sample theme (see the [template registry](template-registry.md#project-manifest)).

## Edge metrics contract

The Edge tab polls its metrics URL every 2 s and expects:

```json
{
  "cpu_percent": 23.1,
  "mem_percent": 61.0,
  "mem_used_mb": 2480,
  "mem_total_mb": 4096,
  "disk_percent": 44.2,
  "temp_c": 51.3,
  "uptime_s": 86400,
  "load": [0.4, 0.5, 0.6],
  "inference_ms": 12.5
}
```

Only `cpu_percent` and `mem_percent` are required. Optional fields are left out rather than
sent as `null`, and their tiles only appear when present. p4n4-api serves everything except
`inference_ms`.

## Platform notes

- **Grafana:** `webview_flutter` supports Android, iOS and macOS only. On web, Grafana is an
  `<iframe>`. On Windows and Linux the tab opens Grafana in the browser.
- **Web:** cameras play in an `<img>` (no CORS needed), port probes use `no-cors` requests,
  and Ollama replies arrive all at once rather than streamed.
- **Plain HTTP:** the p4n4 services use HTTP on the LAN, so cleartext is allowed on every
  platform (`usesCleartextTraffic` on Android, `NSAllowsArbitraryLoads` on iOS, the
  `network.client` entitlement on macOS).
- **Video:** streams are decoded in pure Dart. Known-good sources are mjpg-streamer
  (`/?action=stream`), motion, go2rtc (`/api/stream.mjpeg?src=…`) and any `snapshot.jpg`.
  Dropped streams reconnect with a 2–30 s backoff.
- **Tested:** all five platforms build in CI, but only Linux has been run by hand.
