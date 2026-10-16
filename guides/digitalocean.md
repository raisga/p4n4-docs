# Emulators on DigitalOcean

This guide sets up a DigitalOcean account to run p4n4 projects on throwaway cloud emulators. An emulator is a Droplet that runs [`p4n4-emu`](../reference/emulator.md), so a project gets the same per-board limits it gets on a workstation. Each run has four stages:

1. **Create the emulator**: a Droplet sized for the profile.
2. **Run the project** on it under `p4n4-emu up`, with simulated sensors.
3. **Get the report**: `p4n4-emu report` bundles health, usage, restarts and logs, and the run's cost follows from its run time.
4. **Delete the emulator**: destroy the Droplet and check that nothing is left running.

[Sections 1–3](#1-before-you-start) are done once per account. [Sections 4–7](#4-create-an-emulator) are repeated for every run. [AWS](aws.md), [Google Cloud](gcp.md) and [Azure](azure.md) have the same guide.

```
 workstation (doctl, SSH key)                   DigitalOcean team p4n4-emu
 ────────────────────────────                   ─────────────────────────────────────────────
 doctl compute droplet create ──────────────►  Droplet (x86), project p4n4-emu, tag p4n4-emu
                                                 cloud-init: Docker, uv, p4n4-emu, TTL timer
 ssh p4n4@<ip> (cloud firewall: 22 from you) ◄►  p4n4-emu up --profile rpi5 --native --sim
                                                 p4n4-emu report ─► /tmp/<run>.tar.gz
 scp ◄──────────────────────────────────────────────────────────────┘
 run time × hourly price  (per-second billing)
 doctl compute droplet delete ──────────────►  Droplet and its disk destroyed
                                               (or by the Droplet itself, when its TTL timer fires)
```

**What's different on DigitalOcean.** DigitalOcean has **no Arm Droplets**, so the Raspberry Pi profiles' arm64 images can't run natively. Pick one of two modes per run:

| Mode | How | What it tests | Cost of it |
|---|---|---|---|
| **native** (default here) | `p4n4-emu up --native`: x86 images, the Pi's CPU, memory and disk limits | Whether the stack fits the board: memory headroom, OOM kills, CPU saturation, disk throughput | The images are x86, not the arm64 ones the Pi runs |
| **qemu** | `p4n4-emu up` after `p4n4-emu setup --arch arm64`: the arm64 images, emulated | Whether the arm64 images start and work at all | 3–10× slower than a Pi, so it says nothing about performance |

Run native for soak and sizing tests, and a short qemu run when you change images or architecture-specific code. For native arm64 in the cloud, use [AWS](aws.md), [Google Cloud](gcp.md) or [Azure](azure.md).

**Why the rest is set up this way:**

- The Droplets live in a **team of their own**. A Droplet deletes itself when its TTL is over, which needs an API token on the Droplet. A token's reach is its team, so a separate team keeps that token away from anything else.
- **A powered-off Droplet is still billed**, so the TTL destroys the Droplet rather than shutting it down. Destroying a Droplet also destroys its disk, and nothing else is left over, because the runs use no volumes or reserved IPs.
- SSH is open only to your own IP address, through a cloud firewall, with key authentication only. Root login is turned off.
- Droplets are billed per second, and the price covers the CPU, memory, disk and a transfer allowance. A run's cost is its run time × the size's price, with no billing export to set up.

To deploy a project for real use rather than test it, see [Deploying a template](deployment.md). Its BYOH steps apply to Droplets as well.

---

## 1. Before you start

| On | Needs |
|---|---|
| Workstation | [`doctl`](https://docs.digitalocean.com/reference/doctl/how-to/install/), an SSH key pair (`ssh-keygen -t ed25519`), `jq` |
| DigitalOcean | An account with 2FA on, and a payment method |
| Project | A p4n4 project, or a [template](../reference/template-registry.md) to copy. This guide uses `mqtt-influx-grafana`, as [Deploying a template](deployment.md) does |

The Droplet installs p4n4-emu from the `main` branch on GitHub. [Section 6](#6-get-the-report) needs `p4n4-emu report`, which is on `main` once the commit that adds it is pushed.

---

## 2. Create and secure the account (once)

### 2.1 Sign up

1. Sign up at <https://cloud.digitalocean.com/registrations/new> and add a payment method. Accounts start with a **Droplet limit**, the number of Droplets that can exist at once, and GPU Droplets may need a request first ([2.6](#26-check-the-droplet-limit)).
2. Pick a **region** close to you, for example `nyc3`, `sfo3`, `ams3`, `fra1` or `sgp1`. The GPU profile needs `tor1` ([4.4](#44-gpu-profile-jetson-orin-nano)).

### 2.2 Secure the account and make a team for the emulators

1. **Two-factor authentication.** Account menu → **My Account** → *Two-factor authentication*. Every teammate turns it on for their own login.
2. **A team for the emulators.** Account menu → **Create a team** → `p4n4-emu`. Its Droplets, firewall, tag, project and tokens are separate from any other team you have.
3. **A second owner.** In the team's **Settings → Members**, invite a second trusted person and make them **Owner**, so losing one login doesn't lock you out.

### 2.3 Guard the bill

**Billing alert.** In the team: **Settings → Billing → Billing alerts** → set an amount, for example USD 50. You get an email when the month's usage passes it. There's no API for this setting.

A billing alert only warns. It doesn't stop anything. The TTL in [4.2](#42-bootstrap-file) is what does.

### 2.4 Give teammates access

In the team: **Settings → Members → Invite members**, with the role **Member**. Members can create and destroy Droplets. **Modifier** can't destroy resources, so it can't run this guide. **Billing Viewer** suits someone who only reads the costs.

### 2.5 Configure doctl

Each person creates their own token: in the team, **API → Tokens → Generate New Token**, with an expiry (90 days, say). Either full access, or custom scopes covering `droplet`, `ssh_key`, `firewall`, `project`, `tag`, `sizes`, `regions`, `image` and `billing` (read). Then:

```bash
doctl auth init --context p4n4-emu          # paste the token
doctl auth switch --context p4n4-emu
doctl account get                           # the team's account, its email and Droplet limit

# Your SSH key, once per person
doctl compute ssh-key import "$(whoami)-laptop" --public-key-file ~/.ssh/id_ed25519.pub
```

A token belongs to the team it was created in. With `--context`, doctl keeps it apart from tokens of other teams.

### 2.6 Check the Droplet limit

```bash
doctl account get --format DropletLimit,Status
```

Each run is one Droplet, so the limit is how many runs can exist at once. Raise it under **Settings → Team → Droplet limit → Increase**. GPU Droplets may need a separate request in the control panel.

---

## 3. Shared resources (once)

These are created once, by an owner. Every emulator uses them:

| Resource | Name | For |
|---|---|---|
| Project | `p4n4-emu` | Groups the Droplets in the control panel, and in the invoice's items |
| Tag | `p4n4-emu` | Marks every emulator Droplet. The firewall applies to it |
| Cloud firewall | `p4n4-emu` | SSH only from the team's IP addresses, and all outbound traffic |
| Reaper token | (stored on your workstation) | Lets a Droplet destroy itself at its TTL |

### 3.1 Project and tag

```bash
doctl projects create --name p4n4-emu --purpose "p4n4 emulators" --environment Development
doctl compute tag create p4n4-emu
```

### 3.2 Cloud firewall

A DigitalOcean cloud firewall drops any traffic that no rule allows, **outbound too**. It needs outbound rules, or the Droplet can't pull images or clone from GitHub:

```bash
MY_IP=$(curl -4 -s https://ifconfig.me)
doctl compute firewall create --name p4n4-emu --tag-names p4n4-emu \
  --inbound-rules "protocol:tcp,ports:22,address:$MY_IP/32" \
  --outbound-rules "protocol:tcp,ports:all,address:0.0.0.0/0,address:::/0 protocol:udp,ports:all,address:0.0.0.0/0,address:::/0 protocol:icmp,address:0.0.0.0/0,address:::/0"
```

The firewall applies to every Droplet with the `p4n4-emu` tag, including ones created later. Each teammate adds their own address once, and again when it changes:

```bash
FW=$(doctl compute firewall list --format ID,Name --no-header | awk '$2 == "p4n4-emu" {print $1}')
doctl compute firewall add-rules $FW --inbound-rules "protocol:tcp,ports:22,address:$(curl -4 -s https://ifconfig.me)/32"
# An old address: doctl compute firewall remove-rules $FW --inbound-rules "protocol:tcp,ports:22,address:<old-ip>/32"
```

The firewall never opens port 3000. You see Grafana through an SSH port forward ([5.4](#54-open-grafana)).

### 3.3 Reaper token for the TTL

A Droplet can only delete itself with an API token, because DigitalOcean has no instance identities or roles. Create a token that can do as little as possible: in the team, **API → Tokens → Generate New Token** → name `p4n4-emu-reaper`, expiry 90 days, **Custom scopes** → `droplet:delete`. The scope page adds the read scopes it needs: `droplet`, `regions`, `sizes`, `actions` and `image`. Save the token where only you can read it:

```bash
install -d -m 700 ~/.config/p4n4
printf '%s\n' '<the token>' > ~/.config/p4n4/do-reaper-token && chmod 600 ~/.config/p4n4/do-reaper-token
```

Share it with teammates through a password vault, and make a new one before it expires.

**What the token can do.** It is written into each Droplet's user data, which any process on the Droplet can read from the metadata service, the containers included. With it, a process can read and delete **any Droplet in the team**, but not create resources or touch other teams. This is why the emulators have a team of their own ([2.2](#22-secure-the-account-and-make-a-team-for-the-emulators)): the worst case is losing other emulators.

### 3.4 Shared reports (optional)

Reports land in `~/p4n4-reports` on whoever takes them. For a team archive, DigitalOcean Spaces works with any S3 client, but it has a monthly minimum charge. A shared drive the team already uses works as well.

---

## 4. Create an emulator

These variables are used in every step of a run. Keep them in one terminal, or save them to a file and `source` it:

```bash
doctl auth switch --context p4n4-emu
REGION=nyc3
PROFILE=rpi5                       # the p4n4-emu profile
SIZE=g-4vcpu-16gb                  # from the table below
MODE=native                        # native or qemu
TTL_MIN=480                        # the Droplet destroys itself after 8 h
RUN=$PROFILE-$(date +%Y%m%d-%H%M)  # names and tags this run, e.g. rpi5-20261015-1430
NAME=p4n4-emu-$RUN
KEY=$(ssh-keygen -E md5 -lf ~/.ssh/id_ed25519.pub | awk '{print $2}' | sed 's/^MD5://')   # your key's fingerprint in DigitalOcean
PROJECT_ID=$(doctl projects list --format ID,Name --no-header | awk '$2 == "p4n4-emu" {print $1}')
```

### 4.1 Pick a size for the profile

The Droplet must be **larger** than the board. p4n4-emu caps the containers at the profile's cores and memory, and the OS and Docker need room outside that cap. Use a size with at least as many vCPUs as the board has cores, so `--cpuset auto` can pin each service to a core. Give it about 1 GB more memory than the profile.

| Profile | Board | Size | vCPU / memory | Notes |
|---|---|---|---|---|
| `rpi-zero2w`, `rpi3` | 4 cores, 384 / 768 MB | `c-4` | 4 dedicated / 8 GB | CPU-Optimized |
| `rpi4` | 4 cores, 3.5 GB | `c-4` | 4 dedicated / 8 GB | |
| `rpi5` | 4 cores, 7 GB | `g-4vcpu-16gb` | 4 dedicated / 16 GB | General Purpose |
| `jetson-orin-nano` | 6 cores, 7 GB, GPU | `gpu-4000adax1-20gb` | 8 / 32 GB, NVIDIA RTX 4000 Ada | `tor1` only. See [4.4](#44-gpu-profile-jetson-orin-nano) |
| `mcu-class` | 1 core, 256 MB, x86 | `s-1vcpu-2gb` | 1 shared / 2 GB | Basic |
| `nuc` | 4 cores, 14 GB, x86 | `m-4vcpu-32gb` | 4 dedicated / 32 GB | Memory-Optimized |

**Dedicated or shared.** CPU-Optimized, General Purpose and Memory-Optimized sizes have dedicated vCPUs. Basic sizes (`s-…`) share theirs with other customers and cost less, but their speed varies. Use Basic for smoke tests and dedicated sizes for soak tests and benchmarks.

**Disk.** Every Droplet has a local SSD whose throughput is above every profile's disk limit, so p4n4-emu's limit is the one that applies.

**qemu mode** is CPU-bound, so a larger size (`g-8vcpu-32gb`) shortens the run. The profile's limits still cap the containers at the board's core count.

Sizes and prices change; list the current ones, and those offered in your region:

```bash
doctl compute size list --format Slug,VCPUs,Memory,Disk,PriceHourly,Regions | grep -E "^($SIZE|Slug)"
```

### 4.2 Bootstrap file

cloud-init runs this file on the Droplet's first boot. It creates the `p4n4` user with your SSH key and turns root login off. It installs Docker, `uv` and p4n4-emu. It also sets up the TTL: a systemd timer that destroys the Droplet `TTL_MIN` minutes after first boot. The timer is persistent, so it still fires after a reboot.

```bash
cat > emu-cloud-init.yaml <<'EOF'
#cloud-config
package_update: true
packages: [ca-certificates, curl, git, jq]

users:
  - default                       # root, which gets the SSH keys DigitalOcean injects
  - name: p4n4
    shell: /bin/bash
    lock_passwd: true
    sudo: "ALL=(ALL) NOPASSWD:ALL"

write_files:
  - path: /etc/docker/daemon.json
    content: |
      { "log-driver": "json-file", "log-opts": { "max-size": "10m", "max-file": "5" } }
  - path: /etc/ssh/sshd_config.d/10-p4n4.conf
    content: |
      PermitRootLogin no
      PasswordAuthentication no
  - path: /etc/p4n4-ttl.env
    permissions: "0600"
    content: |
      DO_TOKEN=__REAPER_TOKEN__
  - path: /usr/local/bin/p4n4-ttl
    permissions: "0755"
    content: |
      #!/bin/sh
      # The run's time to live: destroy this Droplet and its disk
      set -eu
      . /etc/p4n4-ttl.env
      id=$(curl -sf http://169.254.169.254/metadata/v1/id)
      curl -sf -X DELETE -H "Authorization: Bearer $DO_TOKEN" "https://api.digitalocean.com/v2/droplets/$id"
  - path: /etc/systemd/system/p4n4-ttl.service
    content: |
      [Unit]
      Description=p4n4 emulator TTL: destroy this Droplet
      Wants=network-online.target
      After=network-online.target

      [Service]
      Type=oneshot
      ExecStart=/usr/local/bin/p4n4-ttl
      Restart=on-failure
      RestartSec=60

runcmd:
  - printf '[Timer]\nOnCalendar=%s\nPersistent=true\n\n[Install]\nWantedBy=timers.target\n' "$(date -u -d '+__TTL_MIN__ min' '+%Y-%m-%d %H:%M:%S UTC')" > /etc/systemd/system/p4n4-ttl.timer
  - systemctl daemon-reload
  - systemctl enable --now p4n4-ttl.timer
  - install -d -o p4n4 -g p4n4 -m 700 /home/p4n4/.ssh
  - install -o p4n4 -g p4n4 -m 600 /root/.ssh/authorized_keys /home/p4n4/.ssh/authorized_keys
  - systemctl restart ssh
  - curl -fsSL https://get.docker.com | sh
  - usermod -aG docker p4n4
  - su - p4n4 -c 'curl -LsSf https://astral.sh/uv/install.sh | sh'
  - su - p4n4 -c 'mkdir -p p4n4/tools && git clone -q https://github.com/raisga/p4n4-lib p4n4/lib && git clone -q https://github.com/raisga/p4n4-emu p4n4/tools/emu && git clone -q https://github.com/raisga/p4n4-templates p4n4/tools/templates'
  - su - p4n4 -c '~/.local/bin/uv tool install ~/p4n4/tools/emu --with ~/p4n4/lib'
EOF
sed -i "s/__TTL_MIN__/$TTL_MIN/; s/__REAPER_TOKEN__/$(cat ~/.config/p4n4/do-reaper-token)/" emu-cloud-init.yaml
chmod 600 emu-cloud-init.yaml
```

- When the timer fires, the Droplet and its disk are destroyed. Copy the report off before then ([section 6](#6-get-the-report)).
- To give a running emulator more time, move the timer on the Droplet, for example to 4 hours from now:

  ```bash
  sudo sed -i "s/^OnCalendar=.*/OnCalendar=$(date -u -d '+4 hours' '+%Y-%m-%d %H:%M:%S UTC')/" /etc/systemd/system/p4n4-ttl.timer
  sudo systemctl daemon-reload && sudo systemctl restart p4n4-ttl.timer
  systemctl list-timers p4n4-ttl.timer      # shows when it fires
  ```

  `sudo systemctl disable --now p4n4-ttl.timer` cancels the TTL. Then only [section 7](#7-delete-the-emulator) deletes the run.
- The file now holds the reaper token, which is why it's `chmod 600`. Delete it after the run, or keep it out of any repository.
- `--with ~/p4n4/lib` installs p4n4-lib from the checkout. p4n4-emu needs a p4n4-lib version that isn't on PyPI yet.
- To test a specific release rather than `main`, add `--branch v<version>` to the `git clone` commands.

### 4.3 Create the Droplet

```bash
read -r ID IP < <(doctl compute droplet create $NAME \
  --region $REGION --size $SIZE --image ubuntu-24-04-x64 \
  --ssh-keys $KEY --project-id $PROJECT_ID \
  --tag-names "p4n4-emu,run:$RUN" \
  --user-data-file emu-cloud-init.yaml \
  --wait --format ID,PublicIPv4 --no-header)
echo "$RUN $ID $IP"
```

- `--wait` returns once the Droplet is running, usually within a minute. cloud-init keeps working for a few minutes after that ([5.1](#51-connect-and-check)).
- The `p4n4-emu` tag puts the Droplet behind the firewall from [3.2](#32-cloud-firewall). The `run:<run>` tag identifies the run in the cleanup checks ([7.2](#72-check-that-nothing-is-left)).
- `--ssh-keys $KEY` adds only your key. To let a teammate in too, add their key's fingerprint, comma-separated (`doctl compute ssh-key list`).

### 4.4 GPU profile (`jetson-orin-nano`)

The `jetson-orin-nano` profile reserves an NVIDIA GPU for `ollama` and `ei-runner`. DigitalOcean's GPU Droplets are x86, so this profile always runs with host-architecture images (`--native`). Set `REGION=tor1` and `SIZE=gpu-4000adax1-20gb`. Check [GPU Droplet availability](https://docs.digitalocean.com/products/droplets/details/gpu-availability/) for other regions and GPU types.

After cloud-init finishes, install the driver and the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) on the Droplet:

```bash
sudo apt-get install -y ubuntu-drivers-common && sudo ubuntu-drivers install --gpgpu
# add NVIDIA's apt repository as the toolkit guide shows, then:
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime docker && sudo reboot
```

After the reboot, `docker info | grep -i runtimes` should list `nvidia`. Start the stack with `p4n4-emu up --profile jetson-orin-nano --native`. GPU Droplets cost far more per hour than the others, so keep `TTL_MIN` short.

---

## 5. Run a project on it

### 5.1 Connect and check

```bash
ssh p4n4@$IP
```

```bash
# On the Droplet
cloud-init status --wait                  # "status: done"; if not, see /var/log/cloud-init-output.log
systemctl list-timers p4n4-ttl.timer      # the TTL: when the run destroys itself
p4n4-emu setup --check-only               # Docker, Compose and cgroup v2 OK
```

If `ssh p4n4@` is refused, cloud-init hasn't copied your key yet. Wait a minute. If `p4n4-emu` or `docker` isn't found yet, cloud-init is still running. The session's groups were also set before `usermod` added `p4n4` to `docker`, so after `cloud-init status --wait`, log out and back in.

Then, depending on the mode:

```bash
# native: calibrate the x86 cores against the boards' cores
p4n4-emu calibrate
p4n4-emu calibrate --show

# qemu: register QEMU for arm64 first, then calibrate the emulated platform
p4n4-emu setup --arch arm64
p4n4-emu calibrate --platform arm64
p4n4-emu calibrate --show
```

In native mode, calibration matters. A dedicated Droplet core does more work than a Pi core, so without calibration a 4-core `rpi5` would get four fast cores. With it, `up` scales each CPU limit by the Pi core's speed relative to the Droplet's ([core speed](https://github.com/raisga/p4n4-emu#hardware-profiles)). Under QEMU an emulated core is usually slower than a Pi core, so the factor stays at 1, and QEMU's own slowness is the limit.

### 5.2 Put the project on it

**From a template.** On the Droplet:

```bash
cp -r ~/p4n4/tools/templates/projects/mqtt-influx-grafana ~/greenhouse
cd ~/greenhouse
cp .env.example .env
sed -i "s/^ARCHIVE_UID=.*/ARCHIVE_UID=$(id -u)/; s/^ARCHIVE_GID=.*/ARCHIVE_GID=$(id -g)/" .env
```

**Your own project.** From your workstation, copy it over, leaving out local data. `.env` can go along, because the Droplet is thrown away. Don't ship production secrets this way, though. Afterwards, set `ARCHIVE_UID`/`ARCHIVE_GID` in `.env` on the Droplet as above.

```bash
rsync -az --exclude 'data/archive/*/' ~/projects/greenhouse/ p4n4@$IP:~/greenhouse/
```

For a multi-layer project (iot + ai, …), ship the whole project directory. `p4n4-emu up` at its root starts every layer ([multi-layer templates](deployment.md#multi-layer-templates)).

### 5.3 Start it under the profile

```bash
# native mode
p4n4-emu up --profile rpi5 --native --dry-run          # look at the overlay first
p4n4-emu up --profile rpi5 --native --sim --sim-devices 3 --cpuset auto

# qemu mode: the same without --native
p4n4-emu up --profile rpi5 --sim --sim-devices 3 --cpuset auto

p4n4-emu status                                        # usage against each service's limits
```

Check the summary that `up` prints:

- **Platform** says `native` in native mode, and `linux/arm64` in qemu mode.
- **Block device detected** names the Droplet's disk (`/dev/vda`), so the disk limits apply. If it says no limit (no block device), the CPU and memory limits still apply, but disk I/O isn't throttled.
- **CPU limit** shows the calibrated factor in native mode, for example `2.00 host cores = 4 × 0.500`.

Leave the stack running as long as the test calls for, from minutes for a smoke test to hours for a soak test, staying within the TTL. The template's `./tests/smoke.sh` runs its own throwaway copy of the stack. Run it on the Droplet as well if you want its end-to-end check.

### 5.4 Open Grafana

The port forward runs in a second terminal on your workstation, over SSH. The firewall never opens port 3000:

```bash
ssh -N -L 3000:localhost:3000 p4n4@$IP
```

Then open <http://localhost:3000>. Add `-L 8086:localhost:8086` for InfluxDB, or `-L 1880:localhost:1880` for Node-RED in the templates that have it.

---

## 6. Get the report

### 6.1 Make the report on the Droplet

`p4n4-emu report` samples every container's CPU and memory several times. It records what Docker knows about each container's life: state, health, restarts and OOM kills. It also saves each container's last log lines. It writes everything to one archive:

| File | Contents |
|---|---|
| `summary.md` | For people: issues found, and per stack a table of state, health, restarts, CPU and memory (mean / peak / limit) |
| `report.json` | For scripts: everything `status --json` has, plus `usage_over_time`, `lifecycle`, `host` (the Droplet's hostname, Docker version, calibration) and `issues` |
| `logs/<container>.log` | The last `--tail` lines (default 500) of each container, with timestamps, the simulator's included |

Issues come in two levels. **Errors** are a container that isn't running, an unhealthy, OOM-killed or restarted container, or limits that differ from the overlay. **Warnings** are a peak at 90 % or more of a CPU or memory limit, cgroup v1, or no stack under p4n4-emu. `--fail-on-issues` exits 1 on any error, for CI.

Run it from your workstation, then copy the archive back:

```bash
ssh p4n4@$IP "bash -lc 'cd ~/greenhouse && p4n4-emu report --samples 12 --interval 5 \
  -l run=$RUN -l profile=$PROFILE -l mode=$MODE -l size=$SIZE -l region=$REGION -o /tmp/$RUN.tar.gz'"

mkdir -p ~/p4n4-reports && cd ~/p4n4-reports
scp p4n4@$IP:/tmp/$RUN.tar.gz .
```

`bash -lc` runs a login shell, which puts `~/.local/bin`, and with it `p4n4-emu`, on the `PATH`. The `mode` label matters when you compare runs, because a native run and a qemu run of the same profile give very different numbers. This report samples for about a minute (12 samples, 5 s apart). Sample longer to catch periodic load, such as `--samples 60 --interval 10`.

Take the report **before** the TTL or [section 7](#7-delete-the-emulator) ends the run. A destroyed Droplet's disk is gone.

### 6.2 Read it

```bash
tar xzf $RUN.tar.gz
cat p4n4-emu-report-*/summary.md          # or open it in an editor that renders Markdown
jq '.issues' p4n4-emu-report-*/report.json
```

Comparing two runs' `report.json` shows a regression, for example an image update that raised a service's peak memory under the same profile and mode:

```bash
jq -r '[.labels.mode] + (.stacks[].services[] | [.service, .usage_over_time.memory.peak]) | @tsv' */report.json
```

### 6.3 Cost report

Droplets are billed **per second**, with a minimum of 60 seconds or USD 0.01. The price covers CPU, memory, the disk and a transfer allowance that a test run doesn't come near. A run's cost is therefore its run time × the size's hourly price. Note the start time and price before deleting the Droplet ([7.1](#71-destroy-the-droplet)):

```bash
CREATED=$(doctl compute droplet get $ID --format Created --no-header)
PRICE=$(doctl compute size list --format Slug,PriceHourly --no-header | awk -v s=$SIZE '$1 == s {print $2}')
```

After the deletion, compute the cost:

```bash
SECS=$(( $(date -u +%s) - $(date -u -d "$CREATED" +%s) ))
awk -v run=$RUN -v s=$SECS -v p=$PRICE \
  'BEGIN {printf "%s: %.2f h × USD %s/h = USD %.2f\n", run, s/3600, p, s/3600*p}' | tee -a ~/p4n4-reports/costs.txt
```

`~/p4n4-reports/costs.txt` keeps one line per run.

**Check it against the invoice.** DigitalOcean updates a preview of the month's invoice daily, itemized by resource and project:

```bash
doctl invoice get preview --format Product,Description,Amount,ProjectName | grep -E 'p4n4-emu|Product'
```

The month's total for the team, which is all emulators, is `doctl balance get` (`MonthToDateUsage`).

---

## 7. Delete the emulator

### 7.1 Destroy the Droplet

Note `CREATED` and `PRICE` first ([6.3](#63-cost-report)), then:

```bash
doctl compute droplet delete $ID --force
```

You don't need `p4n4-emu down` first. Destroying the Droplet destroys its disk, and the containers, volumes and overlays go with it. Billing stops at once. The firewall, tag and project stay, and they're free.

### 7.2 Check that nothing is left

Anything still running costs money. Powered-off Droplets too. Check after each run, and from time to time for everyone's runs:

```bash
# Emulator Droplets that still exist (any run, anyone's)
doctl compute droplet list --tag-name p4n4-emu --format ID,Name,Size,Status,Created,Tags

# Every Droplet in the team, in case one was created without the tag
doctl compute droplet list --format ID,Name,Size,Status,Created

# Volumes and snapshots, which the runs don't create (should be empty)
doctl compute volume list
doctl compute snapshot list
```

A Droplet whose status is `off` is still billed. Destroy it.

### 7.3 Remove the account setup (when you're done with DigitalOcean)

```bash
doctl compute firewall delete $FW --force
doctl compute tag delete p4n4-emu --force
doctl projects delete $PROJECT_ID --force     # only an empty project can be deleted
rm ~/.config/p4n4/do-reaper-token
```

Then delete the `p4n4-emu-reaper` token and your own token under **API → Tokens**. Finally, delete the team: **Settings → Team → Delete team**.

---

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `You have reached the Droplet limit` | The team's limit counts every Droplet that exists. Destroy leftovers ([7.2](#72-check-that-nothing-is-left)) or raise it ([2.6](#26-check-the-droplet-limit)) |
| `size is not available in this region` | The size isn't offered in `$REGION`. See the `size list` check in [4.1](#41-pick-a-size-for-the-profile). GPU sizes are in a few regions only |
| `ssh: connect … timed out` | The firewall has no rule for your current IP (`curl -4 ifconfig.me`). Add it ([3.2](#32-cloud-firewall)), or check that the Droplet has the `p4n4-emu` tag |
| `Permission denied (publickey)` as `p4n4` | cloud-init hasn't copied the key yet (wait a minute), or the Droplet was created without your key (`--ssh-keys`) |
| The Droplet can't reach the internet | The firewall has no outbound rules. Add them as in [3.2](#32-cloud-firewall) |
| `p4n4-emu: command not found` | cloud-init hasn't finished or failed. Run `cloud-init status --wait`, then `sudo cat /var/log/cloud-init-output.log` |
| `permission denied … docker.sock` | The session started before `usermod -aG docker p4n4` ran. Log out and back in |
| `exec format error` in qemu mode | QEMU isn't registered. Run `p4n4-emu setup --arch arm64` and start again |
| `up` says no limit (no block device) | p4n4-emu didn't find the disk under Docker's data root, so disk I/O isn't throttled. CPU and memory limits still apply. Check `findmnt -no SOURCE /var/lib/docker` |
| The run vanished mid-test | The TTL timer fired. Raise `TTL_MIN` for long soaks, or move the timer on the Droplet ([4.2](#42-bootstrap-file)) |
| The TTL didn't destroy the Droplet | Run `sudo systemctl status p4n4-ttl.service` on it. A `401` means the reaper token expired or was deleted: make a new one ([3.3](#33-reaper-token-for-the-ttl)) and destroy the Droplet by hand |
| `report` says no containers found | It ran outside the project directory, or after the stack stopped. `cd` to the project, or pass `--stack-dir` |

## Checklist per run

- [ ] `doctl auth switch --context p4n4-emu`, variables set ([4](#4-create-an-emulator)), `MODE` chosen, `TTL_MIN` long enough for the test
- [ ] Size from the table in [4.1](#41-pick-a-size-for-the-profile), offered in `$REGION`
- [ ] The firewall allows your current IP; the reaper token hasn't expired
- [ ] On the Droplet: `cloud-init status` done, the TTL timer listed, `p4n4-emu setup --check-only` OK, `p4n4-emu calibrate` run (qemu mode: `setup --arch arm64` and `calibrate --platform arm64`)
- [ ] `up` shows the expected platform, a block device and, in native mode, the calibrated CPU factor
- [ ] Report copied back **before** destroying; `CREATED` and `PRICE` noted
- [ ] Droplet destroyed, cost line written, [7.2](#72-check-that-nothing-is-left) shows nothing left
