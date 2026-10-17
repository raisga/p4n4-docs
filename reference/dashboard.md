# Dashboard

`p4n4-dashboard` (`dashboard/`) is a Flutter app for operating a p4n4 deployment.
It runs on Linux, macOS, Windows, Android and iOS, and can be white-labelled per client.

## Tabs

| Tab | What it does | Talks to |
|-----|--------------|----------|
| **Services** | Launcher for every p4n4 service, with live status | [p4n4-api](api.md) `GET /api/v1/stacks`, or a direct probe of each port when the API is unreachable |
| **Edge** | CPU, memory, SoC temperature and inference latency, with 2 minutes of history (10 or 30 in Settings → Appearance) | A metrics URL, by default `GET /api/v1/edge/metrics` ([contract](#edge-metrics-contract)); has a demo mode |
| **Assistant** | Chat with a local model or a stateful agent; admins and power users choose the model or agent for everyone | p4n4-api `POST /api/v1/agents/chat` (Ollama, streaming) or `/api/v1/agents/{id}/chat` (Letta), with the choice in `GET/PUT /api/v1/agents/config` |
| **Grafana** | Embedded Grafana, in kiosk mode by default and in the app's light or dark theme | `http://<host>:3000` |
| **Video** | Live camera feeds from the edge device, in a grid or one at a time, with the project's vision overlay when it has one. Admins can switch to demo cameras (openly licensed sample clips) | Any MJPEG stream, JPEG snapshot URL, or video file (`.mp4`, `.webm`, `.mov`, `.m3u8`); the overlay from `GET /api/v1/project/vision` |
| **Feed** | Opt-in (`feed` in the brand's or project's tabs): the project's live events and alerts; operators and admins act on alerts | p4n4-api `GET /api/v1/feed` and `POST /api/v1/feed/{id}/action`, which relay the feed `.p4n4.json` names |

## Admin, power and normie views

The view comes from the signed-in account and is kept until you sign out.

| | Admin | Power | Normie |
|---|---|---|---|
| Tabs | **Home**, every brand tab, then **Clients** | **Home**, plus the tabs an admin enables (default: all) | **Home**, plus the tabs an admin enables (default: Agent, Grafana, Video) |
| Home | One large health card, edge readings and large shortcuts, without service names, hosts, ports or URLs | Same as admin | Same as admin |
| Clients | Client deployments as connection profiles (name, host, optional API URL) with live status; **Connect** switches the dashboard to one | — | — |
| Services | Launcher, plus stack controls (start/restart/stop) | Launcher; no stack controls | Only if enabled; no stack controls |
| Details | Hosts, URLs and raw errors; camera, agent and Grafana config; edge and camera demo toggles | Same as admin, without the demo toggles | Plain-language messages only |
| Settings | Everything, plus **Views** (the tab order, each view's tabs and other settings, and a preview), **Users**, **Banner**, **Audit log**, **This device** and **Diagnostics** | Account, Appearance, Accessibility, Language & region, Connection, Endpoints and About | Account, Appearance, Accessibility, Language & region and About |

Brand tabs come in one order for every view, after Home: **Agent → Grafana → Video → Edge →
Services** by default. Admins drag them into another order in Settings → Views. The order and
each view's tabs are saved on the connected deployment's p4n4-api
(`GET/PUT /api/v1/dashboard/views`), so they apply on every device. Admins can **preview** the
power or normie view from Settings → Views; the preview stays on that device and ends on
sign-out.

Settings → Views also sets, for the power and normie views, what else each one shows. These
too are kept on p4n4-api (`GET/PUT /api/v1/dashboard/view-settings`) and apply on every
device; devices check for changes to them, and to the tabs, every minute:

| Setting | Default | Effect |
|---|---|---|
| Opens on | Home | The tab the view opens on, when it has that tab |
| Home shows | Everything | Home's cards: the status summary, the device's readings and the project's page (`HOME.md`) |
| Grafana page | The deployment's | Its own Grafana page, e.g. a simpler dashboard for normies. Normies always get Grafana in kiosk mode |
| Cameras | All | Which of the deployment's cameras it shows (demo cameras follow the camera they stand in for) |
| Vision overlay | On | Whether the vision overlay is drawn over its cameras |
| Alerts that pop up | All | All alerts, only those to review, or none |
| Chime for alerts to review | On | Turns the chime off for the view; each device's own sound setting still applies |

Admins always get the defaults. These settings are presentation, like the tabs: p4n4-api
still decides what each role may read.

The first time a normie signs in on a device, a short tour
([`flutter_intro`](https://pub.dev/packages/flutter_intro)) points out the app name, the
navigation, the theme toggle, Settings and Sign out. **Skip tour** or finishing it marks it
seen on that device, and Settings → Account → **Take the tour** shows it again. A brand can
rewrite any step's text (`tour` in `brand.json`).

The views decide what the dashboard shows; p4n4-api enforces what each role can do, so a
normie can only read status and chat with the assistant, whatever the dashboard shows.

Admins also have:

- **Users:** each account's view and last sign-in. Reset a password (the user then chooses a
  new one at their next sign-in, unless the box is cleared), require a new password from the
  first sign-in, **sign a user out everywhere** (a lost tablet, someone who left) while keeping
  their password, or remove the account.
- **Banner:** a message above every view on every device of the deployment (planned
  maintenance, a known outage), info or warning, until cleared or for 1, 4 or 24 hours. It's
  kept by p4n4-api (`/api/v1/dashboard/banner`), and devices check for it every minute.
- **Audit log:** who changed what (users, views, the banner, stack actions…) and when,
  newest first.

## Kiosk mode and idle devices

For a shared screen such as a wall tablet, an admin turns on **kiosk mode** on that device
(Settings → This device) and signs it in with a viewer (normie) account. The device keeps it:

- There's no Sign out or theme toggle, no tour, and Settings holds only Accessibility, so a
  visitor can enlarge the text. A lock button leaves kiosk mode with an **admin's** sign-in, and the
  device then stays signed in as that admin.
- After **Start over when idle** (5 minutes by default; 1 to 30, or never) without a touch,
  click or key, the kiosk goes back to the view's start tab, closes whatever is open, clears the
  Assistant conversation and puts text size, contrast and motion back to how they were when kiosk
  mode was turned on. It happens once per quiet spell. Touches inside the embedded Grafana page
  don't count as activity (they never reach the dashboard), so a long look at a chart there can
  be cut short: pick a start-over time longer than that.
- Turning kiosk mode on also turns on **Sign admins and power users out when idle** (15 minutes)
  if it was off. It applies to any admin or power user signed in on this device, kiosk or not.

Kiosk mode and the idle settings belong to the device, not the deployment. Kiosk mode needs
p4n4-api accounts: without them (`P4N4_API_AUTH=off`) nobody could leave it. If the kiosk's sign-in
ends (its refresh token unused for 7 days, or an admin signing it out everywhere), the sign-in
screen comes back; signing in with the viewer account resumes kiosk mode, and an admin or power
user who signs in gets their own view.

An account that must choose a new password gets only the password form after signing in, and
p4n4-api turns down everything else until it's changed. Anyone signed in with a p4n4-api
account can change their own password in Settings → Account; their other sign-ins end.

> **Sign-in uses p4n4-api accounts** (`p4n4-api users add`, or Settings → Users). `admin`
> accounts get the admin view, `operator` accounts the power view and `normie` accounts the
> normie view. Sign-in is per deployment: the refresh token is
> kept in secure storage and the access token is refreshed automatically. If the API runs
> with `P4N4_API_AUTH=off` or can't be reached, the screen offers a role picker instead,
> which is dropped as soon as the API requires sign-in.

Connection settings (host, API, metrics, Grafana and cameras) belong to the connected
deployment and switch together on **Connect**. Theme, language, accessibility (text size,
high contrast, reduce motion) and region settings (temperature unit, 12/24-hour time) are
app-wide.

Admins and power users also set their own defaults on a device in Settings → Appearance: the
tab it opens on (ahead of the view's start tab), how many minutes the Device tab's charts span
(2, 10 or 30; app-wide) and the camera the Cameras tab shows first. The start tab and camera are
per deployment. An admin previewing another view gets that view's start tab, not their own.

The p4n4-api refresh token is kept in the platform's secure storage (Keychain,
Android Keystore, Windows Credential Manager or the Linux Secret Service); the Letta
password, which older versions kept there, now stays on the server and is removed from
devices on upgrade. Use host `10.0.2.2` to reach your machine from the Android emulator.

## Languages

The dashboard is in English (the default) and Spanish. It follows the device's language,
falling back to English, and **Settings → Language & region** overrides that. A brand can
pick the first-run language with `"locale": "es"` in its `defaults`. Labels live in
`lib/l10n/app_<code>.arb` (`app_en.arb` is the template); to add a language, add an ARB file
with every key of `app_en.arb`. Brand text, product and service names, hosts, URLs and raw
server errors aren't translated.

## Project settings

When p4n4-api answers `GET /api/v1/project`, the dashboard tailors itself to the connected
project's `.p4n4.json`. It reloads when you **Connect** to another deployment, and retries
every 30 s while the API is unreachable:

| Manifest | Effect |
|---|---|
| `layers` | Services and Home show only those stacks (plus the API) |
| `dashboard.tabs` | Brand tabs outside the list are hidden, in both views |
| `dashboard.grafana_path` | The Grafana tab opens this page, unless the deployment has its own path. Project defaults come before brand defaults |
| `dashboard.home` | A Markdown file, relative to the project, shown at the top of Home (default `HOME.md`), or its translation for the app's language (`HOME.es.md`). Links like `[Activity](tab:feed)` open that tab, and `[Is any unit on warning?](ask:)` asks the Assistant. A missing or empty file shows nothing; `p4n4 validate` reports a named file that's missing |
| `dashboard.cameras` | The Video tab's cameras, until the deployment saves its own; they come before a brand `videoUrl`. Each is `{id, name}` plus an absolute `url`, a `port` and `path` on the connected host, or a `device` camera (`back` or `front`) of the phone or tablet |
| `dashboard.feed` | Where the Feed tab's items come from: `{url, token?}`, with `token` naming the layer `.env` variable (`{layer, env}`) p4n4-api sends upstream, so it never reaches a browser |
| `dashboard.vision` | The Video tab's overlay: `{file, live?}`, a JSON file in the project (labels, alert rules, each camera's recorded boxes and behaviours) and optionally the WebSocket of live detections. Alerts it raises pop up as toasts on every tab |
| `dashboard.chats` | The bot service whose chats (e.g. Telegram) get the project's alerts: `{url, token?}`, as for `feed`. Admins link and unlink chats in Settings → Chats |
| `dashboard.theme` | Not read by the app: the white-label theme directory the project's dashboard is built with (`dart run tool/brand.dart install <project>`, [White-label](#white-label)) |

Without the API (or before signing in), nothing changes: every
brand tab and stack is shown. See the [greenhouse use case](https://github.com/raisga/p4n4-templates/blob/main/docs/use-cases/greenhouse-telemetry.md).

## Run as a service

The dashboard ships as a web container (`p4n4-dashboard`, port **8088**): nginx serves the
Flutter web build and proxies p4n4-api (`/api/`, the assistant included), so the browser
talks to one origin and the API needs no CORS. See the [Dashboard service](../stacks/dashboard.md).

```bash
cd dashboard
cp .env.example .env
docker compose up -d          # or `make up-local` to build from source
```

For a p4n4-api on the host, start it on the Docker bridge address
(`P4N4_API_HOST=172.17.0.1`) or `0.0.0.0` so the container can reach it; for the API's own
container, set `P4N4_API_UPSTREAM=http://p4n4-api:8000`. The dashboard signs in
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

`make run` (or `scripts/dev dashboard` from the umbrella repo, see the
[Development guide](../guides/development.md)) runs the web build with hot reload on :8088
and `P4N4_DEV_PROXY=true`, so the app uses `/api/`, which `web_dev_config.yaml` forwards to
`localhost:8000`. With plain `flutter run -d chrome` the app calls the API directly, which then
needs CORS for the page's origin (`P4N4_API_CORS_ORIGINS`). Grafana needs
`GF_SECURITY_ALLOW_EMBEDDING=true`. With no configuration, the web build uses the host that
served the page. An optional `config.json` next to the app sets `host`, `apiBase` and
`grafanaBase`.
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
  and assistant replies arrive all at once rather than streamed.
- **Plain HTTP:** the p4n4 services use HTTP on the LAN, so cleartext is allowed on every
  platform (`usesCleartextTraffic` on Android, `NSAllowsArbitraryLoads` on iOS, the
  `network.client` entitlement on macOS).
- **Video:** streams are decoded in pure Dart. Known-good sources are mjpg-streamer
  (`/?action=stream`), motion, go2rtc (`/api/stream.mjpeg?src=…`) and any `snapshot.jpg`.
  Dropped streams reconnect with a 2–30 s backoff. Video files play muted and looping
  (`video_player` on Android, iOS and macOS, `<video>` on web); Linux and Windows offer
  *Open in browser* for them.
- **Tested:** all five platforms build in CI, but only Linux has been run by hand.
