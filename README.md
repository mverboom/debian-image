# debian-image

A small toolkit for building, verifying, and deploying **bare-metal Debian
root filesystems** from a tarball. The rootfs is produced inside a Docker
container, exported as a (split) tar archive, optionally tested, and then
unpacked onto a target disk and made bootable with the rEFInd boot manager.

The toolkit is made of four bash scripts:

| Script | Purpose | Runs where |
| --- | --- | --- |
| `create-debian-rootfs` | Build a Debian rootfs tarball (with optional kernel, ZFS, PVE, rEFInd) | Host, uses Docker |
| `_cleanup` | Configure & clean the rootfs *inside* the container | Inside the container (bind-mounted, not run directly) |
| `test-rootfs` | Verify a produced tarball has the expected structure/files | Host |
| `setup-efi` | Set up the ESP and install rEFInd on a target disk | Host (needs root) |

`_cleanup` is an internal companion script — you never invoke it yourself.
`create-debian-rootfs` bind-mounts it into the build container.

## Overview of the workflow

```
 create-debian-rootfs  ──►  *.tar.gz (+ *-boot.tar.gz)
                               │
                               ├── test-rootfs   (optional sanity check)
                               │
                               └── unpack onto target disk
                                       │
                                       └── setup-efi  (install rEFInd on the ESP)
                                               │
                                               └── reboot
```

When a kernel is requested (`-k`, `-r`, `-z`, `-B`, or `-P`), the output is
**split into two tarballs** so `/` and `/boot` can be deployed as separate
filesystems:

- `<name>-rootfs.tar.gz` — the whole rootfs **minus** the contents of `/boot`
  (the empty `/boot` mount point is kept).
- `<name>-boot.tar.gz` — the contents of `/boot` **without** the `boot/`
  prefix, so it can be unpacked directly into a mounted `/boot`.

Without a kernel, a single `<name>-rootfs.tar.gz` containing an empty `/boot`
is produced.

## Requirements

- **Docker** (daemon running; the user must be able to pull and run containers).
- A network connection (packages and keyrings are downloaded from Debian /
  Proxmox mirrors).
- For `setup-efi`: root privileges, `parted`, `mkfs.vfat`, `efibootmgr`, and a
  target block device.
- For `test-rootfs`: nothing beyond `tar`, `find`, `awk`, etc. (no root needed;
  ownership checks are done from tar headers, not the filesystem).

### Proxy support

`docker pull` is performed by the **Docker daemon**, which does *not* inherit
your shell's proxy variables. If you are behind a proxy:

1. Configure the daemon's proxy (systemd drop-in or `~/.docker/config.json`),
   then restart Docker.
2. The build script also forwards your shell's `http_proxy` / `https_proxy` /
   `no_proxy` / `all_proxy` / `ftp_proxy` (and uppercase variants) *into the
   container* so `apt-get` and `wget` can reach mirrors through the proxy.

---

## `create-debian-rootfs`

Build a Debian root filesystem tarball using Docker.

### Usage

```
./create-debian-rootfs [OPTIONS] <codename>
```

`<codename>` is a Debian suite name, e.g. `bookworm`, `bullseye`, `trixie`,
`sid`, `stable`, `testing`, `unstable`, `oldstable`.

### Options

| Option | Description |
| --- | --- |
| `-o, --output FILE` | Output filename for the rootfs tarball. Default: `debian-<codename>-rootfs.tar.gz` (or `pve-<codename>-rootfs.tar.gz` with `-P`). A second `-boot` tarball is auto-generated when a kernel is included. |
| `-O, --output-dir DIR` | Output directory; relocates the (default or `-o`) filename into it. The directory is created if missing. When combined with `-o`, only the basename of `-o` is kept. |
| `-a, --arch ARCH` | Target architecture (`amd64`, `arm64`, `armhf`, …). Default: `amd64`. |
| `-k, --with-kernel` | Include the Linux kernel and an initramfs in `/boot`. Produces a split rootfs+boot tarball. |
| `-r, --with-refind` | Include the rEFInd boot manager. **Implies `-k`.** |
| `--esproot DIR` | Directory where the ESP is mounted inside the rootfs (default: `/boot/efi`). Controls where rEFInd files are staged and the layout of the boot tarball. |
| `-B, --backports` | Pull the kernel (and headers, and optionally ZFS) from `<codename>-backports` for a newer kernel. **Implies `-k`.** |
| `-P, --pve` | Install **Proxmox VE** on top of Debian (uses the PVE kernel which already includes ZFS). Mutually exclusive with `-B` and `-z`. **Implies `-k`.** |
| `-z, --with-zfs` | Include ZFS support (DKMS module, userland, initramfs hook, zed). **Implies `-k`.** |
| `-R, --root-password P` | Set the root password and unlock the account. Default leaves root locked. Use the literal value `unlock` to unlock root *without* a password (insecure; testing only). |
| `-p, --packages PKG` | Comma-separated list of extra packages to install on top of the defaults. |
| `-x, --compress TYPE` | Compression: `gzip` (default), `xz`, or `none`. |
| `-f, --force` | Overwrite existing output tarball(s) without prompting (for unattended builds). |
| `-S, --strip-boot-symlinks` | Remove the symlinks in `/boot/pve` from the boot tarball. Only valid with `-P`. Useful when `/boot` is on VFAT (which cannot store symlinks). |
| `-v, --verbose` | Enable verbose `[*]` log lines. |
| `-h, --help` | Show the help message. |

