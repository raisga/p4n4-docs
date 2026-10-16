# Emulators on AWS

This guide sets up an AWS account to run p4n4 projects on throwaway cloud emulators. An emulator is an EC2 instance that runs [`p4n4-emu`](../reference/emulator.md), so a project gets the same per-board limits it gets on a workstation. Each run has four stages:

1. **Create the emulator**: launch an instance sized for the profile.
2. **Run the project** on it under `p4n4-emu up`, with simulated sensors.
3. **Get the report**: `p4n4-emu report` bundles health, usage, restarts and logs, and Cost Explorer reports what the run cost.
4. **Delete the emulator**: terminate the instance and check that nothing is left running.

[Sections 1–3](#1-before-you-start) are done once per account. [Sections 4–7](#4-create-an-emulator) are repeated for every run. To create the shared resources (section 3) and the emulators (section 4) with Terraform instead, see [p4n4-emu's `terraform/`](https://github.com/raisga/p4n4-emu/tree/main/terraform). [Google Cloud](gcp.md), [Azure](azure.md) and [DigitalOcean](digitalocean.md) have the same guide.

```
 workstation (AWS CLI, SSO login)               AWS account
 ────────────────────────────────               ─────────────────────────────────────────────
 aws ec2 run-instances ─────────────────────►  EC2 instance (Graviton arm64 / x86)
                                                 cloud-init: Docker, uv, p4n4-emu
 aws ssm start-session ◄── no inbound ports ──►  p4n4-emu up --profile rpi5 --sim
 port forward :3000 (Grafana) ◄──────────────►   p4n4-emu report ─► S3 bucket (reports/)
 aws s3 cp s3://…/reports/<run>.tar.gz ◄──────────────────────────────┘
 aws ce get-cost-and-usage (tag p4n4-run)
 aws ec2 terminate-instances ───────────────►  instance + disk deleted (or self-terminates at its TTL)
```

**Why it's set up this way:**

- Graviton (arm64) instances run the Raspberry Pi profiles' arm64 images natively, without the 3–10× slowdown of QEMU on an x86 workstation.
- The instance never opens a port. You connect through AWS Systems Manager (SSM) Session Manager, and reports leave the instance through S3.
- Every instance terminates itself once its time to live (TTL) runs out, so an emulator you forget about stops costing money.

To deploy a project for real use rather than test it, see [Deploying a template](deployment.md). Its BYOH steps apply to EC2 as well.

---

## 1. Before you start

| On | Needs |
|---|---|
| Workstation | [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html), the [Session Manager plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html), `jq`, a phone or security key for MFA |
| AWS | A payment card, an email address for the root user (use a shared mailbox such as `aws@example.com`, not one person's) |
| Project | A p4n4 project, or a [template](../reference/template-registry.md) to copy. This guide uses `mqtt-influx-grafana`, as [Deploying a template](deployment.md) does |

The instance installs p4n4-emu from the `main` branch on GitHub. [Section 6](#6-get-the-report) needs `p4n4-emu report`, which is on `main` once the commit that adds it is pushed.

---

## 2. Create and secure the account (once)

### 2.1 Sign up

1. Go to <https://aws.amazon.com> → **Create an AWS account**. Enter the root email and an account name (`p4n4`).
2. **Choose the Paid plan.** New accounts choose between a Free plan and a Paid plan. Both include the sign-up credits. The Free plan blocks instances of size `xlarge` and larger, which the profiles in [4.1](#41-pick-an-instance-for-the-profile) need. It also closes the account when the credits or the plan period run out. You can upgrade a Free plan account later from the Billing console. The current terms are on the [AWS Free Tier page](https://aws.amazon.com/free/).
3. Choose **Basic support** (free).
4. Pick a **home region** close to you where Graviton `m7g`/`c7g` instances are available, for example `us-east-1`, `us-west-2` or `eu-central-1`. Use the same region for everything below.

### 2.2 Lock down the root user

The root user can do anything, including close the account. Use it only for the steps in this section.

1. Sign in as root → account menu → **Security credentials** → **Assign MFA device**. A passkey or hardware key is best; an authenticator app works too.
2. On the same page, check that the root user has **no access keys**. Never create any.
3. Store the root password and MFA recovery details somewhere the team can reach if the owner leaves, such as a shared password vault.

### 2.3 Guard the bill

Do these as root before launching anything:

1. **Billing access for IAM users.** Account menu → **Account** → *IAM user and role access to Billing information* → **Edit** → **Activate**. Without it, the Identity Center users you create in [2.4](#24-create-sign-in-users-iam-identity-center) can't see costs.
2. **Cost Explorer.** Open **Billing and Cost Management → Cost Explorer** once. Opening it turns it on, and the data takes up to 24 hours to appear.
3. **A budget.** **Billing and Cost Management → Budgets → Create budget → Monthly cost budget**. Set an amount, such as USD 50, and email alerts at 80 % of actual spend and 100 % of forecast spend. [3.1](#31-reports-bucket) shows the same budget as a CLI command.
4. **Free Tier alerts.** **Billing preferences → Alert preferences** → turn on *AWS Free Tier alerts*, which warn before the credits run out.

### 2.4 Create sign-in users (IAM Identity Center)

People sign in through IAM Identity Center with short-lived credentials. Nobody uses long-lived IAM user access keys.

1. Open **IAM Identity Center** in the home region → **Enable**. Accept enabling it with AWS Organizations. This account becomes the organization's management account, which is fine for a single account.
2. **Users → Add user** for yourself and for each teammate who will run emulators. Each user gets an invitation email and registers their own MFA.
3. **Permission sets → Create permission set**:
   - `AdministratorAccess` (predefined). Give it only to the one or two people who do [section 3](#3-shared-resources-once).
   - `P4n4Emulator` (custom). Attach the AWS managed policy **PowerUserAccess**, plus this inline policy, which lets users give instances the emulator role from [3.2](#32-instance-role). Replace `<account-id>`:

     ```json
     {
       "Version": "2012-10-17",
       "Statement": [{
         "Effect": "Allow",
         "Action": "iam:PassRole",
         "Resource": "arn:aws:iam::<account-id>:role/p4n4-emu-instance"
       }]
     }
     ```

     PowerUserAccess allows everything except IAM and Organizations. That covers EC2, SSM, S3 and Cost Explorer, but not creating roles or users.
4. **AWS accounts** → select the account → **Assign users or groups** → assign each user `P4n4Emulator`, and the admins `AdministratorAccess` as well.
5. Note the **AWS access portal URL** on the Identity Center dashboard (`https://d-xxxxxxxxxx.awsapps.com/start`).

### 2.5 Configure the CLI

On your workstation:

```bash
aws configure sso
#   SSO session name:        p4n4
#   SSO start URL:           https://d-xxxxxxxxxx.awsapps.com/start
#   SSO region:              us-east-1          (the home region)
#   SSO registration scopes: sso:account:access
#   → a browser window opens; approve, then pick the account and P4n4Emulator
#   CLI default region:      us-east-1
#   CLI profile name:        p4n4
```

Run it a second time with `AdministratorAccess` and the profile name `p4n4-admin` if you're an admin. Then:

```bash
aws sso login --sso-session p4n4          # once a day; it signs in both profiles
export AWS_PROFILE=p4n4
aws sts get-caller-identity               # shows the account and your P4n4Emulator role
session-manager-plugin --version          # the plugin that aws ssm start-session needs
```

### 2.6 Check the vCPU quotas

New accounts can start with low vCPU limits. An emulator needs 4 vCPUs, and 8 for `jetson-orin-nano`:

```bash
# Running On-Demand Standard (A, C, D, H, I, M, R, T, Z) instances, in vCPUs
aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A --query Quota.Value
# Running On-Demand G and VT instances (only for the GPU profile)
aws service-quotas list-service-quotas --service-code ec2 \
  --query "Quotas[?contains(QuotaName, 'G and VT')].[QuotaCode,Value]" --output text
```

If a quota is below what you plan to run at once, request more. Approval can take from minutes to a day or two:

```bash
aws service-quotas request-service-quota-increase --service-code ec2 \
  --quota-code L-1216C47A --desired-value 32
```

---

## 3. Shared resources (once)

These are created once, by an admin. Every emulator uses them:

| Resource | Name | For |
|---|---|---|
| S3 bucket | `p4n4-emu-<account-id>` | Reports (`reports/`) and projects shipped to an emulator (`projects/`) |
| IAM role and instance profile | `p4n4-emu-instance` | Lets an instance use SSM and the bucket, and nothing else |
| Security group | `p4n4-emu` | No inbound rules; all outbound traffic, for SSM, S3, GitHub and image pulls |

```bash
export AWS_PROFILE=p4n4-admin AWS_REGION=us-east-1
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
BUCKET=p4n4-emu-$ACCOUNT
```

### 3.1 Reports bucket

```bash
aws s3api create-bucket --bucket "$BUCKET"
# Outside us-east-1, add: --create-bucket-configuration LocationConstraint=$AWS_REGION
aws s3api put-public-access-block --bucket "$BUCKET" --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3api put-bucket-tagging --bucket "$BUCKET" --tagging 'TagSet=[{Key=Project,Value=p4n4}]'
# Shipped projects are deleted after 7 days, reports after 180
aws s3api put-bucket-lifecycle-configuration --bucket "$BUCKET" --lifecycle-configuration '{
  "Rules": [
    {"ID": "projects", "Status": "Enabled", "Filter": {"Prefix": "projects/"}, "Expiration": {"Days": 7}},
    {"ID": "reports",  "Status": "Enabled", "Filter": {"Prefix": "reports/"},  "Expiration": {"Days": 180}}
  ]}'
```

The budget from [2.3](#23-guard-the-bill), if you'd rather create it from the CLI:

```bash
aws budgets create-budget --account-id "$ACCOUNT" \
  --budget '{"BudgetName":"p4n4-monthly","BudgetLimit":{"Amount":"50","Unit":"USD"},"TimeUnit":"MONTHLY","BudgetType":"COST"}' \
  --notifications-with-subscribers '[
    {"Notification":{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80,"ThresholdType":"PERCENTAGE"},
     "Subscribers":[{"SubscriptionType":"EMAIL","Address":"aws@example.com"}]},
    {"Notification":{"NotificationType":"FORECASTED","ComparisonOperator":"GREATER_THAN","Threshold":100,"ThresholdType":"PERCENTAGE"},
     "Subscribers":[{"SubscriptionType":"EMAIL","Address":"aws@example.com"}]}]'
```

### 3.2 Instance role

```bash
aws iam create-role --role-name p4n4-emu-instance --assume-role-policy-document '{
  "Version": "2012-10-17",
  "Statement": [{"Effect": "Allow", "Principal": {"Service": "ec2.amazonaws.com"}, "Action": "sts:AssumeRole"}]
}'
aws iam attach-role-policy --role-name p4n4-emu-instance \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam put-role-policy --role-name p4n4-emu-instance --policy-name bucket --policy-document "{
  \"Version\": \"2012-10-17\",
  \"Statement\": [
    {\"Effect\": \"Allow\", \"Action\": [\"s3:GetObject\", \"s3:PutObject\"], \"Resource\": \"arn:aws:s3:::$BUCKET/*\"},
    {\"Effect\": \"Allow\", \"Action\": \"s3:ListBucket\", \"Resource\": \"arn:aws:s3:::$BUCKET\"}
  ]}"
aws iam create-instance-profile --instance-profile-name p4n4-emu-instance
aws iam add-role-to-instance-profile --instance-profile-name p4n4-emu-instance --role-name p4n4-emu-instance
```

`AmazonSSMManagedInstanceCore` lets the SSM agent register the instance, so `aws ssm start-session` and `send-command` reach it. Ubuntu's AMIs ship the agent.

### 3.3 Security group

Each region of a new account has a default VPC whose subnets give instances a public IP. The emulators use that VPC:

```bash
VPC=$(aws ec2 describe-vpcs --filters Name=is-default,Values=true --query 'Vpcs[0].VpcId' --output text)
# "None"? Create one: aws ec2 create-default-vpc
aws ec2 create-security-group --group-name p4n4-emu --vpc-id "$VPC" \
  --description "p4n4 emulators: no inbound, SSM only" \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Project,Value=p4n4}]'
```

A new security group has no inbound rules and allows all outbound traffic, which is what an emulator needs. Don't add a rule for port 22: you connect through Session Manager. To see Grafana you forward a port through the session ([5.4](#54-open-grafana)).

---

## 4. Create an emulator

From here on, use the `p4n4` (P4n4Emulator) profile. These variables are used in every step of a run. Keep them in one terminal, or save them to a file and `source` it:

```bash
export AWS_PROFILE=p4n4 AWS_REGION=us-east-1
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
BUCKET=p4n4-emu-$ACCOUNT
SG=$(aws ec2 describe-security-groups --group-names p4n4-emu --query 'SecurityGroups[0].GroupId' --output text)

PROFILE=rpi5                       # the p4n4-emu profile
TYPE=m7g.xlarge                    # from the table below
ARCH=arm64                         # arm64 for Graviton (*g) types, amd64 for x86
TTL_MIN=480                        # the instance terminates itself after 8 h
RUN=$PROFILE-$(date +%Y%m%d-%H%M)  # names and tags this run, e.g. rpi5-20261015-1430
```

### 4.1 Pick an instance for the profile

The instance must be **larger** than the board. p4n4-emu caps the containers at the profile's cores and memory, and the OS, Docker and the SSM agent need room outside that cap. Use an instance with at least as many vCPUs as the board has cores, so `--cpuset auto` can pin each service to a core. Give it about 1 GiB more memory than the profile.

| Profile | Board | Instance | vCPU / memory | Notes |
|---|---|---|---|---|
| `rpi-zero2w`, `rpi3` | 4 cores, 384 / 768 MB | `c7g.xlarge` | 4 / 8 GiB | Calibration scales the CPU limits down to about a quarter of a Graviton core each |
| `rpi4` | 4 cores, 3.5 GB | `c7g.xlarge` | 4 / 8 GiB | |
| `rpi5` | 4 cores, 7 GB | `m7g.xlarge` | 4 / 16 GiB | |
| `jetson-orin-nano` | 6 cores, 7 GB, GPU | `g5g.2xlarge` | 8 / 16 GiB, NVIDIA T4G | See [4.4](#44-gpu-profile-jetson-orin-nano) |
| `mcu-class` | 1 core, 256 MB, x86 | `t3.small` | 2 / 2 GiB | `ARCH=amd64` |
| `nuc` | 4 cores, 14 GB, x86 | `r7i.xlarge` | 4 / 32 GiB | `ARCH=amd64`; also set the disk's throughput to 250 MB/s, see below |

**Cheaper for quick checks.** `t4g.xlarge` (4 / 16 GiB, burstable) costs less than `m7g.xlarge`, but sustained load spends its CPU credits. In the default unlimited mode, the extra credits are billed, and in standard mode the CPU drops to its baseline. Neither is a faithful board, so use burstable types for smoke tests and `m7g`/`c7g` for soak tests and benchmarks.

**Disk.** The root disk is gp3, which gives 125 MB/s and 3,000 IOPS at no extra charge. That is above every profile's disk limit except `nuc`'s 200 MB/s. For `nuc`, add `"Throughput":250` to the `Ebs` block in [4.3](#43-launch-it).

Check that the type exists in your region before launching:

```bash
aws ec2 describe-instance-type-offerings --location-type availability-zone \
  --filters Name=instance-type,Values=$TYPE --query 'InstanceTypeOfferings[].Location' --output text
```

### 4.2 Bootstrap file

cloud-init runs this file on the instance's first boot. It schedules the TTL shutdown first, then installs Docker, the AWS CLI, `uv` and p4n4-emu:

```bash
cat > emu-cloud-init.yaml <<EOF
#cloud-config
package_update: true
packages: [ca-certificates, curl, git, jq]

write_files:
  - path: /etc/docker/daemon.json
    content: |
      { "log-driver": "json-file", "log-opts": { "max-size": "10m", "max-file": "5" } }

runcmd:
  - shutdown -P +$TTL_MIN "p4n4 emulator TTL reached"
  - curl -fsSL https://get.docker.com | sh
  - usermod -aG docker ubuntu
  - snap install aws-cli --classic
  - su - ubuntu -c 'curl -LsSf https://astral.sh/uv/install.sh | sh'
  - su - ubuntu -c 'mkdir -p p4n4/tools && git clone -q https://github.com/raisga/p4n4-lib p4n4/lib && git clone -q https://github.com/raisga/p4n4-emu p4n4/tools/emu && git clone -q https://github.com/raisga/p4n4-templates p4n4/tools/templates'
  - su - ubuntu -c '~/.local/bin/uv tool install ~/p4n4/tools/emu --with ~/p4n4/lib'
EOF
```

- The instance is launched with `--instance-initiated-shutdown-behavior terminate`, so the TTL shutdown **terminates** it and deletes its disk. Upload the report before then ([section 6](#6-get-the-report)). To give a running emulator more time, run `sudo shutdown -P +240` on it (4 more hours from now); `sudo shutdown -c` cancels the TTL.
- `--with ~/p4n4/lib` installs p4n4-lib from the checkout. p4n4-emu needs a p4n4-lib version that isn't on PyPI yet.
- To test a specific release rather than `main`, add `--branch v<version>` to the `git clone` commands.

### 4.3 Launch it

```bash
ID=$(aws ec2 run-instances \
  --image-id resolve:ssm:/aws/service/canonical/ubuntu/server/24.04/stable/current/$ARCH/hvm/ebs-gp3/ami-id \
  --instance-type $TYPE \
  --iam-instance-profile Name=p4n4-emu-instance \
  --security-group-ids $SG \
  --user-data file://emu-cloud-init.yaml \
  --instance-initiated-shutdown-behavior terminate \
  --metadata-options HttpTokens=required \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":40,"VolumeType":"gp3","DeleteOnTermination":true}}]' \
  --tag-specifications \
    "ResourceType=instance,Tags=[{Key=Name,Value=p4n4-emu-$RUN},{Key=Project,Value=p4n4},{Key=p4n4-run,Value=$RUN},{Key=p4n4-profile,Value=$PROFILE}]" \
    "ResourceType=volume,Tags=[{Key=Project,Value=p4n4},{Key=p4n4-run,Value=$RUN}]" \
  --query 'Instances[0].InstanceId' --output text)
echo "$RUN $ID"

aws ec2 wait instance-status-ok --instance-ids $ID      # 2–4 minutes
```

`resolve:ssm:` looks up the current Ubuntu 24.04 image that Canonical publishes for the region and architecture. The `Project` and `p4n4-run` tags are what the cost report ([6.3](#63-cost-report)) and the cleanup checks ([7.2](#72-check-that-nothing-is-left)) filter on. 40 GB leaves room for images, InfluxDB data and an Ollama model.

### 4.4 GPU profile (`jetson-orin-nano`)

The `jetson-orin-nano` profile reserves an NVIDIA GPU for `ollama` and `ei-runner`. That needs the NVIDIA driver and Container Toolkit, which AWS's Deep Learning ARM64 Base GPU AMI includes. Launch with `TYPE=g5g.2xlarge`, `ARCH=arm64` and that image:

```bash
--image-id resolve:ssm:/aws/service/deeplearning/ami/arm64/base-oss-nvidia-driver-gpu-ubuntu-24.04/latest/ami-id
```

Remove the `get.docker.com` line from `emu-cloud-init.yaml`, because the AMI already has Docker. On the instance, `docker info | grep -i runtimes` should list `nvidia` before you run `up`. G instances count against the separate *G and VT* vCPU quota from [2.6](#26-check-the-vcpu-quotas), which starts at 0 on many new accounts.

---

## 5. Run a project on it

### 5.1 Connect and check

```bash
aws ssm start-session --target $ID
```

```bash
# On the instance (the session starts as ssm-user)
sudo -iu ubuntu
cloud-init status --wait                  # "status: done"; if not, see /var/log/cloud-init-output.log
p4n4-emu setup --check-only               # Docker, Compose and cgroup v2 OK
p4n4-emu calibrate                        # benchmarks one Graviton core against the boards' cores
p4n4-emu calibrate --show                 # the CPU factor each profile gets on this instance
```

If `start-session` says `TargetNotConnected`, the SSM agent hasn't registered yet. Wait a minute and try again.

Calibration matters here. A Graviton core does more work than a Pi core, so without calibration a 4-core `rpi5` would get four fast cores. With it, `up` scales each CPU limit by the Pi core's speed relative to the instance's ([core speed](https://github.com/raisga/p4n4-emu#hardware-profiles)).

### 5.2 Put the project on it

**From a template.** On the instance:

```bash
cp -r ~/p4n4/tools/templates/projects/mqtt-influx-grafana ~/greenhouse
cd ~/greenhouse
cp .env.example .env        # ARCHIVE_UID/GID are already 1000, the ubuntu user
```

**Your own project.** On your workstation, upload it to the bucket. Leave out local data. `.env` can go along, because the bucket is private and the instance is thrown away. Don't ship production secrets this way, though.

```bash
tar czf - -C ~/projects --exclude 'greenhouse/data/archive/*/*' greenhouse \
  | aws s3 cp - s3://$BUCKET/projects/$RUN.tar.gz
echo $BUCKET $RUN           # you need both on the instance
```

```bash
# On the instance, with the two values from above
aws s3 cp s3://<bucket>/projects/<run>.tar.gz - | tar xzf - -C ~
cd ~/greenhouse
```

For a multi-layer project (iot + ai, …), ship the whole project directory. `p4n4-emu up` at its root starts every layer ([multi-layer templates](deployment.md#multi-layer-templates)).

### 5.3 Start it under the profile

```bash
p4n4-emu up --profile rpi5 --dry-run      # look at the overlay first
p4n4-emu up --profile rpi5 --sim --sim-devices 3 --cpuset auto
p4n4-emu status                           # usage against each service's limits
```

Check the summary that `up` prints:

- **Platform** says `native`. A Graviton instance runs arm64 images without QEMU.
- **Block device detected** names the root disk (`/dev/nvme0n1`), so the disk limits apply. If it says no limit (no block device), the CPU and memory limits still apply, but disk I/O isn't throttled.
- **CPU limit** shows the calibrated factor, for example `2.80 host cores = 4 × 0.700`.

Under `--cpuset auto`, every service runs on the first four host cores, as it would on a Pi's four. Leave the stack running as long as the test calls for, from minutes for a smoke test to hours for a soak test, staying within the TTL. The template's `./tests/smoke.sh` runs its own throwaway copy of the stack. Run it on the instance as well if you want its end-to-end check.

### 5.4 Open Grafana

The port forward runs in a second terminal on your workstation. No port opens on the instance:

```bash
aws ssm start-session --target $ID --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["3000"],"localPortNumber":["3000"]}'
```

Then open <http://localhost:3000>. Use the same command with `8086` for InfluxDB, or `1880` for Node-RED in the templates that have it.

---

## 6. Get the report

### 6.1 Make the report on the instance

`p4n4-emu report` samples every container's CPU and memory several times. It records what Docker knows about each container's life: state, health, restarts and OOM kills. It also saves each container's last log lines. It writes everything to one archive:

| File | Contents |
|---|---|
| `summary.md` | For people: issues found, and per stack a table of state, health, restarts, CPU and memory (mean / peak / limit) |
| `report.json` | For scripts: everything `status --json` has, plus `usage_over_time`, `lifecycle`, `host` (instance hostname, Docker version, calibration) and `issues` |
| `logs/<container>.log` | The last `--tail` lines (default 500) of each container, with timestamps, the simulator's included |

Issues come in two levels. **Errors** are a container that isn't running, an unhealthy, OOM-killed or restarted container, or limits that differ from the overlay. **Warnings** are a peak at 90 % or more of a CPU or memory limit, cgroup v1, or no stack under p4n4-emu. `--fail-on-issues` exits 1 on any error, for CI.

You can run the report from your workstation, without opening a session. SSM runs it on the instance, which then uploads the archive to the bucket:

```bash
CMD="cd ~/greenhouse && p4n4-emu report --samples 12 --interval 5 \
  -l run=$RUN -l profile=$PROFILE -l instance-type=$TYPE -l instance-id=$ID \
  -o /tmp/$RUN.tar.gz && aws s3 cp /tmp/$RUN.tar.gz s3://$BUCKET/reports/$RUN.tar.gz"

CID=$(aws ssm send-command --instance-ids $ID --document-name AWS-RunShellScript \
  --comment "p4n4-emu report $RUN" \
  --parameters "$(jq -n --arg c "sudo -iu ubuntu bash -lc '$CMD'" '{commands: [$c]}')" \
  --query Command.CommandId --output text)

until s=$(aws ssm get-command-invocation --command-id $CID --instance-id $ID --query Status --output text) \
      && [ "$s" != Pending ] && [ "$s" != InProgress ]; do sleep 5; done
echo "$s"
aws ssm get-command-invocation --command-id $CID --instance-id $ID --query StandardOutputContent --output text
```

This report samples for about a minute (12 samples, 5 s apart). Sample longer to catch periodic load, such as `--samples 60 --interval 10`. In an SSM session, run the same `p4n4-emu report` and `aws s3 cp` lines yourself, as the `ubuntu` user.

Take the report **before** the TTL or [section 7](#7-delete-the-emulator) ends the run. A terminated instance's disk is gone.

### 6.2 Read it

On your workstation:

```bash
mkdir -p ~/p4n4-reports && cd ~/p4n4-reports
aws s3 cp s3://$BUCKET/reports/$RUN.tar.gz .
tar xzf $RUN.tar.gz
cat p4n4-emu-report-*/summary.md          # or open it in an editor that renders Markdown
jq '.issues' p4n4-emu-report-*/report.json
```

`aws s3 ls s3://$BUCKET/reports/` lists past runs. Comparing two runs' `report.json` shows a regression, for example an image update that raised a service's peak memory under the same profile:

```bash
jq -r '.stacks[].services[] | [.service, .usage_over_time.memory.peak] | @tsv' */report.json
```

### 6.3 Cost report

**Tags first, once.** The cost report filters on the `Project` and `p4n4-run` tags. Cost Explorer only knows tags that you activate as cost allocation tags. A tag key can only be activated after a resource carries it, and it can take up to 24 hours after your first run to show up. After that, an admin activates both:

```bash
AWS_PROFILE=p4n4-admin aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status TagKey=Project,Status=Active TagKey=p4n4-run,Status=Active
```

Costs from before activation aren't tagged, so the first run before activation shows up only in the untagged total.

**Per run.** Cost data lags by up to a day, so ask the day after a run:

```bash
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '-7 days' +%F),End=$(date -d '+1 day' +%F) \
  --granularity DAILY --metrics UnblendedCost \
  --filter '{"Tags": {"Key": "Project", "Values": ["p4n4"]}}' \
  --group-by Type=TAG,Key=p4n4-run \
  --query 'ResultsByTime[].Groups[].[Keys[0], Metrics.UnblendedCost.Amount]' --output text
```

Each row is one day's cost of one run (`p4n4-run$rpi5-20261015-1430   0.42`), and includes the instance and its disk. A run that spans midnight has a row for each day. Use `--granularity MONTHLY` for a monthly total. Each Cost Explorer API call costs USD 0.01. The console's **Cost Explorer** shows the same data, grouped by the `p4n4-run` tag.

**Right away.** Before Cost Explorer has the data, estimate from the run time:

```bash
aws ec2 describe-instances --instance-ids $ID \
  --query 'Reservations[0].Instances[0].[InstanceType, LaunchTime]' --output text
```

Multiply the hours since `LaunchTime` by the type's on-demand price on [EC2 pricing](https://aws.amazon.com/ec2/pricing/on-demand/). The disk adds 40 GB × the gp3 price per GB-month, prorated, which is a few cents a day.

---

## 7. Delete the emulator

### 7.1 Terminate the instance

```bash
aws ec2 terminate-instances --instance-ids $ID --query 'TerminatingInstances[0].CurrentState.Name'
aws ec2 wait instance-terminated --instance-ids $ID
aws s3 rm s3://$BUCKET/projects/$RUN.tar.gz       # if you shipped a project
```

You don't need `p4n4-emu down` first. Terminating the instance deletes the root disk (`DeleteOnTermination`), and the containers, volumes and overlays go with it. The report in `reports/` stays until the lifecycle rule expires it.

### 7.2 Check that nothing is left

Anything still running costs money. Check after each run, and from time to time for everyone's runs:

```bash
# Instances not terminated (any run, anyone's)
aws ec2 describe-instances --filters Name=tag:Project,Values=p4n4 \
  Name=instance-state-name,Values=pending,running,stopping,stopped \
  --query 'Reservations[].Instances[].[InstanceId, Tags[?Key==`p4n4-run`]|[0].Value, InstanceType, LaunchTime]' \
  --output table

# Disks left behind (should be empty)
aws ec2 describe-volumes --filters Name=tag:Project,Values=p4n4 \
  --query 'Volumes[].[VolumeId, State, Size, Tags[?Key==`p4n4-run`]|[0].Value]' --output table

# Everything tagged Project=p4n4 (the bucket, the security group and, until it's gone, the instance)
aws resourcegroupstaggingapi get-resources --tag-filters Key=Project,Values=p4n4 \
  --query 'ResourceTagMappingList[].ResourceARN' --output text
```

A terminated instance still appears in `describe-instances` for about an hour, which is why the first command filters on state. Terminated instances cost nothing.

### 7.3 Remove the account setup (when you're done with AWS)

As an admin, in reverse order of [section 3](#3-shared-resources-once):

```bash
aws s3 rb s3://$BUCKET --force                    # deletes every report
aws iam remove-role-from-instance-profile --instance-profile-name p4n4-emu-instance --role-name p4n4-emu-instance
aws iam delete-instance-profile --instance-profile-name p4n4-emu-instance
aws iam delete-role-policy --role-name p4n4-emu-instance --policy-name bucket
aws iam detach-role-policy --role-name p4n4-emu-instance --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role --role-name p4n4-emu-instance
aws ec2 delete-security-group --group-name p4n4-emu
aws budgets delete-budget --account-id $ACCOUNT --budget-name p4n4-monthly
```

To close the account itself, sign in as root → account menu → **Account** → **Close account**.

---

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `VcpuLimitExceeded` on `run-instances` | The vCPU quota is lower than the running instances plus this one. Terminate leftovers ([7.2](#72-check-that-nothing-is-left)) or request an increase ([2.6](#26-check-the-vcpu-quotas)) |
| `InsufficientInstanceCapacity` | The availability zone has no capacity for the type right now. Pass `--subnet-id` of another default subnet (`aws ec2 describe-subnets --filters Name=default-for-az,Values=true`), or try another type from the same family |
| `Unsupported` / unknown instance type | The type isn't offered in the region or zone. See the `describe-instance-type-offerings` check in [4.1](#41-pick-an-instance-for-the-profile) |
| Launch refused for an `xlarge` type on a new account | The account is on the Free plan. Upgrade it to the Paid plan in the Billing console ([2.1](#21-sign-up)) |
| `TargetNotConnected` from `start-session` or `send-command` | The agent hasn't registered yet (wait a minute), the instance has no `p4n4-emu-instance` profile, or it has no outbound internet access (no public IP in a non-default subnet) |
| `p4n4-emu: command not found` | cloud-init hasn't finished or failed. Run `cloud-init status --wait`, then `sudo cat /var/log/cloud-init-output.log`. Use `sudo -iu ubuntu`, not the session's `ssm-user` |
| `permission denied … docker.sock` | The shell started before `usermod -aG docker ubuntu` ran. Run `exit` and `sudo -iu ubuntu` again |
| `up` says no limit (no block device) | p4n4-emu didn't find the disk under Docker's data root, so disk I/O isn't throttled. CPU and memory limits still apply. Check `findmnt -no SOURCE /var/lib/docker` |
| The instance vanished mid-run | The TTL ran out. Raise `TTL_MIN` for long soaks, or extend it on the instance (`sudo shutdown -P +<minutes>`) |
| `send-command` returns `Failed` | Read `StandardErrorContent` from `get-command-invocation`. A path in `CMD` that only exists in your session, such as a different project directory, is the usual cause |
| `report` says no containers found | It ran outside the project directory, or after the stack stopped. `cd` to the project, or pass `--stack-dir` |

## Checklist per run

- [ ] `aws sso login --sso-session p4n4`, variables set ([4](#4-create-an-emulator)), `TTL_MIN` long enough for the test
- [ ] Instance type from the table in [4.1](#41-pick-an-instance-for-the-profile), with `ARCH` to match
- [ ] On the instance: `cloud-init status` done, `p4n4-emu setup --check-only` OK, `p4n4-emu calibrate` run
- [ ] `up` shows `native`, a block device and the calibrated CPU factor
- [ ] Report taken and in `s3://<bucket>/reports/<run>.tar.gz` **before** terminating
- [ ] Instance terminated, [7.2](#72-check-that-nothing-is-left) shows nothing running
- [ ] Next day: the run's cost from Cost Explorer ([6.3](#63-cost-report))
