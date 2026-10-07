# Lab 7B: EBS vs. EFS vs. Instance Store — Block, File and the AZ Boundary

**Session:** 7 — Storage & DNS Performance  
**Exam Domain:** Domain 3 — High-Performing (24%) · Domain 2 — Resilient (26%)  
**Difficulty:** Intermediate  
**Estimated Time:** 50–60 minutes

---

## Overview

"Multiple EC2 instances in different AZs need to share files" → EFS. "A database needs consistent high IOPS" → EBS io2. "Temporary scratch space with the highest I/O" → instance store. The exam expects instant recall of which storage fits, **and why the others don't**.

In this lab you'll **feel** the difference. You'll mount one EFS file system on two instances in **different AZs**, then try (and fail) to attach an EBS volume across AZs, and fix it the way AWS intends: **snapshot → new volume in the other AZ**. Finally you'll raise a volume's IOPS **while it's in use**.

**What you will build:**

```
            us-east-1a                                 us-east-1b
   ┌────────────────────────┐                ┌────────────────────────┐
   │ saa-node-a             │                │ saa-node-b             │
   │  /mnt/efs ─────────────┼──── EFS ───────┼── /mnt/efs  (shared!)  │
   │  /data ◀─ EBS vol-A    │                │  /data ◀─ EBS vol-B    │
   └────────────────────────┘                └──────────▲─────────────┘
              │ snapshot ──────────────────────────────▶│ (restored copy)
```

---

## Prerequisites

- ✅ Sessions 2 and 4 complete (security groups, SSM)
- ✅ Default VPC in us-east-1

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| EC2 t3.micro ×2 + public IPv4 | Test nodes | ~$0.03/hr |
| EBS gp3 1 GiB ×2 | Block volumes | ~$0.0002/hr |
| EBS snapshot | Copy of 1 GiB (mostly empty) | ~$0.00 |
| Amazon EFS (Standard) | A few KB | $0.30/GB-month (≈ $0) |

**Estimated cost for this lab: ~$0.05**

---

## Concepts

| | **EBS** | **EFS** | **Instance Store** | **S3** |
|-|---------|---------|--------------------|--------|
| Type | Block (a virtual disk) | File (NFS v4) | Block (physical disk on host) | Object |
| Scope | **One AZ** | **Regional** (multi-AZ) | One instance | Regional |
| Attach to | One instance* | **Thousands** of instances, Lambda, ECS | Only its host instance | Any client via API |
| Persistence | Independent of the instance | Independent | **Lost on stop/terminate/host failure** | Durable |
| OS | Linux & Windows | **Linux only** (use **FSx for Windows** for SMB) | Any | Any |
| Use for | Boot volumes, databases | Shared content, home dirs, CMS | Cache, buffers, scratch | Objects, backups, data lakes |

<sub>*io1/io2 support **Multi-Attach** within the same AZ, for clustered apps.</sub>

**EBS volume types:**

| Type | Best For | Key Number |
|------|---------|-----------|
| **gp3** | Default, general purpose | 3,000 IOPS baseline, up to 16,000, **independent of size** |
| **io2 Block Express** | Mission-critical databases | Up to 256,000 IOPS, 99.999% durability |
| **st1** | Big sequential throughput (logs, big data) | HDD, throughput-optimized |
| **sc1** | Cold, infrequent | Cheapest HDD |

---

## ⚠️ Placeholders in This Lab

| Placeholder | What It Is |
|-------------|-----------|
| `<DEFAULT_VPC_ID>`, `<SUBNET_1A>`, `<SUBNET_1B>` | Default VPC and subnets (Step 2) |
| `<NODE_SG_ID>`, `<EFS_SG_ID>` | Security groups (Step 3) |
| `<AMI_ID>` | Amazon Linux 2023 (Step 4) |
| `<NODE_A_ID>`, `<NODE_B_ID>` | Instances (Step 4) |
| `<FS_ID>` | EFS file system (Step 5) |
| `<VOL_A_ID>`, `<SNAP_ID>`, `<VOL_B_ID>` | EBS volume, snapshot, restored volume |

---

## Lab Steps

### Step 1: Profile and Folder

**Windows:** `$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"; cd ~\Desktop\saa-session-7`  
**macOS / Linux:** `export AWS_PROFILE="<YOUR_PROFILE_NAME>" && cd ~/Desktop/saa-session-7`

