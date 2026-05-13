# Filesystem Audit — Inside solveit Container

**Date:** 2026-05-13
**Container ID:** `aabb4c8a0f0a`
**Container Runtime:** containerd (overlayfs snapshotter)

---

## Quick Summary

| Property | Value |
|----------|-------|
| **Container sees** | ext4 on local block device `/dev/sda1` |
| **Disk size** | 2.5 TB HDD (rotational=1) |
| **Used / Total** | 1.6 TB / 2.5 TB (64%) |
| **Mount type** | bind mount via containerd volume |
| **Host source path** | `/data/volumes/61jstpii` |
| **Container path** | `/app/data` |
| **Block device** | major:minor 8:1 |

---

## What Is Visible From Inside the Container

### Mount Info (from `/proc/1/mountinfo`)

```
46397 46325 8:1 /data/volumes/61jstpii /app/data rw,relatime - ext4 /dev/sda1 rw,discard,errors=remount-ro,commit=30
```

- **Filesystem type:** `ext4`
- **Device:** `/dev/sda1` (major:minor 8:1)
- **Host-side source path:** `/data/volumes/61jstpii`
- **Mount options:** `rw, relatime, discard, errors=remount-ro, commit=30`

### Disk Characteristics

| Property | Value |
|----------|-------|
| Rotational | 1 (HDD, not SSD) |
| Discard granularity | 4096 bytes |
| Block size | 4096 |
| Total inodes | 335,413,248 |
| Used inodes | 24,143,694 (8%) |

### Network / DNS

- **Nameserver:** 127.0.0.11 (Docker DNS)
- **Search domain:** `genesishosting.com`
- **Host resolver:** 127.0.0.53 (systemd-resolved)

### Container Environment

- **No k8s service account** — not running in Kubernetes
- **No cgroup path** — bare containerd/podman, not orchestrated
- **Isolated mount namespace** — cannot see host's mount table
- **No FUSE mounts** — no s3fs, goofys, rclone
- **No NFS** — none in `/proc/mounts`
- **No Ceph/Gluster/Lustre** — none in `/proc/mounts`

---

## Architecture (Reconstructed)

```
┌─────────────────────────────────────────────────────┐
│                    HOST (geneshosting.com)           │
│                                                     │
│  /dev/sda1 (ext4, 2.5TB HDD)                        │
│    └── /data/volumes/61jstpii  (bind mount source)  │
│                                                     │
│  (Also mounts /etc/hosts, /etc/hostname,             │
│   /etc/resolv.conf from same device)                │
└────────────────┬────────────────────────────────────┘
                 │ containerd bind-mount
                 ▼
┌─────────────────────────────────────────────────────┐
│                  CONTAINER                            │
│                                                     │
│  /app/data  (ext4, device 8:1)                      │
│  Mounts: JCMSSalesPortal, memexsolve, memexcode,    │
│          memexplatform, memexhelper, memexpaas,      │
│          cv, yellowcrop, memexplatform_paas,         │
│          memexplatform_flows, voice-stack            │
│                                                     │
│  2.5TB total, 1.6TB used (64%)                      │
│  335M inodes total, 24M used (8%)                   │
└─────────────────────────────────────────────────────┘
```

---

## What Is NOT Visible From Inside the Container

The container runs in an **isolated mount namespace**, so the following is **hidden**:

1. **What filesystem `/data` is mounted on the host** — `/data` on the host could be NFS, Ceph, a distributed FS, or just a local ext4 partition
2. **Host mount table** — `cat /proc/1/root/proc/mounts` is not accessible
3. **Whether the host has additional storage backends** — could be running FUSE-based FS (JuiceFS, CephFS, etc.) on the host that appears as ext4 to the container

---

## What We Can Rule Out

| FS Type | Verdict | Evidence |
|---------|---------|----------|
| **NFS** | ❌ Not in container | No `nfs` type in `/proc/mounts` |
| **S3+POSIX (s3fs/goofys)** | ❌ Not in container | No FUSE mounts at all; `s3fs`/`rclone` not installed |
| **Ceph/Gluster/Lustre** | ❌ Not in container | No such types in `/proc/mounts` |
| **tmpfs/memory FS** | ❌ Not the data layer | `/app/data` is persistent on disk |
| **SSD/NVMe** | ❌ No | `/sys/block/sda/queue/rotational` = 1 (HDD) |

---

## Most Likely Scenario

```
Host:  /dev/sda1  (local SATA/SAS HDD, ext4, 2.5TB)
           │
           ├── bind mount ──▶ /data
           │                      │
           │                   subdirs:
           │                   /data/volumes/61jstpii  ← this gets mounted into containers
           │                   /data/volumes/<other IDs>
           │
Container: /app/data  (ext4, device 8:1, via bind mount)
```

The `/data/volumes/` naming convention suggests a custom volume management system (possibly related to the container runtime or a custom orchestrator) that manages volumes under `/data`. The container runtime (containerd) takes the host path `/data/volumes/61jstpii` and bind-mounts it as `/app/data` inside the container.

**The `/dev/sda1` on the host is almost certainly a local disk** (the naming `sda1`, `sda14`, `sda15`, `sda16` and the partition layout is typical of a cloud VM root+data disk). However, whether the host's `/data` directory itself sits on a distributed filesystem underneath is **unknown** without host access.

---

## Git Repos in /app/data (Status as of 2026-05-13)

All repos were committed clean on this session:

| Repo | Last Commit |
|------|-------------|
| `memexcode` (dialogs/b10Xp6TI7aVVLp8GyipIkw/) | 36,667 insertions, 13 files |
| `floor_assistant` (dialogs/b10Xp6TI7aVVLp8GyipIkw/) | 710 insertions, 2 files |
| `memexpaas_deploy` (dialogs/b10Xp6TI7aVVLp8GyipIkw/) | 264 insertions, 1 file |
| `solveit-voice` (dialogs/b10Xp6TI7aVVLp8GyipIkw/) | 7,711 insertions, 1 file |
| `JCMSSalesPortal` | leads.db added, 2 files |
| `memexsolve` | memexsolve.db journal files, 2 files |
| `memexplatform_paas` | 96 insertions / 80 deletions, 3 files |

All other repos (`browserhacks`, `cv`, `yellowcrop`, `memexpaas`, `memexplatform`, `memexpaas_browser`, `memexgt`, `memexpass_mathagent`, `pi-mono`, `memexplatform_flows`, `memexhelper`) had **no uncommitted changes** at time of scan.

---

## Key Takeaways

1. **Container sees ext4 on a local HDD** — everything appears as local block storage from inside
2. **The actual backend is unknown** — could be distributed at the host layer
3. **No cloud storage backends detected** (S3, NFS, Ceph, etc.)
4. **Disk is full-ish** — 64% used, 917GB free on 2.5TB
5. **Inodes are healthy** — only 8% used
6. **Container is bare containerd** — no Kubernetes, no orchestration layer visible
