# Testing on a Jetson Orin Nano

This guide turns a Jetson Orin Nano Developer Kit into a remote test host for p4n4 projects. When you finish, you can reach it from anywhere:

- **SSH and remote desktop** over [Tailscale](https://tailscale.com/), with no ports open on the router
- **Selected web UIs** (Grafana, the dashboard) on your own domain through a [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/), behind Cloudflare Access, for people who aren't on your tailnet
- **The p4n4 stacks** running natively on arm64, with Ollama on the Jetson's GPU

```
 you (laptop, phone)                                   Jetson Orin Nano (home / lab LAN)
 ───────────────────                                   ─────────────────────────────────────
 ssh, VS Code, NoMachine ──► tailnet (WireGuard) ────► tailscaled ─► sshd :22, nxserver :4000,
                                                                     every p4n4 port
 teammate's browser ──► Cloudflare edge + Access ──► cloudflared (outbound only)
                                                       ─► Grafana :3000, dashboard :8088
```

Neither tunnel needs inbound ports, so the Jetson can sit behind NAT or CGNAT. Tailscale is the admin path, and anything that can change the system goes through it. Cloudflare only exposes the read-mostly web UIs you choose.

This guide is for **testing**. For a production host, see [Deploying a template](deployment.md) and the [Security guide](security.md).

---

## 1. What you need

| Item | Notes |
|---|---|
| Jetson Orin Nano Developer Kit (8 GB) | Running **JetPack 6.x** (L4T r36, Ubuntu 22.04, arm64). JetPack 6.2 or later adds the *Super* power mode |
| Storage | An NVMe SSD (M.2 2280) is strongly recommended. The images and an Ollama model come to 15–25 GB, and a microSD card is slow for InfluxDB and Docker layers |
| DisplayPort dummy plug | Optional, about $10. It gives a headless Jetson a real display for NoMachine ([5.2](#52-run-it-headless)). The dev kit has DisplayPort only, no HDMI |
| Network | Ethernet is preferred. Wi-Fi works, but set it up before you unplug the monitor |
| Accounts | Tailscale (the free plan is enough), and Cloudflare with a domain on it (only for [section 6](#6-share-web-uis-with-cloudflare-tunnel)) |
| On your laptop | Tailscale, an SSH client, the NoMachine client |

Placeholders used below: the Jetson's user is `p4n4`, its hostname is `jetson`, your tailnet is `example-tailnet.ts.net` and your domain is `example.com`.

### Check the starting point

Do this on the Jetson, with a monitor attached or over SSH on the LAN:

```bash
cat /etc/nv_tegra_release     # R36 (release), REVISION: 4.x → JetPack 6.x
lsb_release -ds               # Ubuntu 22.04.x LTS
df -h /                       # root should be on nvme0n1p1 if you booted from the SSD
uname -m                      # aarch64
```

If it reports `R35`, the Jetson is on JetPack 5. That can't be upgraded in place, so reflash with [SDK Manager or the SD card image](https://developer.nvidia.com/embedded/learn/get-started-jetson-orin-nano-devkit) first. JetPack 6.x itself updates in place with `sudo apt update && sudo apt dist-upgrade`.

---

## 2. Base system

### 2.1 Update, set the hostname, install tools

```bash
sudo apt update && sudo apt dist-upgrade -y
sudo hostnamectl set-hostname jetson
sudo apt install -y curl git htop tmux python3-pip nvme-cli
sudo reboot
```

### 2.2 Power mode and monitoring

List the power modes, then select the highest one. On JetPack 6.2+ that's `MAXN_SUPER`, and its ID depends on the module, so read it from the list rather than assuming a number:

```bash
sudo nvpmodel -q --verbose | grep -E "POWER_MODEL|NV Power Mode"
sudo nvpmodel -m <id>          # e.g. the MAXN_SUPER id; may ask to reboot
sudo jetson_clocks             # optional: pin clocks at max until reboot (more heat, steadier benchmarks)
```

Install `jtop` (from jetson-stats) to watch CPU, GPU, memory, temperature and power. You'll use it to confirm Ollama is on the GPU:

```bash
sudo pip3 install -U jetson-stats
sudo reboot                    # jtop's service starts after a reboot
jtop
```

### 2.3 Memory: swap on the SSD

The 8 GB is shared by the CPU and the GPU. The desktop, the iot stack and a loaded model compete for it. JetPack enables zram by default. A swap file on NVMe keeps the OOM killer away when a model loads:

```bash
sudo systemctl disable --now nvzramconfig
sudo fallocate -l 16G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
free -h
```

Skip this on a microSD card, because swap wears it out. Use a smaller model instead ([7.3](#73-ai-layer-with-the-gpu)).

### 2.4 Keep it running unattended

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
gsettings set org.gnome.desktop.session idle-delay 0          # as the desktop user: no screen blank/lock
gsettings set org.gnome.desktop.screensaver lock-enabled false
```

Give the Jetson a DHCP reservation on your router, so it keeps its LAN address as a fallback if Tailscale is down.

---

## 3. Docker with the NVIDIA runtime

A JetPack 6 image may not include Docker. Install the NVIDIA container runtime and Docker Engine, then make `nvidia` the default runtime so containers can reach the GPU:

```bash
sudo apt install -y nvidia-container
curl -fsSL https://get.docker.com | sudo sh
sudo systemctl enable --now docker
sudo usermod -aG docker $USER          # log out and back in afterwards

sudo nvidia-ctk runtime configure --runtime=docker
```

Edit `/etc/docker/daemon.json` so it reads as follows. The log limits matter because Docker keeps container logs forever by default:

```json
{
  "runtimes": {
    "nvidia": { "args": [], "path": "nvidia-container-runtime" }
  },
  "default-runtime": "nvidia",
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```

If the root filesystem is on a small microSD card and the SSD is mounted elsewhere, add `"data-root": "/mnt/nvme/docker"` too.

```bash
sudo systemctl restart docker
docker info | grep -iE "runtimes|default runtime"   # Default Runtime: nvidia
docker compose version                               # ≥ 2.24.4
```

---

## 4. Tailscale: SSH from anywhere

### 4.1 Join the tailnet

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh --hostname jetson
```

Open the login URL it prints and approve the device. Then, in the [Tailscale admin console](https://login.tailscale.com/admin/machines):

1. On the `jetson` machine, choose **Disable key expiry**. Otherwise it drops off the tailnet after 180 days, usually while you're away from it.
2. Optional: give it a tag (`tag:p4n4-test`) and limit who can reach it in the access controls. A tailnet you share with other people should do this.
3. Keep **MagicDNS** on, so `jetson` (or `jetson.example-tailnet.ts.net`) resolves from every device.

### 4.2 Connect

With `--ssh`, Tailscale SSH answers on port 22 for tailnet traffic. Your Tailscale identity authenticates you, so you don't need keys:

```bash
ssh p4n4@jetson
```

The stock OpenSSH server keeps working on the LAN as a fallback. Harden it to keys only:

```bash
# from the laptop, while password login still works
ssh-copy-id p4n4@<jetson-lan-ip>

# on the Jetson
sudo tee /etc/ssh/sshd_config.d/10-p4n4.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
EOF
sudo systemctl restart ssh
```

**VS Code.** The Remote-SSH extension works with host `p4n4@jetson`. Its server supports arm64 Linux, so you edit and run code on the Jetson directly.

### 4.3 Reach the p4n4 web UIs over the tailnet

Docker publishes most p4n4 ports on every interface, so from a tailnet device `http://jetson:3000` (Grafana), `:1880` (Node-RED), `:5678` (n8n) and `:8088` (the dashboard) just work. Ollama, Letta, the inference runner and p4n4-api bind `127.0.0.1` ([Security guide](security.md#network-exposure)). Use an SSH forward for those rather than changing their bind addresses:

```bash
ssh -L 11434:localhost:11434 -L 8080:localhost:8080 -L 8000:localhost:8000 p4n4@jetson
```

For HTTPS inside the tailnet (useful for the dashboard on a phone), `sudo tailscale serve --bg --https=443 http://localhost:8088` serves `https://jetson.example-tailnet.ts.net` with a real certificate. It stays tailnet-only, so this is not Funnel.

### 4.4 Firewall (optional)

UFW doesn't filter Docker-published ports ([deployment guide 3.4](deployment.md#34-firewall-use-the-providers-firewall)), but it does cover host services such as sshd and NoMachine. To keep NoMachine off the LAN:

```bash
sudo ufw default deny incoming
sudo ufw allow in on tailscale0
sudo ufw allow from 192.168.0.0/16 to any port 22 proto tcp     # LAN SSH fallback; adjust to your subnet
sudo ufw enable
```

---

## 5. NoMachine: remote desktop

### 5.1 Install

Download the **NoMachine for Linux ARM, arm64 (ARMv8) DEB** from [nomachine.com/download](https://www.nomachine.com/download). Copy the current file name from that page, since the version changes:

```bash
wget https://download.nomachine.com/download/<ver>/Arm/nomachine_<ver>_arm64.deb
sudo dpkg -i nomachine_*_arm64.deb
sudo /etc/NX/nxserver --status
```

On the laptop, install the NoMachine client and add a connection to host `jetson` (the MagicDNS name), port `4000`, protocol NX. Log in with the Jetson's Ubuntu account. With a monitor or dummy plug attached, you get the real GNOME session, including the login screen after a reboot.

### 5.2 Run it headless

Without a display, the Jetson's X server has nothing to draw on, and NoMachine shows a black screen or no desktop at all. Pick one fix:

| Option | How | Trade-off |
|---|---|---|
| **A. DisplayPort dummy plug** (recommended) | Plug it in and reboot | Nothing else to configure. GNOME and GPU/EGL apps (camera pipelines, `nvargus`) behave as with a monitor |
| **B. NoMachine virtual desktop** | `sudo systemctl set-default multi-user.target`, install Xfce (`sudo apt install -y xfce4`), reboot, `sudo /etc/NX/nxserver --restart`. NoMachine then creates an Xfce desktop on connect | Frees about 500 MB of RAM for the stacks. GNOME doesn't render reliably in NoMachine's virtual display, hence Xfce. Some EGL apps fail without a real display. Undo with `sudo systemctl set-default graphical.target` |

### 5.3 If NoMachine is slow

In the session menu (Ctrl+Alt+0 → Display), lower the resolution, or select *Prefer speed* in the display settings. Tailscale usually connects peer to peer. If `tailscale status` shows `relay "…"` for the Jetson, it is going through a DERP relay. That happens behind strict NAT, and it adds latency.

---

## 6. Share web UIs with Cloudflare Tunnel

Skip this section if only tailnet members need access. Use it to show a teammate or a client the Grafana or dashboard of a test project. Don't publish anything without authentication: no InfluxDB, Ollama, Letta, the runner, p4n4-api's `:8000`, MQTT or Node-RED. Admin work goes over Tailscale.

### 6.1 Install cloudflared and create the tunnel

1. In the Cloudflare dashboard, go to **Zero Trust → Networks → Tunnels → Create a tunnel**. Choose *Cloudflared* and name it `jetson`.
2. Install the connector on the Jetson, using the token the dashboard shows:

   ```bash
   sudo mkdir -p --mode=0755 /usr/share/keyrings
   curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
   echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared jammy main" \
     | sudo tee /etc/apt/sources.list.d/cloudflared.list
   sudo apt update && sudo apt install -y cloudflared
   sudo cloudflared service install <TUNNEL_TOKEN>
   systemctl status cloudflared
   ```

   The tunnel shows as *Healthy* in the dashboard once the service starts.

### 6.2 Public hostnames

Add these on the tunnel's **Public Hostname** tab:

| Hostname | Service |
|---|---|
| `grafana-jetson.example.com` | `http://localhost:3000` |
| `p4n4-jetson.example.com` | `http://localhost:8088` (the dashboard) |

Grafana needs to know its public URL, or redirects go to `localhost`. Set it in the iot layer's `docker-compose.override.yml`, then run `docker compose up -d grafana`:

```yaml
services:
  grafana:
    environment:
      GF_SERVER_ROOT_URL: https://grafana-jetson.example.com/
```

### 6.3 Put Cloudflare Access in front

In **Zero Trust → Access → Applications → Add an application → Self-hosted**, create one application per hostname (or one with both). Add a policy such as *Allow — Emails: you@…, teammate@…*, or *Emails ending in: @your-company.com*. Visitors then sign in with a one-time PIN or your identity provider before they reach Grafana's own login.

Keep `GRAFANA_ANONYMOUS=false` for anything published here. Access is the outer lock, and Grafana's login is the inner one.

### 6.4 n8n webhooks (optional)

To test workflows that external services call, add `n8n-jetson.example.com → http://localhost:5678`. Set `N8N_HOST` and `WEBHOOK_URL=https://n8n-jetson.example.com/` in the ai layer's override ([its example file](https://github.com/raisga/p4n4-ai/blob/main/docker-compose.override.yml.example) has the block). Add an Access **bypass** policy for the `/webhook/*` path only, so callers can reach the webhooks while the editor stays behind Access.

---

## 7. Run p4n4 projects

### 7.1 Install the CLI

Ubuntu 22.04 ships Python 3.10, and p4n4 needs 3.11 or later. `uv` fetches a managed Python, so you don't have to touch the system one:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
exec $SHELL -l
uv tool install --python 3.12 p4n4
p4n4 --version
```

To test unreleased changes, clone the monorepo with its submodules and use `scripts/dev` instead ([Development](development.md)).

### 7.2 Start a project

Either scaffold one:

```bash
p4n4 init jetson-test --layer iot,ai
cd jetson-test && p4n4 up
```

Or copy a template:

```bash
git clone https://github.com/raisga/p4n4-templates.git ~/p4n4-templates
cp -r ~/p4n4-templates/projects/mqtt-influx-grafana-ollama ~/projects/greenhouse
cd ~/projects/greenhouse
for d in iot ai; do cp $d/.env.example $d/.env; done   # set real secrets before the first start
p4n4 up
```

The images p4n4 uses are multi-arch, so they pull native arm64 builds. To check an image before you rely on it, run `docker manifest inspect <image> | grep arm64`.

### 7.3 AI layer with the GPU

Create `ai/docker-compose.override.yml` (in a single-layer ai project, it goes at the root) so that Ollama runs with the NVIDIA runtime:

```yaml
services:
  ollama:
    runtime: nvidia
    environment:
      NVIDIA_VISIBLE_DEVICES: all
```

On Jetson, use `runtime: nvidia` instead of the `deploy.resources.reservations.devices` block from the example file. The latter is the dGPU path. With `default-runtime` set in [section 3](#3-docker-with-the-nvidia-runtime), the runtime line is technically redundant, but it documents what the service relies on.

```bash
p4n4 up
docker exec p4n4-ollama ollama run <model> "hello"
docker exec p4n4-ollama ollama ps          # PROCESSOR should read "100% GPU"
docker logs p4n4-ollama 2>&1 | grep -i "inference compute"
```

Keep `jtop` open in a second terminal. Its GPU graph should move while the model answers.

**If `ollama ps` shows CPU.** The pinned `ollama/ollama` image may not detect the Jetson's integrated GPU. Use the Jetson build from [jetson-containers](https://github.com/dusty-nv/jetson-containers) instead, picking a tag that matches your L4T release (`r36.4`, …) from [Docker Hub](https://hub.docker.com/r/dustynv/ollama/tags):

```yaml
services:
  ollama:
    image: dustynv/ollama:<version>-r36.4
    runtime: nvidia
    environment:
      OLLAMA_MODELS: /root/.ollama/models
```

That image is community-maintained and lags the stack's pinned Ollama version. Treat results from it as Jetson-specific.

**Model size.** All 8 GB of memory is shared. With the desktop and the iot stack running, stick to models of 4 GB or less (`gemma4:e2b`, `phi3:mini`, `qwen2.5:3b`). Keep `OLLAMA_MAX_LOADED_MODELS=1`, as the ai `.env` has it.

### 7.4 Other things worth testing here

- **Inference runner** (`ei-runner` profile): use the **AARCH64** `.eim` build. For a GPU-enabled `.eim`, add `runtime: nvidia` to `ei-runner` in the same override file ([Edge Inference](../stacks/edge-inference.md)).
- **System 1** (`system1` profile): this is a native arm64 host, which [System 1 §4.7](../decisions/system1.md#47-arm64) still needs results from. First confirm that `ghcr.io/raisga/p4n4-system1` has an arm64 manifest (`docker manifest inspect`). Expect CPU-only PyTorch numbers unless the image is built on an L4T base.
- **Dashboard + API:** `p4n4 init … --layer all` adds the dashboard container. Reach it on `http://jetson:8088` over the tailnet, or through the tunnel ([6.2](#62-public-hostnames)).

### 7.5 After a reboot

The stacks' containers restart on their own (`RESTART_POLICY=unless-stopped`), and so do Tailscale, cloudflared and NoMachine, which run as systemd services. Check them with:

```bash
tailscale status; systemctl is-active cloudflared; sudo /etc/NX/nxserver --status
cd ~/projects/greenhouse && p4n4 status
```

---

## 8. Troubleshooting

| Symptom | Check |
|---|---|
| `ssh p4n4@jetson` hangs | `tailscale status` on both ends. Make sure the Jetson isn't expired in the admin console. Fall back to its LAN IP |
| NoMachine shows a black screen | No display: see [5.2](#52-run-it-headless). After switching to option B, run `sudo /etc/NX/nxserver --restart` |
| Containers can't see the GPU | `docker info` must show `Default Runtime: nvidia`. Re-run `sudo nvidia-ctk runtime configure --runtime=docker` and restart Docker |
| Ollama or InfluxDB is OOM-killed | `docker inspect <container> --format '{{.State.OOMKilled}}'`. Use a smaller model, add swap ([2.3](#23-memory-swap-on-the-ssd)), or go headless with option B |
| Cloudflare hostname returns 502 | The service isn't listening on the Jetson: `curl -I http://localhost:3000`. Check the port in the public hostname |
| Grafana redirects to `localhost` through the tunnel | `GF_SERVER_ROOT_URL` isn't set ([6.2](#62-public-hostnames)) |
| Thermal throttling in long runs | `jtop` → temperature. The dev kit's fan needs airflow. Skip `jetson_clocks` for soak tests |

## Checklist

- [ ] JetPack 6.x on NVMe, highest power mode set, `jtop` installed
- [ ] Docker with `Default Runtime: nvidia` and log limits
- [ ] Tailscale up with `--ssh`, key expiry disabled, MagicDNS name works
- [ ] OpenSSH keys-only. UFW limits NoMachine to `tailscale0` (optional)
- [ ] NoMachine connects over the tailnet, with a dummy plug or an Xfce virtual desktop when headless
- [ ] Cloudflare Tunnel publishes only Grafana and the dashboard, both behind Access
- [ ] A p4n4 project runs, and `ollama ps` reports GPU