---

### Step 2: Network Lookup

```
aws ec2 describe-vpcs --filters Name=is-default,Values=true --query "Vpcs[0].VpcId" --output text
```
```
aws ec2 describe-subnets --filters Name=default-for-az,Values=true Name=availability-zone,Values=us-east-1a,us-east-1b --query "Subnets[].[AvailabilityZone,SubnetId]" --output table
```

---

### Step 3: Security Groups — Nodes and EFS

```
aws ec2 create-security-group --group-name saa-storage-node-sg --description "Storage lab nodes" --vpc-id <DEFAULT_VPC_ID> --query GroupId --output text
```
```
aws ec2 create-security-group --group-name saa-efs-sg --description "NFS from storage nodes" --vpc-id <DEFAULT_VPC_ID> --query GroupId --output text
```
📋 Allow NFS (TCP 2049) into EFS **only from the node SG** (SG chaining from Lab 4B):
```
aws ec2 authorize-security-group-ingress --group-id <EFS_SG_ID> --protocol tcp --port 2049 --source-group <NODE_SG_ID>
```

---

### Step 4: Role and Two Instances in Two AZs

**Create `ec2-trust.json`** (same as earlier labs):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Principal": { "Service": "ec2.amazonaws.com" }, "Action": "sts:AssumeRole" }
  ]
}
```
```
aws iam create-role --role-name saa-storage-role --assume-role-policy-document file://ec2-trust.json
```
```
aws iam attach-role-policy --role-name saa-storage-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```
```
aws iam create-instance-profile --instance-profile-name saa-storage-profile
```
```
aws iam add-role-to-instance-profile --instance-profile-name saa-storage-profile --role-name saa-storage-role
```
```
aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text
```

Wait 10 seconds, then launch **one node per AZ**:
```
aws ec2 run-instances --image-id <AMI_ID> --instance-type t3.micro --subnet-id <SUBNET_1A> --security-group-ids <NODE_SG_ID> --iam-instance-profile Name=saa-storage-profile --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=saa-node-a}]" --query "Instances[0].InstanceId" --output text
```
```
aws ec2 run-instances --image-id <AMI_ID> --instance-type t3.micro --subnet-id <SUBNET_1B> --security-group-ids <NODE_SG_ID> --iam-instance-profile Name=saa-storage-profile --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=saa-node-b}]" --query "Instances[0].InstanceId" --output text
```

> **📝 Save `<NODE_A_ID>` and `<NODE_B_ID>`**

---

## Part 1 — EFS: One File System, Many AZs

### Step 5: Create EFS with Mount Targets in Both AZs

```
aws efs create-file-system --performance-mode generalPurpose --throughput-mode elastic --encrypted --tags Key=Name,Value=saa-shared --query FileSystemId --output text
```

> **📝 Save as `<FS_ID>`**. Wait ~30 seconds until it's `available`:
```
aws efs describe-file-systems --file-system-id <FS_ID> --query "FileSystems[0].LifeCycleState" --output text
```

📋 A **mount target** (an ENI with an IP) in each AZ:
```
aws efs create-mount-target --file-system-id <FS_ID> --subnet-id <SUBNET_1A> --security-groups <EFS_SG_ID>
```
```
aws efs create-mount-target --file-system-id <FS_ID> --subnet-id <SUBNET_1B> --security-groups <EFS_SG_ID>
```

Wait ~90 seconds for the mount targets to become `available`:
```
aws efs describe-mount-targets --file-system-id <FS_ID> --query "MountTargets[].[AvailabilityZoneName,LifeCycleState]" --output table
```

---

### Step 6: Mount on Both Nodes

Connect to **saa-node-a** with Session Manager (**EC2 → select → Connect → Session Manager**). 📋 Run, **replacing `<FS_ID>`**:
```bash
sudo dnf install -y amazon-efs-utils
sudo mkdir -p /mnt/efs
sudo mount -t efs -o tls <FS_ID>:/ /mnt/efs
echo "Written by node A ($(hostname)) at $(date)" | sudo tee /mnt/efs/hello.txt
```

In a **second browser tab**, connect to **saa-node-b** and run the same first three lines (install, mkdir, mount).

🔮 **Predict:** On node B (a different AZ), what does `cat /mnt/efs/hello.txt` show?

```bash
cat /mnt/efs/hello.txt
```

<details>
<summary>🔮 Reveal</summary>

**Node A's message**, read from another AZ. EFS is a **regional, shared** file system. Now write from B and read from A to prove it works both ways. That's why EFS is the answer for *"multiple instances across AZs need shared access to the same files."*

</details>

> 💡 `-o tls` encrypts data **in transit** between the instance and EFS. You enabled encryption **at rest** with `--encrypted` in Step 5.

---

## Part 2 — EBS: Fast, Durable, and Stuck in One AZ

### Step 7: Create and Attach an EBS Volume to Node A

On your **local terminal**:
```
aws ec2 create-volume --availability-zone us-east-1a --size 1 --volume-type gp3 --tag-specifications "ResourceType=volume,Tags=[{Key=Name,Value=saa-vol-a}]" --query VolumeId --output text
```
```
aws ec2 attach-volume --volume-id <VOL_A_ID> --instance-id <NODE_A_ID> --device /dev/sdf
```

On **node A** (Session Manager):
```bash
lsblk
```
You'll see a new 1G disk, usually `nvme1n1`. 📋 Format, mount and write:
```bash
sudo mkfs -t xfs /dev/nvme1n1
sudo mkdir -p /data
sudo mount /dev/nvme1n1 /data
echo "EBS data from node A" | sudo tee /data/block.txt
sudo umount /data
```

---

### Step 8: Try to Move the Volume to Node B

📋 Detach from A:
```
aws ec2 detach-volume --volume-id <VOL_A_ID>
```
```
aws ec2 wait volume-available --volume-ids <VOL_A_ID>
```

🔮 **Predict:** Can you attach it to node B in **us-east-1b**?
```
aws ec2 attach-volume --volume-id <VOL_A_ID> --instance-id <NODE_B_ID> --device /dev/sdf
```

<details>
<summary>🔮 Reveal</summary>

❌ **`InvalidVolume.ZoneMismatch`**. An EBS volume lives in **one AZ** and can only attach to instances in that AZ.

</details>

---

### Step 9: The Right Way — Snapshot and Restore into Another AZ

```
aws ec2 create-snapshot --volume-id <VOL_A_ID> --description "saa move to 1b" --query SnapshotId --output text
```
```
aws ec2 wait snapshot-completed --snapshot-ids <SNAP_ID>
```

> 💡 Snapshots are stored in S3 behind the scenes and are **regional**, so you can create volumes from them in **any AZ** in the Region, or **copy them to another Region** for DR.

```
aws ec2 create-volume --availability-zone us-east-1b --snapshot-id <SNAP_ID> --volume-type gp3 --tag-specifications "ResourceType=volume,Tags=[{Key=Name,Value=saa-vol-b}]" --query VolumeId --output text
```
```
aws ec2 wait volume-available --volume-ids <VOL_B_ID>
```
```
aws ec2 attach-volume --volume-id <VOL_B_ID> --instance-id <NODE_B_ID> --device /dev/sdf
```

On **node B**:
```bash
lsblk
sudo mkdir -p /data
sudo mount /dev/nvme1n1 /data
cat /data/block.txt
```

**✅ You should see** `EBS data from node A`, now in another AZ. 🎉

---

### Step 10: Elastic Volumes — More IOPS, No Downtime

🔮 **Predict:** Can you raise IOPS on `VOL_B` from 3,000 to 4,000 **while it's mounted** and in use?

```
aws ec2 modify-volume --volume-id <VOL_B_ID> --iops 4000
```
```
aws ec2 describe-volumes-modifications --volume-ids <VOL_B_ID> --query "VolumesModifications[0].[ModificationState,OriginalIops,TargetIops]" --output text
```

<details>
<summary>🔮 Reveal</summary>

✅ **Yes.** **Elastic Volumes** let you change size, type and IOPS **online**. The state goes `modifying` → `optimizing` → `completed`. (Growing the **file system** after a size increase is a separate OS step, e.g. `xfs_growfs`.) After a modification, you must wait **6 hours** before modifying the same volume again.

</details>

### 🧩 Checkpoint

An app needs a **temporary scratch disk** with the **highest possible I/O**, and the data can be lost if the instance stops. Which storage?

<details>
<summary>Answer (+10 XP)</summary>

**Instance store** (on instance families that have it, e.g. `i4i`, `m6id`, `c6gd`). It's physically attached NVMe, the fastest option and free with the instance, but **ephemeral**. The t3 family you used today has no instance store.

</details>

---

### Step 11: Console Checkpoint

**✅ Checkpoint:**
1. **EFS → saa-shared → Network**: two mount targets, two AZs.
2. **EC2 → Volumes**: `saa-vol-a` (1a, available) and `saa-vol-b` (1b, in-use, 4000 IOPS).
3. **EC2 → Snapshots**: your snapshot, which you could **Copy** to another Region.

---

## What You Just Did

1. Shared one **EFS** file system across instances in **two AZs**
2. Proved **EBS is AZ-bound**, and moved data the right way with a **snapshot**
3. Changed IOPS **live** with **Elastic Volumes**
4. Placed instance store in the storage decision tree

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Shared file system for many Linux instances across AZs" | **EFS** |
| "Shared file storage for **Windows** / SMB / Active Directory" | **FSx for Windows File Server** |
| "HPC, high-performance parallel file system, integrates with S3" | **FSx for Lustre** |
| "Multi-protocol (NFS + SMB), NetApp features" | **FSx for NetApp ONTAP** |
| "Highest IOPS for a database on EC2" | **EBS io2 Block Express** |
| "Temporary buffer/cache, highest I/O, data loss OK" | **Instance store** |
| "Move an EBS volume to another AZ/Region" | **Snapshot → create volume / copy snapshot** |
| "EFS files rarely accessed after 30 days, lower cost" | **EFS lifecycle management → EFS IA / Archive** |
| "On-prem apps need low-latency access to cloud-backed files" | **Storage Gateway (File Gateway)** |

**🚨 Exam traps**
- EFS is **Linux/NFS**. For Windows, think **FSx for Windows**.
- **gp3 IOPS are independent of size**; gp2 IOPS scale with size (3 IOPS/GB). "Increase gp2 volume size to get more IOPS" was the old answer. Today, **switch to gp3**.
- Instance store data **survives a reboot** but **not a stop, terminate or hardware failure**.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `mount.nfs4: Connection timed out` | EFS SG doesn't allow 2049 from the node SG, or mount targets aren't ready | Check Step 3's rule; wait until mount targets are `available` |
| `amazon-efs-utils` install fails | No internet (wrong subnet) | Default subnets have public IPs; confirm you used them |
| No `nvme1n1` in `lsblk` | Attachment not finished | Wait 10 s; `aws ec2 describe-volumes --volume-ids <VOL_ID>` |
| `mkfs` says device is busy | You mounted it already | `sudo umount /data` first |

---

## 🧹 Cleanup

**1. Terminate instances** (EFS mounts and EBS attachments go with them):
```
aws ec2 terminate-instances --instance-ids <NODE_A_ID> <NODE_B_ID>
aws ec2 wait instance-terminated --instance-ids <NODE_A_ID> <NODE_B_ID>
```

**2. EBS:**
```
aws ec2 delete-volume --volume-id <VOL_A_ID>
aws ec2 delete-volume --volume-id <VOL_B_ID>
aws ec2 delete-snapshot --snapshot-id <SNAP_ID>
```

**3. EFS** (mount targets first):
```
aws efs describe-mount-targets --file-system-id <FS_ID> --query "MountTargets[].MountTargetId" --output text
```
For each ID printed:
```
aws efs delete-mount-target --mount-target-id <MOUNT_TARGET_ID>
```
Wait ~60 seconds, then:
```
aws efs delete-file-system --file-system-id <FS_ID>
```

**4. Security groups and IAM:**
```
aws ec2 delete-security-group --group-id <EFS_SG_ID>
aws ec2 delete-security-group --group-id <NODE_SG_ID>
aws iam remove-role-from-instance-profile --instance-profile-name saa-storage-profile --role-name saa-storage-role
aws iam delete-instance-profile --instance-profile-name saa-storage-profile
aws iam detach-role-policy --role-name saa-storage-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role --role-name saa-storage-role
```

**✅ Checkpoint:** EFS, EBS Volumes and Snapshots show no `saa-*` resources.

---

**🏁 Lab complete: +100 XP.** Next: [Lab 7C — Route 53 Routing Policies](lab-7c-route53-routing-policies.md)
