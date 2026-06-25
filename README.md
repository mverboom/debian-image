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

Implementation details (the `_cleanup` script, what `test-rootfs` checks,
what `setup-efi` does internally, caveats, architecture mapping, and project
layout) live in [INTERNALS.md](INTERNALS.md).

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
- **Bash** 4.0+ (tested with bash 5.x).
- A network connection (packages and keyrings are downloaded from Debian /
  Proxmox mirrors).
- Sufficient disk space for the build and the resulting tarball(s):
  ~55 MB for a minimal image, ~170 MB with a kernel, ~300 MB with ZFS (DKMS
  build also needs room for the compiler toolchain inside the container).
- For `setup-efi`: root privileges, `parted`, `mkfs.vfat`, `efibootmgr`, and a
  target block device.
- For `test-rootfs`: nothing beyond `tar`, `find`, `awk`, etc. (no root needed;
  ownership checks are done from tar headers, not the filesystem).

### Proxy support

If you build behind an HTTP/HTTPS proxy, `create-debian-rootfs` forwards the
standard proxy environment variables from your shell **into the build
container** so that `apt-get` and `wget` (used for packages and keyring
downloads) can reach the network through the proxy. The following variables are
forwarded **only when set** (both lowercase and uppercase variants are
checked):

| Variable | Purpose |
| --- | --- |
| `http_proxy` / `HTTP_PROXY` | Proxy for HTTP requests |
| `https_proxy` / `HTTPS_PROXY` | Proxy for HTTPS requests |
| `ftp_proxy` / `FTP_PROXY` | Proxy for FTP requests |
| `no_proxy` / `NO_PROXY` | Hosts/CIDRs that bypass the proxy |
| `all_proxy` / `ALL_PROXY` | Proxy for other protocols (e.g. SOCKS) |

Proxy values are passed into the container only — they are **not** written
into the rootfs tarball and **not** printed in the logs. Proxy URLs may
contain credentials, so only the variable *names* are logged in verbose mode.

The base image pull (`docker pull debian:<codename>`) is performed by the
**Docker daemon**, which does *not* inherit your shell's proxy variables. If
your daemon cannot reach the registry, configure the daemon's proxy instead,
e.g. via a systemd drop-in:

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf >/dev/null <<'EOF'
[Service]
Environment="HTTP_PROXY=http://corp-proxy:8080"
Environment="HTTPS_PROXY=http://corp-proxy:8080"
Environment="NO_PROXY=localhost,127.0.0.1"
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

Alternatively, set the `proxies` section in `~/.docker/config.json` (see the
Docker documentation). Either approach also injects proxy settings into
containers automatically, which complements the explicit passthrough above.

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

### Backports kernel (`-B` / `--backports`)

`--backports` adds the Debian backports repository and installs the kernel
(and headers, if ZFS is enabled) from it, giving you a newer kernel than the
suite ships. For example, on Debian 12 (bookworm):

- **Standard kernel**: `linux-image-amd64` → 6.1.x
- **Backports kernel**: `linux-image-amd64` from `bookworm-backports` → 6.12.x

Backports also supplies ZFS packages from backports when `--with-zfs` is used,
providing newer ZFS versions.

### ZFS support (`-z` / `--with-zfs`)

`--with-zfs` installs OpenZFS support:

- **zfs-dkms** — ZFS kernel module (built via DKMS for the installed kernel).
- **zfsutils-linux** — userland utilities (`zfs`, `zpool`, …).
- **zfs-zed** — ZFS Event Daemon.
- **zfs-initramfs** — ZFS support in initramfs (for booting from ZFS).
- **linux-headers-\*** — kernel headers needed for DKMS compilation.

ZFS requires the `contrib` and `non-free` repository components, which are
enabled automatically. Building ZFS with DKMS compiles the kernel module
inside the container, which takes several minutes and pulls in build tools
(gcc, make, …). The resulting rootfs tarball is significantly larger (~280 MB
vs ~160 MB without ZFS).

### Proxmox VE (`-P` / `--pve`)

`--pve` installs Proxmox VE on top of the Debian base system — its own kernel,
management tools, and web interface. It is mutually exclusive with `-B`
(backports) and `-z` (ZFS): the PVE kernel already includes ZFS support
built-in, so separate DKMS packages and backports kernels are not needed. The
ZFS userland tools are still required and are installed from the PVE repository
(not via DKMS).

What gets installed: `proxmox-default-kernel` (PVE kernel, includes the ZFS
kernel module and AppArmor), `proxmox-ve` (meta-package: QEMU, LXC, management
tools, web GUI), `zfsutils-linux` and `zfs-zed` and `zfs-initramfs` (from the
PVE repo, no DKMS), `postfix` (required for PVE notifications; configured
non-interactively), `open-iscsi` (required by PVE), and `chrony` (NTP,
recommended by PVE). What gets removed: `linux-image-amd64` (the stock Debian
kernel — PVE ships its own) and `os-prober` (it scans VM partitions and can
create unwanted boot entries).

