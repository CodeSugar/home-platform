# ZFS storage guide

Disks in this server:

| Device | Size | Type | Role |
|---|---|---|---|
| sda | 1.7T | SSD | pool `ssd` (mirror) |
| sdb | 1.8T | SSD | pool `ssd` (mirror) |
| sdc | 9.1T | HDD | pool `hdd` (mirror) |
| sdd | 9.1T | HDD | pool `hdd` (mirror) |

SSD pool → Kubernetes volumes. HDD pool → media, photos, cloud files.

---

## 1. Options: ZFS vs mdadm

| | ZFS mirror | mdadm RAID1 + ext4/XFS |
|---|---|---|
| Silent corruption | Detected and repaired (checksums) | Not detected |
| Snapshots | Built in, instant | Needs LVM |
| Compression | Built in (lz4) | No |
| Encryption | Native, per dataset | LUKS underneath |
| Replication | `zfs send` / `zfs receive` | rsync |
| RAM use | Higher (ARC cache) | Minimal |
| Complexity | Moderate | Very simple |

**Choice: ZFS.** Self-healing, snapshots and per-dataset tuning are worth the learning curve. mdadm only wins on simplicity and low RAM.

---

## 2. ZFS concepts

```
disk  →  vdev  →  pool  →  dataset
```

- **disk** — a physical drive. Always reference it by `/dev/disk/by-id/...`, never `sda`, because those letters change between boots.
- **vdev** — a group of disks with a redundancy type. A `mirror` vdev keeps a full copy on each disk. A pool survives the loss of a vdev only if that vdev is itself redundant.
- **pool** — one or more vdevs presented as a single block of storage. This is the part that's hard to change later; `ashift` is fixed forever at creation.
- **dataset** — a filesystem inside a pool. No fixed size: all datasets share the pool's free space. Each has its own properties, snapshots and optional quota. Child datasets inherit properties from the parent.

Useful detail: datasets are real filesystem boundaries, so hardlinks and instant moves don't work between them. Keep downloads and the media library in the same dataset.

---

## 3. Quick start

### Install ZFS (Debian)

```bash
# add "contrib" to the Components line in /etc/apt/sources.list.d/debian.sources
sudo apt update
sudo apt install linux-headers-amd64 zfs-dkms zfsutils-linux
zfs --version
```

On Debian 12, install from `bookworm-backports` to get OpenZFS 2.2+. With Secure Boot on, enroll the DKMS key (MOK).

### Find the disk IDs

```bash
ls -l /dev/disk/by-id/     # use whole-disk entries, not *-part1
```

### Create the pools

`zpool create` wipes the disks. Check the IDs twice.

```bash
# SSD pool
sudo zpool create -o ashift=12 -o autotrim=on \
  -O compression=lz4 -O atime=off -O xattr=sa -O acltype=posixacl \
  -O mountpoint=/mnt/ssd \
  ssd mirror /dev/disk/by-id/ata-SSD_1 /dev/disk/by-id/ata-SSD_2

# HDD pool (no autotrim)
sudo zpool create -o ashift=12 \
  -O compression=lz4 -O atime=off -O xattr=sa -O acltype=posixacl \
  -O mountpoint=/mnt/hdd \
  hdd mirror /dev/disk/by-id/ata-HDD_1 /dev/disk/by-id/ata-HDD_2
```

### Create the datasets

```bash
# Kubernetes volumes
sudo zfs create ssd/k8s                            # /mnt/ssd/k8s
sudo zfs create -o recordsize=16K ssd/k8s/postgres
sudo zfs set quota=10G ssd/k8s/postgres

# Media and files
sudo zfs create -o recordsize=1M hdd/media         # downloads + movies + series together
sudo zfs create -o recordsize=1M hdd/photos
sudo zfs create hdd/cloud
```

### Verify

```bash
zpool status
zfs list
zdb -C ssd | grep ashift     # must say 12
```

---

## 4. Variables explained

### Pool properties (`-o`, lowercase)

| Variable | Meaning |
|---|---|
| `ashift=12` | Sector size as a power of two (2¹² = 4K). **Cannot be changed after creation.** SSDs often report 512 bytes even though they use 4K internally, so set it explicitly. |
| `autotrim=on` | Tells the SSD which blocks are free, keeping write speed consistent. SSDs only; skip it on the HDD pool. |

### Filesystem properties (`-O`, uppercase — inherited by every dataset)

| Variable | Meaning |
|---|---|
| `compression=lz4` | Fast compression; often speeds things up since less data hits the disk. Gives up instantly on already-compressed files. |
| `atime=off` | Stops writing a timestamp on every file read. Less SSD wear, fewer HDD seeks, smaller snapshots. |
| `xattr=sa` | Stores extended attributes inside file metadata instead of hidden files. Much faster. |
| `acltype=posixacl` | Enables POSIX ACLs (`setfacl`/`getfacl`). Some apps, Samba shares and containers need them. |
| `mountpoint=/mnt/ssd` | Where the pool mounts. Datasets mount below it automatically. |

### Per-dataset properties

| Variable | Meaning |
|---|---|
| `recordsize=1M` | Large blocks for big sequential files (movies, photos). |
| `recordsize=16K` | Small blocks matching database page sizes (Postgres, MariaDB). |
| `quota=10G` | Maximum space a dataset may use. Without it, it can fill the whole pool. |

---

## 5. Recommendations

### Cap the ARC (important with Kubernetes)

ZFS's cache shows up as used memory, not cache, so the kubelet can mistake it for memory pressure and evict pods. With 64 GB RAM, start at 16 GB:

```bash
echo "options zfs zfs_arc_max=17179869184" | sudo tee /etc/modprobe.d/zfs.conf
sudo update-initramfs -u
```

Check hit rates later with `arc_summary` and raise it if the pods leave room.

### Start Kubernetes only after ZFS mounts

Otherwise a failed import leaves empty directories and apps start with no data.

```bash
sudo systemctl edit kubelet.service      # and containerd.service (or k3s.service)
```

```ini
[Unit]
After=zfs-mount.service
Requires=zfs-mount.service
```

### Maintenance

- Monthly scrub (`zpool scrub`, usually a systemd timer shipped by the distro).
- Enable ZED email alerts and `smartd` so a failing disk is noticed.
- Automatic snapshots with `sanoid` on `hdd/photos` and `hdd/cloud`. Exclude any containerd dataset.
- No L2ARC or SLOG needed with 64 GB RAM and this workload.
- A mirror is not a backup. Keep a copy on another machine or off-site.