# MiniPC storage rebalance — Option A: move Docker to a new partition

**Goal:** relieve the Linux root partition (`/dev/nvme0n1p6`, 90 GB, ~80 % full) by reclaiming free space from the Windows partition (162 GB free) **without** the risky surgery of growing/moving the root partition.

**Strategy:** shrink Windows from within Windows → create a new ext4 partition in the freed space → move `/var/lib/docker` (the dominant consumer) onto it. The root partition is never moved or resized; we only **add** a partition in free space and relocate Docker's data.

---

## Current state (2026-06, before any change)

```
Disk /dev/nvme0n1 — 512 GB (KINGSTON OM8PGP4512Q-A0), GPT
 p1   100M  vfat   EFI (SYSTEM)
 p2    16M  MSR
 p3   378G  ntfs   Windows 11  (BitLocker)   used 217G / free 162G   ← shrink source
 p4   500M  vfat   /boot
 p6    90G  ext4   /            used 67G / free 18G (80%)             ← target of relief
 p7   7.2G  swap
 p5   1.0G  ntfs   Recovery

On-disk order: p3(Windows) → p4(/boot) → p6(root) → p7(swap) → p5(recovery)
```

- `/var/lib/docker` = **44 GB** of the 67 GB used on root → moving it frees ~44 GB.
- Docker Root Dir: `/var/lib/docker` (default).
- Three live Docker stacks share this host: `crypto-bot` (has the only critical live data — Postgres), `voice_conversational_agent`, `brief_noticias`.

> **Immediate alternative (no partition work):** deleting `/opt/neuralcore/voice_conversational_agent/.venv` reclaims **6.3 GB** right now (it is `.dockerignore`d and never used by the image). Independent of this plan.

---

## Why Option A (not "grow root")

Shrinking Windows frees space at the **end of p3**, i.e. between Windows and `/boot` — **not adjacent to root** (`/boot` sits between). Growing root would require moving `/boot` and relocating the *start* of the ext4 filesystem — slow, offline (live USB), and risky. Option A sidesteps all of that: a brand-new partition in the freed gap, used for Docker. The root partition's bytes are never touched.

---

## Risks & safety

| Risk | Mitigation |
|---|---|
| Windows/BitLocker corruption on shrink | Shrink **from inside Windows** (Disk Management is BitLocker-aware). Never shrink the NTFS from Linux. |
| Partition-table edit | Back up the GPT first (`sgdisk --backup`). We only **add** a partition in free space; existing partitions are untouched. |
| Losing Docker images/volumes/DB | `rsync -aHAXS` preserves overlay2 hardlinks/xattrs/sparse files. Keep the old copy (`/var/lib/docker.old`) until verified. Postgres data lives in a Docker volume → it rides along with the move. |
| Bad fstab → boot drops to emergency | Use `nofail` in fstab; test-mount before rebooting. |
| Downtime | Docker stop = all three stacks down for the duration of the rsync (~minutes for 44 GB on NVMe). Plan a quiet window. crypto-bot is analysis-only and Hunter is paper, so no live trades are missed. |

**Before starting:** confirm a current Postgres dump exists as an extra safety net (we are NOT touching p6, but cheap insurance):
```bash
sudo docker compose -f /opt/crypto-bot/docker-compose.yml exec -T postgres \
  pg_dump -U cryptobot -Fc cryptobot > /home/ltaverna/cryptobot_predisk_$(date +%Y%m%d).dump
```

---

## Phase 0 — Prep (Linux, non-destructive)

```bash
# Record the starting layout for reference / rollback
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINT,UUID | tee ~/clean_storage_minipc/state_before.txt
sudo parted /dev/nvme0n1 unit GB print free | tee -a ~/clean_storage_minipc/state_before.txt

# Back up the GPT partition table (restore with: sgdisk --load-backup=FILE /dev/nvme0n1)
sudo sgdisk --backup=/home/ltaverna/clean_storage_minipc/nvme0n1-gpt.bak /dev/nvme0n1
```

---

## Phase 1 — Shrink Windows (done FROM Windows)

1. Reboot into **Windows 11**.
2. Open **Disk Management** (`diskmgmt.msc`).
3. Right-click the **C:** volume → **Shrink Volume…**
4. Enter amount to shrink: **≈ 100000 MB (100 GB)** (leave generous headroom in Windows; it still keeps ~62 GB free + the unshrunk used space).
5. Confirm. Windows leaves ~100 GB **Unallocated** after C:. **Do NOT create a Windows partition in it** — leave it raw.
6. Reboot back into **Linux**.

> If Disk Management refuses to shrink that far (immovable files), shrink whatever it allows, or temporarily disable hibernation / pagefile / system protection, or suspend BitLocker (`manage-bde -protectors -disable C:`) — but the BitLocker-aware shrink usually works as-is.