### Default packages

Every build installs a small base set: `systemd-sysv`, `openssh-server`,
`openssh-client`, `iproute2`, `iputils-ping`, `libcap2-bin`, `console-setup`,
`vim`, `lsb-release`, `gnupg`, plus `ifupdown` (Debian-only builds; PVE
manages networking itself so `ifupdown` is omitted for `-P`).

### Option use cases

- **`-o`** — name your build for a custom deployment:
  `./create-debian-rootfs -o myrouter-bookworm.tar.gz bookworm`.
- **`-O`** — write outputs to a shared build directory that other tooling
  watches, e.g. `./create-debian-rootfs -O /var/builds bookworm`.
- **`-a`** — build for an ARM64 SBC: `./create-debian-rootfs -a arm64 -k trixie`.
- **`-k`** — produce a bootable rootfs with a kernel for a target that has no
  other way to install one.
- **`-r`** — stage rEFInd inside the rootfs so `setup-efi` can copy it onto
  the ESP later.
- **`--esproot`** — use a non-standard ESP mount point, e.g. `/boot/esp`,
  when the target's firmware expects rEFInd there.
- **`-B`** — get a newer kernel than the suite ships (e.g. newer hardware
  support on `bookworm`).
- **`-P`** — build a Proxmox VE node image (PVE 8 on bookworm, PVE 9 on
  trixie) with its own kernel and built-in ZFS.
- **`-z`** — build a rootfs that can boot from / manage ZFS pools.
- **`-R`** — set a known root password so you can log in on first boot before
  SSH keys are deployed; `unlock` lets a serial-console session drop to a
  root shell with no password.
- **`-p`** — add your own must-have packages into the image, e.g.
  `-p "curl,git,tmux,rsync"`.
- **`-x xz`** — smaller tarball for archiving/transfer over slow links.
- **`-x none`** — uncompressed, fastest to unpack on the target.
- **`-f`** — overwrite yesterday's tarball in a nightly cron build.
- **`-S`** — make the PVE boot tarball VFAT-safe for a separate FAT32 `/boot`.
- **`-v`** — debug a failing build by following what the script does.

### Output files

Without `-k`/`-r`/`-z`/`-B`/`-P`:

```
debian-<codename>-rootfs.tar.gz        # complete rootfs incl. empty /boot
```

With a kernel (any of `-k`, `-r`, `-z`, `-B`, `-P`):

```
debian-<codename>-rootfs.tar.gz        # rootfs without /boot contents
debian-<codename>-boot.tar.gz          # /boot contents, no boot/ prefix
```

With `-P` the `debian-` prefix becomes `pve-`.

### Examples

Minimal rootfs (single tarball, no kernel):

```bash
./create-debian-rootfs bookworm
```

Rootfs + kernel (two tarballs):

```bash
./create-debian-rootfs -k bookworm
```

Rootfs + kernel + rEFInd:

```bash
./create-debian-rootfs -k -r bookworm
```

Newer kernel from backports:

```bash
./create-debian-rootfs -k -B bookworm
```

Kernel + ZFS support (enables contrib/non-free for DKMS):

```bash
./create-debian-rootfs -k -z bookworm
```

Proxmox VE 8 on Bookworm:

```bash
./create-debian-rootfs -P bookworm
```

Proxmox VE 9 on Trixie:

```bash
./create-debian-rootfs -P trixie
```

PVE with VFAT-safe `/boot` (symlinks stripped):

```bash
./create-debian-rootfs -P -S bookworm
```

