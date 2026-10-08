# ESP relocation on BigStorage (sda) — full guide

> Goal: move the EFI partition out from between NTFS and EndeavourOS so a
> later NTFS shrink flows straight into EndeavourOS with no blocker.
> Status: PLANNED, not started. Nothing below has been executed.
> Danger level: HIGH for steps 5–6 (bulk data move). Everything else is
> reversible file/config work. Read the whole file before starting.
> Externally double-checked (grub-install/efibootmgr dedup, FAT32 copy
> flags, GPT renumber guard, sequential GParted applies — all folded in).

---

## 0. Paste-ready brief for another AI

```
HARDWARE: Acer Nitro AN515-57, UEFI boot, Secure-Boot behavior unknown
(boots grubx64.efi directly today — keep the same loader file and mode).
Two disks, touch ONLY /dev/sda (1.5T SATA INTEL SSDSC2BX016T4R):
  sda1  16M   Microsoft reserved (LEAVE ALONE)
  sda2  1.2T  NTFS "BigStorage", PARTUUID a5296ac4-666e-4844-beed-422bd2d67ea2,
        UUID A6BAFBFEBAFBC937, 166G used / 1.1T free, mounted in Linux at
        /run/media/anato/BigStorage. Shared with Windows (on nvme0n1).
  sda3  2G    vfat ESP, PARTLABEL "EFI", UUID DFAC-A594,
        PARTUUID 2a60b285-f448-4e3c-be85-4a95b4b1bfa5, mounted /boot/efi.
        Contents: 328K total — EFI/boot/bootx64.efi + EFI/endeavouros/grubx64.efi.
        THIS IS THE LIVE BOOT PATH (BootCurrent 0003, see below).
  sda4  254G  ext4 "endeavouros" / (UUID 794d9d51-effd-4af4-979a-064d39a001a9,
        PARTUUID dc97ad9f-38dd-497f-8e83-d0e0e0e2a05790), ~109G used.
        Holds /boot (kernels linux 7.2.8 + linux-lts 6.18.55), /swapfile 32G
        (hibernate resume=UUID=794d9d51… resume_offset=53663744 — offsets
        assume sda4's START never changes until step 6).
        Zero free sectors after sda4 (disk ends at 1490.4 GiB).
DO NOT TOUCH: /dev/nvme0n1 (Windows: 100M ESP, Acer NTFS, recovery),
  sda1, ubuntu NVRAM entries 0000-0002 (stale, harmless).
BOOT: efibootmgr BootCurrent 0003 = SATA disk entry pointing at sda
  partition 3 (HD(3,GPT,2a60b285-…)) — whole-disk GRUB boot, no \EFI path.
  BootOrder 0003,0004,2001,2002,2003,0000,0001,0002. GRUB (blackice theme,
  os-prober ON, timeout 5) chainloads Windows on nvme0n1p1. /etc/grub.d/
  31_efi_bootnext is chmod -x (leave it). fstab mounts ESP by UUID=DFAC-A594.
DESKTOP: Hyprland Wayland, LUKS none, plain ext4 — no encryption
  complications. sddm greeter, NetworkManager, no RAID/LVM.
ASSUMPTIONS TO VERIFY LIVE: Windows fully shut down (no Fast Startup
  hibernation lock, no BitLocker on BigStorage — check first), laptop on
  AC + charged battery, backups done (phase 1).
```

---

## 1. Prep — from the running EndeavourOS (with sudo)