The `pve-no-subscription` repository is added automatically; the
`pve-enterprise` repository that `proxmox-ve` creates by default is
**disabled** (renamed to `.disabled`) since it requires a paid subscription
key. The repository format and keyring are chosen by Debian version:

| Debian | PVE version | Repo format | Keyring |
| --- | --- | --- | --- |
| Trixie (13) | PVE 9 | deb822 `.sources` | `proxmox-archive-keyring-trixie.gpg` |
| Bookworm (12) | PVE 8 | legacy `.list` | `proxmox-release-bookworm.gpg` |
| Bullseye (11) | PVE 7 | legacy `.list` | `proxmox-release-bullseye.gpg` |

For unknown codenames the script attempts to use the codename as the
repository suite with the legacy format.

After deploying, connect to the Proxmox VE web interface at
`https://your-ip-address:8006`. Proxmox VE requires hostname resolution to a
non-loopback IP address — make sure to configure `/etc/hosts` on the deployed
system (see the [Proxmox installation
guide](https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_13_Trixie)).

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

### Option use cases

- **`-k`** — confirm the split boot tarball actually contains a kernel and
  initramfs before deploying.
- **`-r`** — confirm rEFInd was staged so `setup-efi` will find it.
- **`--esproot`** — match a non-default ESP path used during the build.
- **`-z`** — verify ZFS userland, module, initramfs hook, and zed are present.
- **`-P`** — verify a Proxmox image: PVE kernel present, stock kernel gone,
  repos configured, enterprise repo disabled, dependencies present.

See [INTERNALS.md](INTERNALS.md) for the full list of what `test-rootfs`
checks.

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

See [INTERNALS.md](INTERNALS.md) for a step-by-step breakdown of what
`setup-efi` does.

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

## Deploying to a target system

### With a separate `/boot` partition

```bash
# 1. Create the rootfs and boot tarballs
./create-debian-rootfs -k -r bookworm

# 2. Partition the target disk
#    /dev/sda1: EFI System Partition (512MB, FAT32)
#    /dev/sda2: /boot partition (1GB, ext4)
#    /dev/sda3: Root filesystem (rest, ext4)

# 3. Format and mount
mkfs.ext4 /dev/sda3
mkfs.ext4 /dev/sda2
mkfs.vfat -F 32 /dev/sda1

mount /dev/sda3 /mnt/rootfs
mkdir -p /mnt/rootfs/boot
mount /dev/sda2 /mnt/rootfs/boot
mkdir -p /mnt/rootfs/boot/efi
mount /dev/sda1 /mnt/rootfs/boot/efi

# 4. Unpack rootfs (excludes /boot contents)
tar -xpf debian-bookworm-rootfs.tar.gz -C /mnt/rootfs

# 5. Unpack /boot to the boot partition
tar -xpf debian-bookworm-boot.tar.gz -C /mnt/rootfs/boot

# 6. Set up the EFI bootloader
sudo ./setup-efi -e /mnt/rootfs/boot/efi /mnt/rootfs

# 7. Configure root partition in rEFInd
# Edit /mnt/rootfs/boot/efi/EFI/refind/refind.conf
# Replace ROOT_PART with your root partition UUID (find with: blkid /dev/sda3)

# 8. Configure fstab
echo "UUID=$(blkid -o value -s UUID /dev/sda3)  /          ext4  defaults  0  1" >> /mnt/rootfs/etc/fstab
echo "UUID=$(blkid -o value -s UUID /dev/sda2)  /boot      ext4  defaults  0  2" >> /mnt/rootfs/etc/fstab
echo "UUID=$(blkid -o value -s UUID /dev/sda1)  /boot/efi  vfat  defaults  0  2" >> /mnt/rootfs/etc/fstab
```

### With `/boot` on the root partition

```bash
# 1. Create the rootfs and boot tarballs
./create-debian-rootfs -k -r bookworm

# 2. Unpack both to the same root
mkdir -p /mnt/rootfs
tar -xpf debian-bookworm-rootfs.tar.gz -C /mnt/rootfs
tar -xpf debian-bookworm-boot.tar.gz -C /mnt/rootfs

# (boot.tar.gz contains a 'boot/' prefix, so it merges correctly)
```

### Booting from ZFS

If you built with ZFS support (`-z`), the initramfs includes ZFS modules. To
boot from a ZFS root pool:

```bash
# After unpacking, configure ZFS in the initramfs
echo "PASS=rootpool" >> /mnt/rootfs/etc/initramfs-tools/conf.d/zfs

# Update the rEFInd config to load the ZFS module; the kernel line needs:
#   root=ZFS=rootpool/ROOT/debian
```