Everything: backports kernel + ZFS + rEFInd:

```bash
./create-debian-rootfs -k -B -z -r bookworm
```

Custom name and architecture, xz-compressed:

```bash
./create-debian-rootfs -o rootfs.tar.xz -a arm64 trixie
```

Kernel + extra packages + set the root password:

```bash
./create-debian-rootfs -k -p "vim,curl" -R changeme bullseye
```

Unlock root with no password (testing only):

```bash
./create-debian-rootfs -k -R unlock trixie
```

Write outputs into `/var/builds`, using the default name:

```bash
./create-debian-rootfs -O /var/builds trixie
```

Custom name inside `/var/builds`:

```bash
./create-debian-rootfs -O /var/builds -o my-rootfs.tar.xz trixie
```

Unattended overwrite of an existing tarball:

```bash
./create-debian-rootfs -k -f bookworm
```

---

## `_cleanup` (internal)

This is the companion script that `create-debian-rootfs` bind-mounts into the
build container at `/root/cleanup.sh` and runs there. **Do not run it
directly.** All of its configuration is passed in via environment variables
(`-e` flags in `create-debian-rootfs`): `WITH_PVE`, `WITH_BACKPORTS`,
`WITH_ZFS`, `WITH_KERNEL`, `WITH_REFIND`, `CODENAME`, `ROOT_PASSWORD`,
`FULL_PKG_LIST`, `PVE_KEYRING_URL`, `PVE_SUITE`, `PVE_FORMAT`, `KERNEL_PKG`,
`HEADERS_PKG`, `BACKPORTS_KERNEL_PKGS`, `BACKPORTS_ZFS_PKGS`, `MAIN_PKG_LIST`,
`ESPROOT`.

It is responsible for:

- `apt-get update` / package installation (Debian, backports, ZFS repos, or
  the Proxmox repo depending on flags).
- Downloading and installing the Proxmox keyring; adding the
  `pve-no-subscription` repo (DEB822 for trixie+, legacy format otherwise);
  disabling the PVE enterprise repo; removing the stock Debian kernel and
  `os-prober` for PVE builds.
- Writing `RESUME=none` and `noresume` so the initramfs doesn't wait for a
  non-existent resume/swap device.
- Generating the initramfs for the installed kernel.
- Force-creating essential `/sbin/init`, `/sbin/halt`, `/sbin/reboot`, …
  symlinks that maintainer scripts sometimes skip in containers.
- Setting / unlocking the root password.
- Stripping Docker artifacts (`.dockerenv`, `Dockerfile`), apt caches/lists,
  `/tmp` and `/var/tmp`, hostname, `resolv.conf`, `machine-id`, logs, SSH host
  keys, udev persistent-net rules, user caches, bash history, `lost+found`.
- Re-creating essential mount-point and `if-up.d`/`if-down.d` directories.
- Staging rEFInd onto `${ESPROOT}/EFI/refind` when requested.

---

## `test-rootfs`

Verify a produced rootfs tarball by unpacking it into a temp dir and checking
for the expected files, directories, ownership, and permissions. Ownership is
read from tar headers (not the filesystem), so it works **without root**.

### Usage

```
./test-rootfs [OPTIONS] <rootfs-tarball> [<boot-tarball>]
```

The boot tarball is auto-detected from the rootfs filename when `-k` or `-r`
is given and the file exists.

### Options

| Option | Description |
| --- | --- |
| `-k, --with-kernel` | Expect a kernel (`vmlinuz-*`) and initramfs in `/boot`. |
| `-r, --with-refind` | Expect rEFInd files under `${ESPROOT}/EFI/refind` or `/usr/share/refind/refind`. |
| `--esproot DIR` | ESP mount point the build used (default: `/boot/efi`). |
| `-z, --with-zfs` | Expect ZFS userland (`zfs`, `zpool`), DKMS module/initramfs hook, and `zed`. |
| `-P, --pve` | Expect a PVE install: PVE kernel, no stock Debian kernel, pve-manager, the `pve-install-repo`, no `os-prober`, enterprise repo disabled, plus postfix/open-iscsi/chrony. |
| `-h, --help` | Show the help message. |

### What it checks

- Critical top-level directories (`bin`, `etc`, `lib`, `usr`, `var`, `proc`,
  `sys`, `dev`, `run`, `boot`).
- Critical binaries/config (`/bin/sh`, `/bin/bash`, `/etc/passwd`,
  `/etc/shadow`, `/etc/fstab`, `/etc/os-release`, `sshd`, `ssh`, …).