```bash
sudo cp -a /boot/efi /root/esp-backup-$(date +%F)     # 328K, the actual boot files
sudo efibootmgr -v > ~/esp-backup-efibootmgr.txt     # current boot entries
cp /etc/fstab ~/esp-backup-fstab.txt
sudo cp /boot/grub/grub.cfg ~/esp-backup-grub.cfg
lsblk -o NAME,SIZE,FSTYPE,LABEL,PARTUUID,MOUNTPOINTS /dev/sda | tee ~/esp-backup-lsblk.txt
# Copy these files to the Ventoy stick (8G, already flashed with the
# EndeavourOS live ISO — Tier-0 is kilobytes, fits easily; a Tier-1 sda4
# image at 60–90G does NOT fit, keep it on BigStorage/external only) AND
# to another machine. In the live session the Ventoy partition itself is
# mountable for reading them back — no network needed. Verify GParted
# exists in the live session (`which gparted`); if missing,
# `sudo pacman -Sy gparted` needs live internet.
```

Windows-side preconditions (do these IN Windows before anything else):

- `manage-bde -status D:` (whatever letter BigStorage is) → must show
  **no BitLocker**. If encrypted: decrypt fully first or ABORT.
- Power options → **Fast Startup OFF**. Then, from an ELEVATED prompt:
  `chkdsk D: /f` (allow the dismount; a pending Windows update leaves an
  uncommitted `$LogFile` that makes `ntfsresize` abort — chkdsk clears it),
  one final regular boot into Windows, then the final exit via
  `shutdown /s /t 0` (full shutdown that bypasses Fast Startup even if it
  ever gets re-enabled). Never reboot-or-hibernate out of Windows here.
- Decide the shrink target now (1.1T free on BigStorage — even -500G
  leaves 600G headroom).

---

## 2. Live USB session 1 — make room + transplant the boot

