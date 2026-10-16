# Emulators on Google Cloud

This guide sets up a Google Cloud account to run p4n4 projects on throwaway cloud emulators. An emulator is a Compute Engine VM that runs [`p4n4-emu`](../reference/emulator.md), so a project gets the same per-board limits it gets on a workstation. Each run has four stages:

1. **Create the emulator**: a VM sized for the profile.
2. **Run the project** on it under `p4n4-emu up`, with simulated sensors.
3. **Get the report**: `p4n4-emu report` bundles health, usage, restarts and logs, and the billing data shows what the run cost.
4. **Delete the emulator**: delete the VM and check that nothing is left running.

[Sections 1–3](#1-before-you-start) are done once per account. [Sections 4–7](#4-create-an-emulator) are repeated for every run. To create the shared resources (section 3) and the emulators (section 4) with Terraform instead, see [p4n4-emu's `terraform/`](https://github.com/raisga/p4n4-emu/tree/main/terraform). [AWS](aws.md), [Azure](azure.md) and [DigitalOcean](digitalocean.md) have the same guide.

```
 workstation (gcloud, OS Login)                 Google Cloud project p4n4-emu-…
 ──────────────────────────────                 ─────────────────────────────────────────────
 gcloud compute instances create ───────────►  VM (Axion C4A arm64 / x86), no service account
                                                 cloud-init: Docker, uv, p4n4-emu
 gcloud compute ssh --tunnel-through-iap ◄────►  p4n4-emu up --profile rpi5 --sim
   (SSH only from Google's IAP range)            p4n4-emu report ─► /tmp/<run>.tar.gz
 gcloud compute scp ◄─────────────────────────────────────────────────────┘
 gcloud storage cp ─► gs://…/reports/            billing export (label p4n4-run) ─► bq query
 gcloud compute instances delete ───────────►  VM + boot disk deleted (or at --max-run-duration)
```

**Why it's set up this way:**

- Arm instances (Google Axion, C4A) run the Raspberry Pi profiles' arm64 images natively, without the 3–10× slowdown of QEMU on an x86 workstation.
- The VM accepts SSH only through Identity-Aware Proxy (IAP), and you sign in with your Google identity (OS Login). The VM has no service account, so code on it can't call Google Cloud APIs.
- Every VM is created with a maximum run duration, after which Google Cloud deletes it. An emulator you forget about stops costing money.

To deploy a project for real use rather than test it, see [Deploying a template → Google Cloud](deployment.md#51-example-google-cloud-compute-engine).

---

## 1. Before you start

| On | Needs |
|---|---|
| Workstation | The [gcloud CLI](https://cloud.google.com/sdk/docs/install) (it includes `gcloud storage` and `bq`) |
| Google | A Google account with 2-Step Verification on, and a payment card. Use a shared account for the billing owner (`gcp@example.com`), not one person's. Teammates sign in with their own Google accounts |
| Project | A p4n4 project, or a [template](../reference/template-registry.md) to copy. This guide uses `mqtt-influx-grafana`, as [Deploying a template](deployment.md) does |

The VM installs p4n4-emu from the `main` branch on GitHub. [Section 6](#6-get-the-report) needs `p4n4-emu report`, which is on `main` once the commit that adds it is pushed.

---

## 2. Create and secure the account (once)

### 2.1 Sign up and leave the free trial

1. Go to <https://console.cloud.google.com> with the billing owner's Google account and start the free trial. It creates a **Cloud Billing account** and gives credits for 90 days.
2. **Upgrade to a paid account** (Billing → Overview → **Upgrade**). The free trial can't add GPUs or request quota increases, and resources stop when the trial ends. After the upgrade you keep the unused credits until they expire, and you pay only once they're used up. The current terms are on [Google Cloud's free program page](https://cloud.google.com/free/docs/free-cloud-features).
3. Pick a **region** close to you that has C4A (Axion) machines, for example `us-central1`, `us-east4` or `europe-west4`. [4.1](#41-pick-a-machine-for-the-profile) shows how to check.

### 2.2 Secure the owner account

Whoever owns the billing account can see and change every project and every cost. Protect that account:

1. **2-Step Verification** on the billing owner's Google account (myaccount.google.com → Security). A passkey or security key is best.
2. Give at least two trusted people **Billing Account Administrator** (Billing → Account management → **Add principal**), so losing one account doesn't lock you out of billing.
3. Never create service account keys for this setup. Everyone uses their own Google sign-in, and the VMs have no service account at all.

### 2.3 Configure the CLI

```bash
gcloud config configurations create p4n4      # keeps these settings apart from your other projects
gcloud auth login                             # opens a browser
gcloud billing accounts list                  # note the ACCOUNT_ID, e.g. 0X0X0X-0X0X0X-0X0X0X
```

### 2.4 Create the project

Project IDs are unique across all of Google Cloud, so add a suffix:

```bash
BILLING=0X0X0X-0X0X0X-0X0X0X
PROJECT=p4n4-emu-$(openssl rand -hex 3)       # e.g. p4n4-emu-4f2a9c
REGION=us-central1

gcloud projects create $PROJECT --name "p4n4 emulators"
gcloud billing projects link $PROJECT --billing-account $BILLING
gcloud config set project $PROJECT
gcloud config set compute/region $REGION
gcloud services enable compute.googleapis.com iap.googleapis.com oslogin.googleapis.com \
  storage.googleapis.com bigquery.googleapis.com billingbudgets.googleapis.com
```

Keeping the emulators in a project of their own means one `gcloud projects delete` removes all of them ([7.3](#73-remove-the-account-setup-when-youre-done-with-google-cloud)), and the billing reports separate their costs from everything else.

### 2.5 Guard the bill

**A budget.** Alerts at 80 % of actual spend and 100 % of forecast spend go to the billing account's administrators:

```bash
gcloud billing budgets create --billing-account $BILLING \
  --display-name p4n4-emu-monthly --budget-amount 50USD \
  --filter-projects projects/$PROJECT \
  --threshold-rule percent=0.8 \
  --threshold-rule percent=1.0,basis=forecasted-spend
```

A budget only alerts. It doesn't stop anything, and the per-VM run limit in [4.3](#43-create-the-vm) is what does.

**Billing export to BigQuery.** The cost report in [6.3](#63-cost-report) queries it from the CLI. Create the dataset, then turn the export on in the console. There is no gcloud command for that step:

```bash
bq --location US mk --dataset $PROJECT:billing_export
```

Billing → **Billing export** → *Standard usage cost* → **Edit settings** → project `$PROJECT`, dataset `billing_export` → **Save**. Turn it on before the first run: costs from before the export was enabled may be missing from it. The console's billing **Reports** page works without the export.

### 2.6 Give teammates access

The project's creator is its Owner, which covers everything below. For each teammate who will run emulators:

```bash
MEMBER=user:teammate@example.com
for role in roles/compute.instanceAdmin.v1 roles/compute.osAdminLogin \
            roles/iap.tunnelResourceAccessor roles/bigquery.jobUser; do
  gcloud projects add-iam-policy-binding $PROJECT --member $MEMBER --role $role --condition None
done
```

| Role | For |
|---|---|
| `compute.instanceAdmin.v1` | Create, list and delete VMs and disks |
| `compute.osAdminLogin` | Sign in to a VM with OS Login, with `sudo` |
| `iap.tunnelResourceAccessor` | Reach the VM's SSH port through IAP |
| `bigquery.jobUser` | Run the cost query. Also grant **BigQuery Data Viewer** on the `billing_export` dataset (BigQuery → dataset → **Share**) |

The VMs have no service account, so nobody needs `iam.serviceAccountUser`. Access to the reports bucket is granted in [3.1](#31-reports-bucket).

### 2.7 Check the CPU and GPU quotas

```bash
gcloud compute regions describe $REGION --flatten quotas \
  --format 'table(quotas.metric, quotas.limit, quotas.usage)' | grep -E 'CPUS|GPUS'
gcloud compute project-info describe --flatten quotas \
  --format 'table(quotas.metric, quotas.limit)' | grep GPUS_ALL_REGIONS
```

An emulator needs 4 vCPUs of its family (`C4A_CPUS` for Axion, `N4_CPUS` for N4, and so on), and 8 for the GPU profile. GPU quotas (`NVIDIA_L4_GPUS`, `GPUS_ALL_REGIONS`) usually start at 0. Request increases under **IAM & Admin → Quotas & System Limits**; the free trial can't request them.

---

## 3. Shared resources (once)

These are created once, by the project's Owner. Every emulator uses them:

| Resource | Name | For |
|---|---|---|
| VPC network | `p4n4-emu` | The emulators' own network, without the default network's open SSH and RDP rules |
| Firewall rule | `p4n4-emu-iap-ssh` | SSH on port 22, only from IAP's range, only to VMs tagged `p4n4-emu` |
| Storage bucket | `<project-id>` (e.g. `p4n4-emu-4f2a9c`) | The team's archive of reports (`reports/`) |

### 3.1 Reports bucket

```bash
BUCKET=$PROJECT                               # bucket names are global too; the project ID is unique
gcloud storage buckets create gs://$BUCKET --location $REGION \
  --uniform-bucket-level-access --public-access-prevention
cat > lifecycle.json <<'EOF'
{"rule": [{"action": {"type": "Delete"}, "condition": {"age": 180, "matchesPrefix": ["reports/"]}}]}
EOF
gcloud storage buckets update gs://$BUCKET --lifecycle-file lifecycle.json
gcloud storage buckets update gs://$BUCKET --update-labels app=p4n4

# For each teammate:
gcloud storage buckets add-iam-policy-binding gs://$BUCKET \
  --member user:teammate@example.com --role roles/storage.objectAdmin
```

Reports reach the bucket from your workstation, not from the VM ([6.1](#61-make-the-report-on-the-vm)), so the VM needs no access to it.

### 3.2 Network and firewall

```bash
gcloud compute networks create p4n4-emu --subnet-mode auto
gcloud compute firewall-rules create p4n4-emu-iap-ssh --network p4n4-emu \
  --direction INGRESS --allow tcp:22 --source-ranges 35.235.240.0/20 --target-tags p4n4-emu
```

A new VPC network denies all inbound traffic except what its rules allow, and allows all outbound traffic. That leaves SSH from IAP's range, `35.235.240.0/20`, and lets the VM pull images and clone from GitHub through its ephemeral external IP. To see Grafana you forward a port through SSH ([5.4](#54-open-grafana)), so the rule doesn't open port 3000.

---

## 4. Create an emulator

These variables are used in every step of a run. Keep them in one terminal, or save them to a file and `source` it:

```bash
gcloud config configurations activate p4n4
PROJECT=$(gcloud config get project)
BUCKET=$PROJECT
ZONE=us-central1-a                 # a zone of your region that offers $TYPE (see 4.1)
gcloud config set compute/zone $ZONE

PROFILE=rpi5                       # the p4n4-emu profile
TYPE=c4a-standard-4                # from the table below
ARCH=arm64                         # arm64 for C4A / T2A, amd64 for x86
DISK=hyperdisk-balanced            # from the table below
TTL=8h                             # Google Cloud deletes the VM after this
RUN=$PROFILE-$(date +%Y%m%d-%H%M)  # names and labels this run, e.g. rpi5-20261015-1430
VM=p4n4-emu-$RUN
```

### 4.1 Pick a machine for the profile

The VM must be **larger** than the board. p4n4-emu caps the containers at the profile's cores and memory, and the OS and Docker need room outside that cap. Use a machine with at least as many vCPUs as the board has cores, so `--cpuset auto` can pin each service to a core. Give it about 1 GB more memory than the profile.

| Profile | Board | Machine type | vCPU / memory | Boot disk | Notes |
|---|---|---|---|---|---|
| `rpi-zero2w`, `rpi3` | 4 cores, 384 / 768 MB | `c4a-highcpu-4` | 4 / 8 GB | `hyperdisk-balanced` | Calibration scales the CPU limits down to a fraction of an Axion core each |
| `rpi4` | 4 cores, 3.5 GB | `c4a-highcpu-4` | 4 / 8 GB | `hyperdisk-balanced` | |
| `rpi5` | 4 cores, 7 GB | `c4a-standard-4` | 4 / 16 GB | `hyperdisk-balanced` | |
| `jetson-orin-nano` | 6 cores, 7 GB, GPU | `g2-standard-8` | 8 / 32 GB, NVIDIA L4 | `pd-balanced` | x86 with `--native`. See [4.4](#44-gpu-profile-jetson-orin-nano) |
| `mcu-class` | 1 core, 256 MB, x86 | `e2-small` | 2 shared / 2 GB | `pd-balanced` | `ARCH=amd64` |
| `nuc` | 4 cores, 14 GB, x86 | `n4-highmem-4` | 4 / 32 GB | `hyperdisk-balanced` | `ARCH=amd64`; add `--boot-disk-provisioned-throughput 250`, see below |

C4A, N4 and other recent machine series take only Hyperdisk boot disks, which is why the table lists a disk type for each. Tau T2A (`t2a-standard-4`, 4 / 16 GB, `pd-balanced`) is the older Arm option, in fewer regions.

**Cheaper for quick checks.** E2 machines (`e2-standard-4`, x86 only) cost less, but they share physical cores. Use them for smoke tests and dedicated-core machines for soak tests and benchmarks.

**Disk.** Hyperdisk Balanced and Balanced PD both give 3,000 IOPS and 140 MB/s at their base price. That is above every profile's disk limit except `nuc`'s 200 MB/s, which is why `nuc` provisions 250.

Check where the machine type exists before creating the VM:

```bash
gcloud compute machine-types list --filter "name=$TYPE AND zone ~ ^$REGION" --format 'value(zone)'
```

### 4.2 Bootstrap file

cloud-init runs this file on the VM's first boot. It creates a `p4n4` user (uid 2000) for the stack to run as, then installs Docker, `uv` and p4n4-emu. You sign in as your own OS Login user and switch to `p4n4` with `sudo`.

```bash
cat > emu-cloud-init.yaml <<'EOF'
#cloud-config
package_update: true
packages: [ca-certificates, curl, git, jq]

users:
  - name: p4n4
    uid: 2000
    shell: /bin/bash
    lock_passwd: true

write_files:
  - path: /etc/docker/daemon.json
    content: |
      { "log-driver": "json-file", "log-opts": { "max-size": "10m", "max-file": "5" } }

runcmd:
  - curl -fsSL https://get.docker.com | sh
  - usermod -aG docker p4n4
  - su - p4n4 -c 'curl -LsSf https://astral.sh/uv/install.sh | sh'
  - su - p4n4 -c 'mkdir -p p4n4/tools && git clone -q https://github.com/raisga/p4n4-lib p4n4/lib && git clone -q https://github.com/raisga/p4n4-emu p4n4/tools/emu && git clone -q https://github.com/raisga/p4n4-templates p4n4/tools/templates'
  - su - p4n4 -c '~/.local/bin/uv tool install ~/p4n4/tools/emu --with ~/p4n4/lib'
EOF
```

- `--with ~/p4n4/lib` installs p4n4-lib from the checkout. p4n4-emu needs a p4n4-lib version that isn't on PyPI yet.
- To test a specific release rather than `main`, add `--branch v<version>` to the `git clone` commands.

### 4.3 Create the VM

```bash
gcloud compute instances create $VM \
  --machine-type $TYPE \
  --image-family ubuntu-2404-lts-$ARCH --image-project ubuntu-os-cloud \
  --boot-disk-size 40GB --boot-disk-type $DISK \
  --network p4n4-emu --tags p4n4-emu \
  --no-service-account --no-scopes \
  --metadata enable-oslogin=TRUE \
  --metadata-from-file user-data=emu-cloud-init.yaml \
  --max-run-duration $TTL --instance-termination-action DELETE \
  --labels app=p4n4,p4n4-run=$RUN,p4n4-profile=$PROFILE

# Labels on the instance don't carry over to its boot disk, which has the instance's name
gcloud compute disks add-labels $VM --labels app=p4n4,p4n4-run=$RUN
```

- `--max-run-duration` with `DELETE` is the TTL. When it runs out, Google Cloud deletes the VM and its boot disk, so upload the report before then ([section 6](#6-get-the-report)). A running VM's limit can't be extended in place. Stopping and starting the VM restarts the clock (`gcloud compute instances stop $VM && gcloud compute instances start $VM`), and Docker restarts the containers. See [Limit the run time of a VM](https://cloud.google.com/compute/docs/instances/limit-vm-runtime).
- The `app` and `p4n4-run` labels are what the cost report ([6.3](#63-cost-report)) and the cleanup checks ([7.2](#72-check-that-nothing-is-left)) filter on. 40 GB leaves room for images, InfluxDB data and an Ollama model.

### 4.4 GPU profile (`jetson-orin-nano`)

The `jetson-orin-nano` profile reserves an NVIDIA GPU for `ollama` and `ei-runner`. Google Cloud's Arm machines have no GPUs, so this profile runs on x86 with an L4 GPU and host-architecture images (`--native`). Set `TYPE=g2-standard-8`, `ARCH=amd64` and `DISK=pd-balanced`, and add `--maintenance-policy TERMINATE` to `instances create`.

After cloud-init finishes, install the driver and the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) on the VM:

```bash
sudo apt-get install -y ubuntu-drivers-common && sudo ubuntu-drivers install --gpgpu
# add NVIDIA's apt repository as the toolkit guide shows, then:
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime docker && sudo reboot
```

After the reboot, `docker info | grep -i runtimes` should list `nvidia`. Start the stack with `p4n4-emu up --profile jetson-orin-nano --native`. The L4 needs GPU quota ([2.7](#27-check-the-cpu-and-gpu-quotas)), which the free trial doesn't allow.

---

## 5. Run a project on it

### 5.1 Connect and check

```bash
gcloud compute ssh $VM --tunnel-through-iap
```

The first time, gcloud creates an SSH key and registers it with OS Login. If the connection fails with error 4003, sshd isn't up yet: wait a minute and try again.

```bash
# On the VM (you are your OS Login user, e.g. you_example_com)
sudo -iu p4n4
cloud-init status --wait                  # "status: done"; if not, see /var/log/cloud-init-output.log
p4n4-emu setup --check-only               # Docker, Compose and cgroup v2 OK
p4n4-emu calibrate                        # benchmarks one Axion core against the boards' cores
p4n4-emu calibrate --show                 # the CPU factor each profile gets on this VM
```

Calibration matters here. An Axion core does several times the work of a Pi core, so without calibration a 4-core `rpi5` would get four fast cores. With it, `up` scales each CPU limit by the Pi core's speed relative to the VM's ([core speed](https://github.com/raisga/p4n4-emu#hardware-profiles)).

### 5.2 Put the project on it

**From a template.** On the VM, as `p4n4`:

```bash
cp -r ~/p4n4/tools/templates/projects/mqtt-influx-grafana ~/greenhouse
cd ~/greenhouse
cp .env.example .env
sed -i 's/^ARCHIVE_UID=.*/ARCHIVE_UID=2000/; s/^ARCHIVE_GID=.*/ARCHIVE_GID=2000/' .env   # the p4n4 user
```

**Your own project.** From your workstation, stream it straight into the `p4n4` user's home, leaving out local data. `.env` can go along, because the VM is thrown away. Don't ship production secrets this way, though.

```bash
tar czf - -C ~/projects --exclude 'greenhouse/data/archive/*/*' greenhouse \
  | gcloud compute ssh $VM --tunnel-through-iap --command 'sudo -iu p4n4 tar xzf - -C /home/p4n4'
```

Set `ARCHIVE_UID`/`ARCHIVE_GID` to `2000` in the project's `.env` on the VM. For a multi-layer project (iot + ai, …), ship the whole project directory. `p4n4-emu up` at its root starts every layer ([multi-layer templates](deployment.md#multi-layer-templates)).

### 5.3 Start it under the profile

```bash
p4n4-emu up --profile rpi5 --dry-run      # look at the overlay first
p4n4-emu up --profile rpi5 --sim --sim-devices 3 --cpuset auto
p4n4-emu status                           # usage against each service's limits
```

Check the summary that `up` prints:

- **Platform** says `native`. An Arm VM runs arm64 images without QEMU.
- **Block device detected** names the boot disk (`/dev/nvme0n1` on C4A and N4, `/dev/sda` on E2 and G2), so the disk limits apply. If it says no limit (no block device), the CPU and memory limits still apply, but disk I/O isn't throttled.
- **CPU limit** shows the calibrated factor, for example `1.60 host cores = 4 × 0.400`.

Leave the stack running as long as the test calls for, from minutes for a smoke test to hours for a soak test, staying within the TTL. The template's `./tests/smoke.sh` runs its own throwaway copy of the stack. Run it on the VM as well if you want its end-to-end check.

### 5.4 Open Grafana

The port forward runs in a second terminal on your workstation. It goes through the same IAP SSH connection, so no port opens on the VM:

```bash
gcloud compute ssh $VM --tunnel-through-iap -- -N -L 3000:localhost:3000
```

Then open <http://localhost:3000>. Add `-L 8086:localhost:8086` for InfluxDB, or `-L 1880:localhost:1880` for Node-RED in the templates that have it.

---

## 6. Get the report

### 6.1 Make the report on the VM

`p4n4-emu report` samples every container's CPU and memory several times. It records what Docker knows about each container's life: state, health, restarts and OOM kills. It also saves each container's last log lines. It writes everything to one archive:

| File | Contents |
|---|---|
| `summary.md` | For people: issues found, and per stack a table of state, health, restarts, CPU and memory (mean / peak / limit) |
| `report.json` | For scripts: everything `status --json` has, plus `usage_over_time`, `lifecycle`, `host` (the VM's hostname, Docker version, calibration) and `issues` |
| `logs/<container>.log` | The last `--tail` lines (default 500) of each container, with timestamps, the simulator's included |

Issues come in two levels. **Errors** are a container that isn't running, an unhealthy, OOM-killed or restarted container, or limits that differ from the overlay. **Warnings** are a peak at 90 % or more of a CPU or memory limit, cgroup v1, or no stack under p4n4-emu. `--fail-on-issues` exits 1 on any error, for CI.

Run it from your workstation, then copy the archive back and into the team's bucket:

```bash
gcloud compute ssh $VM --tunnel-through-iap --command "sudo -iu p4n4 bash -lc '\
  cd ~/greenhouse && p4n4-emu report --samples 12 --interval 5 \
  -l run=$RUN -l profile=$PROFILE -l machine-type=$TYPE -l zone=$ZONE \
  -o /tmp/$RUN.tar.gz'"

mkdir -p ~/p4n4-reports && cd ~/p4n4-reports
gcloud compute scp --tunnel-through-iap $VM:/tmp/$RUN.tar.gz .
gcloud storage cp $RUN.tar.gz gs://$BUCKET/reports/$RUN.tar.gz
```

This report samples for about a minute (12 samples, 5 s apart). Sample longer to catch periodic load, such as `--samples 60 --interval 10`. In an SSH session, run the same `p4n4-emu report` line yourself, as the `p4n4` user.

Take the report **before** the TTL or [section 7](#7-delete-the-emulator) ends the run. A deleted VM's disk is gone.

### 6.2 Read it

```bash
tar xzf $RUN.tar.gz
cat p4n4-emu-report-*/summary.md          # or open it in an editor that renders Markdown
jq '.issues' p4n4-emu-report-*/report.json
```

`gcloud storage ls gs://$BUCKET/reports/` lists everyone's past runs. Comparing two runs' `report.json` shows a regression, for example an image update that raised a service's peak memory under the same profile:

```bash
jq -r '.stacks[].services[] | [.service, .usage_over_time.memory.peak] | @tsv' */report.json
```

### 6.3 Cost report

Labels reach the billing data on their own; unlike AWS's cost allocation tags, there's nothing to activate. Billing data lags by a few hours to a day.

**In the console.** Billing → **Reports**: set *Projects* to the emulator project, then group by the label `p4n4-run`, or filter on it. Each run's VM and boot disk show up under its label.

**From the CLI**, once the [BigQuery export](#25-guard-the-bill) has data. The export's table is named after the billing account ID, with underscores:

```bash
TABLE=$PROJECT.billing_export.gcp_billing_export_v1_$(echo $BILLING | tr - _)
bq query --use_legacy_sql=false "
SELECT
  (SELECT value FROM UNNEST(labels) WHERE key = 'p4n4-run') AS run,
  ROUND(SUM(cost), 2) AS cost,
  ROUND(SUM(cost) + SUM(IFNULL((SELECT SUM(c.amount) FROM UNNEST(credits) c), 0)), 2) AS after_credits,
  ANY_VALUE(currency) AS currency
FROM \`$TABLE\`
WHERE project.id = '$PROJECT'
  AND usage_start_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  AND EXISTS (SELECT 1 FROM UNNEST(labels) WHERE key = 'p4n4-run')
GROUP BY run ORDER BY run"
```

`cost` is the list price. `after_credits` subtracts the trial credits and discounts such as sustained use, and is what you pay. The bucket isn't labeled per run. Its cost, a few cents a month for reports, is in the project's unlabeled total.

**Right away.** Before the billing data arrives, estimate from the run time:

```bash
gcloud compute instances describe $VM --format 'value(machineType.basename(), creationTimestamp)'
```

Multiply the hours since `creationTimestamp` by the machine type's price on [VM instance pricing](https://cloud.google.com/compute/vm-instance-pricing). The disk adds 40 GB × the disk type's price per GB-month, prorated, which is a few cents a day.

---

## 7. Delete the emulator

### 7.1 Delete the VM

```bash
gcloud compute instances delete $VM --quiet
```

You don't need `p4n4-emu down` first. Deleting the VM deletes its boot disk (boot disks are created with auto-delete on), and the containers, volumes and overlays go with it. The report in `gs://$BUCKET/reports/` stays until the lifecycle rule expires it.

### 7.2 Check that nothing is left

Anything still running costs money. Check after each run, and from time to time for everyone's runs:

```bash
# VMs (any run, anyone's)
gcloud compute instances list --filter 'labels.app=p4n4' \
  --format 'table(name, zone.basename(), machineType.basename(), status, labels.p4n4-run, creationTimestamp)'

# Disks not attached to any VM (should be empty), labeled or not
gcloud compute disks list --filter '-users:*' --format 'table(name, zone.basename(), sizeGb, labels.p4n4-run)'
```

A stopped VM (`TERMINATED` in Compute Engine's terms) costs nothing for its CPU and memory, but its disk is still billed. Delete it the same way.

### 7.3 Remove the account setup (when you're done with Google Cloud)

Everything from [section 3](#3-shared-resources-once) and all the VMs live in the emulator project, so deleting the project removes them all:

```bash
gcloud projects delete $PROJECT                # every VM, disk, the bucket and its reports, the dataset
gcloud billing budgets list --billing-account $BILLING    # then: gcloud billing budgets delete <id> --billing-account $BILLING
```

A deleted project can be restored for 30 days (`gcloud projects undelete`), and stops being billed at once. To close the billing account itself: Billing → Account management → **Close billing account**.

---

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `Quota 'C4A_CPUS' exceeded` (or another `*_CPUS` / `*_GPUS`) | The quota is lower than the running VMs plus this one. Delete leftovers ([7.2](#72-check-that-nothing-is-left)) or request an increase ([2.7](#27-check-the-cpu-and-gpu-quotas)) |
| `ZONE_RESOURCE_POOL_EXHAUSTED` | The zone has no capacity for the type right now. Set another zone of the region (`gcloud config set compute/zone …`) and create the VM again |
| `Invalid value for field 'resource.machineType'` | The type isn't offered in the zone. See the `machine-types list` check in [4.1](#41-pick-a-machine-for-the-profile) |
| `… disk type … is not supported` | The machine series takes only Hyperdisk, or the reverse. Use the boot disk type from the table in [4.1](#41-pick-a-machine-for-the-profile) |
| GPU refused, quota increase refused | The billing account is still a free trial. Upgrade it ([2.1](#21-sign-up-and-leave-the-free-trial)) |
| IAP error 4003 `failed to connect to backend` | sshd isn't up yet (wait a minute), or the VM lacks the `p4n4-emu` tag, so the firewall rule doesn't match it |
| IAP error 4033 `not authorized` | You lack `iap.tunnelResourceAccessor` ([2.6](#26-give-teammates-access)) |
| `Permission denied (publickey)` | You lack `compute.osAdminLogin`, or the VM was created without `enable-oslogin=TRUE` |
| `p4n4-emu: command not found` | cloud-init hasn't finished or failed. Run `cloud-init status --wait`, then `sudo cat /var/log/cloud-init-output.log`. Use `sudo -iu p4n4`, not your own user |
| `up` says no limit (no block device) | p4n4-emu didn't find the disk under Docker's data root, so disk I/O isn't throttled. CPU and memory limits still apply. Check `findmnt -no SOURCE /var/lib/docker` |
| The VM vanished mid-run | `--max-run-duration` ran out. Raise `TTL` for long soaks, or stop and start the VM to restart the clock ([4.3](#43-create-the-vm)) |
| `report` says no containers found | It ran outside the project directory, or after the stack stopped. `cd` to the project, or pass `--stack-dir` |
| The cost query finds no table | The billing export isn't on, or hasn't written yet (allow a day). Check the dataset with `bq ls $PROJECT:billing_export` |

## Checklist per run

- [ ] `gcloud config configurations activate p4n4`, variables set ([4](#4-create-an-emulator)), `TTL` long enough for the test
- [ ] Machine type and boot disk type from the table in [4.1](#41-pick-a-machine-for-the-profile), with `ARCH` to match
- [ ] Boot disk labeled (`gcloud compute disks add-labels`)
- [ ] On the VM: `cloud-init status` done, `p4n4-emu setup --check-only` OK, `p4n4-emu calibrate` run
- [ ] `up` shows `native`, a block device and the calibrated CPU factor
- [ ] Report copied back and in `gs://<bucket>/reports/<run>.tar.gz` **before** deleting
- [ ] VM deleted, [7.2](#72-check-that-nothing-is-left) shows nothing running
- [ ] Next day: the run's cost in Billing → Reports or from the BigQuery query ([6.3](#63-cost-report))