---

## Phase 2 — Create + format the new partition (Linux)

```bash
# 1. Confirm the new free gap appeared (look for a ~100 GB "Free Space" between p3 and p4)
sudo parted /dev/nvme0n1 unit GB print free

# 2. Create the partition IN THE FREE GAP. Using parted with the gap's start/end (GB)
#    Replace <START_GB> and <END_GB> with the gap boundaries reported above.
#    Example if the gap is 306GB→406GB:
sudo parted -a optimal /dev/nvme0n1 mkpart docker-data ext4 <START_GB>GB <END_GB>GB

# 3. Re-read the partition table (safe: only a NEW partition in free space was added)
sudo partprobe /dev/nvme0n1
lsblk -o NAME,SIZE,FSTYPE /dev/nvme0n1     # note the new partition name, e.g. nvme0n1p8

# 4. Format ext4 with a label (replace pX with the new partition, e.g. p8)
sudo mkfs.ext4 -L docker-data /dev/nvme0n1pX

# 5. Record its UUID — needed for fstab
sudo blkid /dev/nvme0n1pX
```

---

## Phase 3 — Move Docker onto the new partition (Linux)

```bash
# 1. Stop Docker (stops ALL containers across all stacks)
sudo systemctl stop docker docker.socket

# 2. Mount the new partition at a staging path
sudo mkdir -p /mnt/docker-data
sudo mount /dev/nvme0n1pX /mnt/docker-data

# 3. Copy Docker's data, preserving overlay2 hardlinks/xattrs/sparse files
sudo rsync -aHAXS --numeric-ids --info=progress2 /var/lib/docker/ /mnt/docker-data/

# 4. Sanity: sizes should match closely
sudo du -sh /var/lib/docker /mnt/docker-data

# 5. Swap the directory in
sudo umount /mnt/docker-data
sudo mv /var/lib/docker /var/lib/docker.old
sudo mkdir /var/lib/docker

# 6. Persist the mount. Append to /etc/fstab (use the UUID from Phase 2 step 5):
echo 'UUID=<NEW_UUID>  /var/lib/docker  ext4  defaults,nofail  0  2' | sudo tee -a /etc/fstab

# 7. Mount it and start Docker
sudo mount /var/lib/docker
sudo systemctl start docker
```

---

## Phase 4 — Verify, then reclaim

```bash
# Docker is on the new partition and sees everything
sudo docker info --format 'Root: {{.DockerRootDir}}'
df -h /var/lib/docker            # should be the new ~100G partition
sudo docker ps -a                # all containers present
sudo docker images              # all images present

# Bring the stacks up / confirm health
sudo docker compose -f /opt/crypto-bot/docker-compose.yml ps      # postgres healthy?
# (voice + brief_noticias come up via their own restart policies)
```

**Only after confirming all three stacks run correctly (give it a few minutes / a container restart):**

```bash
sudo rm -rf /var/lib/docker.old   # frees ~44 GB on the root partition
df -h /                            # root usage should drop sharply
```

---

## Rollback (if anything goes wrong before deleting docker.old)

```bash
sudo systemctl stop docker docker.socket
sudo umount /var/lib/docker
sudo rmdir /var/lib/docker
sudo mv /var/lib/docker.old /var/lib/docker
# remove the fstab line we added, then:
sudo systemctl start docker
```
Docker is back exactly as before. The new partition can be left empty or deleted later.
GPT can be restored from the backup: `sudo sgdisk --load-backup=/home/ltaverna/clean_storage_minipc/nvme0n1-gpt.bak /dev/nvme0n1`.

---

## Verification checklist

- [ ] Postgres dump taken (insurance)
- [ ] GPT backed up (`nvme0n1-gpt.bak`)
- [ ] Windows shrunk; ~100 GB unallocated gap visible in `parted print free`
- [ ] New ext4 partition created + formatted + UUID recorded
- [ ] `rsync` completed; `du -sh` sizes match
- [ ] fstab entry added with `nofail`; `mount /var/lib/docker` works
- [ ] `docker ps -a` + `docker images` show everything; Postgres healthy
- [ ] `/var/lib/docker.old` removed; root usage dropped ~44 GB
- [ ] (optional) old `.venv` deleted for +6.3 GB

---

## Division of labour

- **You (in Windows):** Phase 1 — the Disk Management shrink. BitLocker makes this Windows-side only.
- **Me (in Linux, with your go at each gate):** Phases 0, 2, 3, 4 — partition create, Docker move, verify, reclaim. I will not run any destructive command without confirming the gap/UUID/device names against live output first.