Boot any EndeavourOS/Arch live USB. FIRST, in a terminal, nail down which
node is the internal SSD — a Ventoy stick can steal `/dev/sda`, pushing
the SATA drive to `/dev/sdb` depending on USB enumeration order:
```bash
lsblk -o NAME,SIZE,MODEL,LABEL   # want: INTEL SSDSC2BX016T4R
ls /dev/disk/by-id/ | grep -i intel
```
Prefer the stable path (`/dev/disk/by-id/ata-INTEL_SSDSC2BX016T4R_*`)
over bare `/dev/sdX` in EVERY command below. If this guide says sda and
your terminal says otherwise, translate once here and stay consistent.
Then: mask live-session sleep (live ISOs suspend after ~10–15 min idle
and a mid-copy suspend corrupts the move — do this BEFORE GParted):
```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```
Open GParted (target the confirmed node only).
Confirm `mount | grep -E "sda|sdb"` shows NOTHING mounted for it (unmount
BigStorage if the file manager auto-mounted it) and run `swapoff -a`
(the live session must never activate sda4's swapfile while resizing).

1. **Shrink sda4 from its END by 2048 MiB** (254.0 → ~252 GiB). Offline
   ext4 shrink of the tail only — minutes, low risk. This creates 2 GiB
   free at the very end of the disk.
2. **Create new ESP** in that free space: FAT32, 1024 MiB, flag/label as
   EFI (`PARTLABEL=EFI`), type EF00. It becomes **sda5**. (~1 GiB
   leftover at disk end stays unallocated spare — harmless.)
3. **Copy boot files** (FAT32 has no POSIX metadata — plain `cp -r`,
   NOT `cp -a`, or harmless ownership warnings spam the copy):
   `cp -r <oldESP>/EFI <newESP>/` — expect exactly
   `EFI/boot/bootx64.efi` + `EFI/endeavouros/grubx64.efi`.
   Verify with `diff -r <oldESP>/EFI <newESP>/EFI` (must be silent).
4. **Chroot rewiring** (adjust `sda5` if numbering differs):
    ```bash
    mount /dev/sda4 /mnt
    mount /dev/sda5 /mnt/boot/efi
    NEWUUID=$(blkid -s UUID -o value /dev/sda5)
    sed -i "s/UUID=DFAC-A594/UUID=$NEWUUID/" /mnt/etc/fstab
    grep efi /mnt/etc/fstab   # must show the NEW uuid
    arch-chroot /mnt grub-install --target=x86_64-efi \
      --efi-directory=/boot/efi --bootloader-id=endeavouros --no-nvram
    arch-chroot /mnt grub-mkconfig -o /boot/grub/grub.cfg
    ```
    (`--no-nvram`: grub-install otherwise writes its own firmware entry
    and step 5 would create a DUPLICATE. Entry creation stays manual so
    the label and BootOrder are exactly as specified below.)
5. **New firmware entry, keep the old one:**
    ```bash
    efibootmgr --create --disk /dev/sda --part 5 \
      --label "EndeavourOS-ESP2" --loader /EFI/endeavouros/grubx64.efi
    efibootmgr -v   # note the new Boot number, e.g. 0007
    efibootmgr -o 0007,0003,0004,2001,2002,2003,0000,0001,0002
    ```
6. Reboot, enter firmware boot menu if needed, boot entry 0007.
   Verify ALL THREE before proceeding:
   `efibootmgr | grep BootCurrent` (must be the NEW number),
   `findmnt /boot/efi` (must show the NEW uuid),
   normal desktop + Windows still chainloads from GRUB.
   If anything fails: reboot, pick old entry 0003 — untouched, still works.
   Acer/InsydeH2O note: this firmware family sometimes wipes custom NVRAM
   entries when it notices GPT changes. Extra safety net already in place:
   step 3 copied `EFI/boot/bootx64.efi`, so even if entry 0007 vanishes,
   selecting the SATA SSD itself in the F12 menu still boots GRUB via the
   UEFI fallback path. If that happens, just re-create the entry.

---

## 3. Live USB session 2 — remove blocker, grow root

Only after session-1 verification passes.

1. Boot live USB again. Re-verify the device node (Ventoy shuffle — see
   §2), mask sleep targets again, confirm nothing on the SSD is mounted
   and `swapoff -a` has run. Confirm the machine was FULLY SHUT DOWN coming here (poweroff — never
   boot live media out of a suspend or hibernate; a stale hibernate image
   plus partition changes is the one unrecoverable combination).
2. **Delete old sda3** (2G ESP) → **Apply immediately**, then verify the
   gap. GPT numbers now go sparse (1,2,4,5) — NEVER run "Sort Partition
   Numbers" (gdisk/GParted): renumbering sda5→sda3 would invalidate the
   firmware entry from session 1, which addresses the ESP by partition
   number.
3. **Shrink sda2 NTFS from its END** by your chosen amount → **Apply**
   (GParted `ntfsresize` path; hours only if it were full — at 14% used
   it's fast). If it refuses: NTFS is dirty → back to Windows, `chkdsk`,
   clean shutdown, repeat. A failure HERE aborts cleanly with sda4
   untouched — this ordering is deliberate.
4. **Move sda4 LEFT** to fill the gap → **Apply** (GParted move = copies
   ~109G used, expect 20–60 min. AC plugged + battery charged +
   DO NOT INTERRUPT. Power loss here = restore-from-backup territory).
5. **Extend sda4 RIGHT** into any leftover tail space → **Apply**
   (forward extend = seconds, safe).
6. Reboot into EndeavourOS. Verify: `df -h /`, `swapon --show`,
   `filefrag -v /swapfile | head -4` (first-block offset must still read
   the resume_offset from §0 — the move preserves filesystem-internal
   offsets, confirm rather than assume),
   `cat /proc/cmdline` (resume line must still be there — sda4's
   _contents_ moved but its UUID is unchanged, and resume_offset is a
   filesystem-block offset, unaffected by partition position),
   `systemctl hibernate` test if you dare (it works — see skills).
7. Cleanup (optional): `efibootmgr -b 0003 -B` to remove the old entry,
   likewise stale `ubuntu*` 0000–0002. Only when the new layout has
   survived a week.

---

## 1.5 Backup phase (do this before any live USB work)

Situation: NO backup tools installed, NO external disk attached, remote
server has 3.8G free (useless for this). Root is 141G used (includes the
32G `/swapfile` — exclude it from every backup, it re-creates itself).

Tier 0 — mandatory, 10 minutes (covers all file/config mistakes):

```bash
sudo cp -a /boot/efi /root/esp-backup-$(date +%F)
sudo efibootmgr -v > ~/esp-backup-efibootmgr.txt
cp /etc/fstab ~/esp-backup-fstab.txt
sudo cp /boot/grub/grub.cfg ~/esp-backup-grub.cfg
pacman -Qe > ~/esp-backup-packages.txt        # explicit package list = reinstall recipe
sudo tar -cJf /run/media/anato/BigStorage/anato-etc-backup.tar.xz /etc  # configs
```

Copy the four `~` files + this guide to a USB stick AND photograph the
GParted layout with your phone before pressing Apply. Verify each archive
(`tar -tJf ... | head`, `cat` the txt files).

Tier 1 — the sda4 move insurance (covers the §3 catastrophe case):
best = full partition image to EXTERNAL media via Clonezilla live or
`partclone.ext4 -c -s /dev/sda4 -o /mnt/external/sda4.img` (~60–90G
compressed). If no external disk is available: `fsarchiver savefs` the
root onto BigStorage itself —
`sudo fsarchiver savefs /run/media/anato/BigStorage/sda4-root.fsa /dev/sda4 -e /swapfile`
(install `fsarchiver` first; same-disk image survives software/table
accidents when read back from live USB, but NOT physical disk death —
know which risk you're accepting).
Minimum viable if time-pressed: irreplaceable `/home` data (not caches,
not Steam libraries) onto BigStorage + the Tier-0 set. Then a worst case
costs you reinstall time, not your files.

Error catalogue (why each backup exists):

- Wrong-disk/partition targeting (§2–3): always confirm `/dev/sda` +
  PARTLABEL/PARTUUID triple (`Basic data`/`EFI`/`endeavouros`) before
  every destructive click. nvme0n1 is NEVER touched in this operation.
- Deleting sda2/sda4 instead of sda3: guarded by size check (target is
  exactly the 2G `EFI` one) + the phone photo.
- NTFS refuses shrink: dirty flag from Windows Fast Startup/hibernation
  — refusal is SAFE (no damage), go back to Windows, chkdsk, clean
  shutdown, retry. Never force it.
- fstab UUID typo / grub-install to wrong ESP: boot drops to emergency
  shell or old entry — fix from live USB with Tier-0 files. Never delete
  boot entry 0003 until §3-step-7.
- Power loss during the sda4 leftward move (~109G+, 20–60 min): the ONLY
  potentially total-loss event → Tier-1 image is its antidote. Live ISO
  sleep MUST be disabled; AC + full battery mandatory.
- Post-op Windows "repair" prompts: let chkdsk run on BigStorage if
  asked; decline anything offering to touch Linux partitions.
- Move transparency (no action needed, just know): GParted moves keep
  filesystem UUIDs, so fstab/GRUB/`resume=`/`resume_offset=` all survive
  the move untouched — position changes, identity doesn't.

## 4. Rollback matrix

| Failure point               | Recovery                                             |
| --------------------------- | ---------------------------------------------------- |
| New ESP won't boot          | Boot old entry 0003 (kept until step 7)              |
| fstab wrong UUID            | Live USB → mount → fix UUID from `blkid`             |
| grub-install failed         | Old entry still boots; retry chroot carefully        |
| NTFS won't shrink           | Dirty flag → Windows chkdsk + clean shutdown         |
| Power loss during sda4 move | Restore sda4 from backup; ESPs/boot entries intact   |
| Anything structural         | `esp-backup-*` files + this doc + a fresh AI with §0 |

## 5. Lower-risk alternative (no data move at all)

Skip §3 steps 4–5. After deleting sda3 + shrinking NTFS, format the freed
gap as a new partition for `/home` (or data): migrate `/home`, add one
fstab line, done. Root stays 254G forever, zero bulk copy. Say the word
and this file gets a §3-alt.