- No leftover Docker artifacts (`.dockerenv`, `Dockerfile`).
- Clean apt cache (no `.deb` files).
- Ownership/permissions from tar headers: `/etc/shadow` `0/42` mode `640`,
  `/etc/passwd` `0/0` mode `644`, `/usr/bin/su` setuid, `/tmp` sticky bit,
  presence of setuid/setgid files, ownership diversity, and that the boot
  tarball is all root-owned.
- `/etc/machine-id` is empty/absent.
- `/etc/os-release` identifies as Debian.
- Optional: kernel+initramfs, rEFInd, ZFS, PVE (see flags above).

Exits `0` if all checks pass, `1` if any fail.

### Option use cases

- **`-k`** — confirm the split boot tarball actually contains a kernel and
  initramfs before deploying.
- **`-r`** — confirm rEFInd was staged so `setup-efi` will find it.
- **`--esproot`** — match a non-default ESP path used during the build.
- **`-z`** — verify ZFS userland, module, initramfs hook, and zed are present.
- **`-P`** — verify a Proxmox image: PVE kernel present, stock kernel gone,
  repos configured, enterprise repo disabled, dependencies present.

### Examples

Test a minimal rootfs:

```bash
./test-rootfs debian-bookworm-rootfs.tar.gz
```

Test a split build (boot tarball given explicitly):

```bash
./test-rootfs -k debian-bookworm-rootfs.tar.gz debian-bookworm-boot.tar.gz
```

Test a build with kernel + rEFInd + ZFS (boot tarball auto-detected):

```bash
./test-rootfs -k -r -z debian-bookworm-rootfs.tar.gz
```

Test a Proxmox image:

```bash
./test-rootfs -P pve-bookworm-rootfs.tar.gz
```

---

## `setup-efi`

Set up the EFI System Partition (ESP) and install the rEFInd boot manager on a
target disk, after a Debian rootfs tarball (built with `-r`) has been unpacked.
Requires root for disk operations.

### Usage

```
./setup-efi [OPTIONS] <rootfs-dir>
```

`<rootfs-dir>` is the directory where you unpacked the rootfs tarball.

### Options

| Option | Description |
| --- | --- |
| `-d, --disk DISK` | Target disk (e.g. `/dev/sda`, `/dev/nvme0n1`) for creating/finding the ESP. |
| `-p, --esp-part NUM` | Use an existing ESP partition number on `--disk` instead of creating one. |
| `-e, --esp-dir DIR` | Use an already-mounted ESP directory (skips all disk operations). |
| `-s, --esp-size SIZE` | ESP size in MiB when creating a new one (default: `512`; minimum `100`). |
| `-n, --no-install` | Skip `efibootmgr` registration; just copy rEFInd files and write the config. |
| `--esproot DIR` | ESP mount point the build used (default: `/boot/efi`). Used both to locate rEFInd inside the rootfs and as the mount point for the ESP when using `--disk`. |
| `-h, --help` | Show the help message. |

### What it does

1. Detects or creates an ESP on the target disk (GPT label if none, FAT32,
   `boot`/`esp` flags), or uses an existing partition (`-p`), or an
   already-mounted directory (`-e`).
2. Detects the latest kernel and matching initramfs in the rootfs.
3. Copies rEFInd (EFI binary, sample config, icons, tools) from the rootfs to
   `${ESP_DIR}/EFI/refind`.
4. Generates a `refind.conf` pointing at the detected kernel/initramfs, with
   `noresume` and a placeholder `root=ROOT_PART` you must edit.
5. Registers rEFInd as the default EFI boot entry via `efibootmgr` (unless
   `-n`).
6. Unmounts the ESP if it mounted it itself (EXIT trap).

### Option use cases

