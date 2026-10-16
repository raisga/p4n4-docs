# Emulators on Azure

This guide sets up an Azure account to run p4n4 projects on throwaway cloud emulators. An emulator is an Azure VM that runs [`p4n4-emu`](../reference/emulator.md), so a project gets the same per-board limits it gets on a workstation. Each run has four stages:

1. **Create the emulator**: a VM sized for the profile, in a resource group of its own.
2. **Run the project** on it under `p4n4-emu up`, with simulated sensors.
3. **Get the report**: `p4n4-emu report` bundles health, usage, restarts and logs, and Cost Management shows what the run cost.
4. **Delete the emulator**: delete the run's resource group and check that nothing is left running.

[Sections 1–3](#1-before-you-start) are done once per account. [Sections 4–7](#4-create-an-emulator) are repeated for every run. To create the shared resources (section 3) and the emulators (section 4) with Terraform instead, see [p4n4-emu's `terraform/`](https://github.com/raisga/p4n4-emu/tree/main/terraform). [AWS](aws.md), [Google Cloud](gcp.md) and [DigitalOcean](digitalocean.md) have the same guide.

```
 workstation (az CLI, Entra ID sign-in)         Azure subscription
 ──────────────────────────────────────         ─────────────────────────────────────────────
 az group create p4n4-emu-<run> ────────────►  resource group p4n4-emu-<run>
 az vm create ──────────────────────────────►    VM (Cobalt 100 arm64 / x86), disk, NIC, IP, NSG
                                                 cloud-init: Docker, uv, p4n4-emu, TTL timer
 ssh p4n4@<ip>  (NSG: port 22 from your IP) ◄─►  p4n4-emu up --profile rpi5 --sim
                                                 p4n4-emu report ─► /tmp/<run>.tar.gz
 scp ◄──────────────────────────────────────────────────────────────┘
 az storage blob upload ─► p4n4-emu-shared / reports
 az rest … CostManagement/query (by resource group)
 az group delete p4n4-emu-<run> ────────────►  everything in the run deleted
                                               (or by the VM itself, when its TTL timer fires)
```

**Why it's set up this way:**

- Arm VMs (Azure Cobalt 100, the `p` sizes) run the Raspberry Pi profiles' arm64 images natively, without the 3–10× slowdown of QEMU on an x86 workstation.
- Each run lives in a **resource group of its own**. In Azure, deleting a VM leaves its disk, network interface, public IP and network security group (NSG) behind, and they keep costing money. Deleting the resource group deletes everything at once.
- **Azure has no maximum run time for a VM**, and a VM whose OS shuts itself down stays allocated and billed. So each VM gets a managed identity that can only delete its own resource group, and a timer that does so when the run's TTL is over.
- SSH is open only to your own IP address, through the run's NSG, with key authentication only.

To deploy a project for real use rather than test it, see [Deploying a template](deployment.md). Its BYOH steps apply to Azure VMs as well.

---

## 1. Before you start

| On | Needs |
|---|---|
| Workstation | The [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli), an SSH key pair (`ssh-keygen -t ed25519`), `jq` |
| Azure | A Microsoft account and a payment card. Use a shared address for the account owner (`azure@example.com`), not one person's. An authenticator app or security key for MFA |
| Project | A p4n4 project, or a [template](../reference/template-registry.md) to copy. This guide uses `mqtt-influx-grafana`, as [Deploying a template](deployment.md) does |

The VM installs p4n4-emu from the `main` branch on GitHub. [Section 6](#6-get-the-report) needs `p4n4-emu report`, which is on `main` once the commit that adds it is pushed.

---

## 2. Create and secure the account (once)

### 2.1 Sign up and upgrade to pay-as-you-go

1. Go to <https://azure.microsoft.com/free> and create the account. You get a **subscription**, the unit that is billed and that holds the resources, and a **Microsoft Entra ID tenant**, the directory that holds the users.
2. **Upgrade to pay-as-you-go** (portal → Subscriptions → the subscription → **Upgrade**). Free accounts start with credits and a low vCPU quota, reportedly 4 per region, which is just one emulator. They also can't run GPU sizes. After the upgrade, the remaining credit still applies until it expires. The current terms are on the [Azure free account FAQ](https://azure.microsoft.com/free/free-account-faq/).
3. Pick a **region** close to you that has Cobalt 100 sizes, for example `eastus`, `westus2`, `westeurope` or `northeurope`. [4.1](#41-pick-a-size-for-the-profile) shows how to check.

Use this subscription for the emulators only. Teammates get Contributor on it ([2.5](#25-give-teammates-access)), and the cost reports can then simply cover the whole subscription.

### 2.2 Secure the owner account

1. **MFA.** New tenants have *security defaults* turned on, and these require MFA for every user. Azure also requires MFA for every sign-in to the portal and CLI. Register a security key or an authenticator app for the owner (mysignins.microsoft.com → Security info).
2. **A second owner.** Give a second trusted person **Owner** on the subscription ([2.5](#25-give-teammates-access)), so losing one account doesn't lock you out.
3. Don't create client secrets or service principal passwords for this setup. People sign in with their own Entra ID accounts, and the VMs use managed identities.

### 2.3 Configure the CLI

```bash
az login                                       # opens a browser; MFA
az account list --output table                 # note the subscription ID
az account set --subscription <subscription-id>
SUB=$(az account show --query id -o tsv)
LOCATION=eastus

for ns in Microsoft.Compute Microsoft.Network Microsoft.Storage Microsoft.CostManagement; do
  az provider register --namespace $ns
done
```

New subscriptions don't have every resource provider registered, and the first `az vm create` would fail without `Microsoft.Compute`.

### 2.4 Guard the bill

**A budget**, with emails at 80 % of actual spend and 100 % of forecast spend. `az consumption budget` can't set notifications, so the budget is created through the REST API:

```bash
az rest --method put \
  --url "https://management.azure.com/subscriptions/$SUB/providers/Microsoft.Consumption/budgets/p4n4-emu-monthly?api-version=2023-05-01" \
  --body "{\"properties\": {\"category\": \"Cost\", \"amount\": 50, \"timeGrain\": \"Monthly\",
    \"timePeriod\": {\"startDate\": \"$(date +%Y-%m-01)T00:00:00Z\"},
    \"notifications\": {
      \"actual80\":    {\"enabled\": true, \"operator\": \"GreaterThan\", \"threshold\": 80,  \"thresholdType\": \"Actual\",     \"contactEmails\": [\"azure@example.com\"]},
      \"forecast100\": {\"enabled\": true, \"operator\": \"GreaterThan\", \"threshold\": 100, \"thresholdType\": \"Forecasted\", \"contactEmails\": [\"azure@example.com\"]}}}}"
```

A budget only alerts. It doesn't stop anything. The TTL in [4.2](#42-bootstrap-file) is what does.

### 2.5 Give teammates access

Teammates need an account in the tenant. Either invite their existing work or Microsoft account (portal → **Microsoft Entra ID → Users → New user → Invite external user**), or create one there with **Create new user**. Then grant the roles. Each person gets:

| Role | Scope | For |
|---|---|---|
| Contributor | The subscription | Create and delete resource groups, VMs, disks and networks |
| Role Based Access Control Administrator, **with a condition** | The subscription | Give a new VM's identity the `p4n4 Emulator Reaper` role ([3.2](#32-reaper-role-for-the-ttl)), and nothing else |
| Storage Blob Data Contributor | The storage account ([3.1](#31-reports-storage)) | Upload and read reports. Contributor alone doesn't grant access to blob data |

Contributor can't assign roles. The condition on Role Based Access Control Administrator lets the holder assign exactly one role, and only to managed identities and other service principals. Without it, a teammate could make themselves Owner. Run this after [3.2](#32-reaper-role-for-the-ttl), which creates the role:

```bash
MEMBER=$(az ad user show --id teammate@example.com --query id -o tsv)   # an invited user: their UPN in the tenant
REAPER=$(az role definition list --name "p4n4 Emulator Reaper" --query '[0].name' -o tsv)

az role assignment create --assignee-object-id $MEMBER --assignee-principal-type User \
  --role Contributor --scope /subscriptions/$SUB
az role assignment create --assignee-object-id $MEMBER --assignee-principal-type User \
  --role "Role Based Access Control Administrator" --scope /subscriptions/$SUB \
  --condition-version 2.0 --condition "((!(ActionMatches{'Microsoft.Authorization/roleAssignments/write'})) OR (@Request[Microsoft.Authorization/roleAssignments:RoleDefinitionId] ForAnyOfAnyValues:GuidEquals {$REAPER} AND @Request[Microsoft.Authorization/roleAssignments:PrincipalType] ForAnyOfAnyValues:StringEqualsIgnoreCase {'ServicePrincipal'})) AND ((!(ActionMatches{'Microsoft.Authorization/roleAssignments/delete'})) OR (@Resource[Microsoft.Authorization/roleAssignments:RoleDefinitionId] ForAnyOfAnyValues:GuidEquals {$REAPER} AND @Resource[Microsoft.Authorization/roleAssignments:PrincipalType] ForAnyOfAnyValues:StringEqualsIgnoreCase {'ServicePrincipal'}))"
```

The condition is Microsoft's [constrain roles and principal types](https://learn.microsoft.com/azure/role-based-access-control/delegate-role-assignments-examples) example, with the reaper role in place of AcrPull. The subscription's owners have all three permissions already.

### 2.6 Check the vCPU quotas

```bash
az vm list-usage --location $LOCATION --output table \
  | grep -E 'Total Regional vCPUs|DPSv6|DPLSv6|ESv5|BASv2|NCASv3_T4'
```

An emulator needs 4 vCPUs of its size family (*Standard DPSv6 Family vCPUs* for `D4ps_v6`, and so on), within the regional total, and 8 for the GPU profile. GPU families usually start at 0. Request increases in the portal under **Quotas → Compute**. Increases for the common families are often approved within minutes.

---

## 3. Shared resources (once)

These are created once, by an owner, in their own resource group:

| Resource | Name | For |
|---|---|---|
| Resource group | `p4n4-emu-shared` | Things that outlive the runs |
| Storage account and container | `p4n4emu<random>` / `reports` | The team's archive of reports |
| Custom role | `p4n4 Emulator Reaper` | Lets a VM delete its own run's resource group, and nothing else |

```bash
az group create --name p4n4-emu-shared --location $LOCATION --tags app=p4n4
```

### 3.1 Reports storage

```bash
SA=p4n4emu$(openssl rand -hex 4)               # 3–24 lowercase letters and digits, unique across Azure
az storage account create --name $SA --resource-group p4n4-emu-shared --location $LOCATION \
  --sku Standard_LRS --kind StorageV2 --min-tls-version TLS1_2 \
  --allow-blob-public-access false --allow-shared-key-access false --tags app=p4n4
SA_ID=$(az storage account show --name $SA --query id -o tsv)

# Yourself first (owners don't get blob data access by default), then each teammate
az role assignment create --assignee $(az ad signed-in-user show --query id -o tsv) \
  --role "Storage Blob Data Contributor" --scope $SA_ID
az storage container create --account-name $SA --name reports --auth-mode login

# Delete reports after 180 days
az storage account management-policy create --account-name $SA --resource-group p4n4-emu-shared --policy '{
  "rules": [{"enabled": true, "name": "expire-reports", "type": "Lifecycle", "definition": {
    "filters": {"blobTypes": ["blockBlob"], "prefixMatch": ["reports/"]},
    "actions": {"baseBlob": {"delete": {"daysAfterModificationGreaterThan": 180}}}}}]}'
```

With shared-key access off, everyone uses their Entra ID sign-in (`--auth-mode login`), and no account keys exist to leak. A new role assignment can take a few minutes to apply.

### 3.2 Reaper role for the TTL

```bash
az role definition create --role-definition "{
  \"Name\": \"p4n4 Emulator Reaper\",
  \"Description\": \"Read and delete the resources in its scope: a p4n4 emulator's own resource group.\",
  \"Actions\": [\"*/read\", \"*/delete\"],
  \"AssignableScopes\": [\"/subscriptions/$SUB\"]
}"
```

Each run assigns this role to its VM's managed identity, scoped to that run's resource group ([4.3](#43-create-the-vm)). Deleting a resource group needs delete permission on everything in it, which `*/delete` gives, and nothing more. Code on the VM can reach the identity's token, and so can the containers, through the instance metadata service. The worst that code can do with it is delete the run early. It can't create resources or touch other resource groups.

---

## 4. Create an emulator

These variables are used in every step of a run. Keep them in one terminal, or save them to a file and `source` it:

```bash
SUB=$(az account show --query id -o tsv)
LOCATION=eastus
SA=<storage account from 3.1>

PROFILE=rpi5                       # the p4n4-emu profile
SIZE=Standard_D4ps_v6              # from the table below
IMAGE=Canonical:ubuntu-24_04-lts:server-arm64:latest   # …:server:latest for x86 sizes
DISK_GB=128                        # from the table below
TTL_MIN=480                        # the VM deletes its own resource group after 8 h
RUN=$PROFILE-$(date +%Y%m%d-%H%M)  # names and tags this run, e.g. rpi5-20261015-1430
RG=p4n4-emu-$RUN
VM=emu-$RUN
MY_IP=$(curl -4 -s https://ifconfig.me)
```

### 4.1 Pick a size for the profile

The VM must be **larger** than the board. p4n4-emu caps the containers at the profile's cores and memory, and the OS and Docker need room outside that cap. Use a size with at least as many vCPUs as the board has cores, so `--cpuset auto` can pin each service to a core. Give it about 1 GB more memory than the profile.

| Profile | Board | Size | vCPU / memory | OS disk | Notes |
|---|---|---|---|---|---|
| `rpi-zero2w`, `rpi3` | 4 cores, 384 / 768 MB | `Standard_D4pls_v6` | 4 / 8 GiB | 64 GB | Calibration scales the CPU limits down to a fraction of a Cobalt core each |
| `rpi4` | 4 cores, 3.5 GB | `Standard_D4pls_v6` | 4 / 8 GiB | 64 GB | |
| `rpi5` | 4 cores, 7 GB | `Standard_D4ps_v6` | 4 / 16 GiB | 128 GB | |
| `jetson-orin-nano` | 6 cores, 7 GB, GPU | `Standard_NC8as_T4_v3` | 8 / 56 GiB, NVIDIA T4 | 128 GB | x86 with `--native`. See [4.4](#44-gpu-profile-jetson-orin-nano) |
| `mcu-class` | 1 core, 256 MB, x86 | `Standard_B2als_v2` | 2 / 4 GiB | 64 GB | x86 image |
| `nuc` | 4 cores, 14 GB, x86 | `Standard_E4s_v5` | 4 / 32 GiB | 1024 GB | x86 image; the disk size is for throughput, see below |

**Disk.** Premium SSD's throughput depends on its size: 64 GB (P6) gives 50 MB/s, 128 GB (P10) 100 MB/s, 512 GB (P20) 150 MB/s and 1 TB (P30) 200 MB/s. Small disks can burst above that for a while. The table sizes each disk to at least the profile's disk limit: 50 MB/s for `rpi4`, 100 MB/s for `rpi5` and 200 MB/s for `nuc`. A smaller disk works too, but then the disk, not p4n4-emu, sets the speed limit. A 1 TB disk costs noticeably more per hour, so for `nuc` runs where disk speed doesn't matter, use 128 GB. The VM size also has a disk throughput ceiling, which is listed on its size page.

**Cheaper for quick checks.** B-series sizes (`Standard_B4ps_v2`, arm64, 4 / 16 GiB) cost less, but they're burstable and drop to a fraction of a core under sustained load. Use them for smoke tests and D or E sizes for soak tests and benchmarks.

Check the size is offered to your subscription in the region:

```bash
az vm list-skus --location $LOCATION --size $SIZE --resource-type virtualMachines \
  --query '[].{size:name, zones:locationInfo[0].zones|join(`,`,@), restrictions:restrictions[].reasonCode|join(`,`,@)}' -o table
```

An empty `restrictions` column means you can use it. `NotAvailableForSubscription` means you can't, in that region.

### 4.2 Bootstrap file

cloud-init runs this file on the VM's first boot. It installs Docker, `uv` and p4n4-emu for the admin user `p4n4`. It also sets up the TTL: a systemd timer that, `TTL_MIN` minutes after first boot, deletes the VM's resource group through the VM's managed identity. The timer is persistent, so it still fires after a reboot.

```bash
cat > emu-cloud-init.yaml <<'EOF'
#cloud-config
package_update: true
packages: [ca-certificates, curl, git, jq]

write_files:
  - path: /etc/docker/daemon.json
    content: |
      { "log-driver": "json-file", "log-opts": { "max-size": "10m", "max-file": "5" } }
  - path: /usr/local/bin/p4n4-ttl
    permissions: "0755"
    content: |
      #!/bin/sh
      # The run's time to live: delete this VM's resource group, and everything in it
      set -eu
      imds=http://169.254.169.254/metadata
      token=$(curl -sf -H Metadata:true \
        "$imds/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/" | jq -r .access_token)
      rg=$(curl -sf -H Metadata:true "$imds/instance/compute?api-version=2021-02-01" \
        | jq -r '"/subscriptions/\(.subscriptionId)/resourceGroups/\(.resourceGroupName)"')
      curl -sf -X DELETE -H "Authorization: Bearer $token" "https://management.azure.com$rg?api-version=2021-04-01"
  - path: /etc/systemd/system/p4n4-ttl.service
    content: |
      [Unit]
      Description=p4n4 emulator TTL: delete this VM's resource group
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
  - curl -fsSL https://get.docker.com | sh
  - usermod -aG docker p4n4
  - su - p4n4 -c 'curl -LsSf https://astral.sh/uv/install.sh | sh'
  - su - p4n4 -c 'mkdir -p p4n4/tools && git clone -q https://github.com/raisga/p4n4-lib p4n4/lib && git clone -q https://github.com/raisga/p4n4-emu p4n4/tools/emu && git clone -q https://github.com/raisga/p4n4-templates p4n4/tools/templates'
  - su - p4n4 -c '~/.local/bin/uv tool install ~/p4n4/tools/emu --with ~/p4n4/lib'
EOF
sed -i "s/__TTL_MIN__/$TTL_MIN/" emu-cloud-init.yaml
```

- When the timer fires, the VM, its disk and everything else in the run's resource group are deleted. Upload the report before then ([section 6](#6-get-the-report)).
- To give a running emulator more time, move the timer on the VM, for example to 4 hours from now:

  ```bash
  sudo sed -i "s/^OnCalendar=.*/OnCalendar=$(date -u -d '+4 hours' '+%Y-%m-%d %H:%M:%S UTC')/" /etc/systemd/system/p4n4-ttl.timer
  sudo systemctl daemon-reload && sudo systemctl restart p4n4-ttl.timer
  systemctl list-timers p4n4-ttl.timer      # shows when it fires
  ```

  `sudo systemctl disable --now p4n4-ttl.timer` cancels the TTL. Then only [section 7](#7-delete-the-emulator) deletes the run.
- `--with ~/p4n4/lib` installs p4n4-lib from the checkout. p4n4-emu needs a p4n4-lib version that isn't on PyPI yet.
- To test a specific release rather than `main`, add `--branch v<version>` to the `git clone` commands.

### 4.3 Create the VM

```bash
az group create --name $RG --location $LOCATION \
  --tags app=p4n4 p4n4-run=$RUN p4n4-profile=$PROFILE

az vm create --resource-group $RG --name $VM \
  --size $SIZE --image $IMAGE \
  --os-disk-size-gb $DISK_GB --storage-sku Premium_LRS \
  --admin-username p4n4 --ssh-key-values ~/.ssh/id_ed25519.pub \
  --public-ip-sku Standard --nsg-rule NONE \
  --assign-identity '[system]' --role "p4n4 Emulator Reaper" --scope /subscriptions/$SUB/resourceGroups/$RG \
  --custom-data emu-cloud-init.yaml \
  --tags app=p4n4 p4n4-run=$RUN p4n4-profile=$PROFILE

az network nsg rule create --resource-group $RG --nsg-name ${VM}NSG --name ssh-from-me \
  --priority 100 --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefixes $MY_IP/32 --destination-port-ranges 22

IP=$(az vm show --resource-group $RG --name $VM --show-details --query publicIps -o tsv)
echo "$RUN $IP"
```

- `--nsg-rule NONE` stops `az vm create` from adding its default rule, which opens SSH to the whole internet. The next command opens port 22 to your address only. If your IP changes, update the rule with `az network nsg rule update … --source-address-prefixes <new-ip>/32`.
- `--assign-identity` with `--role` and `--scope` gives the VM's identity the reaper role on its own resource group only. Run by a teammate, this is the one assignment their conditional role ([2.5](#25-give-teammates-access)) allows.
- The admin user `p4n4` is the VM's first user, uid 1000, which is what the template's `ARCHIVE_UID` / `ARCHIVE_GID` default to.
- The tags on the resource group identify the run in the cleanup checks ([7.2](#72-check-that-nothing-is-left)). The cost report ([6.3](#63-cost-report)) groups by resource group name.

### 4.4 GPU profile (`jetson-orin-nano`)

The `jetson-orin-nano` profile reserves an NVIDIA GPU for `ollama` and `ei-runner`. Azure's Cobalt sizes have no GPUs, so this profile runs on x86 with a T4 GPU and host-architecture images (`--native`). Set `SIZE=Standard_NC8as_T4_v3` and `IMAGE=Canonical:ubuntu-24_04-lts:server:latest`.

After cloud-init finishes, install the driver and the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) on the VM:

```bash
sudo apt-get install -y ubuntu-drivers-common && sudo ubuntu-drivers install --gpgpu
# add NVIDIA's apt repository as the toolkit guide shows, then:
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime docker && sudo reboot
```

After the reboot, `docker info | grep -i runtimes` should list `nvidia`. Start the stack with `p4n4-emu up --profile jetson-orin-nano --native`. The T4 family needs quota ([2.6](#26-check-the-vcpu-quotas)), which a free account doesn't have.

---

## 5. Run a project on it

### 5.1 Connect and check

```bash
ssh p4n4@$IP
```

```bash
# On the VM
cloud-init status --wait                  # "status: done"; if not, see /var/log/cloud-init-output.log
systemctl list-timers p4n4-ttl.timer      # the TTL: when the run deletes itself
p4n4-emu setup --check-only               # Docker, Compose and cgroup v2 OK
p4n4-emu calibrate                        # benchmarks one Cobalt core against the boards' cores
p4n4-emu calibrate --show                 # the CPU factor each profile gets on this VM
```

If `p4n4-emu` or `docker` isn't found yet, cloud-init is still running. The session's groups were also set before `usermod` added `p4n4` to `docker`, so after `cloud-init status --wait`, log out and back in.

Calibration matters here. A Cobalt 100 core does several times the work of a Pi core, so without calibration a 4-core `rpi5` would get four fast cores. With it, `up` scales each CPU limit by the Pi core's speed relative to the VM's ([core speed](https://github.com/raisga/p4n4-emu#hardware-profiles)).

### 5.2 Put the project on it

**From a template.** On the VM:

```bash
cp -r ~/p4n4/tools/templates/projects/mqtt-influx-grafana ~/greenhouse
cd ~/greenhouse
cp .env.example .env        # ARCHIVE_UID/GID are already 1000, the p4n4 user
```

**Your own project.** From your workstation, copy it over, leaving out local data. `.env` can go along, because the VM is thrown away. Don't ship production secrets this way, though.

```bash
rsync -az --exclude 'data/archive/*/' ~/projects/greenhouse/ p4n4@$IP:~/greenhouse/
```

For a multi-layer project (iot + ai, …), ship the whole project directory. `p4n4-emu up` at its root starts every layer ([multi-layer templates](deployment.md#multi-layer-templates)).

### 5.3 Start it under the profile

```bash
p4n4-emu up --profile rpi5 --dry-run      # look at the overlay first
p4n4-emu up --profile rpi5 --sim --sim-devices 3 --cpuset auto
p4n4-emu status                           # usage against each service's limits
```

Check the summary that `up` prints:

- **Platform** says `native`. A Cobalt VM runs arm64 images without QEMU.
- **Block device detected** names the OS disk (`/dev/sda` on most sizes, `/dev/nvme0n1` on sizes with NVMe disks), so the disk limits apply. If it says no limit (no block device), the CPU and memory limits still apply, but disk I/O isn't throttled.
- **CPU limit** shows the calibrated factor, for example `1.60 host cores = 4 × 0.400`.

Leave the stack running as long as the test calls for, from minutes for a smoke test to hours for a soak test, staying within the TTL. The template's `./tests/smoke.sh` runs its own throwaway copy of the stack. Run it on the VM as well if you want its end-to-end check.

### 5.4 Open Grafana

The port forward runs in a second terminal on your workstation, over SSH. The NSG never opens port 3000:

```bash
ssh -N -L 3000:localhost:3000 p4n4@$IP
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

Run it from your workstation, then copy the archive back and into the team's storage:

```bash
ssh p4n4@$IP "bash -lc 'cd ~/greenhouse && p4n4-emu report --samples 12 --interval 5 \
  -l run=$RUN -l profile=$PROFILE -l size=$SIZE -l location=$LOCATION -o /tmp/$RUN.tar.gz'"

mkdir -p ~/p4n4-reports && cd ~/p4n4-reports
scp p4n4@$IP:/tmp/$RUN.tar.gz .
az storage blob upload --auth-mode login --account-name $SA --container-name reports \
  --name reports/$RUN.tar.gz --file $RUN.tar.gz
```

`bash -lc` runs a login shell, which puts `~/.local/bin`, and with it `p4n4-emu`, on the `PATH`. This report samples for about a minute (12 samples, 5 s apart). Sample longer to catch periodic load, such as `--samples 60 --interval 10`.

Take the report **before** the TTL or [section 7](#7-delete-the-emulator) ends the run. A deleted VM's disk is gone.

### 6.2 Read it

```bash
tar xzf $RUN.tar.gz
cat p4n4-emu-report-*/summary.md          # or open it in an editor that renders Markdown
jq '.issues' p4n4-emu-report-*/report.json
```

`az storage blob list --auth-mode login --account-name $SA -c reports --prefix reports/ -o table` lists everyone's past runs. Comparing two runs' `report.json` shows a regression, for example an image update that raised a service's peak memory under the same profile:

```bash
jq -r '.stacks[].services[] | [.service, .usage_over_time.memory.peak] | @tsv' */report.json
```

### 6.3 Cost report

Each run has a resource group of its own, so the cost per run is the cost per resource group. Azure keeps a deleted resource group's cost history under its name, so this works after the run is gone, with no tags to set up. Cost data lags by 8 to 24 hours.

**In the portal.** **Cost Management → Cost analysis**, scope: the subscription. Pick the *Resources* or *Resource groups* view, or group by *Resource group name*. Each `p4n4-emu-<run>` is one run.

**From the CLI**, through the Cost Management query API:

```bash
az rest --method post \
  --url "https://management.azure.com/subscriptions/$SUB/providers/Microsoft.CostManagement/query?api-version=2023-03-01" \
  --body "{\"type\": \"ActualCost\", \"timeframe\": \"Custom\",
    \"timePeriod\": {\"from\": \"$(date -d '-30 days' +%F)\", \"to\": \"$(date +%F)\"},
    \"dataset\": {\"granularity\": \"None\",
      \"aggregation\": {\"totalCost\": {\"name\": \"Cost\", \"function\": \"Sum\"}},
      \"grouping\": [{\"type\": \"Dimension\", \"name\": \"ResourceGroupName\"}]}}" \
  --query 'properties.rows' -o tsv | grep -i p4n4-emu
```

Each row is a run's cost over the last 30 days, its resource group and the currency (`0.42  p4n4-emu-rpi5-20261015-1430  USD`). The `p4n4-emu-shared` row is the storage account. `ActualCost` is what you're billed, before credits are applied. The portal's **Cost analysis** shows the credits.

**Right away.** Before Cost Management has the data, estimate from the run time:

```bash
az vm show --resource-group $RG --name $VM --query '[hardwareProfile.vmSize, timeCreated]' -o tsv
```

Multiply the hours since `timeCreated` by the size's price on [Linux VM pricing](https://azure.microsoft.com/pricing/details/virtual-machines/linux/), and add the disk's Premium SSD price and the Standard public IP's, prorated.

---

## 7. Delete the emulator

### 7.1 Delete the run's resource group

```bash
az group delete --name $RG --yes             # 2–5 minutes; add --no-wait to return at once
```

You don't need `p4n4-emu down` first. Deleting the resource group deletes the VM, its OS disk, NIC, public IP and NSG, and the VM's identity with its role assignment. The containers, volumes and overlays go with the disk. The report in the `reports` container stays until the lifecycle rule expires it.

### 7.2 Check that nothing is left

Anything still in a run's resource group costs money. Check after each run, and from time to time for everyone's runs:

```bash
# Run resource groups that still exist (any run, anyone's)
az group list --query "[?tags.\"p4n4-run\"].{group:name, run:tags.\"p4n4-run\", state:properties.provisioningState}" -o table

# VMs anywhere in the subscription, with their power state
az vm list --show-details --query '[].{vm:name, group:resourceGroup, size:hardwareProfile.vmSize, power:powerState}' -o table

# Disks and public IPs not attached to anything
az disk list --query "[?diskState=='Unattached'].{disk:name, group:resourceGroup, gb:diskSizeGb}" -o table
az network public-ip list --query "[?ipConfiguration==null].{ip:name, group:resourceGroup}" -o table
```

A group whose state is `Deleting` is on its way out. A VM shown as `VM stopped` is still allocated and **still billed**. Only `VM deallocated` stops the compute charges, and its disk is billed either way. Delete leftovers with `az group delete`. `NetworkWatcherRG` is created by Azure for every region with a virtual network. It's free, and you can leave it.

### 7.3 Remove the account setup (when you're done with Azure)

```bash
az group delete --name p4n4-emu-shared --yes      # the storage account and every report
az role definition delete --name "p4n4 Emulator Reaper"
az rest --method delete \
  --url "https://management.azure.com/subscriptions/$SUB/providers/Microsoft.Consumption/budgets/p4n4-emu-monthly?api-version=2023-05-01"
```

Delete the role only once no run's resource group is left, because their role assignments use it. To stop all billing, cancel the subscription: portal → **Subscriptions** → the subscription → **Cancel subscription**.

---

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `QuotaExceeded` / `OperationNotAllowed … vCPUs` | The family or regional vCPU quota is lower than the running VMs plus this one. Delete leftovers ([7.2](#72-check-that-nothing-is-left)) or request an increase ([2.6](#26-check-the-vcpu-quotas)) |
| `SkuNotAvailable` | The size isn't offered in the region, or to this subscription there. See the `list-skus` check in [4.1](#41-pick-a-size-for-the-profile), and try another region or size |
| `AllocationFailed` / `ZonalAllocationFailed` | The region has no capacity for the size right now. Try again later, another size from the same family, or another region |
| `The selected VM size … cannot boot Hypervisor Generation '1'` or a security type error | The image or security type doesn't match the size. Arm sizes need the `server-arm64` image. If Trusted Launch is the problem, add `--security-type Standard` |
| `MissingSubscriptionRegistration` | A resource provider isn't registered. Run the `az provider register` loop from [2.3](#23-configure-the-cli) |
| `AuthorizationFailed … roleAssignments/write` on `vm create` | The teammate lacks the conditional Role Based Access Control Administrator assignment ([2.5](#25-give-teammates-access)), or `--role` named a role other than the reaper |
| `ssh: connect … timed out` | The NSG rule doesn't match your current IP (`curl -4 ifconfig.me`), or you're behind a network that blocks outbound port 22 |
| `p4n4-emu: command not found` | cloud-init hasn't finished or failed. Run `cloud-init status --wait`, then `sudo cat /var/log/cloud-init-output.log` |
| `permission denied … docker.sock` | The session started before `usermod -aG docker p4n4` ran. Log out and back in |
| `up` says no limit (no block device) | p4n4-emu didn't find the disk under Docker's data root, so disk I/O isn't throttled. CPU and memory limits still apply. Check `findmnt -no SOURCE /var/lib/docker` |
| The run vanished mid-test | The TTL timer fired. Raise `TTL_MIN` for long soaks, or move the timer on the VM ([4.2](#42-bootstrap-file)) |
| The TTL didn't delete the run | Run `sudo systemctl status p4n4-ttl.service` and `sudo /usr/local/bin/p4n4-ttl` on the VM. Usually the role assignment is missing (`az role assignment list --scope /subscriptions/$SUB/resourceGroups/$RG -o table`). Delete the run by hand ([7.1](#71-delete-the-runs-resource-group)) |
| `report` says no containers found | It ran outside the project directory, or after the stack stopped. `cd` to the project, or pass `--stack-dir` |
| The cost query fails on the `Cost` column | Older pay-as-you-go subscriptions name it `PreTaxCost`. Use that in `aggregation` |
| `az storage blob upload` → `AuthorizationPermissionMismatch` | You lack Storage Blob Data Contributor on the storage account, or the assignment is a few minutes old ([3.1](#31-reports-storage)) |

## Checklist per run

- [ ] `az login`, variables set ([4](#4-create-an-emulator)), `TTL_MIN` long enough for the test
- [ ] Size, image and disk size from the table in [4.1](#41-pick-a-size-for-the-profile)
- [ ] NSG rule allows port 22 from your current IP only
- [ ] On the VM: `cloud-init status` done, the TTL timer listed, `p4n4-emu setup --check-only` OK, `p4n4-emu calibrate` run
- [ ] `up` shows `native`, a block device and the calibrated CPU factor
- [ ] Report copied back and in the `reports` container **before** deleting
- [ ] Resource group deleted, [7.2](#72-check-that-nothing-is-left) shows nothing left
- [ ] Next day: the run's cost in Cost analysis or from the query ([6.3](#63-cost-report))
