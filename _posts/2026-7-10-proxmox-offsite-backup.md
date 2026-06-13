---
title: "💾 Offsite backups with Proxmox Backup Server and a Hetzner Storage Box"
description: "A practical guide to building a 3-2-1 backup setup for your Proxmox homelab: fast local backups with Proxmox Backup Server, synced nightly to a cheap Hetzner Storage Box, with client-side encryption so the cloud never sees your data."
date: 2026-7-10
permalink: /posts/2026/7/PBSHetzner/
categories: [Guides, 💾 Backups]
tags: [Proxmox 🖥️, PBS 💾, Hetzner ☁️, Backups 🗄️, Encryption 🔐, DevOps ⚙️]
pin: false
published: true
---

# Offsite backups with Proxmox Backup Server and a Hetzner Storage Box

## Introduction

A homelab without backups is a homelab on borrowed time. And a homelab with backups on the same hardware is only slightly better: a dead disk, a fried PSU, or a ransomware-encrypted host takes your VMs *and* your backups down in one go.

The classic answer is the **3-2-1 rule**: three copies of your data, on two different media, with one copy offsite. For a Proxmox homelab there is a setup that gets you there for a few euros per month:

- [Proxmox Backup Server](https://www.proxmox.com/en/proxmox-backup-server) (PBS) for fast, deduplicated, incremental backups on local storage
- A [Hetzner Storage Box](https://www.hetzner.com/storage/storage-box) as the cheap offsite copy, starting at 1 TB for about the price of a coffee
- Client-side encryption, so Hetzner only ever stores encrypted chunks

PBS is the natural choice if you already run Proxmox VE: it integrates directly as a storage backend, backups are incremental after the first run thanks to dirty bitmaps, and restores (including single-file restore) happen straight from the PVE web UI.

## The architecture

The trick is to *not* point your backups directly at the Storage Box. PBS datastores are chunk stores with tens of thousands of small files, and running one directly over a WAN mount is painfully slow for backups, verification, and garbage collection.

Instead, PBS backs up to fast local storage first, and a built-in **sync job** copies the snapshots to a second datastore that lives on the Storage Box:

```
┌─────────────┐  backup (LAN, fast)  ┌──────────────────┐
│  Proxmox VE │ ───────────────────► │       PBS        │
│   (hosts)   │                      │  local datastore │
└─────────────┘                      └────────┬─────────┘
                                              │ sync job (nightly)
                                              ▼
                                     ┌──────────────────┐
                                     │  CIFS mount of   │
                                     │ Hetzner Storage  │
                                     │ Box = offsite    │
                                     │    datastore     │
                                     └──────────────────┘
```

Daily restores come from the fast local datastore. The Storage Box copy is the disaster-recovery layer you hope to never need.

## Prerequisites

Before you start, you need:

- A Proxmox VE host (see my [ACME certificate guide](/posts/2025/3/ProxmoxTransip/) if yours still greets you with a self-signed certificate warning)
- PBS installed, either on separate hardware or as a VM, with enough local disk for your backups. Bare metal with its own disks is ideal; a VM works fine as long as its storage does not live on the same disk as the VMs it protects
- A Hetzner Storage Box (the 1 TB BX11 is plenty for most homelabs thanks to deduplication and compression)
- Basic familiarity with the PVE and PBS web interfaces

## Step-by-Step Setup

### Step 1: Prepare the Storage Box

In the Hetzner console:

1. Enable **SMB support** on the Storage Box itself (under "Change settings"); it can take a few minutes to activate.
2. Create a **sub-account** for PBS. Never use the main account credentials in config files; sub-accounts can be scoped to their own directory and revoked independently.
3. Enable **Samba/CIFS** *and* **external reachability** for the sub-account. These are separate toggles, and since your PBS lives outside Hetzner's network, you need both.
4. Note the hostname, it looks like `uXXXXXX-sub1.your-storagebox.de`. For sub-accounts, the share name is the username itself.
5. While you are there, enable **automatic snapshots** on the Storage Box. More on why in the Challenges section.

### Step 2: Mount the Storage Box on the PBS host

Create a credentials file on the PBS host so the password does not end up in `/etc/fstab`:

```bash
cat > /etc/storagebox-credentials << 'EOF'
username=uXXXXXX-sub1
password=<SUB-ACCOUNT PASSWORD>
EOF
chmod 600 /etc/storagebox-credentials
```

Then add the mount to `/etc/fstab`. The `uid=34,gid=34` part is important: that is the `backup` user PBS runs as, and it must own the datastore.

```
//uXXXXXX-sub1.your-storagebox.de/uXXXXXX-sub1 /mnt/storagebox cifs credentials=/etc/storagebox-credentials,uid=34,gid=34,file_mode=0660,dir_mode=0770,vers=3.1.1,seal,_netdev,x-systemd.automount 0 0
```

A few notes on the options:

- `vers=3.1.1,seal` forces modern SMB with transport encryption, so traffic to Hetzner is encrypted even before we add client-side encryption
- `_netdev` and `x-systemd.automount` make sure the mount waits for the network and recovers gracefully after reboots
- Do **not** add `noatime`. PBS garbage collection uses access times to decide which chunks are still alive

Mount and verify:

```bash
mkdir -p /mnt/storagebox
systemctl daemon-reload
mount /mnt/storagebox
touch /mnt/storagebox/test && rm /mnt/storagebox/test
```

### Step 3: Create the offsite datastore

In the PBS web UI, go to **Datastore → Add Datastore** and point it at a directory on the mount, e.g. `/mnt/storagebox/offsite`. Or from the shell:

```bash
proxmox-backup-manager datastore create offsite /mnt/storagebox/offsite
```

Now go get a coffee. Creating a datastore initializes the chunk store, which means creating 65,536 directories, and over a WAN mount that takes a while. This is normal and only happens once.

I assume you already have a local datastore (e.g. `local-store` on the PBS disks); if not, create that one too. It will be instant by comparison, which nicely demonstrates why we do not back up straight to the Storage Box.

### Step 4: Connect PVE to PBS with client-side encryption

On the Proxmox VE side, go to **Datacenter → Storage → Add → Proxmox Backup Server**:

```
ID:           pbs-local
Server:       <PBS IP or hostname>
Username:     <user>@pbs
Datastore:    local-store
Fingerprint:  <copy from the PBS dashboard>
Encryption:   Auto-generate a client encryption key
```

That last setting is the important one. With a client-side encryption key, chunks are encrypted on the PVE host *before* they are sent to PBS. The local PBS never sees plaintext, and neither does the Storage Box after the sync. Hetzner stores noise.

**Back up this key.** Right now. PVE shows the key in a dialog when you create the storage; download it there. After that, it lives at `/etc/pve/priv/storage/pbs-local.enc` on the PVE host. Copy it into your password manager and somewhere offline. Without this key, your backups are exactly as useful as the encrypted noise they look like. PVE will nag you about this for good reason.

Then create a backup job under **Datacenter → Backup** targeting `pbs-local`, e.g. daily at 02:00 with all VMs selected.

### Step 5: Set up the sync job

Back in PBS, go to the **offsite** datastore → **Sync Jobs → Add**:

```
Source Remote:    Local
Source Datastore: local-store
Schedule:         daily 04:00 (after the backup job has finished)
```

Since PBS supports local sync jobs, no second PBS instance is needed: it simply copies snapshots from the local datastore to the offsite one. Only new chunks are transferred, so after the first full sync the nightly delta is small.

If your upload bandwidth is limited, set a rate limit on the sync job so your 04:00 sync does not ruin the morning video calls.

### Step 6: Prune, garbage collect, verify

Backups grow forever unless you tell them not to. Configure per datastore:

**Prune** (what to keep), under **Datastore → Prune & GC**:

```
local-store:  keep-daily=7, keep-weekly=4
offsite:      keep-daily=7, keep-weekly=4, keep-monthly=6
```

The offsite store keeps a longer history; storage there is cheap and it doubles as your archive.

**Garbage collection** (actually freeing the space pruned snapshots referenced): schedule it daily on the local store, weekly on the offsite store. GC over CIFS is slow because it has to touch every chunk, so do not run it more often than needed.

**Verify**: schedule weekly on the local store. On the offsite store, run it sparingly (e.g. monthly) or on demand: verification re-reads every chunk, which over a WAN mount means downloading your entire backup set.

### Step 7: Test a restore

A backup that has never been restored is a hope, not a backup. Pick a sacrificial VM and:

1. Restore the full VM from `pbs-local` to a new VM ID and boot it
2. Open a backup in the PVE UI and use **File Restore** to pull out a single file
3. For the full disaster scenario: temporarily add the offsite datastore as a storage in PVE and restore a VM directly from the Storage Box copy. Slow, but this is exactly what you would do if the local PBS burned down, so know that it works *before* you need it

## Challenges & Learnings

🐌 **The chunk store and the WAN do not love each other**: My first attempt was backing up directly to a datastore on the Storage Box, no local layer. Backups were tolerable thanks to deduplication, but garbage collection took the better part of a day and verify jobs were hopeless. The local-first + sync architecture is not optional, it is the difference between a usable setup and a frustrating one.

🔑 **The encryption key is a single point of failure by design**: Client-side encryption means Hetzner cannot read your data, but it also means *you* cannot read your data without the key. The key lives on the PVE host, which is one of the things you are backing up. Export it and store it outside the homelab, otherwise the disaster that takes out your PVE host also takes out the ability to restore from your perfectly intact offsite backups.

📸 **Sync propagates mistakes, snapshots forgive them**: A sync job faithfully mirrors the source, including prunes. Ransomware or a fat-fingered prune rule on the local store will happily sync its destruction to the offsite copy. The Storage Box's own snapshot feature is the safety net here: enable automatic daily snapshots and even a compromised PBS cannot reach into Hetzner's snapshot history.

⏱️ **GC needs working access times**: PBS garbage collection marks chunks as in-use by updating their atime, and not every network filesystem handles that correctly. PBS actually tests atime support on the datastore and falls back to a more conservative cutoff if needed, but it is worth knowing why that `noatime` mount option, which every performance guide tells you to add, stays out of this fstab entry.

## Troubleshooting

If the mount fails with `permission denied`, check that Samba is enabled on the *sub-account* (not just the main account) and that external reachability is on. Hetzner's error messages via CIFS are not exactly verbose.

If PBS shows the datastore but backups fail with permission errors, the mount is missing the `uid=34,gid=34` options. The `backup` user must own the datastore path; `ls -ln /mnt/storagebox` should show `34 34`.

If the datastore creation seems frozen, it probably is not. Creating 65k chunk directories over CIFS legitimately takes a long time. Check the task log in the PBS UI before assuming it died.

If sync jobs are slow even with little changed data, check whether a GC or verify job is running on the offsite store at the same time. Over a WAN mount these jobs compete for the same limited IOPS, so schedule them on different days.

## Next Steps

- Hook PBS into the monitoring stack from my [Prometheus and Grafana guide](/posts/2026/5/PrometheusGrafanaMonitoring/) using the PBS exporter, and alert on failed backup and sync tasks instead of discovering them weeks later
- Configure PBS notifications (SMTP or webhook) so failed jobs reach you even when Grafana does not
- Look at the S3 datastore backend introduced in PBS 4 paired with object storage as an alternative offsite layer, once it matures beyond technology preview
- Add a third copy: even a single USB disk with a weekly sync job, kept offline, covers the scenarios both online copies share

## Conclusion

With a local PBS datastore for fast daily restores, a nightly sync to a Hetzner Storage Box for disasters, and client-side encryption so the offsite copy is unreadable to anyone but you, the homelab finally satisfies the 3-2-1 rule without enterprise prices. Combined with the [Traefik](/posts/2025/2/traefik/), [monitoring](/posts/2026/5/PrometheusGrafanaMonitoring/), and [Authelia](/posts/2026/6/AutheliaTraefik/) setups from the previous guides, the stack is now routed, observed, locked down, and backed up.

💡 Want to learn more about backup strategies for Proxmox, or need help setting this up for production workloads? Feel free to reach out!