- **`-d`** — let the script partition and format a fresh ESP on a blank disk.
- **`-p`** — reuse an ESP partition you already created (e.g. partition #1)
  without re-partitioning the disk.
- **`-e`** — point at an ESP you've already mounted elsewhere, e.g. for
  removable media or a multi-boot ESP you don't want to touch.
- **`-s`** — choose a larger/smaller ESP (e.g. `-s 1024` if you'll store
  multiple kernels or bootloaders).
- **`-n`** — copy rEFInd files only, e.g. for a removable USB stick where you
  want the firmware's boot picker to find `\EFI\BOOT\BOOTX64.EFI` instead of a
  registered NVRAM entry.
- **`--esproot`** — match a non-default ESP path used at build time.

### Examples

Auto-create an ESP on `/dev/sda` and install rEFInd for an unpacked rootfs:

```bash
sudo ./setup-efi -d /dev/sda /mnt/rootfs
```

Reuse an existing ESP partition (#1) on `/dev/sda`:

```bash
sudo ./setup-efi -d /dev/sda -p 1 /mnt/rootfs
```

Use an already-mounted ESP at `/mnt/efi`:

```bash
sudo ./setup-efi -e /mnt/efi /mnt/rootfs
```

Create a 1 GiB ESP on an NVMe disk, with a non-default ESP path:

```bash
sudo ./setup-efi -d /dev/nvme0n1 -s 1024 --esproot /boot/esp /mnt/rootfs
```

Copy rEFInd files only (no NVRAM registration), e.g. for removable media:

```bash
sudo ./setup-efi -d /dev/sdb -n /mnt/rootfs
```

---

## End-to-end examples

### 1. Minimal bookworm image, tested

```bash
# Build
./create-debian-rootfs bookworm

# Verify
./test-rootfs debian-bookworm-rootfs.tar.gz

# Deploy (no kernel, so no setup-efi step)
sudo mkdir -p /mnt/rootfs
sudo tar --extract --file debian-bookworm-rootfs.tar.gz --directory /mnt/rootfs
```

### 2. Bootable Bookworm with kernel + rEFInd, deployed to `/dev/sda`

```bash
# Build with kernel and rEFInd
./create-debian-rootfs -k -r bookworm

# Verify both tarballs
./test-rootfs -k -r debian-bookworm-rootfs.tar.gz

# Deploy rootfs and /boot separately
sudo mkdir -p /mnt/rootfs
sudo tar --extract --file debian-bookworm-rootfs.tar.gz --directory /mnt/rootfs
sudo mkdir -p /mnt/rootfs/boot
sudo tar --extract --file debian-bookworm-boot.tar.gz --directory /mnt/rootfs/boot

# Install rEFInd on the ESP (script partitions /dev/sda for you)
sudo ./setup-efi -d /dev/sda /mnt/rootfs

# Edit refind.conf to replace ROOT_PART with your root partition's UUID
sudo nano /mnt/rootfs/boot/efi/EFI/refind/refind.conf   # or the ESP path printed
blkid   # find the UUID

# Reboot and select "rEFInd" from the firmware boot menu
```

### 3. Proxmox VE 8 image with VFAT-safe `/boot`, tested, deployed

```bash
# Build PVE image, strip /boot/pve symlinks for a FAT32 /boot
./create-debian-rootfs -P -S bookworm

# Verify it's a proper PVE image
./test-rootfs -P pve-bookworm-rootfs.tar.gz

# Deploy
sudo mkdir -p /mnt/rootfs
sudo tar --extract --file pve-bookworm-rootfs.tar.gz --directory /mnt/rootfs
sudo mkdir -p /mnt/rootfs/boot
sudo tar --extract --file pve-bookworm-boot.tar.gz --directory /mnt/rootfs/boot

# Install rEFInd (reuses or creates an ESP on /dev/sda)
sudo ./setup-efi -d /dev/sda /mnt/rootfs
```

### 4. ARM64 Trixie with xz compression, written to a build directory

```bash
./create-debian-rootfs -O /var/builds -o trixie-arm64.tar.xz -a arm64 -k -r trixie
./test-rootfs -k -r /var/builds/trixie-arm64.tar.xz
```

### 5. Unattended nightly rebuild (cron)

```bash
0 3 * * *  /opt/debian-image/create-debian-rootfs -k -f -O /var/builds bookworm >> /var/log/debian-image.log 2>&1
```

---

## Notes & caveats

- `create-debian-rootfs` reaps leftover `debian-rootfs-build-*` containers from
  interrupted previous runs at startup, and removes its own container via an
  EXIT/INT/TERM/HUP/QUIT trap.
- The split layout deliberately keeps `/boot` as an empty mount point in the
  rootfs tarball so you can mount a separate `/boot` partition onto it.
- `/etc/shadow` ownership (`root:shadow`, `0/42`) and setuid bits are
  preserved by manipulating the tar stream directly rather than extracting to
  disk, which a non-root build user could not preserve.
- For PVE builds, `ifupdown` is **not** installed (PVE manages networking);
  the PVE enterprise repository is disabled in favor of `pve-no-subscription`.
- `setup-efi` always tells you to edit `ROOT_PART` in `refind.conf`; it cannot
  guess your root partition. Use `blkid` to find the UUID.