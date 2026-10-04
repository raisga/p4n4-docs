# Deploying a project template

This guide takes a project created from a [template](../reference/template-registry.md) through three stages:

1. **Rehearse it on your workstation** under [`p4n4-emu`](../reference/emulator.md), sized like the target hardware.
2. **Deploy it to Hetzner Cloud.**
3. **Deploy it to a host you bring yourself (BYOH)**, using Google Cloud (GCP) as the worked example.

The examples use [`mqtt-influx-grafana`](https://github.com/raisga/p4n4-templates/tree/main/mqtt-influx-grafana), a single-layer `iot` template. The [multi-layer templates](#multi-layer-templates) section covers the differences for `mqtt-influx-grafana-ollama` and `retail-vision`.

```
 workstation                              cloud host (Hetzner, GCP, any Linux box)
 ──────────────────────────────           ────────────────────────────────────────────
 template copy ─► p4n4-emu up             same project dir ─► docker compose up -d
                  (rpi5 / nuc limits,      + docker-compose.override.yml
                   QEMU arm64, --sim)        (localhost-only ports, Caddy TLS proxy)
                                           + firewall outside the VM
```

The emulator is a development tool. Nothing it generates gets deployed. A template is a complete, runnable Compose project, so the cloud host runs the template's own `docker-compose.yml` and a small override file.

---

## 1. Prerequisites

| On | Needs |
|---|---|
| Workstation | Docker Engine ≥ 24, Compose ≥ 2.24.4 (the `!override` / `!reset` tags used below), Python ≥ 3.11, [`uv`](https://docs.astral.sh/uv/), cgroup v2, a checkout of this monorepo (for `tools/emu` and `tools/templates`) |
| Cloud host | 64-bit Linux (Ubuntu 24.04 LTS is assumed below), x86_64 or arm64, Docker Engine ≥ 24, Compose ≥ 2.24.4, SSH access, a DNS name if you want HTTPS |
| Optional | The `p4n4` CLI (`uv tool install p4n4`) for `p4n4 validate`, `p4n4 up` and `p4n4 secret rotate`. Plain `docker compose` is enough for a single-layer template |

`p4n4 template install` isn't implemented yet, so you create a project by copying the template directory.

---

## 2. Run the template under the emulator

### 2.1 Install p4n4-emu

Install it once as a tool, so `p4n4-emu` is on your `PATH` in any directory:

```bash
uv tool install --editable tools/emu      # from the monorepo root
p4n4-emu setup --check-only               # Docker, Compose and cgroup v2 should show OK
```

Or run it from the checkout without installing it: `uv run --project <monorepo>/tools/emu p4n4-emu …`. A plain `uv run p4n4-emu` works only inside `tools/emu`.

The Raspberry Pi profiles (`rpi4`, `rpi5`) run **arm64** images. On an x86 workstation, register QEMU once:

```bash
p4n4-emu setup --arch arm64
```

If you skip this step, pass `--native` to `up`. You still get the resource limits, with host-architecture images.

### 2.2 Create the project

```bash
cp -r tools/templates/mqtt-influx-grafana ~/projects/greenhouse
cd ~/projects/greenhouse
cp .env.example .env
```

In `.env`, set `ARCHIVE_UID` / `ARCHIVE_GID` to the output of `id -u` / `id -g`. Change the passwords and token too, if you like. On a workstation the defaults are acceptable.

Rename the project in `.p4n4.json` (`"project": "greenhouse"`). The emulator reads this file to find the project's layers (`"layers": ["iot"]`). In a flat single-layer project like this one, the stack's compose file sits at the project root.

### 2.3 Pick a profile that matches the target

| Profile | CPU | Memory | Disk R/W | Arch | Closest cloud size |
|---|---|---|---|---|---|
| `mcu-class` | 1 | 256 MB | 10 MB/s | x86_64 | Too small for the template; use it to find what fails first |
| `rpi4` | 4 | 3.5 GB | 50 MB/s | arm64 | Hetzner CAX11 (2 vCPU / 4 GB arm64) |
| `rpi5` | 4 | 7 GB | 100 MB/s | arm64 | Hetzner CAX21 (4 vCPU / 8 GB arm64), GCP `t2a-standard-2` (2 vCPU / 8 GB arm64) |
| `nuc` | 4 | 14 GB | 200 MB/s | x86_64 | Hetzner CPX41 or CAX31 (8 vCPU / 16 GB), GCP `e2-standard-4` (4 vCPU / 16 GB) |

Plan names and sizes change over time. Check the provider's current list, and pick a plan with at least the profile's memory. The emulator divides the profile's CPU and memory among the project's services (`p4n4-emu profile show rpi5`). If the stack stays healthy under a profile for a day of simulated traffic, a VM with the same resources will run it.

### 2.4 Start it

From the project directory:

```bash
p4n4-emu up --profile rpi5 --dry-run      # print the overlay (written to ~/.p4n4-emu/overlays/<project>-<hash>/iot.emu.yml)
p4n4-emu up --profile rpi5                # docker compose -f docker-compose.yml -f <overlay> up -d
p4n4-emu status                           # usage against each service's limits
```

The emulator applies the template's `docker-compose.yml`, any `docker-compose.override.yml` and its own overlay, in that order. It reads `.env` like `docker compose` does, so `COMPOSE_PROFILES=demo` still starts the template's simulator.

**Test data.** Use one of these two sources:

- The template's own simulator (`COMPOSE_PROFILES=demo` in `.env`, the default) publishes `dev-01` and `dev-02` readings.
- The emulator's simulator (`--sim`, plus `--sim-devices 3` for three devices) publishes `sensors/emu-sensor-N/{temperature,humidity,pressure,raw}` to `p4n4-mqtt`, the template's broker container. To use only this one, set `COMPOSE_PROFILES=` in `.env`.

```bash
p4n4-emu up --profile rpi5 --sim --sim-devices 3
```

Open Grafana at <http://localhost:3000>. The **Telemetry** dashboard is the home page. Watch memory headroom with `docker stats`. A service that hits its limit is OOM-killed and restarted, and `docker inspect <container> --format '{{.State.OOMKilled}}'` shows whether that happened.

### 2.5 Check and stop it

```bash
./tests/smoke.sh                          # the template's end-to-end test (runs its own throwaway copy)
p4n4-emu down                             # stop; keep volumes
p4n4-emu down --volumes                   # stop and delete data
```

**Emulator limitations.** It can't throttle the network (no `tc netem`). QEMU arm64 is 3–10× slower than native, so Ollama under an ARM profile is impractical; use `--native` for the ai layer. Limits are not enforced on cgroup v1 hosts. See the [emulator reference](../reference/emulator.md#known-limitations).

---

## 3. Prepare the project for a public host

On a workstation, every port binding on `0.0.0.0` is harmless. On a VM with a public IP, the template as shipped exposes MQTT without authentication (`allow_anonymous true`), InfluxDB and Grafana over plain HTTP. Make these changes once, in the project. They apply to Hetzner, GCP and any other host alike.

### 3.1 Secrets

Set real values in `.env` **before the first start on the server**:

```bash
p4n4 secret rotate        # or: p4n4 secret generate, and paste the values
```

InfluxDB reads `DOCKER_INFLUXDB_INIT_*` (username, password, org, bucket, token) only on its first start, when its volume is empty. If you change `INFLUXDB_TOKEN` in `.env` after that, Telegraf and Grafana use the new token while InfluxDB still expects the old one. Never commit `.env`. Copy it to the server separately (see [4.4](#44-ship-the-project)).

Also set on the server:

| Variable | Value |
|---|---|
| `ARCHIVE_UID` / `ARCHIVE_GID` | `id -u` / `id -g` of the deploy user **on the server** |
| `COMPOSE_PROFILES` | Empty, once real devices publish. With `demo`, the simulator writes fake readings next to real ones |
| `GRAFANA_ANONYMOUS` | `false` on the internet. `true` lets anyone who reaches Grafana read every dashboard |

### 3.2 Bind data services to localhost, put Grafana behind TLS

Create `docker-compose.override.yml` next to `docker-compose.yml`. Compose loads it automatically, and so do `p4n4 up` and `p4n4-emu`:

```yaml
# docker-compose.override.yml — production overrides for a public host
services:
  mqtt:
    ports: !override
      - "127.0.0.1:1883:1883"
      - "127.0.0.1:9001:9001"
  influxdb:
    ports: !override
      - "127.0.0.1:8086:8086"
  grafana:
    ports: !reset []                 # reachable only through Caddy
    environment:
      GF_SERVER_ROOT_URL: https://grafana.example.com/
  caddy:
    image: caddy:2
    container_name: p4n4-caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./config/caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy-data:/data
      - caddy-config:/config
    depends_on:
      grafana:
        condition: service_healthy
    networks:
      - p4n4-net

volumes:
  caddy-data:
  caddy-config:
```

```caddyfile
# config/caddy/Caddyfile
grafana.example.com {
    reverse_proxy p4n4-grafana:3000
}
```

Point an `A` (and `AAAA`) record for `grafana.example.com` at the server before the first start. Caddy obtains and renews the Let's Encrypt certificate itself, and it needs ports 80 and 443 reachable to do so. Run `docker compose config` to see the merged result. The `!override` and `!reset` tags need Compose ≥ 2.24.4.

For admin access to InfluxDB (or Grafana before DNS is ready), tunnel over SSH instead of opening ports:

```bash
ssh -L 8086:localhost:8086 -L 3000:localhost:3000 deploy@<server>   # Grafana via this tunnel needs the !reset line removed
```

### 3.3 Let devices reach MQTT

Pick one of these:

| Option | When | What to do |
|---|---|---|
| **Private network** (recommended) | You control the devices or a gateway at the site | Join the server and the devices to WireGuard or Tailscale. Bind MQTT to the VPN address (`"100.x.y.z:1883:1883"`) instead of `127.0.0.1` |
| **MQTT over TLS** | Devices connect over the internet | Add a TLS listener with a password file (below). Open only 8883 |
| **Bridge from a site broker** | The site already has a broker | Leave MQTT closed. Set `MQTT_REMOTE_*` in `.env` so the server's broker **pulls** `sensors/#` from the site broker ([template README](https://github.com/raisga/p4n4-templates/tree/main/mqtt-influx-grafana#external-mqtt-broker)) |

For MQTT over TLS, edit the listeners in `config/mosquitto/mosquitto.conf`. The template already mounts `config/mosquitto/certs/`. With `per_listener_settings true`, the internal listener stays anonymous for Telegraf and the simulator on `p4n4-net`, and the host binds it to `127.0.0.1` only (see the override above). The public TLS listener requires a login:

```conf
per_listener_settings true

listener 1883                    # internal: Telegraf, simulator; host-bound to 127.0.0.1
protocol mqtt
allow_anonymous true

listener 8883                    # public: devices
protocol mqtt
certfile /mosquitto/certs/fullchain.pem
keyfile  /mosquitto/certs/privkey.pem
allow_anonymous false
password_file /mosquitto/config/passwd
```

Remove the file's global `allow_anonymous true` line and the 9001 WebSocket listener, or give the WebSocket listener its own settings. Keep `per_listener_settings` at the top of the file. In the override, add a `volumes:` entry that mounts `config/mosquitto/passwd` at `/mosquitto/config/passwd`, and add `"8883:8883"` to `mqtt`'s ports. Get the certificate for the broker's DNS name, for example with `certbot certonly --standalone -d mqtt.example.com` before Caddy takes port 80, or by copying it from Caddy's `caddy-data` volume. Restart `mqtt` after each renewal. The [Security guide](security.md#mosquitto-authentication) covers password and ACL files.

### 3.4 Firewall: use the provider's firewall

Docker writes its own iptables rules for published ports, ahead of UFW's. A `ufw deny 3000` on the host doesn't stop Docker from accepting traffic on 3000. Filter at the provider's network firewall (Hetzner Cloud Firewall, GCP VPC firewall rules), which sits outside the VM. Bind ports to `127.0.0.1` as in [3.2](#32-bind-data-services-to-localhost-put-grafana-behind-tls), so a firewall mistake doesn't expose them.

Inbound rules for the setup above:

| Port | Source | For |
|---|---|---|
| 22/tcp | Your IP(s) only, or IAP on GCP | SSH |
| 80/tcp, 443/tcp, 443/udp | Anywhere | Caddy (certificate issuance, Grafana over HTTPS/HTTP3) |
| 8883/tcp | Device IP ranges, or anywhere | Only with the MQTT-over-TLS option |
| 41641/udp (Tailscale) or 51820/udp (WireGuard) | Anywhere | Only with the private-network option |

### 3.5 Shared cloud-init

Both providers accept cloud-init user data. This file creates a `deploy` user, installs Docker and caps container log sizes (Docker keeps logs forever by default):

```yaml
#cloud-config
package_update: true
package_upgrade: true
packages: [ca-certificates, curl, rsync, unattended-upgrades]

users:
  - name: deploy
    shell: /bin/bash
    groups: [sudo]
    sudo: "ALL=(ALL) NOPASSWD:ALL"
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... you@laptop

ssh_pwauth: false
disable_root: true

write_files:
  - path: /etc/docker/daemon.json
    content: |
      { "log-driver": "json-file", "log-opts": { "max-size": "10m", "max-file": "5" } }

runcmd:
  - curl -fsSL https://get.docker.com | sh
  - usermod -aG docker deploy
  - systemctl enable --now docker
```

Save it as `cloud-init.yaml`. Membership in the `docker` group is equivalent to root, so treat the `deploy` key accordingly.

---

## 4. Deploy to Hetzner Cloud

Hetzner sells both **arm64** (CAX, Ampere) and **x86** (CX, CPX, CCX) servers. A CAX server runs the same arm64 images you rehearsed under the `rpi4`/`rpi5` profiles, so it's the closest match to an edge Pi.

### 4.1 Set up the CLI

```bash
# https://github.com/hetznercloud/cli
hcloud context create p4n4                # paste an API token (Console → Project → Security → API tokens)
hcloud ssh-key create --name laptop --public-key-from-file ~/.ssh/id_ed25519.pub
```

### 4.2 Firewall and server

```bash
MY_IP=$(curl -s https://ifconfig.me)

hcloud firewall create --name p4n4
hcloud firewall add-rule p4n4 --direction in --protocol tcp --port 22  --source-ips "$MY_IP/32"
hcloud firewall add-rule p4n4 --direction in --protocol tcp --port 80  --source-ips 0.0.0.0/0 --source-ips ::/0
hcloud firewall add-rule p4n4 --direction in --protocol tcp --port 443 --source-ips 0.0.0.0/0 --source-ips ::/0
hcloud firewall add-rule p4n4 --direction in --protocol udp --port 443 --source-ips 0.0.0.0/0 --source-ips ::/0
# Only with MQTT over TLS:
# hcloud firewall add-rule p4n4 --direction in --protocol tcp --port 8883 --source-ips <device-cidr>

hcloud server create \
  --name greenhouse \
  --type cax21 \
  --image ubuntu-24.04 \
  --location fsn1 \
  --ssh-key laptop \
  --firewall p4n4 \
  --user-data-from-file cloud-init.yaml

hcloud server ip greenhouse
```

`cax21` matches the `rpi5` profile. Use `cax11` for `rpi4`, or an x86 type for `nuc`. Run `hcloud server-type list` for current types and `hcloud location list` for locations. CAX servers are available in some locations only.

### 4.3 Persistent storage (optional)

The Docker volumes (InfluxDB, Grafana, Mosquitto) and `data/archive/` live on the server's disk by default. They survive reboots but not `hcloud server delete`. To keep data separate from the server, attach a volume and put Docker's data root on it:

```bash
hcloud volume create --name greenhouse-data --size 50 --server greenhouse --automount --format ext4
# On the server: the mount point is /mnt/HC_Volume_<id>
#   add  "data-root": "/mnt/HC_Volume_<id>/docker"  to /etc/docker/daemon.json, then
#   sudo systemctl restart docker
```

Do this before the first `up`. Moving an existing data root means stopping Docker and copying `/var/lib/docker`.

### 4.4 Ship the project

Wait for cloud-init to finish (`ssh deploy@<ip> cloud-init status --wait`), then copy the project without its local data and secrets:

```bash
rsync -az --exclude .env --exclude 'data/archive/*/' ~/projects/greenhouse/ deploy@<ip>:~/greenhouse/
scp ~/projects/greenhouse/.env.production deploy@<ip>:~/greenhouse/.env    # your production .env
ssh deploy@<ip> 'chmod 600 ~/greenhouse/.env && mkdir -p ~/greenhouse/data/archive/{raw,lineprotocol}'
```

A Git checkout of the project works too, with `.env` copied separately. Don't use `docker context` to run Compose remotely from your workstation. The template bind-mounts `./config` and `./data`, and those paths would resolve on the server, where the files aren't.

### 4.5 Start and verify

```bash
ssh deploy@<ip>
cd ~/greenhouse
docker compose config --quiet && docker compose up -d     # or: p4n4 validate && p4n4 up
docker compose ps                                         # mqtt, influxdb, grafana healthy; caddy, telegraf running
docker compose logs -f caddy                              # "certificate obtained successfully"
```

Open `https://grafana.example.com`. To check ingestion end to end, publish over an SSH tunnel:

```bash
ssh -L 1883:localhost:1883 deploy@<ip>     # in another terminal:
mosquitto_pub -h localhost -t sensors/house-1/temperature -m '{"value": 23.4, "unit": "C"}'
```

### 4.6 Operate it

| Task | How |
|---|---|
| Backups | `hcloud server enable-backup greenhouse` (daily, 7 kept, priced as a share of the server), or snapshots with `hcloud server create-image --type snapshot greenhouse`. For consistent InfluxDB backups, also run `docker exec p4n4-influxdb influx backup /var/lib/influxdb2/backup-$(date +%F) -t "$INFLUXDB_TOKEN"` from cron and copy the result off the server |
| Updates | Raise the image tags in `docker-compose.yml` when you update the template, then `docker compose pull && docker compose up -d`. `unattended-upgrades` covers the OS |
| Resize | `hcloud server change-type greenhouse cax31 --keep-disk` (the server must be powered off). `--keep-disk` lets you scale back down later |
| Logs | `docker compose logs --since 1h <service>`. Log size is capped by the cloud-init `daemon.json` |

---

## 5. Bring your own host (BYOH)

Any Linux host that meets the [prerequisites](#1-prerequisites) runs a template the same way: an on-prem server, a NUC at the site, AWS, Azure, DigitalOcean or a Raspberry Pi. The steps don't depend on the provider:

1. **Provision** Ubuntu 24.04 (or Debian 12), x86_64 or arm64, sized by the emulator profile you rehearsed ([2.3](#23-pick-a-profile-that-matches-the-target)). Give it a static IP and, for HTTPS, a DNS name.
2. **Bootstrap** with the [cloud-init file](#35-shared-cloud-init) if the provider accepts user data. Otherwise run its `runcmd` steps by hand over SSH.
3. **Firewall** at the provider or network level ([3.4](#34-firewall-use-the-providers-firewall)).
4. **Ship** the project and `.env` ([4.4](#44-ship-the-project)). Set `ARCHIVE_UID`/`ARCHIVE_GID` for the server's user.
5. **Start** with `docker compose up -d` and verify ([4.5](#45-start-and-verify)).
6. **Back up** volumes and `data/archive/` off the host, using the provider's disk snapshots plus `influx backup`.

Run `p4n4-emu setup --check-only` on the host itself to check Docker, Compose and cgroup v2 there. To keep a large shared host's p4n4 services inside a fixed budget, run `p4n4-emu up --profile nuc --native` on it. The same overlay that emulates hardware on a workstation then acts as a resource cap.

### 5.1 Example: Google Cloud (Compute Engine)

#### Project and network

```bash
gcloud config set project <project-id>
gcloud config set compute/zone europe-west1-b
gcloud services enable compute.googleapis.com

gcloud compute addresses create greenhouse-ip --region europe-west1   # static external IP
```

The `default` VPC ships `default-allow-ssh` (port 22 from `0.0.0.0/0`) and `default-allow-rdp`. Delete those rules, or use a dedicated network. Then allow SSH only through [Identity-Aware Proxy](https://cloud.google.com/iap/docs/using-tcp-forwarding):

```bash
gcloud compute firewall-rules delete default-allow-ssh default-allow-rdp --quiet

gcloud compute firewall-rules create p4n4-ssh-iap \
  --network default --direction INGRESS --allow tcp:22 \
  --source-ranges 35.235.240.0/20 --target-tags p4n4
gcloud compute firewall-rules create p4n4-web \
  --network default --direction INGRESS --allow tcp:80,tcp:443,udp:443 \
  --source-ranges 0.0.0.0/0 --target-tags p4n4
# Only with MQTT over TLS:
# gcloud compute firewall-rules create p4n4-mqtts --network default --allow tcp:8883 \
#   --source-ranges <device-cidr> --target-tags p4n4
```

#### Instance

```bash
gcloud compute instances create greenhouse \
  --machine-type e2-standard-4 \
  --image-family ubuntu-2404-lts-amd64 --image-project ubuntu-os-cloud \
  --boot-disk-size 50GB --boot-disk-type pd-balanced \
  --address greenhouse-ip \
  --tags p4n4 \
  --metadata-from-file user-data=cloud-init.yaml \
  --metadata enable-oslogin=TRUE
```

| Emulator profile | GCP machine type | Image family |
|---|---|---|
| `rpi4`, `rpi5` (arm64) | `t2a-standard-2` (Tau T2A, 2 vCPU / 8 GB; available in some regions only) | `ubuntu-2404-lts-arm64` |
| `nuc` (x86_64) | `e2-standard-4` (4 vCPU / 16 GB) | `ubuntu-2404-lts-amd64` |

With OS Login, `gcloud compute ssh` signs you in as your Google identity. Drop the `users:` block from `cloud-init.yaml` and run `sudo usermod -aG docker $USER` once. The rest of this guide's `deploy@<ip>` steps then become:

```bash
gcloud compute ssh greenhouse --tunnel-through-iap
gcloud compute scp --tunnel-through-iap --recurse ~/projects/greenhouse greenhouse:~/   # then copy .env separately
gcloud compute ssh greenhouse --tunnel-through-iap -- -L 8086:localhost:8086 -L 1883:localhost:1883
```

The IAP tunnel doesn't copy file permissions the way `rsync` does. Run `chmod 600 ~/greenhouse/.env` on the instance.

#### Storage and backups

```bash
gcloud compute resource-policies create snapshot-schedule greenhouse-daily \
  --region europe-west1 --daily-schedule --start-time 03:00 --max-retention-days 14
gcloud compute disks add-resource-policies greenhouse --resource-policies greenhouse-daily
```

For data that must outlive the instance, attach a separate persistent disk (`gcloud compute disks create` and `instances attach-disk`) and point Docker's `data-root` at it, as in [4.3](#43-persistent-storage-optional).

#### GPU for the ai layer (optional)

Hetzner Cloud has no GPU servers, so for an Ollama-heavy template GCP's L4 GPUs are an option:

```bash
gcloud compute instances create greenhouse-ai \
  --machine-type g2-standard-4 --maintenance-policy TERMINATE \
  --image-family ubuntu-2404-lts-amd64 --image-project ubuntu-os-cloud \
  --boot-disk-size 100GB --tags p4n4 --metadata-from-file user-data=cloud-init.yaml
```

Install the NVIDIA driver and the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) on the instance. Then give Ollama the GPU in `ai/docker-compose.override.yml`:

```yaml
services:
  ollama:
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

---

## Multi-layer templates

`mqtt-influx-grafana-ollama` (iot + ai) and `retail-vision` (iot + ai + edge) keep one Compose project per layer (`iot/`, `ai/`, `edge/`). Each layer has its own `.env` ([ADR-002](../decisions/adr/ADR-002.md)). These steps change:

- **Emulator.** `p4n4-emu up --profile <p>` at the project root starts every layer in dependency order, with the combined limits scaled down to fit one device. Use `--stack ai --native` to keep Ollama out of QEMU.
- **Secrets.** `INFLUXDB_TOKEN`, `INFLUXDB_ORG` and `INFLUXDB_BUCKET` must be identical in every layer's `.env`. `p4n4 secret rotate` keeps them in sync. Don't edit one file by hand.
- **Overrides.** Write one `docker-compose.override.yml` **per layer directory**. Put Caddy in `iot/`. Bind the ai layer's agent (`11434`) to `127.0.0.1` in `ai/docker-compose.override.yml`, since it has no authentication.
- **Start order.** iot first, because it creates `p4n4-net`, and stop in reverse: `p4n4 up` / `p4n4 down`, or `(cd iot && docker compose up -d) && (cd ai && docker compose up -d)`.
- **Sizing.** On first start, Ollama pulls `OLLAMA_MODEL` (`gemma4:e2b`, about 4.6 GB). Leave disk room for it, and expect CPU-only inference on CAX/E2 machines to answer in seconds rather than milliseconds.

## Adding the dashboard and API

To serve [p4n4-dashboard](../stacks/dashboard.md) and [p4n4-api](../reference/api.md) from the same host, keep the API off the public interface (`P4N4_API_HOST=172.17.0.1`). Then proxy the dashboard container through Caddy (`reverse_proxy p4n4-dashboard:8088`) and route Grafana through the dashboard's `/grafana/` path (`GRAFANA_SUB_PATH=/grafana/`) to avoid mixed content. The [Security guide](security.md#dashboard) lists the settings. The [greenhouse use case](../use-cases/greenhouse-telemetry.md) ties the template, the API and a white-label dashboard together.

## Pre-launch checklist

- [ ] Rehearsed under `p4n4-emu` with the profile matching the chosen VM. `tests/smoke.sh` passes
- [ ] `.env` has real secrets, set before the first start on the server. `COMPOSE_PROFILES` is empty
- [ ] `docker-compose.override.yml` binds MQTT and InfluxDB to `127.0.0.1` (or the VPN address). Grafana is reachable only through Caddy
- [ ] Provider firewall allows only 22 (restricted), 80 and 443, plus 8883 or the VPN port if used
- [ ] `allow_anonymous false` if MQTT is reachable from outside the host
- [ ] `GRAFANA_ANONYMOUS=false` unless every viewer may see every dashboard
- [ ] Backups configured and one restore tested
