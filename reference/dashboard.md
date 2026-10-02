# Dashboard

`p4n4-dashboard` (`clients/dashboard/`) is a Flutter app for operating a p4n4 deployment.
It runs on Linux, macOS, Windows, Android and iOS, and can be white-labelled per client.

## Tabs

| Tab | What it does | Talks to |
|-----|--------------|----------|
| **Services** | Launcher for every p4n4 service, with live status | [p4n4-api](api.md) `GET /api/v1/stacks`, or a direct probe of each port when the API is unreachable |
| **Edge** | CPU, memory, SoC temperature and inference latency, with 2 minutes of history | A metrics URL, by default `GET /api/v1/edge/metrics` ([contract](#edge-metrics-contract)); has a demo mode |
| **Agent** | Chat with a local model or a stateful agent | Ollama `/api/chat` (streaming) or Letta `/v1/agents/{id}/messages` |
| **Grafana** | Embedded Grafana, in kiosk mode by default | `http://<host>:3000` |
| **Video** | Live camera feeds from the edge device, one at a time or in a grid | Any MJPEG stream or JPEG snapshot URL |

## Admin and client views

The sign-in screen asks for a role, which is kept until you sign out.

| | Admin | Client |
|---|---|---|
| Tabs | Every brand tab, plus **Clients** | **Home**, plus the tabs an admin enables (default: Agent, Grafana, Video) |
| Home | — | Overall health, per-stack status and edge readings, without hosts, ports or URLs |
| Clients | Client deployments as connection profiles (name, host, optional API URL) with live status; **Connect** switches the dashboard to one | — |
| Settings | Everything, including which tabs clients see | Appearance, account and about |

> **Sign-in is a placeholder.** The role picker doesn't authenticate yet. p4n4-api now
> issues JWTs and requires them for `/api/v1/stacks` and `/api/v1/edge/metrics`, so until
> the dashboard signs in, those calls get `401`: Services falls back to port probes, and
> the Edge tab needs demo mode or another metrics URL.

Connection settings (host, API, metrics, agent, Grafana and cameras) belong to the
connected deployment and switch together on **Connect**. The Letta password is kept in the
platform's secure storage (Keychain, Android Keystore, Windows Credential Manager or the
Linux Secret Service). Use host `10.0.2.2` to reach your machine from the Android emulator.

## Run

```bash
cd clients/dashboard
flutter pub get
flutter run -d linux        # or macos, windows, android, ios
flutter test
```

Release builds: `flutter build apk --split-per-abi | appbundle | ios | macos | windows | linux`.
Linux builds need `libsecret-1-dev`.

## White-label

Brands live in `brands/<id>/` (`brand.json`, logo, fonts, icons). Apply one before
building:

```bash
dart run tool/brand.dart apply acme   # copy the brand in, patch native projects, regenerate icons
dart run tool/brand.dart apply p4n4   # back to the default
```

A brand sets the name, wordmark or logo, colors per theme, fonts, visible tabs, links,
first-run defaults, app IDs and launcher icons. The tool checks every text color for WCAG AA
contrast. Only the applied brand is bundled, so one client's build never contains another's
branding.

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

- **Grafana:** `webview_flutter` supports Android, iOS and macOS only. On Windows and Linux
  the tab opens Grafana in the browser.
- **Plain HTTP:** the p4n4 services use HTTP on the LAN, so cleartext is allowed on every
  platform (`usesCleartextTraffic` on Android, `NSAllowsArbitraryLoads` on iOS, the
  `network.client` entitlement on macOS).
- **Video:** streams are decoded in pure Dart. Known-good sources are mjpg-streamer
  (`/?action=stream`), motion, go2rtc (`/api/stream.mjpeg?src=…`) and any `snapshot.jpg`.
  Dropped streams reconnect with a 2–30 s backoff.
- **Tested:** all five platforms build in CI, but only Linux has been run by hand.
